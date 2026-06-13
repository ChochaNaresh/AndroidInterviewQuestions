# Android Performance, Memory, Threading & System Internals — Short Answers (Quick Revision)

> Condensed answers for rapid review before an interview. For full explanations and code see `android_performance_memory_interview.md`.

---

## 1. Run parallel tasks and get a callback when all complete

**Parallel task execution** means executing multiple independent asynchronous operations concurrently and waiting for all to complete before invoking a callback.

Use **coroutines with `async`/`awaitAll`** inside a `coroutineScope` — each `async` starts concurrently and `awaitAll` suspends until all finish (structured concurrency cancels siblings on failure; use `supervisorScope` to isolate). Alternatives: `combine` for Flows, or classic `CountDownLatch`/`ExecutorService.invokeAll`. Coroutines are preferred (non-blocking, cancellable, lifecycle-aware).

---



## 2. What is ANR (Application Not Responding)? How can it be prevented?

**ANR (Application Not Responding)** means a system-triggered dialog shown to the user when the main thread of an application remains blocked for too long.

The dialog shown when the main thread is blocked too long: input not handled in ~5s, service callbacks ~20s (foreground) / ~200s (background), `onReceive` 10s (foreground) / 60s (background), or an unresponsive ContentProvider. Caused by slow work on the main thread. Prevent by offloading to coroutines/WorkManager, never doing disk/network on the UI thread (StrictMode catches it), and keeping receivers trivial. Monitor via Android Vitals.

---



## 3. ThreadPool advantages

**ThreadPool** means a collection of reusable worker threads managed by a queue to execute multiple tasks efficiently without the overhead of creating new threads for each task.

Reusing a fixed set of worker threads avoids per-task thread creation cost, bounds concurrent threads (preventing resource thrashing), improves throughput via work queues, and centralizes lifecycle/shutdown. Size near core count for CPU work, larger for blocking I/O. On Android, coroutine dispatchers are backed by thread pools so you rarely manage them directly.

---



## 4. Daemon threads vs. user threads

**Daemon threads** means background support threads that are automatically terminated when all non-daemon user threads finish executing.

The JVM stays alive while any **user (non-daemon)** thread runs; it does **not** wait for **daemon** threads and kills them abruptly on exit. Set `isDaemon = true` before `start()`. Never use a daemon thread for work that must complete (e.g. flushing to disk). Coroutine pool threads are daemon by default.

---



## 5. Looper, Handler, and HandlerThread

**Looper, Handler, and HandlerThread** means the components forming Android's message-loop infrastructure to handle inter-thread communication.

**Looper** turns a thread into a loop processing a **MessageQueue** (one per thread; main thread already has one). **Handler** attaches to a Looper to post/send work onto its thread. **HandlerThread** is a Thread with a ready-made Looper for ordered serial background processing. Low-level and serial — coroutines/`Dispatchers.Main` are usually preferred.

---



## 6. Garbage Collection

**Garbage Collection (GC)** means the automatic memory management process that identifies and reclaims heap memory occupied by unreachable objects.

GC (Garbage Collection) automatically reclaims heap memory for objects no longer **reachable** from GC roots. ART uses a concurrent, generational collector (young generation collected often/cheaply). It matters because GC pauses and allocation churn cause jank. Reduce object allocations in hot paths (`onDraw`, RecyclerView binding), reuse objects, and avoid leaks (which keep objects reachable forever).

---



## 7. Memory Leak vs Out of Memory (OOM)

**Memory Leak vs Out of Memory (OOM)** means the difference between holding an unused object's reference preventing GC (leak), and the runtime error when the heap is fully exhausted (OOM).

A **memory leak** is an unneeded object still reachable from a GC root (static Context, inner-class Handler, unregistered listeners), so memory grows. **OOM** is thrown when the heap can't satisfy an allocation even after GC. Leaks are a *cause*; OOM is an *effect* (but a single huge allocation can OOM a leak-free app). Detect with LeakCanary and the Memory Profiler.

---



## 8. Runnable vs Thread

**Runnable vs Thread** means the distinction between an abstract task interface (`Runnable`) and an actual OS-backed execution thread (`Thread`).

