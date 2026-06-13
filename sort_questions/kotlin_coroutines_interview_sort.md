# Kotlin Coroutines — Short Answers (Quick Revision)

> Condensed answers for rapid review before an interview. For full explanations and code see `kotlin_coroutines_interview.md`.

---

## 1. What is a coroutine?

An instance of a suspendable computation — a block of code that can suspend (pause) without blocking the thread and later resume, possibly on another thread. Coroutines are lightweight (thousands multiplex over a small thread pool), cooperative (yield at suspension points), and structured (live in a scope, form parent–child hierarchies).

---

## 2. What is a suspend function? Suspending vs blocking

A `suspend` function can pause a coroutine without blocking the thread; callable only from another suspend function or a coroutine builder. Suspending releases the thread back to the pool while waiting; blocking holds it. Note: `suspend` alone doesn't move work off the current thread — a main-safe function uses `withContext(Dispatchers.IO)` internally.

---

## 3. How does suspend work under the hood (CPS / state machine)?

The compiler rewrites a suspend function using Continuation-Passing Style: it adds a hidden `Continuation` parameter and turns the body into a state machine where each suspension point is a state. On suspend it saves state in the continuation and returns `COROUTINE_SUSPENDED`; when the result is ready, `continuation.resumeWith(...)` re-enters at the saved state. State lives on the heap, not a thread stack.

---

## 4. launch vs async-await

`launch` is fire-and-forget, returns a `Job`, propagates exceptions immediately to the parent. `async` runs a concurrent computation returning a value, returns a `Deferred<T>`, and its exception surfaces on `await()`. Use `async` for parallelism (start several, then await all).

---

## 5. withContext vs async-await

`withContext(ctx)` switches context, runs the block sequentially, and returns its result directly — the idiomatic main-safety pattern for a single switch. `async {}.await()` is for real parallelism (multiple concurrent computations). Single call → `withContext`; multiple parallel → `async`. `withContext` throws at the call site; `async` defers to `await()`.

---

## 6. Dispatchers (Main, IO, Default, Unconfined)

A dispatcher decides which thread(s) a coroutine runs on. `Main` (UI thread, UI work), `IO` (large elastic pool, blocking I/O), `Default` (CPU-core-sized pool, CPU-intensive work), `Unconfined` (advanced/testing). IO and Default share threads; `Main.immediate` avoids re-dispatch. Inject dispatchers for testability.

---

## 7. CoroutineScope

Defines the lifecycle and context for coroutines launched in it — the backbone of structured concurrency (always holds a `Job`). Cancelling a scope cancels all its coroutines, preventing leaks. Tie a scope to a component's lifetime; Android provides `viewModelScope`/`lifecycleScope`.

---

## 8. CoroutineContext

An indexed set of elements configuring a coroutine: `Job` (lifecycle), `CoroutineDispatcher` (threads), `CoroutineName` (debug), `CoroutineExceptionHandler`. Combined with `+`; later elements override same-key earlier ones. A child inherits the parent's context but always gets a **new** child `Job`.

---

## 9. What is a Job?

A cancellable handle to a coroutine's lifecycle (`launch` returns `Job`; `async` returns `Deferred<T>`). States: New → Active → Completing → Completed (or Cancelling → Cancelled). Jobs form a tree — a parent waits for children, cancelling a parent cancels children. With `SupervisorJob`, a child's failure doesn't cancel siblings/parent.

---

## 10. Structured concurrency

Coroutines form a parent–child hierarchy bound to a scope, guaranteeing: no leaks (parent waits for children), cancellation propagates (cancel scope → cancel children), errors propagate (child failure cancels parent/siblings unless supervised). `coroutineScope {}`/`supervisorScope {}` create structured sub-scopes; `GlobalScope` breaks structure — avoid it.

---

## 11. Scopes used in Android: lifecycleScope, viewModelScope, GlobalScope

`viewModelScope` — bound to a ViewModel, cancelled in `onCleared()`, uses `Main.immediate`; for most logic. `lifecycleScope` — bound to a LifecycleOwner, cancelled on DESTROYED; pair with `repeatOnLifecycle` to collect flows while visible. `GlobalScope` — app-wide, never cancelled, leaks — avoid (use an app-scoped scope via DI instead).

---

## 12. job.cancel() vs scope.cancel()

`job.cancel()` cancels that one coroutine (and its children); the scope stays alive and can launch more. `scope.cancel()` cancels every coroutine in the scope and leaves the scope dead — new launches start already-cancelled. Cancel jobs to keep the scope working; cancel the scope when tearing down the component.

---

## 13. Cancellation cooperativeness

Cancellation only sets state to "cancelling" — code must cooperate. All `kotlinx.coroutines` suspend functions check cancellation and throw `CancellationException`, but a tight CPU loop that never suspends won't stop. Make loops cooperative with `isActive`/`ensureActive()`/`yield()`. Never swallow `CancellationException` (rethrow it); use `NonCancellable` for essential suspending cleanup.

---

## 14. yield() in coroutines

A suspend function that (1) checks for cancellation and (2) lets other pending coroutines on the dispatcher run before resuming. Use inside long CPU-bound loops to stay cancellable and fair. Unlike `ensureActive()` (cancellation only), it also reschedules; unlike `delay()`, no fixed delay.

---

## 15. Thread.sleep() vs delay()

