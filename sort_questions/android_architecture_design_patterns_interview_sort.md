# Android Architecture & Design Patterns — Short Answers (Quick Revision)

> Condensed answers for rapid review before an interview. For full explanations and code see `android_architecture_design_patterns_interview.md`.

---

# Part 1 — Application Architecture

## 1. Describe the architecture of your last app

**Clean Architecture** means a software design philosophy that separates concerns into independent layers (entities, use cases, presenters/controllers, and UI/data) to achieve high testability and maintainability.

Structure the answer around layers, data flow, and the libraries enforcing each boundary: UI layer (Compose + ViewModels exposing immutable `UiState` via `StateFlow`, UDF), optional domain layer (use cases), and a data layer (repositories as single source of truth over Retrofit/Room). Hilt wires it; feature + core modules split it. Then justify it with separation of concerns, testability, and config-change survival.

---



## 2. Why use MVP / MVVM architectures?

**MVP and MVVM architectures** means presentation-layer patterns designed to decouple business and presentation logic from Android framework classes like Activities and Fragments.

To keep logic out of Activities/Fragments (hard to test, complex lifecycle). Benefits: separation of concerns, reusable/testable plain-class logic, less duplication, easier maintenance, and fast JVM unit tests without instrumentation.

---



## 3. Describe MVVM

**MVVM (Model-View-ViewModel)** means a presentation-layer architectural pattern that separates the UI (View) from business and presentation logic (ViewModel) using state observation.

MVVM separates UI from logic via a ViewModel that exposes observable state. Model = data/business logic; View = Activity/Fragment/Composable that observes state and forwards events; ViewModel = holds/prepares state, handles actions, has **no reference to the View** (communication is via observation, so it survives config changes). Pros: great testability, less boilerplate than MVP, lifecycle-aware. Cons: ViewModels can bloat without a domain layer. The default for modern Android.

---



## 4. MVC vs MVP vs MVVM

**MVC vs MVP vs MVVM** means the comparison of presentation patterns where MVC uses a Controller, MVP uses a Presenter interacting with a View interface, and MVVM uses a ViewModel with data binding.

- **MVC (Model-View-Controller)** — Controller updates Model and View; on Android the Activity is both View and Controller → god classes.
- **MVP (Model-View-Presenter)** — Presenter holds a **View interface** reference (1:1, bidirectional); good testability but lots of boilerplate; legacy.
- **MVVM (Model-View-ViewModel)** — ViewModel exposes observable state, **no** View reference; View observes (unidirectional); built-in config-change survival; recommended for new apps.

---



## 5. Why should the View be an interface in MVP?

**MVP View interface** means an abstraction layer that allows the Presenter to update the UI without depending directly on concrete Activity or Fragment classes.

So the Presenter depends on an abstraction, not a concrete Activity/Fragment. This decouples logic from the framework, lets you swap the real View for a fake/mock in tests, and honours the Dependency Inversion Principle — making the Presenter unit-testable without an emulator.

---



## 6. What is the use of interfaces in a Presenter?

**Presenter interfaces** means abstraction boundaries that decouple the View from the Presenter, enabling independent development and testing of both components.

They honour Dependency Inversion (depend on abstractions). Benefits: Presenter is testable against a mock/fake View, the concrete View can be swapped without touching the Presenter, the contract is explicit, and the View can be dependency-injected.

---



## 7. MVI (Model-View-Intent)

**MVI (Model-View-Intent)** means a unidirectional, state-driven presentation architectural pattern where user actions are represented as intents that emit new UI state models.

A unidirectional, state-driven evolution of MVVM: the UI is a single immutable state produced by reducing a stream of user **intents**. Flow: View emits Intent → processed into a Result → pure Reducer produces new State → View renders. Pros: single source of truth, predictable/debuggable, highly testable. Cons: more boilerplate, heavier for trivial screens. Use for complex, interdependent state.

---



## 8. Clean Architecture and the dependency rule

**Clean Architecture** means a software design philosophy that organises code into concentric layers (entities, use cases, controllers, and UI) with dependency directions pointing strictly inward.

Concentric layers (Entities → Use Cases → Interface Adapters → Frameworks/Drivers) keep business rules independent of frameworks/UI/DB. **The Dependency Rule:** source-code dependencies point strictly inward; inner layers declare interfaces, outer layers implement them (Dependency Inversion). Android mapping: `domain` (pure Kotlin entities + use cases + repo interfaces), `data` (implementations), `presentation` (ViewModels/UI). Pros: testable, swappable layers. Cons: boilerplate/over-engineering for small apps.

---



## 9. Google's recommended layered architecture (UI / Domain / Data, UDF)

**Google's recommended layered architecture (UI / Domain / Data, UDF)** means an architectural style that structures applications into a mandatory UI layer, an optional Domain layer, and a mandatory Data layer governed by Unidirectional Data Flow.

