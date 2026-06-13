# Kotlin Flow API — Short Answers (Quick Revision)

> Condensed answers for rapid review before an interview. For full explanations and code see `kotlin_flow_interview.md`.

---

## 1. What is a Flow? Builder, operator, collector

A `Flow<T>` is a cold asynchronous stream that emits values over time, built on suspend functions. Three parts: a **builder** produces it (`flow {}`), **operators** transform it lazily (`map`, `filter`), and a **collector** (terminal, e.g. `collect`) starts it. Cold, sequential, context-preserving, cancellable.

---

## 2. Flow builders: flow, flowOf, asFlow

`flow {}` — general, can call suspend functions and `emit`. `flowOf(...)` — fixed known values. `asFlow()` — convert a collection/range/sequence. Don't call `withContext` inside `flow {}` (violates context preservation, throws) — use `flowOn` instead.

---

## 3. flowOn and dispatchers

`flowOn(ctx)` changes the context/dispatcher of the **upstream** (builder + operators above it) only, not downstream or the collector — the Flow equivalent of RxJava's `subscribeOn`. The collector's context determines where `collect` runs (there's no `observeOn`). It also adds a channel buffer between contexts.

---

## 4. Intermediate operators: filter, map, transform

Lazy transformations that run in the collector's context when collected. `filter` keeps matching values; `map` transforms one-to-one; `transform {}` can emit any number per input (superset). Others: `take`, `drop`, `onEach` (side effects), `onStart`, `onCompletion` (runs on termination, `cause` non-null on error).

---

## 5. Combining flows: zip and combine

`zip` pairs values by index, waiting for both to emit the n-th value, completing when the shorter completes — good for parallel calls combined once. `combine` re-emits whenever **either** flow emits, pairing the latest of each — good for combining latest state/filters.

---

## 6. flatMapConcat, flatMapMerge, flatMapLatest

Each maps a value to a flow and flattens. `flatMapConcat` — sequential, order preserved, no concurrency. `flatMapMerge` — concurrent (bounded by `concurrency`, default 16), order not kept. `flatMapLatest` — cancels the previous inner flow when a new value arrives (canonical for instant search).

---

## 7. retry and retryWhen

`retry(retries) { cause -> ... }` re-subscribes to the upstream on failure (filter by exception type). `retryWhen { cause, attempt -> ... }` exposes the attempt index for exponential backoff (`delay`). Place above operators whose side effects shouldn't repeat; pair with `catch` for final failure.

---

## 8. debounce, distinctUntilChanged, sample

`debounce(timeout)` emits only after a quiet period (filters rapid typing bursts). `distinctUntilChanged()` suppresses consecutive duplicates. `sample(period)` emits the latest value at fixed intervals (high-frequency streams). `debounce` waits for quiet; `sample` emits on a clock.

---

## 9. Terminal operators

Suspend functions that start collection and return a result; nothing runs until invoked. `collect` (per-emission), `toList`/`toSet`, `first`/`firstOrNull`, `single`, `reduce`/`fold`, `count`, and `launchIn(scope)` (shorthand for `scope.launch { flow.collect() }`, pairs with `onEach`).

---

## 10. Cold Flow vs Hot Flow

Cold flows (`flow {}`, `flowOf`, Room, Retrofit) start on collection and re-run independently per collector — late collectors miss nothing. Hot flows (`StateFlow`, `SharedFlow`) produce regardless of collectors and share one stream — late collectors may miss earlier emissions (subject to replay).

---

## 11. StateFlow

A hot state-holder flow that always has a current value and emits the latest to new collectors — the coroutine-native `LiveData` replacement. Requires an initial value, is conflated (only latest retained), has built-in `distinctUntilChanged`, and never completes. Expose via `asStateFlow()` to prevent external mutation.

---

## 12. SharedFlow

A hot flow for broadcasting events to multiple collectors. No current value, configurable `replay`, not conflated by default. Use `replay = 0` for one-time UI events (navigation, snackbar). `emit` suspends when the buffer is full; `tryEmit` is non-suspending. A `StateFlow` is a `SharedFlow` with `replay = 1` + conflation.

---

## 13. callbackFlow

Bridges callback-based APIs (location, Firebase, sensors) into a Flow, emitting from callbacks via `trySend`/`send`. `awaitClose { }` is **mandatory** — it suspends until closed and is where you unregister the callback (omitting it throws). Buffered and safe to emit to from other threads.

---

## 14. channelFlow

A channel-backed flow for **concurrent** emission from multiple launched coroutines (all emit via `send`); thread-safe. Unlike plain `flow {}` (sequential, emission confined to one coroutine), `channelFlow` allows concurrent `send`. `callbackFlow` is a specialization for callbacks; use `channelFlow` for parallel producers.

---

## 15. stateIn vs shareIn

Both convert a cold flow into a hot shared flow tied to a scope (share one upstream among collectors). `stateIn` → `StateFlow` (current value, conflated, needs initial value). `shareIn` → `SharedFlow` (no current value, configurable replay). `SharingStarted`: `Eagerly`, `Lazily`, `WhileSubscribed(5_000)` (Android default, survives rotation).

---

## 16. collect vs collectLatest

`collect` processes each value to completion before the next (applies backpressure). `collectLatest` cancels the in-progress block when a new value arrives and restarts with the latest — for when only the most recent matters (latest search query). Analogous: `mapLatest`, `transformLatest`.

---

## 17. Exception handling: the catch operator

The `catch` operator handles **upstream** exceptions only (can log, emit a fallback, or rethrow); it does **not** catch errors thrown inside `collect` — wrap the collector in `try/catch` or move logic into `onEach` above `catch`. Don't swallow `CancellationException`. Pair with `retry` and `onCompletion`.

---

## 18. Flow in Android: lifecycle-aware collection and repeatOnLifecycle

Collecting directly in `lifecycleScope.launch {}` keeps running in the background (wasteful/crash risk). Use `repeatOnLifecycle(Lifecycle.State.STARTED)` to start collection at STARTED and cancel below it, restarting on re-entry. Use `flowWithLifecycle` for a single flow, and `collectAsStateWithLifecycle()` in Compose. In Fragments use `viewLifecycleOwner`.

---

## 19. Retrofit with Flow

Retrofit supports `suspend` functions and `Flow` return types; the flow is cold (call fires on collection). A common repo pattern: `flow { emit(api.get()) }.catch { emit(failure) }.flowOn(Dispatchers.IO)`, then `stateIn` in the ViewModel. Use `suspend` for one-shot; `Flow` for operator composition or streams.

---

## 20. Room with Flow

Room can return a `Flow` from a `@Query` — it's observable, automatically re-running and re-emitting whenever the underlying tables change (single source of truth). Cold and lifecycle-friendly, runs on a background executor (no `flowOn` needed), emits initial value then on writes. Expose via `stateIn(viewModelScope, WhileSubscribed(5_000), emptyList())`.

---

## 21. Instant search using Flow operators

Chain: `debounce(300)` (wait for typing pause) → `filter { length >= 2 }` (skip short) → `distinctUntilChanged()` (skip duplicates) → `flatMapLatest { repo.search(it) }` (cancel stale in-flight search — the key correctness operator) with per-query `onStart`/`catch` → `stateIn` (lifecycle-friendly state). Responsive, network-efficient, race-free.
