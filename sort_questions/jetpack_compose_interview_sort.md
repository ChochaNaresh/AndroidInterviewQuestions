# Jetpack Compose — Short Answers (Quick Revision)

> Condensed answers for rapid review before an interview. For full explanations and code see `jetpack_compose_interview.md`.

---

## 1. Jetpack Compose vs Android View System

Compose is a **declarative**, Kotlin-only toolkit (UI = `@Composable` functions, recomposition updates UI from state, no inflation/`findViewById`). The View system is **imperative** (XML + mutable `View` objects, manual setters, two-pass measure). Compose avoids two-pass measurement pathologies; the cost is a learning curve and recomposition overhead if misused.

---

## 2. Declarative UI in Jetpack Compose

You describe what the UI should look like for a given state and the framework updates the screen — `UI = f(state)`. You never hold or mutate widget objects; when state changes, Compose recomposes and reconciles the difference.

---

## 3. Declarative UI vs Imperative UI

Imperative: you mutate the UI tree step by step (bug surface = state desync — forget a setter and the UI lies). Declarative: you re-describe the whole UI for the new state and the runtime diffs it — easier to reason about, harder to desync, but you give up fine-grained manual control and must understand recomposition for performance.

---

## 4. What are Composable functions?

Functions annotated `@Composable` that emit UI into the Composition. They can only be called from other composables, return `Unit` (not view objects), and must be side-effect free and idempotent (may run many times, any order, any thread, or be skipped). Don't mutate global state or depend on execution order in their body.

---

## 5. What is Recomposition?

Compose re-invoking composables when the state they read changes, updating only affected UI. It skips composables whose stable inputs are unchanged, is scoped to the smallest reading scope, and may run out of order/in parallel/be cancelled. Reads are tracked via the snapshot system: reading `State` subscribes a scope; writing invalidates it.

---

## 6. What is State in Compose?

Any value that triggers recomposition when changed; core type `State<T>`/`MutableState<T>` via `mutableStateOf`, backed by the snapshot system. Must be stored in `remember` to survive recomposition. Other factories: `mutableStateListOf`/`mutableStateMapOf`, primitive-specialized `mutableIntStateOf`, and adapters like `collectAsStateWithLifecycle`/`produceState`/`derivedStateOf`.

---

## 7. How does state management work?

Three pillars: observable state (`mutableStateOf` + snapshot system), `remember` (persist across recompositions), and recomposition (re-read + update UI). The scalable pattern combines state hoisting with unidirectional data flow, often anchored in a `ViewModel` exposing immutable state down and events up. Keep state in the lowest common owner.

---

## 8. Stateful vs Stateless composables

Stateful composables own their own state (via `remember`) — convenient but harder to reuse/test. Stateless composables hold no state; they receive state via parameters and report changes via callbacks — reusable, testable, previewable. Best practice: write stateless composables and hoist state to a stateful caller.

---

## 9. What are side effects?

Any change that escapes a composable's scope (starting a coroutine, DB write, showing a Snackbar, registering a callback). Because composables run unpredictably, wrap such work in Effect APIs: `LaunchedEffect`, `DisposableEffect`, `SideEffect`, `produceState`, `rememberCoroutineScope`, `rememberUpdatedState`, `derivedStateOf`, `snapshotFlow`.

---

## 10. LaunchedEffect vs DisposableEffect

`LaunchedEffect` runs a suspending block in a coroutine — launches on enter, cancels on leave, cancels-and-relaunches on key change (suspend/async work). `DisposableEffect` runs setup and requires `onDispose {}` cleanup — for register-then-unregister (listeners, observers, sensors). Use `LaunchedEffect(Unit)` to run once.

---

## 11. rememberCoroutineScope and use cases

Returns a `CoroutineScope` bound to the call-site composition, used to launch coroutines from **non-composable callbacks** (e.g. `onClick`) where you can't use `LaunchedEffect`. Cancelled when the composable leaves. Use cases: animating on click, `animateScrollTo`, showing Snackbars, drawer open/close.

