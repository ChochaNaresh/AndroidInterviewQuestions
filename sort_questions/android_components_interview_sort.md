# Android Core Components & Lifecycle — Short Answers (Quick Revision)

> Condensed answers for rapid review before an interview. For full explanations and code see `android_components_interview.md`.

---

## Base / Fundamentals

## 1. Why does an Android app lag?

**App lag** means the stutter or visual delay that occurs when the app's main thread fails to render a frame within the display's refresh rate window.

The app misses its frame budget (~16.6 ms at 60 Hz) because the main thread is busy. Common causes: heavy work on the UI thread (network/disk/parsing/bitmap decode), overdraw and deep layouts, GC churn, inefficient lists, and leaks. Fix by moving work off the main thread, reducing allocations/overdraw, and flattening the view hierarchy.

---



## 2. What is `Context`? Application vs Activity Context

**Context** means an abstract handle to the Android environment that provides access to system services, resources, and application information.

`Context` is the handle to system services, resources, and app environment. **Application context** lives for the whole process — use it for singletons/DB/long-lived objects (won't leak an activity). **Activity context** is tied to the activity and carries themed resources — use it for UI work (dialogs, inflation). Holding an Activity context in a static/singleton leaks the activity.

---



## 3. How does Zygote make Android apps start faster?

**Zygote** means a warm parent process that preloads core Android framework classes and resource pools at boot time to rapidly fork new app processes.

Zygote is a process started at boot that preloads and initializes core framework classes and resources once. New apps are created by **forking** Zygote, sharing its warm memory pages via copy-on-write — so no re-loading/re-initialising per app. Result: faster startup and lower memory use.

---



## 4. What are the Android application components?

**Application components** means the fundamental building blocks of an Android application, consisting of Activities, Services, BroadcastReceivers, and ContentProviders.

Four core components: **Activity** (UI screen), **Service** (background work, no UI), **BroadcastReceiver** (responds to broadcast events), and **ContentProvider** (shares data across apps). **Intent** is the messaging object that activates them. Activities, services, manifest receivers, and providers must be declared in the manifest.

---



## 5. What is the project structure of an Android app?

**Project structure** means the layout of an Android project consisting of module configurations, source directories, and resource folders organised for the Gradle build system.

A Gradle project with an `app/` module containing `build.gradle`, and `src/main/` holding `AndroidManifest.xml`, source code, and `res/` (drawable, layout, mipmap, values). Plus `test/` (JVM) and `androidTest/` (instrumented). Resources are accessed via the generated `R` class; multi-module apps add library modules.

---



## 6. What is `AndroidManifest.xml`?

**AndroidManifest.xml** means a mandatory XML descriptor file at the root of the project that declares the app's metadata, components, permissions, and requirements to the OS.

The app's mandatory descriptor read by the OS before any code runs. It declares the package/app id, all components, permissions, intent filters, and hardware/SDK requirements, plus the `<application>` element. Since Android 12, any component with an intent filter must set `android:exported` explicitly.

---



## 7. What is the `Application` class?

**Application** means a base class that represents the entire application process and maintains global state across all components.

The base class representing the whole app process; a single instance is created at process start and lives as long as the process. It's the place for process-wide initialisation and provides `getApplicationContext()`. Keep `onCreate()` fast (it directly delays cold start) and avoid stuffing mutable global state in it.

---



## 8. ART vs Dalvik

**ART (Android Runtime)** means the modern managed runtime environment that compiles DEX bytecode using hybrid Ahead-Of-Time and Just-In-Time compilation, whereas **Dalvik** was the legacy runtime that compiled bytecode strictly via a Just-In-Time compiler.

Both run DEX bytecode. **Dalvik** (≤4.4) used JIT only — small/fast install, slower runtime. **ART** (default since 5.0) originally used AOT at install for faster execution; modern ART is hybrid (interpret + JIT + profile-guided background AOT) with better GC. Trade-off: Dalvik = smaller install/slower run; ART = faster run/better GC.

---



## 9. What is ANR (Application Not Responding)?

**ANR (Application Not Responding)** means a system-triggered dialog shown to the user when the main thread of an application remains blocked for too long.

The system dialog shown when the main thread is blocked too long: input not handled within ~5s, `BroadcastReceiver.onReceive` exceeding ~10s/60s, or service callbacks exceeding ~20s. Caused by blocking work on the main thread. Fix by moving work to background threads/coroutines and using StrictMode to detect violations.

---



## 10. What is ADB (Android Debug Bridge)?

**ADB (Android Debug Bridge)** means a versatile command-line utility that facilitates communication, debugging, and command execution between a development machine and an Android device.

A command-line tool and client-server protocol for communicating with a device/emulator — install apps, push/pull files, read logs, run a shell, and debug. It has a client (your machine), a daemon (`adbd`) on the device, and a server mediating between them.

---



## 11. Process vs Thread vs Task

**Process** means an isolated OS-level resource allocation boundary, **thread** means the smallest unit of execution within a process, and **task** means a collection of user-facing activities arranged in a back stack.

- **Process** — OS-level isolation boundary; each app normally runs in its own process/VM with separate memory.
- **Thread** — unit of execution within a process; threads share memory (main/UI thread handles UI).
- **Task** — user-facing back stack of activities representing a navigation journey; can span apps.

---



## 12. Looper, Handler, and MessageQueue

**Looper** means a helper class that keeps a thread alive to run a message loop, **MessageQueue** is a queue holding runnable tasks or messages, and **Handler** is a mechanism used to post tasks or send messages to a specific looper's queue.

They implement Android's per-thread message loop. **MessageQueue** holds messages/runnables ordered by time. **Looper** owns the queue and continuously dispatches items (main thread has one). **Handler** is bound to a Looper and posts/sends work onto its queue. Use `HandlerThread` to give your own thread a Looper; coroutines largely replace manual Handler use.

---



## Activity & Fragment

## 13. What is an `Activity` and its lifecycle?

**Activity** means a single, focused screen that presents a user interface and manages its lifecycle through a series of system-driven callbacks.

An Activity is a single focused screen. Lifecycle callbacks: `onCreate` (inflate UI, restore state, once) → `onStart` (visible) → `onResume` (interactive/foreground) → `onPause` (losing focus) → `onStop` (not visible) → `onDestroy` (cleanup); `onRestart` runs before `onStart` when returning from stopped.

---



## 14. Difference between `onCreate()` and `onStart()`

**onCreate()** means the one-time lifecycle callback where initial setup and view inflation are performed when an activity is created, whereas **onStart()** is a callback invoked every time the activity becomes visible to the user.

`onCreate()` runs **once** when the instance is created — one-time setup (`setContentView`, restore state); not yet visible. `onStart()` runs **every time** the activity becomes visible — after `onCreate` and after `onRestart`. Mnemonic: `onCreate` = set up once; `onStart` = now visible, possibly again.

---



## 15. Why call `setContentView()` in `onCreate()`?

**setContentView()** means the method that inflates the XML layout and attaches the view tree to the activity's window, which is called in `onCreate()` to ensure view initialisation happens only once.

`onCreate()` runs once per instance, so the view tree is built once rather than re-inflated on every visibility change, and it's the earliest point where `savedInstanceState` is available to restore into views. (In Compose, `setContent { }` plays the same role and is also called in `onCreate`.)

---



## 16. When is only `onDestroy()` called without `onPause()`/`onStop()`?

**Direct destruction** means destroying an activity immediately from `onCreate()` by calling `finish()`, which bypasses the visibility-related `onPause()` and `onStop()` states.

When you call `finish()` inside `onCreate()` before the activity becomes visible — it never reaches `onStart`/`onResume`, so it skips `onPause`/`onStop` and goes straight to `onDestroy`. Common for router/splash activities that decide where to go and finish immediately.

---



## 17. `onSaveInstanceState()` and `onRestoreInstanceState()`

**onSaveInstanceState()** means a callback used to write transient UI state into a `Bundle` before destruction, whereas **onRestoreInstanceState()** is a callback invoked during recreation to restore the saved state.

They preserve small transient UI state across system-initiated destruction (rotation, background process death). `onSaveInstanceState` writes to a `Bundle` (before destruction); `onRestoreInstanceState` (after `onStart`) and `onCreate` receive it — always null-check (null = fresh instance). Keep the bundle small (Binder limit); use `ViewModel`/disk for larger or longer-lived data.

---



## 18. Preventing data loss on rotation (ViewModel + saved state)

**State persistence strategy** means combining `ViewModel` to retain rich data across configuration changes with `SavedStateHandle` to preserve essential transient data across process death.

Layered strategy: **ViewModel** holds in-memory UI data and survives config changes (but not process death); **`SavedStateHandle`/onSaveInstanceState** for small state that must survive process death; **disk/DB (Room/DataStore)** as the source of truth for anything that must truly persist.

---



## 19. Lifecycle sequence across multiple activities (A → B → C transparent)

**Lifecycle coordination** means the deterministic sequence of visibility and focus transitions that occur across activities during navigation.

Key insight: a **transparent/non-fullscreen** activity pauses but does NOT stop the activity beneath it. Opening opaque B over A stops A; opening transparent C over B only pauses B (no `onStop`). Backing out resumes a paused activity directly, while a stopped activity goes through `onRestart` → `onStart` → `onResume`.

---



## 20. What are launch modes?

**Launch modes** means instructions in the manifest or intent flags that define how a new activity instance is instantiated and integrated into a task's back stack.

Control how instances are created/placed in the task:
- **standard** — new instance every time.
- **singleTop** — reuse via `onNewIntent()` if already at top.
- **singleTask** — one instance as task root; reuses and clears activities above.
- **singleInstance** — like singleTask but the only activity in its task.

Equivalent intent flags exist (`FLAG_ACTIVITY_NEW_TASK`, `CLEAR_TOP`, `SINGLE_TOP`).

---



## 21. What is a `Fragment` and its lifecycle?

**Fragment** means a modular, reusable user interface component that lives within a host activity and manages its own lifecycle distinct from the activity's instance.

A reusable UI module hosted inside an activity, with its own lifecycle tied to the host. Callbacks: `onAttach` → `onCreate` → `onCreateView` → `onViewCreated` → `onStart` → `onResume` … `onPause` → `onStop` → `onDestroyView` → `onDestroy` → `onDetach`. The **view lifecycle is shorter** than the fragment's — clear view/binding refs in `onDestroyView` and observe with `viewLifecycleOwner`.

---



## 22. `Fragment` vs `Activity` — relationship and when to use each

**Activity** means a top-level entry point that provides a window and a base context, whereas **Fragment** is a UI sub-component hosted inside an activity for flexible, modular screen designs.

An Activity is a top-level, manifest-declared component with a window; a Fragment is a sub-component hosted inside an activity that can't exist alone. Use fragments for reusable UI, multi-pane layouts, and single-activity + Navigation architecture. Use an activity for a genuine separate entry point or external implicit-intent target.

---



## 23. Why use only the default constructor for a `Fragment`?

**No-argument constructor** means the default constructor that the Android system uses to recreate a fragment via reflection, requiring all initial parameters to be passed via a persistent `Bundle` argument instead.

The framework recreates fragments via reflection using the **no-arg constructor** (after config change/process death), so a custom constructor's data is lost or crashes. Pass data via a `Bundle` set with `setArguments()` (often a `newInstance` factory) — arguments are automatically saved and restored.

---



## 24. `add` vs `replace` and `addToBackStack()`

**add()** means stacking a fragment on top of the container without disturbing existing fragments, **replace()** means removing all existing fragments from the container before adding the new one, and **addToBackStack()** preserves the transaction to allow back-button navigation.

`add` puts a fragment on top while existing ones stay attached (no teardown callbacks). `replace` removes current fragments first (outgoing fragment runs `onPause`/`onStop`/`onDestroyView`). `addToBackStack(name)` records the transaction so Back reverses it; without it a `replace` is irreversible and the prior fragment is lost.

---



## 25. `FragmentPagerAdapter` vs `FragmentStatePagerAdapter` (and ViewPager2)

**ViewPager adapters** means legacy classes that manage fragment pages: `FragmentPagerAdapter` keeps all instances in memory, `FragmentStatePagerAdapter` destroys instances to save memory, and **ViewPager2** is their modern replacement built on `RecyclerView`.

`FragmentPagerAdapter` keeps all visited fragment instances in memory (good for few pages); `FragmentStatePagerAdapter` destroys off-screen fragments and saves only their state (good for many pages). Both are deprecated — use **`ViewPager2` + `FragmentStateAdapter`**, which is RecyclerView-backed and behaves like the state adapter.

---



## 26. What is a retained `Fragment`?

**Retained fragment** means a legacy fragment that had `setRetainInstance(true)` enabled so its instance survived configuration changes, which is now deprecated in favour of `ViewModel`.

A fragment whose instance survived config changes via `setRetainInstance(true)` (only the view was recreated), historically used to hold expensive objects across rotation. It's now **deprecated** — use a **`ViewModel`** instead, which is purpose-built for surviving config changes without the pitfalls.

---



## 27. How to communicate between two Fragments?

**Fragment communication** means transferring data between fragments using a shared, activity-scoped `ViewModel` or the dedicated Fragment Result API to maintain loose coupling.

Never directly. Preferred: a **shared `ViewModel`** scoped to the host activity (`activityViewModels()`) exchanging data via LiveData/StateFlow. For one-off results, the **Fragment Result API** (`setFragmentResult`/`setFragmentResultListener`). Older pattern: via the host activity through an interface (verbose, coupled).

---



## 28. What is a `Bundle`? Size limits

**Bundle** means a key-value mapping class designed to pass parcelable data across IPC boundaries, limited to a total Binder transaction buffer of approximately 1 MB.

A key-value map (backed by `Parcelable`) for passing data between components and across the process boundary — used in intent extras, fragment arguments, and saved state. It travels via a Binder transaction with a ~1 MB buffer per process; exceeding it throws `TransactionTooLargeException`. Never put bitmaps/large lists in it — pass an id/URI instead.

---



## 29. Transferring objects between activities — Serializable vs Parcelable

**Parcelable** means an Android-specific serialisation interface optimised for fast IPC without reflection, whereas **Serializable** is a standard Java interface that relies on slower reflection-based serialisation.

Put data in intent extras. **Serializable** is a Java marker interface using reflection — easy but slow with GC pressure. **Parcelable** is Android-specific, reflection-free, faster, and recommended (use `@Parcelize` to generate boilerplate). Remember the Bundle size limit — pass ids, not large payloads.

---



## 30. `Dialog` vs `DialogFragment`

**DialogFragment** means a fragment wrapper around a dialog that integrates with the `FragmentManager` to survive configuration changes, whereas **Dialog** is a basic floating window that lacks lifecycle awareness.

`Dialog` is a basic floating window, not lifecycle-aware — it dismisses/leaks on rotation. `DialogFragment` wraps a dialog in the fragment lifecycle/`FragmentManager`, so it survives config changes and manages its own state. Always prefer `DialogFragment` for persistent dialogs.

---



## Intents & Broadcasting

## 31. What is an `Intent`? Explicit vs Implicit

**Intent** means a messaging object used to request an action from another component, where **explicit intent** specifies the target component class and **implicit intent** declares a generic action to perform.

An asynchronous messaging object to start an Activity/Service or send a broadcast, carrying action, data, and extras. **Explicit** names the exact target component (in-app navigation). **Implicit** declares an action and lets the system find an app whose intent filter handles it (e.g. open URL, share). Android 11+ may need `<queries>` for package visibility.

---



## 32. What is a `BroadcastReceiver`? Types of broadcasts

**BroadcastReceiver** means a component that listens for and responds to system-wide or application-specific event notifications.

A component reacting to system/app broadcast events via `onReceive` (runs on main thread — keep it quick). Registration is **manifest (static)** — restricted since Android 8 — or **runtime (dynamic)**. Broadcast types: **normal** (async, unordered), **ordered** (priority, abortable), **local** (deprecated), **sticky** (deprecated), and **system** broadcasts.

---



## 33. How broadcasts pass messages around your app

**Broadcast messaging** means a publish-subscribe communication mechanism where publishers send intents and registered receivers consume them via `onReceive()`.

A publish/subscribe mechanism on Intents: a sender builds an Intent and calls `sendBroadcast`; the system matches it to registered receivers (via IntentFilter or explicit target); each receiver's `onReceive` reacts. Lets components communicate without direct references. For in-app messaging today, prefer a shared `ViewModel`/`StateFlow`; reserve broadcasts for cross-app/system events.

---



## 34. What is a `PendingIntent`? (and Sticky Intent)

**PendingIntent** means a token that wraps an intent and grants another application or the system permission to execute it later using the sender's identity and privileges.

A token wrapping an Intent plus permission to execute it, handed to another app/system to fire later on your behalf with your app's identity — even if your process is dead. Used by notifications, alarms, and widgets (`getActivity`/`getService`/`getBroadcast`). Since Android 12 you must set `FLAG_IMMUTABLE` or `FLAG_MUTABLE`. A **Sticky Intent** (deprecated) was a broadcast whose last value stuck for future receivers.

---



## Services

## 35. What is a `Service`? Its lifecycle

**Service** means an application component that performs long-running background tasks without displaying a user interface.

A component for long-running work without a UI; by default it runs on the main thread. **Started** (`startService`): `onCreate` → `onStartCommand` → runs until `stopSelf`/`stopService` → `onDestroy`. **Bound** (`bindService`): `onCreate` → `onBind` → clients interact → `onUnbind` → `onDestroy`. Can be both. For modern background work prefer WorkManager or a foreground service.

---



## 36. On which thread does a `Service` run?

**Service thread execution** means that services run on the application's main thread by default, requiring background threads or coroutines to perform blocking work.

By default the application's **main (UI) thread** — not a separate thread or process. Blocking work in `onStartCommand`/`onBind` can cause an ANR, so spawn your own background thread/coroutine. (`IntentService`, now deprecated, created its own worker thread; `android:process` runs it in another process but still on that process's main thread.)