A **Thread** is an actual OS-backed unit of execution; a **Runnable** is just a task (`run()`) with no threading logic, handed to a Thread/Executor/Handler. Prefer `Runnable` — it allows extending another class, can be reused/submitted to pools, and separates "what to run" from "how." In modern code, use coroutines/executors instead of either directly.

---



## 9. Handling bitmaps that take too much memory

**Bitmap memory management** means techniques to downsample, cache, and reuse image allocations to prevent heap exhaustion.

Bitmap memory is `width × height × bytes/pixel` (a 4000×3000 ARGB_8888 image is ~48 MB). Techniques: decode downsampled with `inSampleSize` (read bounds first), use a cheaper config (`RGB_565`/hardware bitmaps), reuse memory via `inBitmap`, cache with `LruCache` + `DiskLruCache`, recycle/release and honor `onTrimMemory`. In practice use **Coil/Glide**, which do this automatically.

---



## 10. Bitmap pool

**Bitmap pool** means a cache of reusable `Bitmap` objects that avoids garbage collection churn by recycling pixel memory buffers.

A cache of reusable `Bitmap` objects: instead of allocating a new bitmap each decode, take a suitably-sized one from the pool and decode into it via `inBitmap` (with `inMutable = true`), returning it when no longer displayed. Reduces GC churn/jank in image-heavy lists. From Android 4.4+ the reuse candidate need only be **at least as large**. Must be LRU-bounded; Glide implements one internally.

---



## 11. Jetpack DataStore Preferences

**Jetpack DataStore** means a modern, asynchronous key-value persistence library built on Kotlin coroutines and Flow to replace SharedPreferences.

The modern, async, transactional replacement for SharedPreferences using coroutines and `Flow`. Two flavors: **Preferences** (untyped key-value) and **Proto** (typed via protobuf). Advantages: no `commit()`/`apply()` foot-guns, reactive `Flow` reads, surfaces I/O errors, type safety. Not a database — for relational data use Room.

---



## 12. Persisting data in an Android app

**Data persistence** means saving application state locally across process restarts and device reboots using files, databases, or key-value stores.

Pick by shape/size: **DataStore** for small key-value settings, **Room** for structured/relational data, **file storage/MediaStore** for large files/media, **EncryptedSharedPreferences/KeyStore** for secrets, backend+cache for remote. Best practices: persist off the main thread, expose via a repository, use offline-first with Room as source of truth, never store secrets in plain prefs.

---



## 13. What is ORM? How does it work?

**ORM (Object-Relational Mapping)** means a database abstraction that maps database tables to object-oriented classes to eliminate manual SQL database queries.

**Object-Relational Mapping** maps database tables to objects so you use classes/methods instead of raw SQL and manual `Cursor` parsing. Entities map to tables, a mapping layer converts objects ⇄ rows, and DAO methods generate SQL. On Android, **Room** is the standard ORM over SQLite — validates SQL at compile time, supports Flow/LiveData, migrations, relations. Trade-off: can hide inefficient queries (N+1).

---



## 14. Preserve Activity state during screen rotation

**Activity state retention** means preserving transient user interface data across configuration changes like screen rotations using ViewModels or saved instance bundles.

Rotation destroys/recreates the activity by default. Options: `onSaveInstanceState` for small transient state; **ViewModel** (recommended) which survives config changes; **SavedStateHandle** which also survives process death; Compose `rememberSaveable`; or override `configChanges` (discouraged). Rule: ViewModel for retained data, SavedStateHandle/saved state for critical small state, never serialize large objects into the bundle.

---



## 15. Different ways to store data

**Android storage options** means the various persistence mechanisms available including DataStore, Room/SQLite, internal/external files, and secure KeyStore.

- **SharedPreferences** (legacy) / **DataStore** — small key-value.
- **Room/SQLite** — structured, queryable.
- **Internal storage** — private sandbox files (no permission).
- **External/shared storage** — MediaStore/SAF.
- **Encrypted storage** — EncryptedSharedPreferences/KeyStore for secrets.
- **Cache** and **cloud/network** (offline-first).

Choose by size, structure, sensitivity, and access needs.

---



## 16. Scoped Storage

**Scoped Storage** means a privacy-focused storage model that restricts apps to sandboxed directories and requires media APIs or pickers for shared storage access.

