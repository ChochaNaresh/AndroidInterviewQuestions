# Android Build Tools, Gradle & Tooling — Short Answers (Quick Revision)

> Condensed answers for rapid review before an interview. For full explanations and code see `android_tools_gradle_interview.md`.

---

### Gradle & Build System

## 1. What is Gradle?

The build automation system for Android: compiles code, processes resources, runs annotation processors, merges manifests, packages APK/AAB, runs tests, manages dependencies. The **AGP (Android Gradle Plugin)** teaches Gradle everything Android-specific. Key concepts: project, task (DAG - Directed Acyclic Graph), plugin, and the three phases (initialization, configuration, execution). It's incremental, cacheable, and runs as a daemon.

---

## 2. The Gradle build system in Android

Maps config to a packaged artifact: compilers → DEX/resources; AGP merges manifests/resources and generates `R`/`BuildConfig`; R8 shrinks/optimizes/obfuscates/dexes (release); packager zips, signs, aligns. Outputs: **APK (Android Package Kit)** (installable/sideload) and **AAB (Android App Bundle)** (Play publishing format that generates per-device split APKs).

---

## 3. Gradle-related files in an Android project

`settings.gradle.kts` (modules + repos), root `build.gradle.kts` (shared plugins), module `build.gradle.kts` (`android {}`, `dependencies {}`), `gradle.properties` (JVM args/flags), `gradle/libs.versions.toml` (version catalog), `gradle-wrapper.properties` (Gradle version), `gradlew`, `local.properties` (SDK/secrets, not committed), `proguard-rules.pro`.

---

## 4. What is the Kotlin DSL for Gradle?

Writing build scripts in Kotlin (`build.gradle.kts`) instead of Groovy — the default for new projects. Advantages: type safety/compile-time checks, full IDE autocomplete and refactoring, discoverable APIs. Trade-offs: slightly slower first configuration/sync and more verbosity.

---

## 5. implementation vs api in Gradle

Both add a dependency but differ in **transitivity**. `implementation` keeps the dependency internal — consumers don't see it, and changes don't force their recompilation (faster builds). `api` exposes it transitively on consumers' compile classpath. Prefer `implementation`; use `api` only when a type appears in your module's public signatures.

---

## 6. How do you create a custom Gradle task?

Use `tasks.register("name") { doLast { ... } }` for ad-hoc work, or define a typed `DefaultTask` class with `@Input`/`@OutputFile` and a `@TaskAction` for incremental/cacheable logic. Prefer `register` (lazy) over `create` (eager) so the task configures only when needed.

---

## 7. Build variants, build types, and product flavors

A **build variant** = build type × product flavor. **Build types** describe *how* you build (`debug`, `release` — shrinking/signing). **Product flavors** describe *different versions* (`free`/`paid`, `dev`/`prod`), grouped into dimensions. Produces variants like `freeRelease`, with variant-specific source sets and dependency configs.

---

## 8. Multiple APKs for Android apps

Splitting the binary so each device downloads only what it needs. Two approaches: **AAB (recommended)** — upload one bundle, Play's Dynamic Delivery generates per-device split APKs; **APK splits (`splits {}`)** — manual/legacy, useful outside Play (assign distinct `versionCode`s per ABI). Prefer AAB.

---

### Code Shrinking & Obfuscation

## 9. What is ProGuard used for?

Processes release bytecode to do: **shrinking** (remove dead code), **optimization** (rewrite/inline), **obfuscation** (rename to `a`/`b`/`c`), and preverification. Net: smaller APK, fewer methods, harder to reverse-engineer. Note: **R8 has replaced ProGuard** as the default but reads the same `.pro` rules.

---

## 10. What is the proguard-rules.pro file used for?

Holds **keep rules** telling R8/ProGuard what not to remove or rename — needed because static analysis can't see code accessed via reflection, JNI, serialization, or manifest/XML (e.g. Gson/Moshi models, enums, native methods). Libraries ship consumer rules automatically. Missing rules cause release-only crashes or null JSON fields.

---

## 11. ProGuard vs R8

R8 is the **default** in modern AGP, enabled by `isMinifyEnabled = true`. It merges shrinking + optimization + obfuscation + dexing into a **single faster pass** (ProGuard was a separate pass before D8), is more aggressive (inlining, class merging, Kotlin-aware), reads the same `.pro` rules (drop-in), and has compat and full modes.