- **UI layer** — state holders (`ViewModel`) expose UI state; UI is a function of state.
- **Domain layer** (optional) — reusable single-responsibility use cases.
- **Data layer** — repositories are the **single source of truth** over data sources.

**UDF:** state flows down (data → ViewModel → UI), events flow up (UI → ViewModel). Combined with single source of truth, this makes state predictable and testable.

---



## 10. Software Architecture vs Software Design

**Software architecture vs Software design** means the distinction between high-level, system-wide structures and boundaries (architecture), and low-level code-level implementation details and patterns (design).

Architecture is high-level, system-wide, strategic, and expensive to change (layers, modules, tech choices, scalability). Design is low-level, localized, tactical, cheaper to change (class responsibilities, methods, applying patterns). Analogy: architecture is the building's blueprint; design is how each room is laid out.

---



## 11. Benefits of Multi-Module Architecture

**Multi-module architecture benefits** means the advantages of dividing a project into separate Gradle modules including faster build speeds, strict API boundaries, and modular test isolation.

- Faster (parallel/incremental) builds — the biggest win.
- Strict encapsulation via `api`/`implementation` visibility.
- Better separation of concerns and reusability of `core` modules.
- Parallel team work with fewer conflicts; dynamic feature delivery; better testability and clear ownership.

Trade-off: more Gradle config and navigation strategy needed.

---



## 12. Multi-Module Project: Why and When?

**Multi-module project structure** means dividing an application into separate Gradle modules to improve build speed, enforce boundaries, and enable code reuse.

**Why:** build speed, encapsulation, reusability, parallel dev, dynamic delivery, enforced boundaries. **When:** growing codebase/team, multiple independent features, need to enforce layers at compile time, dynamic delivery, or multiple teams/targets. **When NOT (yet):** small apps, prototypes, tiny teams. Pragmatic rule: start single-module, extract modules when build time/coupling/conflicts hurt.

---



# Part 2 — Design Patterns

## 13. Builder

**Builder pattern** means a creational design pattern that allows constructing complex objects step-by-step.

Constructs a complex object step by step, avoiding telescoping constructors. Android examples: `AlertDialog.Builder`, `NotificationCompat.Builder`, `Retrofit.Builder`, `OkHttpClient.Builder`, `WorkRequest.Builder`.

---



## 14. Singleton

**Singleton pattern** means a creational design pattern that ensures a class has only one instance while providing a global access point to it.

Ensures a class has one instance with a global access point. In Kotlin use `object` (thread-safe, lazy); for parameterized singletons use double-checked locking with `@Volatile`. Android: `Application`, Room DB, Retrofit/OkHttp clients (often via Hilt `@Singleton`). Caveat: global mutable state hurts testability — prefer DI-scoped singletons and use `applicationContext` to avoid leaks.

---



## 15. Factory (and Abstract Factory)

**Factory pattern** means a creational design pattern that defines an interface for creating objects, letting subclasses decide which concrete class to instantiate.

**Factory Method** lets implementations decide which class to instantiate; **Abstract Factory** creates families of related objects. Android: `ViewModelProvider.Factory`, `LayoutInflater.Factory`, `BitmapFactory`, Retrofit's `Converter.Factory`/`CallAdapter.Factory`, `ThreadFactory`.

---



## 16. Observer

**Observer pattern** means a behavioural design pattern where an object (subject) maintains a list of dependents (observers) and notifies them automatically of state changes.

Defines a one-to-many dependency so observers are notified automatically when the subject changes — the foundation of reactive UI. Android: `LiveData`, `StateFlow`/`SharedFlow`, RxJava, listeners, `ContentObserver`, `BroadcastReceiver`, Compose recomposition.

---



## 17. Repository

**Repository pattern** means a structural design pattern that mediates between the domain and data-mapping layers, abstracting data sources behind a clean API.

Mediates between domain and data layers, providing a clean data API and acting as the **single source of truth**. Hides where data comes from (network/DB/cache), decouples ViewModels/use cases from data sources, and centralizes caching — making sources easy to swap or mock for testing.

---



## 18. Adapter

**Adapter pattern** means a structural design pattern that converts the interface of a class into another interface that clients expect.

Converts one interface into another that clients expect (a wrapper) so incompatible classes work together. Android: `RecyclerView.Adapter`/`ListAdapter`, `ArrayAdapter`, `FragmentStateAdapter`, Retrofit's `CallAdapter` (adapts `Call` to coroutines/RxJava).

---



## 19. Facade

**Facade pattern** means a structural design pattern that provides a unified, simplified interface to a complex subsystem of classes.

Provides a unified, simplified interface over a complex subsystem, hiding its complexity. Android: `Retrofit` (over OkHttp + converters + adapters), `Glide.with(...)`, `ContextCompat`/`NotificationCompat`, and a `Repository` over multiple data sources.

---



## 20. Dependency Injection

**Dependency Injection** means a design pattern where an object's dependencies are supplied by an external assembler rather than the object instantiating them itself.

