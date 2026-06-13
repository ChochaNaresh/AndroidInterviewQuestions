# Jetpack Compose Interview Questions

A comprehensive, code-first guide to Jetpack Compose interview topics, accurate as of 2026 (Compose BOM 2025.x, Kotlin 2.x with the Compose Compiler Gradle plugin). Each answer covers the concept, idiomatic Kotlin/Compose code, and the trade-offs interviewers expect you to articulate.

---

## Table of Contents

1. [Jetpack Compose vs Android View System](#1-jetpack-compose-vs-android-view-system)
2. [Declarative UI in Jetpack Compose](#2-declarative-ui-in-jetpack-compose)
3. [Declarative UI vs Imperative UI](#3-declarative-ui-vs-imperative-ui)
4. [What are Composable functions?](#4-what-are-composable-functions)
5. [What is Recomposition?](#5-what-is-recomposition)
6. [What is State in Compose?](#6-what-is-state-in-compose)
7. [How does state management work?](#7-how-does-state-management-work)
8. [Stateful vs Stateless composables](#8-stateful-vs-stateless-composables)
9. [What are side effects?](#9-what-are-side-effects)
10. [LaunchedEffect vs DisposableEffect](#10-launchedeffect-vs-disposableeffect)
11. [rememberCoroutineScope and use cases](#11-remembercoroutinescope-and-use-cases)
12. [Observing Flow and LiveData in Compose](#12-observing-flow-and-livedata-in-compose)
13. [Handling asynchronous operations](#13-handling-asynchronous-operations)
14. [Converting non-Compose state into Compose state](#14-converting-non-compose-state-into-compose-state)
15. [derivedStateOf](#15-derivedstateof)
16. [rememberUpdatedState](#16-rememberupdatedstate)
17. [remember vs rememberSaveable](#17-remember-vs-remembersaveable)
18. [Lifecycle of a Composable](#18-lifecycle-of-a-composable)
19. [Handling lifecycle events in Compose](#19-handling-lifecycle-events-in-compose)
20. [Performance best practices](#20-performance-best-practices)
21. [Using Compose and Views together (interop)](#21-using-compose-and-views-together-interop)
22. [State Hoisting](#22-state-hoisting)
23. [CompositionLocal](#23-compositionlocal)
24. [Jetpack Compose Phases](#24-jetpack-compose-phases)
25. [The Modifier](#25-the-modifier)
26. [Semantics](#26-semantics)
27. [Handling user input and events](#27-handling-user-input-and-events)
28. [Navigation in Compose](#28-navigation-in-compose)
29. [Orientation changes](#29-orientation-changes)
30. [Unidirectional Data Flow](#30-unidirectional-data-flow)
31. [Custom Layouts](#31-custom-layouts)

---

## 1. Jetpack Compose vs Android View System

**Jetpack Compose vs View System** means the comparison between the modern declarative Kotlin UI toolkit (Compose), and the legacy imperative XML layout hierarchy (Views).

| Aspect | Compose | View System |
|---|---|---|
| Paradigm | Declarative (describe UI for a state) | Imperative (mutate views) |
| Language | Kotlin only | XML layouts + Java/Kotlin |
| UI element | `@Composable` function (no object) | `View`/`ViewGroup` objects |
| State → UI | Recomposition re-runs functions | You manually call setters |
| Layout | Single-pass measure (mostly) | Two-pass measure (can cause double-taxation in nesting) |
| Lookups | None (no IDs/inflation) | `findViewById`, view binding |
| Theming | `MaterialTheme`, `CompositionLocal` | XML themes/styles |
| Tooling | `@Preview`, live edit | Layout editor |

Compose has no inflation step, no view hierarchy of heavyweight objects, and avoids the classic two-pass measurement pathologies (it forbids multiple measurements without an explicit opt-in like `SubcomposeLayout`). The cost is a learning curve, runtime recomposition overhead if misused, and a still-maturing ecosystem for some specialized widgets.

**📚 Reference:** https://developer.android.com/jetpack/compose/mental-model

---

## 2. Declarative UI in Jetpack Compose

**Compose runtime compiler** means the system that uses a Kotlin compiler plugin to build a slot table, tracking state changes to recompose UI nodes.

You never hold or mutate widget objects.

```kotlin
@Composable
fun Greeting(name: String, expanded: Boolean, onToggle: () -> Unit) {
    Column(Modifier.clickable(onClick = onToggle)) {
        Text("Hello, $name")
        if (expanded) Text("Welcome back!")
    }
}
```

When `expanded` changes, Compose **recomposes** `Greeting`, re-evaluates the `if`, and reconciles the difference. You did not write `textView.setVisibility(...)`. The UI is a *function of state*: `UI = f(state)`.

**📚 Reference:** https://developer.android.com/jetpack/compose/mental-model

---

## 3. Declarative UI vs Imperative UI

**Declarative UI** means writing functions that describe the UI for a given state, whereas **imperative UI** means mutating view properties step-by-step.

```kotlin
// View system (imperative)
fun onLikeChanged(liked: Boolean) {
    likeIcon.setImageResource(if (liked) R.drawable.filled else R.drawable.outline)
    likeCount.text = count.toString()
    likeCount.visibility = if (count > 0) View.VISIBLE else View.GONE
}
```
The bug surface is *state desync*: forget one setter and the UI lies.

**Declarative**: you re-describe the whole UI for the new state; the runtime diffs it.

```kotlin
@Composable
fun Like(liked: Boolean, count: Int) {
    Row {
        Icon(if (liked) Icons.Filled.Favorite else Icons.Outlined.FavoriteBorder, null)
        if (count > 0) Text("$count")
    }
}
```

Trade-off: declarative is easier to reason about and harder to desync, but you give up fine-grained manual control and must trust/understand recomposition for performance.

---

## 4. What are Composable functions?

**Composable function** means a function annotated with `@Composable` that emits user interface nodes into the composition tree.

Key properties:

- They can only be called from other composables (the compiler enforces a `$composer` parameter is threaded through).
- They should be **side-effect free** and **idempotent** — they may run many times, in any order, on any thread, or be skipped.
- They return `Unit` (they emit UI) — they do not return view objects.

```kotlin
@Composable
fun ProfileCard(user: User) {
    Card {
        Column(Modifier.padding(16.dp)) {
            Text(user.name, style = MaterialTheme.typography.titleMedium)
            Text(user.email)
        }
    }
}
```

Rules to respect: don't mutate global state from a composable body, don't depend on execution order between sibling composables, and don't assume a fixed number of executions — put such logic in side-effect APIs or event handlers instead.

---

## 5. What is Recomposition?

**Recomposition** means the process where the Compose runtime executes composables with updated state parameters to repaint changes.

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    Button(onClick = { count++ }) {     // reading count here...
        Text("Count: $count")            // ...so this Text recomposes on change
    }
}
```

Important behaviours:
- Compose **skips** composables whose inputs haven't changed (if their parameters are *stable*).
- Recomposition is **intelligent and scoped** — only the smallest recomposition scope reading the state re-runs.
- It can run **out of order**, **in parallel**, and be **cancelled/restarted** (e.g., during fast scrolling). Therefore composables must be free of order-dependent side effects.

The runtime tracks reads via a **snapshot system**: reading a `State` inside a scope subscribes that scope; writing the state invalidates subscribed scopes for the next frame.

---

## 6. What is State in Compose?

**Compose state** means an observable value wrapper (like `MutableState`) that triggers recomposition in reading composables when modified.

The core type is `State<T>` (read-only) / `MutableState<T>`.

```kotlin
val state: MutableState<Int> = mutableStateOf(0)
state.value++                       // explicit

var count by mutableStateOf(0)      // with property delegate (by)
count++
```

`mutableStateOf` returns an observable holder backed by the snapshot system. To survive recomposition it must be stored in `remember`:

```kotlin
var text by remember { mutableStateOf("") }
```

Other state factories: `mutableStateListOf`, `mutableStateMapOf`, `mutableIntStateOf`/`mutableLongStateOf`/`mutableDoubleStateOf` (primitive-specialized to avoid boxing), and adapters like `collectAsStateWithLifecycle`, `produceState`, `derivedStateOf`.

**📚 Reference:** https://developer.android.com/jetpack/compose/state

---

## 7. How does state management work?

**State management** means architecting data flow using hoisting, state holders, and ViewModels to maintain a unidirectional data flow.

1. **Observable state** (`mutableStateOf`) tracked by the **snapshot** system.
2. **`remember`** to persist that state across recompositions (in the slot table).
3. **Recomposition** that re-reads state and updates the UI.

A scalable pattern combines **state hoisting** with **unidirectional data flow**, often anchored in a `ViewModel`:

```kotlin
class CounterViewModel : ViewModel() {
    private val _state = MutableStateFlow(0)
    val state: StateFlow<Int> = _state.asStateFlow()
    fun increment() { _state.update { it + 1 } }
}

@Composable
fun CounterScreen(vm: CounterViewModel = viewModel()) {
    val count by vm.state.collectAsStateWithLifecycle()
    Counter(count = count, onIncrement = vm::increment)   // stateless child
}
```

Guidance: keep UI state in the lowest common owner, expose immutable state down and events up, and use a `ViewModel` for state that must survive configuration changes / belongs to business logic.

**📚 Reference:** https://developer.android.com/jetpack/compose/state

---

## 8. Stateful vs Stateless composables

**Stateful vs Stateless composable** means the difference between a composable that holds and manages its own internal state (stateful), and a reusable composable that receives state via parameters (stateless).

- **Stateful**: owns/holds its own state internally (via `remember`). Convenient but harder to reuse, test, and control.
- **Stateless**: holds no state; receives state via parameters and reports changes via callbacks. Reusable, testable, previewable.

```kotlin
// Stateful (owns state)
@Composable
fun NameField() {
    var name by remember { mutableStateOf("") }
    OutlinedTextField(name, onValueChange = { name = it })
}

// Stateless (state hoisted out)
@Composable
fun NameField(name: String, onNameChange: (String) -> Unit) {
    OutlinedTextField(name, onValueChange = onNameChange)
}
```

Best practice: write **stateless** composables and hoist state to a stateful caller (state hoisting). This separation makes the leaf reusable in different contexts and trivially testable/previewable.

---

## 9. What are side effects?

**Side effect** means an escape hatch or operation that happens outside the control of the Compose rendering thread, such as updating databases.

Because composables can run repeatedly and unpredictably, such work must be wrapped in **Effect APIs** that give it controlled, lifecycle-aware execution.

Key Effect APIs:

| API | Purpose |
|---|---|
| `LaunchedEffect(keys)` | Run a suspend block tied to composition; cancels/relaunches when a key changes. |
| `DisposableEffect(keys)` | Setup + `onDispose` cleanup; re-runs on key change and on leaving composition. |
| `SideEffect {}` | Publish Compose state to non-Compose code after every successful (re)composition. |
| `produceState` | Convert non-Compose/async sources into `State`. |
| `rememberCoroutineScope()` | A scope tied to composition for launching from callbacks. |
| `rememberUpdatedState` | Capture the latest value inside a long-lived effect without restarting it. |
| `derivedStateOf` | Compute state from other state, recomputing only when inputs meaningfully change. |
| `snapshotFlow` | Convert Compose `State` reads into a `Flow`. |

**📚 Reference:** https://developer.android.com/jetpack/compose/side-effects

---

## 10. LaunchedEffect vs DisposableEffect

**LaunchedEffect vs DisposableEffect** means the choice between running coroutines bound to a composable's lifecycle (LaunchedEffect), and running effects that require cleanup when leaving composition (DisposableEffect).

**`LaunchedEffect`** runs a **suspending** block in a coroutine. It launches when entering composition, cancels on leaving, and **cancels-and-relaunches** when any key changes.

```kotlin
LaunchedEffect(userId) {                 // restarts when userId changes
    val user = repository.fetchUser(userId)   // suspend
    snackbarHostState.showSnackbar("Loaded ${user.name}")
}
```

**`DisposableEffect`** is for effects that need **cleanup**. It runs a setup block and *requires* an `onDispose {}` that runs when leaving composition or before re-running due to a key change.

```kotlin
DisposableEffect(lifecycleOwner) {
    val observer = LifecycleEventObserver { _, event -> /* ... */ }
    lifecycleOwner.lifecycle.addObserver(observer)
    onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
}
```

Rule of thumb: suspend/async work → `LaunchedEffect`; register-then-unregister (listeners, observers, sensors) → `DisposableEffect`. Use `LaunchedEffect(Unit)` (a constant key) to run once for the composition's lifetime.

---

## 11. rememberCoroutineScope and use cases

**rememberCoroutineScope** means an API that returns a coroutine scope bound to the composition lifecycle to launch coroutines in response to user gestures.

Use it to launch coroutines from **non-composable callbacks** (e.g., `onClick`), where you can't call `LaunchedEffect`.

```kotlin
@Composable
fun MessageScreen(snackbarHostState: SnackbarHostState) {
    val scope = rememberCoroutineScope()
    Button(onClick = {
        scope.launch { snackbarHostState.showSnackbar("Sent!") }
    }) { Text("Send") }
}
```

Difference from `LaunchedEffect`:
- `LaunchedEffect` is for work driven by composition/keys; it auto-launches and restarts on key change.
- `rememberCoroutineScope` is for work driven by **user events**; you launch manually. The scope is cancelled automatically when the composable leaves the composition.

Common use cases: animating on click (`scope.launch { animatable.animateTo(...) }`), `scrollState.animateScrollTo(...)`, showing Snackbars, drawer open/close.

---

## 12. Observing Flow and LiveData in Compose

**Reactive state observation** means converting LiveData or Flow streams into Compose `State` instances using `observeAsState()` or `collectAsStateWithLifecycle()`. 

```kotlin
// StateFlow / Flow — lifecycle-aware (recommended)
val ui by viewModel.uiState.collectAsStateWithLifecycle()   // androidx.lifecycle:lifecycle-runtime-compose

// Plain Flow without initial value
val items by repo.itemFlow.collectAsStateWithLifecycle(initialValue = emptyList())

// LiveData
val name by viewModel.name.observeAsState()                 // androidx.compose.runtime:runtime-livedata
```

`collectAsStateWithLifecycle` collects only when the lifecycle is at least `STARTED`, automatically pausing collection in the background — preferred over `collectAsState()` for Android UI to avoid wasted work and resource leaks. `observeAsState` bridges `LiveData` (it observes against the composition's `LifecycleOwner`).

For RxJava there are `subscribeAsState` adapters in `runtime-rxjava2/3`.

---

## 13. Handling asynchronous operations

**Asynchronous operations in Compose** means handling background operations using side-effect APIs like `LaunchedEffect` or `produceState`.

- **Triggered by composition/keys** → `LaunchedEffect`.
- **Triggered by user events** → `rememberCoroutineScope().launch`.
- **Producing State from an async/callback source** → `produceState`.
- **Stream → State** → `collectAsStateWithLifecycle` / `produceState`.

```kotlin
@Composable
fun UserName(userId: String): State<Result<String>> = produceState(
    initialValue = Result.loading(), key1 = userId
) {
    value = try { Result.success(api.getUser(userId).name) }
    catch (e: Exception) { Result.error(e) }
}
```

Best practice: keep heavy/business async work in the `ViewModel` (`viewModelScope`), expose results as `StateFlow`, and let the UI just collect. Reserve `LaunchedEffect`/`produceState` for UI-scoped concerns (animations, one-shot UI effects, source-to-state bridging).

---

## 14. Converting non-Compose state into Compose state

**State bridging** means using `produceState` or `remember` to wrap callbacks or sensor events into Compose-readable state.

```kotlin
@Composable
fun locationState(client: LocationClient): State<Location?> =
    produceState<Location?>(initialValue = null, client) {
        val callback = LocationCallback { value = it }
        client.register(callback)
        awaitDispose { client.unregister(callback) }   // runs when leaving composition
    }
```

Other bridges:
- `Flow`/`LiveData`/`Observable` → `collectAsStateWithLifecycle` / `observeAsState` / `subscribeAsState`.
- A snapshot `State` → `Flow` via `snapshotFlow { ... }` (the reverse direction).

`produceState` internally uses `remember` + `LaunchedEffect`, so it is lifecycle-safe and recomposition-stable.

---

## 15. derivedStateOf

**Modifier** means an ordered collection of elements used to decorate, size, style, or add behaviour to composables.

Use it when a frequently-changing input maps to a rarely-changing output.

```kotlin
val listState = rememberLazyListState()
val showButton by remember {
    derivedStateOf { listState.firstVisibleItemIndex > 0 }
}
// 'showButton' recomposes its readers only on the false<->true transition,
// not on every scroll pixel that changes firstVisibleItemIndex.
```

Without `derivedStateOf`, reading `firstVisibleItemIndex` directly would recompose on every scroll frame. Wrap it in `remember { derivedStateOf { ... } }` so the derivation survives recomposition.

Anti-pattern: don't use it for a 1:1 transform of inputs (e.g., `derivedStateOf { a + b }` when both change together) — there it adds overhead with no filtering benefit; just compute inline.

---

## 16. rememberUpdatedState

**Modifier ordering** means the rule that modifier operations are applied sequentially from left-to-right, making their layout order significant.

```kotlin
@Composable
fun LandingScreen(onTimeout: () -> Unit) {
    val currentOnTimeout by rememberUpdatedState(onTimeout)
    LaunchedEffect(Unit) {                 // key is constant: effect must NOT restart
        delay(3000)
        currentOnTimeout()                 // always the freshest lambda
    }
}
```

Here `LaunchedEffect(Unit)` must run exactly once, but `onTimeout` may change across recompositions. If we keyed the effect on `onTimeout`, the timer would restart whenever the lambda's identity changed. `rememberUpdatedState` decouples "use the latest value" from "restart the effect."

**📚 Reference:** https://developer.android.com/jetpack/compose/side-effects

---

## 17. remember vs rememberSaveable

**remember vs rememberSaveable** means the choice between persisting values across recompositions only (remember), and persisting values across configuration changes and process death (rememberSaveable).

- `remember`: stored in the composition's slot table. Survives recomposition; **lost** on configuration change (e.g., rotation) and process death.
- `rememberSaveable`: also writes to a `Bundle` via the saved-instance-state mechanism. Survives recomposition **and** configuration changes **and** system-initiated process death.

```kotlin
var a by remember { mutableStateOf(0) }          // lost on rotation
var b by rememberSaveable { mutableStateOf(0) }  // survives rotation & process death
```

`rememberSaveable` automatically handles types that fit in a `Bundle` (primitives, `String`, `Parcelable`). For custom types, supply a `Saver`:

```kotlin
data class City(val name: String, val country: String)

val CitySaver = listSaver<City, String>(
    save = { listOf(it.name, it.country) },
    restore = { City(it[0], it[1]) }
)

// For a MutableState<City>, pass the saver via stateSaver:
val city = rememberSaveable(stateSaver = CitySaver) { mutableStateOf(City("Paris", "France")) }
// For a plain (non-State) value, use the saver parameter directly:
val plainCity = rememberSaveable(saver = CitySaver) { City("Paris", "France") }
```

Trade-off: `rememberSaveable` only suits small UI state (Bundle has a size limit). Large or business state belongs in a `ViewModel` (which already survives config changes) or persistent storage.

**📚 Reference:** https://outcomeschool.com/blog/remember-vs-remembersaveable

---

## 18. Lifecycle of a Composable

**Composable lifecycle** means the three phases: entering composition, recomposing zero or more times, and leaving composition.

1. **Enter the Composition** — first time it's invoked and emits nodes.
2. **Recompose 0..n times** — re-invoked when its observed state changes.
3. **Leave the Composition** — removed (e.g., an `if` becomes false, or it scrolls off in a non-keyed list).

```
Enters Composition ──▶ Recomposes (0..n) ──▶ Leaves Composition
```

`remember` values live from enter to leave. `DisposableEffect.onDispose` and the cancellation of `LaunchedEffect`/`rememberCoroutineScope` coroutines all fire on *leave*. **Identity** matters: Compose uses call-site position (and `key { }`) to decide whether a recompose preserves the same instance or replaces it (leave + enter).

**📚 Reference:** https://developer.android.com/jetpack/compose/lifecycle

---

## 19. Handling lifecycle events in Compose

**Lifecycle coordination** means observing Android's lifecycle events (like resume or pause) inside a composable using a `DisposableEffect`.

For the **Android `Lifecycle`** (Activity/Fragment ON_RESUME, ON_PAUSE, etc.), observe `LocalLifecycleOwner`:

```kotlin
@Composable
fun OnLifecycleEvent(onEvent: (event: Lifecycle.Event) -> Unit) {
    val lifecycleOwner = LocalLifecycleOwner.current
    val currentOnEvent by rememberUpdatedState(onEvent)
    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event -> currentOnEvent(event) }
        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
    }
}
```

Modern helpers in `lifecycle-runtime-compose` make this terse:

```kotlin
LifecycleResumeEffect(Unit) {
    startSomething()
    onPauseOrDispose { stopSomething() }
}
LifecycleStartEffect(Unit) { /* ... */ onStopOrDispose { /* ... */ } }

val state by lifecycleOwner.lifecycle.currentStateAsState()  // observe state as Compose State
```

Note the `rememberUpdatedState` + `DisposableEffect(lifecycleOwner)` pattern: the effect keys on the owner (stable), while always calling the latest `onEvent`.

---

## 20. Performance best practices

**Compose performance optimisation** means techniques like using stable parameters, derivedStateOf, lazy layouts, and minimising layout recalculations.

Stability & skipping.** Ensure parameters are *stable* so Compose can skip recomposition. Stable = immutable, or notifies Compose of changes. Mark types `@Immutable`/`@Stable`, use `val`, and prefer immutable/persistent collections (`kotlinx.collections.immutable`'s `ImmutableList`) since `List<T>` is treated as unstable.

```kotlin
@Immutable
data class UiState(val items: ImmutableList<Item>, val title: String)
```
With the Kotlin 2.x Compose compiler, **strong skipping** is enabled by default, which lets Compose skip composables with unstable parameters by comparing instance equality — but stability annotations still produce the best results.

**2. Keys in lazy lists.** Provide stable `key`s so reordering/inserts don't recompose everything:
```kotlin
LazyColumn { items(list, key = { it.id }) { ItemRow(it) } }
```

**3. Defer state reads.** Read state as late (as low) as possible, ideally in layout/draw phases via lambda-based modifiers, so reads don't trigger the composition phase:
```kotlin
// Good: read offset only at layout time, no recomposition
Box(Modifier.offset { IntOffset(0, scroll.value) })
// Bad: reading scroll.value here recomposes on every scroll frame
Box(Modifier.offset(y = scroll.value.dp))
```
Use `derivedStateOf` to throttle high-frequency state into discrete changes.

**4. Avoid backwards writes** — never write to state during composition that you also read; it causes endless recomposition.

**5. Lambdas & remember.** Hoist heavy computations into `remember(keys)`; avoid allocating new lambdas/objects each recomposition where it breaks skipping.

**6. Don't read state too high.** Keep state ownership narrow so invalidation scopes stay small.

Diagnose with the **Layout Inspector** recomposition counts and the **Compose compiler metrics/reports** (stability inference).

---

## 21. Using Compose and Views together (interop)

**UI interoperability** means combining toolkits using `ComposeView` to host composables inside XML layouts, and `AndroidView` to host classic Views inside composables.

**Views inside Compose** — use `AndroidView`:
```kotlin
AndroidView(
    factory = { context -> MapView(context).apply { /* init */ } },
    update = { mapView -> mapView.setZoom(zoom) }   // re-runs on state change
)
```
For XML layouts use `AndroidViewBinding`.

**Compose inside Views** — add a `ComposeView`:
```kotlin
// In XML, place <androidx.compose.ui.platform.ComposeView .../> then:
composeView.setContent { MyComposable() }
```
A whole Activity uses `setContent { }` on `ComponentActivity`.

Trade-offs: interop has a bridging cost; mixing the two scrolling systems (e.g., a `RecyclerView` of `ComposeView`s, or deeply nested `AndroidView`s) can hurt performance. Themes need bridging (`MdcTheme`/`AppCompatTheme` adapters historically, or mirror values manually). Use interop for migration, not as a long-term architecture.

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_outcomeschool-softwareengineering-tech-activity-7235142707138961408-40hQ

---

## 22. State Hoisting

**CompositionLocal** means a mechanism to pass data implicitly down the composition tree without parameter drilling.

The canonical signature is **value down, events up**:

```kotlin
// Stateless
@Composable
fun Counter(count: Int, onIncrement: () -> Unit) {
    Button(onClick = onIncrement) { Text("Count: $count") }
}

// Stateful owner
@Composable
fun CounterScreen() {
    var count by rememberSaveable { mutableStateOf(0) }
    Counter(count = count, onIncrement = { count++ })
}
```

Benefits: **single source of truth**, **encapsulation** (only owners mutate), **shareability/reusability**, **interceptability** (owner can validate/transform events), and **testability**. Hoist to the *lowest common ancestor* of all composables that read or write the state. This is the building block of unidirectional data flow (Section 30).

---

## 23. CompositionLocal

**rememberUpdatedState** means a utility that stores a value in a mutable reference so that long-running side effects can access the latest value without restarting.

```kotlin
val LocalElevation = compositionLocalOf { 4.dp }   // dynamic, tracks reads

@Composable
fun App() {
    CompositionLocalProvider(LocalElevation provides 8.dp) {
        Card(elevation = CardDefaults.cardElevation(LocalElevation.current)) { /* ... */ }
    }
}
```

- `compositionLocalOf` — re-reads invalidate only the reading composables; use when the value changes.
- `staticCompositionLocalOf` — cheaper, but **any** change recomposes the whole provided subtree; use for rarely/never-changing values.

Built-ins: `LocalContext`, `LocalConfiguration`, `LocalDensity`, `LocalLifecycleOwner`, `MaterialTheme`'s colors/typography (themselves CompositionLocals).

Use sparingly: it makes data flow implicit and harder to trace. Prefer explicit parameters for data a composable's correctness depends on; reserve CompositionLocal for ambient, tree-wide concerns.

**📚 Reference:** https://developer.android.com/jetpack/compose/compositionlocal

---

## 24. Jetpack Compose Phases

**Compose phases** means the three sequential steps—Composition (what to show), Layout (where to place), and Drawing (how to paint)—that compose the UI.

1. **Composition** — *what* to show. Runs `@Composable` functions, builds/updates the UI tree (LayoutNodes).
2. **Layout** — *where* to place it. For each node: **measure** children, then **place** them (single measurement pass per node).
3. **Drawing** — *how* it renders. Each node draws into the canvas.

```
State ──▶ Composition ──▶ Layout (measure + place) ──▶ Drawing ──▶ pixels
```

The key performance insight: **state reads are tracked per phase**. If a value is only needed for layout or drawing, read it there (via lambda-accepting modifiers like `offset { }`, `graphicsLayer { }`, `drawBehind { }`) to **skip the composition phase** entirely on change. A value read in composition invalidates all three phases; a value read only in drawing invalidates only drawing.

```kotlin
// Animating color: read in draw phase, skip composition + layout
Box(Modifier.drawBehind { drawRect(color.value) })
```

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_androiddev-compose-jetpack-activity-7359438711530426369-IQ8S

---

## 25. The Modifier

**derivedStateOf** means a state utility that computes a value from other state parameters and recomposes only when the computed output changes.

Modifiers form an **ordered chain**, and **order matters**.

```kotlin
Text(
    "Hi",
    modifier = Modifier
        .padding(16.dp)        // padding applied first...
        .background(Color.Gray) // ...so background does NOT cover this padding
        .clickable { }
        .size(100.dp)
)
```
Swapping `.padding` and `.background` changes whether the gray fills the padded region or not.

Key points:
- Pass a `modifier: Modifier = Modifier` as the **first optional parameter** of your composables and apply it to the root element (convention for reusability).
- Modifiers are immutable; chaining returns a new `Modifier`.
- For custom behaviour, prefer the `Modifier.Node` API (`Modifier.then`, custom `layout`, `drawWithCache`) over deprecated `composed {}` for performance.
- Common: `fillMaxSize`, `weight` (in Row/Column scope), `align`, `clip`, `border`, `pointerInput`, `semantics`.

**📚 Reference:** https://developer.android.com/jetpack/compose/modifiers

---

## 26. Semantics

**Compose Canvas** means a composable layout that provides a drawing scope to paint custom 2D graphics.

Compose generates it automatically for standard components; you augment it via `Modifier.semantics`.

```kotlin
Icon(
    Icons.Default.Favorite, contentDescription = null,
    modifier = Modifier
        .clickable { toggle() }
        .semantics {
            contentDescription = "Like post"
            role = Role.Button
            stateDescription = if (liked) "Liked" else "Not liked"
        }
)
```

- `contentDescription` — label for non-text elements (use `null` for purely decorative icons).
- `mergeDescendants = true` — merge children into one accessible node (e.g., a list item).
- `clearAndSetSemantics { }` — replace the subtree's semantics entirely.
- `testTag("...")` — stable node id used by `composeTestRule.onNodeWithTag(...)`.

The same tree powers **UI testing**, so good semantics improve both accessibility and test reliability.

**📚 Reference:** https://developer.android.com/jetpack/compose/semantics

---

## 27. Handling user input and events

**Gesture handling** means detecting user interactions like taps, drags, or swipes using click modifiers or pointer input detectors.

}, onClick = { ... })
    .toggleable(value = checked, onValueChange = onCheckedChange)
    .draggable(state = rememberDraggableState { delta -> offset += delta }, orientation = Orientation.Horizontal)
    .scrollable(...)
```

**Low-level via `pointerInput`:**
```kotlin
Modifier.pointerInput(Unit) {
    detectTapGestures(onDoubleTap = { ... }, onLongPress = { ... })
    // or detectDragGestures, detectTransformGestures (pinch/zoom/rotate)
}
```

**Text input** uses state-hoisted callbacks:
```kotlin
var text by rememberSaveable { mutableStateOf("") }
TextField(
    value = text, onValueChange = { text = it },
    keyboardOptions = KeyboardOptions(imeAction = ImeAction.Done),
    keyboardActions = KeyboardActions(onDone = { submit() })
)
```

Principle: handle the event in a callback (not in the composable body), then update hoisted state — driving recomposition through the normal UDF loop.

---

## 28. Navigation in Compose

**Compose Navigation** means a navigation framework that manages composable screens as destinations on a back stack using type-safe routes.

As of 2025–2026 the recommended approach is **type-safe navigation** with `@Serializable` route classes.

```kotlin
@Serializable object Home
@Serializable data class Profile(val userId: String)

@Composable
fun AppNav() {
    val navController = rememberNavController()
    NavHost(navController, startDestination = Home) {
        composable<Home> {
            HomeScreen(onUser = { id -> navController.navigate(Profile(id)) })
        }
        composable<Profile> { backStackEntry ->
            val profile: Profile = backStackEntry.toRoute()
            ProfileScreen(userId = profile.userId)
        }
    }
}
```

Key concepts:
- `NavController` owns the back stack; `NavHost` maps destinations to composables.
- Type-safe routes replace string routes + manual argument parsing.
- Scope a `ViewModel` to a destination with `viewModel()` inside `composable { }`, or to a nav graph via `navController.getBackStackEntry(route)`.
- `navController.popBackStack()`, `navigate(route) { popUpTo(...) { inclusive = true }; launchSingleTop = true }` control the stack.
- Newer **Navigation 3** (`androidx.navigation3`) takes a more Compose-native, list-of-keys back-stack approach; either may appear depending on the codebase's adoption.

---

## 29. Orientation changes

**Orientation handling** means adapting Compose layouts to screen rotations using `Configuration` details or size-class multipliers.

Strategies:

1. **`rememberSaveable`** for small UI state — survives via the saved-instance Bundle:
```kotlin
var query by rememberSaveable { mutableStateOf("") }
```
2. **`ViewModel`** for screen/business state — survives configuration changes by design.
3. **Adapt layout** to the new configuration using **window size classes** (the modern, recommended responsive API) rather than reading raw orientation:
```kotlin
val windowSizeClass = calculateWindowSizeClass(activity)
when (windowSizeClass.widthSizeClass) {
    WindowWidthSizeClass.Compact -> SinglePane()
    else -> TwoPane()
}
```
You can read `LocalConfiguration.current.orientation`, but size classes generalize better across foldables/tablets/multi-window.

Avoid `android:configChanges` to "fix" rotation unless you specifically want to handle it manually; prefer letting state survive via the mechanisms above.

---

## 30. Unidirectional Data Flow

**Lazy layouts** means list components (`LazyColumn`/`LazyRow`) that only measure and compose visible items to optimise scroll performance.

```
        events (up)
ViewModel  ◀──────────  Composable
    │
    └────────────────▶  state (down)
```

```kotlin
class TodoViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(TodoUiState())
    val uiState: StateFlow<TodoUiState> = _uiState.asStateFlow()

    fun onAdd(text: String) = _uiState.update { it.copy(items = it.items + text) }
}

@Composable
fun TodoScreen(vm: TodoViewModel = viewModel()) {
    val state by vm.uiState.collectAsStateWithLifecycle()   // state down
    TodoContent(state = state, onAdd = vm::onAdd)            // events up
}
```

Benefits: **single source of truth**, predictable/testable state transitions, no two-way binding bugs, and clean separation between stateless UI and state owner. UDF is state hoisting (Section 22) applied at architectural scale, typically with the `ViewModel` as the state holder.

---

## 31. Custom Layouts

**Custom Layout composable** means creating custom layouts by measuring children manually and positioning them on coordinates.

**`layout` modifier** — adjust a single element:
```kotlin
fun Modifier.firstBaselineToTop(top: Dp) = layout { measurable, constraints ->
    val placeable = measurable.measure(constraints)
    val baseline = placeable[FirstBaseline]
    val height = placeable.height + top.roundToPx() - baseline
    layout(placeable.width, height) {
        placeable.placeRelative(0, top.roundToPx() - baseline)
    }
}
```

**`Layout` composable** — arrange multiple children (e.g., a simple vertical stack):
```kotlin
@Composable
fun MyColumn(modifier: Modifier = Modifier, content: @Composable () -> Unit) {
    Layout(modifier = modifier, content = content) { measurables, constraints ->
        val placeables = measurables.map { it.measure(constraints) }   // measure once
        val width = placeables.maxOf { it.width }
        val height = placeables.sumOf { it.height }
        layout(width, height) {
            var y = 0
            placeables.forEach { p -> p.placeRelative(0, y); y += p.height }
        }
    }
}
```

Rules: each child must be measured **exactly once**; respect incoming `Constraints`. For children that must be composed lazily or measured based on other children's sizes, use **`SubcomposeLayout`** (the only sanctioned way to measure more than once, at a cost). Custom layouts run in the **layout phase**, so they avoid recomposition when only positions change.

---
