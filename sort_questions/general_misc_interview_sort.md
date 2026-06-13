# General & Miscellaneous Android — Short Answers (Quick Revision)

> Condensed answers for rapid review before an interview. For full explanations and code see `general_misc_interview.md`.

---

## 1. Describe SQLite

**SQLite** means a lightweight, serverless, relational SQL database engine embedded directly inside the Android OS.

A lightweight, serverless, zero-config, transactional SQL database embedded in Android; each app gets a private single-file DB in its sandbox. Characteristics: in-process (no server), single file, ACID, dynamic typing, SQL-92 dialect. Exposed via `SQLiteOpenHelper` (handles creation/migrations) — but raw use means boilerplate, no compile-time SQL checks, and manual cursor mapping, which Room fixes.

---



## 2. Have you used Room?

**Room** means an official Jetpack database persistence library that provides a compile-time validated, object-oriented abstraction layer over raw SQLite.

Yes — Room is the Jetpack abstraction over SQLite with three components: `@Entity` (table), `@Dao` (queries, generated impl), `@Database` (access point). Benefits: compile-time SQL validation, coroutines/Flow support, no manual cursor parsing, first-class migrations (`Migration`/`@AutoMigration`), type converters/`@Relation`, and Paging 3 integration.

---



## 3. SQLite vs Room — when to use which

**SQLite vs Room** means the choice between writing manual SQLite helper boilerplate and runtime queries, and using a compile-safe ORM library (Room).

Room adds compile-time SQL verification, low boilerplate (generated DAOs), built-in Flow/LiveData and `suspend` support, and `Migration`/`@AutoMigration` — at the cost of learning annotations. Use Room for virtually all new apps; drop to raw SQLite only for specialized cases (dynamic schemas, legacy `.db`, extreme cursor control). Room sits on SQLite, so no perf loss.

---



## 4. Can we identify users who uninstalled our app?

**Uninstall detection** means the process of tracking app uninstalls indirectly using silent push notifications or backend logs, since the OS provides no direct on-device callback.

You can't directly detect an uninstall on-device (no callback). Infer it server-side: the most reliable signal is **FCM `UNREGISTERED`/404** when sending to a token of a removed app (delayed by days); also app inactivity/heartbeats, analytics/attribution platforms (Firebase/AppsFlyer/Adjust), and aggregate Play Console stats. You can only tie it to a user if you have a server-side identity; comply with GDPR/CCPA.

---



## 5. Android development best practices

**Android development best practices** means a set of guidelines including lifecycle-aware components, offline-first architectures, memory leak prevention, and secure key storage.

Layered architecture (MVVM/MVI, UDF), single source of truth, lifecycle-awareness (`ViewModel`, `repeatOnLifecycle`), DI (Hilt), coroutines + Flow (never block main thread), WorkManager for background, Room + DataStore for persistence, Retrofit/OkHttp networking, Jetpack Compose + Material 3, i18n/RTL, performance (baseline profiles, pagination), security (no secrets in VCS, HTTPS, Keystore), tests + CI + static analysis, and crash/analytics from day one.

---



## 6. React Native vs Flutter

**Cross-platform frameworks** means the comparison between React Native using JavaScript and bridge-based rendering, and Flutter using Dart and a custom graphics engine.

Both target Android + iOS from one codebase. **React Native** uses JS/TS and maps to **native** widgets (good for JS teams, native look). **Flutter** uses Dart and renders its **own** widgets via Skia/Impeller (pixel-perfect consistency, excellent AOT performance). Neither beats native Kotlin + Compose for platform fidelity and immediate API access — cross-platform is a code-sharing trade-off.

---



## 7. What metrics should you measure continuously?

**Continuous performance monitoring** means tracking key production metrics like ANR rates, crash frequencies, rendering frame drops, and battery usage via Android Vitals.

App startup (cold/warm/hot, TTID/TTFD), frame rendering/jank, ANRs (>5s main-thread block), crash rate/crash-free sessions, memory/leaks/OOMs, battery/power, network (latency, payload, errors), app/binary size, CPU, and business/engagement (retention, DAU/MAU). Tools: Android Studio Profiler, Macro/Microbenchmark, JankStats, Perfetto, Firebase Performance, Play Console Android Vitals, LeakCanary.

---



## 8. How to avoid checking API keys into VCS

**VCS secrets management** means keeping API keys and credentials out of Git repositories by storing them in local `local.properties` and reading them via Gradle.

Store keys outside committed source (git-ignored `local.properties`, env vars, CI secrets), then expose to code via Gradle `BuildConfig`/resources or the Secrets Gradle Plugin. Ensure `.gitignore` covers `local.properties`, keystores, sensitive `google-services.json`, `.env`. CI injects secrets at build time. If a key was committed, **rotate/revoke** it — git history keeps it. Note: this keeps keys out of VCS, not secure on-device.

---



## 9. How does Kotlin Multiplatform work?

**Kotlin Multiplatform (KMP)** means a framework that compiles shared Kotlin code into native JVM, JS, or iOS libraries to reuse business logic across multiple platforms.