---



## 37. `Service` vs `IntentService`

**Service vs IntentService** means the distinction between a standard Service running on the main thread, and the deprecated IntentService which ran tasks sequentially on a worker thread.

`Service` runs on the main thread, handles multiple/long-lived tasks, can be started and/or bound, and you manage threading and stopping. `IntentService` (a Service subclass) had its own worker thread, processed intents sequentially in `onHandleIntent`, and auto-stopped when done — for short fire-and-forget tasks. `IntentService` is deprecated; use WorkManager or a coroutine instead.

---



## 38. What is a Foreground Service?

**Foreground service** means a high-priority service that performs work visible to the user and must display a persistent notification.

A service for work the user is actively aware of (playback, navigation, uploads) that shows a persistent notification and gets higher priority, exempt from background limits. Call `startForeground` within ~5s, hold `FOREGROUND_SERVICE` permission, and on Android 14 declare a `foregroundServiceType` plus its matching per-type permission.

---



## 39. What is `JobScheduler`?

**JobScheduler** means a system service that schedules background jobs to run under specific system constraints, batching tasks to optimise battery consumption.

A system service (API 21+) for scheduling deferrable background jobs under constraints (unmetered network, charging, idle, deadline); the OS batches jobs across apps to save battery. You define a `JobService` and submit a `JobInfo`. For most apps **WorkManager** (built on top of it) is recommended instead.