---

## 12. Obfuscation vs minification (and shrinking)

- **Shrinking** — *removing* unused code/resources.
- **Obfuscation** — *renaming* survivors to short names (produces `mapping.txt`).
- **Optimization** — rewriting bytecode (inlining).

"Minification" (`isMinifyEnabled`) is R8's umbrella for shrink + obfuscate + optimize (not JS-style whitespace). Obfuscation is not security — keep secrets server-side.

---

## 13. Things to take care of while using ProGuard/R8

Keep reflection/serialization models, JNI classes/methods, and manifest/XML-referenced components; keep and upload `mapping.txt`; **test the minified release build** (R8 issues only appear there); rely on library consumer rules but verify; don't over-keep (`-keep ** {*;}` defeats shrinking); use `-dontwarn` sparingly; full mode may need extra rules (generics/`TypeToken`).

---

## 14. Desugaring in Android

Lets you use newer Java language features/APIs on older Android by rewriting bytecode. Two layers: **language desugaring** (automatic — lambdas, default methods, try-with-resources) and **core library desugaring** (opt-in — backports `java.time`, `java.util.stream`, `Optional` via `coreLibraryDesugaring`). Trade-off: small APK/build cost for modern APIs at low `minSdk`.

---

### Annotation Processing

## 15. annotationProcessor vs kapt vs KSP in Gradle

All feed annotation-based code generation. `annotationProcessor` = Java-only (JSR-269). `kapt` (Kotlin Annotation Processing Tool) = generates Java **stubs** so Java processors run on Kotlin — slow, on the deprecation path. **KSP** (Kotlin Symbol Processing) = reads Kotlin symbols directly (no stubs), ~2x faster, recommended (KSP2 is stable and recommended, but still opt-in — not the automatic default). Migrate to KSP where supported; keep kapt only for processors lacking KSP.

---

### APK Size & Build Speed

## 16. APK size reduction

- Ship an **AAB** for per-device splits (biggest win).
- Enable R8 (`isMinifyEnabled`) + `isShrinkResources`.
- Use WebP/vector drawables; remove unused locales/densities (`resConfigs`).
- Audit/drop heavy dependencies; move large assets off-device.
- Analyze with the **APK Analyzer**.

---

## 17. How can you speed up the Gradle build?

Enable parallel builds, build cache (`org.gradle.caching`, remote on CI), and configuration cache. Modularize and prefer `implementation` over `api` to shrink recompile scope. Migrate kapt → KSP, keep AGP/Gradle/Kotlin updated, avoid dynamic versions, register tasks lazily, and profile with `--scan`/`--profile`.

---

### Tooling

## 18. What is ADB?

Android Debug Bridge — a CLI that communicates with a device/emulator via a client, a server, and the on-device daemon (`adbd`). Common: `adb devices`, `install`, `logcat`, `shell`, `push`/`pull`, `am start`, `pm clear`, wireless debugging. The backbone of testing/debugging/profiling/CI.

---

## 19. What is Lint and what is it used for?

A **static analysis** tool that scans source/resources/manifest/Gradle without running the app, flagging correctness, security, performance, accessibility, i18n, and API-compatibility issues. Configure via the `lint {}` block (`abortOnError`, baseline, disable rules); suppress with `@SuppressLint`/`tools:ignore`; supports baselines and custom rules; run in CI.

---

## 20. What is StrictMode?

A debug tool detecting main-thread violations — disk/network I/O on the UI thread and resource leaks (unclosed Closeables, leaked Activities/cursors). Two policies: **thread policy** and **VM policy**. Enable only in debug builds in `Application.onCreate()` with `penaltyLog()` (or `penaltyDeath()`); never ship it enabled.

---

## 21. How to use the Android Studio Memory Profiler?

Run with profiling, open Profiler > Memory, watch the live memory graph, force GC and **capture a heap dump** to inspect live objects/retained size/reference chains, and **record allocations** to spot churn. Look for Activities/Fragments retained after destruction and static `Context` refs. LeakCanary is easier for day-to-day leak detection.

---

## 22. How to measure method execution time in Android?

