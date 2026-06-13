# Kotlin Flow API — Android Interview Guide

Kotlin Flow is the reactive streams API built on top of coroutines. It models an
asynchronous sequence of values that are computed and emitted over time, with
full support for structured concurrency, backpressure, and cancellation. For
Android engineers it is the modern replacement for RxJava and `LiveData` for
streaming data from the data layer to the UI.

This guide covers the full Flow surface area asked in interviews: builders,
operators, collectors, dispatchers, cold vs hot flows, `StateFlow`/`SharedFlow`,
`callbackFlow`/`channelFlow`, `stateIn`/`shareIn`, exception handling,
lifecycle-aware collection, Retrofit/Room integration, and instant search.

All examples are accurate as of 2026 (kotlinx.coroutines 1.8+, AndroidX
Lifecycle 2.8+).

---

## Table of Contents

1. [What is a Flow? Builder, operator, collector](#1-what-is-a-flow-builder-operator-collector)
2. [Flow builders: flow, flowOf, asFlow](#2-flow-builders-flow-flowof-asflow)
3. [flowOn and dispatchers](#3-flowon-and-dispatchers)
4. [Intermediate operators: filter, map, transform](#4-intermediate-operators-filter-map-transform)
5. [Combining flows: zip and combine](#5-combining-flows-zip-and-combine)
6. [flatMapConcat, flatMapMerge, flatMapLatest](#6-flatmapconcat-flatmapmerge-flatmaplatest)
7. [retry and retryWhen](#7-retry-and-retrywhen)
8. [debounce, distinctUntilChanged, sample](#8-debounce-distinctuntilchanged-sample)
9. [Terminal operators](#9-terminal-operators)
10. [Cold Flow vs Hot Flow](#10-cold-flow-vs-hot-flow)
11. [StateFlow](#11-stateflow)
12. [SharedFlow](#12-sharedflow)
13. [callbackFlow](#13-callbackflow)
14. [channelFlow](#14-channelflow)
15. [stateIn vs shareIn](#15-statein-vs-sharein)
16. [collect vs collectLatest](#16-collect-vs-collectlatest)
17. [Exception handling: the catch operator](#17-exception-handling-the-catch-operator)
18. [Flow in Android: lifecycle-aware collection and repeatOnLifecycle](#18-flow-in-android-lifecycle-aware-collection-and-repeatonlifecycle)
19. [Retrofit with Flow](#19-retrofit-with-flow)
20. [Room with Flow](#20-room-with-flow)
21. [Instant search using Flow operators](#21-instant-search-using-flow-operators)

---

## 1. What is a Flow? Builder, operator, collector

**Flow** means a cold asynchronous data stream that produces multiple sequential values and executes only upon collection.

It is built on suspend functions,
so emission and collection are non-blocking and respect structured concurrency.

A Flow pipeline has three parts:

- **Builder** — produces the flow (`flow { }`, `flowOf()`, `asFlow()`).
- **Operator** — intermediate transformation applied lazily (`map`, `filter`,
  `flatMapLatest`). Operators return a new flow and do not trigger execution.
- **Collector** — terminal operator that starts the flow (`collect`, `toList`,
  `first`). The flow does nothing until collected.

```kotlin
val numbers: Flow<Int> = flow {          // builder
    for (i in 1..5) emit(i)
}

numbers
    .map { it * it }                      // operator (lazy)
    .filter { it % 2 == 1 }               // operator (lazy)
    .collect { println(it) }              // collector (starts the flow)
// 1, 9, 25
```

Key properties:

- **Cold** — the `flow { }` block runs from scratch for every collector.
- **Sequential** — by default values are emitted and processed one at a time in
  the same coroutine.
- **Context preservation** — emission must happen in the collector's coroutine
  context; you cannot `emit` from a different dispatcher inside `flow { }`
  (use `flowOn` instead).
- **Cancellable** — collection cooperates with coroutine cancellation.

**📚 Reference:** [Mastering Flow API in Kotlin](https://outcomeschool.com/blog/flow-api-in-kotlin)

---

## 2. Flow builders: flow, flowOf, asFlow

**Flow builders** means factory functions like `flow { }`, `flowOf()`, and `.asFlow()` used to construct new data stream instances.

flow { } — most general; can call suspend functions and emit
val f1 = flow {
    emit(fetchUser())          // suspend call allowed
    emit(fetchUser())
}

// 2. flowOf(...) — fixed set of known values
val f2 = flowOf(1, 2, 3)

// 3. asFlow() — convert a collection/sequence/range into a flow
val f3 = (1..3).asFlow()
val f4 = listOf("a", "b").asFlow()

// 4. channelFlow / callbackFlow — for concurrent or callback-based sources
//    (covered in sections 13 and 14)
```

Inside a `flow { }` builder you **must not** switch context manually with
`withContext`. Doing so throws an `IllegalStateException` because it violates
context preservation. Use `flowOn` instead.

```kotlin
// WRONG — throws "Flow invariant is violated"
flow {
    withContext(Dispatchers.IO) { emit(loadFile()) }
}

// CORRECT
flow { emit(loadFile()) }.flowOn(Dispatchers.IO)
```

**📚 Reference:** [Creating Flow Using Flow Builder in Kotlin](https://outcomeschool.com/blog/creating-flow-using-flow-builder-in-kotlin)

---

## 3. flowOn and dispatchers

**flowOn** means the operator used to change the execution context of upstream emissions while maintaining consumer context.

It
does not affect downstream operators or the collector. This is the Flow
equivalent of RxJava's `subscribeOn`.

```kotlin
flow {
    // runs on Dispatchers.IO
    emit(readFromDisk())
}
    .map { parse(it) }            // still IO (upstream of flowOn)
    .flowOn(Dispatchers.IO)
    .map { render(it) }           // runs in collector's context (e.g. Main)
    .collect { updateUi(it) }     // collector context
```

Notes:

- Multiple `flowOn` calls each affect only the segment above them.
- `flowOn` introduces a channel buffer between the two contexts, so the upstream
  can produce concurrently with downstream consumption.
- The collector's context determines where `collect` and downstream operators
  run. Launch collection in `Dispatchers.Main` (e.g. inside `lifecycleScope`)
  to update the UI directly.

There is no `observeOn` in Flow; the collector context plays that role, and
`flowOn` only shifts upstream.

**📚 Reference:** [Mastering Flow API in Kotlin](https://outcomeschool.com/blog/flow-api-in-kotlin)

---

## 4. Intermediate operators: filter, map, transform

**Intermediate operators** means lazy transformation functions (like `map`, `filter`, and `transform`) that return a new Flow without starting collection.

They run sequentially in the collector's context (unless moved
with `flowOn`).

```kotlin
flowOf(1, 2, 3, 4)
    .filter { it % 2 == 0 }     // keep evens
    .map { it * 10 }            // transform
    .collect { println(it) }    // 20, 40

// transform { } — emit any number of values per input (superset of map/filter)
flowOf(1, 2, 3)
    .transform { value ->
        emit("Pre-$value")
        emit("Post-$value")
    }
    .collect { println(it) }

// Other common ones:
// take(n), takeWhile, drop(n), onEach, onStart, onCompletion, scan, runningReduce
flow { emit(1); emit(2); emit(3) }
    .onStart { println("starting") }
    .onEach { println("got $it") }
    .onCompletion { cause -> println("done, cause=$cause") }
    .collect()
```

`onEach` is useful for side effects without consuming the flow; `onStart`/
`onCompletion` run before the first emission and after termination (including on
error/cancellation, where `cause` is non-null).

**📚 Reference:** [Mastering Flow API in Kotlin](https://outcomeschool.com/blog/flow-api-in-kotlin)

---

## 5. Combining flows: zip and combine

**zip vs combine** means the difference between pairing emissions index-for-index (zip), and pairing the latest emissions whenever either flow emits (combine).

`zip` pairs values **by index** — it waits for both flows to emit the n-th value
before producing a result, and completes when the shorter flow completes. It is
ideal for running parallel network calls and combining their results.

```kotlin
val users: Flow<User> = flow { emit(api.getUser()) }
val repos: Flow<List<Repo>> = flow { emit(api.getRepos()) }

users.zip(repos) { user, repoList ->
    UserWithRepos(user, repoList)
}.collect { /* both calls executed in parallel, combined once */ }
```

Because each flow runs in its own producing coroutine, `zip` lets two network
requests proceed concurrently and emits only when both have a value.

`combine` re-emits whenever **either** flow emits, pairing the newest value from
each. Use it when you want the latest of every source (e.g. combining UI filter
state with data).

```kotlin
combine(queryFlow, filterFlow) { query, filter ->
    SearchParams(query, filter)
}.collect { /* fires on every change to query OR filter */ }
```

| Operator  | Emits when            | Pairing       | Completes when     |
|-----------|-----------------------|---------------|--------------------|
| `zip`     | both have a new value | by index      | shorter completes  |
| `combine` | either emits          | latest of each | both complete      |

**📚 Reference:** [Kotlin Flow Zip Operator for Parallel Multiple Network Calls](https://outcomeschool.com/blog/kotlin-flow-zip-operator-parallel-multiple-network-calls), [Long-running tasks in parallel with Kotlin Flow](https://outcomeschool.com/blog/long-running-tasks-in-parallel-with-kotlin-flow)

---

## 6. flatMapConcat, flatMapMerge, flatMapLatest

**Flow flattening operators** means operators that transform emissions into new flows and merge them, handling them sequentially (flatMapConcat), concurrently (flatMapMerge), or by cancelling active ones when new emissions arrive (flatMapLatest).

They differ in how concurrent inner flows are handled.

```kotlin
fun requestDetails(id: Int): Flow<Detail> = flow {
    delay(100); emit(Detail(id))
}

val ids = flowOf(1, 2, 3)
```

- **flatMapConcat** — sequential. Collects each inner flow fully before starting
  the next. Order preserved, no concurrency.

```kotlin
ids.flatMapConcat { requestDetails(it) }.collect { /* 1 then 2 then 3 */ }
```

- **flatMapMerge** — concurrent. Subscribes to inner flows as values arrive and
  merges emissions; order not guaranteed. Concurrency is bounded by the
  `concurrency` parameter (default 16).

```kotlin
ids.flatMapMerge(concurrency = 3) { requestDetails(it) }
    .collect { /* interleaved, all run in parallel */ }
```

- **flatMapLatest** — cancellation-based. When a new upstream value arrives, the
  previous inner flow is **cancelled** and replaced. Only the latest matters —
  the canonical operator for instant search.

```kotlin
ids.flatMapLatest { requestDetails(it) }
    .collect { /* if upstream emits fast, earlier inner flows cancelled */ }
```

| Operator         | Concurrency | Order      | Cancels previous | Typical use            |
|------------------|-------------|------------|------------------|------------------------|
| `flatMapConcat`  | 1           | preserved  | no               | strict ordering        |
| `flatMapMerge`   | N (16)      | not kept   | no               | parallel independent   |
| `flatMapLatest`  | 1 active    | latest     | yes              | search, latest-wins    |

**📚 Reference:** [flatMapConcat, flatMapMerge, and flatMapLatest in Kotlin Flow](https://www.linkedin.com/posts/amit-shekhar-iitbhu_outcomeschool-softwareengineer-tech-activity-7316049746022801409-OoTY)

---

## 7. retry and retryWhen

**retry** means an operator that automatically re-subscribes to the upstream flow when an exception is thrown to handle transient errors.

Useful for transient
network errors.

```kotlin
flow { emit(api.fetch()) }
    .retry(retries = 3) { cause ->
        cause is IOException     // only retry on IO errors
    }
    .catch { e -> emit(fallback) }   // handle final failure
    .collect { render(it) }
```

`retryWhen` gives full control, exposing both the cause and the zero-based
attempt index, so you can implement exponential backoff:

```kotlin
flow { emit(api.fetch()) }
    .retryWhen { cause, attempt ->
        if (cause is IOException && attempt < 3) {
            delay(1000L * (attempt + 1))   // backoff: 1s, 2s, 3s
            true                            // retry
        } else {
            false                           // give up, propagate
        }
    }
    .collect { ... }
```

Important: `retry` resubscribes to the **upstream** of the operator. Place it
above any operators whose side effects you do not want repeated.

**📚 Reference:** [Retry Operator in Kotlin Flow](https://outcomeschool.com/blog/retry-operator-in-kotlin-flow)

---

## 8. debounce, distinctUntilChanged, sample

**Flow filtering operators** means rate-limiting operations that filter emissions by time delay (debounce), deduplicate consecutive values (distinctUntilChanged), or emit periodic snapshots (sample).

- **debounce(timeout)** — emits a value only if `timeout` ms have passed without
  a newer value. Filters out rapid bursts (e.g. fast typing).

```kotlin
queryFlow.debounce(300).collect { search(it) }
```

- **distinctUntilChanged()** — suppresses consecutive duplicate values. Prevents
  redundant work (e.g. user deletes a char and retypes the same query).

```kotlin
queryFlow.distinctUntilChanged().collect { ... }
```

- **sample(period)** — emits the most recent value at fixed intervals,
  discarding intermediate ones. Useful for high-frequency streams like sensors.

```kotlin
sensorFlow.sample(1000).collect { plot(it) }   // at most one per second
```

`debounce` waits for quiet; `sample` emits on a clock regardless of quiet
periods.

**📚 Reference:** [Instant Search Using Kotlin Flow Operators](https://outcomeschool.com/blog/instant-search-using-kotlin-flow-operators)

---

## 9. Terminal operators

**Terminal operators** means suspending functions (like `collect`, `first`, or `toList`) that start the collection of a Flow and return a result.

Nothing runs until a terminal operator is invoked.

```kotlin
// collect — most common, processes each emission
flow.collect { value -> handle(value) }

// toList / toSet — gather into a collection
val list: List<Int> = flowOf(1, 2, 3).toList()

// first / firstOrNull — take the first emission then cancel upstream
val head = flow.first()
val headOrNull = flow.firstOrNull { it > 10 }

// single — expects exactly one value, throws otherwise
val only = flowOf(42).single()

// reduce / fold — accumulate to a single value
val sum = flowOf(1, 2, 3).reduce { acc, v -> acc + v }   // 6
val foldSum = flowOf(1, 2, 3).fold(100) { acc, v -> acc + v }  // 106

// count
val n = flowOf(1, 2, 3).count()

// launchIn — collect in a given scope without a lambda body; pairs with onEach
flow.onEach { handle(it) }.launchIn(viewModelScope)
```

`launchIn(scope)` is shorthand for `scope.launch { flow.collect() }` and is the
idiomatic way to start a flow tied to a scope while keeping the pipeline
declarative.

**📚 Reference:** [Terminal Operators in Kotlin Flow](https://outcomeschool.com/blog/terminal-operators-in-kotlin-flow)

---

## 10. Cold Flow vs Hot Flow

**Cold flow vs Hot flow** means the distinction between streams that run their producer block from scratch for each collector (cold), and streams that broadcast values to multiple collectors concurrently (hot).

`flow { }`, `flowOf`, Room
queries, and Retrofit flows are cold.

A **hot** flow produces values regardless of collectors and shares a single
stream among all collectors. `StateFlow` and `SharedFlow` are hot. Late
collectors may miss earlier emissions (subject to replay configuration).

```kotlin
// COLD: block re-runs for each collector
val cold = flow {
    println("producing")
    emit(1)
}
cold.collect { }   // prints "producing"
cold.collect { }   // prints "producing" AGAIN

// HOT: shared, runs once, independent of collectors
val hot = MutableStateFlow(0)
hot.value = 1      // emitted whether or not anyone collects
```

| Aspect           | Cold Flow                  | Hot Flow (`StateFlow`/`SharedFlow`) |
|------------------|----------------------------|-------------------------------------|
| Starts emitting  | on collection              | independent of collectors           |
| Per-collector    | new independent run        | shared single stream                |
| Examples         | `flow{}`, Room, Retrofit   | `StateFlow`, `SharedFlow`           |
| Missed values    | impossible (starts fresh)  | possible for late collectors        |

**📚 Reference:** [Cold Flow vs Hot Flow](https://outcomeschool.com/blog/cold-flow-vs-hot-flow)

---

## 11. StateFlow

**SharedFlow** means a hot flow that broadcasts values to all active collectors and does not retain state by default.

It is the coroutine-native replacement for
`LiveData` for UI state.

```kotlin
class CounterViewModel : ViewModel() {
    private val _state = MutableStateFlow(UiState.Loading)
    val state: StateFlow<UiState> = _state.asStateFlow()   // read-only exposure

    fun load() = viewModelScope.launch {
        _state.value = UiState.Loading
        _state.value = UiState.Success(repo.fetch())
    }
}
```

Characteristics:

- **Always has a value** — requires an initial value at construction.
- **Conflated** — only the latest value is retained; fast updates may skip
  intermediate values for slow collectors.
- **`distinctUntilChanged` built in** — re-setting an equal value (by `equals`)
  does not emit. Use `data class` state so structural equality works.
- **Never completes** — collection runs until the scope is cancelled.

Use `StateFlow` for observable state (the current screen state). Always expose
`asStateFlow()` to prevent external mutation.

**📚 Reference:** [StateFlow and SharedFlow](https://outcomeschool.com/blog/stateflow-and-sharedflow)

---

## 12. SharedFlow

**StateFlow** means a hot, state-holding flow that always retains its latest value and emits it to new collectors.

Unlike `StateFlow` it has no current value, supports configurable
replay, and does not conflate by default.

```kotlin
class EventViewModel : ViewModel() {
    private val _events = MutableSharedFlow<Event>(
        replay = 0,                                  // no replay for one-shot events
        extraBufferCapacity = 1,
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )
    val events: SharedFlow<Event> = _events.asSharedFlow()

    fun onClick() = viewModelScope.launch {
        _events.emit(Event.NavigateNext)            // suspends if buffer full and no overflow policy
    }
}
```

Use cases:

- One-time UI events (navigation, snackbar, toast) — `replay = 0`.
- A `StateFlow` is literally a `SharedFlow` with `replay = 1` and conflation.

`emit` suspends when the buffer is full; `tryEmit` is non-suspending and returns
`false` if it could not add the value. Configure `onBufferOverflow` to control
backpressure behaviour.

**📚 Reference:** [StateFlow and SharedFlow](https://outcomeschool.com/blog/stateflow-and-sharedflow)

---

## 13. callbackFlow

**conflate** means a Flow operator that skips intermediate emissions to deliver only the latest value when the collector is slower than the emitter.

It runs in a coroutine and lets you
emit from callbacks via a `SendChannel`.

```kotlin
fun locationUpdates(client: FusedLocationProviderClient): Flow<Location> =
    callbackFlow {
        val callback = object : LocationCallback() {
            override fun onLocationResult(result: LocationResult) {
                result.lastLocation?.let { trySend(it) }   // emit into the flow
            }
        }
        client.requestLocationUpdates(request, callback, Looper.getMainLooper())

        // awaitClose runs when the flow is cancelled/completed — cleanup here
        awaitClose { client.removeLocationUpdates(callback) }
    }
```

Key points:

- Use `trySend`/`send` to emit; `send` is suspending, `trySend` is not.
- **`awaitClose` is mandatory** — it suspends until the flow is closed and is
  where you unregister the callback. Omitting it throws
  `IllegalStateException`.
- `callbackFlow` is buffered and can be safely emitted to from other threads,
  which is exactly what callbacks need.

**📚 Reference:** [callbackFlow - Callback to Flow API in Kotlin](https://outcomeschool.com/blog/callback-to-flow-api-in-kotlin)

---

## 14. channelFlow

**buffer** means a Flow operator that runs the collector and emitter in separate coroutines to process emissions concurrently.

It provides a channel-backed flow whose block
can `launch` child coroutines that all emit via `send`.

```kotlin
fun mergedData(): Flow<Data> = channelFlow {
    launch { send(api.sourceA()) }   // concurrent producers
    launch { send(api.sourceB()) }
    // channel closes when the block (and its children) complete
}
```

`channelFlow` vs plain `flow { }`:

- A plain `flow { }` is sequential and **cannot** emit from a different
  coroutine/thread — emission is confined to the builder's coroutine.
- `channelFlow` allows concurrent `send` from multiple launched coroutines and
  is thread-safe.

`callbackFlow` is technically a specialization of `channelFlow` tuned for
callbacks (it requires `awaitClose`). Use `channelFlow` when you spawn parallel
producers; use `callbackFlow` for callback registration.

**📚 Reference:** [Mastering Flow API in Kotlin](https://outcomeschool.com/blog/flow-api-in-kotlin)

---

## 15. stateIn vs shareIn

**stateIn vs shareIn** means the choice between converting a cold flow into a StateFlow holding a state (stateIn), and converting it into a SharedFlow broadcasting events (shareIn).

a Room query) among multiple
collectors without re-running it per collector.

- **`stateIn`** produces a `StateFlow` — has a current value, conflated, requires
  an initial value.

```kotlin
val uiState: StateFlow<List<Item>> = repository.itemsFlow()  // cold
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5_000),
        initialValue = emptyList()
    )
```

- **`shareIn`** produces a `SharedFlow` — no current value, configurable
  `replay`.

```kotlin
val events: SharedFlow<Event> = repository.eventStream()
    .shareIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5_000),
        replay = 0
    )
```

`SharingStarted` strategies:

- `Eagerly` — starts immediately, never stops.
- `Lazily` — starts on first subscriber, never stops.
- `WhileSubscribed(stopTimeoutMillis)` — starts on first subscriber, stops after
  the last unsubscribes plus the timeout. The 5-second timeout is the Android
  recommendation: it survives short configuration changes/rotation without
  restarting the upstream, while still cancelling work when the screen is truly
  gone.

| Operator  | Produces     | Current value | Replay        | Initial value required |
|-----------|--------------|---------------|---------------|------------------------|
| `stateIn` | `StateFlow`  | yes           | always 1      | yes                    |
| `shareIn` | `SharedFlow` | no            | configurable  | no                     |

Use `stateIn` for UI state; use `shareIn` for shared event/data streams.

**📚 Reference:** [stateIn vs shareIn in Kotlin Flow](https://www.linkedin.com/posts/outcomeschool_statein-vs-sharein-in-kotlin-flow-activity-7315963238318321664-BdYk)

---

## 16. collect vs collectLatest

**collect vs collectLatest** means the choice between processing every emission sequentially (collect), and cancelling the active processing block when a new emission arrives (collectLatest).

- **`collect`** processes each value to completion before accepting the next.
  Slow processing applies backpressure to the producer.

```kotlin
flow.collect { value ->
    process(value)   // each fully completes before next
}
```

- **`collectLatest`** **cancels** the in-progress block when a new value arrives
  and restarts it with the latest value. The latest value always wins.

```kotlin
flow.collectLatest { value ->
    delay(100)          // cancelled if a newer value arrives
    updateUi(value)
}
```

Use `collectLatest` when only the most recent value matters and stale processing
should be abandoned (e.g. rendering the latest search query). Use `collect` when
every value must be fully handled. There are analogous `mapLatest` and
`transformLatest` operators with the same cancel-and-restart semantics.

**📚 Reference:** [collect vs collectLatest in Kotlin Flow](https://www.linkedin.com/posts/amit-shekhar-iitbhu_androiddev-kotlin-activity-7371769230389731328-UXPY)

---

## 17. Exception handling: the catch operator

**Flow exception transparency** means handling exceptions using the `catch` operator to catch upstream errors while letting downstream collector errors propagate.

The `catch` operator catches exceptions from **upstream** only. It can log,
emit a fallback value, or rethrow.

```kotlin
flow { emit(api.fetch()) }
    .map { transform(it) }
    .catch { e ->                       // catches errors from upstream (flow + map)
        Log.e("TAG", "failed", e)
        emit(fallbackValue)             // optional recovery emission
    }
    .collect { render(it) }
```

Important rules:

- `catch` does **not** catch exceptions thrown inside `collect`. Wrap the
  collector body in `try/catch`, or move logic into `onEach` placed above
  `catch`.

```kotlin
flow { ... }
    .onEach { mightThrow(it) }   // exception now visible to catch below
    .catch { e -> handle(e) }
    .collect()
```

- Never wrap `emit` in a `try/catch` that swallows downstream
  `CancellationException` — it breaks cooperative cancellation. Use the `catch`
  operator instead.
- `catch` only handles upstream; pair it with `retry`/`retryWhen` for retries
  and `onCompletion` for cleanup (note `onCompletion` sees the cause but does not
  handle it).

**📚 Reference:** [Exception Handling in Kotlin Flow](https://outcomeschool.com/blog/exception-handling-in-kotlin-flow)

---

## 18. Flow in Android: lifecycle-aware collection and repeatOnLifecycle

**repeatOnLifecycle** means a lifecycle-aware utility that automatically collects flows when the UI is active and pauses collection when the UI is in the background.

The recommended pattern is `repeatOnLifecycle`, which
starts collection when the lifecycle reaches the given state and **cancels** it
when it drops below — automatically restarting on the next entry.

```kotlin
class MyFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    render(state)        // only collects while at least STARTED
                }
            }
        }
    }
}
```

Notes:

- `repeatOnLifecycle(STARTED)` is the standard choice: collection is active
  between `onStart` and `onStop`.
- For multiple flows, either nest separate `launch { }` blocks inside one
  `repeatOnLifecycle`, or use `flowWithLifecycle` for a single flow:

```kotlin
viewModel.events
    .flowWithLifecycle(viewLifecycleOwner.lifecycle, Lifecycle.State.STARTED)
    .onEach { handleEvent(it) }
    .launchIn(viewLifecycleOwner.lifecycleScope)
```

- In **Jetpack Compose**, use `collectAsStateWithLifecycle()` (from
  `lifecycle-runtime-compose`), which applies the same lifecycle-aware semantics
  and is preferred over `collectAsState()`.

```kotlin
@Composable
fun Screen(viewModel: MyViewModel) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    Content(state)
}
```

- In Fragments, always use `viewLifecycleOwner`, not the Fragment's own
  lifecycle, to match the view's lifecycle.

**📚 Reference:** [Mastering Flow API in Kotlin](https://outcomeschool.com/blog/flow-api-in-kotlin)

---

## 19. Retrofit with Flow

**Retrofit Flow integration** means defining API service methods that return Flow instances to handle network streams reactively.

The flow
is cold — the network call fires on collection.

```kotlin
interface ApiService {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): User      // suspend (single shot)

    @GET("feed")
    fun getFeed(): Flow<List<Post>>                         // Flow return type
}
```

A common repository pattern wraps the call, moves it to IO, and handles errors:

```kotlin
fun user(id: String): Flow<Result<User>> = flow {
    emit(Result.success(api.getUser(id)))
}
    .catch { e -> emit(Result.failure(e)) }
    .flowOn(Dispatchers.IO)
```

In the ViewModel, convert to UI state with `stateIn`:

```kotlin
val userState = user(id)
    .map { result -> result.fold(::Success, ::Error) }
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), Loading)
```

Trade-off: for a one-shot request a `suspend` function is simpler; return `Flow`
when you want operator composition (`retry`, `combine`, `flowOn`) or a stream of
updates.

**📚 Reference:** [Retrofit with Kotlin Flow](https://outcomeschool.com/blog/retrofit-with-kotlin-flow)

---

## 20. Room with Flow

**Room Flow integration** means defining DAO query methods that return Flow to observe database table changes reactively.

Such a flow is **observable**: Room
automatically re-runs the query and re-emits whenever the underlying tables
change, making it perfect for a single source of truth.

```kotlin
@Dao
interface UserDao {
    @Query("SELECT * FROM users ORDER BY name")
    fun observeUsers(): Flow<List<User>>      // re-emits on table change

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(user: User)            // suspend for one-shot writes
}
```

Characteristics:

- The returned flow is cold and lifecycle-friendly; Room runs the query on a
  background executor, so you generally do not need `flowOn` for the query
  itself.
- It emits an initial value immediately on collection, then on every relevant
  write.
- Combine with `distinctUntilChanged()` if you want to suppress identical
  consecutive results.

ViewModel usage:

```kotlin
val users: StateFlow<List<User>> = userDao.observeUsers()
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())
```

This gives the UI a reactive, always-current list that updates automatically as
the database changes, with the upstream query stopping when no one is watching.

**📚 Reference:** [Room Database with Kotlin Flow](https://outcomeschool.com/blog/room-database-with-kotlin-flow)

---

## 21. Instant search using Flow operators

**Instant search implementation** means combining `debounce`, `filter`, `distinctUntilChanged`, and `flatMapLatest` operators to build responsive search-as-you-type flows.

```kotlin
class SearchViewModel(private val repo: SearchRepository) : ViewModel() {

    private val queryFlow = MutableStateFlow("")

    fun onQueryChanged(query: String) {
        queryFlow.value = query
    }

    val results: StateFlow<SearchUiState> = queryFlow
        .debounce(300)                       // wait for the user to pause typing
        .filter { it.length >= 2 }           // ignore very short queries
        .distinctUntilChanged()              // skip duplicate consecutive queries
        .flatMapLatest { query ->            // cancel the previous in-flight search
            repo.search(query)               // returns Flow<List<Result>>
                .map { SearchUiState.Success(it) as SearchUiState }
                .onStart { emit(SearchUiState.Loading) }
                .catch { emit(SearchUiState.Error(it.message)) }
        }
        .stateIn(
            viewModelScope,
            SharingStarted.WhileSubscribed(5_000),
            SearchUiState.Idle
        )
}
```

Why each operator matters:

- **`debounce(300)`** — avoids firing a request on every keystroke; only searches
  after the user pauses.
- **`filter { length >= 2 }`** — skips wasteful queries for one character.
- **`distinctUntilChanged()`** — avoids re-querying when the effective text is
  unchanged (e.g. delete + retype).
- **`flatMapLatest`** — cancels the previous network call when a new query
  arrives, so stale results never overwrite fresh ones. This is the key
  correctness operator.
- **`catch` / `onStart`** — per-query loading and error states without tearing
  down the outer pipeline.
- **`stateIn`** — exposes the result as lifecycle-friendly observable state.

This pipeline is responsive, network-efficient, and free of race conditions
where an older slow response clobbers a newer one.

**📚 Reference:** [Instant Search Using Kotlin Flow Operators](https://outcomeschool.com/blog/instant-search-using-kotlin-flow-operators), [Unit Testing ViewModel with Kotlin Flow and StateFlow](https://outcomeschool.com/blog/unit-testing-viewmodel-with-kotlin-flow-and-stateflow)
