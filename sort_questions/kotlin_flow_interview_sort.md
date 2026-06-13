# Kotlin Flow API — Short Answers (Quick Revision)

> Condensed answers for rapid review before an interview. For full explanations and code see `kotlin_flow_interview.md`.

---

## 1. What is a Flow? Builder, operator, collector

**Flow** means a cold asynchronous data stream that produces multiple sequential values and executes only upon collection.

A `Flow<T>` is a cold asynchronous stream that emits values over time, built on suspend functions. Three parts: a **builder** produces it (`flow {}`), **operators** transform it lazily (`map`, `filter`), and a **collector** (terminal, e.g. `collect`) starts it. Cold, sequential, context-preserving, cancellable.

---



## 2. Flow builders: flow, flowOf, asFlow

**Flow builders** means factory functions like `flow { }`, `flowOf()`, and `.asFlow()` used to construct new data stream instances.

`flow {}` — general, can call suspend functions and `emit`. `flowOf(...)` — fixed known values. `asFlow()` — convert a collection/range/sequence. Don't call `withContext` inside `flow {}` (violates context preservation, throws) — use `flowOn` instead.

---



## 3. flowOn and dispatchers

**flowOn** means the operator used to change the execution context of upstream emissions while maintaining consumer context.

`flowOn(ctx)` changes the context/dispatcher of the **upstream** (builder + operators above it) only, not downstream or the collector — the Flow equivalent of RxJava's `subscribeOn`. The collector's context determines where `collect` runs (there's no `observeOn`). It also adds a channel buffer between contexts.

---



## 4. Intermediate operators: filter, map, transform

**Intermediate operators** means lazy transformation functions (like `map`, `filter`, and `transform`) that return a new Flow without starting collection.

Lazy transformations that run in the collector's context when collected. `filter` keeps matching values; `map` transforms one-to-one; `transform {}` can emit any number per input (superset). Others: `take`, `drop`, `onEach` (side effects), `onStart`, `onCompletion` (runs on termination, `cause` non-null on error).

---



## 5. Combining flows: zip and combine

**zip vs combine** means the difference between pairing emissions index-for-index (zip), and pairing the latest emissions whenever either flow emits (combine).

`zip` pairs values by index, waiting for both to emit the n-th value, completing when the shorter completes — good for parallel calls combined once. `combine` re-emits whenever **either** flow emits, pairing the latest of each — good for combining latest state/filters.

---



## 6. flatMapConcat, flatMapMerge, flatMapLatest

**Flow flattening operators** means operators that transform emissions into new flows and merge them, handling them sequentially (flatMapConcat), concurrently (flatMapMerge), or by cancelling active ones when new emissions arrive (flatMapLatest).

Each maps a value to a flow and flattens. `flatMapConcat` — sequential, order preserved, no concurrency. `flatMapMerge` — concurrent (bounded by `concurrency`, default 16), order not kept. `flatMapLatest` — cancels the previous inner flow when a new value arrives (canonical for instant search).

---



## 7. retry and retryWhen

**retry** means an operator that automatically re-subscribes to the upstream flow when an exception is thrown to handle transient errors.

`retry(retries) { cause -> ... }` re-subscribes to the upstream on failure (filter by exception type). `retryWhen { cause, attempt -> ... }` exposes the attempt index for exponential backoff (`delay`). Place above operators whose side effects shouldn't repeat; pair with `catch` for final failure.

---



## 8. debounce, distinctUntilChanged, sample

**Flow filtering operators** means rate-limiting operations that filter emissions by time delay (debounce), deduplicate consecutive values (distinctUntilChanged), or emit periodic snapshots (sample).

`debounce(timeout)` emits only after a quiet period (filters rapid typing bursts). `distinctUntilChanged()` suppresses consecutive duplicates. `sample(period)` emits the latest value at fixed intervals (high-frequency streams). `debounce` waits for quiet; `sample` emits on a clock.

---



## 9. Terminal operators

**Terminal operators** means suspending functions (like `collect`, `first`, or `toList`) that start the collection of a Flow and return a result.

Suspend functions that start collection and return a result; nothing runs until invoked. `collect` (per-emission), `toList`/`toSet`, `first`/`firstOrNull`, `single`, `reduce`/`fold`, `count`, and `launchIn(scope)` (shorthand for `scope.launch { flow.collect() }`, pairs with `onEach`).

---



## 10. Cold Flow vs Hot Flow

**Cold flow vs Hot flow** means the distinction between streams that run their producer block from scratch for each collector (cold), and streams that broadcast values to multiple collectors concurrently (hot).

Cold flows (`flow {}`, `flowOf`, Room, Retrofit) start on collection and re-run independently per collector — late collectors miss nothing. Hot flows (`StateFlow`, `SharedFlow`) produce regardless of collectors and share one stream — late collectors may miss earlier emissions (subject to replay).

---



## 11. StateFlow

**SharedFlow** means a hot flow that broadcasts values to all active collectors and does not retain state by default.

A hot state-holder flow that always has a current value and emits the latest to new collectors — the coroutine-native `LiveData` replacement. Requires an initial value, is conflated (only latest retained), has built-in `distinctUntilChanged`, and never completes. Expose via `asStateFlow()` to prevent external mutation.

---



## 12. SharedFlow

**StateFlow** means a hot, state-holding flow that always retains its latest value and emits it to new collectors.

A hot flow for broadcasting events to multiple collectors. No current value, configurable `replay`, not conflated by default. Use `replay = 0` for one-time UI events (navigation, snackbar). `emit` suspends when the buffer is full; `tryEmit` is non-suspending. A `StateFlow` is a `SharedFlow` with `replay = 1` + conflation.

---



## 13. callbackFlow

**conflate** means a Flow operator that skips intermediate emissions to deliver only the latest value when the collector is slower than the emitter.

Bridges callback-based APIs (location, Firebase, sensors) into a Flow, emitting from callbacks via `trySend`/`send`. `awaitClose { }` is **mandatory** — it suspends until closed and is where you unregister the callback (omitting it throws). Buffered and safe to emit to from other threads.

---



## 14. channelFlow

**buffer** means a Flow operator that runs the collector and emitter in separate coroutines to process emissions concurrently.

A channel-backed flow for **concurrent** emission from multiple launched coroutines (all emit via `send`); thread-safe. Unlike plain `flow {}` (sequential, emission confined to one coroutine), `channelFlow` allows concurrent `send`. `callbackFlow` is a specialization for callbacks; use `channelFlow` for parallel producers.

---



## 15. stateIn vs shareIn

**stateIn vs shareIn** means the choice between converting a cold flow into a StateFlow holding a state (stateIn), and converting it into a SharedFlow broadcasting events (shareIn).

Both convert a cold flow into a hot shared flow tied to a scope (share one upstream among collectors). `stateIn` → `StateFlow` (current value, conflated, needs initial value). `shareIn` → `SharedFlow` (no current value, configurable replay). `SharingStarted`: `Eagerly`, `Lazily`, `WhileSubscribed(5_000)` (Android default, survives rotation).

---



## 16. collect vs collectLatest

**collect vs collectLatest** means the choice between processing every emission sequentially (collect), and cancelling the active processing block when a new emission arrives (collectLatest).

`collect` processes each value to completion before the next (applies backpressure). `collectLatest` cancels the in-progress block when a new value arrives and restarts with the latest — for when only the most recent matters (latest search query). Analogous: `mapLatest`, `transformLatest`.

---



## 17. Exception handling: the catch operator

**Flow exception transparency** means handling exceptions using the `catch` operator to catch upstream errors while letting downstream collector errors propagate.

The `catch` operator handles **upstream** exceptions only (can log, emit a fallback, or rethrow); it does **not** catch errors thrown inside `collect` — wrap the collector in `try/catch` or move logic into `onEach` above `catch`. Don't swallow `CancellationException`. Pair with `retry` and `onCompletion`.

---



## 18. Flow in Android: lifecycle-aware collection and repeatOnLifecycle

**repeatOnLifecycle** means a lifecycle-aware utility that automatically collects flows when the UI is active and pauses collection when the UI is in the background.

Collecting directly in `lifecycleScope.launch {}` keeps running in the background (wasteful/crash risk). Use `repeatOnLifecycle(Lifecycle.State.STARTED)` to start collection at STARTED and cancel below it, restarting on re-entry. Use `flowWithLifecycle` for a single flow, and `collectAsStateWithLifecycle()` in Compose. In Fragments use `viewLifecycleOwner`.

---



## 19. Retrofit with Flow

**Retrofit Flow integration** means defining API service methods that return Flow instances to handle network streams reactively.

Retrofit supports `suspend` functions and `Flow` return types; the flow is cold (call fires on collection). A common repo pattern: `flow { emit(api.get()) }.catch { emit(failure) }.flowOn(Dispatchers.IO)`, then `stateIn` in the ViewModel. Use `suspend` for one-shot; `Flow` for operator composition or streams.

---



## 20. Room with Flow

**Room Flow integration** means defining DAO query methods that return Flow to observe database table changes reactively.

Room can return a `Flow` from a `@Query` — it's observable, automatically re-running and re-emitting whenever the underlying tables change (single source of truth). Cold and lifecycle-friendly, runs on a background executor (no `flowOn` needed), emits initial value then on writes. Expose via `stateIn(viewModelScope, WhileSubscribed(5_000), emptyList())`.

---



## 21. Instant search using Flow operators

**Instant search implementation** means combining `debounce`, `filter`, `distinctUntilChanged`, and `flatMapLatest` operators to build responsive search-as-you-type flows.

Chain: `debounce(300)` (wait for typing pause) → `filter { length >= 2 }` (skip short) → `distinctUntilChanged()` (skip duplicates) → `flatMapLatest { repo.search(it) }` (cancel stale in-flight search — the key correctness operator) with per-query `onStart`/`catch` → `stateIn` (lifecycle-friendly state). Responsive, network-efficient, race-free.