A class's dependencies are supplied from outside rather than constructed internally — an application of Inversion of Control / DIP. Benefits: decoupling, easy testing with fakes, smaller classes, centralized wiring. Prefer constructor injection. Android: **Hilt** (on Dagger) is recommended; Koin is a service-locator-style alternative. DI (Dependency Injection) pushes dependencies in; a Service Locator pulls them (hides deps, harder to test).

---



## 21. Strategy

**Strategy pattern** means a behavioural design pattern that defines a family of interchangeable algorithms and encapsulates each one to be swapped at runtime.

Defines a family of interchangeable algorithms, encapsulated and swappable at runtime; favours composition over inheritance and avoids large conditionals. Android: `RecyclerView.LayoutManager`, `Interpolator`s, `Comparator`, Glide caching/transformations. In Kotlin, strategies are often just higher-order functions.

---



## 22. Design patterns used in Android

**Android design patterns** means classic architectural and creational patterns (such as Observer, Singleton, Adapter, and Builder) integrated directly into the Android SDK.

Quick map: Builder (`AlertDialog.Builder`), Singleton (`Application`, Room), Factory (`ViewModelProvider.Factory`), Observer (`LiveData`/`Flow`), Adapter (`RecyclerView.Adapter`), Facade (`Retrofit`, `Glide`), Strategy (`LayoutManager`), Decorator (`ContextWrapper`, OkHttp `Interceptor`), Template Method (lifecycle callbacks), Command (`Runnable`/`Worker`), Memento (`onSaveInstanceState`), Proxy (Binder/AIDL, Retrofit), Composite (`View`/`ViewGroup`), DI (Hilt).

---



## 23. Kotlin Optional Parameters vs Builder Pattern

**Kotlin optional parameters** means a language feature that provides default arguments for function parameters, eliminating the need for Java's verbose Builder pattern in most cases.

Builder mainly solves Java's telescoping-constructor problem; Kotlin's **default + named arguments** solve it at the language level, so a Builder is often unnecessary. Still reach for a Builder for: Java interop, stepwise/conditional construction, validation/transformation in `build()`, or a stable public library API. Rule of thumb: prefer default/named args; use Builder for Java consumers or incremental construction.

---



## 24. Examples of the Observer pattern in Android

**Observer pattern** means a behavioural design pattern where an object maintains a list of dependents and notifies them automatically of any state changes.

`LiveData` (`observe`), Kotlin Flow (`StateFlow`/`SharedFlow` + `collect`), RxJava (`subscribe`), listeners/callbacks (`OnClickListener`, `TextWatcher`, `SensorEventListener`), `BroadcastReceiver`, `ContentObserver`, `OnSharedPreferenceChangeListener`, lifecycle observers, Compose state reads, and `WorkManager` work-info LiveData.

---



## 25. Design patterns in the Retrofit source code

**Retrofit design patterns** means the combination of design patterns—including Dynamic Proxy, Facade, Builder, and Adapter—used to implement Retrofit's API client.

- **Builder** — `Retrofit.Builder()`.
- **Facade** — `Retrofit` over OkHttp + converters + adapters.
- **Proxy (Dynamic Proxy)** — `retrofit.create()` returns a `java.lang.reflect.Proxy` whose `InvocationHandler` builds requests.
- **Factory** — `Converter.Factory`, `CallAdapter.Factory`.
- **Adapter** — `CallAdapter` adapts `Call<T>` to other return types.
- **Strategy / Decorator** — pluggable converters/adapters and OkHttp `Interceptor` chain.

---



## 26. Design patterns in Glide

**Glide design patterns** means the creational and structural patterns—such as Builder, Singleton, Resource Pool, and Active Resources—used by Glide for memory-efficient image loading.

- **Builder/fluent** — `Glide.with().load().into()`.
- **Facade** — hides decoding/caching/threading.
- **Singleton** — process-wide `Glide` engine.
- **Strategy** — `DiskCacheStrategy`, transformations, `ModelLoader`s.
- **Factory** — `ModelLoaderFactory`/`ResourceDecoder`.
- **Object Pool** — `BitmapPool` reuses bitmap memory.
- **Observer** — ties requests to host lifecycle.

---



## 27. Design patterns used in AOSP

**AOSP design patterns** means framework-level architectural patterns—such as Binder for IPC, Template Method for lifecycles, and Composite for view hierarchies—used in Android's operating system source code.

Singleton (system services), Factory (`BitmapFactory`, `LayoutInflater.Factory`), Builder (`Notification.Builder`, `Uri.Builder`), Observer (`BroadcastReceiver`, `ContentObserver`), Adapter, Composite (`View`/`ViewGroup` tree), Decorator (`ContextWrapper`), Template Method (lifecycle callbacks), Proxy/Stub (Binder & AIDL, `Stub.asInterface()`), Command (`Handler`/`Message`), Memento (`onSaveInstanceState`), Facade (`LocationManager`, `ConnectivityManager`).

