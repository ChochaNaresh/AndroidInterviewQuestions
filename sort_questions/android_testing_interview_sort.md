# Android Testing — Short Answers (Quick Revision)

> Condensed answers for rapid review before an interview. For full explanations and code see `android_testing_interview.md`.

---

## 1. What is the testing pyramid?

**Testing pyramid** means a testing model that recommends a large base of fast unit tests, a middle layer of integration tests, and a small peak of end-to-end UI tests.

A model for the ideal proportion of tests: ~70% **unit** (fast, isolated, JVM), ~20% **integration** (components cooperating), ~10% **UI/E2E** (slow, flaky, on device). The principle: push tests down the pyramid — prefer many fast unit tests over few slow UI tests.

---



## 2. Unit tests vs. instrumented tests

**Local unit tests vs instrumented tests** means the distinction between tests running on the local JVM without emulator overhead, and tests running inside a physical or virtual Android device.

- **Unit tests** — `src/test/`, run on local JVM, millisecond-fast, Android framework stubbed; for business logic/ViewModels.
- **Instrumented tests** — `src/androidTest/`, run on device/emulator with the real framework; for UI, navigation, Room.

`android.jar` methods throw "not mocked" on the JVM, so framework code needs Robolectric or an instrumented test.

---



## 3. What is a unit test and what should it do?

**Unit test** means an automated test that isolates and verifies the behaviour of the smallest testable piece of source code, usually a class or function.

Verifies the smallest testable piece (a function/class) in isolation. It should test one logical thing, be isolated (fakes/mocks for deps), be deterministic and fast, follow Arrange–Act–Assert, and run in CI. Remember **FIRST**: Fast, Isolated, Repeatable, Self-validating, Timely.

---



## 4. What is an instrumented test?

**Instrumented test** means a test that runs on a physical device or emulator to test components that depend directly on the Android framework lifecycle or resources.

A test that runs on a device/emulator inside a real Android process (built into an APK, run by `AndroidJUnitRunner`), with access to real framework APIs. Use it for UI (Espresso/Compose), cross-app flows (UI Automator), and Room/WorkManager/file/IO integration.

---



## 5. How do you write testable code? What makes code hard to test?

**Testable code design** means structuring software with loose coupling, dependency injection, and clean boundaries to facilitate mock insertion during testing.

**Hard to test:** concrete dependencies created internally, static methods/singletons, tight coupling to the framework, side effects/non-determinism, god classes. **Testable:** inject dependencies via constructor, program to interfaces, inject `CoroutineDispatcher` and `Clock`, keep business logic free of the Android SDK, and apply single responsibility.

---



## 6. JUnit — structure, annotations, assertions, JUnit4 vs JUnit5

**JUnit** means the standard Java/Kotlin testing framework used to structure, annotate, and run automated unit tests.

JUnit discovers/runs/reports tests; Android defaults to **JUnit 4** (JUnit 5 works for JVM tests but not the device runner). JUnit 4 lifecycle: `@Before`/`@After`/`@BeforeClass`/`@AfterClass`, `@Test`, `@Ignore`. Assertions: `assertEquals`, `assertTrue`, `assertThrows`; many prefer Truth/AssertJ. JUnit 5 uses `@BeforeEach`/`@BeforeAll`, an extension model, and `@Nested`/`@DisplayName`.

---



## 7. Why use Mockito? Core API

**Mockito** means a Java mocking framework used to create mock objects and stub behaviours to isolate the system under test.

Mockito creates test doubles to isolate the SUT: **stub** returns (`when(...).thenReturn(...)`), **verify** interactions (`verify(...)`), throw exceptions, and **capture** args (`ArgumentCaptor`). On Kotlin (final by default) the inline mock maker handles final classes — it's the **default since Mockito 5** (the separate `mockito-inline` artifact was only needed in older versions); add **mockito-kotlin** for idiomatic `mock()`/`whenever()`. Matchers: `any()`, `eq()`; modes: `times(n)`, `never()`.

---



## 8. MockK — the Kotlin mocking library

**MockK** means a Kotlin-first mocking library designed to support Kotlin features like coroutines, properties, and companion objects.

A Kotlin-native mocking library that handles final classes, objects, extension functions, and especially **suspend functions** (`coEvery`/`coVerify`). Features: `every { } returns/throws/answers`, relaxed mocks (`relaxed = true`), spies (`spyk`), `mockkStatic`/`mockkObject`/`mockkConstructor`, `slot<T>()` for capturing, and annotations (`@MockK`, `@RelaxedMockK`, `@InjectMockKs`).

---



## 9. Mockito vs. MockK

**Mockito vs MockK** means the choice between a Java-based mocking framework requiring helper adapters for Kotlin (Mockito), and a native Kotlin mocking library (MockK).

MockK is Kotlin-native (final classes, suspend funcs, objects, extensions work out of the box; strict by default; `every { } returns`). Mockito is a Java library with a Kotlin wrapper (the inline mock maker handles finals — default since Mockito 5; still awkward with suspend/objects; `whenever().thenReturn()`). Rule of thumb: MockK for new Kotlin/coroutine projects; Mockito for Java-heavy/legacy.

---



## 10. Fakes vs. Mocks vs. Stubs vs. Spies

**Test doubles** means the various mock objects used in testing, where **fake** implements simplified working logic, and **mock** verifies specific interaction expectations.