---



## 40. How does `WorkManager` guarantee task execution?

**WorkManager** means a persistent Jetpack background library that schedules deferrable, guaranteed tasks and stores their states in a local SQLite database to survive app restarts and reboots.

It persists every `WorkRequest` to an internal Room/SQLite database so work survives app kill and reboot (rescheduled via a boot receiver). It delegates to the best OS scheduler (`JobScheduler` on API 23+), supports constraints and retry with backoff. The guarantee is **eventual** execution once constraints are met — not exact-time or instant.

---



## 41. What can you use for background processing in Android?

**Background processing** means a strategy where you use Kotlin coroutines for short-lived in-process tasks, `WorkManager` for guaranteed deferrable tasks, and foreground services for immediate user-visible tasks.

- **Coroutines/`Dispatchers.IO`** — in-process work while the app/screen is alive.
- **WorkManager** — deferrable, guaranteed work surviving exit/reboot.
- **Foreground Service** — immediate, user-visible ongoing work.
- **AlarmManager** — exact/inexact time-based triggers.
- Avoid: `AsyncTask`, `IntentService`, `LocalBroadcastManager`.

---



## Inter-Process Communication

## 42. How can two distinct Android apps interact?

**App interaction** means communication between separate applications using implicit intents, content providers, shared databases, or bound services over AIDL.

