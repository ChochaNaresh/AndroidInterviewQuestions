# Android Libraries — Short Answers (Quick Revision)

> Condensed answers for rapid review before an interview. For full explanations and code see `android_libraries_interview.md`.

---

## Retrofit / OkHttp

## 1. What is an OkHttp Interceptor?

**OkHttp Interceptor** means a mechanism in OkHttp that observes, mutates, retries, and short-circuits outgoing requests and incoming responses.

A mechanism to observe, mutate, retry, and short-circuit requests/responses in a chain (each calls `chain.proceed(request)`). **Application interceptors** (`addInterceptor`) run once per call and see the original request — good for auth headers, logging. **Network interceptors** (`addNetworkInterceptor`) sit closer to the wire and see redirects/retries and cache-vs-network.

---



## 2. HTTP caching with OkHttp

**HTTP caching** means OkHttp's built-in mechanism to store and serve network responses locally based on standard HTTP cache headers.

Attach a `Cache` to the client; OkHttp then honors HTTP cache headers (`Cache-Control`, `ETag`) automatically. When servers send poor headers, use interceptors to rewrite `Cache-Control` (e.g. force `max-age`, or serve stale with `only-if-cached` when offline). Not a substitute for Room when you need offline-first querying.

---



## 3. How to enable logging in OkHttp

**OkHttp Logging Interceptor** means a network tool that logs request and response lines, headers, and body payloads at configurable detail levels during development.

Add `HttpLoggingInterceptor` with a level (`NONE`/`BASIC`/`HEADERS`/`BODY`). Add it **last** so it captures headers added earlier; gate `BODY`/`HEADERS` on `BuildConfig.DEBUG` (it leaks tokens/PII) and use `redactHeader("Authorization")`.

---



## 4. What is a Multipart request?

**Multipart request** means an HTTP request format that allows transmitting multiple payloads of different data types (such as files and form fields) in a single body.

A `multipart/form-data` request sends several parts of different types (files + form fields) in one body, separated by a boundary — the standard way to upload files with metadata. With Retrofit use `@Multipart` + `@Part` (`MultipartBody.Part` for files, `RequestBody` for fields); `@PartMap` for dynamic fields. Stream large files.

---



## 5. Retrofit essentials and how it relates to OkHttp

**Retrofit** means a type-safe HTTP client libraries wrapper that turns a Kotlin or Java interface into a REST API client using dynamic proxies.

Retrofit is a type-safe HTTP client that turns an interface into an API implementation, handling URL building, serialisation (Gson/Moshi/kotlinx), and adapting to `suspend`/`Call`/Flow/RxJava. **OkHttp** is the underlying engine (connection pooling, interceptors, caching, TLS); Retrofit delegates all network I/O to an `OkHttpClient`.

---



## Dependency Injection (Dagger / Hilt)

## 6. Why use a DI framework like Dagger?

**Dependency Injection (DI) framework** means a tool like Dagger that automates the creation, scoping, and wiring of dependencies in an object graph at compile time.

It automates dependency wiring. Benefits: compile-time graph validation (missing/cyclic deps fail the build), no runtime reflection (fast), decoupling/testability, consistent scope/lifetime management, and less boilerplate at scale. Trade-off: steep learning curve and build-time codegen; small apps may use manual DI or Koin.

---



## 7. @Inject, @Module, @Provides, @Component explained

**Dagger core annotations** means `@Inject` to request or mark constructors for injection, `@Module` to declare dependency provider methods, `@Provides` to build dependency instances, and `@Component` to bridge providers and injection sites.

- **`@Inject`**: tells Dagger how to construct a class (on constructor) or requests a dependency (on field).
- **`@Module`**: a class providing things Dagger can't construct (interfaces, third-party types).
- **`@Provides`**: a module method returning a built instance (`@Binds` is the leaner interface-to-impl alternative).
- **`@Component`**: the interface tying modules to injection sites; defines the object graph (Dagger generates `DaggerXxxComponent`).

---



## 8. How does Dagger work?

**Dagger internals** means Dagger's compile-time annotation processor that validates the dependency graph and generates boilerplate-free Java factories to resolve dependencies at runtime.