---

## 12. Observing Flow and LiveData in Compose

Convert streams to Compose `State`: `collectAsStateWithLifecycle()` for StateFlow/Flow (lifecycle-aware, collects only when ≥ STARTED — preferred), `observeAsState()` for LiveData, `subscribeAsState()` for RxJava. Prefer `collectAsStateWithLifecycle` over plain `collectAsState()` on Android to avoid wasted background work.

---

## 13. Handling asynchronous operations

Choose by trigger: composition/keys → `LaunchedEffect`; user events → `rememberCoroutineScope().launch`; async/callback source → `produceState`; stream → `collectAsStateWithLifecycle`. Best practice: keep heavy/business work in the `ViewModel` (`viewModelScope`) exposed as `StateFlow`; reserve effects for UI-scoped concerns.

---

## 14. Converting non-Compose state into Compose state

Use `produceState` to bridge callback/async APIs into observable `State`, cleaning up with `awaitDispose`. Other bridges: `collectAsStateWithLifecycle`/`observeAsState`/`subscribeAsState` for streams, and `snapshotFlow { }` for the reverse (State → Flow). `produceState` uses `remember` + `LaunchedEffect` internally, so it's lifecycle-safe.

---

## 15. derivedStateOf

Creates a `State` computed from other state reads that notifies readers only when the computed result actually changes. Use when a frequently-changing input maps to a rarely-changing output (e.g. `firstVisibleItemIndex > 0`) — wrap in `remember { derivedStateOf { } }`. Anti-pattern: don't use it for a 1:1 transform of inputs that change together.

---

## 16. rememberUpdatedState

Captures the latest value so a long-running effect can read it without restarting. Used when `LaunchedEffect(Unit)` must run once but a lambda/value (e.g. `onTimeout`) may change across recompositions — it decouples "use the latest value" from "restart the effect."

---

## 17. remember vs rememberSaveable

`remember` stores in the composition slot table — survives recomposition but is lost on config change and process death. `rememberSaveable` also writes to a `Bundle`, surviving config changes and process death. It auto-handles Bundle-compatible types; supply a `Saver` for custom ones. Suits only small UI state — large/business state belongs in a `ViewModel`.

---

## 18. Lifecycle of a Composable

Three events: enter the Composition, recompose 0..n times, leave the Composition. `remember` values live enter→leave; `DisposableEffect.onDispose` and `LaunchedEffect`/scope cancellation fire on leave. Identity is based on call-site position (and `key { }`), which decides whether a recompose preserves or replaces an instance.

---

## 19. Handling lifecycle events in Compose

For composition lifecycle use Effect APIs. For the Android `Lifecycle` (ON_RESUME, etc.), observe `LocalLifecycleOwner` with `DisposableEffect(lifecycleOwner)` + a `LifecycleEventObserver` (use `rememberUpdatedState` for the callback). Modern helpers in `lifecycle-runtime-compose`: `LifecycleResumeEffect`, `LifecycleStartEffect`, `currentStateAsState()`.

---

## 20. Performance best practices

- Ensure parameters are **stable** so Compose can skip; mark `@Immutable`/`@Stable`, use immutable collections (`List` is unstable). Strong skipping is on by default with the Kotlin 2.x compiler.
- Provide stable `key`s in lazy lists.
- **Defer state reads** to layout/draw via lambda modifiers (`offset { }`, `drawBehind { }`).
- Avoid backwards writes (writing state you also read during composition).
- Hoist heavy work into `remember`; keep state ownership narrow.

---

## 21. Using Compose and Views together (interop)

Both directions work for incremental migration. **Views in Compose**: `AndroidView` (or `AndroidViewBinding` for XML). **Compose in Views**: `ComposeView.setContent { }` (or `setContent {}` on a `ComponentActivity`). Trade-offs: bridging cost, performance issues when mixing scrolling systems, and theme bridging. Use for migration, not long-term architecture.

