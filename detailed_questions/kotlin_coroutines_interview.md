# Kotlin Coroutines — Android Interview Guide

A comprehensive, code-first reference for Kotlin Coroutines interview preparation, focused on Android usage. Each question includes runnable Kotlin snippets, trade-offs, and common pitfalls. Accurate as of 2026 against `kotlinx.coroutines` (1.10.x / 1.11.x) and the AndroidX Lifecycle KTX libraries.

> Coroutines are Kotlin's answer to asynchronous and concurrent programming: lightweight, structured, and cancellable. Most Android networking, database, and background work today is written with coroutines (often combined with Flow).

---

## Table of Contents

1. [What is a coroutine?](#1-what-is-a-coroutine)
2. [What is a suspend function? Suspending vs blocking](#2-what-is-a-suspend-function-suspending-vs-blocking)
3. [How does suspend work under the hood (CPS / state machine)?](#3-how-does-suspend-work-under-the-hood-cps--state-machine)
4. [launch vs async-await](#4-launch-vs-asyncawait)
5. [withContext vs async-await](#5-withcontext-vs-asyncawait)
6. [Dispatchers (Main, IO, Default, Unconfined)](#6-dispatchers-main-io-default-unconfined)
7. [CoroutineScope](#7-coroutinescope)
8. [CoroutineContext](#8-coroutinecontext)
9. [What is a Job?](#9-what-is-a-job)
10. [Structured concurrency](#10-structured-concurrency)
11. [Scopes used in Android: lifecycleScope, viewModelScope, GlobalScope](#11-scopes-used-in-android-lifecyclescope-viewmodelscope-globalscope)
12. [job.cancel() vs scope.cancel()](#12-jobcancel-vs-scopecancel)
13. [Cancellation cooperativeness](#13-cancellation-cooperativeness)
14. [yield() in coroutines](#14-yield-in-coroutines)
15. [Thread.sleep() vs delay()](#15-threadsleep-vs-delay)
16. [runBlocking](#16-runblocking)
17. [coroutineScope vs supervisorScope](#17-coroutinescope-vs-supervisorscope)
18. [Exception handling: try/catch, CoroutineExceptionHandler, SupervisorJob](#18-exception-handling-trycatch-coroutineexceptionhandler-supervisorjob)
19. [Exception in async without await()](#19-exception-in-async-without-await)
20. [Running coroutines in series vs parallel](#20-running-coroutines-in-series-vs-parallel)
21. [Combining multiple coroutine results](#21-combining-multiple-coroutine-results)
22. [Timeouts (withTimeout / withTimeoutOrNull)](#22-timeouts-withtimeout--withtimeoutornull)
23. [Debounce using coroutines](#23-debounce-using-coroutines)
24. [suspendCoroutine vs suspendCancellableCoroutine (callback bridging)](#24-suspendcoroutine-vs-suspendcancellablecoroutine-callback-bridging)
25. [Coroutines with Retrofit and Room](#25-coroutines-with-retrofit-and-room)
26. [Testing coroutines](#26-testing-coroutines)
27. [Channels (and produce / fan-out / fan-in)](#27-channels-and-produce--fan-out--fan-in)
28. [The select expression](#28-the-select-expression)
29. [Shared mutable state: Mutex and Semaphore](#29-shared-mutable-state-mutex-and-semaphore)

---

## 1. What is a coroutine?

**Coroutine** means a lightweight, cooperative thread of execution that can be suspended and resumed without blocking the underlying thread.

It is a block of code that can **suspend** (pause) without blocking the underlying thread, and later **resume** — possibly on a different thread. Coroutines let you write asynchronous code in a sequential, top-to-bottom style, instead of nesting callbacks.

Key properties:

- **Lightweight** — you can launch hundreds of thousands of coroutines. They are not 1:1 with OS threads; many coroutines multiplex over a small thread pool.
- **Cooperative** — a coroutine yields control at suspension points (`suspend` calls). The thread is freed to do other work while it is suspended.
- **Structured** — coroutines live inside a `CoroutineScope` and form parent–child hierarchies, which manage lifetime, cancellation, and error propagation.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    launch {                       // launches a coroutine
        delay(1000)                // suspends, does NOT block the thread
        println("World")
    }
    println("Hello")               // runs first
}
// Output: Hello \n World
```

**Coroutine vs thread:** a thread is an OS-level resource (~1 MB stack each, expensive context switches). A coroutine is a cheap object on the heap; thousands can run on one thread. Demonstration:

```kotlin
runBlocking {
    repeat(100_000) {
        launch { delay(1000); print(".") }   // fine with coroutines
    }
}
// The equivalent with 100,000 threads would crash with OutOfMemoryError.
```

**📚 Reference:** https://outcomeschool.com/blog/kotlin-coroutines

---

## 2. What is a suspend function? Suspending vs blocking

**Coroutine vs Thread** means the distinction between a managed user-space execution task (coroutine) and an OS-level execution thread.

It can only be called from another `suspend` function or from inside a coroutine builder (`launch`, `async`, `runBlocking`, etc.).

```kotlin
suspend fun fetchUser(): User {
    delay(1000)            // suspension point
    return api.getUser()
}
```

**Suspending ≠ blocking:**

| | Blocking | Suspending |
|---|---|---|
| Thread | Held/occupied until done | Released back to the pool while waiting |
| Mechanism | `Thread.sleep`, blocking I/O | `delay`, suspend calls, state machine |
| Cost | Wastes a thread | Frees the thread for other coroutines |
| Cancellable | Generally no | Yes (cooperatively) |

```kotlin
// Blocking: the thread sits idle for 1s and can do nothing else.
fun blocking() { Thread.sleep(1000) }

// Suspending: the thread is freed for 1s and can run other coroutines.
suspend fun suspending() { delay(1000) }
```

The `suspend` keyword does **not** itself move work off the main thread. A `suspend` function declared on `Dispatchers.Main` still runs on the main thread unless it (or a function it calls, e.g. via `withContext`) switches dispatchers. Calling a CPU-heavy or blocking operation directly inside a `suspend` function without switching context will still freeze the UI. A well-designed suspend function should be **main-safe** — safe to call from the main thread because it internally moves blocking work off it.

```kotlin
// Main-safe suspend function
suspend fun readFile(path: String): String = withContext(Dispatchers.IO) {
    File(path).readText()   // blocking I/O, but off the main thread
}
```

**📚 Reference:** https://outcomeschool.com/blog/suspend-function-in-kotlin-coroutines · https://www.youtube.com/watch?v=V2lL_aJp17I

---

## 3. How does suspend work under the hood (CPS / state machine)?

**Continuation-Passing Style (CPS)** means the compiler transformation that converts suspending functions into state machines to handle execution resume states.

It adds a hidden `Continuation` parameter and rewrites the body into a **state machine**, where each suspension point is a state. When a coroutine suspends, it saves its state (local variables + which state it's in) in the continuation object and returns the special marker `COROUTINE_SUSPENDED`. When the awaited result is ready, `continuation.resumeWith(result)` is called, re-entering the function at the saved state.

Conceptually:

```kotlin
// You write:
suspend fun example() {
    val a = stepOne()   // suspension point
    val b = stepTwo(a)  // suspension point
    println(b)
}

// Compiler produces something like (simplified):
fun example(cont: Continuation<Unit>): Any? {
    val sm = cont as? ExampleStateMachine ?: ExampleStateMachine(cont)
    when (sm.label) {
        0 -> { sm.label = 1; val r = stepOne(sm); if (r == COROUTINE_SUSPENDED) return r; sm.a = r }
        1 -> { sm.label = 2; val r = stepTwo(sm.a, sm); if (r == COROUTINE_SUSPENDED) return r; sm.b = r }
        2 -> { println(sm.b); return Unit }
    }
}
```

This is why coroutines do not need an extra OS thread per suspension — state is stored on the heap, not on a thread's call stack. Useful to explain in interviews to show depth.

---

## 4. launch vs async-await

**launch vs async** means the choice between a fire-and-forget coroutine builder returning a `Job` (launch), and a builder returning a `Deferred` value that must be awaited (async).

- **`launch`** — fire-and-forget. Returns a `Job`. Use when you do **not** need a result. An exception is propagated immediately to the parent.
- **`async`** — concurrent computation that returns a value. Returns a `Deferred<T>` (a `Job` with a result). Call `await()` to get the result. An exception is held inside the `Deferred` and re-thrown when you call `await()`.

```kotlin
scope.launch {
    // launch: no result needed
    saveToDatabase(data)
}

scope.launch {
    // async: produce a value, retrieve with await()
    val deferred: Deferred<User> = async { fetchUser() }
    val user = deferred.await()
    render(user)
}
```

**Concurrency with async** — the canonical use is running things in parallel:

```kotlin
suspend fun loadDashboard() = coroutineScope {
    val user = async { fetchUser() }       // starts immediately
    val feed = async { fetchFeed() }       // starts immediately, runs concurrently
    Dashboard(user.await(), feed.await())  // wait for both
}
```

**Pitfalls:**
- Don't use `async { ... }.await()` immediately on one call when you don't need concurrency — that's just `withContext` with extra overhead (see Q5).
- `async` swallows exceptions until `await()`; if you never await, the exception can be lost or surface only through the parent/handler (see Q19).

**📚 Reference:** https://outcomeschool.com/blog/launch-vs-async-in-kotlin-coroutines · https://www.youtube.com/watch?v=B4AfTPpCU5o

---

## 5. withContext vs async-await

**withContext** means a suspending function that switches context and returns the result, whereas **async** means a coroutine builder that starts a concurrent task and returns a `Deferred` object.

- **`withContext(ctx) { }`** — switches the coroutine context (commonly the dispatcher), runs the block, and **returns its result directly**. It is **sequential** — the caller suspends until the block finishes. Use it to move a single piece of work to another dispatcher (the idiomatic main-safety pattern).
- **`async { }.await()`** — designed for **concurrency**: start multiple computations that run in parallel, then await them.

```kotlin
// Use withContext for a single context switch (most common):
suspend fun getUser(): User = withContext(Dispatchers.IO) {
    api.getUser()
}

// Use async/await when you need real parallelism:
suspend fun load() = coroutineScope {
    val a = async(Dispatchers.IO) { api.getA() }
    val b = async(Dispatchers.IO) { api.getB() }
    combine(a.await(), b.await())
}
```

Rule of thumb: **single call → `withContext`; multiple parallel calls → `async`.** Using `async { }.await()` for a single sequential call works but creates an unnecessary `Deferred` and child job, so `withContext` is preferred there.

**Exception behaviour differs too:** `withContext` throws directly at the call site (catch with normal `try/catch`). `async` defers the throw to `await()`.

**📚 Reference:** https://outcomeschool.com/blog/kotlin-withcontext-vs-async-await

---

## 6. Dispatchers (Main, IO, Default, Unconfined)

**Coroutine dispatcher** means an execution coordinator that determines which thread or thread pool runs the coroutine.

It is part of the `CoroutineContext`.

| Dispatcher | Backing | Use for |
|---|---|---|
| `Dispatchers.Main` | Android main/UI thread | UI updates, lightweight work, calling suspend fns |
| `Dispatchers.IO` | Shared elastic pool (default 64 threads, scalable) | Blocking I/O: network, disk, database |
| `Dispatchers.Default` | Pool sized to CPU cores | CPU-intensive work: parsing, sorting, JSON, image processing |
| `Dispatchers.Unconfined` | Caller thread, then wherever it resumes | Advanced/testing; not for app code |

```kotlin
viewModelScope.launch {                 // starts on Main
    val data = withContext(Dispatchers.IO) { repo.load() }   // off-load I/O
    val parsed = withContext(Dispatchers.Default) { parse(data) }  // CPU work
    textView.text = parsed.title        // back on Main automatically
}
```

Key facts:
- `Dispatchers.IO` and `Dispatchers.Default` **share threads**; switching between them with `withContext` may reuse the same thread without an actual thread switch (optimised).
- `Dispatchers.Main.immediate` runs synchronously if already on the main thread, avoiding an unnecessary re-dispatch.
- Inject dispatchers (don't hard-code) so you can substitute test dispatchers — pass a `CoroutineDispatcher` into your repositories/use cases.

```kotlin
class Repo(private val io: CoroutineDispatcher = Dispatchers.IO) {
    suspend fun load() = withContext(io) { /* ... */ }
}
```

**📚 Reference:** https://outcomeschool.com/blog/dispatchers-in-kotlin-coroutines

---

## 7. CoroutineScope

**Coroutine dispatchers (Main, IO, Default, Unconfined)** means the standard dispatchers used for UI operations, disk/network I/O, CPU-bound tasks, and thread-agnostic execution respectively.

Every coroutine belongs to a scope; this is the backbone of structured concurrency. A scope is essentially a holder for a `CoroutineContext` (which always includes a `Job`).

```kotlin
class MyController {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)

    fun start() {
        scope.launch { doWork() }
    }

    fun clear() {
        scope.cancel()   // cancels every coroutine started in this scope
    }
}
```

When you cancel a scope, all coroutines launched in it are cancelled. This prevents leaks: you tie a scope to a component's lifetime (Activity, ViewModel, custom object) and cancel it when the component is destroyed. Android provides ready-made scopes (`viewModelScope`, `lifecycleScope`) so you rarely create your own (see Q11).

**📚 Reference:** https://outcomeschool.com/blog/kotlin-coroutines

---

## 8. CoroutineContext

**Suspending function** means a function marked with `suspend` that can pause execution without blocking the host thread and resume later.

The main elements are:

- **`Job`** — controls lifecycle/cancellation.
- **`CoroutineDispatcher`** — the thread/threads.
- **`CoroutineName`** — name for debugging.
- **`CoroutineExceptionHandler`** — last-resort uncaught-exception handler.

Elements are combined with the `+` operator; later elements override earlier ones of the same key:

```kotlin
val context = Dispatchers.IO + CoroutineName("loader") + SupervisorJob()

scope.launch(Dispatchers.Default + CoroutineName("worker")) {
    println(coroutineContext[CoroutineName])   // CoroutineName(worker)
}
```

**Context inheritance:** a child coroutine inherits its parent's context, with two rules — the elements you explicitly pass to the builder override inherited ones, and the child always gets a **new `Job`** that is a child of the parent's `Job` (so it never literally reuses the parent's Job instance). This new child Job is what links cancellation and exception propagation between parent and child.

**📚 Reference:** https://outcomeschool.com/blog/coroutinecontext-in-kotlin

---

## 9. What is a Job?

**CoroutineContext** means a persistent map of key-value elements that configure a coroutine's behaviour, including its Job, Dispatcher, and ExceptionHandler.

`launch` returns a `Job`; `async` returns a `Deferred<T>`, which is a `Job` that also carries a result.

A Job has states: **New → Active → Completing → Completed**, with **Cancelling → Cancelled** on cancellation.

Common operations:

```kotlin
val job = scope.launch { doWork() }

job.start()          // start if created lazily
job.cancel()         // request cancellation
job.cancelAndJoin()  // cancel and wait until it finishes
job.join()           // suspend until the job completes
job.isActive         // true while running
job.isCancelled
job.isCompleted
job.invokeOnCompletion { cause -> /* cleanup */ }
```

**Parent–child relationship:** jobs form a tree. A parent does not complete until all children complete. Cancelling a parent cancels all children. By default, a child's failure cancels the parent and its siblings (unless a `SupervisorJob` is used — see Q17/Q18).

**`Job()` vs `SupervisorJob()`:** with a regular `Job`, a failing child cancels the whole hierarchy. With a `SupervisorJob`, child failures are isolated — one child failing does not cancel its siblings or the parent.

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_job-in-coroutines-activity-7372874272559656961-LIaI

---

## 10. Structured concurrency

**CoroutineScope** means an interface that provides and manages the lifecycle boundaries of coroutines to enforce structured concurrency.

1. **No leaks** — a parent waits for all children to finish before it completes; nothing is "forgotten."
2. **Cancellation propagates** — cancelling a scope/parent cancels all children.
3. **Errors propagate** — an uncaught failure in a child cancels its parent and siblings (unless supervised).

This is what makes coroutines safe compared to raw threads or callbacks. `coroutineScope { }` and `supervisorScope { }` create structured sub-scopes:

```kotlin
suspend fun loadAll() = coroutineScope {
    val a = async { loadA() }
    val b = async { loadB() }
    a.await() + b.await()
}   // loadAll() does not return until BOTH a and b complete or fail.
    // If loadA() throws, loadB() is cancelled and the exception propagates.
```

The opposite — launching into `GlobalScope` — breaks structure: such coroutines outlive your component, aren't cancelled automatically, and leak. Avoid it.

---

## 11. Scopes used in Android: lifecycleScope, viewModelScope, GlobalScope

**Android coroutine scopes** means lifecycle-aware scopes including `viewModelScope` and `lifecycleScope` that automatically cancel active jobs when components are destroyed.

**`viewModelScope`** — bound to a `ViewModel`; cancelled automatically when `onCleared()` is called. Uses `Dispatchers.Main.immediate`. Use for most business logic.

```kotlin
class MyViewModel : ViewModel() {
    fun load() {
        viewModelScope.launch {
            val data = repo.fetch()    // cancelled if ViewModel is cleared
            _state.value = data
        }
    }
}
```

**`lifecycleScope`** — bound to a `LifecycleOwner` (Activity/Fragment); cancelled when the lifecycle is `DESTROYED`. Often combined with `repeatOnLifecycle` to collect flows only while the UI is visible (the modern replacement for `launchWhenStarted`, which is deprecated).

```kotlin
class MyFragment : Fragment() {
    override fun onViewCreated(view: View, s: Bundle?) {
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { render(it) }   // only while STARTED
            }
        }
    }
}
```

**`GlobalScope`** — application-wide scope that is **never** cancelled. Coroutines launched here are not tied to any lifecycle and easily leak. It is marked `@DelicateCoroutinesApi`. **Avoid in app code.** If you genuinely need work to outlive a screen (e.g., a fire-and-forget upload), create an application-scoped `CoroutineScope` (often via Hilt) instead.

```kotlin
// Prefer this over GlobalScope for app-lifetime work:
@Singleton
class AppScope @Inject constructor() :
    CoroutineScope by CoroutineScope(SupervisorJob() + Dispatchers.Default)
```

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_softwareengineer-androiddev-android-activity-7278639980053217280-8eZO

---

## 12. job.cancel() vs scope.cancel()

**job.cancel() vs scope.cancel()** means the choice between cancelling a single coroutine and its children (job.cancel), and cancelling all coroutines started in a scope permanently (scope.cancel).

- **`job.cancel()`** — cancels **that one coroutine** (the Job returned by `launch`/`async`) and its children. Other coroutines in the same scope keep running.
- **`scope.cancel()`** — cancels the scope's `Job`, which cancels **all** coroutines launched in that scope.

```kotlin
val scope = CoroutineScope(Job() + Dispatchers.Default)
val job1 = scope.launch { taskA() }
val job2 = scope.launch { taskB() }

job1.cancel()    // cancels only taskA; taskB still runs
scope.cancel()   // cancels everything in the scope (taskB too)
```

**Important difference for reuse:** after `scope.cancel()`, the scope's `Job` is in a cancelled state and the scope is **dead** — launching new coroutines in it does nothing (they start already-cancelled). After `job.cancel()`, the scope itself is still alive and can launch new coroutines. So cancel individual jobs when you want the scope to keep working; cancel the scope when you're tearing down the whole component.

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineer-tech-activity-7321028335784894465-HCrl

---

## 13. Cancellation cooperativeness

**Cooperative cancellation** means checking active cancellation flags (like `isActive` or `ensureActive()`) during long-running tasks to respond to cancellation.

All suspending functions in `kotlinx.coroutines` (`delay`, `withContext`, `yield`, etc.) are cancellable and throw `CancellationException` at their suspension points. But a tight CPU loop that never suspends will **not** stop on its own.

```kotlin
// NOT cancellable — ignores cancellation, runs to completion:
val job = scope.launch(Dispatchers.Default) {
    var i = 0
    while (i < 1_000_000) { i++; heavyWork() }   // never checks cancellation
}
job.cancel()   // has no effect until the loop finishes
```

Make CPU loops cooperative with `isActive`, `ensureActive()`, or `yield()`:

```kotlin
scope.launch(Dispatchers.Default) {
    var i = 0
    while (isActive && i < 1_000_000) {   // or: ensureActive() inside the loop
        i++; heavyWork()
    }
}
```

**Cleanup on cancellation:** `CancellationException` is special — it signals normal cancellation and is **not** treated as an error. Don't swallow it. If you `try/catch` it, rethrow it:

```kotlin
try {
    work()
} catch (e: CancellationException) {
    throw e               // never swallow cancellation
} catch (e: Exception) {
    handle(e)
} finally {
    // finally runs, but suspending calls here are already cancelled.
    withContext(NonCancellable) { closeResources() }   // to suspend during cleanup
}
```

Use `NonCancellable` only for short, essential cleanup — overusing it defeats cancellation.

---

## 14. yield() in coroutines

**yield()** means a suspending function that yields execution to allow other pending tasks to run on the same dispatcher while checking cancellation status.

It does two things: (1) checks for cancellation (throws `CancellationException` if cancelled), and (2) if there are other coroutines waiting on the same dispatcher, lets them run before resuming.

```kotlin
scope.launch(Dispatchers.Default) {
    val results = mutableListOf<Int>()
    for (item in bigList) {
        results += process(item)
        yield()   // stays cooperative: allows cancellation + fair scheduling
    }
}
```

Use it inside long CPU-bound loops so the coroutine stays cancellable and doesn't monopolize the thread. Compared to `ensureActive()` (which only checks cancellation), `yield()` also offers a suspension/rescheduling point. It does not introduce a fixed delay like `delay()`.

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_outcomeschool-softwareengineer-tech-activity-7298934394113638400-ZY-y

---

## 15. Thread.sleep() vs delay()

**Thread.sleep() vs delay()** means the difference between blocking the current OS thread (sleep), and suspending the coroutine execution without blocking the thread (delay).

| | `Thread.sleep(ms)` | `delay(ms)` |
|---|---|---|
| Type | Regular blocking call | `suspend` function |
| Effect | **Blocks** the thread; nothing else runs on it | **Suspends** the coroutine; thread is freed |
| Cancellation | Not cancellable | Cancellable (cancellation check) |
| Where usable | Anywhere | Only in coroutines/suspend functions |

```kotlin
runBlocking {
    launch { delay(1000); println("A") }       // suspends; thread free
    launch { Thread.sleep(1000); println("B") } // blocks the only thread!
}
```

In a single-threaded dispatcher, `Thread.sleep` inside one coroutine blocks the entire dispatcher, starving every other coroutine on it (and on `Dispatchers.Main` it freezes the UI / triggers ANR). Always prefer `delay()` inside coroutines. Use `Thread.sleep` only when you are intentionally outside coroutine context (rare) or simulating blocking in a learning example.

**📚 Reference:** https://x.com/amitiitbhu/status/1812806101944946962

---

## 16. runBlocking

**runBlocking** means a coroutine builder that blocks the current thread until its code block and all children finish executing.

It **blocks** the current thread until the coroutine inside it (and all its children) complete. It runs the block on the calling thread.

```kotlin
fun main() = runBlocking {     // blocks main thread until done
    val data = fetchData()     // can call suspend functions here
    println(data)
}
```

When to use:
- **`main()` functions** and small scripts.
- **Tests** (though `runTest` is preferred for coroutine tests — see Q26).
- Bridging a suspend API into legacy blocking/synchronous code.

When **not** to use:
- **Never on the Android main thread** — it blocks the UI thread and causes ANRs.
- Not inside other coroutines — that defeats the purpose; use `coroutineScope`/`withContext` instead.

`runBlocking` differs from other builders: it's the only one that blocks a thread, and it's a top-level function (not an extension on `CoroutineScope`).

---

## 17. coroutineScope vs supervisorScope

**coroutineScope vs supervisorScope** means the difference between a scope that propagates any child failure to cancel all siblings (coroutineScope), and a scope that isolates child failures (supervisorScope).

The difference is **how child failures are handled**.

- **`coroutineScope`** — failure of **any** child cancels the whole scope (all other children) and rethrows the exception. "All or nothing."
- **`supervisorScope`** — children fail **independently**; one child's failure does not cancel its siblings or the scope. Each child's failure must be handled on its own.

```kotlin
// coroutineScope: if taskB fails, taskA is cancelled and loadAll() throws.
suspend fun loadAll() = coroutineScope {
    val a = async { taskA() }
    val b = async { taskB() }   // if this throws...
    a.await() to b.await()      // ...a is cancelled, exception propagates
}

// supervisorScope: childB failing leaves childA running.
suspend fun loadIndependently() = supervisorScope {
    launch { taskA() }
    launch {
        try { taskB() } catch (e: Exception) { log(e) }  // handle locally
    }
}
```

Use `coroutineScope` when results are interdependent (if one fails, the rest are useless). Use `supervisorScope` when tasks are independent and partial success is acceptable (e.g., loading several widgets where one failure shouldn't blank the whole screen).

Note: with `supervisorScope` + `async`, exceptions are still delivered through `await()`; with `supervisorScope` + `launch`, an uncaught child exception is routed to the `CoroutineExceptionHandler`.

**📚 Reference:** https://outcomeschool.com/blog/coroutinescope-vs-supervisorscope

---

## 18. Exception handling: try/catch, CoroutineExceptionHandler, SupervisorJob

**Coroutine exception handling** means managing errors using try-catch blocks, CoroutineExceptionHandler, or SupervisorJob to prevent app crashes.

**(1) try/catch** — for exceptions at a specific suspending call. Works for `withContext` and `await()` results.

```kotlin
viewModelScope.launch {
    try {
        val user = withContext(Dispatchers.IO) { api.getUser() }
        _state.value = Success(user)
    } catch (e: IOException) {
        _state.value = Error(e)
    }
}
```

**(2) CoroutineExceptionHandler** — a context element that catches **uncaught** exceptions from `launch` coroutines (a last-resort/global handler). It does **not** work for `async` (those surface via `await()`), and it must be installed on the **root** coroutine/scope, not on a child launched inside another.

```kotlin
val handler = CoroutineExceptionHandler { _, throwable ->
    Log.e("App", "Uncaught: $throwable")
}
val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main + handler)
scope.launch { throw RuntimeException("boom") }   // caught by handler
```

**(3) SupervisorJob** — changes propagation so one child's failure doesn't cancel siblings/parent. Pair it with a `CoroutineExceptionHandler` to log per-child failures without tearing down the scope.

```kotlin
val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main + handler)
scope.launch { taskA() }   // if this fails...
scope.launch { taskB() }   // ...this keeps running
```

**Propagation summary:**
- Regular `Job`: child failure → cancels parent and siblings.
- `SupervisorJob`/`supervisorScope`: child failure isolated.
- `launch`: exception thrown immediately (to parent / handler).
- `async`: exception deferred until `await()`.
- `CancellationException` is normal cancellation, not an error — never treat/log it as a failure.

---

## 19. Exception in async without await()

**Exception propagation in async** means that uncaught exceptions inside `async` are immediately propagated to the parent coroutine, regardless of whether `await()` is called.

The exception is stored inside the resulting `Deferred`. Whether it surfaces depends on the scope's Job:

- In a **regular scope/`coroutineScope`** (non-supervisor), `async` is still a child of the parent Job. When the child fails, structured concurrency **propagates the failure to the parent immediately**, cancelling the parent and siblings — even if you never call `await()`. The exception is not silently lost; it crashes/propagates through the parent.

```kotlin
suspend fun demo() = coroutineScope {
    async { throw RuntimeException("boom") }   // no await()
    delay(1000)
    // The parent (coroutineScope) is cancelled by the child's failure;
    // demo() throws RuntimeException even though await() was never called.
}
```

- In a **`supervisorScope`** (or under a `SupervisorJob`), the `async` failure is **isolated and not propagated to the parent**. If you never call `await()`, the exception is effectively **swallowed** — nobody observes it. This is a real footgun.

```kotlin
suspend fun demo() = supervisorScope {
    val d = async { throw RuntimeException("boom") }
    delay(1000)
    // No propagation, no await -> exception is lost silently.
    // It only surfaces if you call d.await().
}
```

**Takeaway:** with a top-level/`coroutineScope` `async`, an unawaited exception still brings down the parent. With a supervised `async`, you must call `await()` (or otherwise inspect the `Deferred`) to observe the exception. Always `await()` your `async` results.

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_kotlin-androiddev-activity-7353980436689145856-CTeE

---

## 20. Running coroutines in series vs parallel

**Sequential vs parallel execution** means the difference between suspending at each task call (sequential), and launching multiple concurrent tasks using `async` to run them together (parallel).

Total time ≈ sum of durations.

```kotlin
suspend fun series() = coroutineScope {
    val a = fetchA()   // wait for A...
    val b = fetchB()   // ...then start B
    a + b
}   // total ≈ timeA + timeB
```

**Parallel (concurrent):** start both, then await. Total time ≈ max of durations.

```kotlin
suspend fun parallel() = coroutineScope {
    val a = async { fetchA() }   // starts now
    val b = async { fetchB() }   // starts now, concurrent with A
    a.await() + b.await()        // wait for both
}   // total ≈ max(timeA, timeB)
```

**Pitfall — accidental serialisation:** `async { fetchA() }.await() + async { fetchB() }.await()` is sequential because you `await()` A before constructing B's `async`. Start all `async` blocks first, then await.

```kotlin
// WRONG (sequential despite async):
val a = async { fetchA() }.await()
val b = async { fetchB() }.await()

// RIGHT (parallel):
val a = async { fetchA() }
val b = async { fetchB() }
a.await() + b.await()
```

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_outcomeschool-softwareengineer-tech-activity-7297944609614139392-h9NZ/

---

## 21. Combining multiple coroutine results

**Result aggregation** means combining outcomes from multiple parallel coroutines using `awaitAll` or `zip` inside structured scopes.

```kotlin
data class Dashboard(val user: User, val posts: List<Post>, val notifs: Int)

suspend fun loadDashboard(): Dashboard = coroutineScope {
    val user   = async { api.getUser() }
    val posts  = async { api.getPosts() }
    val notifs = async { api.getNotifications() }
    Dashboard(user.await(), posts.await(), notifs.await())   // all in parallel
}
```

For a **dynamic list** of tasks, map to `async` and use `awaitAll()`:

```kotlin
suspend fun loadAll(ids: List<Int>): List<Item> = coroutineScope {
    ids.map { id -> async { api.getItem(id) } }
       .awaitAll()
}
```

`awaitAll()` waits for every deferred; if any fails, it cancels the rest and throws. If you want partial success, wrap each call in a `runCatching`/try-catch or use a `supervisorScope`.

```kotlin
suspend fun loadAllTolerant(ids: List<Int>): List<Result<Item>> = supervisorScope {
    ids.map { id -> async { runCatching { api.getItem(id) } } }.awaitAll()
}
```

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_softwareengineer-androiddev-android-activity-7267827335393873920-v6t3 · https://outcomeschool.com/blog/parallel-multiple-network-calls-using-kotlin-coroutines

---

## 22. Timeouts (withTimeout / withTimeoutOrNull)

**withTimeout** means a suspending function that cancels the execution block and throws a `TimeoutCancellationException` if the execution exceeds the specified limit.

- **`withTimeout(ms)`** — runs a block; if it doesn't finish in time, cancels it and throws `TimeoutCancellationException`.
- **`withTimeoutOrNull(ms)`** — same, but returns `null` instead of throwing on timeout.

```kotlin
// Throws on timeout:
suspend fun load(): Data = withTimeout(5000) {
    api.fetch()
}

// Returns null on timeout:
suspend fun loadOrNull(): Data? = withTimeoutOrNull(5000) {
    api.fetch()
}
```

`TimeoutCancellationException` is a subclass of `CancellationException`, so it cancels the block cooperatively (the work must be cancellable for the timeout to take effect). Be careful with cleanup inside the block — resources opened there are cancelled on timeout; use `try/finally` (with `NonCancellable` if you must suspend during cleanup).

```kotlin
val result = try {
    withTimeout(3000) { downloadFile() }
} catch (e: TimeoutCancellationException) {
    showRetry()
    null
}
```

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_softwareengineer-androiddev-android-activity-7265684677770846208-Ug0Y

---

## 23. Debounce using coroutines

**Debouncing** means waiting for a specified quiet period of inactivity before executing a task, commonly used in search queries.

With coroutines, cancel the previous pending job whenever a new event arrives and `delay` before doing the work.

```kotlin
class SearchController(private val scope: CoroutineScope) {
    private var debounceJob: Job? = null

    fun onQueryChanged(query: String) {
        debounceJob?.cancel()            // cancel the previous pending search
        debounceJob = scope.launch {
            delay(300)                   // quiet period
            search(query)                // runs only if no new input for 300ms
        }
    }
}
```

The idiomatic modern approach uses **Flow's `debounce` operator** (no manual job juggling):

```kotlin
val queryFlow = MutableStateFlow("")

scope.launch {
    queryFlow
        .debounce(300)
        .distinctUntilChanged()
        .flatMapLatest { q -> repo.search(q) }   // cancels stale searches
        .collect { render(it) }
}
```

`flatMapLatest` ensures an in-flight search is cancelled when a newer query arrives — combining debounce with cancellation of outdated work.

**📚 Reference:** https://outcomeschool.substack.com/p/how-to-implement-debounce-using-coroutines

---

## 24. suspendCoroutine vs suspendCancellableCoroutine (callback bridging)

**suspendCoroutine vs suspendCancellableCoroutine** means the choice between wrapping callbacks into suspend functions, and using the cancellable version to handle job cancellation.

They give you a `Continuation` you resume when the callback fires.

- **`suspendCoroutine`** — basic bridge; the resulting suspend call is **not cancellable**.
- **`suspendCancellableCoroutine`** — cancellable; lets you register `invokeOnCancellation` to release resources / cancel the underlying operation when the coroutine is cancelled. **Prefer this** for any real async work.

```kotlin
// Bridging a callback API with cancellation support:
suspend fun fetchLocation(): Location = suspendCancellableCoroutine { cont ->
    val callback = object : LocationCallback {
        override fun onResult(loc: Location) { cont.resume(loc) }
        override fun onError(e: Exception)   { cont.resumeWithException(e) }
    }
    locationClient.request(callback)

    cont.invokeOnCancellation {
        locationClient.removeCallback(callback)   // cleanup on cancellation
    }
}
```

Rules:
- Resume the continuation **exactly once** (`resume`, `resumeWithException`, or `cont.resume(value) { }`). Resuming twice throws `IllegalStateException`.
- With `suspendCancellableCoroutine`, check `cont.isActive` before resuming if the callback may fire after cancellation.
- This is exactly how libraries like Retrofit's `await()` and Play Services' `.await()` are implemented.

**📚 Reference:** https://outcomeschool.com/blog/callback-to-coroutines-in-kotlin

---

## 25. Coroutines with Retrofit and Room

**Jetpack coroutine integration** means using native `suspend` keyword declarations in Room DAOs and Retrofit services to offload database and network operations.

```kotlin
interface ApiService {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Int): User          // returns body

    @GET("users/{id}")
    suspend fun getUserResponse(@Path("id") id: Int): Response<User>  // full response
}

// Usage — Retrofit runs the call off the main thread; just catch errors:
suspend fun loadUser(id: Int): Result<User> = try {
    Result.success(api.getUser(id))
} catch (e: Exception) {
    Result.failure(e)
}
```

You do **not** need to wrap a suspend Retrofit call in `withContext(Dispatchers.IO)` — Retrofit dispatches the network I/O itself.

**Room** supports `suspend` DAO functions (one-shot) and `Flow` return types (observable queries):

```kotlin
@Dao
interface UserDao {
    @Insert
    suspend fun insert(user: User)                 // one-shot, main-safe

    @Query("SELECT * FROM users WHERE id = :id")
    suspend fun getUser(id: Int): User?

    @Query("SELECT * FROM users")
    fun observeUsers(): Flow<List<User>>           // emits on every change
}
```

Room runs `suspend` DAO calls on its own background executor, so they are main-safe. `Flow` queries emit a new list automatically whenever the table changes — collect them from `viewModelScope` / `lifecycleScope`.

**📚 Reference:** https://outcomeschool.com/blog/retrofit-with-kotlin-coroutines · https://outcomeschool.com/blog/room-database-with-kotlin-coroutines

---

## 26. Testing coroutines

**Coroutine testing** means controlling asynchronous execution using `runTest` and `TestDispatcher` to skip delays and ensure deterministic test assertions.

The entry point is **`runTest`**, which provides a `TestScope` with a virtual-time `TestDispatcher` — `delay` is skipped (auto-advanced), so tests are fast and deterministic.

```kotlin
@Test
fun loadsUser() = runTest {
    val vm = MyViewModel(repo)
    vm.load()
    advanceUntilIdle()                       // run all pending coroutines
    assertEquals(expected, vm.state.value)
}
```

**Inject a test dispatcher** for the main thread (Android's `Dispatchers.Main` isn't available in unit tests):

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class MainDispatcherRule(
    val dispatcher: TestDispatcher = StandardTestDispatcher()
) : TestWatcher() {
    override fun starting(d: Description) = Dispatchers.setMain(dispatcher)
    override fun finished(d: Description)  = Dispatchers.resetMain()
}

class MyViewModelTest {
    @get:Rule val mainRule = MainDispatcherRule()

    @Test fun test() = runTest { /* ... */ }
}
```

- **`StandardTestDispatcher`** — coroutines are queued; you control execution with `advanceUntilIdle()` / `advanceTimeBy()` / `runCurrent()`.
- **`UnconfinedTestDispatcher`** — eager execution (starts coroutines immediately), handy for simple tests.
- Always **inject dispatchers** into your classes (Q6) so tests can substitute the test dispatcher.

**📚 Reference:** https://outcomeschool.com/blog/unit-testing-viewmodel-with-kotlin-coroutines-and-livedata

---

## 27. Channels (and produce / fan-out / fan-in)

**Flow vs Coroutines** means the distinction between a stream producing multiple sequential values asynchronously (Flow), and a single asynchronous execution task (Coroutine).

Where a `Flow` is a *cold stream* (each collector re-runs the producer), a `Channel` is a *hot, one-shot pipe*: each element is received by exactly **one** consumer.

```kotlin
val channel = Channel<Int>()
launch {                        // producer
    for (x in 1..5) channel.send(x * x)
    channel.close()             // signals "no more elements"
}
for (y in channel) println(y)   // consumer; loop ends when channel is closed
```

**Buffering / overflow strategies** via the capacity argument:

| Capacity | Behaviour |
| --- | --- |
| `RENDEZVOUS` (0, default) | `send` suspends until a `receive` is ready |
| `BUFFERED` / `n` | buffer up to n; `send` suspends when full |
| `CONFLATED` | keeps only the latest, never suspends |
| `UNLIMITED` | never suspends (unbounded buffer) |

**`produce {}`** is the idiomatic builder — it returns a `ReceiveChannel` and auto-closes when the block ends:

```kotlin
fun CoroutineScope.numbers() = produce {
    for (n in 1..3) send(n)
}
```

- **Fan-out**: multiple coroutines `receive` from one channel → distributes work across consumers.
- **Fan-in**: multiple producers `send` into one channel → merges results.

**When to prefer Flow over Channel:** for most app-level streams (UI state, DB/network results) use `Flow`/`StateFlow`/`SharedFlow` — they're cold, lifecycle-friendly, and composable. Reach for raw `Channel` only for genuine *point-to-point hand-off* between coroutines (e.g. a work queue). Note `callbackFlow`/`channelFlow` use a channel internally to bridge callbacks into a flow.

**Why it matters:** Channels test whether you understand the *communicating* side of coroutines (vs the *streaming* side that Flow covers), and the cold-vs-hot distinction is a frequent follow-up.

**📚 Reference:** https://kotlinlang.org/docs/channels.html

---

## 28. The select expression

**Flow vs RxJava** means the comparison between Kotlin's native, coroutine-powered reactive streams (Flow), and the external, complex Java-based reactive framework (RxJava).

It's the building block for racing operations, timeouts against multiple channels, and "first response wins" patterns.

```kotlin
suspend fun fastest(a: Deferred<String>, b: Deferred<String>): String =
    select {
        a.onAwait { it }     // whichever Deferred completes first
        b.onAwait { it }
    }

// Race a value against a timeout channel, or merge two channels:
suspend fun receiveFromEither(c1: ReceiveChannel<Int>, c2: ReceiveChannel<Int>): Int =
    select {
        c1.onReceive { it }
        c2.onReceive { it }
    }
```

Clauses include `onAwait` (Deferred), `onReceive`/`onReceiveCatching` (channels), `onSend` (channels), and `onTimeout`. `select` is biased toward the first clause when several are ready. As of recent `kotlinx.coroutines` the API is stable (was `@ExperimentalCoroutinesApi`).

**Why it matters:** It's the canonical answer to "how do you take the first of several async results / implement a race?" without busy-waiting — and shows familiarity beyond the everyday `launch`/`async`.

**📚 Reference:** https://kotlinlang.org/docs/select-expression.html

---

## 29. Shared mutable state: Mutex and Semaphore

**Mutex** means a mutual exclusion lock that suspends coroutines instead of blocking threads to protect shared mutable state.

`kotlinx.coroutines` provides **suspending** primitives:

**`Mutex`** — mutual exclusion; `withLock` suspends (doesn't block) if the lock is held. Unlike a reentrant lock, `Mutex` is **non-reentrant** (locking twice from the same coroutine deadlocks).

```kotlin
val mutex = Mutex()
var counter = 0

suspend fun increment() = mutex.withLock { counter++ }   // safe across coroutines
```

**`Semaphore`** — limits the number of coroutines in a section concurrently (e.g. cap concurrent network calls):

```kotlin
val limit = Semaphore(permits = 4)
suspend fun fetch(url: String) = limit.withPermit { httpClient.get(url) }
```

Ordered by preference for protecting state:
1. **Don't share** — confine mutation to a single coroutine/dispatcher (e.g. `withContext(singleThread)` or an actor-like `Channel`).
2. Use **thread-safe / immutable** structures (`AtomicInteger`, immutable `data class` + `update {}` on `MutableStateFlow`).
3. Use **`Mutex`** only when fine-grained locking is genuinely needed.

`MutableStateFlow.update { }` is the everyday Android answer — it does a lock-free compare-and-set, so you rarely need an explicit `Mutex` for UI state.

**Why it matters:** Interviewers probe whether you know that classic JVM locks block threads and break structured concurrency, and that coroutines have suspending equivalents — plus the better habit of *avoiding* shared state altogether.

**📚 Reference:** https://kotlinlang.org/docs/shared-mutable-state-and-concurrency.html
