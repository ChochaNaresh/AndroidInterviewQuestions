# General & Miscellaneous Android Interview Questions

This guide covers the broad "other topics" that appear in Android interviews but do not fit neatly into a single category: local persistence (SQLite, Room), cross-platform frameworks, securing API keys, network security configuration, push notifications via FCM, exact-time local notifications, dark theme, memory and profiling, annotation processing, and analytics signals such as uninstall tracking.

It also includes a curated set of HR/soft-skill questions, a **Preparation Tips** section, and a **Mock Interview** question list so you can rehearse end to end. Answers are written to be accurate as of 2026 (Android 14/15 era APIs, FCM HTTP v1).

---

## Table of Contents

1. [Describe SQLite](#1-describe-sqlite)
2. [Have you used Room?](#2-have-you-used-room)
3. [SQLite vs Room — when to use which](#3-sqlite-vs-room--when-to-use-which)
4. [Can we identify users who uninstalled our app?](#4-can-we-identify-users-who-uninstalled-our-app)
5. [Android development best practices](#5-android-development-best-practices)
6. [React Native vs Flutter](#6-react-native-vs-flutter)
7. [What metrics should you measure continuously?](#7-what-metrics-should-you-measure-continuously)
8. [How to avoid checking API keys into VCS](#8-how-to-avoid-checking-api-keys-into-vcs)
9. [How does Kotlin Multiplatform work?](#9-how-does-kotlin-multiplatform-work)
10. [How to use memory heap dumps](#10-how-to-use-memory-heap-dumps)
11. [How to implement Dark Theme](#11-how-to-implement-dark-theme)
12. [How to secure API keys in an Android app](#12-how-to-secure-api-keys-in-an-android-app)
13. [What is cleartext traffic? Network Security Config](#13-what-is-cleartext-traffic-network-security-config)
14. [Memory usage in Android](#14-memory-usage-in-android)
15. [Explain annotation processing](#15-explain-annotation-processing)
16. [How does the Android push notification system work?](#16-how-does-the-android-push-notification-system-work)
17. [Android push notification flow using FCM](#17-android-push-notification-flow-using-fcm)
18. [How to show a local notification at an exact time](#18-how-to-show-a-local-notification-at-an-exact-time)
19. [General / soft-skill HR questions](#19-general--soft-skill-hr-questions)
20. [Preparation Tips](#20-preparation-tips)
21. [Mock Interview Questions](#21-mock-interview-questions)

---

## 1. Describe SQLite

**SQLite** means a lightweight, serverless, relational SQL database engine embedded directly inside the Android OS.

Every app gets its own private SQLite database stored in the app's sandbox (`/data/data/<package>/databases/`). It is the canonical relational store on the platform.

Key characteristics:

- **Serverless / embedded** — there is no separate database process; the engine runs in-process as a C library.
- **Single file** — the entire database (tables, indices, schema) lives in one file.
- **ACID transactions** — atomic, consistent, isolated, durable, even across app/system crashes.
- **Dynamic typing** — SQLite uses type affinity rather than rigid column types.
- **SQL dialect** — supports most of SQL-92 (SELECT, JOIN, indexes, triggers, views).

Raw SQLite is exposed through `SQLiteOpenHelper`, which manages database creation and version migrations:

```kotlin
class AppDbHelper(context: Context) :
    SQLiteOpenHelper(context, "app.db", null, 1) {

    override fun onCreate(db: SQLiteDatabase) {
        db.execSQL(
            "CREATE TABLE user(" +
                "id INTEGER PRIMARY KEY AUTOINCREMENT, " +
                "name TEXT NOT NULL, " +
                "email TEXT)"
        )
    }

    override fun onUpgrade(db: SQLiteDatabase, oldV: Int, newV: Int) {
        db.execSQL("ALTER TABLE user ADD COLUMN phone TEXT")
    }
}

// Insert
val values = ContentValues().apply {
    put("name", "Ada")
    put("email", "ada@example.com")
}
helper.writableDatabase.insert("user", null, values)
```

The downsides of using raw SQLite directly: lots of boilerplate, no compile-time verification of SQL, manual cursor-to-object mapping, and error-prone migrations. That is exactly the gap Room fills.

**📚 Reference:** <https://developer.android.com/training/data-storage/sqlite>

---

## 2. Have you used Room?

**Room** means an official Jetpack database persistence library that provides a compile-time validated, object-oriented abstraction layer over raw SQLite.

**Room** is the Jetpack persistence library that provides an abstraction layer over SQLite. It keeps the power of SQLite while removing boilerplate and adding **compile-time verification** of SQL queries.

Room has three core components:

- **`@Entity`** — a class that maps to a table.
- **`@Dao`** — interface with annotated methods describing queries; Room generates the implementation.
- **`@Database`** — holds the database and is the main access point; lists entities and the version.

```kotlin
@Entity(tableName = "user")
data class User(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val name: String,
    val email: String?
)

@Dao
interface UserDao {
    @Query("SELECT * FROM user ORDER BY name")
    fun observeAll(): Flow<List<User>>          // reactive query

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(user: User)              // coroutine support

    @Query("DELETE FROM user WHERE id = :id")
    suspend fun deleteById(id: Long)
}

@Database(entities = [User::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}

val db = Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
    .build()
```

Why Room is preferred:

- **Compile-time SQL validation** — a typo in a `@Query` fails the build, not at runtime.
- **Coroutines + Flow** support — `suspend` DAO methods and observable `Flow`/`LiveData` queries that auto-emit on data change.
- **Boilerplate elimination** — no manual `Cursor` parsing.
- **Migrations** are first-class via `Migration` objects (and `@AutoMigration` for schema-diff-based migrations from a schema JSON).
- **Type converters** (`@TypeConverter`) for non-primitive types, and `@Relation`/embedded objects for joins.
- Integrates with **Paging 3** and Jetpack DataStore patterns.

Example migration:

```kotlin
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE user ADD COLUMN phone TEXT")
    }
}
Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
    .addMigrations(MIGRATION_1_2)
    .build()
```

**📚 Reference:** <https://developer.android.com/training/data-storage/room>

---

## 3. SQLite vs Room — when to use which

**SQLite vs Room** means the choice between writing manual SQLite helper boilerplate and runtime queries, and using a compile-safe ORM library (Room).

| Aspect | Raw SQLite (`SQLiteOpenHelper`) | Room |
|---|---|---|
| SQL verification | Runtime only | **Compile time** |
| Boilerplate | High (manual cursors) | Low (generated DAOs) |
| Reactive queries | Manual | `Flow` / `LiveData` built-in |
| Coroutines | Manual threading | `suspend` support |
| Migrations | Manual `onUpgrade` | `Migration` + `@AutoMigration` |
| Learning curve | Pure SQL | SQL + annotations |

**Use Room** for virtually all new apps. **Drop to raw SQLite** only for highly specialized cases (e.g., dynamic schemas, an existing legacy `.db`, or extreme control over the cursor lifecycle). Room sits on top of SQLite, so you do not lose performance.

---

## 4. Can we identify users who uninstalled our app?

**Uninstall detection** means the process of tracking app uninstalls indirectly using silent push notifications or backend logs, since the OS provides no direct on-device callback.

What you can do is **infer** uninstalls server-side:

1. **FCM unregistration signal (most reliable).** When you send a message to a token that belongs to an uninstalled app, FCM eventually returns an error such as `UNREGISTERED` (HTTP 404 / `messaging/registration-token-not-registered`). Your backend treats that as "device gone" and marks the user inactive. There is normally a delay (days) because FCM only learns the app is gone when it next tries to reach the device.

2. **App Inactivity / heartbeat.** Track `lastSeen` timestamps on your server (periodic pings, session start). A device that stops sending heartbeats for a long window is likely uninstalled or dormant.

3. **Analytics / attribution platforms.** Tools such as Firebase Analytics, AppsFlyer, Adjust, or Branch offer **uninstall measurement**. They typically send a periodic silent push; when delivery fails (token unregistered) they classify it as an uninstall and report an uninstall event/cohort.

4. **Google Play install/uninstall referrer & Play Console statistics** give aggregate uninstall counts but not individual user identity.

**Privacy note:** you can only correlate this to a "user" if you already have a server-side identity (account/login). You cannot identify anonymous users after uninstall, and you must comply with GDPR/CCPA and Play policy when tracking this.

```text
Server sends push --> FCM --> device unreachable / app removed
FCM responds: 404  UNREGISTERED
Server: mark token stale -> count as probable uninstall
```

---

## 5. Android development best practices

**Android development best practices** means a set of guidelines including lifecycle-aware components, offline-first architectures, memory leak prevention, and secure key storage.

- **Architecture:** Use a clear, layered architecture (MVVM / MVI) with unidirectional data flow. Follow Google's recommended app architecture: UI layer (Compose/Views + ViewModel) -> domain layer (optional use cases) -> data layer (repositories + data sources).
- **Single source of truth:** Repositories expose data; UI observes immutable state.
- **Lifecycle-awareness:** Use `ViewModel`, `StateFlow`/`LiveData`, `repeatOnLifecycle`, and lifecycle-scoped coroutines to avoid leaks and wasted work.
- **Dependency injection:** Hilt/Dagger (or Koin) instead of manual singletons.
- **Reactive & async:** Kotlin coroutines + Flow for concurrency; never block the main thread.
- **Background work:** WorkManager for deferrable/guaranteed work; avoid raw services.
- **Persistence:** Room for relational data; DataStore (not SharedPreferences) for key-value/proto.
- **Networking:** Retrofit + OkHttp + a serializer (Moshi/kotlinx.serialisation); centralized error handling.
- **UI:** Prefer Jetpack Compose for new UI; follow Material 3; support multiple screen sizes and dark theme.
- **Resources & i18n:** Externalize strings; support RTL and localization; use vector drawables.
- **Performance:** Avoid overdraw, lazy-load, paginate, use baseline profiles, cache wisely.
- **Security:** No secrets in VCS, HTTPS only, Android Keystore (with DataStore + Tink for encrypted storage now that `EncryptedSharedPreferences` is deprecated), network security config.
- **Quality:** Unit + UI tests, CI/CD, static analysis (ktlint/detekt/Lint), code review, gradle modularization.
- **Observability:** Crash reporting (Crashlytics) and analytics from day one.

**📚 Reference:** <https://outcomeschool.com/blog/android-development-best-practices>

---

## 6. React Native vs Flutter

**Cross-platform frameworks** means the comparison between React Native using JavaScript and bridge-based rendering, and Flutter using Dart and a custom graphics engine.

The difference is the language, rendering model, and ecosystem.

| Dimension | React Native | Flutter |
|---|---|---|
| Language | JavaScript / TypeScript | Dart |
| UI rendering | Maps to **native** platform widgets (bridge / JSI / Fabric) | Renders **its own** widgets with the Skia/Impeller engine (pixel-level control) |
| Look & feel | Native by default | Consistent across platforms; native look via Material/Cupertino widgets |
| Performance | Good; historically bridge overhead, now improved with JSI/Fabric | Generally excellent, AOT-compiled to native ARM |
| Hot reload | Yes (Fast Refresh) | Yes (very fast) |
| Backed by | Meta | Google |
| Ecosystem | Huge JS/npm ecosystem, large community | Growing fast, rich first-party widget set |
| Code sharing | Logic + UI; some native modules needed | Logic + UI; strong out-of-the-box |

**Summary answer:** React Native is attractive for teams with JavaScript/React expertise and where a native look-and-feel and reuse of web skills matter. Flutter offers pixel-perfect consistent UI, strong performance via AOT compilation, and a complete widget toolkit, at the cost of learning Dart. For purely native Android with the best platform fidelity and access to every new API immediately, neither beats native Kotlin + Compose; cross-platform is a tradeoff for code sharing across iOS/Android.

**📚 Reference:** <https://outcomeschool.com/blog/react-native-vs-flutter>

---

## 7. What metrics should you measure continuously?

**Continuous performance monitoring** means tracking key production metrics like ANR rates, crash frequencies, rendering frame drops, and battery usage via Android Vitals.

- **App startup time** — cold, warm, and hot start (Time To Initial Display / Time To Full Display). Use Macrobenchmark and Play Console's Android Vitals.
- **Frame rendering / jank** — frozen frames (>700ms) and slow frames (>16ms at 60fps). Track via `JankStats` and Android Vitals.
- **ANRs (Application Not Responding)** — main-thread blocks > 5s. A key Play Console "bad behaviour" metric that affects ranking.
- **Crash rate / crash-free sessions** — via Crashlytics / Play Console.
- **Memory usage & leaks** — heap growth, leaks (LeakCanary), OOMs.
- **Battery / power** — wakelocks, background CPU, network wakeups.
- **Network** — request latency, payload size, error rates, cache hit ratio.
- **App (binary) size** — APK/AAB and download size; affects install conversion.
- **CPU usage** — sustained CPU, jank correlation.
- **Business / engagement metrics** — retention, DAU/MAU, conversion (via analytics).

Tools: **Android Studio Profiler**, **Macrobenchmark/Microbenchmark**, **JankStats**, **Perfetto/Systrace**, **Firebase Performance Monitoring**, **Play Console Android Vitals**, **LeakCanary**.

**📚 Reference:** <https://outcomeschool.com/blog/android-app-performance-metrics>

---

## 8. How to avoid checking API keys into VCS

**VCS secrets management** means keeping API keys and credentials out of Git repositories by storing them in local `local.properties` and reading them via Gradle.

1. **Store keys outside committed source** — typically in `local.properties` (which is git-ignored by default) or in environment variables / CI secret stores.

```properties
# local.properties  (NOT committed)
MAPS_API_KEY=AIzaSyXXXXXXXXXXXXXXXX
```

2. **Expose them to code via Gradle `BuildConfig` or resources** without hardcoding:

```kotlin
// app/build.gradle.kts
import java.util.Properties

val props = Properties().apply {
    val f = rootProject.file("local.properties")
    if (f.exists()) f.inputStream().use { load(it) }
}

android {
    defaultConfig {
        buildConfigField(
            "String", "MAPS_API_KEY", "\"${props["MAPS_API_KEY"]}\""
        )
    }
    buildFeatures { buildConfig = true }
}
```

```kotlin
val key = BuildConfig.MAPS_API_KEY
```

3. **Use the Secrets Gradle Plugin** (`com.google.android.libraries.mapsplatform.secrets-gradle-plugin`) for the common manifest-placeholder case (e.g., Maps keys).

4. **`.gitignore`** must include `local.properties`, keystores (`*.jks`/`*.keystore`), `google-services.json` if sensitive, and any `.env` files.

5. **CI/CD** injects secrets at build time from the platform's secret manager (GitHub Actions secrets, etc.).

6. **If a key was already committed**, rotate/revoke it — git history retains it forever; removing the file is not enough.

> Important: `BuildConfig` only keeps keys out of *source control*. It does **not** make them secure on-device (see Q12).

---

## 9. How does Kotlin Multiplatform work?

**Kotlin Multiplatform (KMP)** means a framework that compiles shared Kotlin code into native JVM, JS, or iOS libraries to reuse business logic across multiple platforms.

How it works:

- **`commonMain`** holds platform-agnostic code (models, repositories, networking, use cases).
- Kotlin compiles this to **different targets** using different backends:
  - **Android/JVM** -> JVM bytecode.
  - **iOS/native** -> via **Kotlin/Native** (LLVM) producing a native framework consumable by Swift/Obj-C.
  - **JS/Wasm** -> for web.
- **`expect`/`actual`** mechanism declares an API in common code and provides platform-specific implementations:

```kotlin
// commonMain
expect fun platformName(): String

// androidMain
actual fun platformName(): String = "Android ${Build.VERSION.SDK_INT}"

// iosMain
actual fun platformName(): String = UIDevice.currentDevice.systemName()
```

- You typically share **logic** and keep UI native (Jetpack Compose on Android, SwiftUI on iOS). **Compose Multiplatform** can additionally share UI.
- Common libraries: Ktor (networking), kotlinx.coroutines, kotlinx.serialisation, SQLDelight (DB), Koin (DI).

Benefit: one tested implementation of core logic, native performance and UX on each platform, incremental adoption (you can share just one module).

**📚 Reference:** <https://outcomeschool.com/blog/how-does-the-kotlin-multiplatform-work>

---

## 10. How to use memory heap dumps

**Memory heap dumps** means capturing a snapshot of all active Java/Kotlin object instances in memory at a specific point in time to diagnose memory leaks.

You use it to find **memory leaks** and **bloat**.

Workflow:

1. **Capture** a dump with the **Android Studio Memory Profiler** ("Capture heap dump") or via `Debug.dumpHprofData(path)`.
2. **Analyze** the `.hprof`:
   - Sort by **retained size** (memory that would be freed if the object were GC'd) and **shallow size**.
   - Group by class to find unexpectedly large counts (e.g., thousands of `Bitmap`s).
   - Inspect the **reference chain / dominator tree** to see *why* an object is still alive — the GC root path reveals the leak.
3. **Common leak signatures:** an `Activity`/`Fragment`/`Context` retained after destruction, static references holding views, long-lived listeners/handlers, inner classes holding the outer context, unclosed resources.
4. **Automate leak detection** with **LeakCanary**, which watches destroyed objects, triggers a dump, and analyzes the shortest path to the GC root automatically.
5. Tools for offline analysis: **Eclipse MAT**, **Android Studio's profiler**.

```kotlin
// Programmatic heap dump
Debug.dumpHprofData(File(filesDir, "dump.hprof").absolutePath)
```

The key interpretation skill: large **retained size** + a reference chain to a destroyed component = a leak to fix.

---

## 11. How to implement Dark Theme

**Dark theme implementation** means defining DayNight styling themes and configuring alternative drawable/color resource qualifiers for night mode.

Implement it via **resource qualifiers** and **`DayNight`/Material 3** themes so the system swaps resources automatically.

**Views (XML) approach:**

1. Use a `DayNight` parent theme:

```xml
<!-- res/values/themes.xml -->
<style name="Theme.App" parent="Theme.Material3.DayNight" />
```

2. Provide dark resources in `-night` qualified folders:

```text
res/values/colors.xml          (light)
res/values-night/colors.xml    (dark)
res/drawable/...               res/drawable-night/...
```

3. Reference **theme attributes** (e.g., `?attr/colorSurface`, `?android:attr/textColorPrimary`) instead of hardcoded colors so they resolve per mode.

4. Let users override the mode:

```kotlin
AppCompatDelegate.setDefaultNightMode(
    AppCompatDelegate.MODE_NIGHT_YES          // or NO / FOLLOW_SYSTEM
)
```

**Jetpack Compose approach:**

```kotlin
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val colors = if (darkTheme) darkColorScheme() else lightColorScheme()
    MaterialTheme(colorScheme = colors, content = content)
}
```

Best practices: support **Follow System** (Android 10+ system-wide toggle), test contrast, avoid pure black where it harms readability, use dynamic color (Material You) on Android 12+, and ensure `forceDarkAllowed` is only used as a fallback for legacy screens.

---

## 12. How to secure API keys in an Android app

**API key protection** means obfuscating credentials, using native C++ wrappers, or restricting key access on the server backend since any client APK can be decompiled.

So "securing" keys is about raising cost and moving secrets off the device.

Defense in depth, strongest first:

1. **Don't ship the secret at all — proxy through your backend.** The app calls *your* server, which holds the third-party key and forwards the request. The client never sees the key. This is the only truly safe option for high-value secrets.

2. **Use short-lived tokens** issued after authentication (OAuth access tokens, signed STS tokens) instead of long-lived static keys.

3. **Restrict the key server-side.** For keys that must ship (e.g., Maps API keys), lock them down by **package name + SHA-1 signing certificate**, by API, and by quota in the provider console — so a stolen key is useless elsewhere.

4. **Use the Android Keystore** for secrets generated/stored at runtime (e.g., per-user tokens). The Keystore keeps key material in hardware (TEE/StrongBox) and never exposes it to the app. To persist encrypted values, derive a key from the Keystore and store ciphertext yourself, or use a library on top of it.

> Note: `EncryptedSharedPreferences` / `EncryptedFile` from `androidx.security:security-crypto` were a common option, but that library was **deprecated in April 2025** (1.1.0-alpha07) and is no longer maintained. For new code prefer **Jetpack DataStore encrypted with Google Tink** (encrypt the stream before it hits disk), or use the Android Keystore directly. Example below shown for reference on existing projects:

```kotlin
val masterKey = MasterKey.Builder(context)
    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
    .build()

val prefs = EncryptedSharedPreferences.create(
    context, "secure_prefs", masterKey,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
)
```

5. **NDK / native code** — storing keys in C/C++ raises the bar (harder to read than Java strings) but is still extractable; treat as obfuscation, not security.

6. **Obfuscation (R8/ProGuard)** makes reverse engineering harder but does not encrypt strings.

7. **Play Integrity API / App Attest** to verify requests come from a genuine, unmodified app.

**Interview soundbite:** keep keys out of VCS (Q8) *and* out of the client wherever possible; for keys that must ship, restrict them by signing cert + package and use server-side proxying for anything sensitive.

---

## 13. What is cleartext traffic? Network Security Config

**Cleartext traffic** means unencrypted network data sent over HTTP, which is disabled by default on modern Android versions in favour of HTTPS.

It can be read or tampered with by anyone on the path (Wi-Fi sniffing, MITM), so it is a security risk.

**Default behaviour:** Since Android 9 (API 28), **cleartext is disabled by default** — apps targeting API 28+ that attempt an `http://` connection get an error unless explicitly allowed.

**Network Security Config** is an XML file that declaratively controls TLS/cleartext policy, trust anchors, and certificate pinning without code changes:

```xml
<!-- AndroidManifest.xml -->
<application
    android:networkSecurityConfig="@xml/network_security_config" ... />
```

```xml
<!-- res/xml/network_security_config.xml -->
<network-security-config>
    <!-- Block cleartext globally -->
    <base-config cleartextTrafficPermitted="false" />

    <!-- Allow cleartext only for a specific dev/test host -->
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="true">localhost</domain>
    </domain-config>

    <!-- Certificate pinning example -->
    <domain-config>
        <domain includeSubdomains="true">api.example.com</domain>
        <pin-set>
            <pin digest="SHA-256">base64PinValue=</pin>
        </pin-set>
    </domain-config>
</network-security-config>
```

You can also set `android:usesCleartextTraffic="false"` on `<application>` as a coarse switch. Best practice: **HTTPS everywhere**, cleartext disabled, and consider certificate pinning for sensitive APIs (with care for rotation).

**📚 Reference:** <https://developer.android.com/training/articles/security-config> · <https://www.linkedin.com/posts/amit-shekhar-iitbhu_softwareengineer-androiddev-android-activity-7281879316601126913-x1az>

---

## 14. Memory usage in Android

**Android process memory limits** means the restricted RAM boundaries (heaps) allocated to individual applications by the OS, which vary by device RAM size.

Exceeding it triggers an `OutOfMemoryError`. The runtime (ART) garbage-collects unreachable objects, but GC has a CPU/jank cost, so the goal is to keep allocations low and lifetimes short.

Key points to cover:

- **Memory model:** Java/Kotlin heap (managed by ART GC), native heap, graphics/bitmap memory, code, and stack. Tools report **PSS/RSS** and per-category usage.
- **Low-memory handling:** The system kills background processes under pressure (LMK). Respond to `onTrimMemory()` / `ComponentCallbacks2` levels to release caches.
- **Common causes of high usage / leaks:**
  - Holding `Activity`/`Context` references beyond their lifecycle (static fields, singletons, long-lived listeners).
  - Large/unrecycled `Bitmap`s — use `inSampleSize`, Glide/Coil with downsampling and memory caches.
  - Unbounded caches/collections; not removing observers; leaked handlers/threads.
  - Inner/anonymous classes capturing outer context.
- **Mitigation:** `WeakReference` where appropriate, lifecycle-aware components, `ViewModel` for retained state, paginate large lists, clear references in `onDestroy`, use `LruCache`.
- **Tools:** Android Studio **Memory Profiler**, **LeakCanary**, heap dumps (Q10), `adb shell dumpsys meminfo <pkg>`.

```kotlin
override fun onTrimMemory(level: Int) {
    if (level >= TRIM_MEMORY_RUNNING_LOW) {
        imageCache.evictAll()   // free non-critical memory
    }
}
```

---

## 15. Explain annotation processing

**Annotation processing** means a compile-time build step that scans source code annotations and generates additional Java/Kotlin source files.

It powers many Android libraries — Room, Dagger/Hilt, Moshi, Glide, Data Binding — letting them avoid runtime reflection by generating boilerplate at build time.

How it works:

- An **annotation processor** runs during compilation, scans annotated elements (classes, methods, fields), and emits new `.java`/`.kt` files that the compiler then compiles alongside your code.
- Java's standard mechanism is **APT** via `javax.annotation.processing.Processor` (registered with `@AutoService` or `META-INF/services`).
- For Kotlin, the older bridge is **`kapt`** (Kotlin Annotation Processing Tool), which generates Java stubs so Java processors can run. The modern, faster replacement is **KSP (Kotlin Symbol Processing)**, which understands Kotlin directly and is significantly faster — Room/Hilt/Moshi now support KSP.

```kotlin
// build.gradle.kts — KSP example
plugins { id("com.google.devtools.ksp") }
dependencies {
    implementation("androidx.room:room-runtime:2.x")
    ksp("androidx.room:room-compiler:2.x")   // generates DAO impls
}
```

Why it matters: generated code is **type-safe, debuggable, and reflection-free**, so it is faster and catches errors at compile time. Trade-off: it adds to build time (KSP mitigates this).

When asked "what's the difference between kapt and KSP?": kapt generates Java stubs and is slower; KSP operates on Kotlin's symbol model directly, is faster, and is the recommended path going forward.

---

## 16. How does the Android push notification system work?

**Android push notification system** means the architecture where a server sends payloads to Google's FCM servers, which then deliver them to a background system service on the device.

On Android the transport is **Firebase Cloud Messaging (FCM)**, which relies on a persistent, battery-efficient connection between the device and Google's servers.

High-level flow:

```text
[Your App Server]  --(send msg + device token)-->  [FCM Backend (Google)]
[FCM Backend]      --(persistent connection)----->  [Google Play Services on device]
[Play Services]    --(deliver)-------------------->  [Your App]
```

1. **Registration.** On first run the app's FCM SDK requests a **registration token** that uniquely identifies that app install on that device.
2. **Token upload.** The app sends this token to *your* backend, which stores it against the user.
3. **Send.** Your backend (or a console) sends a message to FCM targeting a token, a **topic**, or a **condition**.
4. **Routing & delivery.** FCM holds a single, persistent, multiplexed connection via **Google Play Services**; it routes the message to the right device, queuing it if the device is offline.
5. **Display / handling.** The app either shows the notification (notification message, handled by the SDK in background) or processes a data payload in code.

Notes: FCM is the standard transport; **WorkManager/AlarmManager are for local, not push, scheduling**. Without Google Play Services (e.g., some China/AOSP devices), vendors provide their own push (Huawei HMS, etc.). From Android 13+, posting notifications requires the runtime **`POST_NOTIFICATIONS`** permission.

**📚 Reference:** <https://youtu.be/810IFG2sWlc> · <https://firebase.google.com/docs/cloud-messaging>

---

## 17. Android push notification flow using FCM

**FCM push flow** means the sequence of registering a device token, sending a JSON payload from a backend via FCM HTTP v1, and receiving it in `FirebaseMessagingService`.

### Client setup

1. Add the Firebase SDK and `google-services.json`; declare a messaging service.

```kotlin
class MyFirebaseService : FirebaseMessagingService() {

    // Called when a new token is generated / rotated
    override fun onNewToken(token: String) {
        sendTokenToServer(token)   // upload to your backend
    }

    // Called for data messages (and notification messages while foregrounded)
    override fun onMessageReceived(message: RemoteMessage) {
        message.notification?.let { showNotification(it.title, it.body) }
        val data = message.data    // custom key-value payload
    }
}
```

```xml
<service
    android:name=".MyFirebaseService"
    android:exported="false">
    <intent-filter>
        <action android:name="com.google.firebase.MESSAGING_EVENT" />
    </intent-filter>
</service>
```

2. Retrieve the token explicitly when needed:

```kotlin
FirebaseMessaging.getInstance().token.addOnSuccessListener { token ->
    sendTokenToServer(token)
}
```

### Server send (HTTP v1)

The server POSTs an OAuth2-authenticated request to:

```text
POST https://fcm.googleapis.com/v1/projects/<PROJECT_ID>/messages:send
Authorization: Bearer <OAuth2 access token>
```

```json
{
  "message": {
    "token": "<device-registration-token>",
    "notification": { "title": "Order shipped", "body": "Arrives tomorrow" },
    "data": { "orderId": "12345", "type": "SHIPMENT" },
    "android": { "priority": "high" }
  }
}
```

### Notification vs data messages — delivery behaviour

- **Notification message** (`notification` block): when the app is in the **background/killed**, the **FCM SDK auto-displays** it in the system tray; `onMessageReceived` is *not* called until the user taps it (the data payload arrives in the launch intent).
- **Data message** (`data` only): **always** delivered to `onMessageReceived` so your code fully controls handling — required if you want custom logic in the background.
- **Foreground:** `onMessageReceived` is called for both, giving you both payloads.

### Targeting options

- **Token** — a specific device.
- **Topic** — devices subscribed via `subscribeToTopic("news")`; broadcast to many.
- **Condition** — boolean combination of topics.

### Reliability

- Send **high priority** for time-sensitive messages (wakes the device); normal priority may be batched/deferred under Doze.
- Handle the **`UNREGISTERED`/404** response to prune stale tokens (links back to Q4 uninstall detection).
- The legacy FCM server keys / legacy HTTP API are deprecated; use **HTTP v1** with OAuth2 service-account credentials.

**📚 Reference:** <https://www.youtube.com/watch?v=TrufwW4ILHg> · <https://firebase.google.com/docs/cloud-messaging/send/v1-api>

---

## 18. How to show a local notification at an exact time

**Exact time notifications** means scheduling precise alarms using `AlarmManager` combined with intent broadcasts to display local notifications.

This is **local scheduling** (no server, works offline) — different from FCM push.

### The APIs

- `setExact(...)` / `setExactAndAllowWhileIdle(...)` — fire at an exact time; the `...AllowWhileIdle` variant fires even during **Doze**.
- For repeating exact behaviour, reschedule from the receiver (exact repeating was removed).

### Permissions (this is the part interviewers probe)

Exact alarms are **restricted** because they impact battery:

- **Android 12 (API 31):** introduced the **`SCHEDULE_EXACT_ALARM`** permission. It was granted by default but revocable by the user.
- **Android 13 (API 33)+ / Android 14 (API 34):** `SCHEDULE_EXACT_ALARM` is **denied by default** for most newly installed apps targeting API 33+. You must check and, if needed, send the user to settings to grant it.
- **`USE_EXACT_ALARM`** is a *normal* permission auto-granted at install, but it is **only allowed for apps whose core purpose is alarms/calendars/timers** (Play policy will reject misuse). Use this only if exact timing is the app's primary function; otherwise use `SCHEDULE_EXACT_ALARM`.

```xml
<!-- Most apps: user-grantable -->
<uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM" />
<!-- Only alarm/calendar/clock apps -->
<uses-permission android:name="android.permission.USE_EXACT_ALARM" />
```

```kotlin
val am = context.getSystemService(Context.ALARM_SERVICE) as AlarmManager

// Android 12+: verify the permission before scheduling, or you get SecurityException
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S && !am.canScheduleExactAlarms()) {
    // Route user to grant it
    context.startActivity(
        Intent(Settings.ACTION_REQUEST_SCHEDULE_EXACT_ALARM)
    )
    return
}

val intent = Intent(context, AlarmReceiver::class.java)
val pi = PendingIntent.getBroadcast(
    context, 0, intent,
    PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT
)

am.setExactAndAllowWhileIdle(
    AlarmManager.RTC_WAKEUP,
    triggerAtMillis,           // exact wall-clock time
    pi
)
```

```kotlin
class AlarmReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val notif = NotificationCompat.Builder(context, CHANNEL_ID)
            .setSmallIcon(R.drawable.ic_alarm)
            .setContentTitle("Reminder")
            .setContentText("It's time!")
            .setAutoCancel(true)
            .build()
        // Needs POST_NOTIFICATIONS permission (Android 13+)
        NotificationManagerCompat.from(context).notify(1, notif)
    }
}
```

Additional notes:

- **`SecurityException`** is thrown if you call `setExactAndAllowWhileIdle()`/`setExact()` without the required permission on Android 12+.
- Listen for **`AlarmManager.ACTION_SCHEDULE_EXACT_ALARM_PERMISSION_STATE_CHANGED`** to react when the user grants the permission.
- **Reboots clear alarms** — re-register on `BOOT_COMPLETED` (with `RECEIVE_BOOT_COMPLETED`).
- **Android 13+** requires runtime **`POST_NOTIFICATIONS`** to actually display the notification.
- If exact timing is *not* truly required, prefer **WorkManager** (inexact, battery-friendlier) — interviewers like to hear that you reach for exact alarms only when justified.

**📚 Reference:** <https://developer.android.com/develop/background-work/services/alarms> · <https://developer.android.com/about/versions/14/changes/schedule-exact-alarms>

---

## 19. General / soft-skill HR questions

**HR interview questions** means the behavioural and scenario-based questions used by recruiters to assess communication, teamwork, and cultural alignment.

Prepare honest, concrete answers (use the STAR method — see Q20).

1. Name the last three apps you worked on. Which did you like most and why?
2. What is your favorite programming language, and why?
3. Describe the software development process at your most recent job. What did you enjoy? What would you change?
4. What is the most challenging thing you have done in an application?
5. Which websites/blogs/channels do you follow to stay updated? (e.g., [developer.android.com](https://developer.android.com/), [StackOverflow](https://stackoverflow.com/questions/tagged/android), [Android Weekly](https://androidweekly.net/), [Kotlin Weekly](http://www.kotlinweekly.net/), [Droidcon](https://www.droidcon.com/), [KotlinConf](https://kotlinconf.com/))
6. Why do you consider yourself a Senior/Junior developer? How do you define seniority?
7. Do you do any documentation?
8. What is your most proud Android development achievement?
9. What are your strengths and areas for improvement in Android development?
10. What project management tools have you used? (Jira, Asana, Trello, etc.)
11. Are you familiar with Agile, Scrum, Sprints, etc.?
12. What aspects of your job do you enjoy?
13. What type of team do you like to work in?
14. **What is GitFlow, and do you follow it?** — A Git branching model with long-lived feature branches plus multiple primary branches (`main`, `develop`, release, hotfix).
15. **What is trunk-based development?** — A version-control practice where developers merge small, frequent updates directly into a shared `main`/trunk, using short-lived branches instead of long-lived feature branches.
16. Describe Test-Driven Development (TDD) — write a failing test, make it pass, refactor (red-green-refactor).
17. Do you prefer working on app backend/core or UI/view?
18. "Teach me, as a non-technical person, something technical" — assesses your communication and ability to simplify.

**📚 Reference:** Repo `GENERAL.md`

---

## 20. Preparation Tips

**Interview preparation strategies** means reviewing fundamentals, mock-interviewing out loud, coding on whiteboards, and preparing project architecture deep dives.

### Dos

- Get enough sleep the night before; being well-rested improves focus.
- Research the company, the role, and your interviewer(s) thoroughly.
- Practice Android coding problems and technical questions beforehand.
- Do mock interviews with a friend or colleague.
- Dress appropriately and professionally.
- Arrive (or log in) early so you have time to settle.
- Take deep breaths and relax; manage nerves.
- Visualize succeeding to build a positive mindset.
- Remember the interviewer is human too.
- Be honest about your skills — nobody expects you to know everything.
- **Communicate your thought process** and ask clarifying questions.
- Ask questions about the company, role, and team.
- Follow up with a thank-you note/email.

### Don'ts

- Don't memorize canned answers; adapt to the conversation flow.
- Don't be afraid to ask questions.
- Don't be late or unprepared.
- Don't be arrogant or dismissive — stay humble and respectful.
- Don't oversell yourself or pretend to be someone you're not.
- Don't get distracted (e.g., dwelling on a previous question) and miss what's being asked now.

### The STAR method (for behavioural questions)

Structure stories as:

- **S**ituation — the context/challenge you faced.
- **T**ask — your specific responsibility/goal.
- **A**ction — what *you* did and decisions you made.
- **R**esult — the outcome, ideally quantified (e.g., "reduced cold start by 30%").

### Smart questions to ask at the end

- How would you describe the company's culture?
- What is your favorite thing about working here?
- How do you see the company evolving over the next five years?
- How does the company define and demonstrate its values?
- What qualities make someone successful in this role/company?

**📚 Reference:** Repo `PREPARATIONS.md`

---

## 21. Mock Interview Questions

**Mock interview practice** means rehearsing typical question sets covering OOP, Kotlin, Jetpack, architecture, and DSA to build confidence.

Many overlap with deeper topics elsewhere in this guide — pointers note where to read more.

1. **Tell me about the most challenging Android app you've worked on and your specific contributions.** — Behavioural; use STAR (Q20). Cover architecture, integration, security.
2. **How do you ensure your apps adhere to Material Design guidelines?** — Material Components/Material 3 library, official docs, design reviews, UI tests. See Dark Theme (Q11).
3. **How do you manage compatibility across different Android OS versions?** — AndroidX/Jetpack libraries, sensible `minSdk`/`targetSdk`, feature checks via `Build.VERSION`, testing on emulators + real devices.
4. **Walk me through identifying and resolving a tricky bug.** — Reproduce -> Logcat/ADB -> isolate -> fix -> regression test. Mention debugger and crash reports.
5. **What strategies optimise app performance?** — Memory, app size, CPU; Android Profiler; lazy loading; efficient data structures. See Metrics (Q7), Memory (Q14).
6. **Describe integrating a third-party API.** — Retrofit/OkHttp, secure transport, error handling, retries. See securing keys (Q12), cleartext (Q13).
7. **Experience with Git/SVN?** — Branching, merging, PR collaboration, GitFlow vs trunk-based (Q19).
8. **Experience with automated testing frameworks?** — JUnit, Mockito, Espresso, UI Automator, CI/CD integration.
9. **How do you design for multiple device sizes/resolutions?** — ConstraintLayout, size qualifiers, vector assets, responsive Compose layouts.
10. **How do you handle memory management issues?** — Memory Profiler, WeakReferences, lifecycle-aware context handling, LeakCanary. See Q10 and Q14.
11. **How do you ensure secure data storage and transmission?** — Encryption at rest (Keystore/EncryptedSharedPreferences) and in transit (HTTPS/TLS), OWASP MASVS. See Q12, Q13.
12. **A time you troubleshot a major issue in a live app.** — Behavioural; e.g., ANR/deadlock found via crash reports + analysis, then fixed. STAR.
13. **How do you stay up to date with Android?** — Blogs, conferences, forums, open source. See Q19 references.
14. **How do you design intuitive, user-friendly interfaces?** — Usability testing, user feedback, consistent navigation, Material patterns.
15. **How do you reduce battery consumption?** — Batch network calls, WorkManager/JobScheduler for background work, avoid wakelocks, profile power. See Q7, Q18.
16. **How do you handle localization and internationalization?** — String resources per locale, RTL support, layout expansion, `Locale` handling.
17. **How do you make an app accessible?** — Content descriptions, contrast ratios, TalkBack/screen-reader support, touch target sizing.
18. **How do you deploy apps?** — Play Console, staged/percentage rollouts, monitoring vitals, gathering feedback.

**📚 Reference:** Repo `ANDROID_MOCK1.md`

---

*Sources: Repo A `README.md` "Other Topics"; Repo B `GENERAL.md`, `PREPARATIONS.md`, `ANDROID_MOCK1.md`. Android Developers, Firebase, and outcomeschool references linked inline. Verified against current Android 14/15 exact-alarm rules and FCM HTTP v1 as of 2026.*
