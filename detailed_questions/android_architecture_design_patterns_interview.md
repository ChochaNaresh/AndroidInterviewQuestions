# Android App Architecture & Design Patterns — Interview Guide

A comprehensive, interview-ready reference covering Android application architecture (MVC, MVP, MVVM, MVI, Clean Architecture, Google's recommended layered architecture, and multi-module architecture) and the classic Gang-of-Four / structural design patterns as they appear in Android and popular libraries (Retrofit, Glide, OkHttp, Room, AOSP). Every answer includes definitions, trade-offs, when-to-use guidance, and concise Kotlin examples. Accurate as of 2026.

---

## Table of Contents

### Part 1 — Application Architecture
1. [Describe the architecture of your last app](#1-describe-the-architecture-of-your-last-app)
2. [Why use MVP / MVVM architectures?](#2-why-use-mvp--mvvm-architectures)
3. [Describe MVVM](#3-describe-mvvm)
4. [MVC vs MVP vs MVVM](#4-mvc-vs-mvp-vs-mvvm)
5. [Why should the View be an interface in MVP?](#5-why-should-the-view-be-an-interface-in-mvp)
6. [What is the use of interfaces in a Presenter?](#6-what-is-the-use-of-interfaces-in-a-presenter)
7. [MVI (Model-View-Intent)](#7-mvi-model-view-intent)
8. [Clean Architecture and the dependency rule](#8-clean-architecture-and-the-dependency-rule)
9. [Google's recommended layered architecture (UI / Domain / Data, UDF)](#9-googles-recommended-layered-architecture-ui--domain--data-udf)
10. [Software Architecture vs Software Design](#10-software-architecture-vs-software-design)
11. [Benefits of Multi-Module Architecture](#11-benefits-of-multi-module-architecture)
12. [Multi-Module Project: Why and When?](#12-multi-module-project-why-and-when)

### Part 2 — Design Patterns
13. [Builder](#13-builder)
14. [Singleton](#14-singleton)
15. [Factory (and Abstract Factory)](#15-factory-and-abstract-factory)
16. [Observer](#16-observer)
17. [Repository](#17-repository)
18. [Adapter](#18-adapter)
19. [Facade](#19-facade)
20. [Dependency Injection](#20-dependency-injection)
21. [Strategy](#21-strategy)
22. [Design patterns used in Android](#22-design-patterns-used-in-android)
23. [Kotlin Optional Parameters vs Builder Pattern](#23-kotlin-optional-parameters-vs-builder-pattern)
24. [Examples of the Observer pattern in Android](#24-examples-of-the-observer-pattern-in-android)
25. [Design patterns in the Retrofit source code](#25-design-patterns-in-the-retrofit-source-code)
26. [Design patterns in Glide](#26-design-patterns-in-glide)
27. [Design patterns used in AOSP](#27-design-patterns-used-in-aosp)

---

# Part 1 — Application Architecture

## 1. Describe the architecture of your last app

**Clean Architecture** means a software design philosophy that separates concerns into independent layers (entities, use cases, presenters/controllers, and UI/data) to achieve high testability and maintainability.

Structure the answer around **layers, data flow, and the libraries that enforce each boundary**.

A strong, modern answer:

> "We followed Google's recommended layered architecture with MVVM in the UI layer. We had three layers:
> - **UI layer** — Jetpack Compose screens + `ViewModel`s. ViewModels expose a single immutable `UiState` via `StateFlow`; the UI is a pure function of state (unidirectional data flow). User events flow up as method calls.
> - **Domain layer** (optional) — single-responsibility *use cases* (interactors) that encapsulate business rules shared by multiple ViewModels.
> - **Data layer** — *repositories* that are the single source of truth, coordinating remote (Retrofit) and local (Room/DataStore) data sources, exposing domain models via Flows.
> - **DI** with Hilt wires everything; the app is split into **feature modules** plus `core:*` modules.
> - Testing: ViewModels and use cases are plain JVM unit tests; repositories use fakes; Compose UI tests cover screens."

Then describe **why**: separation of concerns, testability, the ability to swap data sources, and survivability across configuration changes.

Things to mention as trade-offs you consciously made: whether you used a domain layer (skip it for simple CRUD apps), single vs multi-module, MVVM vs MVI.

---

## 2. Why use MVP / MVVM architectures?

**MVP and MVVM architectures** means presentation-layer patterns designed to decouple business and presentation logic from Android framework classes like Activities and Fragments.

- **Avoid "god" Activities/Fragments** stuffed with UI, business, and data logic.
- **Reusable, testable code** — logic lives in plain classes you can test on the JVM without instrumentation/emulator.
- **Avoid duplicated code** across views that share behaviour.
- **Easier to maintain and extend** — clear responsibilities and boundaries.
- **Test logic without instrumentation** — fast, deterministic unit tests.

In short: separation of concerns → decoupling from the framework → testability and maintainability.

---

## 3. Describe MVVM

**MVVM (Model-View-ViewModel)** means a presentation-layer architectural pattern that separates the UI (View) from business and presentation logic (ViewModel) using state observation.

**Components**
- **Model** — data and business logic (repositories, use cases, entities). Knows nothing about the UI.
- **View** — Activity/Fragment or Composable. Observes the ViewModel and renders state; forwards user events. Contains no business logic.
- **ViewModel** — holds and prepares state for the View, handles user actions, talks to the Model. Exposes observable streams (`StateFlow`/`LiveData`). Crucially, it has **no reference to the View** — communication is via observation, so it survives configuration changes.

**Data binding direction:** The View *observes* the ViewModel. There is no two-way coupling — this is what distinguishes MVVM from MVP (where the Presenter holds a View reference).

**Pros**
- Excellent testability (ViewModel is a plain class).
- Jetpack `ViewModel` survives configuration changes.
- Lifecycle-aware observation reduces leaks.
- Less boilerplate than MVP (no View interface needed).

**Cons**
- Can lead to bloated ViewModels if the domain layer is skipped.
- Observable state management has a learning curve.
- Hard to enforce a single source of truth without discipline.

**When to use:** The default for modern Android, especially with Compose. Strongly preferred over MVP for new apps.

```kotlin
data class NewsUiState(
    val isLoading: Boolean = false,
    val articles: List<Article> = emptyList(),
    val error: String? = null,
)

class NewsViewModel(
    private val repository: NewsRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(NewsUiState())
    val uiState: StateFlow<NewsUiState> = _uiState.asStateFlow()

    fun loadNews() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, error = null) }
            runCatching { repository.getArticles() }
                .onSuccess { list -> _uiState.update { it.copy(isLoading = false, articles = list) } }
                .onFailure { e -> _uiState.update { it.copy(isLoading = false, error = e.message) } }
        }
    }
}

// View (Compose) just renders state and forwards events
@Composable
fun NewsScreen(viewModel: NewsViewModel) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    LaunchedEffect(Unit) { viewModel.loadNews() }
    when {
        state.isLoading -> CircularProgressIndicator()
        state.error != null -> ErrorView(state.error!!)
        else -> ArticleList(state.articles)
    }
}
```

**📚 Reference:** https://outcomeschool.com/blog/mvvm-architecture-android

---

## 4. MVC vs MVP vs MVVM

**MVC vs MVP vs MVVM** means the comparison of presentation patterns where MVC uses a Controller, MVP uses a Presenter interacting with a View interface, and MVVM uses a ViewModel with data binding.

| Aspect | MVC | MVP | MVVM |
|---|---|---|---|
| Logic holder | Controller | Presenter | ViewModel |
| View ↔ logic coupling | Controller often tied to View; in classic Android the Activity acts as both View and Controller | Presenter holds a **View interface** reference (1:1) | ViewModel has **no** View reference; View observes it |
| Communication | Controller manipulates Model & View | Bidirectional via interface (View ↔ Presenter) | Unidirectional observation (View observes ViewModel) |
| Testability | Poor on Android (Activity = View+Controller) | Good (mock the View interface) | Very good (no Android deps in ViewModel) |
| Boilerplate | Low but messy | High (View contracts/interfaces) | Low–medium |
| Config-change survival | Manual | Manual | Built-in with Jetpack `ViewModel` |

**MVC (Model-View-Controller)** — Model, View, Controller. The Controller handles input and updates the Model and View. On Android the Activity tends to be *both* View and Controller, which leads to god classes; rarely used cleanly today.

**MVP (Model-View-Presenter)** — The **Presenter** holds a reference to the View through an **interface** (the contract). The Presenter pulls data from the Model and pushes it to the View by calling interface methods. Decouples View from logic; was the dominant pattern pre-2017.

```kotlin
// MVP contract
interface LoginView {
    fun showLoading()
    fun showError(message: String)
    fun navigateHome()
}

class LoginPresenter(private val view: LoginView, private val repo: AuthRepository) {
    fun login(user: String, pass: String) {
        view.showLoading()
        if (repo.authenticate(user, pass)) view.navigateHome()
        else view.showError("Invalid credentials")
    }
}
```

**MVVM** — see Q3. The ViewModel exposes observable state; the View subscribes. No View interface, no back-reference.

**Bottom line:** MVVM (often layered over Clean Architecture) is the recommended choice for new Android apps; MVP is legacy; MVC is rarely a deliberate choice on Android.

---

## 5. Why should the View be an interface in MVP?

**MVP View interface** means an abstraction layer that allows the Presenter to update the UI without depending directly on concrete Activity or Fragment classes.

- **Decouple logic from the concrete view implementation** — the Presenter depends on an abstraction, not on an Activity/Fragment.
- **Abstract away the Android framework** so the presentation layer has no direct dependency on framework classes, keeping it pure and portable.
- **Swap implementations easily** — e.g. switch the View used in production for a fake/mock in tests.
- **Follow SOLID's Dependency Inversion Principle** — high-level policy (the Presenter) must not depend on low-level details (the concrete View); both depend on the abstraction. This makes the Presenter unit-testable with a mock View, no emulator required.

---

## 6. What is the use of interfaces in a Presenter?

**Presenter interfaces** means abstraction boundaries that decouple the View from the Presenter, enabling independent development and testing of both components.

Benefits:
- The Presenter can be unit-tested against a **mock/fake View**.
- The concrete View can be replaced without touching the Presenter.
- It enforces a clear, documented contract between View and Presenter.
- It enables Dependency Injection of the View into the Presenter.

---

## 7. MVI (Model-View-Intent)

**MVI (Model-View-Intent)** means a unidirectional, state-driven presentation architectural pattern where user actions are represented as intents that emit new UI state models.

The UI is modelled as a **single immutable state** produced by reducing a stream of user **intents** (events).

**Components**
- **Intent** — a user/system event expressed as a sealed type (e.g. `LoadData`, `Refresh`, `ItemClicked`). (Not the Android `Intent` class.)
- **Model / State** — a single immutable data class representing the entire screen state at a point in time.
- **View** — renders the current state and emits intents.
- **Reducer** — pure function `(currentState, result) -> newState`.

**Flow (cyclic, unidirectional):** View emits Intent → processed (often via use case) into a Result → Reducer produces new State → View renders State.

**Pros**
- **Single source of truth** — one immutable state object; predictable and easy to debug (time-travel, state logging).
- No inconsistent partial-state bugs.
- Highly testable (pure reducers).
- Pairs naturally with Compose and coroutines/Flow.

**Cons**
- More boilerplate (sealed intents, results, reducer).
- Creating a new state object per change can feel heavy for trivial screens.
- Steeper learning curve.

**When to use:** Screens with complex, interdependent state and many event sources. For simple screens, plain MVVM with a `UiState` is enough (and is itself "MVI-lite").

```kotlin
// Intents
sealed interface CounterIntent {
    object Increment : CounterIntent
    object Decrement : CounterIntent
}

// Single immutable state
data class CounterState(val count: Int = 0)

class CounterViewModel : ViewModel() {
    private val _state = MutableStateFlow(CounterState())
    val state: StateFlow<CounterState> = _state.asStateFlow()

    fun onIntent(intent: CounterIntent) {
        _state.update { current -> reduce(current, intent) }
    }

    private fun reduce(state: CounterState, intent: CounterIntent) = when (intent) {
        CounterIntent.Increment -> state.copy(count = state.count + 1)
        CounterIntent.Decrement -> state.copy(count = state.count - 1)
    }
}
```

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_outcomeschool-softwareengineering-tech-activity-7288428333430644738-muDc

---

## 8. Clean Architecture and the dependency rule

**Clean Architecture** means a software design philosophy that organises code into concentric layers (entities, use cases, controllers, and UI) with dependency directions pointing strictly inward.

Martin) organises code into concentric layers so that business rules are independent of frameworks, UI, and databases.

**Layers (inner → outer)**
1. **Entities** — enterprise-wide business rules / core domain models.
2. **Use Cases (Interactors)** — application-specific business rules; orchestrate entities.
3. **Interface Adapters** — presenters, ViewModels, repository *implementations*, mappers that convert data between use cases and the outer world.
4. **Frameworks & Drivers** — Android SDK, Retrofit, Room, UI, DB — the volatile details.

**The Dependency Rule** — *source-code dependencies point strictly inward, from outer rings to inner rings.* The innermost layer knows nothing about the outer ones. Inner layers declare interfaces; outer layers implement them. The flow of control may cross boundaries outward, but **dependencies are inverted** so they always point in (Dependency Inversion Principle).

**Typical Android mapping**
- `domain` module: entities + use cases + repository *interfaces* — pure Kotlin, no Android.
- `data` module: repository *implementations*, Retrofit/Room data sources, DTOs and mappers.
- `presentation`/feature modules: ViewModels, Compose UI.

**Pros**
- Framework-independent core; business rules testable in isolation.
- Easy to swap UI, DB, or network layers.
- Clear boundaries scale well in large teams/codebases.

**Cons**
- Significant boilerplate (models + mappers per layer).
- Over-engineering for small apps.

```kotlin
// domain (pure Kotlin) — interface declared by the inner layer
interface UserRepository {
    suspend fun getUser(id: String): User
}

// domain — use case depends only on the abstraction
class GetUserUseCase(private val repo: UserRepository) {
    suspend operator fun invoke(id: String): User = repo.getUser(id)
}

// data — outer layer implements the inner interface (dependency points inward)
class UserRepositoryImpl(
    private val api: UserApi,
    private val dao: UserDao,
) : UserRepository {
    override suspend fun getUser(id: String): User =
        dao.find(id)?.toDomain() ?: api.fetch(id).toDomain().also { dao.save(it.toEntity()) }
}
```

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_androiddev-kotlin-activity-7123192186627571714-6143

---

## 9. Google's recommended layered architecture (UI / Domain / Data, UDF)

**Google's recommended layered architecture (UI / Domain / Data, UDF)** means an architectural style that structures applications into a mandatory UI layer, an optional Domain layer, and a mandatory Data layer governed by Unidirectional Data Flow.

- **UI layer** — displays application data and reacts to user input. State holders (`ViewModel`) hold UI state, expose it to the UI, and handle UI logic. The UI is a function of state.
- **Domain layer** (optional) — encapsulates complex or reusable business logic as **use cases / interactors**, each responsible for a single functionality. Add it when business logic is complex or shared across multiple ViewModels; skip it otherwise.
- **Data layer** — the **single source of truth**. **Repositories** coordinate data sources (network, DB, files), resolve conflicts, and decide where authoritative data lives. Other layers must never touch data sources directly.

**Unidirectional Data Flow (UDF)** — state flows **down** (data → ViewModel → UI) and events flow **up** (UI → ViewModel). The ViewModel holds and exposes state; the UI notifies it of user events; the ViewModel handles them and updates state; the new state is rendered. Combined with the single-source-of-truth principle, this makes state predictable and easy to test.

**Key recommendations**: ViewModels expose immutable UI state via observable streams (`StateFlow`), repositories expose data via Flows, dependencies are injected (Hilt), and each class has a single responsibility.

```kotlin
// Data layer — single source of truth
class TaskRepository(private val api: TaskApi, private val dao: TaskDao) {
    fun observeTasks(): Flow<List<Task>> = dao.observeAll().map { it.map(TaskEntity::toDomain) }
    suspend fun refresh() = dao.upsertAll(api.getTasks().map { it.toEntity() })
}

// Domain layer — single-responsibility use case (optional)
class GetActiveTasksUseCase(private val repo: TaskRepository) {
    operator fun invoke(): Flow<List<Task>> = repo.observeTasks().map { it.filter { t -> !t.done } }
}

// UI layer — state down, events up (UDF)
class TaskViewModel(getActive: GetActiveTasksUseCase) : ViewModel() {
    val uiState: StateFlow<List<Task>> = getActive()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())
}
```

**📚 Reference:** https://developer.android.com/topic/architecture · https://developer.android.com/topic/architecture/ui-layer · https://developer.android.com/topic/architecture/domain-layer · https://developer.android.com/topic/architecture/recommendations

---

## 10. Software Architecture vs Software Design

**Software architecture vs Software design** means the distinction between high-level, system-wide structures and boundaries (architecture), and low-level code-level implementation details and patterns (design).

| | Software Architecture | Software Design |
|---|---|---|
| Scope | High-level, system-wide skeleton | Low-level, module/class internals |
| Concerns | Layers, modules, communication, tech choices, scalability, deployment | Class responsibilities, methods, data structures, design patterns |
| Questions | *What* are the major parts and how do they interact? | *How* is each part implemented? |
| Examples | Choosing MVVM + Clean + multi-module; offline-first; REST vs gRPC | Applying Repository/Factory/Strategy; class APIs; algorithms |
| Change cost | Expensive to change later | Cheaper, more localized |
| Audience | Architects, tech leads | Developers implementing features |

Analogy: architecture is the *blueprint of a building* (number of floors, load-bearing structure, where utilities run); design is *how each room is laid out and furnished*. Architecture is the strategic, hard-to-reverse decisions; design is the tactical, localized decisions made while implementing.

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineering-tech-activity-7230434583584858112-mMEG

---

## 11. Benefits of Multi-Module Architecture

**Multi-module architecture benefits** means the advantages of dividing a project into separate Gradle modules including faster build speeds, strict API boundaries, and modular test isolation.

`app`, `feature:*`, `core:network`, `core:database`, `core:ui`, `domain`, `data`) yields:

- **Faster builds** — Gradle builds modules in parallel and only recompiles changed modules (incremental/cached builds). The single biggest win on large projects.
- **Strict encapsulation & boundaries** — `api`/`implementation` visibility prevents feature A from reaching into feature B's internals; the compiler enforces architecture.
- **Better separation of concerns** — each module has one clear responsibility.
- **Reusability** — `core` modules are shared; modules can be reused across apps (e.g. wear, TV, instant apps).
- **Parallel team work** — teams own modules independently with fewer merge conflicts.
- **On-demand / dynamic feature delivery** — modules can ship as Play Feature Delivery dynamic modules.
- **Improved testability** — smaller, isolated modules with focused tests; faster test runs.
- **Clear ownership** — code owners per module.

**Trade-offs:** more Gradle configuration, navigation between feature modules needs a strategy (API modules, a navigation abstraction), and the initial setup cost is real.

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_androiddev-activity-7371759614591234048-oVH4

---

## 12. Multi-Module Project: Why and When?

**Multi-module project structure** means dividing an application into separate Gradle modules to improve build speed, enforce boundaries, and enable code reuse.

**When to adopt:**
- The codebase/team is **growing** and build times or merge conflicts hurt.
- You have **multiple features** that are logically independent.
- You want to **enforce** architectural layers at the compiler level (not just by convention).
- You need **dynamic feature delivery** or to share code across multiple app targets (phone/Wear/TV).
- Multiple teams need to **own** distinct areas.

**When NOT to (yet):**
- Small apps or prototypes / MVPs — a single, well-packaged module is simpler and faster to iterate on.
- Tiny teams where module overhead outweighs benefits.

**Common module taxonomy:** `app` (entry point, DI graph, navigation host) → `feature:*` (UI + ViewModels per feature) → `core:*` (`network`, `database`, `ui`, `common`, `designsystem`) → `domain` / `data`. A pragmatic rule: start single-module, extract modules when pain (build time, coupling, conflicts) appears.

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineer-tech-activity-7317400201558663173-n-LA

---

# Part 2 — Design Patterns

## 13. Builder

**Builder pattern** means a creational design pattern that allows constructing complex objects step-by-step.

**Where in Android/libraries:**
- `AlertDialog.Builder`, `Notification.Builder` / `NotificationCompat.Builder`
- `OkHttpClient.Builder`, OkHttp `Request.Builder`
- `Retrofit.Builder`
- `Glide` request builders
- `WorkRequest.Builder` (WorkManager), `Uri.Builder`

```kotlin
class HttpRequest private constructor(
    val url: String,
    val method: String,
    val headers: Map<String, String>,
) {
    class Builder(private val url: String) {
        private var method = "GET"
        private val headers = mutableMapOf<String, String>()
        fun method(m: String) = apply { method = m }
        fun header(k: String, v: String) = apply { headers[k] = v }
        fun build() = HttpRequest(url, method, headers)
    }
}

val request = HttpRequest.Builder("https://api.example.com")
    .method("POST")
    .header("Authorization", "Bearer token")
    .build()
```

**📚 Reference:** https://outcomeschool.substack.com/p/design-patterns-in-android-libraries

---

## 14. Singleton

**Singleton pattern** means a creational design pattern that ensures a class has only one instance while providing a global access point to it.

**Where in Android:** `Application` class, Room database instances, Retrofit/OkHttp clients, `LocalBroadcastManager`, repositories — typically scoped as singletons (often via Hilt `@Singleton`).

In Kotlin use `object` (thread-safe, lazily initialized on first access by the classloader):

```kotlin
object AnalyticsTracker {
    fun track(event: String) { /* ... */ }
}
```

For a singleton needing a parameter (e.g. `Context`), use the double-checked locking idiom:

```kotlin
class Database private constructor(context: Context) {
    companion object {
        @Volatile private var instance: Database? = null
        fun getInstance(context: Context): Database =
            instance ?: synchronized(this) {
                instance ?: Database(context.applicationContext).also { instance = it }
            }
    }
}
```

**Trade-offs / caveats:** Global mutable state hurts testability and hides dependencies; prefer DI-scoped singletons over hand-rolled ones. Watch for `Context` leaks (always use `applicationContext`). Singletons are essentially a global, so overuse couples code.

---

## 15. Factory (and Abstract Factory)

**Factory pattern** means a creational design pattern that defines an interface for creating objects, letting subclasses decide which concrete class to instantiate.

- **Abstract Factory** — provide an interface for creating *families* of related objects without specifying concrete classes.

**Where in Android/libraries:** `LayoutInflater.Factory`, `ViewModelProvider.Factory`, `Retrofit`'s `Converter.Factory` and `CallAdapter.Factory`, `BitmapFactory`, `ThreadFactory`, `SSLSocketFactory`.

```kotlin
interface Notification { fun send(message: String) }
class EmailNotification : Notification { override fun send(message: String) { /* ... */ } }
class SmsNotification : Notification { override fun send(message: String) { /* ... */ } }

enum class Channel { EMAIL, SMS }

object NotificationFactory {
    fun create(channel: Channel): Notification = when (channel) {
        Channel.EMAIL -> EmailNotification()
        Channel.SMS -> SmsNotification()
    }
}

NotificationFactory.create(Channel.EMAIL).send("Hello")
```

The classic Android example — providing dependencies to a ViewModel:

```kotlin
class MyViewModelFactory(private val repo: Repo) : ViewModelProvider.Factory {
    override fun <T : ViewModel> create(modelClass: Class<T>): T =
        MyViewModel(repo) as T
}
```

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_outcomeschool-softwareengineer-tech-activity-7300164529571733505-MYfP

---

## 16. Observer

**Observer pattern** means a behavioural design pattern where an object (subject) maintains a list of dependents (observers) and notifies them automatically of state changes.

The foundation of reactive UIs.

**Where in Android:** `LiveData`, `StateFlow`/`SharedFlow`, RxJava `Observable`, `Flow.collect`, `OnClickListener` and all listener callbacks, `ContentObserver`, `BroadcastReceiver`, Jetpack Compose `State` recomposition. (See Q24 for a fuller list.)

```kotlin
// Hand-rolled observer
fun interface Observer<T> { fun onChanged(value: T) }

class Observable<T>(initial: T) {
    private val observers = mutableListOf<Observer<T>>()
    var value: T = initial
        set(new) { field = new; observers.forEach { it.onChanged(new) } }
    fun observe(o: Observer<T>) { observers += o; o.onChanged(value) }
}

val temperature = Observable(20)
temperature.observe { println("Temp is now $it") }
temperature.value = 25 // prints "Temp is now 25"
```

Idiomatic Kotlin/Android equivalent:

```kotlin
val state = MutableStateFlow(0)
lifecycleScope.launch { state.collect { println("Value: $it") } }
state.value = 1
```

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineer-tech-activity-7298657173910233089-3qzw

---

## 17. Repository

**Repository pattern** means a structural design pattern that mediates between the domain and data-mapping layers, abstracting data sources behind a clean API.

Hides the details of *where* data comes from (network, DB, cache) from the rest of the app.

**Where in Android:** Core of Google's recommended architecture data layer; ubiquitous in MVVM/Clean apps.

**Benefits:** decouples ViewModels/use cases from data sources; centralizes caching and conflict resolution; makes swapping or mocking data sources trivial (great for testing).

```kotlin
interface ArticleRepository {
    fun observeArticles(): Flow<List<Article>>
    suspend fun refresh()
}

class ArticleRepositoryImpl(
    private val api: ArticleApi,
    private val dao: ArticleDao,
) : ArticleRepository {
    // DB is the single source of truth
    override fun observeArticles(): Flow<List<Article>> =
        dao.observeAll().map { entities -> entities.map { it.toDomain() } }

    override suspend fun refresh() {
        val remote = api.getArticles()
        dao.upsertAll(remote.map { it.toEntity() })
    }
}
```

---

## 18. Adapter

**Adapter pattern** means a structural design pattern that converts the interface of a class into another interface that clients expect.

**Where in Android:** `RecyclerView.Adapter` / `ListAdapter`, `ArrayAdapter`, `PagerAdapter` / `FragmentStateAdapter`, Retrofit's `CallAdapter` (adapts a `Call` to RxJava/coroutines/etc.). `RecyclerView.Adapter` adapts your data model into `ViewHolder`s the RecyclerView understands.

```kotlin
// Adaptee: a legacy logger with an incompatible API
class LegacyLogger { fun writeLine(level: Int, text: String) { /* ... */ } }

// Target interface the app expects
interface AppLogger { fun log(message: String) }

// Adapter
class LegacyLoggerAdapter(private val legacy: LegacyLogger) : AppLogger {
    override fun log(message: String) = legacy.writeLine(1, message)
}
```

RecyclerView example:

```kotlin
class UserAdapter : ListAdapter<User, UserAdapter.VH>(DiffCallback) {
    inner class VH(val binding: ItemUserBinding) : RecyclerView.ViewHolder(binding.root)
    override fun onCreateViewHolder(p: ViewGroup, t: Int) =
        VH(ItemUserBinding.inflate(LayoutInflater.from(p.context), p, false))
    override fun onBindViewHolder(h: VH, pos: Int) { h.binding.name.text = getItem(pos).name }
}
```

---

## 19. Facade

**Facade pattern** means a structural design pattern that provides a unified, simplified interface to a complex subsystem of classes.

**Where in Android:** `Retrofit` itself is a facade over OkHttp + converters + call adapters; `Glide.with(...)` hides decoding/caching/threading; `ContextCompat` / `NotificationCompat` simplify version-aware APIs; a `Repository` is often a facade over multiple data sources.

```kotlin
// Subsystems
class AudioSystem { fun on() {}; fun setVolume(v: Int) {} }
class VideoSystem { fun on() {}; fun setResolution(r: String) {} }
class StreamingSystem { fun connect(url: String) {} }

// Facade: one simple method hides the multi-step subsystem orchestration
class MediaCenterFacade {
    private val audio = AudioSystem()
    private val video = VideoSystem()
    private val streaming = StreamingSystem()

    fun watchMovie(url: String) {
        audio.on(); audio.setVolume(50)
        video.on(); video.setResolution("4K")
        streaming.connect(url)
    }
}

MediaCenterFacade().watchMovie("https://...")
```

---

## 20. Dependency Injection

**Dependency Injection** means a design pattern where an object's dependencies are supplied by an external assembler rather than the object instantiating them itself.

**Why use it:**
- **Decoupling** — classes depend on abstractions, not concrete creations.
- **Easier unit testing** — inject fakes/mocks.
- **Smaller, simpler classes** — no construction logic; easy to move things around.
- **Centralized wiring** of the object graph.

**Forms:** constructor injection (preferred), field/setter injection, method injection.

**In Android:** **Hilt** (built on Dagger) is the recommended framework; Koin is a popular service-locator-style alternative.

```kotlin
// Manual constructor injection
class LoginViewModel(private val repo: AuthRepository) { /* ... */ }

// Hilt
@HiltViewModel
class FeedViewModel @Inject constructor(
    private val repository: FeedRepository
) : ViewModel()

@Module
@InstallIn(SingletonComponent::class)
object DataModule {
    @Provides @Singleton
    fun provideApi(): FeedApi = Retrofit.Builder()
        .baseUrl("https://api.example.com")
        .build()
        .create(FeedApi::class.java)
}
```

**Note:** DI (Dependency Injection) vs Service Locator — with DI dependencies are *pushed in* (you can't use the object without them); with a Service Locator the object *pulls* them from a registry, which hides dependencies and is harder to test.

---

## 21. Strategy

**Strategy pattern** means a behavioural design pattern that defines a family of interchangeable algorithms and encapsulates each one to be swapped at runtime.

Lets the algorithm vary independently from the clients that use it. Favours composition over inheritance and avoids large conditionals.

**Where in Android/libraries:** `RecyclerView.LayoutManager` (Linear/Grid/Staggered are swappable layout strategies), `Interpolator`s in animations, Glide/OkHttp caching and transformation strategies, `Comparator` for sorting, pluggable image-transformation strategies.

```kotlin
fun interface DiscountStrategy { fun apply(price: Double): Double }

val noDiscount = DiscountStrategy { it }
val seasonal = DiscountStrategy { it * 0.9 }
val clearance = DiscountStrategy { it * 0.5 }

class Checkout(private var strategy: DiscountStrategy) {
    fun setStrategy(s: DiscountStrategy) { strategy = s }
    fun total(price: Double): Double = strategy.apply(price)
}

val checkout = Checkout(seasonal)
checkout.total(100.0)            // 90.0
checkout.setStrategy(clearance)
checkout.total(100.0)            // 50.0
```

In Kotlin, strategies are often just higher-order functions (`(Double) -> Double`) instead of interfaces.

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_androiddev-android-activity-7396043401357533184-a6OQ

---

## 22. Design patterns used in Android

**Android design patterns** means classic architectural and creational patterns (such as Observer, Singleton, Adapter, and Builder) integrated directly into the Android SDK.

| Pattern | Android / Jetpack usage |
|---|---|
| **Builder** | `AlertDialog.Builder`, `NotificationCompat.Builder`, `Retrofit.Builder`, `OkHttpClient.Builder`, `WorkRequest.Builder` |
| **Singleton** | `Application`, Room DB, Retrofit/OkHttp clients, Hilt `@Singleton` |
| **Factory** | `ViewModelProvider.Factory`, `LayoutInflater.Factory`, `BitmapFactory`, Retrofit `Converter.Factory` |
| **Observer** | `LiveData`, `StateFlow`/`SharedFlow`, RxJava, listeners, `ContentObserver`, `BroadcastReceiver`, Compose recomposition |
| **Adapter** | `RecyclerView.Adapter`, `ArrayAdapter`, Retrofit `CallAdapter` |
| **Facade** | `Retrofit`, `Glide.with()`, `ContextCompat`, `NotificationCompat` |
| **Strategy** | `RecyclerView.LayoutManager`, `Interpolator`, `Comparator` |
| **Decorator** | `ContextWrapper`/`ContextThemeWrapper`, OkHttp `Interceptor` chain, Java I/O streams |
| **Template Method** | `Activity`/`Fragment` lifecycle callbacks, `AsyncTask` (legacy), `RecyclerView.Adapter` hooks |
| **Command** | `Runnable`, WorkManager `Worker`, `Handler.post` |
| **Memento** | `onSaveInstanceState` / `SavedStateHandle` |
| **Proxy** | Binder/AIDL stubs (`Stub.asInterface`), dynamic proxies in Retrofit |
| **Composite** | The `View`/`ViewGroup` hierarchy |
| **Prototype** | `Intent`/`Bundle` cloning, `View` copy patterns |
| **Dependency Injection** | Hilt / Dagger |
| **MVVM/MVI/Repository** | Jetpack architecture components |

**📚 Reference:** https://outcomeschool.substack.com/p/design-patterns-in-android-libraries

---

## 23. Kotlin Optional Parameters vs Builder Pattern

**Kotlin optional parameters** means a language feature that provides default arguments for function parameters, eliminating the need for Java's verbose Builder pattern in most cases.

Kotlin solves that at the language level with **default arguments** and **named arguments**, so a Builder is often unnecessary.

```kotlin
// Kotlin: default + named arguments replace a Builder
data class HttpRequest(
    val url: String,
    val method: String = "GET",
    val timeoutMs: Long = 30_000,
    val headers: Map<String, String> = emptyMap(),
)

val req = HttpRequest(
    url = "https://api.example.com",
    method = "POST",
    headers = mapOf("Authorization" to "Bearer token"),
)
```

**When Kotlin default/named params are enough:**
- Simple, immutable value objects with optional fields.
- All parameters known at one call site.

**When a Builder still earns its place:**
- **Java interop** — callers from Java can't use Kotlin named/default args, so a Builder gives them a fluent API.
- **Stepwise / conditional construction** — building an object across multiple statements or loops (e.g. adding headers conditionally).
- **Validation or transformation** in `build()` before producing the final object.
- **Many invalid combinations** you want to guard against, or an immutable object assembled incrementally.
- A widely used **public library API** where a stable, discoverable fluent interface is valuable.

**Rule of thumb:** prefer constructors with default/named arguments in Kotlin; reach for a Builder for Java consumers, incremental construction, or complex validation.

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_softwareengineer-androiddev-android-activity-7270656113845370881-LDW8

---

## 24. Examples of the Observer pattern in Android

**Observer pattern** means a behavioural design pattern where an object maintains a list of dependents and notifies them automatically of any state changes.

Concrete examples:

- **`LiveData`** — `observe(lifecycleOwner) { ... }`; lifecycle-aware, emits on the main thread.
- **Kotlin Flow** — `StateFlow`/`SharedFlow` with `collect`; `collectAsStateWithLifecycle()` in Compose.
- **RxJava** — `Observable`/`Flowable`/`Single` with `subscribe`.
- **Listeners / callbacks** — `View.OnClickListener`, `TextWatcher`, `RecyclerView.OnScrollListener`, `SensorEventListener`, `LocationListener`.
- **`BroadcastReceiver`** — observes system/app broadcasts.
- **`ContentObserver`** — observes changes to a `ContentProvider` URI (e.g. contacts, media).
- **`OnSharedPreferenceChangeListener`** — observes SharedPreferences changes.
- **Lifecycle observers** — `DefaultLifecycleObserver` / `LifecycleEventObserver`.
- **Jetpack Compose** — reading a snapshot `State<T>` inside a composable subscribes it; changes trigger recomposition. (A `StateFlow`/`MutableStateFlow` must first be turned into `State` via `collectAsStateWithLifecycle()`/`collectAsState()`.)
- **`WorkManager`** — `getWorkInfoByIdLiveData()` to observe work status.

```kotlin
// LiveData
viewModel.user.observe(viewLifecycleOwner) { user -> binding.name.text = user.name }

// StateFlow in Compose
val state by viewModel.uiState.collectAsStateWithLifecycle()

// Flow collection
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.events.collect { showSnackbar(it) }
    }
}
```

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_androiddev-activity-7288114509120970752-Z6c8

---

## 25. Design patterns in the Retrofit source code

**Retrofit design patterns** means the combination of design patterns—including Dynamic Proxy, Facade, Builder, and Adapter—used to implement Retrofit's API client.

- **Builder** — `Retrofit.Builder()` assembles base URL, converters, call adapters, and the OkHttp client.
- **Facade** — `Retrofit` presents one simple entry point over OkHttp + converters + call adapters, hiding the HTTP machinery.
- **Proxy (Dynamic Proxy)** — `retrofit.create(MyApi::class.java)` returns a `java.lang.reflect.Proxy`; calls to your interface methods are intercepted by an `InvocationHandler` that builds and executes the request. This is the core trick that turns an annotated interface into working network calls.
- **Factory** — `Converter.Factory` (e.g. Gson/Moshi/kotlinx-serialisation) and `CallAdapter.Factory` (coroutines/RxJava) create converters/adapters; Retrofit iterates registered factories to find one that handles a type.
- **Adapter** — `CallAdapter` adapts the native `Call<T>` into other return types (`suspend`, RxJava `Observable`, etc.).
- **Strategy** — pluggable converters and call adapters are interchangeable strategies selected at runtime.
- **Decorator / Chain of Responsibility** — via the underlying OkHttp `Interceptor` chain.

```kotlin
val retrofit = Retrofit.Builder()                    // Builder
    .baseUrl("https://api.example.com")
    .addConverterFactory(MoshiConverterFactory.create()) // Factory + Strategy
    .build()

val api = retrofit.create(ApiService::class.java)    // Dynamic Proxy
```

**📚 Reference:** https://outcomeschool.substack.com/p/design-patterns-in-android-libraries

---

## 26. Design patterns in Glide

**Glide design patterns** means the creational and structural patterns—such as Builder, Singleton, Resource Pool, and Active Resources—used by Glide for memory-efficient image loading.

- **Builder / Fluent interface** — `Glide.with(context).load(url).placeholder(R.drawable.ph).into(imageView)` chains a `RequestBuilder`.
- **Facade** — `Glide.with(...)` hides decoding, threading, transformations, and the multi-tier cache behind a simple API.
- **Singleton** — the `Glide` engine instance is a process-wide singleton (`Glide.get(context)`).
- **Strategy** — `DiskCacheStrategy` (e.g. `ALL`, `NONE`, `RESOURCE`, `DATA`), transformations (`CenterCrop`, `CircleCrop`), and pluggable `ModelLoader`s are interchangeable strategies.
- **Factory** — `ModelLoaderFactory` / `ResourceDecoder` create loaders and decoders for different model/resource types via the registry.
- **Object Pool / Flyweight** — `BitmapPool` reuses bitmap memory to reduce GC pressure.
- **Observer / Lifecycle integration** — Glide ties requests to the host's lifecycle (a `RequestManagerFragment`/lifecycle listener) to pause/resume/clear automatically.
- **Decorator** — chained transformations wrap one another.

```kotlin
Glide.with(context)                 // Facade + Singleton engine
    .load(url)                      // Builder
    .diskCacheStrategy(DiskCacheStrategy.ALL) // Strategy
    .transform(CircleCrop())        // Strategy/Decorator
    .placeholder(R.drawable.ph)
    .into(imageView)
```

**📚 Reference:** https://outcomeschool.substack.com/p/design-patterns-in-android-libraries

---

## 27. Design patterns used in AOSP

**AOSP design patterns** means framework-level architectural patterns—such as Binder for IPC, Template Method for lifecycles, and Composite for view hierarchies—used in Android's operating system source code.

- **Singleton** — system service accessors and global managers.
- **Factory** — `BitmapFactory`, `LayoutInflater.Factory`, `ThreadFactory`.
- **Builder** — `Notification.Builder`, `AlertDialog.Builder`, `Uri.Builder`.
- **Observer** — `BroadcastReceiver`, `ContentObserver`, `Observable`/`Observer`, listener callbacks.
- **Adapter** — `Adapter`/`RecyclerView.Adapter`, `ArrayAdapter`.
- **Composite** — the `View`/`ViewGroup` tree (a `ViewGroup` is both a node and a container of `View`s).
- **Decorator** — `ContextWrapper` and `ContextThemeWrapper` wrap a base `Context` to add/override behaviour.
- **Template Method** — lifecycle callbacks (`onCreate`, `onStart`, …) where the framework defines the skeleton and apps fill in steps.
- **Proxy / Stub (Binder & AIDL)** — IPC uses `Binder` proxies; AIDL generates `Stub` (server) and proxy (client) classes; `Stub.asInterface()` returns the right one. This is core to Android's process model.
- **Command** — `Runnable`/`Message` posted to a `Handler`/`Looper` message queue for deferred execution.
- **Memento** — `onSaveInstanceState`/`onRestoreInstanceState` capture and restore UI state.
- **Prototype** — `Bundle`/`Parcelable` copying.
- **Facade** — high-level managers (e.g. `LocationManager`, `ConnectivityManager`) front complex subsystems.

```kotlin
// Decorator: ContextWrapper delegates to a base Context, allowing overrides
class LoggingContext(base: Context) : ContextWrapper(base) {
    override fun startActivity(intent: Intent) {
        Log.d("Ctx", "startActivity: ${intent.component}")
        super.startActivity(intent) // delegate to wrapped Context
    }
}
```

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_outcomeschool-softwareengineer-tech-activity-7373202625908936704-7FKH

---

*End of guide.*