KMP lets you write **shared business logic once in Kotlin** and reuse it across platforms while keeping UI native. `commonMain` holds platform-agnostic code, compiled to different targets (JVM, Kotlin/Native via LLVM for iOS, JS/Wasm). The **`expect`/`actual`** mechanism declares an API in common code with platform-specific implementations. Common libs: Ktor, coroutines, kotlinx.serialisation, SQLDelight, Koin. Compose Multiplatform can also share UI.

---



## 10. How to use memory heap dumps

**Memory heap dumps** means capturing a snapshot of all active Java/Kotlin object instances in memory at a specific point in time to diagnose memory leaks.

A heap dump is a snapshot of all heap objects — sizes and reference chains — used to find leaks/bloat. Workflow: capture (Memory Profiler or `Debug.dumpHprofData`), analyze the `.hprof` by **retained size** and class counts, inspect the **reference chain/dominator tree** to see why an object is alive. Common leaks: retained `Activity`/`Context`, static view refs, long-lived listeners. Automate with **LeakCanary**.

---



## 11. How to implement Dark Theme

**Dark theme implementation** means defining DayNight styling themes and configuring alternative drawable/color resource qualifiers for night mode.

Use resource qualifiers + `DayNight`/Material 3 themes. **Views:** a `Theme.Material3.DayNight` parent, `-night` qualified resource folders, and theme attributes (`?attr/colorSurface`) instead of hardcoded colors; let users override via `AppCompatDelegate.setDefaultNightMode(...)`. **Compose:** pick `darkColorScheme()`/`lightColorScheme()` based on `isSystemInDarkTheme()`. Support Follow System, test contrast, use dynamic color on Android 12+.

---



## 12. How to secure API keys in an Android app

**API key protection** means obfuscating credentials, using native C++ wrappers, or restricting key access on the server backend since any client APK can be decompiled.

Mindset: an APK can always be decompiled, so move secrets off-device. Strongest first: (1) **proxy through your backend** so the client never sees the key; (2) short-lived tokens after auth; (3) restrict shippable keys by package + SHA-1 + API + quota; (4) **Android Keystore** for runtime secrets (note: `EncryptedSharedPreferences`/`androidx.security-crypto` was **deprecated in 2025** — the modern path is **DataStore encrypted with Google Tink**, or the Android Keystore directly); (5) NDK/obfuscation only raise the bar (not security); (6) Play Integrity to verify genuine apps.

---



## 13. What is cleartext traffic? Network Security Config

**Cleartext traffic** means unencrypted network data sent over HTTP, which is disabled by default on modern Android versions in favour of HTTPS.

Cleartext = unencrypted traffic (plain HTTP), readable/tamperable by anyone on the path. Since Android 9 (API 28) it's **disabled by default** for apps targeting API 28+. **Network Security Config** is an XML file declaratively controlling cleartext policy, trust anchors, and certificate pinning without code. Best practice: HTTPS everywhere, cleartext off, consider pinning for sensitive APIs (mind rotation).

---



## 14. Memory usage in Android

**Android process memory limits** means the restricted RAM boundaries (heaps) allocated to individual applications by the OS, which vary by device RAM size.

Each app runs in its own process with a **bounded heap**; exceeding it throws `OutOfMemoryError`. ART GCs unreachable objects (at a jank cost), so keep allocations low and lifetimes short. Respond to `onTrimMemory()` to release caches. Common leaks: retained `Context`/`Activity`, large unrecycled `Bitmap`s, unbounded caches, un-removed observers. Mitigate with `WeakReference`, lifecycle-aware components, `ViewModel`, pagination, `LruCache`. Tools: Memory Profiler, LeakCanary, `dumpsys meminfo`.

---



## 15. Explain annotation processing

**Annotation processing** means a compile-time build step that scans source code annotations and generates additional Java/Kotlin source files.

A compile-time mechanism that reads annotations and **generates code** (or validates), letting libraries (Room, Hilt, Moshi) avoid runtime reflection. A processor scans annotated elements and emits `.java`/`.kt` files compiled alongside your code. Java uses APT; Kotlin used **kapt** (generates Java stubs, slower) and now prefers **KSP** (reads Kotlin symbols directly, faster). Generated code is type-safe, debuggable, reflection-free; trade-off is build time.

---



## 16. How does the Android push notification system work?

**Android push notification system** means the architecture where a server sends payloads to Google's FCM servers, which then deliver them to a background system service on the device.

A server displays a message without polling, transported by **FCM** over a persistent, battery-efficient connection via Google Play Services. Flow: app gets a **registration token** → uploads it to your backend → backend sends to FCM (token/topic/condition) → FCM routes/queues → Play Services delivers → app shows or handles it. WorkManager/AlarmManager are for local scheduling, not push. Android 13+ needs `POST_NOTIFICATIONS`.

---



## 17. Android push notification flow using FCM

**FCM push flow** means the sequence of registering a device token, sending a JSON payload from a backend via FCM HTTP v1, and receiving it in `FirebaseMessagingService`.

