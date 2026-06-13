# Android Libraries Interview Guide (Networking, DI, RxJava, Image Loading)

A comprehensive, code-first reference for the most commonly used Android libraries in technical interviews: **Retrofit/OkHttp** (networking), **Dagger/Hilt** (dependency injection), **RxJava** (reactive programming), **Glide/Fresco/Coil** (image loading), plus supporting libraries like **Kotlin Flow** and **App Startup**. Each entry gives a complete answer with Kotlin snippets and trade-offs. Accurate as of 2026.

---

## Table of Contents

### Retrofit / OkHttp
1. [What is an OkHttp Interceptor?](#1-what-is-an-okhttp-interceptor)
2. [HTTP caching with OkHttp](#2-http-caching-with-okhttp)
3. [How to enable logging in OkHttp](#3-how-to-enable-logging-in-okhttp)
4. [What is a Multipart request?](#4-what-is-a-multipart-request)
5. [Retrofit essentials and how it relates to OkHttp](#5-retrofit-essentials-and-how-it-relates-to-okhttp)

### Dependency Injection (Dagger / Hilt)
6. [Why use a DI framework like Dagger?](#6-why-use-a-di-framework-like-dagger)
7. [@Inject, @Module, @Provides, @Component explained](#7-inject-module-provides-component-explained)
8. [How does Dagger work?](#8-how-does-dagger-work)
9. [What is a Component in Dagger?](#9-what-is-a-component-in-dagger)
10. [What is a Module in Dagger?](#10-what-is-a-module-in-dagger)
11. [How does a custom scope work in Dagger?](#11-how-does-a-custom-scope-work-in-dagger)
12. [Dagger 2 vs Dagger-Hilt: how to choose](#12-dagger-2-vs-dagger-hilt-how-to-choose)

### RxJava
13. [Tell me about RxJava](#13-tell-me-about-rxjava)
14. [Types of Observables in RxJava](#14-types-of-observables-in-rxjava)
15. [Error handling in RxJava](#15-error-handling-in-rxjava)
16. [Map vs FlatMap (and SwitchMap, ConcatMap)](#16-map-vs-flatmap-and-switchmap-concatmap)
17. [create vs fromCallable](#17-create-vs-fromcallable)
18. [The defer operator](#18-the-defer-operator)
19. [Timer, Delay and Interval](#19-timer-delay-and-interval)
20. [Parallel network calls with Zip](#20-parallel-network-calls-with-zip)
21. [Concat vs Merge](#21-concat-vs-merge)
22. [Subjects in RxJava](#22-subjects-in-rxjava)
23. [dispose() vs clear() on CompositeDisposable](#23-dispose-vs-clear-on-compositedisposable)
24. [Schedulers.io() vs Schedulers.computation()](#24-schedulersio-vs-schedulerscomputation)
25. [Instant search with RxJava](#25-instant-search-with-rxjava)
26. [Pagination in RecyclerView with RxJava](#26-pagination-in-recyclerview-with-rxjava)

### Image Loading (Glide / Fresco / Coil)
27. [How do Glide and Fresco work internally?](#27-how-do-glide-and-fresco-work-internally)
28. [Glide vs Fresco vs Coil](#28-glide-vs-fresco-vs-coil)

### Other Libraries
29. [What is Flow in Kotlin?](#29-what-is-flow-in-kotlin)
30. [What is the App Startup library?](#30-what-is-the-app-startup-library)
31. [Common Android libraries and ORMs](#31-common-android-libraries-and-orms)

---

## Retrofit / OkHttp

## 1. What is an OkHttp Interceptor?

An **Interceptor** is a powerful mechanism in OkHttp to observe, mutate, retry, and short-circuit requests and responses. It sits in the request/response pipeline and forms a chain — each interceptor can inspect the outgoing `Request`, call `chain.proceed(request)` to continue down the chain, and then inspect/transform the returned `Response`.

There are two kinds:

- **Application interceptors** — registered via `addInterceptor()`. They run once per call (do not see redirects/retries), always see the original request (no auto-added headers like `Accept-Encoding`), and can short-circuit without a network call. Use these for cross-cutting app logic (auth headers, analytics).
- **Network interceptors** — registered via `addNetworkInterceptor()`. They sit closer to the wire: they see intermediate redirect/retry requests, the network-level request (with OkHttp's added headers), and have access to the `Connection`. Use these for low-level concerns (e.g. inspecting served-from-cache vs network).

```kotlin
class AuthInterceptor(private val tokenProvider: TokenProvider) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val original = chain.request()
        val request = original.newBuilder()
            .header("Authorization", "Bearer ${tokenProvider.token()}")
            .header("Accept", "application/json")
            .build()
        return chain.proceed(request)
    }
}

val client = OkHttpClient.Builder()
    .addInterceptor(AuthInterceptor(tokenProvider))   // application interceptor
    .addNetworkInterceptor(StethoInterceptor())       // network interceptor
    .build()
```

**Common uses:** injecting auth tokens, adding common headers, logging, retrying on failure, modifying response cache headers, and mocking responses in tests.

**Trade-off:** an application interceptor is simpler and runs once; a network interceptor is needed when you must observe what actually hits the network (redirects, transparent gzip, cache-vs-network).

**📚 Reference:** https://outcomeschool.com/blog/okhttp-interceptor

---

## 2. HTTP caching with OkHttp

OkHttp has a built-in HTTP response cache that follows the HTTP caching spec (`Cache-Control`, `ETag`, `Last-Modified`). You enable it by attaching a `Cache` to the client:

```kotlin
val cacheSize = 10L * 1024 * 1024 // 10 MB
val cache = Cache(File(context.cacheDir, "http_cache"), cacheSize)

val client = OkHttpClient.Builder()
    .cache(cache)
    .build()
```

If the server sends proper caching headers, OkHttp serves responses from disk automatically. When the server does **not** send good headers, you control caching with interceptors:

```kotlin
// Force a cache lifetime on responses (network interceptor — rewrites server headers)
val cacheInterceptor = Interceptor { chain ->
    val response = chain.proceed(chain.request())
    val maxAge = 60 // seconds
    response.newBuilder()
        .header("Cache-Control", "public, max-age=$maxAge")
        .removeHeader("Pragma")
        .build()
}

// Serve from cache when offline (application interceptor)
val offlineInterceptor = Interceptor { chain ->
    var request = chain.request()
    if (!isNetworkAvailable()) {
        request = request.newBuilder()
            .header("Cache-Control", "public, only-if-cached, max-stale=${60 * 60 * 24 * 7}")
            .build()
    }
    chain.proceed(request)
}

val client = OkHttpClient.Builder()
    .cache(cache)
    .addNetworkInterceptor(cacheInterceptor)
    .addInterceptor(offlineInterceptor)
    .build()
```

Per-request control with Retrofit uses `@Headers`/`CacheControl`. Key directives: `max-age` (fresh window), `max-stale` (accept stale up to N seconds), `only-if-cached` (cache only, fail if absent), `no-cache` (revalidate before use), `no-store` (never cache).

**Trade-off:** HTTP caching is great for GET endpoints with cacheable semantics; it is not a substitute for a local database (Room) when you need offline-first querying, relations, or cache invalidation logic.

**📚 Reference:** https://outcomeschool.com/blog/caching-with-okhttp-interceptor-and-retrofit

---

## 3. How to enable logging in OkHttp

Use the official `logging-interceptor` artifact. It logs request/response lines, headers, and bodies depending on the level.

```kotlin
// build.gradle: implementation("com.squareup.okhttp3:logging-interceptor:<version>")

val logging = HttpLoggingInterceptor().apply {
    level = if (BuildConfig.DEBUG)
        HttpLoggingInterceptor.Level.BODY
    else
        HttpLoggingInterceptor.Level.NONE
}

val client = OkHttpClient.Builder()
    .addInterceptor(logging)
    .build()
```

Levels: `NONE`, `BASIC` (request/response line), `HEADERS` (line + headers), `BODY` (line + headers + body).

**Important practices:**
- Add the logging interceptor **last** so it logs everything earlier interceptors added.
- Never use `BODY` (or `HEADERS`) in release builds — it leaks tokens, PII, and request bodies to logcat. Gate it on `BuildConfig.DEBUG`.
- Redact sensitive headers with `logging.redactHeader("Authorization")`.

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_softwareengineer-androiddev-android-activity-7274652524433850368-qC4D

---

## 4. What is a Multipart request?

A **multipart request** (`multipart/form-data`) lets you send several parts of different types in one HTTP body — typically file(s) plus form fields. It is the standard way to upload images/files alongside metadata.

With Retrofit:

```kotlin
interface UploadApi {
    @Multipart
    @POST("upload")
    suspend fun uploadProfile(
        @Part("user_id") userId: RequestBody,
        @Part avatar: MultipartBody.Part
    ): Response<UploadResponse>
}

fun buildAvatarPart(file: File): MultipartBody.Part {
    val requestFile = file.asRequestBody("image/*".toMediaTypeOrNull())
    // "avatar" must match the server's expected field name
    return MultipartBody.Part.createFormData("avatar", file.name, requestFile)
}

fun buildField(value: String): RequestBody =
    value.toRequestBody("text/plain".toMediaTypeOrNull())
```

Building it directly with OkHttp:

```kotlin
val body = MultipartBody.Builder()
    .setType(MultipartBody.FORM)
    .addFormDataPart("user_id", "42")
    .addFormDataPart("avatar", file.name, file.asRequestBody("image/*".toMediaTypeOrNull()))
    .build()

val request = Request.Builder().url(url).post(body).build()
```

Each part has its own headers and `Content-Type`, separated by a boundary string OkHttp generates. Use `@PartMap` for a dynamic set of fields, and stream large files (don't read the whole file into memory) for big uploads.

---

## 5. Retrofit essentials and how it relates to OkHttp

**Retrofit** is a type-safe HTTP client that turns a Kotlin/Java interface into a working API implementation. It handles URL building, serialization (via converters like Gson/Moshi/kotlinx-serialization), and adapting the call into `suspend` functions, `Call<T>`, RxJava types, or `Flow`. **OkHttp** is the underlying engine that actually performs the request (connection pooling, interceptors, caching, TLS). Retrofit delegates all network I/O to an `OkHttpClient`.

```kotlin
interface GithubApi {
    @GET("users/{user}/repos")
    suspend fun listRepos(@Path("user") user: String): List<Repo>
}

val retrofit = Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .client(okHttpClient)                              // your configured OkHttp
    .addConverterFactory(MoshiConverterFactory.create())
    .build()

val api: GithubApi = retrofit.create(GithubApi::class.java)
```

Key annotations: `@GET/@POST/@PUT/@DELETE`, `@Path`, `@Query`, `@Body`, `@Header`/`@Headers`, `@Field`/`@FormUrlEncoded`, `@Part`/`@Multipart`. Configure timeouts, interceptors, and caching on the `OkHttpClient`, then pass it to Retrofit.

---

## Dependency Injection (Dagger / Hilt)

## 6. Why use a DI framework like Dagger?

**Dependency Injection** means a class receives its dependencies from the outside rather than constructing them itself. You can do DI by hand (constructor injection everywhere), but at app scale that means writing and maintaining a large amount of "wiring" code — factories, object graphs, lifecycle/scope management. A framework like Dagger automates this.

Benefits:
- **Compile-time correctness** — Dagger generates code and validates the entire dependency graph at compile time. A missing or cyclic dependency is a build error, not a runtime crash. (Unlike reflection-based DI such as Guice or service-locator patterns.)
- **No reflection at runtime** — generated code is fast; important on mobile.
- **Decoupling & testability** — classes depend on interfaces; you swap real implementations for fakes/mocks in tests by replacing modules/components.
- **Scope and lifetime management** — singletons, per-activity, per-fragment instances are handled consistently.
- **Less boilerplate at scale** — you declare *how* to build things once; Dagger generates the factories and the wiring.

**Trade-off:** Dagger has a steep learning curve and adds build-time codegen. For small apps, manual DI or a lighter runtime container (Koin) can be enough.

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_androiddev-activity-7038019147737276417-tH4s

---

## 7. @Inject, @Module, @Provides, @Component explained

These are the four core Dagger annotations:

**`@Inject`** — marks how Dagger can *create* or *populate* something.
- On a constructor: tells Dagger it can construct that class (and what it needs).
- On a field/method: requests Dagger to fill that dependency (field injection, used for framework types like Activities you don't construct yourself).

```kotlin
class Engine @Inject constructor() // Dagger knows how to build Engine

class Car @Inject constructor(private val engine: Engine) // and Car (needs Engine)
```

**`@Module`** — a class that tells Dagger how to provide things it can't construct itself, e.g. interfaces, third-party classes (Retrofit, OkHttp), or types needing builder logic.

**`@Provides`** — a method inside a module that returns a fully built instance. `@Binds` is a leaner alternative for binding an interface to its implementation.

```kotlin
@Module
class NetworkModule {
    @Provides
    @Singleton
    fun provideOkHttp(): OkHttpClient = OkHttpClient.Builder().build()

    @Provides
    @Singleton
    fun provideRetrofit(client: OkHttpClient): Retrofit =
        Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(client)
            .build()

    @Provides
    fun provideApi(retrofit: Retrofit): GithubApi = retrofit.create(GithubApi::class.java)
}
```

**`@Component`** — the bridge between modules (providers) and the classes requesting dependencies (injection sites). It defines the object graph.

```kotlin
@Singleton
@Component(modules = [NetworkModule::class])
interface AppComponent {
    fun inject(activity: MainActivity)   // field injection target
    fun githubApi(): GithubApi           // expose a dependency
}

// Usage
val component = DaggerAppComponent.create()
component.inject(this) // fills @Inject fields in MainActivity
```

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_androiddev-android-kotlin-activity-7318124163481628673-EAYM

---

## 8. How does Dagger work?

Dagger is an **annotation processor**. At compile time it:

1. **Scans** your annotations (`@Inject`, `@Module`, `@Provides`, `@Binds`, `@Component`, scopes, qualifiers).
2. **Builds a dependency graph** — for each requested type it finds a provider (an `@Inject` constructor, a `@Provides`/`@Binds` method) and resolves that provider's own dependencies, recursively.
3. **Validates** the graph: every dependency must be satisfiable, with no cycles and no scope violations. Errors fail the build.
4. **Generates code** — `Factory` classes for each provider and a `DaggerXxxComponent` implementation of your component interface. These factories construct objects and pass dependencies in the right order. `@Singleton`/scoped bindings get wrapped in `DoubleCheck` providers so the instance is created once and reused.

At **runtime** there is no reflection — your code simply calls the generated `DaggerAppComponent` and the generated factories run plain constructor calls. This is why Dagger is fast and its errors surface at build time. Modern Dagger/Hilt runs this codegen through **KSP** (faster than the older KAPT).

---

## 9. What is a Component in Dagger?

A **Component** is the interface that ties the graph together — it knows about the modules (sources of dependencies) and exposes injection methods / provision methods to consumers. Dagger generates a concrete `Dagger<Name>` implementation.

```kotlin
@Singleton
@Component(modules = [NetworkModule::class, AppModule::class])
interface AppComponent {
    fun inject(app: MyApplication)
    fun repository(): UserRepository

    @Component.Factory
    interface Factory {
        fun create(@BindsInstance context: Context): AppComponent
    }
}
```

Components can be related:
- **Subcomponents** (`@Subcomponent`) inherit the parent's graph and add a narrower scope (e.g. `ActivityComponent` under `AppComponent`).
- **Component dependencies** (`dependencies = [...]`) compose graphs more loosely by depending on another component's exposed provisions.

`@BindsInstance` lets you feed runtime values (like `Context` or a base URL) into the graph at component creation.

---

## 10. What is a Module in Dagger?

A **Module** is a class (`@Module`) that contains the recipes (`@Provides`/`@Binds` methods) for objects that Dagger cannot create via an `@Inject` constructor. Typical cases: interfaces, third-party classes you don't own (Retrofit, Room, OkHttp), and objects requiring custom builder logic or configuration.

- **`@Provides`** — a method body that builds and returns the instance. Use when construction needs logic.
- **`@Binds`** — an abstract method that maps an interface to an implementation; no body, generates less code. Must live in an abstract `@Module` (or use an `object`/companion for `@Provides` to make them static and faster).

```kotlin
@Module
abstract class RepositoryModule {
    @Binds
    abstract fun bindUserRepo(impl: UserRepositoryImpl): UserRepository

    companion object {
        @Provides
        @Singleton
        fun provideDb(context: Context): AppDatabase =
            Room.databaseBuilder(context, AppDatabase::class.java, "app.db").build()
    }
}
```

Modules are attached to components via `@Component(modules = [...])`. In Hilt, modules are installed into a Hilt component with `@InstallIn(SingletonComponent::class)` etc.

---

## 11. How does a custom scope work in Dagger?

A **scope** tells Dagger to keep a **single instance** of a binding for the lifetime of the component that carries that scope. `@Singleton` is just a built-in scope annotation; you can define your own.

A scope is a custom annotation marked with `@Scope`:

```kotlin
@Scope
@Retention(AnnotationRetention.RUNTIME)
annotation class ActivityScope
```

Use it on the binding and on the (sub)component that owns it:

```kotlin
@ActivityScope
@Subcomponent(modules = [ActivityModule::class])
interface ActivityComponent {
    fun inject(activity: MainActivity)
}

@ActivityScope
class SessionTracker @Inject constructor() // one instance per ActivityComponent
```

How it works under the hood: a scoped binding is created once and cached (in a `DoubleCheck` provider) **inside the component instance**. As long as you hold the same component, you get the same instance. Create a new component (e.g. a new Activity creates a new `ActivityComponent`), and you get a fresh instance — the previous one becomes eligible for GC when its component is released. Rules: a scoped binding can only live in a component with the same scope, and a subcomponent's scope must differ from its parent's. Hilt predefines scopes (`@Singleton`, `@ActivityRetainedScoped`, `@ActivityScoped`, `@ViewModelScoped`, etc.) tied to Android lifecycles so you rarely define your own.

---

## 12. Dagger 2 vs Dagger-Hilt: how to choose

**Hilt is built on top of Dagger 2.** It is Google's opinionated, Android-specific layer that removes most of Dagger's boilerplate by providing predefined components and scopes tied to the Android lifecycle.

| Aspect | Dagger 2 (plain) | Hilt |
|---|---|---|
| Components | You define and wire them manually | Predefined (`SingletonComponent`, `ActivityComponent`, `ViewModelComponent`, …) |
| Android entry points | Manual `inject()` calls | `@AndroidEntryPoint`, `@HiltAndroidApp` do it |
| Scopes | Define your own | Predefined, lifecycle-bound |
| ViewModel support | Manual factory plumbing | `@HiltViewModel` + `hiltViewModel()` |
| Module install | `@Component(modules=...)` | `@InstallIn(...)` |
| Boilerplate | High | Low |
| Flexibility | Maximum | Slightly constrained to Android patterns |

```kotlin
// Hilt
@HiltAndroidApp
class MyApp : Application()

@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    @Inject lateinit var repo: UserRepository
}

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides @Singleton
    fun provideRetrofit(): Retrofit = Retrofit.Builder().baseUrl("https://api.example.com/").build()
}

@HiltViewModel
class HomeViewModel @Inject constructor(private val repo: UserRepository) : ViewModel()
```

**How to choose:**
- **Pick Hilt** for virtually all new Android apps — less boilerplate, standard lifecycle scoping, first-class ViewModel/Compose/WorkManager integration, and it's Google's recommended approach. As of 2026, current Dagger/Hilt (2.5x series) runs on KSP for faster builds (KSP support landed in 2.48; KAPT is deprecated for new projects).
- **Pick plain Dagger 2** when you need DI in a non-Android module (pure Kotlin/Java library), need full control over the component hierarchy, or are working in a large multi-module setup with custom graph topology that Hilt's fixed components don't fit. You can also mix: Hilt for the Android layer, plain Dagger components elsewhere.

**📚 Reference:** https://developer.android.com/jetpack/androidx/releases/hilt , https://dagger.dev/dev-guide/ksp.html

---

## RxJava

## 13. Tell me about RxJava

**RxJava** is a library for composing asynchronous and event-based programs using **observable sequences**. You model data/events as streams that emit items over time, and you transform/combine/observe them with a large set of operators, while declaratively controlling **on which thread** work happens.

Core building blocks:
- **Observable / Flowable / Single / Maybe / Completable** — the producer types (see Q14).
- **Observer / Subscriber** — the consumer (`onNext`, `onError`, `onComplete`).
- **Operators** — `map`, `flatMap`, `filter`, `zip`, `concat`, `debounce`, etc.
- **Schedulers** — `subscribeOn`/`observeOn` to move work between threads.
- **Disposable** — lets you cancel a subscription (critical for avoiding leaks on Android).

```kotlin
val disposable = repository.fetchUsers()        // Single<List<User>>
    .subscribeOn(Schedulers.io())                // do work on IO threads
    .observeOn(AndroidSchedulers.mainThread())   // deliver result on main thread
    .subscribe(
        { users -> render(users) },
        { error -> showError(error) }
    )
```

The contract: an Observable emits zero or more `onNext`, then terminates with exactly one `onComplete` **or** one `onError`. **Flowable** adds **backpressure** handling for fast producers. RxAndroid adds `AndroidSchedulers.mainThread()`.

Strengths: powerful composition of async work, threading, error handling. Trade-offs: steep learning curve, easy to leak subscriptions, and for many use cases Kotlin **Coroutines + Flow** now cover the same ground with simpler syntax (see Q29).

---

## 14. Types of Observables in RxJava

| Type | Emits | Terminal | Use case |
|---|---|---|---|
| **Observable\<T>** | 0..N items | `onComplete`/`onError` | Streams of events; **no** backpressure |
| **Flowable\<T>** | 0..N items | `onComplete`/`onError` | Like Observable but **with backpressure** for fast/large sources |
| **Single\<T>** | exactly 1 item | `onSuccess`/`onError` | One-shot result (a network call) |
| **Maybe\<T>** | 0 or 1 item | `onSuccess`/`onComplete`/`onError` | Optional result (DB lookup that may be empty) |
| **Completable** | no items | `onComplete`/`onError` | Fire-and-forget (a write/delete) |

```kotlin
Single.just(42)                        // emits 42 then onSuccess
Maybe.empty<Int>()                     // emits nothing then onComplete
Completable.fromAction { saveToDb() }  // just completes
Flowable.range(1, 1_000_000)           // backpressure-aware
    .onBackpressureBuffer()
```

**Backpressure** matters when a producer emits faster than the consumer can handle. `Observable` has no backpressure (can OOM with floods of events), so use **`Flowable`** with a strategy (`BUFFER`, `DROP`, `LATEST`, `ERROR`) for high-volume sources like sensor data; use `Observable` for UI events and bounded streams.

**📚 Reference:** https://outcomeschool.com/blog/types-of-observables-in-rxjava

---

## 15. Error handling in RxJava

An error sends `onError` and **terminates** the stream. RxJava offers operators to recover or transform errors:

- **`onErrorReturn { fallbackValue }`** — emit a fallback item then complete.
- **`onErrorReturnItem(item)`** — same, with a constant.
- **`onErrorResumeNext { fallbackObservable }`** — switch to another stream on error.
- **`retry()` / `retry(n)`** — resubscribe on error (immediate).
- **`retryWhen { errors -> ... }`** — controlled retry with backoff/conditions.
- **`doOnError { }`** — side effect (logging) without handling.
- The **`subscribe(onNext, onError)`** consumer — final catch-all; always provide an `onError` or an unhandled error crashes via `UndeliverableException`.

```kotlin
api.getUser(id)
    .retryWhen { errors ->
        errors.zipWith(Flowable.range(1, 3)) { err, attempt -> attempt }
            .flatMap { attempt -> Flowable.timer(attempt * 1000L, TimeUnit.MILLISECONDS) } // backoff
    }
    .onErrorReturn { User.EMPTY }            // graceful fallback
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe({ render(it) }, { showError(it) })
```

Best practices: always supply an `onError` handler; set a global `RxJavaPlugins.setErrorHandler {}` to avoid crashes from undeliverable errors after disposal; prefer `onErrorResumeNext` for fallback streams and `retryWhen` for backoff.

---

## 16. Map vs FlatMap (and SwitchMap, ConcatMap)

**`map`** transforms each item **synchronously** 1-to-1 (`T -> R`). The result is a plain value.

```
source:  --1----2----3-->
map(*10):--10---20---30->
```
```kotlin
Observable.just(1, 2, 3).map { it * 10 } // 10, 20, 30
```

**`flatMap`** transforms each item into **another Observable** and **merges** their emissions. Use it when each item triggers an async operation (e.g. a network call per id). Order is **not guaranteed** — inner streams run concurrently and interleave.

```
source:   --A--------B------->
flatMap:    \          \
             a1--a2      b1--b2
merged:   ----a1--b1-a2----b2->   (interleaved, order not preserved)
```
```kotlin
userIds.flatMap { id -> api.getUser(id).toObservable() } // concurrent, unordered
```

Related variants:

- **`concatMap`** — like `flatMap` but processes inner streams **one at a time, in order** (no interleaving). Use when order matters.
- **`switchMap`** — when a new item arrives, it **cancels the previous** inner stream and switches to the new one. Perfect for search/typeahead where only the latest query matters.

```kotlin
queries.switchMap { q -> api.search(q).toObservable() } // only latest query's results survive
```

| Operator | Concurrency | Order | Cancels previous |
|---|---|---|---|
| map | sync, 1:1 | preserved | n/a |
| flatMap | concurrent | not preserved | no |
| concatMap | sequential | preserved | no |
| switchMap | one active | latest only | yes |

**📚 Reference:** https://outcomeschool.com/blog/rxjava-map-vs-flatmap

---

## 17. create vs fromCallable

Both create a source, but they fit different needs.

**`create`** — full manual control of the emitter. You call `onNext`/`onError`/`onComplete` yourself, possibly many times, and you must handle disposal/backpressure. Use it to **bridge a callback-based API** into Rx.

```kotlin
Observable.create<Location> { emitter ->
    val listener = LocationListener { loc -> emitter.onNext(loc) }
    locationManager.requestUpdates(listener)
    emitter.setCancellable { locationManager.removeUpdates(listener) }
}
```

**`fromCallable`** — wraps a single function that returns one value (or throws). It emits that value lazily **at subscription time**, then completes. Exceptions are routed to `onError` automatically. Use it for a **single deferred computation/blocking call**.

```kotlin
Single.fromCallable { database.loadUser(id) } // runs lazily per subscriber
    .subscribeOn(Schedulers.io())
```

**Rule of thumb:** use `fromCallable` for a one-shot value-producing function (and to get lazy + automatic error handling); use `create` only when you need fine-grained emitter control or to adapt push-based callbacks. Avoid `Observable.just(expensiveCall())` because `just` evaluates **eagerly** at assembly time, not per subscription.

**📚 Reference:** https://outcomeschool.com/blog/rxjava-create-and-fromcallable-operator

---

## 18. The defer operator

**`defer`** does not create its Observable until a subscriber subscribes — and it creates a **fresh** one for **each** subscriber. This makes the source **lazy** and ensures it captures current state at subscription time, not at assembly time.

```kotlin
var token = "old"

// WITHOUT defer: just(token) captures "old" at assembly time
val eager = Observable.just(token)

// WITH defer: re-reads token each time someone subscribes
val lazy = Observable.defer { Observable.just(token) }

token = "new"
eager.subscribe { println(it) } // prints "old"
lazy.subscribe { println(it) }  // prints "new"
```

Use `defer` when the source depends on mutable/external state that may change between building the chain and subscribing, or when you want each subscriber to get an independent, freshly evaluated stream (e.g. reading the current time, a current config, or a per-subscription DB snapshot).

**📚 Reference:** https://outcomeschool.com/blog/rxjava-defer-operator

---

## 19. Timer, Delay and Interval

These three are time-based but behave differently:

**`timer`** — emits a **single** item (0L) after a delay, then completes.
```
timer(2s): -----------0|   (after 2s)
```
```kotlin
Observable.timer(2, TimeUnit.SECONDS).subscribe { /* fires once */ }
```

**`interval`** — emits an **increasing sequence** (0, 1, 2, …) repeatedly at a fixed period; never completes on its own.
```
interval(1s): --0--1--2--3--... 
```
```kotlin
Observable.interval(1, TimeUnit.SECONDS).subscribe { tick -> /* 0,1,2,... */ }
```

**`delay`** — does not create emissions; it **shifts existing emissions** later in time by a fixed duration.
```
source:     --A--B--C-->
delay(1s):  ----A--B--C->
```
```kotlin
Observable.just("A", "B", "C").delay(1, TimeUnit.SECONDS)
```

Summary: `timer` = one delayed emission; `interval` = repeating ticks (remember to dispose, since it's infinite); `delay` = postpone an existing stream's items. All run on the `computation` scheduler by default.

**📚 Reference:** https://outcomeschool.com/blog/rxjava-interval-operator

---

## 20. Parallel network calls with Zip

**`zip`** combines the latest emissions of multiple sources **by index** and emits a combined result only when **all** sources have emitted. Run each source on `Schedulers.io()` so they execute **in parallel**, then zip the results.

```kotlin
val user: Single<User> = api.getUser(id).subscribeOn(Schedulers.io())
val posts: Single<List<Post>> = api.getPosts(id).subscribeOn(Schedulers.io())

Single.zip(user, posts) { u, p -> Profile(u, p) }  // waits for both
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe({ render(it) }, { showError(it) })
```

```
userApi:  ----U------>
postsApi: ------P---->
zip:      ------(U,P)->   (combined when both ready; total time ≈ slowest call)
```

Key points: both calls start concurrently (each `subscribeOn(io())`), so total latency is roughly the slower of the two, not the sum. `zip` pairs by position; if you want "whatever each emitted most recently" use `combineLatest`. If **any** source errors, `zip` errors.

**📚 Reference:** https://outcomeschool.com/blog/rxjava-zip-operator

---

## 21. Concat vs Merge

Both combine multiple Observables into one, but differ in **ordering and concurrency**:

**`concat`** — subscribes to sources **sequentially**; it fully consumes the first before starting the second. Order is preserved, no interleaving. A source that never completes blocks the rest.
```
A: --1--2--|
B:           --3--4--|
concat: --1--2--3--4--|
```

**`merge`** — subscribes to **all sources at once**; emissions interleave as they arrive. Faster (concurrent) but no ordering guarantee.
```
A: --1----2---|
B: ---3---4---|
merge: --1-3--2-4--|   (interleaved)
```

```kotlin
Observable.concat(cacheSource, networkSource)  // cache first, then network, in order
Observable.merge(sensorA, sensorB)             // both streams together, interleaved
```

**Choose `concat`** when order matters or sources must run one after another (e.g. emit cached data, then fresh data). **Choose `merge`** when you want maximum concurrency and order doesn't matter (e.g. combining independent event streams).

**📚 Reference:** https://outcomeschool.com/blog/rxjava-concat-operator , https://outcomeschool.com/blog/rxjava-merge-operator

---

## 22. Subjects in RxJava

A **Subject** is both an **Observable and an Observer** — it can subscribe to sources and you can also call `onNext`/`onError`/`onComplete` on it manually. It's a hot, multicast bridge often used to push events into an Rx pipeline (e.g. UI clicks). The four common types differ in what they replay to new subscribers:

| Subject | Behavior on subscribe |
|---|---|
| **PublishSubject** | Emits only items published **after** subscription |
| **BehaviorSubject** | Emits the **most recent** item (or a default), then subsequent items |
| **ReplaySubject** | Emits **all** previously published items (optionally a bounded buffer) |
| **AsyncSubject** | Emits **only the last** item, and only when the source **completes** |

```kotlin
val publish = PublishSubject.create<Int>()
publish.onNext(1)                 // lost (no subscribers yet)
publish.subscribe { println(it) }
publish.onNext(2)                 // prints 2

val behavior = BehaviorSubject.createDefault(0)
behavior.onNext(1)
behavior.subscribe { println(it) } // prints 1 (latest), then future items

val replay = ReplaySubject.create<Int>()
replay.onNext(1); replay.onNext(2)
replay.subscribe { println(it) }   // prints 1, 2
```

Use cases: `BehaviorSubject` for current-state (like a simple state holder), `PublishSubject` for one-off events (clicks, navigation), `ReplaySubject` to cache an event log. Note: Subjects are not thread-safe for concurrent `onNext`; wrap with `.toSerialized()` if needed. In modern Kotlin code, `StateFlow`/`SharedFlow` are the idiomatic equivalents.

**📚 Reference:** https://outcomeschool.com/blog/rxjava-subject-publish-replay-behavior-async

---

## 23. dispose() vs clear() on CompositeDisposable

A **`CompositeDisposable`** is a container that holds multiple `Disposable`s so you can cancel them together (typically in `onDestroy`/`onCleared` to avoid leaks).

- **`clear()`** — disposes all currently held disposables **but keeps the container usable**. You can add new disposables afterward and they will work.
- **`dispose()`** — disposes all held disposables **and marks the container as disposed**. Any disposable added afterward is **immediately disposed** (effectively ignored); the container is dead.

```kotlin
class MyViewModel : ViewModel() {
    private val disposables = CompositeDisposable()

    fun load() {
        disposables.add(
            repo.fetch().subscribe({ /*...*/ }, { /*...*/ })
        )
    }

    override fun onCleared() {
        disposables.clear()   // cancel work; ViewModel could still add more if reused
        super.onCleared()
    }
}
```

**Rule of thumb:** use **`clear()`** when the owning object may live on and add new subscriptions later (e.g. a ViewModel/Activity that reloads). Use **`dispose()`** only when you are permanently done with the container and will never add to it again. Using `dispose()` then adding a new disposable is a common bug — the new one silently never runs.

**📚 Reference:** https://outcomeschool.com/blog/dispose-vs-clear-compositedisposable-rxjava

---

## 24. Schedulers.io() vs Schedulers.computation()

Both move work off the main thread, but they're tuned for different workloads:

**`Schedulers.io()`** — backed by an **unbounded, elastic** thread pool that grows as needed and reuses idle threads. Designed for **I/O-bound, often blocking** work that spends most time waiting: network calls, disk/database reads, file access.

**`Schedulers.computation()`** — backed by a **fixed pool sized to the number of CPU cores**. Designed for **CPU-bound** work: parsing, image/bitmap processing, math, sorting large lists. Capping at core count avoids context-switch thrashing.

```kotlin
api.download()                       // I/O
    .subscribeOn(Schedulers.io())
    .observeOn(Schedulers.computation())
    .map { decodeAndResize(it) }     // CPU work
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe { show(it) }
```

**Why it matters:** running blocking I/O on `computation()` can starve the small fixed pool and stall other CPU work; running heavy CPU work on `io()` can spawn excessive threads. Match the scheduler to the workload. Other schedulers: `single()` (one shared thread), `newThread()` (new thread per task), `trampoline()` (queue on current thread), and `AndroidSchedulers.mainThread()` (UI thread).

---

## 25. Instant search with RxJava

A responsive search-as-you-type pipeline combines four operators on a stream of query strings (often fed by a `PublishSubject` from an `EditText`):

- **`debounce`** — wait until the user stops typing (e.g. 300 ms) before reacting; avoids a request per keystroke.
- **`filter`** — ignore too-short queries.
- **`distinctUntilChanged`** — skip if the query didn't actually change.
- **`switchMap`** — cancel the in-flight request when a newer query arrives, so only the latest results render.

```kotlin
val querySubject = PublishSubject.create<String>()

querySubject
    .debounce(300, TimeUnit.MILLISECONDS)
    .map { it.trim() }
    .filter { it.length >= 2 }
    .distinctUntilChanged()
    .switchMap { q ->
        api.search(q)
            .subscribeOn(Schedulers.io())
            .toObservable()
            .onErrorReturn { emptyList() }   // keep the stream alive
    }
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe { results -> showResults(results) }

// editText.doAfterTextChanged { querySubject.onNext(it.toString()) }
```

`switchMap` (not `flatMap`) is the key choice: it guarantees stale responses for older queries are discarded.

**📚 Reference:** https://outcomeschool.com/blog/instant-search-using-rxjava-operators

---

## 26. Pagination in RecyclerView with RxJava

The idea: turn "user scrolled near the bottom" into a stream of page-load events, throttle them, and load the next page sequentially.

```kotlin
val loadMore = PublishSubject.create<Int>() // emits next page number

recyclerView.addOnScrollListener(object : RecyclerView.OnScrollListener() {
    override fun onScrolled(rv: RecyclerView, dx: Int, dy: Int) {
        val lm = rv.layoutManager as LinearLayoutManager
        val visible = lm.childCount
        val total = lm.itemCount
        val firstVisible = lm.findFirstVisibleItemPosition()
        if (!isLoading && (visible + firstVisible) >= total - THRESHOLD) {
            loadMore.onNext(currentPage + 1)
        }
    }
})

loadMore
    .distinctUntilChanged()
    .concatMap { page ->                       // concatMap keeps pages in order
        api.getItems(page)
            .subscribeOn(Schedulers.io())
            .toObservable()
            .doOnSubscribe { isLoading = true }
            .doFinally { isLoading = false }
    }
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe({ pageItems ->
        adapter.append(pageItems)
        currentPage++
    }, { showError(it) })
```

Key choices: a `PublishSubject` converts scroll callbacks into a stream; an `isLoading` guard prevents duplicate triggers; **`concatMap`** (not `flatMap`) ensures pages append **in order**. In modern apps the **Paging 3** library handles this end-to-end and is usually preferred over a hand-rolled Rx pipeline.

**📚 Reference:** https://outcomeschool.com/blog/pagination-in-recyclerview-using-rxjava-operators

---

## Image Loading (Glide / Fresco / Coil)

## 27. How do Glide and Fresco work internally?

Both solve the same hard problems — decoding images efficiently, avoiding `OutOfMemoryError`, smooth scrolling, and caching — but with different architectures.

**Glide:**
- **Multi-layer caching:** an **active resources** cache (images currently in use, held via weak references), a **memory cache** (`LruCache` of decoded `Bitmap`s), and a **disk cache** (original or transformed images). A request first checks active → memory → disk → network.
- **Bitmap pool:** Glide reuses `Bitmap` objects via a pool (`inBitmap`) instead of allocating new ones, cutting GC pressure and memory churn — a major source of jank when scrolling lists.
- **Downsampling:** it decodes images at the **target view's size** (using `inSampleSize`/`BitmapFactory.Options`), so a 4000×3000 photo loading into a 200×200 `ImageView` decodes small, not full-resolution.
- **Lifecycle awareness:** requests are bound to the Activity/Fragment lifecycle and paused/cancelled automatically (e.g. when the view scrolls off or the screen is destroyed), preventing wasted work and leaks.

**Fresco** (Facebook):
- Stores decoded bitmaps in **ashmem / native (off-heap) memory** (on older Android via the "purgeable" decode trick) rather than the Java heap, so large images don't count against the Java heap limit — this is its signature advantage for image-heavy apps.
- Uses a **three-level pipeline cache:** bitmap (decoded) cache, encoded memory cache, and disk cache.
- Centers on **Drawees** — `DraweeView` + a hierarchy that manages placeholders, progress bars, retry, failure images, and rounded/scaled rendering declaratively.
- Built-in **progressive JPEG**, animated GIF/WebP support, and a configurable `ImagePipeline`.

Shared techniques that make both fast and memory-safe: **decode to target size (downsampling)**, **reuse bitmaps (pool)**, **multi-tier caching (memory + disk)**, and **cancel/throttle requests** for off-screen views.

**📚 Reference:** https://outcomeschool.com/blog/android-image-loading-library-optimize-memory-usage , https://outcomeschool.com/blog/android-image-loading-library-use-bitmap-pool-for-responsive-ui , https://outcomeschool.com/blog/android-image-loading-library-solve-the-slow-loading-issue

---

## 28. Glide vs Fresco vs Coil

| | Glide | Fresco | Coil |
|---|---|---|---|
| Language/style | Java, builder API | Java, Drawee views | Kotlin-first, coroutines |
| Memory model | Java heap + bitmap pool | Native/ashmem (off-heap) | Java heap |
| Best at | General use, smooth lists, GIFs | Heavy image apps, memory-critical, progressive JPEG | Modern Kotlin/Compose apps, small footprint |
| Integration | Wide, mature | Requires `DraweeView` | Jetpack Compose `AsyncImage`; Coil 3 is Compose Multiplatform with pluggable Ktor/OkHttp networking |
| Dependency size | Medium | Large | Small |

```kotlin
// Glide
Glide.with(context).load(url).placeholder(R.drawable.ph).into(imageView)

// Coil (Kotlin / Compose)
imageView.load(url) { crossfade(true) }
// Compose:
AsyncImage(model = url, contentDescription = null)

// Fresco
val draweeView = SimpleDraweeView(context)
draweeView.setImageURI(url)
```

**How to choose (2026):** **Coil** (now at v3, a Kotlin Multiplatform library) is the idiomatic choice for new Kotlin/Compose apps — small, coroutine-based, first-class Compose support, and it works across Android/iOS/desktop via Compose Multiplatform. **Glide** remains a robust, battle-tested default with excellent caching and animated-image support. **Fresco** still wins when memory pressure from many/large images is the dominant concern (off-heap bitmaps), at the cost of a larger dependency and the `DraweeView` requirement; it is still maintained but chosen less often for greenfield projects. Picasso is largely legacy at this point.

**📚 Reference:** https://frescolib.org/ , https://github.com/facebook/fresco

---

## Other Libraries

## 29. What is Flow in Kotlin?

**Flow** is Kotlin Coroutines' API for asynchronous **streams** of values — the coroutine-native equivalent of an RxJava `Observable`. A flow emits multiple values over time, is **cold** by default (the producer runs fresh for each collector, only when collected), and integrates with structured concurrency and `suspend` functions.

```kotlin
fun tickerFlow(): Flow<Int> = flow {       // cold producer
    var i = 0
    while (true) { emit(i++); delay(1000) }
}

// Collecting (a terminal, suspending operation)
lifecycleScope.launch {
    repository.observeUsers()              // Flow<List<User>>
        .map { it.filter(User::isActive) } // operators like Rx
        .flowOn(Dispatchers.IO)            // upstream runs on IO
        .catch { e -> emit(emptyList()) }  // error handling
        .collect { render(it) }            // runs on the launching dispatcher
}
```

Key points:
- **Cold** flows (`flow { }`) vs **hot** flows: **`StateFlow`** (always has a current value; like `BehaviorSubject`/`LiveData`) and **`SharedFlow`** (multicast events; like `PublishSubject`).
- **`flowOn`** controls the dispatcher of upstream operators; `collect` runs on the collector's context — cleaner than Rx's `subscribeOn`/`observeOn`.
- Rich operators: `map`, `filter`, `flatMapLatest` (≈ `switchMap`), `debounce`, `combine`, `zip`, `buffer`/`conflate` (backpressure via suspension).
- Cancellation is automatic via the coroutine/lifecycle scope — no manual `Disposable`. Use `repeatOnLifecycle(STARTED)` / `flowWithLifecycle` to collect safely on Android.

**Flow vs RxJava:** Flow needs no extra dependency in Kotlin projects, has simpler threading and lifecycle/cancellation, and uses suspension for backpressure. RxJava has a larger operator set and broad maturity. For new Kotlin code, Flow + Coroutines is the recommended default.

**📚 Reference:** https://outcomeschool.com/blog/flow-api-in-kotlin

---

## 30. What is the App Startup library?

**App Startup** (`androidx.startup`) is a Jetpack library that provides a single, efficient way to **initialize components at app launch**. Without it, each library that needs init often adds its own `ContentProvider` (because content providers run before `Application.onCreate`), and many providers measurably slow cold start. App Startup replaces all of them with **one** shared `InitializationProvider`.

You define an `Initializer`:

```kotlin
class LoggerInitializer : Initializer<Logger> {
    override fun create(context: Context): Logger {
        val logger = Logger.init(context)
        return logger
    }
    // declare initializers that must run before this one
    override fun dependencies(): List<Class<out Initializer<*>>> =
        listOf(TimberInitializer::class.java)
}
```

Register it in the manifest under the single provider:

```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">
    <meta-data
        android:name="com.example.LoggerInitializer"
        android:value="androidx.startup" />
</provider>
```

Benefits:
- **One** `ContentProvider` instead of many → faster cold start.
- **Dependency ordering** via `dependencies()` — App Startup builds a topological order and initializes things in the right sequence.
- **Lazy/manual init** — you can opt a component out of automatic startup (remove its `<meta-data>`) and call `AppInitializer.getInstance(context).initializeComponent(MyInitializer::class.java)` on demand to defer cost off the critical path.

**📚 Reference:** https://outcomeschool.com/blog/app-startup-library

---

## 31. Common Android libraries and ORMs

Interviewers often open with a breadth question — "which libraries do you know, and do you build your own or use existing ones?" A practical answer: prefer well-maintained, widely-used libraries for solved problems (networking, DI, images), and only write custom utilities for app-specific logic — reinventing networking/caching is rarely worth the maintenance cost.

**Frequently used libraries by category:**
- **Networking:** Retrofit, OkHttp, Ktor Client
- **Serialization:** Moshi, kotlinx.serialization, Gson (legacy)
- **DI:** Hilt/Dagger, Koin
- **Reactive/async:** Kotlin Coroutines + Flow, RxJava/RxAndroid
- **Image loading:** Coil, Glide, Fresco, Picasso (legacy)
- **Jetpack:** ViewModel, Lifecycle, Navigation, Paging 3, WorkManager, Compose, DataStore, App Startup
- **Logging/debug:** Timber, LeakCanary, Chucker
- **Testing:** JUnit, Mockito/MockK, Espresso, Turbine (Flow), Robolectric

**ORM / persistence:**
- **Object-Relational Mapping (ORM)** is a design pattern for accessing a relational database through objects in an object-oriented language, so you work with typed entities instead of raw SQL/cursors.
- **Room** — Google's recommended ORM/persistence layer over SQLite: compile-time SQL verification, DAOs, and observable queries returning `Flow`/`LiveData`/RxJava types. The default choice today.
- **Realm** — an object database (not SQLite-based) with its own engine and live objects.
- **DBFlow, OrmLite** — older SQLite ORMs, largely legacy now.

```kotlin
@Entity
data class User(@PrimaryKey val id: Int, val name: String)

@Dao
interface UserDao {
    @Query("SELECT * FROM User")
    fun observeAll(): Flow<List<User>>      // observable query

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(user: User)
}
```
