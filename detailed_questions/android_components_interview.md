# Android Core Components & Lifecycle — Interview Guide

A comprehensive, interview-ready guide to Android's core building blocks: the
application model and `Context`, the `Activity` and `Fragment` lifecycles,
`Intent`s and broadcasting, `Service`s and background work, and inter-process
communication. Answers merge two question banks, are written out in full with
Kotlin snippets and trade-offs, and are accurate as of 2026 (Android 15 / API 35,
targeting practices for API 34+). Where an older API has been superseded
(e.g. `FragmentPagerAdapter` → `ViewPager2`, `IntentService`/`JobScheduler` →
`WorkManager`), the modern alternative is explained.

---

## Table of Contents

### Base / Fundamentals
1. [Why does an Android app lag?](#1-why-does-an-android-app-lag)
2. [What is `Context`? Application vs Activity Context](#2-what-is-context-application-vs-activity-context)
3. [How does Zygote make Android apps start faster?](#3-how-does-zygote-make-android-apps-start-faster)
4. [What are the Android application components?](#4-what-are-the-android-application-components)
5. [What is the project structure of an Android app?](#5-what-is-the-project-structure-of-an-android-app)
6. [What is `AndroidManifest.xml`?](#6-what-is-androidmanifestxml)
7. [What is the `Application` class?](#7-what-is-the-application-class)
8. [ART vs Dalvik](#8-art-vs-dalvik)
9. [What is ANR (Application Not Responding)?](#9-what-is-anr-application-not-responding)
10. [What is ADB (Android Debug Bridge)?](#10-what-is-adb-android-debug-bridge)
11. [Process vs Thread vs Task](#11-process-vs-thread-vs-task)
12. [Looper, Handler, and MessageQueue](#12-looper-handler-and-messagequeue)

### Activity & Fragment
13. [What is an `Activity` and its lifecycle?](#13-what-is-an-activity-and-its-lifecycle)
14. [Difference between `onCreate()` and `onStart()`](#14-difference-between-oncreate-and-onstart)
15. [Why call `setContentView()` in `onCreate()`?](#15-why-call-setcontentview-in-oncreate)
16. [When is only `onDestroy()` called without `onPause()`/`onStop()`?](#16-when-is-only-ondestroy-called-without-onpauseonstop)
17. [`onSaveInstanceState()` and `onRestoreInstanceState()`](#17-onsaveinstancestate-and-onrestoreinstancestate)
18. [Preventing data loss on rotation (ViewModel + saved state)](#18-preventing-data-loss-on-rotation-viewmodel--saved-state)
19. [Lifecycle sequence across multiple activities (A → B → C transparent)](#19-lifecycle-sequence-across-multiple-activities-a--b--c-transparent)
20. [What are launch modes?](#20-what-are-launch-modes)
21. [What is a `Fragment` and its lifecycle?](#21-what-is-a-fragment-and-its-lifecycle)
22. [`Fragment` vs `Activity` — relationship and when to use each](#22-fragment-vs-activity--relationship-and-when-to-use-each)
23. [Why use only the default constructor for a `Fragment`?](#23-why-use-only-the-default-constructor-for-a-fragment)
24. [`add` vs `replace` and `addToBackStack()`](#24-add-vs-replace-and-addtobackstack)
25. [`FragmentPagerAdapter` vs `FragmentStatePagerAdapter` (and ViewPager2)](#25-fragmentpageradapter-vs-fragmentstatepageradapter-and-viewpager2)
26. [What is a retained `Fragment`?](#26-what-is-a-retained-fragment)
27. [How to communicate between two Fragments?](#27-how-to-communicate-between-two-fragments)
28. [What is a `Bundle`? Size limits](#28-what-is-a-bundle-size-limits)
29. [Transferring objects between activities — Serializable vs Parcelable](#29-transferring-objects-between-activities--serializable-vs-parcelable)
30. [`Dialog` vs `DialogFragment`](#30-dialog-vs-dialogfragment)

### Intents & Broadcasting
31. [What is an `Intent`? Explicit vs Implicit](#31-what-is-an-intent-explicit-vs-implicit)
32. [What is a `BroadcastReceiver`? Types of broadcasts](#32-what-is-a-broadcastreceiver-types-of-broadcasts)
33. [How broadcasts pass messages around your app](#33-how-broadcasts-pass-messages-around-your-app)
34. [What is a `PendingIntent`? (and Sticky Intent)](#34-what-is-a-pendingintent-and-sticky-intent)

### Services
35. [What is a `Service`? Its lifecycle](#35-what-is-a-service-its-lifecycle)
36. [On which thread does a `Service` run?](#36-on-which-thread-does-a-service-run)
37. [`Service` vs `IntentService`](#37-service-vs-intentservice)
38. [What is a Foreground Service?](#38-what-is-a-foreground-service)
39. [What is `JobScheduler`?](#39-what-is-jobscheduler)
40. [How does `WorkManager` guarantee task execution?](#40-how-does-workmanager-guarantee-task-execution)
41. [What can you use for background processing in Android?](#41-what-can-you-use-for-background-processing-in-android)

### Inter-Process Communication
42. [How can two distinct Android apps interact?](#42-how-can-two-distinct-android-apps-interact)
43. [Can an app run in multiple processes? How?](#43-can-an-app-run-in-multiple-processes-how)
44. [What is AIDL? Steps to create a bound service with AIDL](#44-what-is-aidl-steps-to-create-a-bound-service-with-aidl)
45. [What is a `ContentProvider` and when is it used?](#45-what-is-a-contentprovider-and-when-is-it-used)

---

## Base / Fundamentals

## 1. Why does an Android app lag?

An app "lags" when it fails to render a frame within the budget the display
imposes. On a 60 Hz screen you have ~16.6 ms per frame (8.3 ms at 120 Hz). If the
main (UI) thread is busy past that window, the frame is **janky** (skipped or
late), and the user perceives stutter.

Common causes:

- **Heavy work on the main thread** — network calls, disk/DB I/O, JSON parsing,
  bitmap decoding, or large `RecyclerView` binds running on the UI thread.
- **Overdraw and deep view hierarchies** — the same pixel painted many times, or
  nested layouts forcing repeated measure/layout passes. Flatten with
  `ConstraintLayout`.
- **Garbage-collection pressure** — excessive object allocation (especially in
  `onDraw()` / `onBindViewHolder()`) triggers frequent GC pauses.
- **Inefficient lists** — no view recycling, no `DiffUtil`, expensive work in
  `onBindViewHolder()`.
- **Memory leaks** — leaked activities/bitmaps shrink the available heap and
  increase GC.
- **Layout/measure thrash**, unoptimized images, and synchronous animations.

How to diagnose: Android Studio Profiler, Systrace/Perfetto, `Choreographer`
frame callbacks, StrictMode (to catch disk/network on the main thread), and
LeakCanary for leaks.

The fix is almost always: **move work off the main thread** (coroutines/`Dispatchers.IO`),
**reduce allocations and overdraw**, and **keep the view hierarchy shallow**.

**📚 Reference:** https://outcomeschool.com/blog/android-app-lag

---

## 2. What is `Context`? Application vs Activity Context

`Context` is an abstract handle to the Android system and to your app's
environment. It provides access to resources (`getString`, `getDrawable`),
assets, system services (`getSystemService`), preferences, databases, file
directories, and the ability to start activities, bind services, and send
broadcasts. Almost every Android API needs a `Context`.

Two contexts you must distinguish:

- **Application Context** (`applicationContext`): tied to the lifecycle of the
  **whole app process**. It outlives any single activity. Use it for things that
  must outlive a screen — singletons, a long-lived database/Retrofit instance, or
  anything stored in a static/global field. Using it cannot leak an activity.
- **Activity Context** (`this` in an `Activity`): tied to the **activity's
  lifecycle** and carries the activity's themed resources. Use it for UI work —
  inflating layouts, creating dialogs, starting another activity — where the
  result must respect the current theme/window.

Rule of thumb:

```kotlin
// Long-lived singleton -> application context (no leak)
val db = Room.databaseBuilder(context.applicationContext, AppDatabase::class.java, "app").build()

// UI that needs the activity theme -> activity context
AlertDialog.Builder(this).setTitle("Hi").show()
```

Pitfall: holding an `Activity` context in a static field, a `companion object`,
or a long-lived singleton leaks the entire activity (and its view tree). Prefer
`applicationContext` for anything that lives beyond the screen.

**📚 Reference:** https://outcomeschool.com/blog/context-in-android-application

---

## 3. How does Zygote make Android apps start faster?

**Zygote** is a special process started by `init` at boot. It preloads and
initializes the core Android framework classes and shared resources (the ART
runtime, common system classes, drawables, etc.) **once**.

When you launch an app, Android does **not** start a fresh VM from scratch.
Instead, Zygote **forks** itself. Thanks to the OS copy-on-write (COW) memory
model, the new process initially shares Zygote's already-warm memory pages —
the preloaded classes and resources are not re-loaded or re-initialized. Pages
are only copied when the child actually writes to them.

Benefits:

- **Faster startup** — no re-loading/JIT-warming of framework classes per app.
- **Lower memory use** — shared, read-only framework pages are common across all
  app processes instead of duplicated.

So every app process is a child of Zygote that inherits a ready-to-run runtime,
which is the core reason cold starts are not catastrophically slow.

**📚 Reference:** https://outcomeschool.substack.com/p/how-zygote-makes-android-apps-start

---

## 4. What are the Android application components?

App components are the essential building blocks and entry points of an app:

- **Activity** — a single screen with a UI; handles user interaction.
- **Service** — runs long-running work in the background with no UI (e.g. music
  playback, syncing).
- **BroadcastReceiver** — responds to system-wide or app broadcast events
  (e.g. connectivity change, boot completed).
- **ContentProvider** — manages a shared set of app data and exposes it to other
  apps through a `ContentResolver` interface.

Supporting pieces:

- **Intent** — the asynchronous message used to activate Activities, Services,
  and (for some) BroadcastReceivers, and to pass data between components.

Activities, Services, and (manifest-registered) Receivers, plus ContentProviders,
must be declared in `AndroidManifest.xml`.

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineer-tech-activity-7302543942028251137-S3vJ

---

## 5. What is the project structure of an Android app?

A typical Gradle-based Android project:

```text
ProjectRoot/
├── app/                          # an application module
│   ├── build.gradle(.kts)        # module config: SDK versions, deps, build types
│   ├── proguard-rules.pro        # R8/ProGuard keep rules
│   └── src/
│       ├── main/
│       │   ├── AndroidManifest.xml
│       │   ├── java|kotlin/...   # source code
│       │   └── res/              # resources
│       │       ├── drawable/     # images, vector/shape XML
│       │       ├── layout/       # XML UI layouts
│       │       ├── mipmap/       # launcher icons
│       │       ├── values/       # strings.xml, colors.xml, themes/styles
│       │       └── ...
│       ├── test/                 # local JVM unit tests
│       └── androidTest/          # instrumented (on-device) tests
├── build.gradle(.kts)            # top-level build config
├── settings.gradle(.kts)         # included modules
├── gradle.properties             # global Gradle/JVM flags
└── gradle/                       # wrapper + version catalogs (libs.versions.toml)
```

- **`src/main`** holds production code, manifest, and resources; build-variant
  folders (e.g. `src/debug`, `src/free`) can override them.
- **Resources** in `res/` are referenced via the generated `R` class
  (`R.string.app_name`).
- **Modules** keep code modular; multi-module apps add `:core`, `:feature_x`
  library modules.

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_android-app-project-structure-activity-7302902092812226560-bM8D

---

## 6. What is `AndroidManifest.xml`?

The manifest is the app's mandatory descriptor, read by the OS and Play Store
**before** any code runs. It declares:

- The **package name** / application id (identity).
- All **components** — `<activity>`, `<service>`, `<receiver>`, `<provider>` —
  the system needs every component declared here to be able to start it.
- **Permissions** the app requests (`<uses-permission>`) and permissions it
  defines.
- **Intent filters** that advertise what implicit intents a component can handle
  (e.g. the launcher activity uses `MAIN` + `LAUNCHER`).
- **Hardware/software requirements** (`<uses-feature>`), min/target SDK
  (usually now in Gradle), the `<application>` element (theme, icon, the custom
  `Application` class, `android:exported`, etc.).

```xml
<application android:name=".MyApp" android:theme="@style/AppTheme">
    <activity android:name=".MainActivity" android:exported="true">
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />
            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>
    <service android:name=".SyncService" android:foregroundServiceType="dataSync" />
</application>
```

Note: since Android 12 (API 31), any component with an intent filter must set
`android:exported` explicitly.

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineer-tech-activity-7305255303908888576-c0Nf

---

## 7. What is the `Application` class?

`Application` is the base class representing the whole app process. A single
instance is created when the process starts and lives as long as the process
does — making it the natural place for **process-wide initialization** and
global state.

```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        // Runs once, before any Activity/Service. Keep it fast (affects cold start).
        Timber.plant(Timber.DebugTree())
        // init DI graph, crash reporting, etc.
    }
}
```

Register it via `android:name=".MyApp"` in the manifest's `<application>` tag.

Key points:

- `getApplicationContext()` returns this instance — a safe long-lived `Context`.
- Useful callbacks: `onCreate()`, `onLowMemory()`, `onTrimMemory()`,
  `onConfigurationChanged()`, and `registerActivityLifecycleCallbacks()`.
- **Do not** do heavy/blocking work in `onCreate()` — it directly delays cold
  start. Prefer App Startup library or lazy init.
- It is **not** a substitute for a DI container; avoid stuffing mutable global
  state here.

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineer-tech-activity-7304864185010528256-2GGU

---

## 8. ART vs Dalvik

Both are execution environments that run Android's DEX (Dalvik Executable)
bytecode, but differ in how they compile it:

- **Dalvik** (default up to Android 4.4): used a **Just-In-Time (JIT)** compiler,
  compiling bytecode to native code at runtime as needed. Small install
  footprint, faster install, but slower execution and more runtime jank.
- **ART** (experimental in 4.4, default since 5.0): originally used
  **Ahead-Of-Time (AOT)** compilation at install time, producing native code for
  faster execution and startup, at the cost of more storage and longer installs.
  Modern ART uses a **hybrid**: interpret + JIT first, with **profile-guided AOT**
  compilation done in the background (e.g. while charging) for hot code paths —
  combining fast installs with fast steady-state performance. ART also has
  improved garbage collection.

Trade-off summary: Dalvik = smaller/faster install, slower run; ART = faster run
& better GC, more storage. Apps written for Dalvik generally run on ART.

---

## 9. What is ANR (Application Not Responding)?

An **ANR** is the system dialog shown when the app's main thread is blocked too
long, signaling the UI is frozen. Triggers:

- **Input dispatch timeout** — no response to an input event within ~5 seconds.
- **`BroadcastReceiver`** — `onReceive()` not finishing within ~10s (foreground)
  / ~60s (background).
- **`Service`** lifecycle callbacks not completing in time (~20s foreground).

Cause: long/blocking work on the main thread (network, disk, DB, heavy
computation, deadlocks). Fix: move work to background threads/coroutines, never
block the UI thread, use `StrictMode` to detect violations, and inspect
`/data/anr/traces.txt` (or Play Console ANR reports) for the stack.

---

## 10. What is ADB (Android Debug Bridge)?

ADB is a command-line tool and client-server protocol for communicating with a
device/emulator. It is used to install/uninstall apps, push/pull files, read
logs, run a shell on the device, and debug. Common commands:

```bash
adb devices                       # list connected devices
adb install app.apk               # install an APK
adb logcat                        # stream device logs
adb shell                         # open a shell on the device
adb shell am start -n pkg/.Activity
adb push local /sdcard/remote     # copy file to device
```

It consists of a **client** (your machine), a **daemon (adbd)** running on the
device, and a **server** process mediating between them.

---

## 11. Process vs Thread vs Task

- **Process** — an OS-level execution context. Each Android app normally runs in
  its **own process** with its **own VM instance** and isolated memory. The
  system can kill processes to reclaim resources. Components can be split into
  multiple processes via `android:process` in the manifest.
- **Thread** — the smallest unit of execution **within** a process. Threads in a
  process share memory. The **main/UI thread** handles UI and event dispatch;
  additional threads handle background work.
- **Task** — a **user-facing** concept: an ordered stack (back stack) of
  activities the user interacts with as they move through the app(s). A task can
  span multiple apps/processes and is governed by launch modes and task affinity.

In short: process = resource/isolation boundary; thread = unit of execution;
task = the user's navigation journey through activities.

---

## 12. Looper, Handler, and MessageQueue

These three implement Android's per-thread message loop:

- **MessageQueue** — a queue of `Message`/`Runnable` items to be processed,
  ordered by dispatch time.
- **Looper** — turns a normal thread into a looping thread: it owns the
  `MessageQueue` and continuously pulls items off it and dispatches them. The
  main thread has a Looper set up by the framework (`Looper.getMainLooper()`).
- **Handler** — bound to a specific `Looper`; you use it to **post** `Runnable`s
  or **send** `Message`s onto that looper's queue, optionally with a delay. This
  is how you schedule work to run on (and marshal results back to) a particular
  thread.

```kotlin
val mainHandler = Handler(Looper.getMainLooper())
mainHandler.post { textView.text = "Updated on UI thread" }   // run on main thread
```

To make your own thread loop, use `HandlerThread` (it creates a Looper for you).
In modern code, coroutines/`Dispatchers.Main` largely replace manual Handler use.

---

## Activity & Fragment

## 13. What is an `Activity` and its lifecycle?

An **Activity** is a single, focused screen the user interacts with. The system
drives it through lifecycle callbacks as it becomes visible, gains/loses focus,
and is destroyed. Override these to acquire/release resources at the right time.

Lifecycle callbacks (in order for a typical launch → exit):

1. **`onCreate(savedInstanceState)`** — created. Inflate UI (`setContentView`),
   initialize, restore state. Called once per instance.
2. **`onStart()`** — activity becomes **visible** (but not yet interactive).
3. **`onResume()`** — activity is in the **foreground** and **interactive**; at
   the top of the stack. (App is now in the "resumed/running" state.)
4. **`onPause()`** — losing focus (another activity comes in front, e.g. a
   dialog or transparent activity). Still partially visible. Keep it quick — the
   next activity won't resume until this returns.
5. **`onStop()`** — no longer visible (fully obscured or backgrounded). Release
   heavier resources here.
6. **`onRestart()`** — called before `onStart()` when coming back from stopped.
7. **`onDestroy()`** — being destroyed (user finished it, or system reclaimed it,
   or config change). Final cleanup.

Described as a diagram:

```text
onCreate -> onStart -> onResume -> [RUNNING]
                                      |
            (another activity/dialog) v
                                  onPause -> onStop -> onDestroy
                                     |          |
                          (back to front)  onRestart -> onStart -> onResume
```

**📚 Reference:** https://developer.android.com/guide/components/activities/activity-lifecycle

---

## 14. Difference between `onCreate()` and `onStart()`

- **`onCreate()`** is called **once** when the activity instance is first
  created. It's where one-time setup happens: `setContentView()`, view binding,
  reading the `savedInstanceState` bundle, wiring ViewModels. The activity is
  **not yet visible**.
- **`onStart()`** is called **every time** the activity becomes **visible** to
  the user — both after `onCreate()` on first launch and after `onRestart()` when
  the user returns from a stopped state. It runs more than once over the
  activity's life and is where you (re)start visible-only work (e.g. register a
  location listener you stop in `onStop()`).

Mnemonic: `onCreate` = "set up once"; `onStart` = "now visible, possibly again".

**📚 Reference:** https://developer.android.com/guide/components/activities/activity-lifecycle

---

## 15. Why call `setContentView()` in `onCreate()`?

`setContentView()` inflates the XML layout and attaches the resulting view
hierarchy to the activity's window so the UI exists and can be referenced (e.g.
`findViewById`). It belongs in `onCreate()` because:

- `onCreate()` runs **once** per instance — you want to build the view tree once,
  not on every visibility change. Putting it in `onStart()`/`onResume()` would
  re-inflate repeatedly, wasting work and discarding view state.
- The view tree must exist before later callbacks and before the user sees the
  screen; `onCreate()` is the earliest, one-time point where you also have the
  `savedInstanceState` to restore into those views.

In Jetpack Compose, `setContent { }` plays the analogous role and is likewise
called in `onCreate()`.

**📚 Reference:** https://www.youtube.com/watch?v=U1aHAt7XC5I

---

## 16. When is only `onDestroy()` called without `onPause()`/`onStop()`?

If you call **`finish()` inside `onCreate()`** (before the activity ever becomes
visible/resumed), the activity never reaches `onStart()`/`onResume()`, so it
also never goes through `onPause()`/`onStop()` — the system goes straight from
`onCreate()` to **`onDestroy()`**.

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    if (!userLoggedIn) {
        startActivity(Intent(this, LoginActivity::class.java))
        finish()           // -> onDestroy() will be the next callback, no onPause/onStop
        return
    }
    setContentView(R.layout.activity_main)
}
```

This is common for "router"/splash activities that decide where to go and finish
immediately. (Note: `onPause`/`onStop` are skipped precisely because the activity
was never started/resumed.)

**📚 Reference:** https://www.youtube.com/watch?v=B2kY_ckZa-g

---

## 17. `onSaveInstanceState()` and `onRestoreInstanceState()`

These handle **transient UI state** across system-initiated destruction (config
change like rotation, or process death while in the background).

- **`onSaveInstanceState(outState: Bundle)`** — called before the activity may be
  destroyed (typically after `onStop` on modern versions). Write small UI state
  into the `Bundle`.
- **`onRestoreInstanceState(savedInstanceState: Bundle)`** — called after
  `onStart()` when the activity is recreated, giving back the same `Bundle`.

The same `Bundle` is also passed to `onCreate(savedInstanceState)`. Because
`onCreate()` runs on both fresh creation and recreation, **always null-check**
the bundle: `null` means a brand-new instance.

```kotlin
override fun onSaveInstanceState(outState: Bundle) {
    super.onSaveInstanceState(outState)
    outState.putString(KEY_QUERY, searchView.query.toString())
}

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)
    savedInstanceState?.getString(KEY_QUERY)?.let { restoreQuery(it) }
}
```

Limits & trade-offs: the bundle must stay **small** (it's parcelled across a
Binder; large data risks `TransactionTooLargeException`). It is **not** called on
explicit user finish (back press, `finish()`). For larger or longer-lived data,
use a `ViewModel` (survives config change but **not** process death) and persist
real data to disk/DB. `ViewModel` + `SavedStateHandle` covers both.

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineer-tech-activity-7301985158608392193-pA5M

---

## 18. Preventing data loss on rotation (ViewModel + saved state)

On rotation (and other config changes) the activity is **destroyed and
recreated** by default. Use a layered strategy:

1. **`ViewModel`** for in-memory UI data. A `ViewModel` is lifecycle-aware and is
   **not** destroyed on configuration change — the recreated activity reconnects
   to the same instance. Rotate three times and you have three activity instances
   but one `ViewModel`. Store fetched lists, computed state, etc. here.
2. **`onSaveInstanceState()` / `SavedStateHandle`** for small transient UI state
   (a search query, scroll position) that must also survive **process death**
   (which a plain `ViewModel` does not).
3. **Disk/DB (Room, DataStore)** as the source of truth for anything that must
   truly persist.

```kotlin
class SearchViewModel(private val state: SavedStateHandle) : ViewModel() {
    val results = MutableLiveData<List<Item>>()              // survives rotation
    var query: String
        get() = state["query"] ?: ""
        set(v) { state["query"] = v }                        // survives process death
}
```

Example: keep the search query in `SavedStateHandle`, and the resulting list in
the `ViewModel`. (Alternatively, `android:configChanges` lets you handle a config
change yourself without recreation, but it's discouraged for orientation as it
bypasses resource re-selection.)

---

## 19. Lifecycle sequence across multiple activities (A → B → C transparent)

Activity C is **transparent** (it does not fully cover B). Sequence:

1. **Go A → B** (B opaque): `A.onPause` → `B.onCreate` → `B.onStart` →
   `B.onResume` → `A.onStop` (A is fully covered, so it stops).
2. **Go B → C** (C transparent/partially visible): `B.onPause` → `C.onCreate` →
   `C.onStart` → `C.onResume`. **B does NOT get `onStop`** because it is still
   partially visible behind the transparent C.
3. **Rotate the phone** (C on top): the visible/recreating activities go through
   destroy→create. C: `C.onPause` → `C.onSaveInstanceState` → `C.onStop` →
   `C.onDestroy` → `C.onCreate` → `C.onStart` → `C.onRestoreInstanceState` →
   `C.onResume`. B (visible behind, depending on OEM/version) typically also
   recreates similarly. (A is stopped; it recreates when next shown.)
4. **Press back to B** (finish C): `C.onPause` → `B.onResume` → `C.onStop` →
   `C.onDestroy`. (B was only paused, so it just resumes — no `onCreate`.)
5. **Press back to A** (finish B): `B.onPause` → `A.onRestart` → `A.onStart` →
   `A.onResume` → `B.onStop` → `B.onDestroy`. (A was stopped, so it restarts.)

Key insight tested here: a transparent/non-fullscreen activity pauses but does
**not** stop the activity beneath it.

---

## 20. What are launch modes?

`launchMode` (in the manifest) and intent flags control how an activity instance
is created and placed in the back stack/task.

- **`standard`** (default): always creates a **new instance** in the caller's
  task. Multiple instances can coexist (even of the same activity).
- **`singleTop`**: if an instance is **already at the top** of the task, the
  intent is delivered to it via **`onNewIntent()`** instead of creating a new
  one. If it's not at the top, a new instance is created.
- **`singleTask`**: at most **one instance** exists, as the **root** of its task.
  A new intent re-uses it via `onNewIntent()` and **clears activities above it**.
  Good for an app's main entry/home.
- **`singleInstance`**: like `singleTask`, but the activity is the **only** member
  of its task — no other activities are placed in that task.

Equivalent intent flags: `FLAG_ACTIVITY_NEW_TASK`, `FLAG_ACTIVITY_CLEAR_TOP`,
`FLAG_ACTIVITY_SINGLE_TOP`.

```kotlin
// onNewIntent fires for singleTop/singleTask re-use
override fun onNewIntent(intent: Intent) {
    super.onNewIntent(intent)
    handleDeepLink(intent)
}
```

Trade-off: `singleTask`/`singleInstance` change task/back-stack behavior and can
surprise users; reserve them for genuine single-instance screens (launchers,
auth, system-like dialogs).

**📚 Reference:** https://outcomeschool.com/blog/singletask-launchmode-in-android · https://youtu.be/WYkQEnm4jeI

---

## 21. What is a `Fragment` and its lifecycle?

A **Fragment** is a reusable, modular portion of UI hosted **inside** an activity
(or another fragment). It has its own lifecycle and back stack but is always tied
to its host's lifecycle. Fragments enable adaptive layouts (e.g. master-detail on
tablets), view-pager pages, and navigation destinations.

A Fragment has **two intertwined lifecycles**: the fragment instance and its
**view**. Key callbacks (creation order):

1. **`onAttach(context)`** — attached to its host.
2. **`onCreate(savedInstanceState)`** — fragment created (no view yet); init
   non-view state.
3. **`onCreateView(...)`** — inflate and return the fragment's view hierarchy.
4. **`onViewCreated(view, ...)`** — view exists; set up views, observers.
5. **`onViewStateRestored(...)`** — saved view state restored.
6. **`onStart()` → `onResume()`** — mirror the host: visible, then interactive.

Teardown order:

7. **`onPause()` → `onStop()`**.
8. **`onSaveInstanceState(...)`**.
9. **`onDestroyView()`** — the **view** is destroyed (but the fragment instance
   may live on, e.g. on the back stack). **Clear view references / binding here**
   to avoid leaks.
10. **`onDestroy()`** → **`onDetach()`** — fragment fully gone.

Critical nuance: the **view lifecycle is shorter** than the fragment lifecycle.
A fragment can be removed to the back stack — its view is destroyed
(`onDestroyView`) while the instance survives, then `onCreateView` runs again when
it returns. Always observe LiveData/flows with `viewLifecycleOwner`, not the
fragment, in `onViewCreated`.

**📚 Reference:** https://developer.android.com/guide/fragments/lifecycle

---

## 22. `Fragment` vs `Activity` — relationship and when to use each

- An **Activity** is a top-level, system-managed component with a window and an
  entry in the manifest; it's an entry point the OS can launch.
- A **Fragment** is a sub-component that **lives inside** an activity (or another
  fragment) and manages a piece of that activity's UI. It cannot exist on its own
  — it needs a host. A single activity can host many fragments and swap them.

Relationship: the activity provides the window and overall lifecycle; fragments
are modular UI chunks the activity composes and the `FragmentManager` controls.

**When to use a Fragment rather than an Activity:**

- You have UI components/flows **reused across screens** (one fragment, many
  hosts).
- You want **multiple panes side-by-side** (tablet master-detail, `ViewPager2`
  tabs) within one screen.
- You're using the **single-activity architecture** with Jetpack **Navigation**,
  where destinations are fragments — cheaper transactions than full activities,
  shared `ViewModel`/scope, and simpler back-stack handling.

When an Activity is appropriate: a genuinely separate entry point, a screen
launched by an external implicit intent, or a distinct task.

**📚 Reference:** https://stackoverflow.com/questions/10478233/why-fragments-and-when-to-use-fragments-instead-of-activities

---

## 23. Why use only the default constructor for a `Fragment`?

The framework **recreates fragments using reflection** by calling the **no-arg
(default) constructor** — for example after a configuration change or process
death restoration. If you add a custom constructor with parameters, the system
cannot call it during recreation, so it either crashes or silently creates the
fragment without your data, losing state.

Correct pattern: pass arguments via a `Bundle` set with `setArguments()` (often
via a `newInstance` factory). The system **automatically saves and restores
`arguments`**, so the fragment is reconstructed with the same data it was created
with.

```kotlin
class DetailFragment : Fragment() {
    companion object {
        private const val ARG_ID = "id"
        fun newInstance(id: Long) = DetailFragment().apply {
            arguments = bundleOf(ARG_ID to id)
        }
    }
    private val id: Long get() = requireArguments().getLong(ARG_ID)
}
```

This guarantees the fragment is restored to the same state it was initialized
with.

**📚 Reference:** https://www.youtube.com/watch?v=CitBt0FZFIc · https://outcomeschool.com/blog/default-constructor-to-create-a-fragment

---

## 24. `add` vs `replace` and `addToBackStack()`

Within a `FragmentTransaction`:

- **`add(containerId, fragment)`** — **adds** a new fragment on top; the existing
  fragment(s) **remain attached and their views stay**. The old fragment is not
  destroyed and does **not** run `onPause`/`onStop`/`onDestroyView` — both can be
  active (and overlapping) in the same container.
- **`replace(containerId, fragment)`** — **removes** all fragments currently in
  the container, then adds the new one. The removed fragment runs `onPause` →
  `onStop` → `onDestroyView` (its view is gone). When you navigate back, that
  fragment's `onCreateView` is invoked again.

In short: with `replace`, lifecycle callbacks (`onPause`, `onStop`,
`onCreateView`, etc.) fire for the outgoing fragment; with `add` they don't,
because it's never removed.

**`addToBackStack(name)`**: records the transaction on the back stack so pressing
**Back** reverses it (restoring the previous fragment state) instead of leaving
the app. Without it, a `replace` is irreversible by Back and the prior fragment
is lost.

```kotlin
supportFragmentManager.commit {
    setReorderingAllowed(true)
    replace(R.id.container, DetailFragment.newInstance(id))
    addToBackStack("detail")   // Back returns to the previous fragment
}
```

**📚 Reference:** https://stackoverflow.com/questions/24466302/basic-difference-between-add-and-replace-method-of-fragment/24466345 · https://stackoverflow.com/questions/22984950/what-is-the-meaning-of-addtobackstack-with-null-parameter

---

## 25. `FragmentPagerAdapter` vs `FragmentStatePagerAdapter` (and ViewPager2)

These were the two adapters for the legacy **`ViewPager`**:

- **`FragmentPagerAdapter`** — keeps **every visited fragment instance in
  memory**; only the **view** is destroyed when off-screen (it calls
  `detach()`, not `remove()`). Revisiting recreates the view but reuses the
  fragment instance. Best for a **small, fixed** number of pages (e.g. a few
  tabs). Can use a lot of memory if there are many pages.
- **`FragmentStatePagerAdapter`** — **destroys the fragment instance** when it's
  off-screen, saving only its `savedInstanceState`, and recreates it on
  revisit. Far lower memory; best for a **large or unknown** number of pages.

**Modern alternative (2026): both are deprecated.** Use **`ViewPager2`** with a
**`FragmentStateAdapter`** (which behaves like the old `FragmentStatePagerAdapter`
— it saves/restores state and recreates fragments as needed). `ViewPager2` is
backed by `RecyclerView`, supports vertical paging, RTL, and `DiffUtil`-style
updates.

```kotlin
class PagerAdapter(fa: FragmentActivity) : FragmentStateAdapter(fa) {
    override fun getItemCount() = 3
    override fun createFragment(position: Int): Fragment = when (position) {
        0 -> HomeFragment()
        1 -> SearchFragment()
        else -> ProfileFragment()
    }
}
viewPager2.adapter = PagerAdapter(this)
```

Migration mapping: `getCount()` → `getItemCount()`, `getItem()` →
`createFragment()`; whether you used `FragmentPagerAdapter` or
`FragmentStatePagerAdapter`, you now use `FragmentStateAdapter`. Use
`setOffscreenPageLimit` if you need adjacent pages kept alive.

**📚 Reference:** https://developer.android.com/develop/ui/views/animations/vp2-migration

---

## 26. What is a retained `Fragment`?

A retained fragment is one whose **instance survives configuration changes** —
`setRetainInstance(true)` told the `FragmentManager` not to destroy/recreate the
fragment on rotation (only its view goes through `onDestroyView`/`onCreateView`).
Historically this was used to hold expensive objects (threads, bitmaps, network
state) across rotation without re-fetching.

**Important (2026):** `setRetainInstance` is **deprecated**. The recommended
replacement is **`ViewModel`**, which is purpose-built to survive config changes,
is scoped correctly, and avoids the leak/edge-case pitfalls of retained
fragments (which couldn't be added to the back stack and complicated nested
fragments). Mention the legacy concept, but state you'd use a `ViewModel` today.

```kotlin
// Modern equivalent of "retained" data
class MyViewModel : ViewModel() { /* survives rotation, cleared on real finish */ }
private val vm: MyViewModel by viewModels()
```

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_softwareengineer-androiddev-android-activity-7265620144289193984-hlpH

---

## 27. How to communicate between two Fragments?

Fragments should **never talk to each other directly** (it couples them and
breaks reuse). Options, preferred first:

1. **Shared `ViewModel` scoped to the host activity** — both fragments obtain the
   same `ViewModel` via `activityViewModels()` and communicate through it
   (LiveData/`StateFlow`). The cleanest, lifecycle-safe approach.

   ```kotlin
   // In both fragments:
   private val shared: SharedViewModel by activityViewModels()
   // Fragment A:
   shared.select(item)
   // Fragment B observes:
   shared.selected.observe(viewLifecycleOwner) { render(it) }
   ```

2. **Fragment Result API** — for one-off results between a fragment and its
   parent/sibling without a shared ViewModel:

   ```kotlin
   // Receiver (set in onCreate/onViewCreated):
   parentFragmentManager.setFragmentResultListener("key", viewLifecycleOwner) { _, bundle ->
       val value = bundle.getString("value")
   }
   // Sender:
   parentFragmentManager.setFragmentResult("key", bundleOf("value" to "hi"))
   ```

3. **Via the host activity through an interface** (older pattern): the fragment
   defines a callback interface that the activity implements; the activity relays
   to the other fragment. Verbose and tightly coupled — prefer the above.

Avoid `findFragmentById`/casting to call another fragment's methods directly.

---

## 28. What is a `Bundle`? Size limits

A **`Bundle`** is a key-value map for passing data between Android components and
across the process boundary. It stores values via the **`Parcelable`** mechanism
(it serializes to a `Parcel`), which is why it's used for `Intent` extras,
fragment `arguments`, and `onSaveInstanceState`.

- Used in: `Intent.putExtra`/extras, `Fragment.setArguments`,
  `onSaveInstanceState`/`onRestoreInstanceState`, `startActivityForResult`
  results, etc.
- Supports primitives, `String`, `CharSequence`, `Parcelable`, `Serializable`,
  arrays, and nested `Bundle`s.

**Size limit / pitfall:** a `Bundle` passed via IPC goes through a Binder
transaction whose buffer is **~1 MB total per process** (practically you should
keep a single transaction well under that — a few hundred KB at most). Exceeding
it throws **`TransactionTooLargeException`**. Never put bitmaps, large lists, or
big blobs in a Bundle/Intent — pass an **id/URI** and load the data from a
repository/DB instead.

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_outcomeschool-softwareengineer-tech-activity-7355846437718413316-jRI4

---

## 29. Transferring objects between activities — Serializable vs Parcelable

Put data in `Intent` extras / a `Bundle`. For custom objects you need either
`Serializable` or `Parcelable`:

- **`Serializable`** — a Java marker interface; serialization is automatic but
  uses **reflection**, which is **slow** and creates many temporary objects (GC
  pressure). Easy to use, poor performance.
- **`Parcelable`** — an Android-specific interface optimized for IPC; you
  describe how to write/read fields, so **no reflection** → significantly faster
  and more memory-efficient. The recommended approach on Android.

Use the **`@Parcelize`** Kotlin plugin to generate the boilerplate:

```kotlin
@Parcelize
data class User(val id: Long, val name: String) : Parcelable

startActivity(Intent(this, DetailActivity::class.java).putExtra("user", user))
// Receiver:
val user = intent.getParcelableExtra("user", User::class.java) // API 33+ typed overload
```

Trade-off: `Parcelable` is faster but more verbose (mitigated by `@Parcelize`);
`Serializable` is simpler but slower. Remember the Bundle size limit — pass ids,
not large payloads. (`transient` can be used to exclude a field from Java
serialization.)

---

## 30. `Dialog` vs `DialogFragment`

- **`Dialog`** — a basic floating window. It is **not lifecycle-aware** and is
  effectively owned by the activity directly; on a **configuration change
  (rotation) it is dismissed/leaks** because nothing re-creates it, and you can
  hit window-leak warnings if the activity is destroyed while the dialog shows.
- **`DialogFragment`** — wraps a dialog inside a fragment, so it participates in
  the fragment lifecycle and **`FragmentManager`**. It **survives configuration
  changes** (re-created and re-shown automatically), handles its own state, and
  integrates with the back stack.

Recommendation: always use **`DialogFragment`** for dialogs that should persist
across rotation. Use a bare `Dialog` only for trivial, transient prompts.

```kotlin
class ConfirmDialog : DialogFragment() {
    override fun onCreateDialog(savedInstanceState: Bundle?): Dialog =
        AlertDialog.Builder(requireContext())
            .setTitle("Delete?")
            .setPositiveButton("Yes") { _, _ -> /* ... */ }
            .setNegativeButton("Cancel", null)
            .create()
}
ConfirmDialog().show(supportFragmentManager, "confirm")
```

**📚 Reference:** https://stackoverflow.com/questions/7977392/android-dialogfragment-vs-dialog

---

## Intents & Broadcasting

## 31. What is an `Intent`? Explicit vs Implicit

An **`Intent`** is an asynchronous messaging object used to request an action
from a component — start an `Activity`, start/bind a `Service`, or deliver a
`broadcast`. It carries the action, optional data (`Uri`), category, type, and
**extras** (a `Bundle`).

**Explicit Intent** — names the exact target component (by class/package). Used
for navigation **within your app**.

```kotlin
val intent = Intent(this, ActivityTwo::class.java)
intent.putExtra("id", 42)
startActivity(intent)
```

**Implicit Intent** — declares an **action** to perform without naming a
component; the system finds any app whose `intent-filter` can handle it (and
shows a chooser if multiple match). Used to leverage other apps (open a URL,
share, dial).

```kotlin
val intent = Intent(Intent.ACTION_VIEW, Uri.parse("https://google.com"))
if (intent.resolveActivity(packageManager) != null) startActivity(intent)
```

Note (Android 11+): to query/launch other apps you may need a `<queries>` element
in the manifest due to package-visibility restrictions.

**📚 Reference:** https://developer.android.com/guide/components/intents-filters

---

## 32. What is a `BroadcastReceiver`? Types of broadcasts

A **`BroadcastReceiver`** is a component that responds to system-wide or app
**broadcast** messages (events) — e.g. connectivity changes, battery low, boot
completed, or your own custom events. It has a single entry point,
`onReceive(context, intent)`, which must finish quickly (it runs on the main
thread; long work risks an ANR — offload to `goAsync()` or `WorkManager`).

Registration:

- **Manifest-declared (static)** — declared in `AndroidManifest.xml`. Heavily
  restricted since Android 8 (Oreo): most **implicit** system broadcasts can no
  longer be received via the manifest (use explicit, or runtime registration, or
  `WorkManager`/`JobScheduler` constraints instead).
- **Context-registered (dynamic)** — registered/unregistered at runtime with
  `registerReceiver()`; only active while registered. Android 13+ requires
  specifying `RECEIVER_EXPORTED`/`RECEIVER_NOT_EXPORTED`.

**Types of broadcasts:**

- **Normal broadcasts** (`sendBroadcast`) — asynchronous, delivered to all
  matching receivers in undefined order; can't be aborted.
- **Ordered broadcasts** (`sendOrderedBroadcast`) — delivered one receiver at a
  time in priority order; a receiver can pass results along or **abort** the
  broadcast.
- **Local broadcasts** — historically `LocalBroadcastManager` for in-app-only
  events; now **deprecated** — use a shared `ViewModel`/`Flow`/`LiveData` or
  observable pattern instead.
- **Sticky broadcasts** — `sendStickyBroadcast`, also **deprecated/removed**
  (the last value "stuck" for future receivers).
- **System broadcasts** — sent by the OS (e.g. `BOOT_COMPLETED`,
  `CONNECTIVITY_CHANGE`).

```kotlin
val receiver = object : BroadcastReceiver() {
    override fun onReceive(c: Context, i: Intent) { /* handle */ }
}
ContextCompat.registerReceiver(
    this, receiver, IntentFilter(Intent.ACTION_BATTERY_LOW),
    ContextCompat.RECEIVER_NOT_EXPORTED
)
```

**📚 Reference:** https://developer.android.com/guide/components/broadcasts

---

## 33. How broadcasts pass messages around your app

Broadcasts are a **publish/subscribe** mechanism built on `Intent`s:

1. A **sender** builds an `Intent` describing an event (action + optional extras)
   and calls `sendBroadcast(intent)` (or ordered/explicit variants).
2. The system matches the intent against registered receivers — via
   `IntentFilter`s (manifest or runtime) or an explicit component target.
3. Each matching **`BroadcastReceiver.onReceive()`** runs with the intent,
   reads the extras, and reacts.

This lets a **Service** notify the **UI**, or one part of the app signal another,
without holding direct references. Classic example: a download `Service`
broadcasts "download complete" and an `Activity`'s receiver updates the UI.

**Modern guidance:** for purely **in-app** messaging, prefer a shared
`ViewModel` exposing a `StateFlow`/`LiveData`, or an app-scoped `Flow` — it's
lifecycle-aware, type-safe, and avoids the overhead and Oreo+ restrictions of
broadcasts. Reserve broadcasts for genuine cross-app/system events.

```kotlin
// Service signals UI (explicit, in-app)
sendBroadcast(Intent("com.app.DOWNLOAD_DONE").setPackage(packageName).putExtra("id", id))
```

**📚 Reference:** https://stackoverflow.com/questions/7276537/using-a-broadcast-intent-broadcast-receiver-to-send-messages-from-a-service-to-a

---

## 34. What is a `PendingIntent`? (and Sticky Intent)

A **`PendingIntent`** is a token that **wraps an `Intent` plus the permission to
execute it**, which you hand to **another app or the system** so it can fire that
intent **later, on your behalf**, with **your** app's identity/permissions — even
if your process is dead at that time.

Common uses: **notification** actions/content, **alarms** (`AlarmManager`),
**App Widgets**, and geofencing/location callbacks.

Factory methods: `getActivity()`, `getService()`, `getBroadcast()`,
`getForegroundService()`.

```kotlin
val intent = Intent(this, DetailActivity::class.java).putExtra("id", 7)
val pi = PendingIntent.getActivity(
    this, 0, intent,
    PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE // immutable required (API 31+)
)
notificationBuilder.setContentIntent(pi)
```

Note: since **Android 12 (API 31)** you must explicitly set
**`FLAG_IMMUTABLE`** or `FLAG_MUTABLE`.

**Sticky Intent** (related concept): an old broadcast (`sendStickyBroadcast`)
whose last value "stuck" with the system so future registrants got the most
recent value immediately (e.g. `ACTION_BATTERY_CHANGED`). It is **deprecated**
because it had no delivery guarantees or security. Don't use it in new code.

---

## Services

## 35. What is a `Service`? Its lifecycle

A **`Service`** is a component that performs **long-running work without a UI**.
It can keep running even when the user switches away (subject to modern
background limits). It does **not** create its own thread — by default it runs on
the **main thread** (see Q36). Note: a component can **bind** to a service to
interact with it, including **IPC** if the service runs in a separate process.

Two ways to run a service, with two lifecycle paths:

- **Started service** (`startService` / `ContextCompat.startForegroundService`):
  - `onCreate()` (once) → `onStartCommand()` (each start request) → runs until
    `stopSelf()` or `stopService()` → `onDestroy()`.
  - `onStartCommand` returns a restart policy: `START_STICKY`,
    `START_NOT_STICKY`, or `START_REDELIVER_INTENT`.
- **Bound service** (`bindService`):
  - `onCreate()` → `onBind()` (returns an `IBinder`) → clients interact →
    last client `unbindService()` → `onUnbind()` → `onDestroy()`.

A service can be **both** started and bound; it lives until it's neither started
nor bound.

```kotlin
class MyService : Service() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        // do work (on a background thread!), then maybe stopSelf()
        return START_STICKY
    }
    override fun onBind(intent: Intent?): IBinder? = null
    override fun onDestroy() { /* cleanup */ }
}
```

Modern context: background started-services are heavily restricted (Android 8+);
for deferrable/guaranteed background work prefer **`WorkManager`**. Use a
**foreground service** (Q38) for user-visible ongoing work like playback or
navigation.

**📚 Reference:** https://developer.android.com/guide/components/services · https://www.linkedin.com/posts/amit-shekhar-iitbhu_softwareengineer-androiddev-android-activity-7265212180570992640-CJn_

---

## 36. On which thread does a `Service` run?

By default a `Service` runs on the application's **main (UI) thread** — the same
thread as your activities. It is **not** a separate thread or process. Therefore,
doing blocking work (network, disk, heavy CPU) directly in `onStartCommand()` or
`onBind()` will **block the UI and can cause an ANR**.

To do real work, **create your own background thread / coroutine** inside the
service:

```kotlin
override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
    CoroutineScope(Dispatchers.IO).launch {
        doWork()
        stopSelf(startId)
    }
    return START_NOT_STICKY
}
```

(Exception/aside: `IntentService` — now deprecated — created its own worker
thread automatically; see Q37.) If you set `android:process` in the manifest, the
service runs in a separate **process**, but still on that process's main thread by
default.

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_softwareengineer-androiddev-android-activity-7283717741130215424-Vn39

---

## 37. `Service` vs `IntentService`

- **`Service`** — runs on the **main thread**; you manage threading yourself; can
  be started and/or bound; handles **multiple, concurrent or long-lived** tasks;
  you control when it stops. Can run in a separate process.
- **`IntentService`** (subclass of `Service`) — created its **own background
  worker thread**, processed each start `Intent` **sequentially** in
  `onHandleIntent()` (off the main thread), and **stopped itself automatically**
  when the queue was empty. Designed for **short, fire-and-forget** background
  tasks. It always ran in the app process and couldn't be bound meaningfully.

Summary: `IntentService` = short async tasks on a built-in worker thread,
auto-stops; `Service` = long/flexible work, your own threading, manual stop.

**Modern note (2026):** `IntentService` is **deprecated**. Its use cases are now
served by **`WorkManager`** (deferrable/guaranteed background work) or
**`JobIntentService`** (also deprecated) / a coroutine inside a foreground
service for immediate work. Don't use `IntentService` in new code.

**📚 Reference:** https://stackoverflow.com/questions/15524280/service-vs-intentservice-in-the-android-platform

---

## 38. What is a Foreground Service?

A **foreground service** performs work the **user is actively aware of** and must
display a **persistent notification** so it isn't silently killed. Examples:
music playback, navigation, active file upload/download, fitness tracking. Because
the user sees it, the system gives it higher priority and does **not** subject it
to the same background-execution limits.

Modern requirements (Android 9+ → 14):

- Start with `ContextCompat.startForegroundService(...)` and call
  **`startForeground(id, notification)`** within ~5 seconds, or the system kills
  the app with an exception.
- Needs the **`FOREGROUND_SERVICE`** permission.
- **Android 14 (API 34):** you **must declare a `foregroundServiceType`** on the
  `<service>` in the manifest **and** hold the matching per-type permission (e.g.
  `FOREGROUND_SERVICE_LOCATION`, `FOREGROUND_SERVICE_MEDIA_PLAYBACK`,
  `FOREGROUND_SERVICE_DATA_SYNC`, etc.). Calling `startForeground` with a type you didn't declare
  throws a `SecurityException` / `MissingForegroundServiceTypeException`. Multiple
  types combine with `|`. Some types also require runtime permission first
  (e.g. location). Google Play additionally requires declaring FGS usage in the
  Play Console.

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />
<service android:name=".PlayerService"
         android:foregroundServiceType="mediaPlayback" android:exported="false" />
```

```kotlin
val notif = NotificationCompat.Builder(this, CHANNEL_ID)
    .setSmallIcon(R.drawable.ic_play).setContentTitle("Playing").build()
ServiceCompat.startForeground(this, 1, notif, FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK)
```

Trade-off: high priority and survivability, but a mandatory visible notification
and strict policy/permission rules. For deferrable work that doesn't need to be
immediate, prefer `WorkManager` (which can promote itself to a foreground service
for long tasks).

**📚 Reference:** https://developer.android.com/develop/background-work/services/fgs/service-types · https://www.linkedin.com/posts/outcomeschool_foreground-service-in-android-activity-7303268432030879745-TFVI

---

## 39. What is `JobScheduler`?

`JobScheduler` is a system service (API 21+) for scheduling **deferrable
background jobs** that run under **constraints** — e.g. only on Wi-Fi/unmetered
network, while charging, when idle, or with a deadline. The OS **batches** jobs
across apps to save battery and intelligently picks when to run them. You define a
`JobService` and submit a `JobInfo`.

```kotlin
val job = JobInfo.Builder(JOB_ID, ComponentName(this, MyJobService::class.java))
    .setRequiredNetworkType(JobInfo.NETWORK_TYPE_UNMETERED)
    .setRequiresCharging(true)
    .build()
(getSystemService(JOB_SCHEDULER_SERVICE) as JobScheduler).schedule(job)
```

**Modern note (2026):** for most apps, **`WorkManager`** is the recommended API —
it is built **on top of** `JobScheduler` (and older mechanisms) under the hood,
adds **persistence/guarantees**, simpler chaining, and a unified API across OS
versions. Use raw `JobScheduler` only for special low-level needs; otherwise
prefer `WorkManager`.

**📚 Reference:** https://developer.android.com/reference/android/app/job/JobScheduler

---

## 40. How does `WorkManager` guarantee task execution?

`WorkManager` is the Jetpack library for **deferrable, guaranteed** background
work — work that must run eventually even across **app exits, device reboots, and
process death**, optionally under constraints.

How it guarantees execution:

- **Persistence:** every enqueued `WorkRequest` is written to an internal
  **Room/SQLite database**. The work survives the app being killed or the device
  rebooting — `WorkManager` reschedules persisted work after reboot (via a boot
  receiver).
- **OS scheduler delegation:** under the hood it uses the **best available
  scheduler** for the API level — `JobScheduler` on API 23+ (and historically
  `AlarmManager` + broadcast receivers / `GcmNetworkManager` on older versions) —
  so the OS actually runs the job at an appropriate time.
- **Constraints & retries:** you attach `Constraints` (network, charging, storage,
  idle). If a worker returns **`Result.retry()`**, WorkManager re-runs it with a
  configurable **backoff** policy. If constraints aren't met, it waits.
- **Guaranteed-to-run, not guaranteed-immediate:** the promise is the work will
  run **eventually** once constraints are satisfied — it is **not** for exact-time
  or instant execution.

```kotlin
val work = OneTimeWorkRequestBuilder<UploadWorker>()
    .setConstraints(Constraints.Builder()
        .setRequiredNetworkType(NetworkType.CONNECTED).build())
    .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 10, TimeUnit.SECONDS)
    .build()
WorkManager.getInstance(context).enqueue(work)
```

It also supports unique work, chaining (`beginWith().then()`), parallel work, and
long-running tasks via `setForeground()`. Use it for syncing, uploads, periodic
cleanup, etc. — not for exact alarms (use `AlarmManager`) or instant in-process
work (use coroutines).

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_how-does-workmanager-guarantee-task-execution-activity-7320301933691310080-GpCR

---

## 41. What can you use for background processing in Android?

Pick the tool by **whether work must finish, when, and whether the user sees it**:

- **Kotlin Coroutines / `Dispatchers.IO`** — in-process async work tied to a
  lifecycle/`ViewModel` scope. Best for work that only needs to run **while the
  app/screen is alive** (network calls, DB). Not guaranteed across process death.
- **`WorkManager`** — **deferrable, guaranteed** work that must survive app
  exit/reboot, optionally under constraints (sync, upload, periodic). The default
  recommendation for persistent background tasks.
- **Foreground Service** — **immediate, user-visible, ongoing** work (playback,
  navigation, live location). High priority, requires a notification + FGS type.
- **`JobScheduler`** — low-level constrained job scheduling (usually via
  WorkManager instead).
- **`AlarmManager`** — **exact or inexact time-based** triggers
  (`setExactAndAllowWhileIdle` for alarms/reminders). Needs the exact-alarm
  permission on Android 12+/13+.
- **Deprecated/avoid:** `AsyncTask`, `IntentService`, `Loader`,
  `LocalBroadcastManager`.

Rule of thumb (2026): instant + only-while-alive → **coroutines**; must run
eventually/with constraints → **WorkManager**; user-aware ongoing → **foreground
service**; exact time → **AlarmManager**.

**📚 Reference:** https://developer.android.com/guide/background

---

## Inter-Process Communication

## 42. How can two distinct Android apps interact?

Several mechanisms depending on coupling and security:

- **Implicit Intents + intent filters** — the most common, loosely coupled way.
  One app sends an action (`ACTION_VIEW`, `ACTION_SEND`, custom action); another
  app that declared a matching `intent-filter` handles it. Great for "share to",
  "open with", deep links. Can pass small data via extras and get a result via
  the **Activity Result API**.

  ```kotlin
  val send = Intent(Intent.ACTION_SEND).apply {
      type = "text/plain"; putExtra(Intent.EXTRA_TEXT, "Hello")
  }
  startActivity(Intent.createChooser(send, "Share via"))
  ```

- **`ContentProvider` + `ContentResolver`** — share **structured data** (rows,
  files) between apps with URI-based, permission-controlled access.
- **Bound Service via AIDL / Messenger** — for ongoing, programmatic **IPC**
  (method calls) across apps/processes (see Q44).
- **Broadcasts** — send an explicit/permission-protected broadcast another app
  receives.
- **`FileProvider`** — share files securely via `content://` URIs with temporary
  grants.

Android 11+ **package visibility** rules mean you often need a `<queries>` entry
to see/launch other apps. Protect cross-app surfaces with **permissions** and
correct `android:exported` settings.

**📚 Reference:** https://developer.android.com/training/basics/intents

---

## 43. Can an app run in multiple processes? How?

Yes. By default all components run in one process named after the app's package,
but you can place a component in another process with the **`android:process`**
attribute in the manifest:

```xml
<service
    android:name=".SyncService"
    android:process=":sync" />        <!-- private process -->
<provider
    android:name=".DataProvider"
    android:process="com.example.shared" />  <!-- global, can be shared with other apps -->
```

- A name starting with **`:`** creates a **private** process local to your app
  (e.g. `:sync`).
- A name starting with a **lowercase letter** (a full name) creates a **global**
  process that other apps (with the same shared user id / signature) could share.

Why do it: isolate crash-prone or memory-heavy work (e.g. a WebView, ML, or media
process) so a crash/leak doesn't take down the main process; or stay alive when
the UI process is killed.

Caveats: each process gets its **own VM and its own `Application` instance** (your
`Application.onCreate` runs per process — guard init accordingly). Processes have
**separate memory**, so you can't share objects directly — you must use **IPC**
(AIDL/Messenger/ContentProvider/broadcasts) to communicate, which adds complexity
and overhead. Singletons and static state are **not** shared across processes.

**📚 Reference:** https://stackoverflow.com/questions/6567768/how-can-an-android-application-have-more-than-one-process

---

## 44. What is AIDL? Steps to create a bound service with AIDL

**AIDL (Android Interface Definition Language)** lets you define a programmatic
interface that two processes can use to communicate via **IPC**. Android uses the
**Binder** mechanism; AIDL generates the marshalling (proxy/stub) code that
serializes method calls and arguments across the process boundary. Use AIDL
**only when** you need multiple apps/processes to call into your service
**concurrently** with method-level IPC; for single-app cross-process messaging,
`Messenger` (which serializes calls onto one thread) is simpler.

Steps to create an AIDL-backed bound service:

1. **Define the `.aidl` interface** in `src/main/aidl/...`, e.g.
   `IRemoteService.aidl`:

   ```aidl
   package com.example;
   interface IRemoteService {
       int add(int a, int b);
   }
   ```

   Build generates an `IRemoteService` Java interface with an abstract `Stub`.

2. **Implement the `Stub`** inside a `Service` and return it from `onBind()`:

   ```kotlin
   class RemoteService : Service() {
       private val binder = object : IRemoteService.Stub() {
           override fun add(a: Int, b: Int): Int = a + b
       }
       override fun onBind(intent: Intent?): IBinder = binder
   }
   ```

3. **Declare the service** in the manifest (with an action / `android:exported`
   if other apps will bind).

4. **Client binds** with `bindService()`, and in `onServiceConnected` converts the
   `IBinder` to the interface with `IRemoteService.Stub.asInterface(binder)`, then
   calls methods:

   ```kotlin
   private val conn = object : ServiceConnection {
       override fun onServiceConnected(n: ComponentName, b: IBinder) {
           val service = IRemoteService.Stub.asInterface(b)
           val sum = service.add(2, 3)
       }
       override fun onServiceDisconnected(n: ComponentName) {}
   }
   bindService(Intent(this, RemoteService::class.java), conn, BIND_AUTO_CREATE)
   ```

Notes: only AIDL-supported types and `Parcelable`s can cross the boundary; remote
calls may run on a **Binder thread pool** (so the implementation must be
thread-safe); handle `DeadObjectException` / `RemoteException`.

**📚 Reference:** https://developer.android.com/guide/components/aidl

---

## 45. What is a `ContentProvider` and when is it used?

A **`ContentProvider`** encapsulates a set of data and exposes it through a
standard, URI-addressed interface (`content://authority/path`) accessed via a
**`ContentResolver`**. It abstracts the underlying storage (SQLite, files,
network) behind `query()`, `insert()`, `update()`, `delete()`, and `getType()`.

When it's used:

- **Primarily to share data with *other* apps** in a controlled, secure way —
  this is its main reason to exist. (For data used only inside your own app, you
  generally **don't** need a ContentProvider; use Room/DataStore directly.)
- To **integrate with framework features** that consume providers: Contacts,
  MediaStore, Calendar, search suggestions, `FileProvider` for sharing files,
  app widgets/`RemoteViews` adapters, sync adapters, and the Documents UI.

It enforces access with **read/write permissions** and **URI permission grants**
(`FLAG_GRANT_READ_URI_PERMISSION`), so you expose exactly what you intend.

```kotlin
val cursor = contentResolver.query(
    ContactsContract.Contacts.CONTENT_URI, null, null, null, null
)
```

Trade-off: powerful and secure for cross-app/structured data and a natural fit for
content URIs and observers (`notifyChange` + `ContentObserver`), but it's
**boilerplate-heavy** and overkill for purely internal storage.

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_softwareengineer-androiddev-android-activity-7268117553040764931-64fI
