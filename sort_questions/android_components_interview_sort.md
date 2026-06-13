# Android Core Components & Lifecycle — Short Answers (Quick Revision)

> Condensed answers for rapid review before an interview. For full explanations and code see `android_components_interview.md`.

---

## Base / Fundamentals

## 1. Why does an Android app lag?

The app misses its frame budget (~16.6 ms at 60 Hz) because the main thread is busy. Common causes: heavy work on the UI thread (network/disk/parsing/bitmap decode), overdraw and deep layouts, GC churn, inefficient lists, and leaks. Fix by moving work off the main thread, reducing allocations/overdraw, and flattening the view hierarchy.

---

## 2. What is `Context`? Application vs Activity Context

`Context` is the handle to system services, resources, and app environment. **Application context** lives for the whole process — use it for singletons/DB/long-lived objects (won't leak an activity). **Activity context** is tied to the activity and carries themed resources — use it for UI work (dialogs, inflation). Holding an Activity context in a static/singleton leaks the activity.

---

## 3. How does Zygote make Android apps start faster?

Zygote is a process started at boot that preloads and initializes core framework classes and resources once. New apps are created by **forking** Zygote, sharing its warm memory pages via copy-on-write — so no re-loading/re-initializing per app. Result: faster startup and lower memory use.

---

## 4. What are the Android application components?

Four core components: **Activity** (UI screen), **Service** (background work, no UI), **BroadcastReceiver** (responds to broadcast events), and **ContentProvider** (shares data across apps). **Intent** is the messaging object that activates them. Activities, services, manifest receivers, and providers must be declared in the manifest.

---

## 5. What is the project structure of an Android app?

A Gradle project with an `app/` module containing `build.gradle`, and `src/main/` holding `AndroidManifest.xml`, source code, and `res/` (drawable, layout, mipmap, values). Plus `test/` (JVM) and `androidTest/` (instrumented). Resources are accessed via the generated `R` class; multi-module apps add library modules.

---

## 6. What is `AndroidManifest.xml`?

The app's mandatory descriptor read by the OS before any code runs. It declares the package/app id, all components, permissions, intent filters, and hardware/SDK requirements, plus the `<application>` element. Since Android 12, any component with an intent filter must set `android:exported` explicitly.

---

## 7. What is the `Application` class?

The base class representing the whole app process; a single instance is created at process start and lives as long as the process. It's the place for process-wide initialization and provides `getApplicationContext()`. Keep `onCreate()` fast (it directly delays cold start) and avoid stuffing mutable global state in it.

---

## 8. ART vs Dalvik

Both run DEX bytecode. **Dalvik** (≤4.4) used JIT only — small/fast install, slower runtime. **ART** (default since 5.0) originally used AOT at install for faster execution; modern ART is hybrid (interpret + JIT + profile-guided background AOT) with better GC. Trade-off: Dalvik = smaller install/slower run; ART = faster run/better GC.

---

## 9. What is ANR (Application Not Responding)?

The system dialog shown when the main thread is blocked too long: input not handled within ~5s, `BroadcastReceiver.onReceive` exceeding ~10s/60s, or service callbacks exceeding ~20s. Caused by blocking work on the main thread. Fix by moving work to background threads/coroutines and using StrictMode to detect violations.

---

## 10. What is ADB (Android Debug Bridge)?

A command-line tool and client-server protocol for communicating with a device/emulator — install apps, push/pull files, read logs, run a shell, and debug. It has a client (your machine), a daemon (`adbd`) on the device, and a server mediating between them.

---

## 11. Process vs Thread vs Task

- **Process** — OS-level isolation boundary; each app normally runs in its own process/VM with separate memory.
- **Thread** — unit of execution within a process; threads share memory (main/UI thread handles UI).
- **Task** — user-facing back stack of activities representing a navigation journey; can span apps.

---

## 12. Looper, Handler, and MessageQueue

They implement Android's per-thread message loop. **MessageQueue** holds messages/runnables ordered by time. **Looper** owns the queue and continuously dispatches items (main thread has one). **Handler** is bound to a Looper and posts/sends work onto its queue. Use `HandlerThread` to give your own thread a Looper; coroutines largely replace manual Handler use.

---

## Activity & Fragment

## 13. What is an `Activity` and its lifecycle?

An Activity is a single focused screen. Lifecycle callbacks: `onCreate` (inflate UI, restore state, once) → `onStart` (visible) → `onResume` (interactive/foreground) → `onPause` (losing focus) → `onStop` (not visible) → `onDestroy` (cleanup); `onRestart` runs before `onStart` when returning from stopped.

---

## 14. Difference between `onCreate()` and `onStart()`

`onCreate()` runs **once** when the instance is created — one-time setup (`setContentView`, restore state); not yet visible. `onStart()` runs **every time** the activity becomes visible — after `onCreate` and after `onRestart`. Mnemonic: `onCreate` = set up once; `onStart` = now visible, possibly again.

---

## 15. Why call `setContentView()` in `onCreate()`?

`onCreate()` runs once per instance, so the view tree is built once rather than re-inflated on every visibility change, and it's the earliest point where `savedInstanceState` is available to restore into views. (In Compose, `setContent { }` plays the same role and is also called in `onCreate`.)

---

## 16. When is only `onDestroy()` called without `onPause()`/`onStop()`?

When you call `finish()` inside `onCreate()` before the activity becomes visible — it never reaches `onStart`/`onResume`, so it skips `onPause`/`onStop` and goes straight to `onDestroy`. Common for router/splash activities that decide where to go and finish immediately.

---

## 17. `onSaveInstanceState()` and `onRestoreInstanceState()`

They preserve small transient UI state across system-initiated destruction (rotation, background process death). `onSaveInstanceState` writes to a `Bundle` (before destruction); `onRestoreInstanceState` (after `onStart`) and `onCreate` receive it — always null-check (null = fresh instance). Keep the bundle small (Binder limit); use `ViewModel`/disk for larger or longer-lived data.

---

## 18. Preventing data loss on rotation (ViewModel + saved state)

Layered strategy: **ViewModel** holds in-memory UI data and survives config changes (but not process death); **`SavedStateHandle`/onSaveInstanceState** for small state that must survive process death; **disk/DB (Room/DataStore)** as the source of truth for anything that must truly persist.

---

## 19. Lifecycle sequence across multiple activities (A → B → C transparent)

Key insight: a **transparent/non-fullscreen** activity pauses but does NOT stop the activity beneath it. Opening opaque B over A stops A; opening transparent C over B only pauses B (no `onStop`). Backing out resumes a paused activity directly, while a stopped activity goes through `onRestart` → `onStart` → `onResume`.

---

## 20. What are launch modes?

Control how instances are created/placed in the task:
- **standard** — new instance every time.
- **singleTop** — reuse via `onNewIntent()` if already at top.
- **singleTask** — one instance as task root; reuses and clears activities above.
- **singleInstance** — like singleTask but the only activity in its task.

Equivalent intent flags exist (`FLAG_ACTIVITY_NEW_TASK`, `CLEAR_TOP`, `SINGLE_TOP`).

---

## 21. What is a `Fragment` and its lifecycle?

A reusable UI module hosted inside an activity, with its own lifecycle tied to the host. Callbacks: `onAttach` → `onCreate` → `onCreateView` → `onViewCreated` → `onStart` → `onResume` … `onPause` → `onStop` → `onDestroyView` → `onDestroy` → `onDetach`. The **view lifecycle is shorter** than the fragment's — clear view/binding refs in `onDestroyView` and observe with `viewLifecycleOwner`.

---

## 22. `Fragment` vs `Activity` — relationship and when to use each

An Activity is a top-level, manifest-declared component with a window; a Fragment is a sub-component hosted inside an activity that can't exist alone. Use fragments for reusable UI, multi-pane layouts, and single-activity + Navigation architecture. Use an activity for a genuine separate entry point or external implicit-intent target.

---

## 23. Why use only the default constructor for a `Fragment`?

The framework recreates fragments via reflection using the **no-arg constructor** (after config change/process death), so a custom constructor's data is lost or crashes. Pass data via a `Bundle` set with `setArguments()` (often a `newInstance` factory) — arguments are automatically saved and restored.

---

## 24. `add` vs `replace` and `addToBackStack()`

`add` puts a fragment on top while existing ones stay attached (no teardown callbacks). `replace` removes current fragments first (outgoing fragment runs `onPause`/`onStop`/`onDestroyView`). `addToBackStack(name)` records the transaction so Back reverses it; without it a `replace` is irreversible and the prior fragment is lost.

---

## 25. `FragmentPagerAdapter` vs `FragmentStatePagerAdapter` (and ViewPager2)

`FragmentPagerAdapter` keeps all visited fragment instances in memory (good for few pages); `FragmentStatePagerAdapter` destroys off-screen fragments and saves only their state (good for many pages). Both are deprecated — use **`ViewPager2` + `FragmentStateAdapter`**, which is RecyclerView-backed and behaves like the state adapter.

---

## 26. What is a retained `Fragment`?

A fragment whose instance survived config changes via `setRetainInstance(true)` (only the view was recreated), historically used to hold expensive objects across rotation. It's now **deprecated** — use a **`ViewModel`** instead, which is purpose-built for surviving config changes without the pitfalls.

---

## 27. How to communicate between two Fragments?

Never directly. Preferred: a **shared `ViewModel`** scoped to the host activity (`activityViewModels()`) exchanging data via LiveData/StateFlow. For one-off results, the **Fragment Result API** (`setFragmentResult`/`setFragmentResultListener`). Older pattern: via the host activity through an interface (verbose, coupled).

---

## 28. What is a `Bundle`? Size limits

A key-value map (backed by `Parcelable`) for passing data between components and across the process boundary — used in intent extras, fragment arguments, and saved state. It travels via a Binder transaction with a ~1 MB buffer per process; exceeding it throws `TransactionTooLargeException`. Never put bitmaps/large lists in it — pass an id/URI instead.

---

## 29. Transferring objects between activities — Serializable vs Parcelable

Put data in intent extras. **Serializable** is a Java marker interface using reflection — easy but slow with GC pressure. **Parcelable** is Android-specific, reflection-free, faster, and recommended (use `@Parcelize` to generate boilerplate). Remember the Bundle size limit — pass ids, not large payloads.

---

## 30. `Dialog` vs `DialogFragment`

`Dialog` is a basic floating window, not lifecycle-aware — it dismisses/leaks on rotation. `DialogFragment` wraps a dialog in the fragment lifecycle/`FragmentManager`, so it survives config changes and manages its own state. Always prefer `DialogFragment` for persistent dialogs.

---

## Intents & Broadcasting

## 31. What is an `Intent`? Explicit vs Implicit

An asynchronous messaging object to start an Activity/Service or send a broadcast, carrying action, data, and extras. **Explicit** names the exact target component (in-app navigation). **Implicit** declares an action and lets the system find an app whose intent filter handles it (e.g. open URL, share). Android 11+ may need `<queries>` for package visibility.

---

## 32. What is a `BroadcastReceiver`? Types of broadcasts

A component reacting to system/app broadcast events via `onReceive` (runs on main thread — keep it quick). Registration is **manifest (static)** — restricted since Android 8 — or **runtime (dynamic)**. Broadcast types: **normal** (async, unordered), **ordered** (priority, abortable), **local** (deprecated), **sticky** (deprecated), and **system** broadcasts.

---

## 33. How broadcasts pass messages around your app

A publish/subscribe mechanism on Intents: a sender builds an Intent and calls `sendBroadcast`; the system matches it to registered receivers (via IntentFilter or explicit target); each receiver's `onReceive` reacts. Lets components communicate without direct references. For in-app messaging today, prefer a shared `ViewModel`/`StateFlow`; reserve broadcasts for cross-app/system events.

---

## 34. What is a `PendingIntent`? (and Sticky Intent)

A token wrapping an Intent plus permission to execute it, handed to another app/system to fire later on your behalf with your app's identity — even if your process is dead. Used by notifications, alarms, and widgets (`getActivity`/`getService`/`getBroadcast`). Since Android 12 you must set `FLAG_IMMUTABLE` or `FLAG_MUTABLE`. A **Sticky Intent** (deprecated) was a broadcast whose last value stuck for future receivers.

---

## Services

## 35. What is a `Service`? Its lifecycle

A component for long-running work without a UI; by default it runs on the main thread. **Started** (`startService`): `onCreate` → `onStartCommand` → runs until `stopSelf`/`stopService` → `onDestroy`. **Bound** (`bindService`): `onCreate` → `onBind` → clients interact → `onUnbind` → `onDestroy`. Can be both. For modern background work prefer WorkManager or a foreground service.

---

## 36. On which thread does a `Service` run?

By default the application's **main (UI) thread** — not a separate thread or process. Blocking work in `onStartCommand`/`onBind` can cause an ANR, so spawn your own background thread/coroutine. (`IntentService`, now deprecated, created its own worker thread; `android:process` runs it in another process but still on that process's main thread.)