`Thread.sleep` blocks the thread (nothing else runs on it), not cancellable, usable anywhere. `delay` is a suspend function that frees the thread, is cancellable, and only works in coroutines. `Thread.sleep` on a single-threaded/Main dispatcher freezes everything (ANR). Always prefer `delay()` in coroutines.

---

## 16. runBlocking

A bridge from blocking to suspending code: it blocks the current thread until the coroutine (and children) complete, running the block on the calling thread. Use in `main()`, tests (though `runTest` is preferred), or bridging suspend into legacy code. Never on the Android main thread (ANR) or inside other coroutines.

---

## 17. coroutineScope vs supervisorScope

Both create a structured sub-scope and return when children complete. `coroutineScope` — any child failure cancels the whole scope and rethrows ("all or nothing"). `supervisorScope` — children fail independently; handle each failure locally. Use `coroutineScope` for interdependent results, `supervisorScope` when partial success is acceptable.

---

## 18. Exception handling: try/catch, CoroutineExceptionHandler, SupervisorJob

(1) `try/catch` for a specific suspending call (`withContext`, `await()`). (2) `CoroutineExceptionHandler` — last-resort handler for **uncaught** `launch` exceptions, installed on the root scope (doesn't work for `async`). (3) `SupervisorJob` — isolates child failures. `launch` throws immediately; `async` defers to `await()`; `CancellationException` is normal, not an error.

---

## 19. Exception in async without await()

In a regular scope/`coroutineScope`, the failing `async` child propagates to the parent immediately via structured concurrency — it crashes/propagates even without `await()`. In a `supervisorScope`/under `SupervisorJob`, the failure is isolated and silently swallowed unless you call `await()`. Always `await()` your `async` results.

---

## 20. Running coroutines in series vs parallel

Series: suspend at each call before the next → total ≈ sum of durations. Parallel: start all with `async`, then await → total ≈ max. Pitfall: `async { a() }.await() + async { b() }.await()` is sequential because you await A before constructing B — start all `async` first, then await.

---

## 21. Combining multiple coroutine results

Use `async` per task inside `coroutineScope`, then await all. For a dynamic list, `ids.map { async { ... } }.awaitAll()`. `awaitAll()` cancels the rest and throws if any fails; for partial success wrap each in `runCatching` and use `supervisorScope`.

---

## 22. Timeouts (withTimeout / withTimeoutOrNull)

`withTimeout(ms)` cancels and throws `TimeoutCancellationException` if the block doesn't finish in time; `withTimeoutOrNull(ms)` returns `null` instead. The exception is a `CancellationException` subclass, so the work must be cancellable for the timeout to take effect. Mind cleanup in the block (use `try/finally`).

---

## 23. Debounce using coroutines

Wait for input to stop before acting. Manually: cancel the previous job on each event, then `delay(300)` before working. Idiomatic: Flow's `debounce(300)` + `distinctUntilChanged()` + `flatMapLatest { repo.search(it) }`, which also cancels stale in-flight searches.

---

## 24. suspendCoroutine vs suspendCancellableCoroutine (callback bridging)

Both convert callback APIs into suspend functions via a `Continuation`. `suspendCoroutine` is not cancellable; `suspendCancellableCoroutine` is cancellable and supports `invokeOnCancellation` for cleanup — prefer it. Resume exactly once (`resume`/`resumeWithException`); resuming twice throws.

---

## 25. Coroutines with Retrofit and Room

Retrofit supports `suspend` functions natively and dispatches network I/O itself — no need for `withContext(Dispatchers.IO)`. Room supports `suspend` DAO functions (one-shot, main-safe via its own executor) and `Flow` return types (observable queries that re-emit on table changes).

---

## 26. Testing coroutines

Use `kotlinx-coroutines-test`. `runTest` gives a `TestScope` with virtual time (skips `delay`), plus `advanceUntilIdle()`/`advanceTimeBy()`/`runCurrent()`. Use a `MainDispatcherRule` with `Dispatchers.setMain(testDispatcher)`. `StandardTestDispatcher` queues coroutines; `UnconfinedTestDispatcher` runs eagerly. Always inject dispatchers.

---

## 27. Channels (and produce / fan-out / fan-in)

A concurrency-safe queue for communicating between coroutines (suspending `send`/`receive`); unlike a cold Flow, it's a hot one-shot pipe where each element goes to exactly one consumer. Capacities: RENDEZVOUS (default), BUFFERED, CONFLATED, UNLIMITED. `produce {}` builds a `ReceiveChannel`. Fan-out = many consumers; fan-in = many producers. Prefer Flow for app streams; use Channel for point-to-point hand-off.

---

## 28. The select expression

`select {}` awaits multiple suspending sources at once and proceeds with whichever is ready first — a coroutine-aware race. Clauses: `onAwait` (Deferred), `onReceive` (channels), `onSend`, `onTimeout`. Biased toward the first ready clause. The canonical answer to "first of several results / implement a race."

---

## 29. Shared mutable state: Mutex and Semaphore

Don't use blocking `synchronized`/`ReentrantLock` (they block the thread). `Mutex.withLock` suspends instead of blocking (non-reentrant). `Semaphore.withPermit` caps concurrent coroutines in a section. Preference order: avoid sharing (confine to one coroutine), use immutable/atomic structures (`MutableStateFlow.update {}`), then `Mutex` if truly needed.
