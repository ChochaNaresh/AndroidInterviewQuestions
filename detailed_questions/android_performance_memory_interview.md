# Android Performance, Memory, Threading, Storage & System Internals — Interview Guide

A comprehensive question-and-answer guide for Android interviews covering long-running operations and threading, multimedia memory handling, data persistence, memory and battery optimization, screen-size support, permissions, native programming, and Android system internals. Answers include Kotlin snippets, concrete techniques, and trade-offs, and are accurate as of 2026.

---

## Table of Contents

### Long-running Operations
1. [Run parallel tasks and get a callback when all complete](#1-run-parallel-tasks-and-get-a-callback-when-all-complete)
2. [What is ANR? How can it be prevented?](#2-what-is-anr-how-can-it-be-prevented)
3. [ThreadPool advantages](#3-threadpool-advantages)
4. [Daemon threads vs. user threads](#4-daemon-threads-vs-user-threads)
5. [Looper, Handler, and HandlerThread](#5-looper-handler-and-handlerthread)
6. [Garbage Collection](#6-garbage-collection)
7. [Memory Leak vs Out of Memory (OOM)](#7-memory-leak-vs-out-of-memory-oom)
8. [Runnable vs Thread](#8-runnable-vs-thread)

### Working With Multimedia
9. [Handling bitmaps that take too much memory](#9-handling-bitmaps-that-take-too-much-memory)
10. [Bitmap pool](#10-bitmap-pool)

### Data Saving
11. [Jetpack DataStore Preferences](#11-jetpack-datastore-preferences)
12. [Persisting data in an Android app](#12-persisting-data-in-an-android-app)
13. [What is ORM? How does it work?](#13-what-is-orm-how-does-it-work)
14. [Preserve Activity state during screen rotation](#14-preserve-activity-state-during-screen-rotation)
15. [Different ways to store data](#15-different-ways-to-store-data)
16. [Scoped Storage](#16-scoped-storage)
17. [How to encrypt data in Android](#17-how-to-encrypt-data-in-android)
18. [SharedPreferences commit() vs apply()](#18-sharedpreferences-commit-vs-apply)

### Memory Optimizations
19. [Improve Android app performance](#19-improve-android-app-performance)
20. [The onTrimMemory() method](#20-the-ontrimmemory-method)
21. [Identify and fix OutOfMemory issues](#21-identify-and-fix-outofmemory-issues)
22. [Find memory leaks](#22-find-memory-leaks)

### Battery Life Optimizations
23. [Adaptive Battery using ML](#23-adaptive-battery-using-ml)
24. [Reduce battery usage](#24-reduce-battery-usage)
25. [Doze and App Standby](#25-doze-and-app-standby)
26. [What is overdraw?](#26-what-is-overdraw)

### Supporting Different Screen Sizes
27. [Support different resolutions and screen sizes](#27-support-different-resolutions-and-screen-sizes)

### Permissions
28. [Permission protection levels](#28-permission-protection-levels)

### Native Programming
29. [What is the NDK and why is it useful?](#29-what-is-the-ndk-and-why-is-it-useful)
30. [What is RenderScript?](#30-what-is-renderscript)

### Android System Internal
31. [What is Android Runtime?](#31-what-is-android-runtime)
32. [Dalvik, ART, JIT, and AOT](#32-dalvik-art-jit-and-aot)
33. [Differences between Dalvik and ART](#33-differences-between-dalvik-and-art)
34. [Baseline Profiles](#34-baseline-profiles)
35. [What is DEX?](#35-what-is-dex)
36. [What is Multidex?](#36-what-is-multidex)
37. [Can you manually call the Garbage Collector?](#37-can-you-manually-call-the-garbage-collector)
38. [App starts: Hot, Warm & Cold](#38-app-starts-hot-warm--cold)

---

## 1. Run parallel tasks and get a callback when all complete

You want to launch several independent tasks concurrently and resume only once every one of them has finished.

**Coroutines with `async`/`awaitAll`** — the idiomatic approach. `async` starts a task and returns a `Deferred`; `awaitAll` suspends until all complete and collects results.

```kotlin
suspend fun loadDashboard(): Dashboard = coroutineScope {
    val user    = async { userRepo.fetch() }      // runs concurrently
    val feed    = async { feedRepo.fetch() }
    val notifs  = async { notifRepo.fetch() }

    // suspends until ALL three finish; rethrows the first failure
    val (u, f, n) = awaitAll(user, feed, notifs)
    Dashboard(u as User, f as Feed, n as Notifications)
}
```

`coroutineScope` gives **structured concurrency**: if any child throws, the others are cancelled and the exception propagates. Use `supervisorScope` if you want one failure not to cancel the siblings.

**Kotlin Flow** when tasks emit streams and you want to combine the latest values:

```kotlin
combine(flowA, flowB, flowC) { a, b, c -> Triple(a, b, c) }
    .collect { (a, b, c) -> /* called whenever all have emitted */ }
```

**Classic alternatives:** `CountDownLatch` (block a thread until N tasks call `countDown()`), or an `ExecutorService` with `invokeAll(tasks)` which returns once every `Callable` is done. Coroutines are preferred on Android because they are non-blocking, cancellable, and lifecycle-aware.

**Trade-off:** `async` parallelizes only if the underlying dispatcher has multiple threads or the work is non-blocking I/O; CPU-bound work should run on `Dispatchers.Default`, blocking I/O on `Dispatchers.IO`.

**📚 Reference:** https://outcomeschool.com/blog/long-running-tasks-in-parallel-with-kotlin-flow

---

## 2. What is ANR? How can it be prevented?

**ANR (Application Not Responding)** is a system dialog shown when the app's main (UI) thread is blocked for too long. Triggers:

- **Input dispatching timeout:** no response to an input event within **5 seconds**.
- **Service** `onCreate`/`onStartCommand`/`onBind` not finishing within **~20 s** (foreground) / 200 s (background).
- **BroadcastReceiver** `onReceive` not finishing within **10 s** (foreground) / **60 s** (background).
- `ContentProvider` not responding in time.

**Root cause:** doing slow work (network, disk I/O, large DB queries, heavy computation, synchronized locks) on the main thread.

**Prevention:**
- Move all blocking work off the main thread — coroutines on `Dispatchers.IO`/`Default`, `WorkManager` for deferrable jobs.
- Never do disk or network I/O on the main thread (StrictMode helps catch this in debug).
- Keep `BroadcastReceiver.onReceive` trivial; hand off to `WorkManager` or `goAsync()`.
- Avoid lock contention and synchronous binder calls on the UI thread.
- Use `Coroutine` + lifecycle scopes so work is cancelled when the screen goes away.

```kotlin
class StrictModeInit {
    fun enable() {
        StrictMode.setThreadPolicy(
            StrictMode.ThreadPolicy.Builder()
                .detectDiskReads().detectDiskWrites().detectNetwork()
                .penaltyLog()
                .build()
        )
    }
}
```

Monitor production ANRs via **Android Vitals** in the Play Console; the Play Store penalizes apps with high ANR rates in ranking.

**📚 Reference:** https://developer.android.com/topic/performance/vitals/anr.html

---

## 3. ThreadPool advantages

A **thread pool** reuses a fixed set of worker threads to execute many tasks, rather than creating a new thread per task.

**Advantages:**
- **Lower overhead:** thread creation/teardown is expensive (stack allocation, OS scheduling). A pool amortizes this cost.
- **Bounded resource use:** caps the number of concurrent threads, preventing the "thousands of threads" thrashing that exhausts memory and CPU.
- **Better throughput:** work queues smooth out bursts; tasks wait in a queue instead of overwhelming the system.
- **Lifecycle management:** centralized shutdown, scheduling, and rejection policies.

```kotlin
val pool = Executors.newFixedThreadPool(4)
pool.execute { doWork() }
pool.shutdown()
```

For CPU-bound work, size the pool near the core count (`Runtime.getRuntime().availableProcessors()`); for blocking I/O, use a larger pool or a cached pool. On Android, coroutine dispatchers (`Dispatchers.Default`, `Dispatchers.IO`) are backed by thread pools, so you rarely manage them directly.

**Trade-off:** a fixed pool that is too small can stall under load; `IO` dispatcher (default 64 threads) is for blocking calls, not CPU work.

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_outcomeschool-softwareengineer-tech-activity-7371117281713508352-x2Uf

---

## 4. Daemon threads vs. user threads

The JVM distinguishes two kinds of threads:

- **User (non-daemon) threads:** the JVM stays alive as long as at least one user thread is running. The main thread is a user thread.
- **Daemon threads:** background "service" threads (e.g., garbage collector). The JVM **does not wait** for daemon threads to finish; when all user threads exit, the JVM shuts down and daemon threads are abruptly terminated.

```kotlin
val t = Thread { while (true) { /* background work */ } }
t.isDaemon = true   // must be set BEFORE start()
t.start()
```

| | User thread | Daemon thread |
|---|---|---|
| Keeps JVM alive | Yes | No |
| Use for | Core app work | Background/support tasks |
| On JVM exit | JVM waits for it | Killed immediately |
| `isDaemon` default | false (inherits from creator) | must set true |

**Caution:** never use a daemon thread for work that must complete (e.g., flushing to disk) — it can be killed mid-operation. On Android this is rarely set manually; coroutine pool threads are daemon by default.

**📚 Reference:** https://x.com/amitiitbhu/status/1817783254885478872

---

## 5. Looper, Handler, and HandlerThread

These form Android's **message-loop** infrastructure for thread communication.

- **Looper** — turns a normal thread into a loop that processes a `MessageQueue` indefinitely. Each thread has at most one Looper. The main thread already has one (set up by `ActivityThread`).
- **MessageQueue** — the FIFO (priority by time) queue of `Message`/`Runnable` items the Looper drains.
- **Handler** — attached to a Looper; lets you **post** `Runnable`s or **send** `Message`s into that Looper's queue, and it processes them on that Looper's thread. This is how background threads schedule work back on the UI thread.
- **HandlerThread** — a convenience `Thread` subclass that already has a prepared Looper, so you can attach a Handler to it without writing the `prepare()`/`loop()` boilerplate.

```kotlin
// Run work on a background thread, then post the result to the UI thread.
val bg = HandlerThread("io").apply { start() }
val bgHandler = Handler(bg.looper)
val uiHandler = Handler(Looper.getMainLooper())

bgHandler.post {
    val result = expensiveWork()
    uiHandler.post { textView.text = result }
}
// when done: bg.quitSafely()
```

Manual Looper setup looks like:

```kotlin
class MyThread : Thread() {
    override fun run() {
        Looper.prepare()
        val handler = Handler(Looper.myLooper()!!) { msg -> /* handle */ true }
        Looper.loop()   // blocks, processing messages
    }
}
```

**Trade-off:** Looper/Handler is low-level and serial (one message at a time). Coroutines/`Dispatchers.Main` are built on top of the main Looper and are usually preferred for app code; HandlerThread still shines for ordered serial background processing (e.g., sensor or camera frame handling).

---

## 6. Garbage Collection

**Garbage Collection (GC)** automatically reclaims heap memory occupied by objects that are no longer reachable from **GC roots** (active threads' stacks, static fields, JNI references). The developer doesn't free memory manually; the runtime does.

**Reachability:** an object is *live* if a chain of references leads to it from a GC root. Anything unreachable is garbage and may be collected. ART uses a **generational** and **concurrent** collector — most objects die young, so the young generation is collected frequently and cheaply, while the old generation is collected less often.

**Why it matters on Android:**
- GC runs pause application threads (modern ART minimizes this with concurrent collection, but stop-the-world phases still exist).
- Excessive allocations (allocation churn) trigger frequent GCs that cause **jank** (dropped frames).
- A frequently GC-ing app shows `GC_FOR_ALLOC`/concurrent GC log lines and stutters.

**Practical guidance:** reduce object churn in hot paths (avoid allocations inside `onDraw`, loops, RecyclerView binding), reuse objects/pools, prefer primitive arrays over boxed collections, and avoid leaking objects (which keeps them reachable forever, defeating GC — see Q7).

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_java-tech-softwareengineer-activity-7308111597581799425-qZN0

---

## 7. Memory Leak vs Out of Memory (OOM)

These are related but distinct.

**Memory leak:** an object is no longer needed but is still **reachable** from a GC root, so GC cannot reclaim it. Memory usage grows over time. Common Android leaks:
- Holding an `Activity`/`Context` in a static field, singleton, or long-lived object.
- A non-static inner class / anonymous `Runnable`/`Handler` holding an implicit reference to its outer Activity.
- Unregistered listeners, observers, broadcast receivers, or callbacks.
- Long-lived background threads referencing UI objects.

**OutOfMemoryError (OOM):** thrown when the app requests memory the heap cannot supply even after GC. Causes:
- Accumulated memory leaks finally exhausting the heap.
- A single huge allocation (e.g., decoding a full-resolution bitmap).
- Holding too much data in memory (large caches, unbounded lists).

**Relationship:** leaks are a *cause*; OOM is an *effect*. Not every OOM is from a leak (a one-shot giant allocation can OOM a clean app), and a leak may never reach OOM if the app restarts often.

```kotlin
// Leak: anonymous Runnable + Handler keeps the Activity alive for 10 minutes.
handler.postDelayed({ updateUi() }, 10 * 60 * 1000)  // ❌
// Fix: use a static/weak reference or remove callbacks in onDestroy().
override fun onDestroy() { handler.removeCallbacksAndMessages(null); super.onDestroy() }
```

Detect leaks with **LeakCanary**; profile heap with **Android Studio Memory Profiler**.

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_outcomeschool-softwareengineer-tech-activity-7344037400857219072-O_pe

---

## 8. Runnable vs Thread

Both relate to running code, but they are different abstractions:

- **`Thread`** is an actual unit of execution — a class representing an OS-backed thread. You subclass it (or pass it work) and call `start()`.
- **`Runnable`** is just a `@FunctionalInterface` describing a *task* (`run()`). It contains no threading logic; it must be handed to something that runs it (a `Thread`, an `Executor`, a `Handler`).

```kotlin
// As a Runnable (preferred: separates "what to run" from "how to run it")
val task = Runnable { doWork() }
Thread(task).start()

// As a Thread subclass (couples task and thread)
class Worker : Thread() { override fun run() { doWork() } }
Worker().start()
```

**Why prefer `Runnable`:**
- A class can implement `Runnable` and still extend another class (Java has no multiple inheritance).
- The same `Runnable` can be reused across threads or submitted to a thread pool / executor.
- Better separation of concerns; thread pools, `Handler.post`, and coroutines all consume `Runnable`/tasks.

Subclassing `Thread` is only sensible when you genuinely need to customize thread behavior. In modern code, you rarely use either directly — coroutines and executors are preferred.

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_softwareengineer-androiddev-android-activity-7279784055284420609-Xa8b

---

## 9. Handling bitmaps that take too much memory

A bitmap's memory is `width × height × bytes-per-pixel`. A 4000×3000 photo in `ARGB_8888` (4 bytes/pixel) is ~48 MB — far larger than the JPEG on disk. Decoding several at full size quickly causes OOM.

**Techniques:**

**1. Decode downsampled (inSampleSize).** Read bounds first, then load only the resolution you need.

```kotlin
fun decodeSampled(res: Resources, id: Int, reqW: Int, reqH: Int): Bitmap {
    val opts = BitmapFactory.Options().apply { inJustDecodeBounds = true }
    BitmapFactory.decodeResource(res, id, opts)        // reads size only, no pixels
    opts.inSampleSize = calculateInSampleSize(opts, reqW, reqH)
    opts.inJustDecodeBounds = false
    return BitmapFactory.decodeResource(res, id, opts)
}

fun calculateInSampleSize(o: BitmapFactory.Options, reqW: Int, reqH: Int): Int {
    var sample = 1
    val (h, w) = o.outHeight to o.outWidth
    while (h / sample > reqH || w / sample > reqW) sample *= 2
    return sample
}
```

**2. Use a cheaper config.** `RGB_565` (2 bytes/pixel) halves memory when alpha isn't needed; `ARGB_8888` is the default. (`HARDWARE` bitmaps keep pixels in graphics memory off the Java heap.)

**3. Reuse memory** with `inBitmap` (decode into an existing bitmap) — basis of bitmap pools.

**4. Cache in two tiers:** an in-memory `LruCache<String, Bitmap>` sized to a fraction of the app heap, plus a `DiskLruCache`.

```kotlin
val cacheSize = (Runtime.getRuntime().maxMemory() / 1024 / 8).toInt() // KB, ~1/8 heap
val memCache = object : LruCache<String, Bitmap>(cacheSize) {
    override fun sizeOf(key: String, b: Bitmap) = b.byteCount / 1024
}
```

**5. Recycle / release** large bitmaps you no longer need, and respond to `onTrimMemory` by evicting caches.

**6. In practice, use Coil or Glide** — they handle sampling, caching, pooling, lifecycle-aware cancellation, and hardware bitmaps automatically.

**📚 Reference:** https://developer.android.com/topic/performance/graphics/load-bitmap and https://developer.android.com/topic/performance/graphics/manage-memory

---

## 10. Bitmap pool

A **bitmap pool** is a cache of reusable `Bitmap` objects. Instead of allocating a new bitmap (and garbaging the old one) every time you decode an image, you take a suitably-sized bitmap out of the pool and decode into it via `BitmapFactory.Options.inBitmap`.

**Why:** bitmap allocation/deallocation is the biggest source of GC churn in image-heavy apps (lists, grids, carousels). Reusing memory means fewer allocations, fewer GCs, less jank.

**How it works:**
1. Maintain a pool keyed by allocation size (and config).
2. When decoding, set `options.inMutable = true` and `options.inBitmap = pool.get(width, height, config)`.
3. The decoder writes pixels into that existing buffer.
4. When a bitmap is no longer displayed, return it to the pool instead of letting GC collect it.

```kotlin
val opts = BitmapFactory.Options().apply {
    inMutable = true
    inBitmap = bitmapPool.getReusable(reqW, reqH, Bitmap.Config.ARGB_8888)
}
val bmp = BitmapFactory.decodeStream(input, null, opts)
```

From Android 4.4+, `inBitmap` only requires the reuse candidate to be **at least as large** as the decoded image (older versions required exact dimensions). Glide implements a sophisticated LRU bitmap pool internally; you rarely build one by hand.

**Trade-off:** the pool itself consumes memory, so it must be size-bounded (LRU) and trimmed under memory pressure.

**📚 Reference:** https://outcomeschool.com/blog/bitmap-pool

---

## 11. Jetpack DataStore Preferences

**DataStore** is the modern replacement for `SharedPreferences`. It stores key-value data **asynchronously and transactionally** using Kotlin coroutines and `Flow`, avoiding the main-thread I/O and the silent error swallowing of `SharedPreferences`.

Two flavors:
- **Preferences DataStore** — untyped key-value (like SharedPreferences, but async + Flow).
- **Proto DataStore** — typed, schema-defined via protocol buffers.

```kotlin
val Context.dataStore by preferencesDataStore(name = "settings")
val DARK_MODE = booleanPreferencesKey("dark_mode")

// Read as a Flow (reactive, no blocking)
val darkModeFlow: Flow<Boolean> = context.dataStore.data
    .map { prefs -> prefs[DARK_MODE] ?: false }

// Write (suspend, transactional)
suspend fun setDarkMode(enabled: Boolean) {
    context.dataStore.edit { it[DARK_MODE] = enabled }
}
```

**Advantages over SharedPreferences:**
- No `apply()`/`commit()` foot-guns; writes are atomic and won't block the UI thread.
- Reads are exposed as `Flow`, so the UI updates reactively.
- Surfaces I/O errors instead of swallowing them.
- Type safety with Proto DataStore.

**Trade-offs:** not a database — no querying, relations, or partial updates of large objects; for structured/relational data use Room. Migration helper `SharedPreferencesMigration` moves existing prefs into DataStore.

**📚 Reference:** https://outcomeschool.com/blog/jetpack-datastore-preferences

---

## 12. Persisting data in an Android app

"Persisting" means keeping data across process death / device reboot. Pick the mechanism by data shape and size:

| Need | Use |
|---|---|
| Small key-value settings | **DataStore** (Preferences/Proto); SharedPreferences (legacy) |
| Structured / relational / queryable | **Room** (SQLite ORM) |
| Large binary / files / media | Internal/external **file storage**, `MediaStore` |
| Sensitive secrets | EncryptedSharedPreferences / encrypted files / `KeyStore` |
| Remote / sync | Backend API + local cache (offline-first) |

**Best practices:**
- Do all persistence off the main thread (coroutines + `Dispatchers.IO`, or libraries that enforce it like Room/DataStore).
- Expose data via a **repository** so the UI is agnostic to storage.
- Use **offline-first**: Room as single source of truth, sync with the network.
- Don't store secrets in plain SharedPreferences.

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_outcomeschool-softwareengineer-tech-activity-7301836718003888128-oNi2

---

## 13. What is ORM? How does it work?

**ORM (Object-Relational Mapping)** maps relational database tables to programming-language objects, so you work with classes and methods instead of writing raw SQL and manually marshalling `Cursor` rows.

**How it works:**
- **Entities** — classes annotated to map to tables; fields map to columns.
- **Mapping layer** — converts objects ⇄ rows, and method calls / query annotations ⇄ SQL.
- **DAO / repository** — methods like `insert`, `query`, `delete` generate the SQL.

On Android, **Room** is the standard ORM (a layer over SQLite):

```kotlin
@Entity(tableName = "users")
data class User(@PrimaryKey val id: Int, val name: String)

@Dao
interface UserDao {
    @Query("SELECT * FROM users WHERE id = :id")
    suspend fun getUser(id: Int): User?

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(user: User)
}
```

Room validates SQL **at compile time**, generates the boilerplate, supports `Flow`/`LiveData` observation, migrations, and relations.

**Trade-offs:** ORMs add an abstraction layer and can hide inefficient queries (e.g., N+1). For performance-critical or complex queries you may still drop to raw SQL (Room supports `@RawQuery`).

---

## 14. Preserve Activity state during screen rotation

A configuration change (rotation) by default **destroys and recreates** the Activity, losing transient state. Approaches, simplest to most robust:

**1. `onSaveInstanceState` / restore** — for small transient UI state (scroll position, text). Limited to a few hundred KB (it's a `Bundle` parcelled across process boundaries); not for large data.

```kotlin
override fun onSaveInstanceState(out: Bundle) {
    super.onSaveInstanceState(out); out.putInt("count", count)
}
// restore in onCreate(savedInstanceState) or onRestoreInstanceState
```

**2. `ViewModel`** — the recommended approach. A `ViewModel` survives configuration changes (it's scoped to the Activity's lifecycle, not its instances), so data held there is retained without re-fetching.

```kotlin
class CounterViewModel : ViewModel() { var count = 0 }
val vm: CounterViewModel by viewModels()
```

**3. `SavedStateHandle`** — inside a ViewModel, survives **process death** too (backed by saved-instance-state). Best of both: retained across rotation *and* system-initiated process kill.

**4. Compose `rememberSaveable`** — preserves composable state across rotation and process death.

**5. Override the config change** (`android:configChanges="orientation|screenSize"`) — Activity isn't recreated; you handle layout yourself. Generally discouraged except for special cases (e.g., video/GL surfaces), as it bypasses resource reloading.

**Rule of thumb:** ViewModel for retained data, SavedStateHandle/onSaveInstanceState for critical small state that must survive process death, never serialize large objects into the bundle.

**📚 Reference:** https://www.youtube.com/watch?v=ORtieK5f_zg

---

## 15. Different ways to store data

1. **SharedPreferences** — legacy key-value for small settings (being superseded by DataStore).
2. **Jetpack DataStore** — modern async key-value (Preferences) or typed (Proto).
3. **Room / SQLite** — structured, relational, queryable data.
4. **Internal storage** — private files in the app sandbox (`filesDir`, `cacheDir`); removed on uninstall, no permission needed.
5. **External / shared storage** — media and documents via `MediaStore` and the Storage Access Framework (scoped storage).
6. **Encrypted storage** — `EncryptedSharedPreferences` / encrypted files / Android `KeyStore` for secrets.
7. **Cache** — `cacheDir` and `LruCache`/`DiskLruCache` for transient data the system can evict.
8. **Cloud / network** — backend API, Firebase, with a local cache for offline-first.

Choose by: size, structure, sensitivity, whether it must survive uninstall, and whether other apps need access.

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_outcomeschool-softwareengineer-tech-activity-7301836718003888128-oNi2

---

## 16. Scoped Storage

**Scoped Storage** is a privacy model (introduced in Android 10, **mandatory for apps targeting API 30+ on Android 11+**) that restricts an app's broad access to external storage. Instead of reading/writing anywhere with `READ/WRITE_EXTERNAL_STORAGE`, each app gets a sandboxed view.

**Key rules:**
- **App-specific directories** (`getExternalFilesDir()`) — full access, no permission, deleted on uninstall.
- **Shared media** (images/video/audio) — accessed via the **MediaStore** API. An app can freely read/write media it created; to modify or delete media created by *other* apps it must request user consent (e.g., `createWriteRequest`/`createDeleteRequest`).
- **Documents & other files** — accessed via the **Storage Access Framework (SAF)** / document picker, where the user explicitly grants access; no broad permission needed.
- `READ_EXTERNAL_STORAGE` only grants visibility into the shared media collections, not the whole filesystem. On Android 13+ it's replaced by granular `READ_MEDIA_IMAGES` / `READ_MEDIA_VIDEO` / `READ_MEDIA_AUDIO`, and Android 14 adds the partial **Selected Photos Access** (`READ_MEDIA_VISUAL_USER_SELECTED`).
- `MANAGE_EXTERNAL_STORAGE` ("All files access") grants broad access but is restricted by Google Play to apps with a qualifying use case (file managers, backup, antivirus).

```kotlin
// Save an image to shared media via MediaStore (scoped-storage compliant)
val values = ContentValues().apply {
    put(MediaStore.Images.Media.DISPLAY_NAME, "photo.jpg")
    put(MediaStore.Images.Media.MIME_TYPE, "image/jpeg")
    put(MediaStore.Images.Media.RELATIVE_PATH, Environment.DIRECTORY_PICTURES + "/MyApp")
}
val uri = contentResolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, values)
uri?.let { contentResolver.openOutputStream(it)?.use { os -> bitmap.compress(JPEG, 90, os) } }
```

**Benefit:** stronger user privacy and no need for broad storage permissions. **Trade-off:** apps that genuinely need raw filesystem access (file managers) must justify `MANAGE_EXTERNAL_STORAGE`, and migration from old `File`-path code requires using MediaStore/SAF and the photo picker.

**📚 Reference:** https://developer.android.com/about/versions/11/privacy/storage and https://source.android.com/docs/core/storage/scoped

---

## 17. How to encrypt data in Android

**1. Android Keystore** — the foundation. Keys are generated and stored in hardware-backed secure storage (TEE / StrongBox); the key material never leaves the secure hardware, and you can require user authentication to use a key.

```kotlin
val keyGen = KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore")
keyGen.init(
    KeyGenParameterSpec.Builder("my_key",
        KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT)
        .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
        .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
        .build()
)
val key = keyGen.generateKey()
val cipher = Cipher.getInstance("AES/GCM/NoPadding").apply { init(Cipher.ENCRYPT_MODE, key) }
val ciphertext = cipher.doFinal(plaintext.toByteArray())  // store ciphertext + cipher.iv
```

**2. Jetpack Security (EncryptedSharedPreferences / EncryptedFile)** — high-level wrappers using a Keystore-backed master key for transparent encrypted key-value and file storage. (Note: the original `androidx.security:security-crypto` was deprecated; the actively maintained path is its successor / Tink-based crypto, but the concept of Keystore-backed encrypted prefs/files remains the standard.)

```kotlin
val masterKey = MasterKey.Builder(context)
    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM).build()
val prefs = EncryptedSharedPreferences.create(
    context, "secret_prefs", masterKey,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
)
```

**3. SQLCipher** for encrypting a full Room/SQLite database.

**4. Transport encryption** — always use TLS/HTTPS; pin certificates for sensitive apps.

**Best practices:** never hardcode keys; use the Keystore, prefer AES-GCM (authenticated encryption), store the IV alongside ciphertext, and tie sensitive keys to biometric/user authentication where appropriate.

---

## 18. SharedPreferences commit() vs apply()

Both persist edits made via `SharedPreferences.Editor`, but differently:

- **`commit()`** — writes to disk **synchronously** on the calling thread and **returns a `Boolean`** indicating success/failure. Blocks; if called on the UI thread it can cause jank/ANR.
- **`apply()`** — writes the in-memory change immediately and schedules the disk write **asynchronously**, returning `void`. No success result. Multiple `apply()`s are coalesced. If a `commit()` is issued while an `apply()` is still pending, the `commit()` blocks until the pending async write finishes.

```kotlin
prefs.edit().putString("token", t).apply()   // async, no result — preferred on UI thread
val ok = prefs.edit().putBoolean("done", true).commit()  // sync, returns success
```

**Guidance:** use `apply()` by default (don't block the UI). Use `commit()` only when you genuinely need to know whether the write succeeded *and* you're already off the main thread (or in a context like `BroadcastReceiver`/process-shutdown where you need the write completed before returning). Better still, use **DataStore**, which is async, transactional, and surfaces errors.

---

## 19. Improve Android app performance

A checklist across the main axes:

**Startup**
- Reduce work in `Application.onCreate` and the first Activity; lazy-init libraries (App Startup library).
- Apply **Baseline Profiles** (see Q34) to AOT-compile hot startup paths.
- Avoid synchronous I/O / heavy dependency graphs at launch.

**Rendering / UI**
- Keep frames under the budget (~16 ms at 60 Hz, ~8 ms at 120 Hz). Avoid work in `onDraw`/`onBindViewHolder`.
- Flatten layouts; avoid deep nesting and overdraw (Q26). Use `ConstraintLayout`, `merge`, `ViewStub`.
- For Compose: stable parameters, `derivedStateOf`, `key`, avoid unnecessary recomposition, use `LazyColumn` keys.
- Use RecyclerView (DiffUtil, view recycling) for lists.

**Threading**
- Never block the main thread (Q2). Offload to coroutines/WorkManager.

**Memory**
- Avoid leaks (Q7/Q22), reuse bitmaps (Q10), respond to `onTrimMemory` (Q20), use appropriate caches.

**Network**
- Cache, batch, paginate, compress, use efficient formats (protobuf); coalesce requests; use WorkManager for deferrable sync.

**App size**
- R8 minification + resource shrinking, App Bundles, remove unused dependencies.

**Measure first:** use **Macrobenchmark** + **JankStats** + **Perfetto/System Trace** + **Android Vitals**; optimize what the data shows, not guesses.

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineer-tech-activity-7314877712815345664-iB7z

---

## 20. The onTrimMemory() method

`onTrimMemory(level)` is a callback (on `Application`, `Activity`, `Service`, `ComponentCallbacks2`) the system invokes to tell your app to **release memory** because the device is under memory pressure or your app's UI visibility changed. Responding well makes your process less likely to be killed.

**Key levels:**
- `TRIM_MEMORY_UI_HIDDEN` — your UI is no longer visible; release UI-only resources (caches tied to views).
- `TRIM_MEMORY_RUNNING_MODERATE` / `_LOW` / `_CRITICAL` — your app is still running but the system is low on memory; progressively release more.
- `TRIM_MEMORY_BACKGROUND` / `_MODERATE` / `_COMPLETE` — your app is cached in the background; the higher the level, the closer you are to being killed, so release as much as possible.

```kotlin
override fun onTrimMemory(level: Int) {
    super.onTrimMemory(level)
    when (level) {
        TRIM_MEMORY_UI_HIDDEN -> imageMemoryCache.evictAll()
        TRIM_MEMORY_COMPLETE, TRIM_MEMORY_MODERATE -> {
            imageMemoryCache.evictAll(); clearNonEssentialCaches()
        }
    }
}
```

**Why it matters:** apps that free memory under pressure stay in the LRU cache longer (faster warm starts) and reduce system-wide jank/OOM kills. (`onLowMemory()` is the older, coarser callback; `onTrimMemory` supersedes it.)

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_softwareengineer-androiddev-android-activity-7267752779727679488--kk4

---

## 21. Identify and fix OutOfMemory issues

**Identify:**
- **Reproduce & capture:** an `OutOfMemoryError` stack trace shows the allocation site (often bitmap decode or large collection).
- **Memory Profiler (Android Studio):** watch the heap graph for a sawtooth that trends upward (leak) vs. a stable ceiling. Capture a **heap dump** and inspect retained sizes and dominators.
- **LeakCanary:** auto-detects retained objects after they should have been collected and gives the reference chain.
- **Allocation tracking:** find churn-heavy code paths.

**Common causes & fixes:**
- **Oversized bitmaps** → downsample with `inSampleSize`, use `RGB_565`/hardware bitmaps, bitmap pool, or Coil/Glide (Q9, Q10).
- **Leaks** (static Context, unregistered listeners, Handlers, leaked Activities) → break the reference, use weak references, unregister in lifecycle callbacks (Q7).
- **Unbounded caches/lists** → bound with `LruCache`, paginate (Paging 3).
- **Large in-memory data** → stream instead of loading whole; use Room paging.
- **Honor `onTrimMemory`** to release under pressure (Q20).

**Last resorts (use sparingly):** `largeHeap="true"` in the manifest gives a bigger heap but is a band-aid that masks the real problem and increases GC cost; multiple processes to isolate memory-heavy work.

---

## 22. Find memory leaks

**Primary tool: LeakCanary.** Add the dependency; on debug builds it watches destroyed Activities/Fragments/ViewModels, forces a GC, and if an object that should be gone is still retained, it dumps the heap and reports the **shortest strong reference chain** from a GC root to the leaked object — usually pointing straight at the bug.

```kotlin
// build.gradle (debug only)
debugImplementation("com.squareup.leakcanary:leakcanary-android:<version>")
```

**Android Studio Memory Profiler:** capture two heap dumps (before/after an action that should free memory), compare retained counts, and inspect the reference path of suspicious instances (e.g., multiple `MainActivity` instances alive = leaked Activity).

**Common leak sources to check:**
- Static fields / singletons / companion objects holding a `Context`, `Activity`, or `View`.
- Inner/anonymous classes (`Handler`, `Runnable`, listeners, coroutines) capturing the outer Activity — use `WeakReference` or cancel/remove in `onDestroy`.
- Unregistered `BroadcastReceiver`s, `ContentObserver`s, sensor/location listeners, RxJava/coroutine subscriptions.
- Long-lived background threads referencing UI.
- Non-cancelled coroutine scopes (use `lifecycleScope`/`viewModelScope`).

**Prevention practices:** use lifecycle-aware components (ViewModel, lifecycle-scoped coroutines, `repeatOnLifecycle`), avoid passing `Activity` Context to long-lived objects (use `applicationContext`), and always pair register/unregister in symmetric lifecycle callbacks.

---

## 23. Adaptive Battery using ML

**Adaptive Battery** (introduced in Android 9 "Pie", built with DeepMind) uses on-device **machine learning** to predict which apps you'll use in the next few hours and which you won't. It places rarely-/soon-not-used apps into restrictive **App Standby Buckets**, limiting their background work (jobs, alarms, network) to conserve battery.

**App Standby Buckets** (the mechanism the ML drives):
- **Active** — app in use now; no restrictions.
- **Working set** — used regularly; mild restrictions.
- **Frequent** — used often but not daily; more restriction.
- **Rare** — seldom used; heavy restriction on jobs/alarms/network.
- **Restricted** — most aggressive bucket (Android 11+) for heavy background offenders.

The system continually re-buckets apps based on predicted usage. Your `JobScheduler`/`WorkManager` jobs and alarms are deferred more aggressively the deeper the bucket.

**Developer implications:**
- Don't fight the system: use `WorkManager` with appropriate constraints rather than wakelocks/exact alarms.
- Use **high-priority FCM** messages for truly time-sensitive delivery.
- Test with `adb shell am set-standby-bucket <pkg> rare` and Battery Historian.
- Behave well so the ML keeps your app in a friendlier bucket.

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_machinelearning-android-mobiledevelopment-activity-7375026631951917056-k0CB

---

## 24. Reduce battery usage

Battery drain comes mainly from **radio (network), wakeups, wakelocks, CPU, GPS, and screen**. Techniques:

**Network**
- Batch and coalesce requests; avoid frequent polling — use FCM push instead.
- Cache aggressively; use efficient payloads (gzip, protobuf); prefetch on Wi-Fi/charging.
- Use **WorkManager** with constraints (`requiresCharging`, `requiredNetworkType`, `requiresDeviceIdle`) so work runs at battery-friendly times.

**Wakeups & scheduling**
- Avoid exact/repeating `AlarmManager`; prefer `WorkManager`/`JobScheduler` which batch system-wide.
- Don't hold **wakelocks**; if unavoidable, hold the shortest time and always release.
- Respect **Doze** and App Standby (Q25) — don't try to circumvent them.

**Location**
- Use the lowest sufficient accuracy and longest interval; use `FusedLocationProviderClient`; stop updates when not needed; use geofencing/activity recognition instead of continuous GPS.

**Compute & UI**
- Avoid busy loops and background CPU churn; reduce wakeful work.
- Reduce screen-on costs: dark themes on OLED, avoid keeping the screen awake unnecessarily.

**Measure** with Battery Historian, the Energy Profiler, and Android Vitals "excessive background battery usage" / "wakelock" metrics.

---

## 25. Doze and App Standby

Both are **power-management** features that restrict background work to save battery.

**Doze (device-level):** when the device is unplugged, stationary, and the screen is off for a while, the system enters Doze and **batches** deferred work into periodic **maintenance windows**, suspending most background activity in between:
- Network access is suspended (except during maintenance windows).
- `WorkManager`/`JobScheduler` jobs and syncs are deferred.
- Standard `AlarmManager` alarms are deferred to maintenance windows (use `setAndAllowWhileIdle`/`setExactAndAllowWhileIdle` sparingly for critical alarms).
- Wakelocks are ignored.
- A lighter "Doze on the go" kicks in even when moving (Android 7+).
High-priority FCM messages can still reach the app for urgent delivery.

**App Standby (app-level):** the system deems an *individual* app idle if the user hasn't interacted with it and it has no foreground process/notification. Idle apps have their network access and jobs restricted, deferred to roughly once-a-day windows (and more granularly via the **App Standby Buckets** the Adaptive Battery ML manages — see Q23).

**Developer guidance:** design for deferred execution; use WorkManager (Doze-aware), use FCM high-priority for time-critical messages, request battery-optimization exemption only when truly justified (Play restricts it), and test with `adb shell dumpsys deviceidle force-idle`.

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_androiddev-activity-7319939901418795008-KRql

---

## 26. What is overdraw?

**Overdraw** happens when the GPU paints the same pixel **multiple times in a single frame** — e.g., an opaque background drawn, then a card drawn on top, then a button on top of that. Each redundant layer wastes GPU fill-rate and can cause jank, especially on low-end devices.

**Visualize it:** Developer Options → **Debug GPU Overdraw**. The screen is tinted:
- No color = no overdraw (1x)
- Blue = 1x overdraw, Green = 2x, Pink = 3x, Red = 4x+ (bad).

**How to reduce:**
- Remove unnecessary backgrounds — don't set a window background *and* an opaque view background that covers it; remove the theme background when a full-screen view covers it (`getWindow().setBackgroundDrawable(null)`).
- Flatten the view hierarchy (fewer stacked opaque layers); use `ConstraintLayout`.
- Avoid painting views fully hidden behind others.
- Use `clipRect`/`canvas.quickReject` in custom views to skip drawing off-screen/occluded regions.
- Don't stack multiple opaque layers when one suffices.

**Goal:** most of the screen should be 1x–2x overdraw; large red areas indicate layout waste to fix.

**📚 Reference:** https://developer.android.com/topic/performance/rendering/overdraw.html

---

## 27. Support different resolutions and screen sizes

Android runs on phones, foldables, tablets, Chromebooks, and TVs with varied densities and sizes. Build adaptive, not pixel-perfect, layouts.

**Density independence:**
- Use **dp** for dimensions and **sp** for text (scales with user font setting), never raw px.
- Provide bitmaps per density bucket (`drawable-mdpi/hdpi/xhdpi/xxhdpi/xxxhdpi`) or use **vector drawables** (density-independent, single asset).

**Flexible layouts:**
- Use `ConstraintLayout`, `match_parent`/`0dp` with weights/constraints, and `wrap_content`; avoid hardcoded sizes and absolute positioning.
- Use `ScrollView`/scrollable containers so content fits small screens.

**Alternative resources via qualifiers:**
- **Smallest-width** qualifiers: `res/layout-sw600dp/` (≥600dp = typical tablet), `res/values-sw600dp/`.
- **Available width/height** (`-w600dp`, `-h720dp`) for responsive switching (e.g., list-detail two-pane on wide screens).
- Orientation (`-land`/`-port`), density, locale qualifiers.

**Modern adaptive APIs:**
- **WindowSizeClass** (Compact/Medium/Expanded) to drive responsive UI in Views and Compose.
- **Jetpack Compose** adaptive layouts + Material 3 **adaptive** components (list-detail, supporting pane) for foldables/large screens.
- Handle foldables (hinge/posture) via Jetpack WindowManager.

**Test** on multiple emulators/resizable emulator, foldable configs, and different font scales.

**📚 Reference:** https://developer.android.com/training/multiscreen/screensizes

---

## 28. Permission protection levels

A permission's **protection level** (declared in its `<permission>` definition) determines how the system grants it.

**Core levels:**
- **`normal`** — low-risk permissions (e.g., `INTERNET`, `VIBRATE`, `ACCESS_NETWORK_STATE`). Granted automatically at install; no user prompt.
- **`dangerous`** — touch sensitive user data or device features (camera, location, contacts, microphone, storage). Must be requested at **runtime** (Android 6.0+) and the user can grant/deny/revoke. Grouped into permission groups.
- **`signature`** — granted automatically only if the requesting app is **signed with the same certificate** as the app/system that declared the permission. Used for trusted inter-app communication.
- **`signatureOrSystem` / `system`** (legacy) — granted to apps signed with the system key or installed on the system image. Largely deprecated in favor of `signature` + flags.

**Protection flags** (modifiers combined with a base level), e.g.: `privileged`, `preinstalled`, `appop`, `instant`, `runtime`, `development`, `setup`, `role` — refine when/how special and signature permissions are granted.

```xml
<!-- Declaring a custom signature-level permission -->
<permission android:name="com.example.PRIVATE"
            android:protectionLevel="signature" />

<!-- Requesting a dangerous permission (also request at runtime) -->
<uses-permission android:name="android.permission.CAMERA" />
```

```kotlin
// Runtime request for a dangerous permission
val launcher = registerForActivityResult(ActivityResultContracts.RequestPermission()) { granted ->
    if (granted) startCamera() else showRationale()
}
launcher.launch(Manifest.permission.CAMERA)
```

**Special permissions** (e.g., `SYSTEM_ALERT_WINDOW`, `MANAGE_EXTERNAL_STORAGE`) are neither normal nor standard-dangerous — they require sending the user to a dedicated Settings screen.

---

## 29. What is the NDK and why is it useful?

The **NDK (Native Development Kit)** lets you write parts of your app in **C/C++** (and other native languages) and call them from Kotlin/Java via **JNI (Java Native Interface)**. The native code compiles to platform `.so` libraries packaged in the APK/AAB.

**Why it's useful:**
- **Performance-critical code** — signal/image/audio/video processing, physics, math-heavy loops where native + SIMD beats JVM/ART.
- **Reusing existing C/C++ libraries** — codecs, crypto, computer-vision (OpenCV), game engines, FFmpeg, etc.
- **Cross-platform code sharing** — a C/C++ core shared between Android and iOS.
- **Low-level access** — OpenGL ES/Vulkan, OpenSL ES, hardware APIs.
- **Games / real-time** — engines (Unity, Unreal) use native code.

```kotlin
class NativeLib {
    external fun process(input: IntArray): IntArray   // implemented in C/C++
    companion object { init { System.loadLibrary("native-lib") } }
}
```

```cpp
extern "C" JNIEXPORT jintArray JNICALL
Java_com_example_NativeLib_process(JNIEnv* env, jobject, jintArray input) { /* ... */ }
```

**Trade-offs:** JNI boundary crossings have overhead (don't use native for trivial work), debugging is harder, you must manage memory manually, build complexity (CMake/ndk-build) increases, and you ship per-ABI binaries (use App Bundles to deliver per-device ABI). Use the NDK only where it provides a clear, measured win — most app logic should stay in Kotlin.

**📚 Reference:** https://outcomeschool.com/blog/ndk-and-renderscript

---

## 30. What is RenderScript?

**RenderScript** was Android's framework for high-performance, data-parallel computation — useful for image processing, blur, blending, and other per-pixel work — that the runtime could distribute across CPU cores and (historically) GPU/DSP.

**Status (important as of 2026): RenderScript is deprecated.** It was deprecated in **Android 12 (API 31)** and Google recommends migrating off it; support is expected to be removed in a future release.

**What to use instead:**
- **RenderScript Intrinsics Replacement Toolkit** — a drop-in, standalone replacement for the common intrinsics (blur, blend, convolve, color matrix, resize); runs on CPU and is roughly 2x faster than the old CPU intrinsics.
- **Vulkan** or **OpenGL ES (3.1+) compute** — for true GPU-accelerated compute (recommended for full GPU advantage; usable when `minSdk` ≥ 24).
- **RenderEffect** (`View.setRenderEffect`, API 31+) — for view blur, replacing the common "blur with RenderScript" use case.
- **AGSL (Android Graphics Shading Language)** / `RuntimeShader` (API 33+) — programmable shader effects on Canvas.

```kotlin
// Blur a View on Android 12+ without RenderScript:
myView.setRenderEffect(RenderEffect.createBlurEffect(20f, 20f, Shader.TileMode.CLAMP))
```

**Summary for interview:** describe what RenderScript did, state clearly that it's deprecated since Android 12, and name the modern replacements (Intrinsics Replacement Toolkit, Vulkan/GLES compute, RenderEffect, AGSL).

**📚 Reference:** https://outcomeschool.com/blog/ndk-and-renderscript and https://developer.android.com/guide/topics/renderscript/migrate

---

## 31. What is Android Runtime?

**Android Runtime (ART)** is the managed runtime environment that executes Android apps. App code is compiled to **DEX bytecode** (Dalvik Executable), and ART runs that bytecode — handling execution, memory management, and garbage collection. ART replaced the older **Dalvik** VM as the default from **Android 5.0 (Lollipop)**.

Key responsibilities:
- Loading and executing DEX bytecode.
- **Compilation:** a hybrid of **AOT** (ahead-of-time), **JIT** (just-in-time), and **profile-guided** compilation (see Q32).
- **Memory management & garbage collection** (concurrent, generational; see Q6).
- Providing core runtime services and interpreting/optimizing hot code.

Each app runs in its own process with its own ART instance, isolated by the Linux kernel and the app sandbox. ART continues to evolve (it ships as an updatable system module / Mainline on recent Android versions, so runtime improvements arrive without a full OS update).

**📚 Reference:** https://outcomeschool.com/blog/dalvik-art-jit-aot

---

## 32. Dalvik, ART, JIT, and AOT

**Compilation strategies:**
- **JIT (Just-In-Time):** bytecode is compiled to native machine code **at runtime**, on demand, while the app runs. Fast install, smaller storage, but adds runtime compilation overhead and a warm-up cost.
- **AOT (Ahead-Of-Time):** bytecode is compiled to native code **before execution** (e.g., at install). Faster execution and startup (no runtime compile), but slower/longer installs and larger storage footprint.

**The runtimes:**
- **Dalvik** (Android ≤ 4.4) — register-based VM using **JIT** only. Compiled hot code each run.
- **ART** (Android 5.0+) — originally used **full AOT at install** (long installs, big footprint). Since **Android 7.0 (Nougat)**, ART uses a **hybrid**: interpret first, **JIT** hot methods while recording a **profile**, then **AOT-compile the profiled hot code in the background while the device is idle/charging** (profile-guided compilation). This gives fast installs *and* good runtime performance.

**Baseline Profiles** (Q34) extend this by shipping a profile of hot paths with the app so the AOT compilation happens at install rather than waiting to be learned from usage.

**📚 Reference:** https://outcomeschool.com/blog/dalvik-art-jit-aot

---

## 33. Differences between Dalvik and ART

| Aspect | Dalvik (≤ Android 4.4) | ART (Android 5.0+) |
|---|---|---|
| Compilation | JIT only (compile at runtime each launch) | Hybrid: AOT + JIT + profile-guided (since 7.0) |
| App startup | Slower (JIT warm-up every run) | Faster (precompiled hot code) |
| Runtime performance | Lower | Higher |
| Install time / size | Faster install, smaller | (Early ART) slower install, larger; later mitigated by profile-guided AOT |
| Garbage collection | More/longer pauses | Concurrent, generational; fewer/shorter pauses |
| Battery | More CPU at runtime for JIT | Less runtime compilation work |
| Debugging/diagnostics | Basic | Improved diagnostics, better stack traces |
| Updatability | Part of OS | ART is an updatable Mainline module on recent versions |

**One-liner:** Dalvik JIT-compiled bytecode every run; ART precompiles hot code (profile-guided AOT) for faster, smoother, more power-efficient execution, with a far better garbage collector.

**📚 Reference:** https://outcomeschool.com/blog/dalvik-art-jit-aot

---

## 34. Baseline Profiles

A **Baseline Profile** is a list of classes and methods (hot code paths — startup, critical user journeys, scrolling) shipped **with your app**. At install time, ART **AOT-compiles** exactly those paths instead of waiting to learn them from usage via JIT, so the app is fast from the very first run.

**Benefits:**
- Faster **app startup** (often 20–40% improvement on cold start).
- Smoother first scrolls / interactions (less JIT warm-up jank).
- Improvements apply from the first launch, before profile-guided compilation would have kicked in.

**How to create them:**
- Use the **Baseline Profile Gradle plugin** + **Macrobenchmark** module with a `BaselineProfileRule` test that exercises startup and key journeys; it generates `baseline-prof.txt`, packaged into the app.

```kotlin
@RunWith(AndroidJUnit4::class)
class BaselineProfileGenerator {
    @get:Rule val rule = BaselineProfileRule()

    @Test fun generate() = rule.collect(packageName = "com.example.app") {
        startActivityAndWait()
        // scroll the main list, navigate key screens...
    }
}
```

Libraries can also ship their own Baseline Profiles, which get merged. Combine with **Startup Profiles** (R8 uses them to optimize DEX layout). Verify improvements with Macrobenchmark `StartupTimingMetric`.

**📚 Reference:** https://outcomeschool.substack.com/p/baseline-profiles-in-android

---

## 35. What is DEX?

**DEX (Dalvik Executable)** is Android's bytecode format. The Kotlin/Java compiler produces `.class` JVM bytecode; the **D8** compiler (and **R8** for optimization/shrinking) converts that into a `classes.dex` file packaged in the APK/AAB. The Android runtime (Dalvik historically, ART now) executes DEX, not standard JVM `.class` files.

**Why a separate format:**
- DEX is optimized for **memory-constrained devices** — it uses a **register-based** instruction set (vs. the JVM's stack-based one), needing fewer instructions.
- Multiple classes are packed into a single `.dex` with **shared constant pools**, reducing duplication and size.

**Build pipeline:** `.kt/.java` → `javac`/`kotlinc` → `.class` → **D8/R8** → `classes.dex` → APK/AAB. R8 additionally minifies, shrinks unused code/resources, and obfuscates.

**Limit:** a single DEX file can reference at most **65,536 (64K) methods** (the `methodIdx` is a 16-bit field) — which leads to Multidex (Q36).

**📚 Reference:** https://developer.android.com/reference/dalvik/system/DexFile

---

## 36. What is Multidex?

A single DEX file can reference a maximum of **65,536 methods** (the "64K reference limit"). Large apps (with many libraries) exceed this, causing a build error. **Multidex** splits the app's code across **multiple DEX files** (`classes.dex`, `classes2.dex`, …) so it can reference more than 64K methods.

**Enabling it:**

```kotlin
android {
    defaultConfig { multiDexEnabled = true }
}
dependencies { implementation("androidx.multidex:multidex:<version>") }
```

- On **Android 5.0+ (API 21+, ART)**, multidex is supported **natively** — ART loads multiple DEX files directly; you only need `multiDexEnabled = true`.
- On **API < 21 (Dalvik)**, you also need the `androidx.multidex` library and to install it (extend `MultiDexApplication` or call `MultiDex.install` in `attachBaseContext`), because Dalvik loads only the primary DEX at startup and the rest must be loaded explicitly. Since modern `minSdk` is almost always ≥ 21, the legacy support library is rarely needed now.

**Best practice:** rather than relying on multidex, **reduce method count** with R8 (remove unused code/dependencies) — fewer methods means smaller, faster apps and avoids legacy-multidex startup overhead.

**📚 Reference:** https://www.youtube.com/watch?v=R0zd8lmHnmE

---

## 37. Can you manually call the Garbage Collector?

You can **request** a GC, but you **cannot force** it. `System.gc()` (or `Runtime.getRuntime().gc()`) is only a **hint** to the JVM/ART; the runtime is free to ignore it or run it whenever it chooses.

```kotlin
System.gc()   // a suggestion, not a guarantee
```

**Why you generally shouldn't:**
- ART's GC is concurrent, generational, and far better at deciding *when* to collect than you are.
- A manual GC can trigger a **stop-the-world** pause at the worst moment, causing **jank**, and may hurt performance rather than help.
- It doesn't fix leaks: objects still **reachable** from a GC root won't be collected no matter how many times you call `gc()`. The real fix is to drop references (set to `null`, unregister listeners, use weak references).

**Legitimate (rare) uses:** before taking a heap dump in profiling/tests to get a clean snapshot, or in a benchmark harness to reduce noise — never as a production "memory fix." If you're tempted to call it to solve OOM, you actually have a leak or oversized allocation to fix (Q21, Q22).

**📚 Reference:** https://www.youtube.com/watch?v=fPEjpFKo1-Q

---

## 38. App starts: Hot, Warm & Cold

App startup is classified by how much the system must do before the app is interactive. Faster startup is a key performance metric (tracked in Android Vitals).

**Cold start (slowest):** the app process does **not** exist. The system must: fork/create the process, initialize the `Application` object (`onCreate`), then create and draw the first Activity. This is the most expensive and the case you optimize for.

**Warm start (medium):** the process is **alive** but the Activity must be recreated — e.g., the user pressed Back then relaunched, or the Activity was destroyed (low memory) while the process lingered. Skips process and `Application` creation but still rebuilds the Activity and its UI (often restoring from `savedInstanceState`).

**Hot start (fastest):** the process **and** the Activity already exist (e.g., app brought back to foreground from background). The system just brings the existing Activity to the front — minimal work, near-instant.

| | Process | Application.onCreate | Activity recreated |
|---|---|---|---|
| Cold | created | runs | yes |
| Warm | exists | skipped | yes |
| Hot | exists | skipped | no (already exists) |

**Optimizing cold start (the target):**
- Trim `Application.onCreate` and first-Activity work; lazy-init with the **App Startup** library.
- Ship **Baseline Profiles** (Q34).
- Avoid heavy synchronous I/O / large dependency graphs at launch.
- Use a proper themed launch (windowBackground) instead of a fake splash with work on the main thread; use the **SplashScreen API**.
- Defer non-critical initialization until after first frame.
- Measure with **Macrobenchmark** `StartupTimingMetric` and **Android Vitals**.

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_androiddev-activity-7374668708679462912-DPH3