---

## 37. `Service` vs `IntentService`

`Service` runs on the main thread, handles multiple/long-lived tasks, can be started and/or bound, and you manage threading and stopping. `IntentService` (a Service subclass) had its own worker thread, processed intents sequentially in `onHandleIntent`, and auto-stopped when done — for short fire-and-forget tasks. `IntentService` is deprecated; use WorkManager or a coroutine instead.

---

## 38. What is a Foreground Service?

A service for work the user is actively aware of (playback, navigation, uploads) that shows a persistent notification and gets higher priority, exempt from background limits. Call `startForeground` within ~5s, hold `FOREGROUND_SERVICE` permission, and on Android 14 declare a `foregroundServiceType` plus its matching per-type permission.

---

## 39. What is `JobScheduler`?

A system service (API 21+) for scheduling deferrable background jobs under constraints (unmetered network, charging, idle, deadline); the OS batches jobs across apps to save battery. You define a `JobService` and submit a `JobInfo`. For most apps **WorkManager** (built on top of it) is recommended instead.

---

## 40. How does `WorkManager` guarantee task execution?

It persists every `WorkRequest` to an internal Room/SQLite database so work survives app kill and reboot (rescheduled via a boot receiver). It delegates to the best OS scheduler (`JobScheduler` on API 23+), supports constraints and retry with backoff. The guarantee is **eventual** execution once constraints are met — not exact-time or instant.

