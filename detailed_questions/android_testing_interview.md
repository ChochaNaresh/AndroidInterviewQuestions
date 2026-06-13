# Android Testing — Interview Preparation Guide

A comprehensive, code-first reference for Android testing interview questions, covering the testing pyramid, unit vs. instrumented tests, JUnit, Mockito and MockK, Espresso, UI Automator, Robolectric, Turbine, coroutine testing, ViewModel testing with LiveData and StateFlow, fakes vs. mocks, and code coverage. All APIs are current as of 2026 (kotlinx-coroutines-test `runTest`, MockK, Turbine 1.x, JUnit4/5).

---

## Table of Contents

1. [What is the testing pyramid?](#1-what-is-the-testing-pyramid)
2. [Unit tests vs. instrumented tests](#2-unit-tests-vs-instrumented-tests)
3. [What is a unit test and what should it do?](#3-what-is-a-unit-test-and-what-should-it-do)
4. [What is an instrumented test?](#4-what-is-an-instrumented-test)
5. [How do you write testable code? What makes code hard to test?](#5-how-do-you-write-testable-code-what-makes-code-hard-to-test)
6. [JUnit — structure, annotations, assertions, JUnit4 vs JUnit5](#6-junit--structure-annotations-assertions-junit4-vs-junit5)
7. [Why use Mockito? Core API](#7-why-use-mockito-core-api)
8. [MockK — the Kotlin mocking library](#8-mockk--the-kotlin-mocking-library)
9. [Mockito vs. MockK](#9-mockito-vs-mockk)
10. [Fakes vs. Mocks vs. Stubs vs. Spies](#10-fakes-vs-mocks-vs-stubs-vs-spies)
11. [What is Espresso?](#11-what-is-espresso)
12. [What is UI Automator?](#12-what-is-ui-automator)
13. [What is Robolectric? Pros and cons](#13-what-is-robolectric-pros-and-cons)
14. [Testing coroutines — runTest and TestDispatchers](#14-testing-coroutines--runtest-and-testdispatchers)
15. [Turbine — testing Kotlin Flow](#15-turbine--testing-kotlin-flow)
16. [Testing a ViewModel with Coroutines and LiveData](#16-testing-a-viewmodel-with-coroutines-and-livedata)
17. [Testing a ViewModel with Flow and StateFlow](#17-testing-a-viewmodel-with-flow-and-stateflow)
18. [Code coverage and JaCoCo](#18-code-coverage-and-jacoco)
19. [Testing Jetpack Compose UI](#19-testing-jetpack-compose-ui)

---

## 1. What is the testing pyramid?

The **testing pyramid** is a model describing the ideal proportion of automated tests by type. It balances confidence against cost (speed, stability, maintenance).

```
            /\
           /  \      UI / End-to-End tests  (~10%)
          /----\     - slow, flaky, expensive; run on device/emulator
         /      \    Integration tests       (~20%)
        /--------\   - verify components working together
       /          \  Unit tests              (~70%)
      /------------\ - fast, isolated, run on the JVM
```

- **Unit tests (base, most numerous):** Test a single class/function in isolation. Run on the local JVM (`src/test/`), millisecond-fast, deterministic. Collaborators are replaced with fakes/mocks.
- **Integration tests (middle):** Verify that several components cooperate correctly — e.g. a repository talking to a DAO and a network layer, or a ViewModel + use case + repository. May use Robolectric or an in-memory Room DB.
- **UI / End-to-end tests (top, fewest):** Drive the real UI as a user would. Run on a device/emulator (`src/androidTest/`) using Espresso or UI Automator. Slowest and most fragile, so kept few.

Google also describes a refinement (small/medium/large tests) with a roughly **70 / 20 / 10** split. The key principle: **push tests down the pyramid** — prefer many fast unit tests over a few slow UI tests.

**📚 Reference:** <https://developer.android.com/training/testing/fundamentals>

---

## 2. Unit tests vs. instrumented tests

| | **Local unit tests** | **Instrumented tests** |
|---|---|---|
| Source set | `src/test/` | `src/androidTest/` |
| Runs on | Local JVM (your machine / CI) | Android device or emulator |
| Speed | Milliseconds | Seconds (deploy + boot) |
| Android framework | Stubbed (`android.jar` throws by default) | Real framework available |
| Typical tools | JUnit, Mockito/MockK, Robolectric, Truth | AndroidJUnitRunner, Espresso, UI Automator |
| Use for | Business logic, ViewModels, use cases, mappers | UI, navigation, DB (Room), system interactions |
| Gradle task | `testDebugUnitTest` | `connectedDebugAndroidTest` |

By default, the Android `android.jar` on the unit-test classpath has every method throw a `RuntimeException("Method ... not mocked")`. That is why pure Android-framework code (e.g. `TextUtils`, `Log`) needs Robolectric or an instrumented test. Keep framework dependencies out of business logic so most of it can be unit-tested on the JVM.

**📚 Reference:** <https://developer.android.com/training/testing/unit-testing/local-unit-tests> · <https://developer.android.com/training/testing/unit-testing/instrumented-unit-tests>

---

## 3. What is a unit test and what should it do?

A **unit test** verifies the behavior of the smallest testable piece of code — usually a single function or class — in isolation from its collaborators.

What it should do:
- **Test one logical thing** with a clear name describing the scenario and expectation.
- **Be isolated** — replace external dependencies (network, DB, time, Android framework) with fakes/mocks so only the unit under test (SUT) is exercised.
- **Be deterministic and fast** — same result every run, no real I/O, no real threads/clocks.
- **Follow Arrange–Act–Assert (AAA)** / Given–When–Then.
- **Run in CI** (GitHub Actions, GitLab CI, CircleCI) on every push to guard code quality and catch regressions early.

```kotlin
class PriceCalculatorTest {

    private val calculator = PriceCalculator()

    @Test
    fun `applies 10 percent discount when total exceeds threshold`() {
        // Arrange
        val items = listOf(Item(price = 60.0), Item(price = 50.0)) // total 110

        // Act
        val result = calculator.totalWithDiscount(items)

        // Assert
        assertEquals(99.0, result, 0.001)
    }
}
```

**FIRST** principles for good unit tests: **F**ast, **I**solated/Independent, **R**epeatable, **S**elf-validating, **T**imely.

**📚 Reference:** <https://developer.android.com/training/testing/unit-testing/local-unit-tests>

---

## 4. What is an instrumented test?

An **instrumented test** runs on a physical device or emulator, inside an actual Android process, so it has access to real framework APIs (`Context`, `Resources`, `SharedPreferences`, Room, sensors, etc.). It is built into an APK and executed by the `AndroidJUnitRunner`.

Use it for:
- UI interaction and assertions (Espresso, Compose UI tests).
- Cross-app / system flows (UI Automator).
- Room database, `WorkManager`, file/IO, `SharedPreferences` integration.

```kotlin
@RunWith(AndroidJUnit4::class)
class GreetingInstrumentedTest {

    @Test
    fun usesAppContext() {
        val context = InstrumentationRegistry.getInstrumentation().targetContext
        assertEquals("com.example.app", context.packageName)
    }
}
```

```kotlin
// app/build.gradle.kts
android {
    defaultConfig {
        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }
}
dependencies {
    androidTestImplementation("androidx.test.ext:junit:1.2.1")
    androidTestImplementation("androidx.test.espresso:espresso-core:3.6.1")
}
```

**📚 Reference:** <https://developer.android.com/training/testing/unit-testing/instrumented-unit-tests>

---

## 5. How do you write testable code? What makes code hard to test?

**What makes code hard to test:**
- **Hard (concrete) dependencies** created inside a class with `new`/constructor calls — you cannot substitute a fake.
- **Static methods and singletons** (`Object.method()`, `Calendar.getInstance()`, `System.currentTimeMillis()`) — global state that cannot be swapped per test.
- **Tight coupling to the Android framework** (`Context`, `Log`, `TextUtils`) inside business logic.
- **Side effects and non-determinism** — real network, real DB, real time/random, real threads.
- **God classes** doing too much; private logic that can only be reached through complex setup.

**How to write testable code:**
- **Dependency injection** — pass collaborators in via the constructor (manual DI or Hilt/Dagger) so tests inject fakes.
- **Program to interfaces / abstractions**, not concrete classes.
- **Inject dispatchers** (`CoroutineDispatcher`) instead of hard-coding `Dispatchers.IO`.
- **Abstract time and randomness** behind an injectable `Clock`/provider.
- **Keep business logic free of the Android SDK** (plain Kotlin in `:domain` modules).
- **Single Responsibility** — small, focused classes.

```kotlin
// Hard to test: concrete dependency + hard-coded dispatcher + static time
class BadViewModel : ViewModel() {
    private val repo = UserRepository()            // can't replace
    fun load() = viewModelScope.launch(Dispatchers.IO) {
        val t = System.currentTimeMillis()         // non-deterministic
    }
}

// Testable: injected dependencies, interface, injected dispatcher & clock
class GoodViewModel(
    private val repo: UserRepository,              // interface
    private val ioDispatcher: CoroutineDispatcher,
    private val clock: Clock,
) : ViewModel() {
    fun load() = viewModelScope.launch(ioDispatcher) {
        val now = clock.now()
        // ...
    }
}
```

---

## 6. JUnit — structure, annotations, assertions, JUnit4 vs JUnit5

**JUnit** is the standard Java/Kotlin testing framework that discovers, runs, and reports tests. Android tooling defaults to **JUnit 4**; **JUnit 5** (Jupiter) is usable for JVM unit tests via a plugin but is not supported on the instrumented (device) runner.

**JUnit 4 lifecycle:**

```kotlin
class CartTest {

    private lateinit var cart: Cart

    @Before fun setUp() { cart = Cart() }      // before each test
    @After  fun tearDown() { cart.clear() }    // after each test

    companion object {
        @BeforeClass @JvmStatic fun initAll() {}  // once before all
        @AfterClass  @JvmStatic fun cleanAll() {}  // once after all
    }

    @Test
    fun addingItem_increasesSize() {
        cart.add(Item("book"))
        assertEquals(1, cart.size)
        assertTrue(cart.contains("book"))
    }

    @Test(expected = IllegalStateException::class)
    fun checkout_emptyCart_throws() {
        cart.checkout()
    }

    @Test @Ignore("flaky on CI")
    fun skipped() {}
}
```

**Common assertions:** `assertEquals`, `assertTrue/False`, `assertNull/NotNull`, `assertSame`, `assertThrows`, `fail()`. Many teams prefer **Truth** (`assertThat(x).isEqualTo(y)`, `assertThat(list).containsExactly(...)`) or AssertJ for fluent, readable assertions.

```kotlin
// JUnit assertThrows (JUnit 4.13+ / JUnit 5)
val ex = assertThrows(IllegalStateException::class.java) { cart.checkout() }
assertEquals("Cart is empty", ex.message)
```

**JUnit 4 vs JUnit 5:**

| | JUnit 4 | JUnit 5 (Jupiter) |
|---|---|---|
| Annotations | `@Before`, `@After`, `@BeforeClass` | `@BeforeEach`, `@AfterEach`, `@BeforeAll` |
| Extensibility | Runners + Rules | Extension model (`@ExtendWith`) |
| Parameterized | `@RunWith(Parameterized)` | `@ParameterizedTest` + sources |
| Nested/Display | limited | `@Nested`, `@DisplayName` |
| Android instrumented | Supported | **Not supported** on device runner |

For Android, JUnit 4 with `@get:Rule` (e.g. `InstantTaskExecutorRule`, `MainDispatcherRule`) remains the default.

---

## 7. Why use Mockito? Core API

**Mockito** is a Java mocking framework that creates **test doubles** so you can isolate the unit under test, stub return values, and verify interactions. It lets you test a class without its real (slow, non-deterministic, or unavailable) collaborators.

Use Mockito to:
- **Stub** a collaborator's return value (`when(...).thenReturn(...)`).
- **Verify** that the SUT called a collaborator correctly (`verify(...)`).
- **Throw** exceptions to test error paths.
- **Capture** arguments with `ArgumentCaptor`.

On Kotlin, classes/methods are `final` by default, so you need the **inline mock maker** to mock them. Since **Mockito 5** the inline mock maker is the default (in older versions you added the `mockito-inline` artifact). For idiomatic Kotlin syntax, add **mockito-kotlin** (`mock()`, `whenever()`, `verify(view).method()`).

```kotlin
// dependencies (JVM unit test)
// testImplementation("org.mockito:mockito-core:5.x")
// testImplementation("org.mockito.kotlin:mockito-kotlin:5.x")

class UserPresenterTest {

    private val repo: UserRepository = mock()
    private val view: UserView = mock()
    private val presenter = UserPresenter(repo, view)

    @Test
    fun showsUserName_whenLoadSucceeds() {
        whenever(repo.getUser(1)).thenReturn(User(1, "Ada"))   // stub

        presenter.load(1)

        verify(view).showName("Ada")                            // verify interaction
        verify(repo).getUser(1)
        verifyNoMoreInteractions(view)
    }

    @Test
    fun showsError_whenRepoThrows() {
        whenever(repo.getUser(any())).thenThrow(RuntimeException("boom"))
        presenter.load(1)
        verify(view).showError()
    }

    @Test
    fun capturesSavedUser() {
        val captor = argumentCaptor<User>()
        presenter.save("Linus")
        verify(repo).save(captor.capture())
        assertEquals("Linus", captor.firstValue.name)
    }
}
```

Argument matchers: `any()`, `eq()`, `anyInt()`, `argThat { ... }`. Stubbing styles: `thenReturn`, `thenThrow`, `thenAnswer`. Verification modes: `times(n)`, `never()`, `atLeastOnce()`, `verifyNoInteractions()`.

**📚 Reference:** <https://site.mockito.org/>

---

## 8. MockK — the Kotlin mocking library

**MockK** is a mocking library built from the ground up **for Kotlin**. It natively understands Kotlin features that trip up Mockito: `final` classes (no extra plugin needed), `object`/singletons, extension functions, top-level functions, and especially **suspend functions** (`coEvery`/`coVerify`).

```kotlin
// testImplementation("io.mockk:mockk:1.13.x")

class OrderServiceTest {

    private val repo: OrderRepository = mockk()
    private val service = OrderService(repo)

    @Test
    fun placesOrder() = runTest {
        // stub a suspend function with coEvery
        coEvery { repo.save(any()) } returns 42L

        val id = service.place(Order(item = "book"))

        assertEquals(42L, id)
        coVerify(exactly = 1) { repo.save(match { it.item == "book" }) }
    }
}
```

Key features:
- **`every { } returns / throws / answers`** for normal functions; **`coEvery { }`** for `suspend` functions.
- **`verify { }` / `coVerify { }`** with modes `exactly`, `atLeast`, `atMost`, `verifyOrder`, `verifySequence`.
- **`mockk(relaxed = true)`** — a *relaxed mock* returns sensible defaults (0, "", empty collections, a relaxed mock) for unstubbed calls, so you only stub what matters. **`relaxUnitFun = true`** relaxes only `Unit`-returning functions.
- **`spyk(realObject)`** — a *spy* (partial mock): real implementation by default, override selected calls.
- **`mockkStatic(...)` / `mockkObject(...)` / `mockkConstructor(...)`** — mock static, object, and constructor calls.
- **`slot<T>()`** with `capture(slot)` to capture arguments.
- Annotations: `@MockK`, `@RelaxedMockK`, `@SpyK`, `@InjectMockKs`, initialized with `MockKAnnotations.init(this)`.

```kotlin
class CaptureExample {
    private val repo = mockk<UserRepository>(relaxed = true)

    @Test
    fun capturesArg() {
        val slot = slot<User>()
        every { repo.save(capture(slot)) } returns Unit
        UserService(repo).register("Grace")
        assertEquals("Grace", slot.captured.name)
    }
}
```

**📚 Reference:** <https://mockk.io/>

---

## 9. Mockito vs. MockK

| | **Mockito (+ mockito-kotlin)** | **MockK** |
|---|---|---|
| Origin | Java library, Kotlin wrapper | Kotlin-native |
| `final` classes | inline mock maker (default since Mockito 5) | works out of the box |
| Suspend functions | needs `runTest`; no native API | first-class `coEvery` / `coVerify` |
| Object / static / constructor | awkward | `mockkObject` / `mockkStatic` / `mockkConstructor` |
| Extension functions | not supported well | supported |
| Default behavior | mocks return null/defaults | strict unless `relaxed = true` |
| DSL | `whenever(x).thenReturn(y)` | `every { x } returns y` |

**Rule of thumb:** For new Kotlin projects (especially with coroutines/Flow), **MockK** is the natural choice. **Mockito** remains common in Java-heavy or legacy codebases and is lighter weight. Both isolate the SUT; the choice is about Kotlin ergonomics.

**📚 Reference:** <https://blog.logrocket.com/unit-testing-kotlin-projects-with-mockk-vs-mockito/>

---

## 10. Fakes vs. Mocks vs. Stubs vs. Spies

These are kinds of **test doubles** (Gerard Meszaros' taxonomy). Interviewers especially probe **fake vs. mock**.

- **Dummy** — passed to satisfy a signature, never actually used.
- **Stub** — returns hard-coded answers to calls; no logic. Used for **state** setup.
- **Mock** — a programmed double with **expectations**; you **verify interactions** (which methods were called, how often, with what). Used for **behavior** verification.
- **Spy** — wraps a real object; records calls and lets you override some (partial mock).
- **Fake** — a **working lightweight implementation** (real logic, not production-grade). Classic example: an in-memory repository or in-memory Room DB.

**Fake vs. Mock — the key interview point:**

| | **Mock** | **Fake** |
|---|---|---|
| Has working logic? | No — returns programmed values | Yes — real, simplified behavior |
| What you assert | Interactions (`verify`) | End state / outputs |
| Created by | Mocking framework | Hand-written class |
| Brittleness | Tied to call sequence → brittle | Robust to refactors |
| Best for | Verifying a side effect was triggered | Stateful collaborators (repos, DBs) |

```kotlin
// Fake: a real, in-memory implementation of the interface
class FakeUserRepository : UserRepository {
    private val users = mutableMapOf<Int, User>()
    override suspend fun save(user: User) { users[user.id] = user }
    override suspend fun getUser(id: Int): User? = users[id]
}

@Test
fun fake_storesAndReturnsUser() = runTest {
    val repo = FakeUserRepository()
    val service = UserService(repo)
    service.register(User(1, "Ada"))
    assertEquals("Ada", repo.getUser(1)?.name)   // assert on state
}
```

**Google's guidance:** prefer **fakes** for your own collaborators (repositories, data sources) because they survive refactors and exercise real logic; reserve **mocks** for verifying that an interaction occurred (e.g. analytics event sent). Avoid over-mocking, which couples tests to implementation details.

---

## 11. What is Espresso?

**Espresso** is Google's UI testing framework for **in-app** instrumented tests. It is fast and reliable because it **automatically synchronizes** with the UI thread and the message queue — it waits until the app is idle before performing the next action, eliminating most `sleep()`-based flakiness.

Three core building blocks:
- **`onView(matcher)`** — find a view using a **ViewMatcher** (`withId`, `withText`, `withContentDescription`).
- **`.perform(action)`** — a **ViewAction** (`click()`, `typeText()`, `scrollTo()`, `swipeLeft()`).
- **`.check(assertion)`** — a **ViewAssertion** (`matches(isDisplayed())`, `doesNotExist()`).

```kotlin
@RunWith(AndroidJUnit4::class)
class LoginScreenTest {

    @get:Rule
    val activityRule = ActivityScenarioRule(LoginActivity::class.java)

    @Test
    fun emptyPassword_showsError() {
        onView(withId(R.id.username)).perform(typeText("ada"), closeSoftKeyboard())
        onView(withId(R.id.loginButton)).perform(click())

        onView(withId(R.id.error))
            .check(matches(allOf(isDisplayed(), withText("Password required"))))
    }
}
```

- **Espresso-Intents** verifies/stubs outgoing intents; **Espresso-Web** drives WebViews; **IdlingResource** tells Espresso to wait on background async work it can't see automatically.
- For **Jetpack Compose**, the equivalent is the Compose testing API (`composeTestRule.onNodeWithText(...).performClick().assertIsDisplayed()`), which also auto-synchronizes.

**📚 Reference:** <https://developer.android.com/training/testing/espresso/basics>

---

## 12. What is UI Automator?

**UI Automator** is a UI testing framework for **black-box, cross-app** testing. Unlike Espresso (scoped to your own app), UI Automator can interact with **any** app and the **system UI** — open the notification shade, press Home/Back/Recents, toggle settings, grant runtime permissions, interact with another app, and come back.

Core APIs:
- **`UiDevice`** — represents the device; performs system actions (`pressHome()`, `pressBack()`, `openNotification()`, `wait(...)`).
- **`UiSelector` / `BySelector`** — locate elements by text, resource-id, class, description.
- **`UiObject` / `UiObject2`** — found elements you can click/setText/scroll.

```kotlin
@RunWith(AndroidJUnit4::class)
class CrossAppTest {

    private val device = UiDevice.getInstance(InstrumentationRegistry.getInstrumentation())

    @Test
    fun launchSettings_fromHome() {
        device.pressHome()
        device.openNotification()
        device.wait(Until.hasObject(By.text("Clear all")), 3_000)

        val button = device.findObject(By.res("com.example", "startButton"))
        button.click()
    }
}
```

**Espresso vs. UI Automator:** use **Espresso** for testing your own app's UI (faster, auto-sync, view-level matchers); use **UI Automator** when a flow leaves your app or touches system UI/permissions. They are often combined in the same test.

**📚 Reference:** <https://developer.android.com/training/testing/other-components/ui-automator>

---

## 13. What is Robolectric? Pros and cons

**Robolectric** is a framework that lets you run Android-dependent tests on the **local JVM** — no emulator or device required. It provides "**shadow**" objects that simulate Android framework classes (`Activity`, `Resources`, `SharedPreferences`, `Looper`, etc.), so framework calls return real-ish values instead of the default `"Method not mocked"` exception.

```kotlin
@RunWith(AndroidJUnit4::class)   // Robolectric runner via robolectric.properties / config
@Config(sdk = [34])
class MainActivityRobolectricTest {

    @Test
    fun titleIsSet() {
        val activity = Robolectric.buildActivity(MainActivity::class.java)
            .setup()
            .get()
        val title = activity.findViewById<TextView>(R.id.title)
        assertEquals("Welcome", title.text.toString())
    }
}
```

**Pros:**
- **Fast** — runs on the JVM, no device deploy/boot; good for CI.
- Lets you test code that touches the framework (resources, parsing, simple Activities/Fragments) **without** an instrumented test.
- Integrates with AndroidX Test / Espresso APIs (`ActivityScenario`, `onView`) for off-device UI-ish tests.

**Cons / disadvantages (common interview answer):**
- **Simulation, not the real thing** — shadows can diverge from real device behavior, so a passing Robolectric test does not guarantee correctness on hardware (false confidence).
- **Incomplete coverage** — not every framework class/feature is shadowed; some APIs are unimplemented or behave differently.
- **Slower than pure JVM unit tests** — it loads a large Android runtime simulation and per-SDK jars.
- **Maintenance lag** — must keep up with new Android SDK levels; new platform behavior may not be modeled yet.
- **Hides integration bugs** — things like real rendering, animations, IPC, hardware, and timing aren't truly exercised, so it can't replace instrumented UI tests.

**Where it fits:** between pure unit tests and instrumented tests. Use it for framework-touching logic you want fast feedback on; still keep a smaller set of real instrumented/UI tests for end-to-end confidence.

**📚 Reference:** <http://robolectric.org/> · <https://stackoverflow.com/questions/18271474/robolectric-vs-android-test-framework>

---

## 14. Testing coroutines — runTest and TestDispatchers

Use the **`kotlinx-coroutines-test`** artifact. It controls **virtual time** so delays are skipped and execution is deterministic.

```kotlin
// testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.x")
```

**`runTest { }`** is the entry point. It runs the test body in a `TestScope` backed by a `TestDispatcher` and a shared `TestCoroutineScheduler`. It auto-advances virtual time (skipping `delay`s) and fails the test if child coroutines are still running or leaked at the end.

```kotlin
@Test
fun fetch_returnsData() = runTest {
    val repo = FakeRepository()           // suspend fun with delay(1000)
    val result = repo.fetch()             // delay is skipped via virtual time
    assertEquals("data", result)
}
```

**Two `TestDispatcher` implementations:**

| | **StandardTestDispatcher** (runTest default) | **UnconfinedTestDispatcher** |
|---|---|---|
| New coroutines | **Queued** on the scheduler; do not run until you yield/advance | Start **eagerly** on the current thread immediately |
| Control | Precise step-by-step control of execution order | Less control, simpler code |
| Best for | Complex ordering, testing intermediate states | Simple tests, collecting state without manual advancing |

**Scheduler controls:**
- `advanceUntilIdle()` — run all pending coroutines until none remain.
- `advanceTimeBy(ms)` — advance virtual time by a duration, running coroutines scheduled within it.
- `runCurrent()` — run coroutines scheduled at the current virtual time.

```kotlin
@Test
fun standardDispatcher_needsAdvancing() = runTest {
    var done = false
    launch { delay(5_000); done = true }   // queued, not yet run
    assertFalse(done)
    advanceUntilIdle()                      // now it runs
    assertTrue(done)
}
```

**Replacing `Dispatchers.Main`** (needed for `viewModelScope`, which uses `Dispatchers.Main`). A reusable JUnit rule:

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class MainDispatcherRule(
    private val dispatcher: TestDispatcher = UnconfinedTestDispatcher(),
) : TestWatcher() {
    override fun starting(description: Description) = Dispatchers.setMain(dispatcher)
    override fun finished(description: Description) = Dispatchers.resetMain()
}
```

```kotlin
class MyViewModelTest {
    @get:Rule val mainDispatcherRule = MainDispatcherRule()
    // ... viewModelScope.launch now runs on the test dispatcher
}
```

**Best practice:** inject dispatchers into your classes (don't hard-code `Dispatchers.IO`) so tests can pass `StandardTestDispatcher(testScheduler)` and share one scheduler/virtual clock.

**📚 Reference:** <https://developer.android.com/kotlin/coroutines/test> · <https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-test/>

---

## 15. Turbine — testing Kotlin Flow

**Turbine** (Cash App) is a small library that turns push-based `Flow` emissions into **pull-based suspend calls**, making Flow/StateFlow tests readable and non-flaky. You call `.test { }` on a flow and then `awaitItem()` / `awaitComplete()` / `awaitError()` to consume emissions one at a time.

```kotlin
// testImplementation("app.cash.turbine:turbine:1.x")

@Test
fun emitsAllValues() = runTest {
    flowOf("one", "two").test {
        assertEquals("one", awaitItem())
        assertEquals("two", awaitItem())
        awaitComplete()                 // assert the flow finished
    }
}
```

Core API inside the `test { }` block:
- **`awaitItem()`** — suspend until the next emission; return it (fails if it's a completion/error instead).
- **`awaitComplete()`** — assert the flow completed.
- **`awaitError()`** — assert (and return) a terminal error.
- **`expectNoEvents()`** — assert nothing has been emitted yet.
- **`skipItems(n)`** — drop the next *n* emissions (handy for skipping a StateFlow's initial value).
- **`cancelAndIgnoreRemainingEvents()`** — stop collecting a never-completing flow (e.g. a hot `StateFlow`).

```kotlin
@Test
fun stateFlow_emitsLoadingThenSuccess() = runTest {
    val viewModel = MyViewModel(FakeRepo())

    viewModel.uiState.test {
        assertEquals(UiState.Loading, awaitItem())   // initial
        viewModel.load()
        assertEquals(UiState.Success("data"), awaitItem())
        cancelAndIgnoreRemainingEvents()             // hot flow never completes
    }
}
```

Turbine fails the test if an unconsumed event remains, an unexpected event type arrives, or a timeout (default ~1s) is reached — catching subtle bugs that manual `collect`/list-capturing tests miss. There is also `turbineScope { }` for testing multiple flows together.

**📚 Reference:** <https://github.com/cashapp/turbine> · <https://developer.android.com/kotlin/flow/test>

---

## 16. Testing a ViewModel with Coroutines and LiveData

Two essentials:
1. **`InstantTaskExecutorRule`** (from `androidx.arch.core:core-testing`) — runs LiveData's background `ArchTaskExecutor` work **synchronously on the same thread**, so `setValue`/`postValue` are observed immediately in tests.
2. **`MainDispatcherRule`** — replaces `Dispatchers.Main` used by `viewModelScope`.

```kotlin
// testImplementation("androidx.arch.core:core-testing:2.2.0")

class UserViewModel(
    private val repo: UserRepository,
) : ViewModel() {
    private val _user = MutableLiveData<UiState>()
    val user: LiveData<UiState> = _user

    fun load(id: Int) {
        _user.value = UiState.Loading
        viewModelScope.launch {
            _user.value = try {
                UiState.Success(repo.getUser(id))
            } catch (e: Exception) {
                UiState.Error(e.message ?: "error")
            }
        }
    }
}
```

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class UserViewModelTest {

    @get:Rule val instantTaskRule = InstantTaskExecutorRule()   // (1) LiveData synchronous
    @get:Rule val mainDispatcherRule = MainDispatcherRule()     // (2) Dispatchers.Main

    private val repo: UserRepository = mockk()
    private lateinit var viewModel: UserViewModel

    @Before fun setUp() { viewModel = UserViewModel(repo) }

    @Test
    fun load_success_emitsSuccess() = runTest {
        coEvery { repo.getUser(1) } returns User(1, "Ada")

        viewModel.load(1)
        advanceUntilIdle()

        assertEquals(UiState.Success(User(1, "Ada")), viewModel.user.value)
    }

    @Test
    fun load_failure_emitsError() = runTest {
        coEvery { repo.getUser(any()) } throws IOException("network")

        viewModel.load(1)
        advanceUntilIdle()

        assertEquals(UiState.Error("network"), viewModel.user.value)
    }
}
```

To assert **a sequence** of LiveData emissions (Loading → Success), capture them with an observer:

```kotlin
@Test
fun load_emitsLoadingThenSuccess() = runTest {
    coEvery { repo.getUser(1) } returns User(1, "Ada")
    val emissions = mutableListOf<UiState>()
    val observer = Observer<UiState> { emissions.add(it) }
    viewModel.user.observeForever(observer)

    viewModel.load(1)
    advanceUntilIdle()

    assertEquals(UiState.Loading, emissions.first())
    assertTrue(emissions.last() is UiState.Success)
    viewModel.user.removeObserver(observer)
}
```

**📚 Reference:** <https://outcomeschool.com/blog/unit-testing-viewmodel-with-kotlin-coroutines-and-livedata>

---

## 17. Testing a ViewModel with Flow and StateFlow

`StateFlow` is a **hot** flow that always has a current value and never completes, so collecting it requires care. The cleanest approach is **Turbine**; you can also read `.value` directly for the latest state.

```kotlin
class SearchViewModel(
    private val repo: SearchRepository,
) : ViewModel() {

    private val _uiState = MutableStateFlow<UiState>(UiState.Idle)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    fun search(query: String) {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            _uiState.value = try {
                UiState.Success(repo.search(query))
            } catch (e: Exception) {
                UiState.Error(e.message ?: "error")
            }
        }
    }
}
```

**Approach A — Turbine (recommended for asserting the sequence):**

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class SearchViewModelTest {

    @get:Rule val mainDispatcherRule = MainDispatcherRule()

    private val repo: SearchRepository = mockk()
    private val viewModel by lazy { SearchViewModel(repo) }

    @Test
    fun search_emitsLoadingThenSuccess() = runTest {
        coEvery { repo.search("kotlin") } returns listOf("k1", "k2")

        viewModel.uiState.test {
            assertEquals(UiState.Idle, awaitItem())          // initial value
            viewModel.search("kotlin")
            assertEquals(UiState.Loading, awaitItem())
            assertEquals(UiState.Success(listOf("k1", "k2")), awaitItem())
            cancelAndIgnoreRemainingEvents()                 // hot flow
        }
    }
}
```

**Approach B — read `.value` after advancing virtual time (latest state only):**

```kotlin
@Test
fun search_finalStateIsSuccess() = runTest {
    coEvery { repo.search("kotlin") } returns listOf("k1")
    viewModel.search("kotlin")
    advanceUntilIdle()
    assertEquals(UiState.Success(listOf("k1")), viewModel.uiState.value)
}
```

**Note on `stateIn`:** a `StateFlow` produced by `stateIn(..., SharingStarted.WhileSubscribed())` only runs its upstream while there is a collector. In tests, either use `SharingStarted.Eagerly`, or start collecting (Turbine's `test {}` / a `backgroundScope.launch { collect {} }`) so the upstream actually runs.

**📚 Reference:** <https://outcomeschool.com/blog/unit-testing-viewmodel-with-kotlin-flow-and-stateflow>

---

## 18. Code coverage and JaCoCo

**Code coverage** measures how much of your production code is exercised by tests, reported as percentages of:
- **Line coverage** — lines executed.
- **Branch coverage** — `if`/`when`/conditional branches taken.
- **Method / class coverage** — methods and classes touched.

It is a **guidance metric, not a goal**: high coverage means a lot of code *ran* during tests, not that behavior was *meaningfully asserted*. 100% coverage with weak assertions is still poor testing; conversely, you don't need to cover trivial getters/generated code. Use it to find **untested critical paths**, set a sensible team threshold (often 70–85% on business logic), and gate it in CI.

**JaCoCo** (Java Code Coverage) is the standard tool on Android/JVM. It instruments bytecode and produces HTML/XML reports.

```kotlin
// build.gradle.kts
plugins {
    jacoco
}

tasks.register<JacocoReport>("jacocoTestReport") {
    dependsOn("testDebugUnitTest")
    reports {
        html.required.set(true)
        xml.required.set(true)   // for CI tools (SonarQube, Codecov)
    }
    val fileFilter = listOf(
        "**/R.class", "**/R$*.class", "**/BuildConfig.*",
        "**/Manifest*.*", "**/*Test*.*", "android/**/*.*",
        "**/*_Hilt*.*", "**/hilt_aggregated_deps/**", "**/*Module.*"
    )
    val debugTree = fileTree("${buildDir}/tmp/kotlin-classes/debug") { exclude(fileFilter) }
    classDirectories.setFrom(debugTree)
    sourceDirectories.setFrom(files("src/main/java", "src/main/kotlin"))
    executionData.setFrom(fileTree(buildDir) { include("**/*.exec", "**/*.ec") })
}
```

Run with `./gradlew jacocoTestReport`; the HTML report lands in `build/reports/jacoco/`. Common CI integration enforces a minimum via `JacocoCoverageVerification` (`violationRules { rule { limit { minimum = "0.80".toBigDecimal() } } }`) and uploads XML to Codecov/SonarQube. Exclude generated/DI/framework code (as above) so the number reflects real logic.

**Interview soundbite:** coverage tells you what *wasn't* tested, not whether tests are *good*. Pair it with meaningful assertions and mutation testing (e.g. Pitest) if you want to validate test quality.

**📚 Reference:** <https://www.eclemma.org/jacoco/> · <https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineer-tech-activity-7298564060105609217-8zXA>

---

## 19. Testing Jetpack Compose UI

Compose UI tests don't drive a `View` tree — they query the **semantics tree** (the same accessibility metadata screen readers use). You assert on nodes by their semantics and perform actions on them. Tests run as instrumented tests (on device/emulator) or with **Robolectric** for JVM-only Compose tests.

**Rules:** use `createComposeRule()` to set content directly, or `createAndroidComposeRule<MyActivity>()` when you need a real `Activity` (e.g. testing navigation in an existing screen).

```kotlin
class CounterTest {
    @get:Rule val composeTestRule = createComposeRule()

    @Test fun increments_onClick() {
        composeTestRule.setContent {
            MaterialTheme { Counter() }
        }

        // Finders → Assertions → Actions
        composeTestRule.onNodeWithText("Count: 0").assertIsDisplayed()
        composeTestRule.onNodeWithContentDescription("Increment").performClick()
        composeTestRule.onNodeWithText("Count: 1").assertExists()
    }
}
```

Key APIs:
- **Finders**: `onNodeWithText`, `onNodeWithTag` (pair with `Modifier.testTag("…")`), `onNodeWithContentDescription`, `onAllNodes…`.
- **Assertions**: `assertIsDisplayed()`, `assertExists()`, `assertIsEnabled()`, `assertTextEquals(…)`, `assertIsSelected()`.
- **Actions**: `performClick()`, `performTextInput(…)`, `performScrollTo()`, `performTouchInput { swipeUp() }`.

**Synchronization:** the test rule auto-syncs with Compose's recomposition and `LaunchedEffect`/`withFrameNanos` work, so you usually don't sleep. For animations or custom idling, disable auto-advance with `composeTestRule.mainClock.autoAdvance = false` and step time via `mainClock.advanceTimeBy(…)`, or register an `IdlingResource`.

**Compose + Espresso interop:** in an Activity that mixes Views and Compose, `createAndroidComposeRule` lets Espresso (Views) and the Compose rule coexist; `onView(...)` drives Views while `onNode(...)` drives composables.

**Debugging:** `composeTestRule.onRoot().printToLog("TAG")` dumps the semantics tree so you can see why a finder didn't match.

**Why it matters:** Compose is the default UI toolkit for new apps, so "how do you test a composable?" is now a standard question. The expected answer hits the **semantics tree**, the `createComposeRule` vs `createAndroidComposeRule` choice, the finder→assertion→action flow, `testTag`, and automatic recomposition synchronization.

**📚 Reference:** <https://developer.android.com/develop/ui/compose/testing>

---

*Sources verified June 2026: [Android Developers — Testing coroutines](https://developer.android.com/kotlin/coroutines/test), [kotlinx-coroutines-test API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-test/), [Turbine](https://github.com/cashapp/turbine), [MockK](https://mockk.io/), [Mockito](https://site.mockito.org/), [Robolectric](http://robolectric.org/).*