A privacy model (Android 10; mandatory targeting API 30+) giving each app a sandboxed view of external storage instead of broad access. App-specific dirs need no permission; shared media goes through **MediaStore** (free access to own media, user consent to modify others'); documents via **SAF/picker**. Android 13+ uses granular `READ_MEDIA_*`; `MANAGE_EXTERNAL_STORAGE` is Play-restricted.

---



## 17. How to encrypt data in Android

**Android cryptography** means securing sensitive data using hardware-backed keystores, master keys, and authenticated encryption algorithms.

Foundation is the **Android Keystore** — keys are generated/stored in hardware-backed secure storage and never leave it. Use **Jetpack Security (EncryptedSharedPreferences/EncryptedFile)** for transparent encrypted storage, **SQLCipher** for full DB encryption, and **TLS/HTTPS** (with pinning) for transport. Best practices: never hardcode keys, prefer AES-GCM, store the IV with ciphertext, tie sensitive keys to user auth.

---



## 18. SharedPreferences commit() vs apply()

**commit() vs apply()** means the choice between a synchronous, blocking write returning a status (commit), and an asynchronous, non-blocking disk write (apply) in SharedPreferences.

`commit()` writes **synchronously** on the calling thread and returns a success `Boolean` (can block/jank the UI). `apply()` updates memory immediately and writes to disk **asynchronously**, returning void (no result, coalesced). Use `apply()` by default; use `commit()` only when you need the result and are off the main thread. Better still, use DataStore.

---



## 19. Improve Android app performance

**App performance optimisation** means a systematic approach to reducing startup time, layout hierarchies, memory churn, battery drain, and network latency.

- **Startup:** trim `Application.onCreate`, lazy-init, Baseline Profiles.
- **Rendering:** stay under frame budget, flatten layouts, reduce overdraw, RecyclerView/DiffUtil; minimize Compose recomposition.
- **Threading:** never block the main thread.
- **Memory:** avoid leaks, reuse bitmaps, honor `onTrimMemory`.
- **Network/size:** cache/batch/paginate, R8 + App Bundles.

Measure first with Macrobenchmark/JankStats/Perfetto/Vitals.

---



## 20. The onTrimMemory() method

**onTrimMemory()** means a system callback that alerts application components to release non-essential resources when the device is under memory pressure.

A callback the system invokes to ask your app to **release memory** under pressure or when UI visibility changes; responding well keeps your process from being killed. Levels range from `TRIM_MEMORY_UI_HIDDEN` (release UI caches) through `RUNNING_*` to `BACKGROUND/MODERATE/COMPLETE` (release as much as possible). It supersedes the coarser `onLowMemory()`.

---



## 21. Identify and fix OutOfMemory issues

**OOM diagnosis and resolution** means locating memory leaks or giant allocations using profiles or heap dumps and replacing them with bounded collections or downsampled bitmaps.

**Identify:** OOM stack trace shows the allocation site; use the Memory Profiler (heap dump, retained sizes), LeakCanary, and allocation tracking. **Fix:** downsample oversized bitmaps (or use Coil/Glide), break leaks (weak refs, unregister), bound caches/lists (LruCache, Paging 3), stream large data, and honor `onTrimMemory`. `largeHeap="true"` is a band-aid, not a fix.

---



## 22. Find memory leaks

**Memory leak detection** means identifying retained references using LeakCanary or memory profiling tools to ensure objects are garbage collected.

**LeakCanary** (debug builds) watches destroyed Activities/Fragments/ViewModels, forces GC, and reports the shortest strong reference chain to retained objects. The **Memory Profiler** compares before/after heap dumps (e.g. multiple live `MainActivity` instances = leak). Check static fields/singletons holding Context, inner-class Handlers/listeners, unregistered receivers/observers, and non-cancelled coroutine scopes.

---



## 23. Adaptive Battery using ML

**Adaptive Battery** means a power-management system that uses machine learning to restrict background tasks of rarely-used apps into standby buckets.

Introduced in Android 9, it uses on-device machine learning to predict which apps you'll use and places rarely-used ones into restrictive **App Standby Buckets** (Active → Working set → Frequent → Rare → Restricted), deferring their jobs/alarms/network. Developer guidance: use WorkManager with constraints, use high-priority FCM for urgent delivery, and behave well to stay in a friendlier bucket.

---



## 24. Reduce battery usage

**Battery consumption optimisation** means minimising radio usage, CPU cycles, wakelocks, and GPS updates by batching tasks and using WorkManager.

Drain comes from radio, wakeups, wakelocks, CPU, GPS, and screen. **Network:** batch/coalesce, use FCM push not polling, cache, prefetch on Wi-Fi/charging. **Scheduling:** prefer WorkManager/JobScheduler over exact alarms; avoid wakelocks; respect Doze. **Location:** lowest sufficient accuracy, `FusedLocationProviderClient`, geofencing. Measure with Battery Historian/Energy Profiler/Vitals.

---



## 25. Doze and App Standby

**Doze and App Standby** means system-level power-saving features that restrict network access and defer background jobs when the device is stationary or individual apps are idle.

Both restrict background work to save battery. **Doze** (device-level: unplugged, stationary, screen off) batches deferred work into periodic maintenance windows, suspending network/jobs/alarms and ignoring wakelocks in between. **App Standby** (app-level) restricts individual idle apps the user hasn't interacted with. Design for deferred execution (WorkManager), use high-priority FCM for urgent messages.

---



## 26. What is overdraw?

**Overdraw** means a rendering inefficiency where the GPU paints the same screen pixels multiple times within a single frame.

When the GPU paints the same pixel multiple times in one frame (stacked opaque layers), wasting fill-rate and causing jank. Visualize with Developer Options → **Debug GPU Overdraw** (blue=1x … red=4x+). Reduce by removing redundant backgrounds, flattening the hierarchy (`ConstraintLayout`), not painting hidden views, and using `clipRect`/`quickReject`. Aim for mostly 1x–2x.

---



## 27. Support different resolutions and screen sizes

**Responsive layout design** means building layouts using density-independent pixels (dp), scale-independent text (sp), vector graphics, and alternative resource qualifiers.

Build adaptive layouts. Use **dp/sp** (never px) and per-density drawables or **vector drawables**. Use flexible layouts (`ConstraintLayout`, constraints/weights) and **resource qualifiers** (`sw600dp`, `-w600dp`, `-land`). Use modern adaptive APIs: **WindowSizeClass**, Compose adaptive/Material 3 components, and Jetpack WindowManager for foldables. Test across emulators, foldables, and font scales.

---



## 28. Permission protection levels

**Permission protection levels** means the classification of permissions into normal, dangerous, signature, or special categories that dictate how the OS grants access.

The protection level determines how a permission is granted: **normal** (low-risk, auto-granted at install, e.g. INTERNET), **dangerous** (sensitive data/features, requested at runtime, revocable, e.g. CAMERA/location), **signature** (auto-granted only to apps signed with the same certificate), and legacy **signatureOrSystem/system**. Protection flags refine special grants; some special permissions (e.g. `SYSTEM_ALERT_WINDOW`) require a Settings screen.

---



## 29. What is the NDK and why is it useful?

**NDK (Native Development Kit)** means a set of tools allowing developers to implement parts of an Android app in C/C++ for performance or native library reuse.

The **Native Development Kit** lets you write C/C++ compiled to `.so` libraries and call it from Kotlin/Java via **JNI**. Useful for performance-critical code (signal/image/audio processing, math), reusing existing C/C++ libraries, cross-platform code sharing, low-level access (Vulkan/OpenGL ES), and games. Trade-offs: JNI overhead, harder debugging, manual memory management, build complexity, per-ABI binaries — use only for measured wins.

---



## 30. What is RenderScript?

**RenderScript** means a deprecated framework for high-performance, data-parallel computation on the CPU/GPU, replaced by Vulkan and GPU compute.

Android's deprecated framework for high-performance data-parallel computation (image processing, blur). **Deprecated in Android 12 (API 31).** Replacements: the **RenderScript Intrinsics Replacement Toolkit** (drop-in CPU intrinsics), **Vulkan/OpenGL ES compute** for GPU work, **RenderEffect** (`setRenderEffect`, API 31+) for view blur, and **AGSL/RuntimeShader** (API 33+) for shader effects.

---



## 31. What is Android Runtime?

**ART (Android Runtime)** means the managed execution environment on Android that runs DEX bytecode using hybrid AOT and JIT compilation.

**ART** is the managed runtime that executes Android apps — it runs DEX bytecode, handles compilation (hybrid AOT/JIT/profile-guided), and manages memory/GC (concurrent, generational). It replaced Dalvik as default in Android 5.0. Each app runs in its own process with its own ART instance; ART now ships as an updatable Mainline module.

---



## 32. Dalvik, ART, JIT, and AOT

**Compilation strategies** means the methods of translating bytecode into native machine instructions, including Just-In-Time (JIT) and Ahead-Of-Time (AOT) compilation.

**JIT** compiles bytecode to native code at runtime (fast install, runtime warm-up cost); **AOT** compiles before execution (faster startup, slower/larger install). **Dalvik** (≤4.4) was JIT-only. **ART** (5.0+) originally did full AOT at install, but since Android 7.0 uses a hybrid: interpret + JIT hot methods while recording a profile, then background **profile-guided AOT** — fast installs and good performance.

---



## 33. Differences between Dalvik and ART

**Dalvik vs ART** means the comparison between the legacy Dalvik VM using JIT compilation, and the modern Android Runtime (ART) using a hybrid AOT/JIT model.

| Aspect | Dalvik | ART |
|---|---|---|
| Compilation | JIT only | Hybrid AOT + JIT + profile-guided |
| Startup/runtime | Slower | Faster |
| Install | Faster/smaller | Mitigated by profile-guided AOT |
| GC | More/longer pauses | Concurrent, generational |
| Updatability | Part of OS | Updatable Mainline module |

One-liner: Dalvik JIT-compiled every run; ART precompiles hot code for faster, smoother, more power-efficient execution with a better GC.

---



## 34. Baseline Profiles

**Baseline Profiles** means a pre-compiled list of hot classes and methods included in the APK to speed up app startup and reduce jank.

A list of hot code paths (startup, key journeys, scrolling) shipped **with your app**, so at install ART AOT-compiles exactly those paths instead of learning them via JIT — making the app fast from the first run (often 20–40% faster cold start). Generated with the **Baseline Profile Gradle plugin + Macrobenchmark** (`BaselineProfileRule`), producing `baseline-prof.txt`. Library profiles are merged.

---



## 35. What is DEX?

**DEX (Dalvik Executable)** means the optimised bytecode format that compiles Kotlin/Java files to run on the Android Runtime.

**Dalvik Executable** is Android's bytecode format. Kotlin/Java → `.class` → **D8** (and **R8** for shrinking/optimisation) → `classes.dex` in the APK/AAB; ART executes DEX, not JVM `.class`. It's register-based (fewer instructions) and packs classes with shared constant pools for memory-constrained devices. A single DEX can reference at most **65,536 (64K) methods**.

---



## 36. What is Multidex?

**Multidex** means a configuration that allows an application to build and load multiple DEX files to bypass the 65,536 method reference limit.

Because a single DEX caps at **65,536 method references**, large apps split code across multiple DEX files (`classes.dex`, `classes2.dex`, …). On API 21+ (ART) it's native — just set `multiDexEnabled = true`. On API < 21 (Dalvik) you also need the `androidx.multidex` library and `MultiDexApplication`. Better: reduce method count with R8 rather than relying on multidex.

---



## 37. Can you manually call the Garbage Collector?

**Garbage Collector invocation** means requesting a memory cleanup using `System.gc()`, which acts only as a non-binding hint to the JVM.

You can **request** GC with `System.gc()`, but it's only a hint — you cannot force it. Avoid it: ART's GC decides timing better, a manual GC can cause a stop-the-world jank pause, and it doesn't fix leaks (reachable objects won't be collected — drop references instead). Legitimate rare uses: clean heap dump before profiling or benchmark noise reduction.

---



## 38. App starts: Hot, Warm & Cold

**App startup states** means the three launch categories (Cold, Warm, Hot) defined by whether the process must be created from scratch or resumed from memory.

- **Cold (slowest):** process doesn't exist — fork process, run `Application.onCreate`, create/draw first Activity.
- **Warm (medium):** process alive but Activity recreated (skips Application creation).
- **Hot (fastest):** process and Activity exist — just brought to front.

Optimise cold start: trim `Application.onCreate`, ship Baseline Profiles, lazy-init, use SplashScreen API, defer non-critical init; measure with Macrobenchmark/Vitals.