---

## 41. What can you use for background processing in Android?

- **Coroutines/`Dispatchers.IO`** — in-process work while the app/screen is alive.
- **WorkManager** — deferrable, guaranteed work surviving exit/reboot.
- **Foreground Service** — immediate, user-visible ongoing work.
- **AlarmManager** — exact/inexact time-based triggers.
- Avoid: `AsyncTask`, `IntentService`, `LocalBroadcastManager`.

---

## Inter-Process Communication

## 42. How can two distinct Android apps interact?

Via **implicit Intents + intent filters** (share/open/deep link), **ContentProvider + ContentResolver** (structured data), **bound service with AIDL/Messenger** (programmatic IPC), **broadcasts**, and **FileProvider** (secure file sharing). Android 11+ package visibility may require `<queries>`; protect surfaces with permissions and `android:exported`.

---

## 43. Can an app run in multiple processes? How?

Yes — set `android:process` on a component in the manifest. A name starting with `:` creates a private process; a full lowercase name creates a global (shareable) one. Useful to isolate crash-prone/memory-heavy work. Caveats: each process gets its own VM and `Application` instance, separate memory (no shared singletons/statics), and must communicate via IPC.

---

## 44. What is AIDL? Steps to create a bound service with AIDL

AIDL (Android Interface Definition Language) defines a programmatic interface for cross-process IPC; it generates Binder marshalling (proxy/stub) code. Steps: (1) define the `.aidl` interface; (2) implement the generated `Stub` in a `Service` and return it from `onBind`; (3) declare the service in the manifest; (4) client binds, calls `Stub.asInterface(binder)`, then invokes methods. Calls may run on a Binder thread pool, so be thread-safe.

---

## 45. What is a `ContentProvider` and when is it used?

A component that exposes data via a URI-addressed interface (`content://`) accessed through a `ContentResolver`, abstracting storage behind `query`/`insert`/`update`/`delete`. Its main purpose is **securely sharing data with other apps** and integrating with framework features (Contacts, MediaStore, FileProvider). For internal-only data, use Room/DataStore directly — a ContentProvider is boilerplate-heavy overkill.