Client: add the SDK + `google-services.json`, declare a `FirebaseMessagingService` (`onNewToken` to upload the token, `onMessageReceived` for data/foreground). Server: OAuth2 POST to the **HTTP v1** endpoint with a message JSON. Delivery: a **notification** message is auto-shown by the SDK when backgrounded (`onMessageReceived` only on tap); a **data** message always hits `onMessageReceived`. Target by token/topic/condition; use high priority for time-sensitive, prune `UNREGISTERED` tokens.

---



## 18. How to show a local notification at an exact time

**Exact time notifications** means scheduling precise alarms using `AlarmManager` combined with intent broadcasts to display local notifications.

Use **`AlarmManager`** with `setExact`/`setExactAndAllowWhileIdle` (fires during Doze) + a `BroadcastReceiver` that posts the notification — local, offline, not FCM. Permissions matter: `SCHEDULE_EXACT_ALARM` (Android 12+, user-grantable, **denied by default** on API 33+ — check `canScheduleExactAlarms()`); `USE_EXACT_ALARM` only for alarm/calendar apps. Reboots clear alarms (re-register on `BOOT_COMPLETED`); Android 13+ needs `POST_NOTIFICATIONS`. Prefer WorkManager when exact timing isn't required.

---



## 19. General / soft-skill HR questions

**HR interview questions** means the behavioural and scenario-based questions used by recruiters to assess communication, teamwork, and cultural alignment.

One-line guidance per item — answer honestly and concretely (use STAR):

1. Last three apps / favorite — name real projects, explain *why* one stood out.
2. Favorite language — justify with concrete strengths, not hype.
3. Dev process at last job — describe it, what you enjoyed and would improve.
4. Most challenging thing — pick a real technical challenge with impact.
5. How you stay updated — name real sources (developer.android.com, Android Weekly, Kotlin Weekly, conferences).
6. Why Senior/Junior — define seniority by ownership/impact/mentoring, not years.
7. Documentation — show you value it (READMEs, ADRs, code comments).
8. Proudest achievement — quantify the outcome.
9. Strengths/areas to improve — be self-aware and honest.
10. Project management tools — name what you've used (Jira/Trello).
11. Agile/Scrum/Sprints — show practical familiarity.
12. What you enjoy — be genuine.
13. Team type you like — align with collaboration.
14. GitFlow — branching model with long-lived feature + primary branches; say whether you follow it.
15. Trunk-based development — small frequent merges to `main` via short-lived branches.
16. TDD — red-green-refactor (failing test → pass → refactor).
17. Backend/core vs UI preference — be honest, show flexibility.
18. "Teach me something technical" — simplify clearly for a non-technical listener.

---



## 20. Preparation Tips

**Interview preparation strategies** means reviewing fundamentals, mock-interviewing out loud, coding on whiteboards, and preparing project architecture deep dives.

**Dos:** sleep well, research the company/role/interviewer, practice problems, do mock interviews, arrive early, manage nerves, be honest, **communicate your thought process**, ask clarifying questions, follow up with a thank-you. **Don'ts:** don't memorize canned answers, don't be late/unprepared, don't be arrogant or oversell, don't get distracted by a prior question. **STAR:** Situation, Task, Action, Result (quantify outcomes). Have smart end-of-interview questions ready (culture, values, success traits).

---



## 21. Mock Interview Questions

**Mock interview practice** means rehearsing typical question sets covering OOP, Kotlin, Jetpack, architecture, and DSA to build confidence.

One-line guidance per item (cross-references in full guide):

1. Most challenging app + contributions — STAR; cover architecture/integration/security.
2. Material Design adherence — Material 3, design reviews, UI tests.
3. OS-version compatibility — AndroidX, sensible min/target SDK, `Build.VERSION` checks, broad testing.
4. Tricky bug walkthrough — reproduce → Logcat/ADB → isolate → fix → regression test.
5. Performance optimisation — Profiler, lazy loading, efficient data structures.
6. Integrating a third-party API — Retrofit/OkHttp, secure transport, error handling/retries.
7. Git/SVN experience — branching, merging, PRs, GitFlow vs trunk-based.
8. Automated testing — JUnit, Mockito, Espresso, UI Automator, CI/CD.
9. Multiple device sizes — ConstraintLayout, qualifiers, vectors, responsive Compose.
10. Memory management — Memory Profiler, WeakReferences, lifecycle-aware Context, LeakCanary.
11. Secure storage/transmission — Android Keystore (or DataStore + Tink; `EncryptedSharedPreferences` is deprecated as of 2025) + HTTPS/TLS, OWASP MASVS.
12. Troubleshooting a live issue — STAR; e.g. ANR found via crash reports, then fixed.
13. Staying up to date — blogs, conferences, forums, open source.
14. Intuitive UI design — usability testing, feedback, consistent navigation.
15. Reduce battery consumption — batch network, WorkManager, avoid wakelocks, profile power.
16. Localization/i18n — string resources per locale, RTL, layout expansion, `Locale`.
17. Accessibility — content descriptions, contrast, TalkBack, touch target sizing.
18. Deployment — Play Console, staged rollouts, monitor vitals, gather feedback.

