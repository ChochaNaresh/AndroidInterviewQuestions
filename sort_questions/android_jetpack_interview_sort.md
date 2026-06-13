# Android Jetpack & Core Android — Short Answers (Quick Revision)

> Condensed answers for rapid review before an interview. For full explanations and code see `android_jetpack_interview.md`.

---

## 1. What is Android Jetpack and why use it?

A suite of `androidx.*` libraries, tools, and architecture guidance (successor to the Support Library), grouped into Foundation, Architecture, Behavior, and UI. Used because it provides backward compatibility (unbundled from the OS), eliminates boilerplate, prevents lifecycle bugs/leaks, and underpins Google's recommended app architecture.

---

## 2. What is a ViewModel and how is it useful?

A lifecycle-aware class that stores and manages UI state, surviving configuration changes (rotation, etc.). Useful for separation of concerns, avoiding re-fetching data, testability, and `viewModelScope` for auto-cancelled coroutines. Never hold a reference to a View/Activity/Fragment context.

---

## 3. How does ViewModel work internally?

`ComponentActivity`/`Fragment` are `ViewModelStoreOwner`s holding a `ViewModelStore` (a `HashMap` of ViewModels). On a config change the store is retained via `onRetainNonConfigurationInstance()`/`getLastNonConfigurationInstance()`, so the same instances are handed to the new Activity. `onCleared()` is only called on permanent destruction (checked via `isChangingConfigurations()`). It does NOT survive process death — use `SavedStateHandle` for that.

---

## 4. What are Android Architecture Components?

The Jetpack subset for robust, testable apps: ViewModel, LiveData, Lifecycle, Room, WorkManager, Navigation, Paging, DataStore, SavedState. They form the layered recommended architecture (UI → domain → data) with separation of concerns and a single source of truth.

---

## 5. How do you share a ViewModel between Fragments? (SharedViewModel)

Scope a single ViewModel to the host Activity with `by activityViewModels()`, so both fragments get the same instance. Other options: `by navGraphViewModels(graphId)` for nav-graph scope, or `viewModels(ownerProducer = { requireParentFragment() })`. Benefits: loose coupling, survives config changes, single source of truth.

---

## 6. What is LiveData?

A lifecycle-aware observable data holder that only delivers updates to observers in an active (`STARTED`/`RESUMED`) state. Benefits: no leaks (auto-removes on `DESTROYED`), no crashes from stopped activities, always up to date, sticky. Limitations (why teams moved to StateFlow): main-thread-only `setValue`, limited operators, Android-only dependency.

---

## 7. StateFlow vs LiveData

- **StateFlow**: Kotlin coroutines, KMP-friendly, full Flow operators, any dispatcher, requires an initial value, conflates on `equals`. Lifecycle safety needs `repeatOnLifecycle`/`collectAsStateWithLifecycle`.
- **LiveData**: Android-only, built-in lifecycle awareness, limited operators, `setValue` main-thread only, optional initial value.
- Prefer StateFlow for new Kotlin/Compose code; LiveData for legacy/Java interop. For one-off events use `Channel`/`SharedFlow`.

---

## 8. How is LiveData different from ObservableField?

LiveData is lifecycle-aware and leak-safe (observers tied to a `LifecycleOwner`); `ObservableField` (Data Binding) is not lifecycle-aware and requires manual callback removal. Use LiveData/StateFlow for UI state; `ObservableField` is mostly legacy Data Binding code.

---

## 9. setValue vs postValue in LiveData

- `setValue()`: main thread only, synchronous, applies immediately.
- `postValue()`: any thread, asynchronous (posts to main thread); if called rapidly, only the last value is delivered (conflation).
Use `setValue` on the main thread, `postValue` from background. StateFlow's `.value` is thread-safe and avoids this.

---

## 10. Explain WorkManager and its use cases

The recommended Jetpack solution for persistent, deferrable, guaranteed background work that survives app exit and reboot. Supports constraints, retry/backoff, chaining, unique/periodic work, and expedited/foreground work. Use cases: uploading logs/media, periodic sync, downloading for offline. Not for immediate in-process work or exact timing.

---

## 11. Minimum repeat interval for a PeriodicWorkRequest

**15 minutes** (`MIN_PERIODIC_INTERVAL_MILLIS = 900,000 ms`); smaller values are clamped up. Minimum flex interval is 5 minutes. For more frequent work, self-re-enqueue a `OneTimeWorkRequest` or use `AlarmManager` (both less battery-friendly).

---

## 12. How does WorkManager guarantee task execution?

Through persistence (requests stored in an internal Room DB, surviving process death) plus reliable rescheduling: it delegates to `JobScheduler`, reschedules after reboot via a `BOOT_COMPLETED` receiver, retries with backoff on `Result.retry()`, and enforces constraints. It is "eventually, when constraints allow" — NOT exact-time (Doze/standby can defer it).

---

## 13. Serializable vs Parcelable — which is best?

**Parcelable** is recommended on Android: it's far faster (no reflection) and produces less garbage, designed for IPC. With `kotlin-parcelize` (`@Parcelize`) the boilerplate is gone. Use `Serializable` only for simple pure-Java/Kotlin modules or rare, non-performance-critical persistence.

---

## 14. Why is Bundle used for passing data instead of a Map?