- **Dummy** — fills a signature, unused.
- **Stub** — returns hard-coded answers (state setup).
- **Mock** — programmed with expectations; you **verify interactions** (behaviour).
- **Spy** — wraps a real object, overrides some calls.
- **Fake** — a working lightweight implementation (e.g. in-memory repo).

Key point: mocks verify interactions (brittle); fakes have real logic and assert end state (robust). Google: prefer fakes for your own collaborators, mocks for verifying a side effect.

---



## 11. What is Espresso?

**Espresso** means Google's UI testing framework that provides automatic synchronisation between test actions and the application's UI thread.

Google's UI testing framework for **in-app** instrumented tests; it auto-synchronizes with the UI thread to avoid `sleep()` flakiness. Three blocks: `onView(matcher)` → `.perform(action)` → `.check(assertion)`. Extras: Espresso-Intents, Espresso-Web, IdlingResource. Compose has an equivalent auto-syncing API.

---



## 12. What is UI Automator?

**UI Automator** means a black-box UI testing framework that allows interaction with system-level elements and cross-app workflows.

A framework for **black-box, cross-app** testing that can drive any app and the system UI (notification shade, Home/Back, settings, permissions). Core APIs: `UiDevice`, `UiSelector`/`BySelector`, `UiObject`/`UiObject2`. Use Espresso for your own app, UI Automator when a flow leaves your app or touches system UI.

---



## 13. What is Robolectric? Pros and cons

**Robolectric** means a testing framework that executes Android-dependent tests on the local JVM by providing simulated Android framework shadows.

Runs Android-dependent tests on the local JVM using "shadow" objects simulating framework classes — no device needed. **Pros:** fast (JVM), tests framework-touching code without instrumentation, integrates with AndroidX Test. **Cons:** it's a simulation (false confidence), incomplete coverage, slower than pure JVM tests, lags new SDKs, and hides integration bugs. Sits between unit and instrumented tests.

---



## 14. Testing coroutines — runTest and TestDispatchers

**Coroutine testing** means controlling asynchronous execution using `runTest` and `TestDispatcher` to skip delays and ensure deterministic test assertions.

Use `kotlinx-coroutines-test`. `runTest { }` runs in a `TestScope` with virtual time (skips delays) and fails on leaked coroutines. **StandardTestDispatcher** queues coroutines (needs `advanceUntilIdle()`/`advanceTimeBy()`); **UnconfinedTestDispatcher** runs eagerly. Replace `Dispatchers.Main` (for `viewModelScope`) with a `MainDispatcherRule`. Best practice: inject dispatchers rather than hard-coding `Dispatchers.IO`.

---



## 15. Turbine — testing Kotlin Flow

**Turbine** means a lightweight library from Cash App that simplifies Kotlin Flow testing by converting asynchronous emissions into a synchronous queue.

A library that turns Flow emissions into pull-based suspend calls. Call `.test { }` and use `awaitItem()`, `awaitComplete()`, `awaitError()`, `expectNoEvents()`, `skipItems(n)`, and `cancelAndIgnoreRemainingEvents()` (for hot `StateFlow`). It fails on unconsumed/unexpected events or timeout, catching bugs manual `collect` misses.

---



## 16. Testing a ViewModel with Coroutines and LiveData

**ViewModel LiveData testing** means verifying state changes by swapping the main thread execution with `InstantTaskExecutorRule` and mocking coroutines.

Two rules: **`InstantTaskExecutorRule`** (runs LiveData work synchronously) and **`MainDispatcherRule`** (replaces `Dispatchers.Main`). Stub the repo (e.g. `coEvery`), call the function, `advanceUntilIdle()`, then assert `viewModel.liveData.value`. To assert a sequence (Loading → Success), capture emissions with an `observeForever` observer.

---



## 17. Testing a ViewModel with Flow and StateFlow

**ViewModel Flow testing** means validating Flow emissions using Turbine or collecting emissions inside a test coroutine scope.

`StateFlow` is hot (always has a value, never completes). Best approach: **Turbine** — assert the initial value, trigger the action, await Loading/Success, then `cancelAndIgnoreRemainingEvents()`. Or read `.value` after `advanceUntilIdle()` for the latest state only. Note: `stateIn(WhileSubscribed())` only runs upstream while collected — use `Eagerly` or start collecting in tests.

---



## 18. Code coverage and JaCoCo

**Code coverage** means a metric generated by tools like JaCoCo that measures the percentage of production code executed by automated tests.

Coverage measures how much code tests execute (line/branch/method). It's a guidance metric, not a goal — high coverage with weak assertions is still poor testing. **JaCoCo** instruments bytecode and produces HTML/XML reports; exclude generated/DI/framework code, set a sensible threshold (70–85%), and gate in CI. Soundbite: coverage tells you what *wasn't* tested, not whether tests are good.

---



## 19. Testing Jetpack Compose UI

**Compose UI testing** means verifying declarative layouts by query matching semantic nodes and asserting UI states using Compose test rules.

Compose tests query the **semantics tree**, not a View tree. Use `createComposeRule()` (set content directly) or `createAndroidComposeRule<Activity>()` (need a real Activity). Flow: finders (`onNodeWithText`/`onNodeWithTag` + `testTag`) → assertions (`assertIsDisplayed`) → actions (`performClick`). The rule auto-syncs with recomposition; disable with `mainClock.autoAdvance = false` for animations.