It's a compile-time annotation processor that scans annotations, builds and validates the dependency graph (no cycles/scope violations), and generates `Factory` classes plus the `DaggerComponent` implementation. At runtime there's no reflection — just generated factory/constructor calls. Scoped bindings use `DoubleCheck`. Modern Dagger/Hilt runs on KSP.

---



## 9. What is a Component in Dagger?

**Dagger Component** means the interface that defines the dependency graph and exposes injection or provision methods to connect modules with dependency consumers.

The interface that ties the graph together — it knows the modules and exposes injection/provision methods to consumers (Dagger generates the concrete `Dagger<Name>`). Components relate via `@Subcomponent` (inherit parent graph, narrower scope) or component dependencies. `@BindsInstance` feeds runtime values (e.g. `Context`) into the graph.

---



## 10. What is a Module in Dagger?

**Dagger Module** means a class annotated with `@Module` that provides dependency recipes for classes Dagger cannot instantiate automatically.

A `@Module` class containing recipes for objects Dagger can't build via an `@Inject` constructor (interfaces, third-party types, configured objects). `@Provides` builds with logic; `@Binds` (abstract, no body) maps an interface to an implementation and generates less code. Attached via `@Component(modules=...)`, or `@InstallIn` in Hilt.

---



## 11. How does a custom scope work in Dagger?

**Dagger Scope** means a custom annotation that instructs Dagger to retain a single instance of a dependency for the lifetime of the component instance holding that scope.

A scope (a custom `@Scope` annotation) tells Dagger to keep a single cached instance of a binding for the lifetime of the component carrying that scope. The scope goes on both the binding and its (sub)component; the instance is cached in a `DoubleCheck` inside the component. A new component → fresh instance. Hilt predefines lifecycle-bound scopes (`@Singleton`, `@ActivityScoped`, `@ViewModelScoped`).

---



## 12. Dagger 2 vs Dagger-Hilt: how to choose

**Hilt** means Google's Android-specific dependency injection library built on top of Dagger 2 that provides predefined components, scopes, and lifecycle-integrated injection points.

Hilt is built on Dagger 2 and is Google's opinionated Android layer: predefined components/scopes, `@AndroidEntryPoint`/`@HiltAndroidApp`, `@HiltViewModel`, `@InstallIn` — far less boilerplate. **Pick Hilt** for virtually all new Android apps. **Pick plain Dagger 2** for non-Android modules or when you need full control over a custom component hierarchy.

---



## RxJava

## 13. Tell me about RxJava

**RxJava** means a library for composing asynchronous and event-driven programs using observable sequences and reactive stream operators.

A library for composing async/event-based programs with observable sequences. Building blocks: producers (Observable/Flowable/Single/Maybe/Completable), Observer (`onNext`/`onError`/`onComplete`), operators (`map`, `flatMap`, `zip`…), Schedulers (`subscribeOn`/`observeOn`), and Disposable (cancellation). Powerful but steep; for Kotlin, Coroutines + Flow often replace it.

---



## 14. Types of Observables in RxJava

**RxJava Observables** means the reactive stream types—including Observable, Flowable, Single, Maybe, and Completable—that emit asynchronous data packets to observers.

- **Observable\<T>**: 0..N items, no backpressure (UI events).
- **Flowable\<T>**: 0..N items with backpressure (fast/large sources).
- **Single\<T>**: exactly one item (a network call).
- **Maybe\<T>**: 0 or 1 item (optional DB lookup).
- **Completable**: no items, just completes (a write/delete).

---



## 15. Error handling in RxJava

**RxJava error handling** means managing stream errors using operators like `onErrorReturn` or `retry` to prevent stream termination.

An error sends `onError` and terminates the stream. Recovery operators: `onErrorReturn`/`onErrorReturnItem` (fallback value), `onErrorResumeNext` (fallback stream), `retry`/`retry(n)`, `retryWhen` (backoff), `doOnError` (logging). Always supply an `onError` handler and set a global `RxJavaPlugins.setErrorHandler`.

---



## 16. Map vs FlatMap (and SwitchMap, ConcatMap)

