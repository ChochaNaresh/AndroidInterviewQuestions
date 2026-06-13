# Android Architecture & Design Patterns — Short Answers (Quick Revision)

> Condensed answers for rapid review before an interview. For full explanations and code see `android_architecture_design_patterns_interview.md`.

---

# Part 1 — Application Architecture

## 1. Describe the architecture of your last app

Structure the answer around layers, data flow, and the libraries enforcing each boundary: UI layer (Compose + ViewModels exposing immutable `UiState` via `StateFlow`, UDF), optional domain layer (use cases), and a data layer (repositories as single source of truth over Retrofit/Room). Hilt wires it; feature + core modules split it. Then justify it with separation of concerns, testability, and config-change survival.

---

## 2. Why use MVP / MVVM architectures?

To keep logic out of Activities/Fragments (hard to test, complex lifecycle). Benefits: separation of concerns, reusable/testable plain-class logic, less duplication, easier maintenance, and fast JVM unit tests without instrumentation.

---

## 3. Describe MVVM

MVVM separates UI from logic via a ViewModel that exposes observable state. Model = data/business logic; View = Activity/Fragment/Composable that observes state and forwards events; ViewModel = holds/prepares state, handles actions, has **no reference to the View** (communication is via observation, so it survives config changes). Pros: great testability, less boilerplate than MVP, lifecycle-aware. Cons: ViewModels can bloat without a domain layer. The default for modern Android.

---

## 4. MVC vs MVP vs MVVM

- **MVC (Model-View-Controller)** — Controller updates Model and View; on Android the Activity is both View and Controller → god classes.
- **MVP (Model-View-Presenter)** — Presenter holds a **View interface** reference (1:1, bidirectional); good testability but lots of boilerplate; legacy.
- **MVVM (Model-View-ViewModel)** — ViewModel exposes observable state, **no** View reference; View observes (unidirectional); built-in config-change survival; recommended for new apps.

---

## 5. Why should the View be an interface in MVP?

So the Presenter depends on an abstraction, not a concrete Activity/Fragment. This decouples logic from the framework, lets you swap the real View for a fake/mock in tests, and honours the Dependency Inversion Principle — making the Presenter unit-testable without an emulator.

---

## 6. What is the use of interfaces in a Presenter?

They honour Dependency Inversion (depend on abstractions). Benefits: Presenter is testable against a mock/fake View, the concrete View can be swapped without touching the Presenter, the contract is explicit, and the View can be dependency-injected.

---

## 7. MVI (Model-View-Intent)

A unidirectional, state-driven evolution of MVVM: the UI is a single immutable state produced by reducing a stream of user **intents**. Flow: View emits Intent → processed into a Result → pure Reducer produces new State → View renders. Pros: single source of truth, predictable/debuggable, highly testable. Cons: more boilerplate, heavier for trivial screens. Use for complex, interdependent state.

---

## 8. Clean Architecture and the dependency rule

Concentric layers (Entities → Use Cases → Interface Adapters → Frameworks/Drivers) keep business rules independent of frameworks/UI/DB. **The Dependency Rule:** source-code dependencies point strictly inward; inner layers declare interfaces, outer layers implement them (Dependency Inversion). Android mapping: `domain` (pure Kotlin entities + use cases + repo interfaces), `data` (implementations), `presentation` (ViewModels/UI). Pros: testable, swappable layers. Cons: boilerplate/over-engineering for small apps.

---

## 9. Google's recommended layered architecture (UI / Domain / Data, UDF)

- **UI layer** — state holders (`ViewModel`) expose UI state; UI is a function of state.
- **Domain layer** (optional) — reusable single-responsibility use cases.
- **Data layer** — repositories are the **single source of truth** over data sources.

**UDF:** state flows down (data → ViewModel → UI), events flow up (UI → ViewModel). Combined with single source of truth, this makes state predictable and testable.

---

## 10. Software Architecture vs Software Design

Architecture is high-level, system-wide, strategic, and expensive to change (layers, modules, tech choices, scalability). Design is low-level, localized, tactical, cheaper to change (class responsibilities, methods, applying patterns). Analogy: architecture is the building's blueprint; design is how each room is laid out.

---

## 11. Benefits of Multi-Module Architecture

- Faster (parallel/incremental) builds — the biggest win.
- Strict encapsulation via `api`/`implementation` visibility.
- Better separation of concerns and reusability of `core` modules.
- Parallel team work with fewer conflicts; dynamic feature delivery; better testability and clear ownership.

Trade-off: more Gradle config and navigation strategy needed.

---

## 12. Multi-Module Project: Why and When?

**Why:** build speed, encapsulation, reusability, parallel dev, dynamic delivery, enforced boundaries. **When:** growing codebase/team, multiple independent features, need to enforce layers at compile time, dynamic delivery, or multiple teams/targets. **When NOT (yet):** small apps, prototypes, tiny teams. Pragmatic rule: start single-module, extract modules when build time/coupling/conflicts hurt.

---

# Part 2 — Design Patterns

## 13. Builder

Constructs a complex object step by step, avoiding telescoping constructors. Android examples: `AlertDialog.Builder`, `NotificationCompat.Builder`, `Retrofit.Builder`, `OkHttpClient.Builder`, `WorkRequest.Builder`.

---

## 14. Singleton

Ensures a class has one instance with a global access point. In Kotlin use `object` (thread-safe, lazy); for parameterized singletons use double-checked locking with `@Volatile`. Android: `Application`, Room DB, Retrofit/OkHttp clients (often via Hilt `@Singleton`). Caveat: global mutable state hurts testability — prefer DI-scoped singletons and use `applicationContext` to avoid leaks.