A Bundle is a Parcelable-aware, IPC-safe, type-restricted map: it can marshal itself into a `Parcel` for cross-process transport, only accepts serializable types, supports lazy unmarshalling, and is the framework's standard contract (`onSaveInstanceState`, Intent extras, `Fragment.arguments`). A plain Map has no wire format and can't reliably cross processes.

---

## 15. How do you troubleshoot a crashing application?

- Reproduce and read the stack trace in Logcat (`FATAL EXCEPTION`, top-down, follow `Caused by:`).
- Use production crash tools: Android Vitals, Firebase Crashlytics.
- Categorize: NPE, ANR (main thread >5s), OOM (LeakCanary), lifecycle/`IllegalStateException`.
- Add breadcrumbs/custom keys; use StrictMode in debug.
- Fix, add a regression test, and defensive error handling.

---

## 16. Explain the Android push notification system

Delivered via **Firebase Cloud Messaging (FCM)**: app gets a registration token → uploads it to your server → server sends a message (notification/data payload) to FCM over HTTP v1 → FCM routes it over Google Play services' persistent socket → device displays it or wakes `FirebaseMessagingService.onMessageReceived()`. Notification messages auto-display when backgrounded; data messages always hit `onMessageReceived`. Requires a `NotificationChannel` (8.0+) and `POST_NOTIFICATIONS` permission (13+).

---

## 17. What is AAPT?

**Android Asset Packaging Tool** — compiles and packages app resources. AAPT2 works in two phases: compile (each resource → binary `.flat`, enabling incremental builds) and link (merges resources, generates the `R` class and binary `resources.arsc`). Run automatically by the Android Gradle Plugin.

---

## 18. FlatBuffers vs JSON

- **JSON**: text, human-readable, schema-less, must fully parse/allocate; best for REST APIs, config, modest payloads.
- **FlatBuffers**: compact binary, schema-defined (`.fbs` + `flatc`), zero-copy access (no parsing/allocation); best for performance-critical, large, frequently-read local data. Trade-off: rigid schema, build step, not human-readable.

---

## 19. HashMap, ArrayMap and SparseArray

- **HashMap**: standard, O(1) average, but heavy memory (boxes primitive keys, per-entry node objects). Best for large/general maps.
- **ArrayMap**: two arrays + binary search (O(log n)), much less memory; best for small object-keyed maps.
- **SparseArray** (and typed variants): maps primitive `int`/`long` keys without autoboxing; best when keys are int/long.

---

## 20. Advantages of SparseArray

Over `HashMap<Integer, V>`: no autoboxing of keys, no per-entry node objects, better memory locality/smaller footprint, and typed variants (`SparseIntArray`, etc.) avoid boxing values too. Trade-off: O(log n) lookup and keys must be int/long. Best for small-to-medium int-keyed maps (view IDs, positions).

---

## 21. What are Annotations?

Metadata attached to code, consumed by the compiler, build-time annotation processors, or runtime reflection. Examples: `@Override`, `@JvmStatic`, AndroidX `@Nullable`/`@StringRes`/`@MainThread`. Retention: `SOURCE` (lint/processors), `BINARY`/CLASS, `RUNTIME` (reflection). Libraries like Room, Hilt, Retrofit use them with processors (KAPT/KSP) to generate code.

---

## 22. How to create a custom Annotation?

Declare with `annotation class` and configure with meta-annotations `@Target` (where) and `@Retention` (how long). Parameters allow primitives, `String`, enums, `KClass`, annotations, and arrays.

```kotlin
@Target(AnnotationTarget.CLASS, AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class Loggable(val tag: String = "DEFAULT")
```

A `RUNTIME` annotation is read via reflection; a `SOURCE` one is consumed by a KSP/KAPT processor to generate code.

---

## 23. What is the Android Support Library? Why was it introduced?

A set of libraries providing backward-compatible versions of newer APIs (e.g. `AppCompatActivity`, `RecyclerView`) so apps could use modern features on older versions, solving fragmentation. In 2018 it was refactored/rebranded into **AndroidX** (`androidx.*`) with independent semantic versioning; the old Support Library is deprecated.

---

## 24. What is Android Data Binding?

A Jetpack library that binds XML UI components directly to data declaratively (via a `<layout>`/`<data>` block and `@{...}` expressions), reducing `findViewById` glue. Supports two-way binding (`@={...}`) and binding adapters. **View Binding** is lighter (only type-safe view references); Data Binding adds expressions/adapters. For new code, prefer Compose or View Binding.

---

## 25. SavedStateHandle (surviving process death in a ViewModel)

A key-value map (backed by the saved-instance `Bundle`) injected into a ViewModel constructor that persists small UI state across both config changes AND process death. Use `state["key"]` or `state.getStateFlow("key", default)`. It also receives navigation arguments automatically. Store only small serializable state (~KBs); large data belongs in a repository.

---

## 26. Paging 3 (PagingSource, Pager, RemoteMediator)

Loads large datasets incrementally as the user scrolls. Core pieces: `PagingSource` (implements `load()`), `Pager` + `PagingConfig` (exposes `Flow<PagingData<T>>`), `PagingDataAdapter`/`collectAsLazyPagingItems()` (consumes + surfaces `LoadState`), and `RemoteMediator` for offline-first with Room as source of truth. `cachedIn(scope)` keeps pages alive across config changes. The recommended answer for "infinite scrolling."