**RxJava map vs flatMap** means the distinction between transforming emissions synchronously 1-to-1 (map), and transforming emissions into new Observables to merge them asynchronously (flatMap).

- **map**: synchronous 1:1 transform.
- **flatMap**: each item → an Observable, merged concurrently, order not preserved.
- **concatMap**: like flatMap but sequential and ordered.
- **switchMap**: cancels the previous inner stream when a new item arrives (perfect for search/typeahead).

---



## 17. create vs fromCallable

**RxJava create vs defer** means the choice between creating an Observable manually with an emitter block (create), and creating an Observable lazily for each subscriber (defer).

- **`create`**: full manual emitter control (`onNext`/`onError`/`onComplete`, handle disposal) — used to bridge callback-based APIs.
- **`fromCallable`**: wraps a single function evaluated lazily at subscription, with automatic error routing — for a one-shot deferred/blocking call.
Avoid `just(expensiveCall())` (eager evaluation).

---



## 18. The defer operator

**defer operator** means an RxJava operator that waits to instantiate the target Observable until subscription time.

`defer` delays creating the Observable until subscription and creates a fresh one per subscriber, capturing current state at subscription (not assembly) time. Use it when the source depends on mutable/external state, or when each subscriber needs an independent, freshly-evaluated stream.

---



## 19. Timer, Delay and Interval

**RxJava time operators** means time-shifting elements including shifting all emissions (delay), emitting a single item after a delay (timer), or emitting periodic items (interval).

- **timer**: emits a single item once after a delay, then completes.
- **interval**: emits an increasing sequence at a fixed period, never completes (remember to dispose).
- **delay**: shifts an existing stream's emissions later in time.
All run on the `computation` scheduler by default.

---



## 20. Parallel network calls with Zip

**zip operator** means an RxJava operator that combines emissions from multiple sources by index to produce a single merged emission.

`zip` combines emissions by index and emits only when all sources have emitted. Give each source its own `subscribeOn(Schedulers.io())` so they run in parallel — total latency ≈ the slowest call. `zip` pairs by position; use `combineLatest` for "most recent each." If any source errors, zip errors.

---



## 21. Concat vs Merge

**RxJava merge vs concat** means the choice between interleaving emissions from multiple sources concurrently (merge), and processing multiple sources sequentially (concat).

- **concat**: subscribes sequentially, preserves order, no interleaving (a never-completing source blocks the rest). Use when order matters (cache then network).
- **merge**: subscribes to all at once, interleaves emissions, concurrent, no ordering. Use for max concurrency when order doesn't matter.

---



## 22. Subjects in RxJava

**Subject** means a reactive bridge class that acts as both an Observable and an Observer to multicast emissions.

A Subject is both an Observable and an Observer (hot, multicast). Types differ in what they replay:
- **PublishSubject**: only items after subscription.
- **BehaviourSubject**: the most recent item (or default), then subsequent.
- **ReplaySubject**: all previous items.
- **AsyncSubject**: only the last item, on completion.
Not thread-safe (use `.toSerialized()`); `StateFlow`/`SharedFlow` are the modern equivalents.

---



## 23. dispose() vs clear() on CompositeDisposable

**CompositeDisposable** means a container that collects multiple RxJava subscriptions to dispose of them all at once in lifecycle tear-down callbacks.

- **`clear()`**: disposes current disposables but keeps the container usable (add more later).
- **`dispose()`**: disposes all and marks the container dead — anything added after is immediately disposed.
Use `clear()` for reusable owners (ViewModel/Activity); `dispose()` only when permanently done.

---



## 24. Schedulers.io() vs Schedulers.computation()

**RxJava Schedulers** means thread execution coordinators, where `Schedulers.io()` handles blocking I/O tasks and `Schedulers.computation()` handles CPU-bound operations.

- **`io()`**: unbounded elastic thread pool for I/O-bound, blocking work (network, disk, DB).
- **`computation()`**: fixed pool sized to CPU cores for CPU-bound work (parsing, image processing, math).
Mismatching them (blocking I/O on computation, or heavy CPU on io) starves pools or spawns excessive threads.

---



## 25. Instant search with RxJava