---

## 22. State Hoisting

Moving state up to a composable's caller, making the child stateless — "value down, events up." Benefits: single source of truth, encapsulation, reusability, interceptability, testability. Hoist to the lowest common ancestor of all composables that read/write the state. It's the building block of unidirectional data flow.

---

## 23. CompositionLocal

Provides a value implicitly to a subtree, avoiding prop drilling of cross-cutting data (theme, density, locale). `compositionLocalOf` invalidates only reading composables (use for changing values); `staticCompositionLocalOf` is cheaper but recomposes the whole subtree on change (rarely-changing values). Built-ins: `LocalContext`, `LocalDensity`, `LocalLifecycleOwner`. Use sparingly — prefer explicit parameters for correctness-critical data.

---

## 24. Jetpack Compose Phases

Each frame runs up to three phases: **Composition** (what to show — runs composables, builds LayoutNodes), **Layout** (where — measure + place, single pass per node), **Drawing** (how — draws to canvas). State reads are tracked per phase: reading a value only in layout/draw (via lambda modifiers) skips composition on change — a value read in composition invalidates all three phases.

---

## 25. The Modifier

Decorates/configures a composable (size, padding, background, click, layout, semantics, drawing) as an **ordered chain** where order matters (e.g. `padding` before `background` vs after changes the result). Pass `modifier: Modifier = Modifier` as the first optional parameter and apply it to the root. Modifiers are immutable; prefer the `Modifier.Node` API over deprecated `composed {}`.

---

## 26. Semantics

A parallel tree describing the UI's meaning, consumed by accessibility (TalkBack), autofill, and UI tests. Compose generates it automatically; augment with `Modifier.semantics`. Key props: `contentDescription` (null for decorative), `mergeDescendants`, `clearAndSetSemantics`, `testTag` (for `onNodeWithTag`). The same tree powers testing, so good semantics improve both.

---

## 27. Handling user input and events

High-level gesture modifiers: `clickable`, `combinedClickable`, `toggleable`, `draggable`, `scrollable`. Low-level via `pointerInput { detectTapGestures/detectDragGestures/detectTransformGestures }`. Text input uses state-hoisted callbacks (`TextField(value, onValueChange)`). Principle: handle the event in a callback, then update hoisted state through the UDF loop.

---

## 28. Navigation in Compose

Use Navigation Compose with **type-safe** `@Serializable` route classes (2025–2026 recommended). `NavController` owns the back stack; `NavHost` maps destinations (`composable<Route>`) to composables; read args with `backStackEntry.toRoute()`. Scope ViewModels to a destination/graph, and control the stack with `popBackStack`, `popUpTo`, `launchSingleTop`. Navigation 3 is the newer Compose-native approach.

---

## 29. Orientation changes

Rotation recreates the Activity, losing `remember`-only state. Strategies: `rememberSaveable` for small UI state, `ViewModel` for screen/business state, and **window size classes** (`calculateWindowSizeClass`) to adapt layout (preferred over reading raw orientation, generalizes to foldables/tablets). Avoid `android:configChanges` unless you specifically want manual handling.

---

## 30. Unidirectional Data Flow

State flows down, events flow up — a single predictable loop, typically with a `ViewModel` as state holder (`StateFlow` collected via `collectAsStateWithLifecycle`, events as method references). Benefits: single source of truth, predictable/testable transitions, no two-way binding bugs, clean separation. It's state hoisting applied at architectural scale.

---

## 31. Custom Layouts

When `Row`/`Column`/`Box` aren't enough, use the `layout` modifier (adjust a single element) or the `Layout` composable (arrange multiple children) to measure and place yourself. Rules: measure each child **exactly once** and respect incoming `Constraints`. Use `SubcomposeLayout` (at a cost) when you must measure more than once or compose lazily. Custom layouts run in the layout phase, avoiding recomposition when only positions change.
