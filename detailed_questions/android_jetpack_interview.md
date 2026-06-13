# Android Jetpack & Core Android Interview Guide

A comprehensive, up-to-date (2026) interview-prep guide covering **Android Jetpack** (architecture components — ViewModel, LiveData, StateFlow, WorkManager, DataStore, Navigation) and a set of assorted **core Android** topics (Serializable vs Parcelable, Bundle, crash troubleshooting, push notifications, AAPT, FlatBuffers, ArrayMap/SparseArray, annotations, Data Binding, and more).

Each question includes detailed explanations, Kotlin code snippets, trade-offs, and modern guidance. Answers reflect the recommended `ViewModel + StateFlow` architecture, with DataStore replacing SharedPreferences and View Binding / Compose replacing Data Binding for most new code.

---

## Table of Contents

### Android Jetpack
1. [What is Android Jetpack and why use it?](#1-what-is-android-jetpack-and-why-use-it)
2. [What is a ViewModel and how is it useful?](#2-what-is-a-viewmodel-and-how-is-it-useful)
3. [How does ViewModel work internally?](#3-how-does-viewmodel-work-internally)
4. [What are Android Architecture Components?](#4-what-are-android-architecture-components)
5. [How do you share a ViewModel between Fragments? (SharedViewModel)](#5-how-do-you-share-a-viewmodel-between-fragments-sharedviewmodel)
6. [What is LiveData?](#6-what-is-livedata)
7. [StateFlow vs LiveData](#7-stateflow-vs-livedata)
8. [How is LiveData different from ObservableField?](#8-how-is-livedata-different-from-observablefield)
9. [setValue vs postValue in LiveData](#9-setvalue-vs-postvalue-in-livedata)
10. [Explain WorkManager and its use cases](#10-explain-workmanager-and-its-use-cases)
11. [Minimum repeat interval for a PeriodicWorkRequest](#11-minimum-repeat-interval-for-a-periodicworkrequest)
12. [How does WorkManager guarantee task execution?](#12-how-does-workmanager-guarantee-task-execution)
25. [SavedStateHandle (surviving process death in a ViewModel)](#25-savedstatehandle-surviving-process-death-in-a-viewmodel)
26. [Paging 3 (PagingSource, Pager, RemoteMediator)](#26-paging-3-pagingsource-pager-remotemediator)

### Others
13. [Serializable vs Parcelable — which is best?](#13-serializable-vs-parcelable--which-is-best)
14. [Why is Bundle used for passing data instead of a Map?](#14-why-is-bundle-used-for-passing-data-instead-of-a-map)
15. [How do you troubleshoot a crashing application?](#15-how-do-you-troubleshoot-a-crashing-application)
16. [Explain the Android push notification system](#16-explain-the-android-push-notification-system)
17. [What is AAPT?](#17-what-is-aapt)
18. [FlatBuffers vs JSON](#18-flatbuffers-vs-json)
19. [HashMap, ArrayMap and SparseArray](#19-hashmap-arraymap-and-sparsearray)
20. [Advantages of SparseArray](#20-advantages-of-sparsearray)
21. [What are Annotations?](#21-what-are-annotations)
22. [How to create a custom Annotation?](#22-how-to-create-a-custom-annotation)
23. [What is the Android Support Library? Why was it introduced?](#23-what-is-the-android-support-library-why-was-it-introduced)
24. [What is Android Data Binding?](#24-what-is-android-data-binding)

---

## 1. What is Android Jetpack and why use it?

**Android Jetpack** is a suite of libraries, tools, and architectural guidance from Google that helps developers write high-quality apps more easily. It is the successor to the Android Support Library and is published under the `androidx.*` namespace.

Jetpack components are grouped into four categories:

- **Foundation** — `AppCompat`, `Android KTX` (Kotlin extensions), `Multidex`, `Test`.
- **Architecture** — `ViewModel`, `LiveData`, `Room`, `WorkManager`, `Navigation`, `Paging`, `DataStore`, `Lifecycle`.
- **Behavior** — `CameraX`, `Notifications`, `Permissions`, `Sharing`, `Media`.
- **UI** — `Jetpack Compose`, `View Binding`, `Fragment`, `Animation & Transitions`, `Emoji`.

**Why use it:**

- **Backward compatibility** — `androidx` libraries are unbundled from the OS and decoupled from the platform release cycle, so new features reach older API levels.
- **Eliminates boilerplate** — components like `ViewModel`, `Room`, and `WorkManager` handle lifecycle, persistence, and background scheduling for you.
- **Reduces crashes & leaks** — lifecycle-aware components (`LiveData`, `Lifecycle`) prevent common bugs like updating a destroyed UI or leaking an Activity.
- **Recommended architecture** — Jetpack underpins the official guide to app architecture (UI layer → domain layer → data layer).
- **Consistent versioning** — components are versioned independently and can be adopted incrementally.

**📚 Reference:** [Android Jetpack](https://developer.android.com/jetpack)

---

## 2. What is a ViewModel and how is it useful?

A **`ViewModel`** is a lifecycle-aware class designed to **store and manage UI-related state** in a way that survives configuration changes (rotation, language change, multi-window resize, dark-mode toggle, etc.).

```kotlin
class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

    fun loadUser(id: String) {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            val user = repository.getUser(id)
            _uiState.update { it.copy(isLoading = false, user = user) }
        }
    }
}
```

Usage in a Fragment/Activity:

```kotlin
class UserFragment : Fragment() {
    private val viewModel: UserViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        viewLifecycleOwner.lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state -> render(state) }
            }
        }
    }
}
```

**Why it is useful:**

- **Survives configuration changes** — data is retained when the Activity/Fragment is recreated, avoiding re-fetching from network/DB.
- **Separation of concerns** — keeps UI logic and state out of the Activity/Fragment (which are hard to test and tightly coupled to the framework).
- **`viewModelScope`** — provides a `CoroutineScope` automatically cancelled in `onCleared()`, preventing coroutine leaks.
- **Testable** — a plain class with no Android view dependencies.
- **Scoping** — can be scoped to an Activity, Fragment, Navigation graph, or `@HiltViewModel`-injected.

**Important:** A ViewModel must **never** hold a reference to a View, Activity context, or Fragment — that causes memory leaks. For app `Context`, use `AndroidViewModel(application)` or inject an application context.

**📚 Reference:** [What is a ViewModel and how is it useful?](https://youtu.be/ORtieK5f_zg)

---

## 3. How does ViewModel work internally?

The key promise of a ViewModel is surviving configuration changes. Here is the mechanism:

1. **`ViewModelStoreOwner`** — `ComponentActivity` and `Fragment` implement this interface, which exposes a `ViewModelStore`.
2. **`ViewModelStore`** — a thin wrapper around a `HashMap<String, ViewModel>`. It is the actual container that holds your ViewModel instances keyed by a canonical class name (or a custom key).
3. **`ViewModelProvider`** — when you call `by viewModels()` / `ViewModelProvider(owner).get(...)`, it looks up the requested ViewModel in the owner's `ViewModelStore`. If present, it returns the **same instance**; if not, it uses a `ViewModelProvider.Factory` to create one and stores it.

**Surviving a configuration change (Activity case):**

When the system recreates an Activity due to a configuration change, the old instance is destroyed and a new one is created. The `ViewModelStore` is preserved across this recreation via the framework's **`onRetainNonConfigurationInstance()` / `getLastNonConfigurationInstance()`** mechanism:

- Before destruction, `ComponentActivity` returns its `ViewModelStore` inside a `NonConfigurationInstances` object from `onRetainNonConfigurationInstance()`.
- The newly created Activity retrieves that same `ViewModelStore` via `getLastNonConfigurationInstance()`.

Because the same `ViewModelStore` (and thus the same ViewModel instances) is handed to the new Activity, your data survives rotation **without** serialization. This is fundamentally different from `onSaveInstanceState`, which serializes a small `Bundle` to disk and survives process death.

**Distinguishing a config change from a true finish:**

`onCleared()` is only called when the owner is **permanently** going away (e.g., Activity finishing, Fragment removed). On a config change, the framework knows it is retaining the store, so `onCleared()` is *not* called. `ComponentActivity` checks `isChangingConfigurations()` to decide whether to clear the store.

```kotlin
// Simplified shape of ViewModelStore
class ViewModelStore {
    private val map = HashMap<String, ViewModel>()
    fun put(key: String, vm: ViewModel) { /* ... */ }
    fun get(key: String): ViewModel? = map[key]
    fun clear() {
        map.values.forEach { it.clear() } // triggers onCleared()
        map.clear()
    }
}
```

**Note on process death:** A ViewModel does **not** survive system-initiated process death. For that, combine it with `SavedStateHandle` (passed into the ViewModel constructor) which is backed by the saved-instance-state `Bundle`.

```kotlin
class SearchViewModel(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    val query: StateFlow<String> =
        savedStateHandle.getStateFlow("query", "")

    fun setQuery(q: String) { savedStateHandle["query"] = q }
}
```

**📚 Reference:** [How ViewModel works under the hood](https://proandroiddev.com/how-viewmodel-works-under-the-hood-52a4f1ff64cf)

---

## 4. What are Android Architecture Components?

**Architecture Components** are the subset of Jetpack libraries that help you design robust, testable, maintainable apps. They are the building blocks of Google's recommended app architecture.

| Component | Purpose |
|-----------|---------|
| **ViewModel** | Holds & manages UI state across configuration changes |
| **LiveData** | Lifecycle-aware observable data holder |
| **Lifecycle** | Lifecycle-aware components; `LifecycleOwner`, `LifecycleObserver` |
| **Room** | SQLite ORM with compile-time query verification |
| **WorkManager** | Deferrable, guaranteed background work |
| **Navigation** | In-app navigation, deep links, type-safe args |
| **Paging** | Loads large data sets page-by-page |
| **DataStore** | Modern, coroutine/Flow-based key-value & typed storage (replaces SharedPreferences) |
| **SavedState** | Saves/restores UI state through process death |

**Recommended architecture (layered):**

- **UI layer** — Activity/Fragment/Compose + ViewModel exposing a `StateFlow<UiState>`.
- **Domain layer** (optional) — use cases encapsulating business logic.
- **Data layer** — repositories backed by data sources (Room, Retrofit, DataStore).

Principles: **separation of concerns**, **drive UI from immutable state models**, and a **single source of truth** per data type, owned by the layer that creates/edits it.

**📚 Reference:** [Guide to app architecture](https://developer.android.com/topic/architecture)

---

## 5. How do you share a ViewModel between Fragments? (SharedViewModel)

When multiple fragments hosted by the same Activity need to share data (e.g., a master/detail flow, or a multi-step form), you scope a single ViewModel to the **Activity** so both fragments get the **same instance**.

Use the `by activityViewModels()` delegate (from `androidx.fragment:fragment-ktx`):

```kotlin
class SharedViewModel : ViewModel() {
    private val _selectedItem = MutableStateFlow<Item?>(null)
    val selectedItem: StateFlow<Item?> = _selectedItem.asStateFlow()

    fun select(item: Item) { _selectedItem.value = item }
}

class ListFragment : Fragment() {
    // scoped to the host Activity
    private val sharedViewModel: SharedViewModel by activityViewModels()

    private fun onItemClicked(item: Item) = sharedViewModel.select(item)
}

class DetailFragment : Fragment() {
    private val sharedViewModel: SharedViewModel by activityViewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        viewLifecycleOwner.lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                sharedViewModel.selectedItem.collect { item -> /* render */ }
            }
        }
    }
}
```

**Other scoping options:**

- **Navigation-graph scope** — share only among destinations within a nested nav graph, so the ViewModel is cleared when you leave the graph:
  ```kotlin
  private val vm: SharedViewModel by navGraphViewModels(R.id.checkout_graph)
  ```
- **Parent fragment scope** — when child fragments share with a parent:
  ```kotlin
  private val vm: SharedViewModel by viewModels(
      ownerProducer = { requireParentFragment() }
  )
  ```

**Benefits over fragment-to-fragment callbacks / interfaces:** loose coupling (fragments never reference each other), survives configuration changes, and a single source of truth.

**📚 Reference:** [SharedViewModel in Android](https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineering-tech-activity-7262328327531577344-fwSE)

---

## 6. What is LiveData?

**`LiveData`** is a **lifecycle-aware**, observable data holder. Unlike a plain observable, it respects the lifecycle of its observers (Activities, Fragments) and only delivers updates to observers in an **active** state (`STARTED` or `RESUMED`).

```kotlin
class CounterViewModel : ViewModel() {
    private val _count = MutableLiveData(0)
    val count: LiveData<Int> = _count

    fun increment() { _count.value = (_count.value ?: 0) + 1 }
}

// Observing
viewModel.count.observe(viewLifecycleOwner) { value ->
    textView.text = value.toString()
}
```

**Key benefits:**

- **No memory leaks** — observers are bound to a `LifecycleOwner` and automatically removed when it is `DESTROYED`.
- **No crashes from stopped activities** — inactive observers don't receive events.
- **Always up to date** — an observer becoming active receives the latest value.
- **Configuration-change resilient** — when paired with a ViewModel, the data outlives recreation.
- **Sticky** — a newly registered observer immediately receives the current value.

**Limitations** (why teams moved to StateFlow): main-thread-only updates via `setValue`, no built-in operators (`map`/`switchMap` are limited), and a Java/Android dependency that doesn't fit Kotlin-multiplatform or pure-coroutine codebases.

**📚 Reference:** [What is LiveData in Android?](https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineer-tech-activity-7371905311613509632-dGPM)

---

## 7. StateFlow vs LiveData

Both are observable state holders. **`StateFlow`** is a Kotlin coroutines `Flow` that always has a value and emits updates to collectors; **`LiveData`** is an Android lifecycle-aware holder. In modern apps, `StateFlow` is generally preferred for ViewModel state.

| Aspect | StateFlow | LiveData |
|--------|-----------|----------|
| Library | Kotlin Coroutines (`kotlinx.coroutines`) | Android (`androidx.lifecycle`) |
| Lifecycle awareness | Not built-in; use `repeatOnLifecycle` / `flowWithLifecycle` | Built-in (active observers only) |
| Initial value | **Required** | Optional (can be empty) |
| Threading | Any dispatcher | `setValue` main thread only |
| Operators | Full Flow operators (`map`, `combine`, `filter`, …) | Limited (`map`, `switchMap`, `distinctUntilChanged`) |
| Equality / conflation | Emits only on value change (`equals`) | Emits on every `setValue`/`postValue` |
| Platform | KMP-friendly, pure Kotlin | Android only |

**Correct StateFlow collection (lifecycle-safe):**

```kotlin
viewLifecycleOwner.lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state -> render(state) }
    }
}
```

`repeatOnLifecycle(STARTED)` cancels collection when the UI goes below `STARTED` and restarts it on return — this is what gives StateFlow LiveData-like lifecycle safety. Without it, a raw `collect` keeps running in the background and can waste resources or update a stopped UI.

**When to use which:**
- **StateFlow** — new Kotlin codebases, Compose, KMP, when you want Flow operators or off-main-thread updates.
- **LiveData** — legacy codebases, Java interop, or when you want automatic lifecycle handling without `repeatOnLifecycle`.

For one-off events (navigation, snackbars), prefer a `Channel`/`SharedFlow` over StateFlow (which conflates and replays). In Compose, expose state via `StateFlow` and consume with `collectAsStateWithLifecycle()`.

**📚 Reference:** [StateFlow vs LiveData](https://www.linkedin.com/posts/amit-shekhar-iitbhu_outcomeschool-softwareengineer-tech-activity-7316320147646885888-ZpyD)

---

## 8. How is LiveData different from ObservableField?

Both can drive Data Binding, but they differ fundamentally:

| Aspect | LiveData | ObservableField |
|--------|----------|-----------------|
| Library | `androidx.lifecycle` | Data Binding (`androidx.databinding`) |
| Lifecycle aware | **Yes** — observers tied to a `LifecycleOwner` | **No** — observers are not lifecycle aware |
| Memory-leak safety | Auto-removes observers on `DESTROYED` | You must remove `OnPropertyChangedCallback` manually |
| Typical use | ViewModel state exposed to UI / Data Binding | Older Data Binding fields bound directly in XML |
| Threading | `setValue` main thread; `postValue` from background | Set from any thread (no built-in lifecycle guard) |
| Survives config change | Yes, with a ViewModel | Yes, with a ViewModel, but no lifecycle gating |

`ObservableField<T>` (and `ObservableInt`, `ObservableBoolean`, etc.) was introduced with the Data Binding library to make individual fields observable in XML without implementing the full `Observable`/`BaseObservable` notification boilerplate.

**Bottom line:** Use **LiveData** (or **StateFlow**) for UI state because of lifecycle awareness and leak safety. `ObservableField` is mostly relevant in older Data Binding code; new code uses StateFlow + `collectAsStateWithLifecycle`/Data Binding's `LiveData` support, or Compose state.

**📚 Reference:** [LiveData overview](https://developer.android.com/topic/libraries/architecture/livedata)

---

## 9. setValue vs postValue in LiveData

Both update the value held by `MutableLiveData`, but they differ in threading and timing.

```kotlin
val data = MutableLiveData<Int>()

// Main thread ONLY — synchronous, updates immediately
data.setValue(5)      // Kotlin: data.value = 5

// Any thread — posts a task to the main thread
data.postValue(10)
```

| | `setValue()` | `postValue()` |
|--|--------------|---------------|
| Thread | Main thread only (throws if called off-main) | Any thread |
| Timing | Synchronous; value set immediately | Asynchronous; dispatched to main thread later |
| Multiple rapid calls | Each one applies | **Only the last value** delivered if main thread hasn't yet run |

**The conflation gotcha with `postValue`:** If you call `postValue` multiple times before the main thread runs the posted task, intermediate values are dropped — only the most recent value is dispatched to observers.

```kotlin
data.postValue("a")
data.postValue("b")
// Observer may only ever see "b"
```

**Rule of thumb:** Use `setValue` when you're already on the main thread; use `postValue` from a background thread. (With coroutines, prefer switching to `Dispatchers.Main` and using `setValue`, or use StateFlow whose `.value` setter is thread-safe and not subject to this conflation.)

**📚 Reference:** [setValue() vs postValue() in LiveData](https://www.linkedin.com/posts/outcomeschool_softwareengineer-androiddev-android-activity-7274283452563107840-SXXl)

---

## 10. Explain WorkManager and its use cases

**`WorkManager`** is the recommended Jetpack solution for **persistent**, **deferrable** background work that must run reliably even if the app exits or the device restarts. It chooses the best underlying scheduler (`JobScheduler`, or its own internal mechanisms) based on the API level and device, so you write the logic once.

**Define a Worker:**

```kotlin
class UploadWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        return try {
            val fileUri = inputData.getString("uri") ?: return Result.failure()
            uploadFile(fileUri)
            Result.success(workDataOf("status" to "done"))
        } catch (e: IOException) {
            Result.retry() // re-runs per the backoff policy
        }
    }
}
```

**One-time work with constraints:**

```kotlin
val constraints = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.UNMETERED)
    .setRequiresCharging(true)
    .build()

val request = OneTimeWorkRequestBuilder<UploadWorker>()
    .setConstraints(constraints)
    .setInputData(workDataOf("uri" to fileUri))
    .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 10, TimeUnit.SECONDS)
    .build()

WorkManager.getInstance(context).enqueue(request)
```

**Periodic work:**

```kotlin
val periodic = PeriodicWorkRequestBuilder<SyncWorker>(1, TimeUnit.HOURS).build()
WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "sync", ExistingPeriodicWorkPolicy.KEEP, periodic
)
```

**Use cases (deferrable, guaranteed work):**
- Uploading logs/analytics or media to a server.
- Periodically syncing local data with the network.
- Applying filters to images and saving them.
- Downloading content for offline use.
- Sending queued requests once connectivity returns.

**Capabilities:** constraints (network/charging/idle/storage), retry & exponential backoff, chaining (`beginWith().then()`), unique work, parallel work, `WorkManager.getWorkInfoByIdFlow(...)` for observing state, and `ExpeditedWork` (`setExpedited`) for important user-initiated tasks. Use `setForeground()` for long-running work that needs a foreground notification.

**When NOT to use it:** in-process immediate work (use coroutines), exact-alarm scheduling (use `AlarmManager`), or work tied to a precise UI moment.

**📚 Reference:** [Schedule tasks with WorkManager](https://developer.android.com/topic/libraries/architecture/workmanager)

---

## 11. Minimum repeat interval for a PeriodicWorkRequest

The **minimum repeat interval for a `PeriodicWorkRequest` is 15 minutes** (`PeriodicWorkRequest.MIN_PERIODIC_INTERVAL_MILLIS = 900,000 ms`). This matches the `JobScheduler` constraint. If you request a smaller interval, WorkManager silently **clamps it up to 15 minutes**.

```kotlin
// Runs at most every 15 minutes (anything smaller is clamped to 15 min)
val request = PeriodicWorkRequestBuilder<SyncWorker>(
    15, TimeUnit.MINUTES
).build()
```

**Flex interval:** You can also specify a *flex period* — the worker runs somewhere within the flex window at the end of each interval. The minimum flex is `MIN_PERIODIC_FLEX_MILLIS = 5 minutes`.

```kotlin
// Repeat every 1 hour; allowed to run during the last 15 minutes of each hour
val request = PeriodicWorkRequestBuilder<SyncWorker>(
    repeatInterval = 1, repeatIntervalTimeUnit = TimeUnit.HOURS,
    flexTimeInterval = 15, flexTimeIntervalUnit = TimeUnit.MINUTES
).build()
```

**Need work more often than 15 minutes?** WorkManager cannot do it for periodic work. Common workarounds: chain a `OneTimeWorkRequest` that re-enqueues itself with `setInitialDelay`, or use `AlarmManager` for exact timing — but neither is battery-friendly, and the OS may still defer them under Doze / App Standby.

**📚 Reference:** [PeriodicWorkRequest | API reference](https://developer.android.com/reference/kotlin/androidx/work/PeriodicWorkRequest), [Minimum interval discussion](https://www.linkedin.com/posts/outcomeschool_androiddev-kotlin-activity-7314239540246810625-Xu0T)

---

## 12. How does WorkManager guarantee task execution?

WorkManager guarantees that enqueued work **will eventually run** even across app exits, process death, and device reboots — provided the work's constraints are met. It achieves this through **persistence + a reliable rescheduling mechanism**.

**How it works:**

1. **Persistent storage (Room/SQLite):** Every enqueued `WorkRequest`, its constraints, state, and inputs are written to an internal Room database (`androidx.work.impl.WorkDatabase`). This is the source of truth, so work is not lost if the process is killed.

2. **Delegation to the OS scheduler:** On modern devices WorkManager uses `JobScheduler` to wake the app when constraints (network, charging, etc.) are satisfied. On older API levels it falls back to other internal schedulers.

3. **Reboot persistence:** WorkManager registers a `RescheduleReceiver` for `BOOT_COMPLETED`. After a reboot, it reads pending work from its database and reschedules it with the system.

4. **Retries with backoff:** If `doWork()` returns `Result.retry()` (e.g., a transient network error), WorkManager reschedules the work according to the configured `BackoffPolicy` (LINEAR or EXPONENTIAL) and delay.

5. **Constraint enforcement:** Work runs only when its constraints hold; otherwise it stays pending until they are met.

**What the guarantee is NOT:**
- It is **not** an *exact-time* guarantee. Doze mode, App Standby buckets, and battery optimizations can defer execution. For exact timing use `AlarmManager`.
- It is scoped to the app being able to run; if the user force-stops the app or the OEM aggressively kills it, execution can be blocked until the app is launched again.

In short: **persistence (survives process death & reboot) + OS-level scheduling + retry/backoff** is what makes WorkManager's execution *guaranteed* in the "eventually, when constraints allow" sense.

**📚 Reference:** [How does WorkManager guarantee task execution?](https://www.linkedin.com/posts/outcomeschool_how-does-workmanager-guarantee-task-execution-activity-7320301933691310080-GpCR)

---

## 13. Serializable vs Parcelable — which is best?

Both let you pass objects between Android components (e.g., through an `Intent` or `Bundle`), but they work very differently.

| | Serializable | Parcelable |
|--|--------------|------------|
| Origin | Standard Java (`java.io.Serializable`) | Android-specific (`android.os.Parcelable`) |
| Mechanism | Reflection-based; JVM auto-serializes | You define how to flatten/unflatten (or use `@Parcelize`) |
| Performance | Slow; heavy reflection; lots of GC garbage | Fast; designed for Android IPC |
| Boilerplate | Marker interface — almost none | More, but **`kotlin-parcelize` removes it** |
| Best for | Quick/rare use, simple POJOs, persistence | Passing data between components — the **recommended** choice |

**Parcelable with `@Parcelize` (recommended):**

```kotlin
// build.gradle: id("kotlin-parcelize")
@Parcelize
data class User(
    val id: String,
    val name: String,
    val age: Int
) : Parcelable
```

Passing it:

```kotlin
intent.putExtra("user", user)
val user = intent.getParcelableExtra("user", User::class.java) // API 33+ overload
```

**Serializable equivalent:**

```kotlin
data class User(val id: String, val name: String) : Serializable
```

**Which is best?** **Parcelable** is the recommended approach in Android because it is significantly faster (no reflection) and produces less garbage — important on mobile. With the `kotlin-parcelize` plugin and `@Parcelize`, the historical boilerplate disadvantage is gone, so there is little reason to prefer `Serializable`. Use `Serializable` only when you control no Android code (e.g., a shared pure-Java/Kotlin module) or for occasional, non-performance-critical persistence.

**📚 Reference:** [Parcelable vs Serializable](https://outcomeschool.com/blog/parcelable-vs-serializable)

---

## 14. Why is Bundle used for passing data instead of a Map?

A `Bundle` is a key-value container, so it looks similar to a `Map`, but it exists for reasons a plain `Map` cannot satisfy:

1. **IPC / cross-process serialization.** When data crosses process boundaries (an `Intent` to another app, an Activity restart handled by `system_server`, a `Service` binding), it must be **flattened into a `Parcel`**. `Bundle` is built on the `Parcelable` machinery and knows how to marshal/unmarshal itself efficiently. A generic `Map` has no defined, efficient wire format for IPC.

2. **Type safety constrained to parcelable/serializable values.** A `Bundle` only accepts types it knows how to serialize (primitives, `String`, `Parcelable`, `Serializable`, arrays, etc.). This restriction is what makes safe transport possible. A `Map<String, Any>` could hold arbitrary non-serializable objects (e.g., a live `View` or socket), which could not be transported and would fail or leak.

3. **Lazy unmarshalling / performance.** `Bundle` can keep its contents in serialized (`Parcel`) form and only deserialize values when you read them — useful for performance and memory.

4. **Framework contract.** The Android framework's APIs (`onSaveInstanceState(Bundle)`, `Intent` extras, `Fragment.arguments`, broadcast extras) are all defined in terms of `Bundle`. It is the standard, expected currency for component-to-component communication.

In short: a `Bundle` is a **Parcelable-aware, IPC-safe, type-restricted** map. A plain `Map` is just in-memory data with no serialization guarantees, so it cannot reliably cross processes or survive process death.

**📚 Reference:** [Parcelables and bundles](https://developer.android.com/guide/components/activities/parcelables-and-bundles)

---

## 15. How do you troubleshoot a crashing application?

A systematic approach:

**1. Reproduce & capture the stack trace.**
- Use **Logcat** in Android Studio; filter by your package and the `E` (Error) level. A crash logs `FATAL EXCEPTION` plus the full stack trace and the exception type (e.g., `NullPointerException`, `IllegalStateException`, `OutOfMemoryError`).
  ```
  adb logcat -v time *:E
  ```
- Read **top-down**: the first line of your own code in the trace is usually the culprit (`Caused by:` lines point at the root cause).

**2. Use crash-reporting tools in production.**
- **Android Vitals** (Google Play Console) surfaces crash rate, ANR rate, and clustered stack traces from real users.
- **Firebase Crashlytics** gives real-time crash reports with stack traces, device/OS breakdowns, breadcrumbs (custom logs/keys), and the ability to log non-fatals.

**3. Categorize the crash.**
- **NPE** — missing null checks; lean on Kotlin null-safety.
- **ANR (Application Not Responding)** — main thread blocked > 5s; move work off the main thread (coroutines).
- **OutOfMemory** — leaks (use **LeakCanary**), large bitmaps, unbounded caches.
- **Lifecycle/state crashes** — `IllegalStateException` from committing fragment transactions after `onSaveInstanceState`, or touching a destroyed view.

**4. Add context for hard-to-reproduce crashes.**
- Log breadcrumbs and custom keys (user flow, feature flags) in Crashlytics.
- Use **Strict Mode** in debug to catch disk/network on the main thread.

**5. Fix, verify, and prevent regressions.**
- Write a unit/instrumentation test reproducing the bug.
- Add defensive checks and proper error handling (`try/catch`, `runCatching`, sealed `Result` states).

**📚 Reference:** [Diagnose and fix crashes](https://developer.android.com/topic/performance/vitals/crash)

---

## 16. Explain the Android push notification system

Android push notifications are delivered through **Firebase Cloud Messaging (FCM)**, Google's messaging service that sits on top of a persistent connection between the device and Google's servers.

**End-to-end flow:**

1. **Registration.** On first launch the app's FCM SDK requests a unique **registration token** from FCM servers. This token identifies that specific app instance on that device.
2. **Token upload.** The app sends this token to **your app server**, which stores it (mapped to the user).
3. **Send.** When your server wants to notify a user, it sends a message request (with the target token, and either a `notification` and/or `data` payload) to the **FCM backend** over HTTP v1, authenticated with service-account credentials.
4. **Routing & delivery.** FCM routes the message to the device over the **persistent socket connection** maintained by Google Play services. If the device is offline, FCM queues the message and delivers it when the device reconnects (subject to TTL).
5. **On-device handling.** Google Play services receives the message and either displays it automatically (notification messages while the app is in the background) or wakes your app's `FirebaseMessagingService.onMessageReceived()` (data messages, or foreground).

```kotlin
class MyMessagingService : FirebaseMessagingService() {

    override fun onNewToken(token: String) {
        sendTokenToServer(token) // refresh on your backend
    }

    override fun onMessageReceived(message: RemoteMessage) {
        message.notification?.let { showNotification(it.title, it.body) }
        // message.data holds custom key/value payload
    }
}
```

**Message types:**
- **Notification message** — FCM displays it for you when the app is backgrounded; `onMessageReceived` only fires in the foreground.
- **Data message** — always delivered to `onMessageReceived`, giving you full control over display and processing.

**Modern requirements:** Notifications require a **`NotificationChannel`** (Android 8.0+) and the runtime **`POST_NOTIFICATIONS`** permission (Android 13+). For battery-sensitive delivery you can set message priority and use **expedited / high-priority** messages sparingly.

**📚 Reference:** [How does the Android push notification system work?](https://youtu.be/810IFG2sWlc)

---

## 17. What is AAPT?

**AAPT** stands for **Android Asset Packaging Tool**. It is a build tool (part of the Android SDK build tools) that **compiles and packages your app's resources**. The current version is **AAPT2** (a two-step compile + link tool that enables incremental, faster builds), invoked automatically by the Android Gradle Plugin.

**What AAPT2 does:**

1. **Compile phase** — compiles each resource file (layouts, drawables, `values/*.xml`, etc.) into an intermediate binary `.flat` format. Because each file compiles independently, only changed resources are recompiled (incremental builds).
2. **Link phase** — merges all compiled resources, resolves references, generates the **`R.java`** class (resource ID constants), produces the binary **`resources.arsc`** table, and creates the (initial) APK with binary XML.

**Why it matters:**
- It turns human-readable XML/resources into the compact binary forms the Android runtime can load quickly.
- It generates the `R` class so you can reference resources in code as `R.layout.activity_main`, `R.string.app_name`, etc.
- It validates resources and enforces things like resource naming and configuration qualifiers (e.g., `values-night`, `drawable-xxhdpi`).

You rarely call AAPT2 directly — Gradle handles it — but interviewers ask to confirm you understand how XML resources become the `R` class and `resources.arsc`.

**📚 Reference:** [AAPT2](https://developer.android.com/studio/command-line/aapt2)

---

## 18. FlatBuffers vs JSON

Both are data-serialization formats, but they target different priorities. **JSON** is a human-readable text format; **FlatBuffers** is a high-performance binary format (from Google) designed for zero-copy access.

| Aspect | JSON | FlatBuffers |
|--------|------|-------------|
| Format | Text, human-readable | Binary |
| Parsing | Must fully parse/deserialize into objects | **Zero-copy** — read fields directly from the buffer; no parsing step |
| Memory | Allocates many objects; GC pressure | Almost no allocations; access in place |
| Speed | Slower (parse + allocate) | Very fast access; great for large/frequent reads |
| Schema | Schema-less (flexible) | Schema-defined (`.fbs` file → generated code) |
| Size | Larger, verbose | Compact |
| Debuggability | Easy to read/edit by hand | Not human-readable |
| Setup | Trivial (built-in / Gson / Moshi / kotlinx.serialization) | Requires `flatc` compiler + generated classes |

**Why FlatBuffers on Android:** Because you can access serialized data **without parsing or allocating**, it avoids GC churn — valuable for performance-critical paths, large datasets, game state, or frequently-read cached data on mobile. The trade-off is a rigid schema, a build step, and no human readability.

**When to use which:**
- **JSON** — REST APIs, config, logging, anything that benefits from readability and flexibility, and where payloads are modest.
- **FlatBuffers** — performance-critical local data, large/complex structures read very often, latency- and memory-sensitive code.

**📚 Reference:** [FlatBuffers documentation](https://flatbuffers.dev/)

---

## 19. HashMap, ArrayMap and SparseArray

All three are key-value containers, but `ArrayMap` and `SparseArray` (from `androidx.collection`) are **memory-optimized for Android's typically small collections**, trading some CPU for far less memory and fewer allocations.

**`HashMap<K, V>`**
- Standard Java map backed by an **array of bucket entries**; each entry is a heap object (`Node`) holding hash, key, value, next.
- **O(1)** average get/put.
- **Costly on memory:** auto-boxes primitive keys (e.g., `Integer`), allocates an `Entry` object per item, and pre-allocates a relatively large backing array. Wasteful for the small maps common in apps.

**`ArrayMap<K, V>`**
- Backed by **two arrays**: a sorted `int[]` of hashes and an `Object[]` of interleaved keys/values. Lookups use **binary search** over the hash array → **O(log n)**.
- Uses far less memory than `HashMap` (no per-entry `Node` objects) and shrinks its backing arrays when items are removed.
- Best for **small** maps (rule of thumb: up to a few hundred entries). For large or write-heavy maps, `HashMap` is faster.

**`SparseArray` (and `SparseIntArray`, `SparseLongArray`, `SparseBooleanArray`, `LongSparseArray`)**
- Maps **primitive `int` (or `long`) keys to objects/primitives** without autoboxing the key (and, for typed variants, without boxing the value either).
- Backed by parallel primitive `int[]` keys + value array, binary-searched → **O(log n)**.

```kotlin
// Avoids boxing Int keys into Integer objects
val cache = SparseArray<User>()
cache.put(42, user)
val u = cache.get(42)

// No boxing of either key or value
val counts = SparseIntArray()
counts.put(1, 100)
```

**Summary:**
- Use **`HashMap`** for large maps, non-`int` keys where speed matters, or general-purpose use.
- Use **`ArrayMap`** for small object-keyed maps to save memory.
- Use **`SparseArray`/`SparseIntArray`** when keys are `int`/`long` to avoid autoboxing and per-entry objects.

**📚 Reference:** [Optimization using ArrayMap and SparseArray](https://outcomeschool.com/blog/optimization-using-arraymap-and-sparsearray)

---

## 20. Advantages of SparseArray

`SparseArray` (and its siblings `SparseIntArray`, `SparseBooleanArray`, `SparseLongArray`, `LongSparseArray`) is an Android-specific structure that maps **primitive integer keys to values**. Its advantages over `HashMap<Integer, V>`:

1. **No autoboxing of keys.** A `HashMap` with `Integer` keys boxes every `int` into an `Integer` object. `SparseArray` stores keys in a primitive `int[]`, eliminating that allocation and the associated GC pressure.

2. **No per-entry `Entry`/`Node` objects.** `HashMap` allocates a wrapper node per mapping. `SparseArray` uses two parallel arrays (keys + values), so there are no entry objects — **much lower memory overhead**, which is exactly what matters on memory-constrained mobile devices.

3. **Better memory locality / smaller footprint.** Contiguous arrays are cache-friendly and compact compared to a scattered set of heap nodes.

4. **Typed variants avoid boxing values too.** `SparseIntArray` (int→int), `SparseBooleanArray` (int→boolean), `SparseLongArray` (int→long) store both keys and values as primitives — zero boxing on either side.

**Trade-offs / caveats:**
- Lookups are **O(log n)** (binary search) rather than `HashMap`'s O(1) average. For **small** collections (the common case in apps) this difference is negligible; for **very large** datasets `HashMap` can be faster.
- Keys must be `int` (or `long`). For arbitrary object keys, use `ArrayMap`/`HashMap`.

**Best for:** small-to-medium maps keyed by `int`/`long` — view IDs, positions, resource IDs, adapter caches — where reducing allocations and memory matters more than raw lookup speed.

**📚 Reference:** [Advantages of SparseArray in Android](https://www.linkedin.com/posts/amit-shekhar-iitbhu_outcomeschool-softwareengineer-tech-activity-7370662503024025600-rhVN)

---

## 21. What are Annotations?

**Annotations** are a form of **metadata** attached to code (classes, methods, fields, parameters, etc.). They do not change program logic by themselves; instead they provide information that can be consumed by the **compiler**, **build-time tools (annotation processors)**, or read at **runtime via reflection**.

**Built-in / common annotations:**
- `@Override`, `@Deprecated`, `@SuppressWarnings` (standard Java/Kotlin).
- Kotlin: `@JvmStatic`, `@JvmField`, `@JvmOverloads`, `@Throws`.
- AndroidX (`androidx.annotation`): `@Nullable`, `@NonNull`, `@StringRes`, `@DrawableRes`, `@ColorInt`, `@MainThread`, `@WorkerThread`, `@VisibleForTesting`, `@RequiresPermission`, `@IntDef`/`@StringDef` (typed enums without enum cost).

```kotlin
fun setTitle(@StringRes resId: Int) { /* lint flags non-string-resource args */ }

@MainThread
fun updateUi() { /* lint warns if called off the main thread */ }
```

**How annotations are processed (retention):**
- **`SOURCE`** — discarded by the compiler; used by lint/processors only (e.g., `@StringRes`).
- **`BINARY` (CLASS)** — kept in the `.class` file but not available via reflection.
- **`RUNTIME`** — retained and readable at runtime via reflection (e.g., Retrofit's `@GET`, Room's `@Entity`, Gson's `@SerializedName`).

**Why they matter in Android:** Libraries like **Room, Dagger/Hilt, Retrofit, Moshi, and Glide** use annotations heavily. Annotation processors (**KAPT**, and the newer **KSP** — Kotlin Symbol Processing) read these annotations at build time to **generate code** (DAOs, dependency graphs, adapters), removing boilerplate.

**📚 Reference:** [Creating custom annotations](https://outcomeschool.com/blog/creating-custom-annotations)

---

## 22. How to create a custom Annotation?

You declare a custom annotation with the `annotation class` keyword in Kotlin and configure it with meta-annotations.

**Meta-annotations:**
- **`@Target`** — where the annotation may be applied (`CLASS`, `FUNCTION`, `PROPERTY`, `FIELD`, `VALUE_PARAMETER`, etc.).
- **`@Retention`** — how long it is kept (`SOURCE`, `BINARY`, `RUNTIME`; default is `RUNTIME` in Kotlin).
- **`@Repeatable`**, **`@MustBeDocumented`** — optional behaviors.

**Example — a simple runtime annotation read by reflection:**

```kotlin
@Target(AnnotationTarget.CLASS, AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class Loggable(val tag: String = "DEFAULT")

@Loggable(tag = "Payment")
class PaymentService {
    @Loggable(tag = "charge")
    fun charge(amount: Int) { /* ... */ }
}

// Read it via reflection
fun inspect(obj: Any) {
    val ann = obj::class.annotations.filterIsInstance<Loggable>().firstOrNull()
    ann?.let { println("Tag = ${it.tag}") }
}
```

**Rules for annotation parameters:** allowed types are primitives, `String`, enums, other annotations, `KClass`, and arrays of these. Parameters can have default values (as above).

**Example — a source-retention marker consumed by an annotation processor (KSP):**

```kotlin
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.SOURCE)
annotation class GenerateBuilder
```

A KSP/KAPT processor would scan for `@GenerateBuilder`-annotated classes at compile time and generate a builder class. This is exactly how Room (`@Entity`, `@Dao`) and Hilt (`@HiltViewModel`, `@Inject`) work under the hood.

**📚 Reference:** [Creating custom annotations](https://outcomeschool.com/blog/creating-custom-annotations)

---

## 23. What is the Android Support Library? Why was it introduced?

The **Android Support Library** was a set of code libraries from Google that provided **backward-compatible versions** of newer framework APIs and added components not present in the platform framework, so apps could use modern features on older Android versions.

**Why it was introduced:**
- Android's framework APIs are tied to the OS version; a feature added in a new release isn't available on older devices.
- Device fragmentation meant developers had to support a wide range of API levels.
- The Support Library let you write against a single API and have it work across versions (e.g., `AppCompatActivity` brought modern `Toolbar`/Material theming to old versions; `Fragment`, `RecyclerView`, `ViewPager`, `CardView` were delivered as libraries).

**Examples:** `support-v4`, `appcompat-v7`, `RecyclerView`, `CardView`, `Design` (Material components), `ConstraintLayout`.

**What happened to it (the modern context):** In 2018, the Support Library was **refactored and rebranded into AndroidX** (the `androidx.*` namespace) as part of Jetpack. AndroidX uses **semantic, independent versioning** (each library versions separately), a clean package naming scheme, and replaces the old `android.support.*` packages. The **Support Library is deprecated** — all new development uses AndroidX, and tools like Android Studio's "Migrate to AndroidX" automate the rename via the `jetifier`.

So in interviews: the Support Library solved **backward compatibility and fragmentation**; today it lives on as **AndroidX/Jetpack**.

**📚 Reference:** [Android Support Library overview](https://www.linkedin.com/posts/outcomeschool_androiddev-activity-7275725268680392705-waUi)

---

## 24. What is Android Data Binding?

The **Data Binding Library** is a Jetpack support library that lets you bind UI components in **XML layouts** directly to data sources **declaratively**, reducing the glue code (`findViewById`, manual setters) you'd otherwise write.

**Enable it:**

```kotlin
// build.gradle (module)
android { buildFeatures { dataBinding = true } }
```

**Layout uses a `<layout>` root with a `<data>` block:**

```xml
<layout xmlns:android="http://schemas.android.com/apk/res/android">
    <data>
        <variable name="user" type="com.example.User" />
        <variable name="vm" type="com.example.UserViewModel" />
    </data>

    <LinearLayout ...>
        <TextView
            android:text="@{user.name}" ... />
        <Button
            android:onClick="@{() -> vm.onSave()}" ... />
    </LinearLayout>
</layout>
```

**Bind in code (a `Binding` class is generated from the layout name):**

```kotlin
val binding: ActivityMainBinding =
    DataBindingUtil.setContentView(this, R.layout.activity_main)
binding.user = user
binding.vm = viewModel
binding.lifecycleOwner = this // required so LiveData/StateFlow updates the UI
```

**Features:** expression language in XML (`@{...}`), two-way binding (`@={...}`), binding adapters (`@BindingAdapter` to define custom XML attributes), and observability via `LiveData`/`ObservableField`.

**Data Binding vs View Binding:**
- **View Binding** — only generates type-safe references to views (replaces `findViewById`); no expressions, lightweight, faster builds. Preferred when you just need null/type-safe view access.
- **Data Binding** — adds the declarative binding/expression layer and binding adapters, at the cost of build time and complexity.

**Modern guidance (2026):** For new projects, **Jetpack Compose** is the recommended UI toolkit and obviates XML data binding entirely. In View-based code, **View Binding** is the common choice; **Data Binding** is mainly seen in existing codebases or where two-way binding/binding adapters are genuinely useful.

**📚 Reference:** [Data Binding Library](https://developer.android.com/topic/libraries/data-binding)

---

## 25. SavedStateHandle (surviving process death in a ViewModel)

A `ViewModel` survives **configuration changes** (rotation) but **not process death** — if Android kills your app in the background to reclaim memory, the `ViewModel` is destroyed and recreated fresh when the user returns. `SavedStateHandle` is the bridge that lets a `ViewModel` persist small amounts of UI state across **both** cases, replacing the old `onSaveInstanceState(Bundle)` dance.

It's a key-value map backed by the saved-instance-state `Bundle`, injected automatically into the ViewModel constructor:

```kotlin
class SearchViewModel(
    private val state: SavedStateHandle
) : ViewModel() {

    // Direct get/set (survives process death):
    var query: String
        get() = state["query"] ?: ""
        set(value) { state["query"] = value }

    // As an observable StateFlow (recommended):
    val queryFlow: StateFlow<String> =
        state.getStateFlow("query", "")

    fun onQueryChanged(q: String) { state["query"] = q }
}
```

With Hilt, just add `@HiltViewModel` and inject it — Hilt supplies the `SavedStateHandle`. It also receives **navigation arguments** automatically (a destination's args land as keys in the handle), so it's the idiomatic way to read nav args in a ViewModel.

**What to store:** only small, serializable UI state (a query string, selected id, scroll position) — the same Bundle size limits (~1 MB, realistically keep it to KBs) apply. Large data belongs in a repository/DB; the handle stores the *key* to reload it.

**Why it matters:** A very common follow-up to "does ViewModel survive rotation?" is "what about process death?" — `SavedStateHandle` (plus the `getStateFlow`/nav-args integration) is the modern answer, and it shows you understand the full Android lifecycle, not just configuration changes.

**📚 Reference:** [Saved state module for ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel/viewmodel-savedstate)

---

## 26. Paging 3 (PagingSource, Pager, RemoteMediator)

The **Paging 3** library (`androidx.paging`) loads large datasets **incrementally** — a page at a time as the user scrolls — instead of loading everything into memory. It handles in-flight request de-duplication, retry/refresh, loading/error UI state, and integrates with coroutines/Flow, RxJava, and both RecyclerView and Compose.

Core pieces:
- **`PagingSource<Key, Value>`** — defines *how* to load one page (e.g. a network page or a DB query). You implement `load()`.
- **`Pager`** — configured with `PagingConfig` (page size, etc.); exposes a **`Flow<PagingData<T>>`**.
- **`PagingDataAdapter`** (RecyclerView) or **`collectAsLazyPagingItems()`** (Compose) — consumes `PagingData` and drives the UI, surfacing `LoadState` (loading/error/append/prepend).
- **`RemoteMediator`** — for the **offline-first** pattern: Room is the single source of truth, and the mediator fetches from the network to fill the database as the user pages.

```kotlin
class RepoPagingSource(private val api: Api) : PagingSource<Int, Repo>() {
    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, Repo> = try {
        val page = params.key ?: 1
        val repos = api.getRepos(page, params.loadSize)
        LoadResult.Page(
            data = repos,
            prevKey = if (page == 1) null else page - 1,
            nextKey = if (repos.isEmpty()) null else page + 1
        )
    } catch (e: IOException) { LoadResult.Error(e) }

    override fun getRefreshKey(state: PagingState<Int, Repo>) =
        state.anchorPosition?.let { state.closestPageToPosition(it)?.prevKey?.plus(1) }
}

class RepoViewModel(api: Api) : ViewModel() {
    val repos: Flow<PagingData<Repo>> =
        Pager(PagingConfig(pageSize = 20)) { RepoPagingSource(api) }
            .flow.cachedIn(viewModelScope)   // cache across config changes
}
```

`cachedIn(scope)` keeps loaded pages alive across configuration changes. Room can generate a `PagingSource` directly (return `PagingSource<Int, Entity>` from a `@Query` DAO method).

**Why it matters:** "How do you implement infinite scrolling / load a large list efficiently?" is a classic question. Paging 3 is the recommended answer; mentioning `PagingSource` + `Pager` + `cachedIn`, `LoadState`-driven UI, and `RemoteMediator` for offline-first signals current, production-grade knowledge.

**📚 Reference:** [Paging 3 overview](https://developer.android.com/topic/libraries/architecture/paging/v3-overview)

---

*End of guide. Answers are accurate as of 2026 and reflect current Jetpack recommendations (ViewModel + StateFlow, DataStore, Navigation, WorkManager, View Binding/Compose over Data Binding).*