Quick: `System.nanoTime()` deltas, Kotlin's `measureTimeMillis`/`measureNanoTime`/`measureTime`. Timeline: the `Trace` API (`Trace.beginSection`/`endSection`). Rigorous: Android Studio CPU Profiler/System Trace and **Jetpack Microbenchmark/Macrobenchmark** (handle warmup, iterations, stable medians). Don't trust a single reading on a JIT'd runtime.

---

## 23. Can you access your SQLite database for debugging?

Yes: **Database Inspector** (built into Android Studio, API 26+, live SQL on a running debuggable app — the modern default); **Android-Debug-Database** library (browser UI, good for physical devices + SharedPreferences); or **ADB pull** the `.db` file and open it in any SQLite client.

---

### CI/CD & Release

## 24. CI/CD pipeline for Android

On each push/PR: checkout + set up JDK/SDK, cache Gradle, static checks (`lint`/`detekt`/`ktlint`), unit tests, instrumented tests (emulator/Firebase Test Lab), build (`bundleRelease` with signing from secrets), deploy (Gradle Play Publisher/fastlane; Firebase App Distribution for testers). Keep secrets in CI, gate merges on green checks.

---

## 25. Android app release checklist for production launch

Bump `versionCode`/`versionName`; release build with minify + shrink resources, debuggable off; sign with upload key + Play App Signing; publish an **AAB**; verify the minified build and keep `mapping.txt`; remove unused permissions; point to production config; meet compliance (privacy, Data Safety, target API, 16 KB); wire crash/analytics; QA across SDKs/form factors/upgrade path; staged rollout with monitoring.

---

## 26. Git

Distributed VCS for tracking changes and collaboration. Essentials: branching, `add`/`commit`, `pull --rebase`, `merge` vs `rebase` (merge keeps history; rebase is linear but rewrites — don't rebase shared branches), `revert` (safe undo) vs `reset --hard` (destructive), `stash`, `cherry-pick`. Know branching strategies (trunk-based, Git Flow, GitHub Flow), PR review, conflict resolution, tags, `.gitignore`.

---

## 27. Firebase

Google's mobile/web backend suite integrated via the Firebase BoM. Common modules: Authentication, Firestore/Realtime DB, Cloud Storage, Cloud Messaging (FCM), Crashlytics, Analytics, Remote Config, App Distribution, Test Lab.

---

### Miscellaneous

## 28. How to change parameters in an app without an app update?

Use **Firebase Remote Config** (or any server-driven config / feature-flag service): define key/values in the console, fetch and activate at runtime to override bundled defaults — enabling feature flags/kill switches, gradual rollout, threshold tuning, and A/B tests without shipping a build. For constantly changing data, use your own backend.

---

## 29. What is Write-Ahead Logging (WAL)?

A SQLite journaling mode (Room default on supported APIs) that appends changes to a separate `-wal` file first, updating the main DB later at a **checkpoint**. Benefits: concurrency (readers + one writer simultaneously, fewer "database is locked"), better performance (sequential appends), and crash safety. Trade-offs: extra files, checkpointing, one writer at a time.

---

## 30. 16 KB page size for Android apps

Newer 64-bit devices/Android 15+ support a 16 KB memory page size (faster launches/boot, lower overhead). **Kotlin/Java-only apps are already compatible.** Apps with **native code (`.so`)** must be rebuilt 16 KB-aligned (recent NDK) or they crash on 16 KB devices. Required for apps targeting Android 15+ on Play (began Nov 1, 2025, with a 2026 compliance window).

---

### Modern Gradle

## 31. Gradle Version Catalogs (libs.versions.toml)

A central, type-safe registry of dependency coordinates/versions in `gradle/libs.versions.toml` (`[versions]`/`[libraries]`/`[bundles]`/`[plugins]`), accessed via a generated `libs.*` accessor with IDE autocomplete. Default in new projects. Benefits: single source of truth across modules, no version drift, `bundles` grouping, shareable; editing it doesn't invalidate config like `buildSrc` does.

---

## 32. Build cache vs configuration cache

Two distinct caches. **Build cache** caches **task outputs** (execution phase), is local **and remote** (shareable across CI/machines), enabled by `org.gradle.caching=true`. **Configuration cache** caches the **configuration-phase result** (the task graph), skipping re-running build scripts, enabled by `org.gradle.configuration-cache=true`. Nail the phase each targets and that the build cache can be remote.