Via **implicit Intents + intent filters** (share/open/deep link), **ContentProvider + ContentResolver** (structured data), **bound service with AIDL/Messenger** (programmatic IPC), **broadcasts**, and **FileProvider** (secure file sharing). Android 11+ package visibility may require `<queries>`; protect surfaces with permissions and `android:exported`.

---



## 43. Can an app run in multiple processes? How?

**Multi-process app** means an application configured to run separate components in isolated process spaces using the `android:process` attribute in the manifest.

Yes — set `android:process` on a component in the manifest. A name starting with `:` creates a private process; a full lowercase name creates a global (shareable) one. Useful to isolate crash-prone/memory-heavy work. Caveats: each process gets its own VM and `Application` instance, separate memory (no shared singletons/statics), and must communicate via IPC.

---



## 44. What is AIDL? Steps to create a bound service with AIDL

**AIDL (Android Interface Definition Language)** means a compiler-generated interface that allows processes to interact via Binder-based inter-process communication.

AIDL (Android Interface Definition Language) defines a programmatic interface for cross-process IPC; it generates Binder marshalling (proxy/stub) code. Steps: (1) define the `.aidl` interface; (2) implement the generated `Stub` in a `Service` and return it from `onBind`; (3) declare the service in the manifest; (4) client binds, calls `Stub.asInterface(binder)`, then invokes methods. Calls may run on a Binder thread pool, so be thread-safe.

---



## 45. What is a `ContentProvider` and when is it used?

**ContentProvider** means a component that encapsulates structured data access and exposes it securely to other applications using content URIs.

A component that exposes data via a URI-addressed interface (`content://`) accessed through a `ContentResolver`, abstracting storage behind `query`/`insert`/`update`/`delete`. Its main purpose is **securely sharing data with other apps** and integrating with framework features (Contacts, MediaStore, FileProvider). For internal-only data, use Room/DataStore directly — a ContentProvider is boilerplate-heavy overkill.