---

## 15. Factory (and Abstract Factory)

**Factory Method** lets implementations decide which class to instantiate; **Abstract Factory** creates families of related objects. Android: `ViewModelProvider.Factory`, `LayoutInflater.Factory`, `BitmapFactory`, Retrofit's `Converter.Factory`/`CallAdapter.Factory`, `ThreadFactory`.

---

## 16. Observer

Defines a one-to-many dependency so observers are notified automatically when the subject changes — the foundation of reactive UI. Android: `LiveData`, `StateFlow`/`SharedFlow`, RxJava, listeners, `ContentObserver`, `BroadcastReceiver`, Compose recomposition.

---

## 17. Repository

Mediates between domain and data layers, providing a clean data API and acting as the **single source of truth**. Hides where data comes from (network/DB/cache), decouples ViewModels/use cases from data sources, and centralizes caching — making sources easy to swap or mock for testing.

---

## 18. Adapter

Converts one interface into another that clients expect (a wrapper) so incompatible classes work together. Android: `RecyclerView.Adapter`/`ListAdapter`, `ArrayAdapter`, `FragmentStateAdapter`, Retrofit's `CallAdapter` (adapts `Call` to coroutines/RxJava).

---

## 19. Facade

Provides a unified, simplified interface over a complex subsystem, hiding its complexity. Android: `Retrofit` (over OkHttp + converters + adapters), `Glide.with(...)`, `ContextCompat`/`NotificationCompat`, and a `Repository` over multiple data sources.

---

## 20. Dependency Injection

A class's dependencies are supplied from outside rather than constructed internally — an application of Inversion of Control / DIP. Benefits: decoupling, easy testing with fakes, smaller classes, centralized wiring. Prefer constructor injection. Android: **Hilt** (on Dagger) is recommended; Koin is a service-locator-style alternative. DI (Dependency Injection) pushes dependencies in; a Service Locator pulls them (hides deps, harder to test).

---

## 21. Strategy

Defines a family of interchangeable algorithms, encapsulated and swappable at runtime; favours composition over inheritance and avoids large conditionals. Android: `RecyclerView.LayoutManager`, `Interpolator`s, `Comparator`, Glide caching/transformations. In Kotlin, strategies are often just higher-order functions.

---

## 22. Design patterns used in Android

Quick map: Builder (`AlertDialog.Builder`), Singleton (`Application`, Room), Factory (`ViewModelProvider.Factory`), Observer (`LiveData`/`Flow`), Adapter (`RecyclerView.Adapter`), Facade (`Retrofit`, `Glide`), Strategy (`LayoutManager`), Decorator (`ContextWrapper`, OkHttp `Interceptor`), Template Method (lifecycle callbacks), Command (`Runnable`/`Worker`), Memento (`onSaveInstanceState`), Proxy (Binder/AIDL, Retrofit), Composite (`View`/`ViewGroup`), DI (Hilt).

---

## 23. Kotlin Optional Parameters vs Builder Pattern

Builder mainly solves Java's telescoping-constructor problem; Kotlin's **default + named arguments** solve it at the language level, so a Builder is often unnecessary. Still reach for a Builder for: Java interop, stepwise/conditional construction, validation/transformation in `build()`, or a stable public library API. Rule of thumb: prefer default/named args; use Builder for Java consumers or incremental construction.

---

## 24. Examples of the Observer pattern in Android

`LiveData` (`observe`), Kotlin Flow (`StateFlow`/`SharedFlow` + `collect`), RxJava (`subscribe`), listeners/callbacks (`OnClickListener`, `TextWatcher`, `SensorEventListener`), `BroadcastReceiver`, `ContentObserver`, `OnSharedPreferenceChangeListener`, lifecycle observers, Compose state reads, and `WorkManager` work-info LiveData.

---

## 25. Design patterns in the Retrofit source code

- **Builder** — `Retrofit.Builder()`.
- **Facade** — `Retrofit` over OkHttp + converters + adapters.
- **Proxy (Dynamic Proxy)** — `retrofit.create()` returns a `java.lang.reflect.Proxy` whose `InvocationHandler` builds requests.
- **Factory** — `Converter.Factory`, `CallAdapter.Factory`.
- **Adapter** — `CallAdapter` adapts `Call<T>` to other return types.
- **Strategy / Decorator** — pluggable converters/adapters and OkHttp `Interceptor` chain.

---

## 26. Design patterns in Glide

- **Builder/fluent** — `Glide.with().load().into()`.
- **Facade** — hides decoding/caching/threading.
- **Singleton** — process-wide `Glide` engine.
- **Strategy** — `DiskCacheStrategy`, transformations, `ModelLoader`s.
- **Factory** — `ModelLoaderFactory`/`ResourceDecoder`.
- **Object Pool** — `BitmapPool` reuses bitmap memory.
- **Observer** — ties requests to host lifecycle.

---

## 27. Design patterns used in AOSP

Singleton (system services), Factory (`BitmapFactory`, `LayoutInflater.Factory`), Builder (`Notification.Builder`, `Uri.Builder`), Observer (`BroadcastReceiver`, `ContentObserver`), Adapter, Composite (`View`/`ViewGroup` tree), Decorator (`ContextWrapper`), Template Method (lifecycle callbacks), Proxy/Stub (Binder & AIDL, `Stub.asInterface()`), Command (`Handler`/`Message`), Memento (`onSaveInstanceState`), Facade (`LocationManager`, `ConnectivityManager`).