**Search debounce implementation** means chaining `debounce`, `filter`, `distinctUntilChanged`, and `switchMap` operators to build responsive type-ahead inputs.

Combine four operators on a query stream (often a `PublishSubject` from an `EditText`): `debounce` (wait for typing to pause), `filter` (ignore too-short queries), `distinctUntilChanged` (skip unchanged), and `switchMap` (cancel in-flight request when a newer query arrives). `switchMap` is the key choice so stale results are discarded.

---



## 26. Pagination in RecyclerView with RxJava

**Pagination stream design** means converting list scroll offsets into page request triggers, throttling emissions, and appending new page results.

Turn "scrolled near bottom" into a stream (a `PublishSubject` emitting next page numbers) from an `OnScrollListener`, guard with `isLoading`, then `concatMap` to load pages **in order** and append. In modern apps the Paging 3 library handles this end-to-end and is usually preferred.

---



## Image Loading (Glide / Fresco / Coil)

## 27. How do Glide and Fresco work internally?

**Glide vs Fresco vs Coil** means the comparison between Glide (standard disk/memory caching), Fresco (low-end device memory handling), and Coil (lightweight Kotlin-first loading).

- **Glide**: multi-layer caching (active resources → memory `LruCache` → disk → network), a bitmap pool (`inBitmap` reuse) to cut GC churn, downsampling to the target view size, and lifecycle-aware request pausing/cancellation.
- **Fresco**: stores decoded bitmaps in native/ashmem (off-heap) memory, uses a three-level pipeline cache, centers on Drawees, and supports progressive JPEG.
Shared techniques: downsampling, bitmap reuse, multi-tier caching, cancel/throttle off-screen requests.

---



## 28. Glide vs Fresco vs Coil

**Android image libraries** means the comparison between Glide (flexible caching), Fresco (low-end memory efficiency), and Coil (Kotlin-first lightweight loading).

- **Glide**: Java, builder API, Java heap + bitmap pool; robust battle-tested default with great caching/GIFs.
- **Fresco**: native/off-heap memory, requires `DraweeView`; best for memory-critical, image-heavy apps; larger dependency.
- **Coil**: Kotlin-first, coroutine-based, small, first-class Compose (`AsyncImage`) support; idiomatic for new Kotlin/Compose apps. Coil 3 is a Kotlin Multiplatform library (Android/iOS/JVM/JS/wasm) with pluggable networking — add `coil-network-okhttp` or `coil-network-ktor` depending on your stack.

---



## Other Libraries

## 29. What is Flow in Kotlin?

**Kotlin Flow vs RxJava** means the choice between native coroutine-powered reactive streams (Flow), and the external Java-based reactive framework (RxJava).

Kotlin Coroutines' API for async streams — the coroutine-native equivalent of an RxJava Observable. Cold by default (producer runs fresh per collector). `flowOn` controls upstream dispatcher; `collect` is a suspending terminal. Hot variants: `StateFlow` (current value, like LiveData/BehaviourSubject) and `SharedFlow` (multicast events). Cancellation is automatic via scope. For new Kotlin code, Flow + Coroutines is the default.

---



## 30. What is the App Startup library?

**App Startup library** means a Jetpack library that provides a unified, efficient way to initialise component providers at application launch.

`androidx.startup` initializes components at launch through a single shared `InitializationProvider` instead of each library adding its own `ContentProvider` (which slows cold start). You define `Initializer`s with `dependencies()` for topological ordering; you can also opt out for lazy/manual init via `AppInitializer`.

---



## 31. Common Android libraries and ORMs

**Third-party libraries** means standard frameworks for networking, caching, serialisation, and dependency injection chosen based on size, performance, and maintenance.

Prefer well-maintained libraries for solved problems; write custom only for app-specific logic. Common picks: Retrofit/OkHttp/Ktor (networking), Moshi/kotlinx.serialisation (serialisation), Hilt/Koin (DI), Coroutines+Flow/RxJava (async), Coil/Glide/Fresco (images), Timber/LeakCanary (debug). **ORM/persistence**: Room is the default (compile-time SQL verification, observable `Flow`/`LiveData` queries); Realm is an object DB; DBFlow/OrmLite are legacy.

