# Android Build Tools, Gradle & Tooling — Interview Prep Guide

A comprehensive, code-first reference for Android build and tooling interview questions. Covers the Gradle build system (with Kotlin DSL), code shrinking with R8/ProGuard, annotation processing (annotationProcessor/kapt/KSP), APK size and build-speed optimization, developer tooling (ADB, Lint, StrictMode, Profiler, on-device DB debugging), CI/CD and release, plus assorted topics (WAL, Remote Config, 16 KB page size).

Accurate as of 2026: **R8 is the default code shrinker**, **KSP (KSP2) is the recommended annotation processor and kapt is on the deprecation path**, and Google Play's **16 KB page size requirement** is in force.

---

## Table of Contents

### Gradle & Build System
1. [What is Gradle?](#1-what-is-gradle)
2. [The Gradle build system in Android](#2-the-gradle-build-system-in-android)
3. [Gradle-related files in an Android project](#3-gradle-related-files-in-an-android-project)
4. [What is the Kotlin DSL for Gradle?](#4-what-is-the-kotlin-dsl-for-gradle)
5. [implementation vs api in Gradle](#5-implementation-vs-api-in-gradle)
6. [How do you create a custom Gradle task?](#6-how-do-you-create-a-custom-gradle-task)
7. [Build variants, build types, and product flavors](#7-build-variants-build-types-and-product-flavors)
8. [Multiple APKs for Android apps](#8-multiple-apks-for-android-apps)

### Code Shrinking & Obfuscation
9. [What is ProGuard used for?](#9-what-is-proguard-used-for)
10. [What is the proguard-rules.pro file used for?](#10-what-is-the-proguard-rulespro-file-used-for)
11. [ProGuard vs R8](#11-proguard-vs-r8)
12. [Obfuscation vs minification (and shrinking)](#12-obfuscation-vs-minification-and-shrinking)
13. [Things to take care of while using ProGuard/R8](#13-things-to-take-care-of-while-using-proguardr8)
14. [Desugaring in Android](#14-desugaring-in-android)

### Annotation Processing
15. [annotationProcessor vs kapt vs KSP in Gradle](#15-annotationprocessor-vs-kapt-vs-ksp-in-gradle)

### APK Size & Build Speed
16. [APK size reduction](#16-apk-size-reduction)
17. [How can you speed up the Gradle build?](#17-how-can-you-speed-up-the-gradle-build)

### Tooling
18. [What is ADB?](#18-what-is-adb)
19. [What is Lint and what is it used for?](#19-what-is-lint-and-what-is-it-used-for)
20. [What is StrictMode?](#20-what-is-strictmode)
21. [How to use the Android Studio Memory Profiler?](#21-how-to-use-the-android-studio-memory-profiler)
22. [How to measure method execution time in Android?](#22-how-to-measure-method-execution-time-in-android)
23. [Can you access your SQLite database for debugging?](#23-can-you-access-your-sqlite-database-for-debugging)

### CI/CD & Release
24. [CI/CD pipeline for Android](#24-cicd-pipeline-for-android)
25. [Android app release checklist for production launch](#25-android-app-release-checklist-for-production-launch)
26. [Git](#26-git)
27. [Firebase](#27-firebase)

### Miscellaneous
28. [How to change parameters in an app without an app update?](#28-how-to-change-parameters-in-an-app-without-an-app-update)
29. [What is Write-Ahead Logging (WAL)?](#29-what-is-write-ahead-logging-wal)
30. [16 KB page size for Android apps](#30-16-kb-page-size-for-android-apps)

### Modern Gradle
31. [Gradle Version Catalogs (libs.versions.toml)](#31-gradle-version-catalogs-libsversionstoml)
32. [Build cache vs configuration cache](#32-build-cache-vs-configuration-cache)

---

## 1. What is Gradle?

Gradle is the **build automation system** used by Android. It compiles your source code, processes resources, runs annotation processors, merges manifests, packages everything into an APK or AAB, runs tests, and manages dependencies. The **Android Gradle Plugin (AGP)** sits on top of Gradle and teaches it everything Android-specific (variants, manifests, resources, R8, etc.).

Key concepts:

- **Project** — a buildable unit. An Android build is a *multi-project* build: the root project plus modules (`:app`, `:core`, `:feature:login`, …).
- **Task** — a single unit of work (`compileDebugKotlin`, `assembleRelease`, `lint`). Tasks form a directed acyclic graph (DAG) based on dependencies.
- **Plugin** — bundles tasks and conventions (`com.android.application`, `org.jetbrains.kotlin.android`).
- **Build phases** — *Initialization* (which projects participate), *Configuration* (run build scripts, build the task graph), *Execution* (run the requested tasks).

```bash
./gradlew assembleDebug      # build the debug APK
./gradlew :app:assembleRelease
./gradlew test               # run unit tests
./gradlew tasks              # list available tasks
./gradlew :app:dependencies  # print the dependency tree
```

Gradle is incremental (only re-runs work whose inputs changed), supports a build cache, and runs as a long-lived daemon for speed.

**📚 Reference:** <https://developer.android.com/build>

---

## 2. The Gradle build system in Android

The Android build maps high-level config to a fully packaged artifact:

1. **Compilers** turn Kotlin/Java source and resources into `.class`/DEX and compiled resources.
2. **AGP** merges all manifests, processes and merges resources, runs annotation processors/KSP, and generates `R` and `BuildConfig`.
3. **R8** shrinks, optimizes, obfuscates, and dexes (in release builds).
4. The **packager** zips DEX, resources, native libs, and assets into an APK/AAB, then signs and aligns it.

Output formats:

- **APK (Android Package Kit)** — installable package, used for sideloading and testing.
- **AAB (Android App Bundle)** — the publishing format for Google Play. You upload the AAB; Play generates optimized, device-specific APKs (split by ABI, density, language) via the bundletool mechanism.

```bash
./gradlew bundleRelease   # produces an .aab for Play upload
./gradlew assembleRelease # produces signed APK(s)
```

The build is driven entirely by the Gradle scripts described next.

**📚 Reference:** <https://developer.android.com/build/gradle-build-overview>

---

## 3. Gradle-related files in an Android project

| File | Purpose |
|------|---------|
| `settings.gradle.kts` | Declares which modules are included and configures dependency/plugin repositories (`pluginManagement`, `dependencyResolutionManagement`). |
| Root `build.gradle.kts` | Top-level build script. Declares plugins (often with `apply false`) shared across modules. |
| Module `build.gradle.kts` (`:app`) | Per-module config: applied plugins, `android { }` block, `dependencies { }`. |
| `gradle.properties` | JVM args, AndroidX flags, feature toggles (e.g., `org.gradle.jvmargs`, `org.gradle.caching`). |
| `gradle/libs.versions.toml` | **Version catalog** — centralizes dependency and plugin versions. |
| `gradle/wrapper/gradle-wrapper.properties` | Pins the Gradle version used by `./gradlew`. |
| `gradlew` / `gradlew.bat` | Wrapper scripts — run the build without a locally installed Gradle. |
| `local.properties` | Machine-specific (SDK path, secrets). **Not** committed to VCS. |
| `proguard-rules.pro` | R8/ProGuard keep rules (see Q10). |

```toml
# gradle/libs.versions.toml
[versions]
agp = "8.7.0"
kotlin = "2.0.21"
[libraries]
retrofit = { module = "com.squareup.retrofit2:retrofit", version = "2.11.0" }
[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
```

```kotlin
// :app/build.gradle.kts
plugins {
    alias(libs.plugins.android.application)
}
dependencies {
    implementation(libs.retrofit)
}
```

**📚 Reference:** <https://developer.android.com/build#build-files>

---

## 4. What is the Kotlin DSL for Gradle?

The **Kotlin DSL** lets you write build scripts in Kotlin (`build.gradle.kts`) instead of Groovy (`build.gradle`). It is the default for new Android projects.

Advantages:

- **Type safety & compile-time checks** — typos fail at sync, not at task execution.
- **IDE support** — full autocompletion, navigation, and refactoring because the script is statically typed.
- **Readable, discoverable APIs** — accessors are generated for plugins and extensions.

Trade-offs:

- Slightly **slower first configuration / IDE sync** than Groovy because scripts are compiled, though the build cache mitigates this.
- More verbose in places (named arguments, explicit strings).

```kotlin
// Groovy:  applicationId "com.example.app"
// Kotlin:  applicationId = "com.example.app"

android {
    namespace = "com.example.app"
    compileSdk = 35
    defaultConfig {
        applicationId = "com.example.app"
        minSdk = 24
        targetSdk = 35
        versionCode = 1
        versionName = "1.0"
    }
}
```

**📚 Reference:** <https://www.linkedin.com/posts/amit-shekhar-iitbhu_softwareengineer-androiddev-android-activity-7282624364695404546-BvbS> · <https://docs.gradle.org/current/userguide/kotlin_dsl.html>

---

## 5. implementation vs api in Gradle

Both add a dependency to a module, but they differ in **transitivity** — whether downstream modules can also see that dependency.

- **`implementation`** — the dependency is an *internal* detail. Consumers of your module do **not** get it on their compile classpath. Changing this dependency does not force recompilation of dependent modules.
- **`api`** — the dependency is **exposed** as part of your module's public API. Consumers transitively get it on their compile classpath.

Prefer `implementation` by default: it keeps the dependency graph clean and dramatically improves incremental build speed, because a change inside an `implementation` dependency only recompiles the one module that declares it. Use `api` only when a type from that dependency appears in your module's **public** signatures (return types, public params) that consumers must reference.

```kotlin
dependencies {
    implementation(libs.okhttp)            // hidden from consumers
    api(libs.rxjava)                        // leaked to consumers
    compileOnly(libs.annotations)           // needed at compile time only
    runtimeOnly(libs.slf4j.simple)          // needed at runtime only (e.g., logging backend)
    testImplementation(libs.junit)          // test source set
    debugImplementation(libs.leakcanary)    // debug variant only
    ksp(libs.room.compiler)                 // annotation processor (see Q15)
}
```

**📚 Reference:** <https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineering-tech-activity-7287325379441082368-_P9m>

---

## 6. How do you create a custom Gradle task?

A task is a named unit of work. You can register tasks ad hoc or, for reusable logic, define a typed task class.

```kotlin
// Simple ad-hoc task (lazy registration — preferred over create)
tasks.register("printVersion") {
    group = "custom"
    description = "Prints the app version name"
    doLast {
        println("Version: ${android.defaultConfig.versionName}")
    }
}
```

```kotlin
// Typed task with inputs/outputs for incremental builds + caching
abstract class GenerateBuildInfoTask : DefaultTask() {
    @get:Input abstract val gitSha: Property<String>
    @get:OutputFile abstract val outputFile: RegularFileProperty

    @TaskAction
    fun generate() {
        outputFile.get().asFile.writeText("git=${gitSha.get()}")
    }
}

tasks.register<GenerateBuildInfoTask>("generateBuildInfo") {
    gitSha.set("abc123")
    outputFile.set(layout.buildDirectory.file("buildinfo.txt"))
}
```

```bash
./gradlew printVersion
./gradlew generateBuildInfo
```

Declaring `@Input`/`@OutputFile` lets Gradle skip the task when inputs are unchanged and cache its outputs. Use `register` (lazy) rather than `create` (eager) so the task is only configured when actually needed.

**📚 Reference:** <https://www.linkedin.com/posts/amit-shekhar-iitbhu_softwareengineer-androiddev-android-activity-7282991795133685760-uUxE/>

---

## 7. Build variants, build types, and product flavors

A **build variant** is the cross-product of a **build type** and a **product flavor**.

- **Build types** describe *how* you build: usually `debug` (debuggable, no shrinking) and `release` (shrunk, signed). You define R8/signing config here.
- **Product flavors** describe *different versions* of the app: e.g., `free` vs `paid`, or `dev`/`staging`/`prod`. Flavors live in **dimensions**.

```kotlin
android {
    buildTypes {
        debug {
            applicationIdSuffix = ".debug"
            isMinifyEnabled = false
        }
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }

    flavorDimensions += "tier"
    productFlavors {
        create("free")  { dimension = "tier"; applicationIdSuffix = ".free" }
        create("paid")  { dimension = "tier" }
    }
}
```

This produces variants like `freeDebug`, `freeRelease`, `paidDebug`, `paidRelease`. Variant-specific source sets (`src/free/`, `src/release/`) and dependency configs (`freeImplementation(...)`) let you swap code, resources, and libraries per variant.

```bash
./gradlew assembleFreeRelease
```

**📚 Reference:** <https://www.linkedin.com/posts/amit-shekhar-iitbhu_outcomeschool-softwareengineer-tech-activity-7298197111030824962-6vW0> · <https://developer.android.com/build/build-variants>

---

## 8. Multiple APKs for Android apps

Historically apps shipped a single "fat" APK containing every ABI, density, and language, which bloated download size. **Multiple APKs** address this by splitting the binary so each device downloads only what it needs.

Two approaches:

1. **App Bundles (AAB) — recommended.** Upload one AAB to Play; Play's *Dynamic Delivery* generates and serves per-device split APKs automatically (by ABI, screen density, language). No manual config of multiple APKs needed.
2. **APK splits (`splits { }`) — manual/legacy.** Useful when you distribute outside Play.

```kotlin
android {
    splits {
        abi {
            isEnable = true
            reset()
            include("armeabi-v7a", "arm64-v8a", "x86_64")
            isUniversalApk = false
        }
        density {
            isEnable = true
            include("mdpi", "hdpi", "xhdpi", "xxhdpi")
        }
    }
}
```

When generating multiple APKs you must assign distinct `versionCode`s per ABI so Play serves the right one. With AAB, this complexity disappears. Prefer AAB for Play distribution.

**📚 Reference:** <https://developer.android.com/build/configure-apk-splits>

---

## 9. What is ProGuard used for?

ProGuard is a tool that processes the compiled bytecode of a release build to make the app smaller and harder to reverse-engineer. It does four things:

1. **Shrinking** — removes unused classes, methods, and fields (dead-code elimination / tree-shaking).
2. **Optimization** — rewrites/inlines bytecode for efficiency.
3. **Obfuscation** — renames classes/methods/fields to short, meaningless names (`a`, `b`, `c`).
4. **Preverification** — adds verification metadata (legacy JVM concern).

Net effects: **smaller APK**, fewer methods (helps the 64K method limit), and code that is harder to read if decompiled.

Important: as of modern Android, **R8 has replaced ProGuard as the default shrinker** (see Q11). R8 reads the *same* ProGuard configuration files, so the ProGuard *rules* knowledge still applies even though the ProGuard engine itself is no longer used.

```kotlin
android {
    buildTypes {
        release { isMinifyEnabled = true }   // enables R8 (ProGuard-compatible)
    }
}
```

**📚 Reference:** <https://www.linkedin.com/posts/amit-shekhar-iitbhu_outcomeschool-activity-7218840466283220993-afVr>

---

## 10. What is the proguard-rules.pro file used for?

`proguard-rules.pro` holds your project's **keep rules** — instructions that tell R8/ProGuard what it must **not** remove or rename. You need it because static analysis cannot see code accessed only via **reflection, JNI, serialization, or from the manifest/XML**.

Common rules:

```text
# Keep a class and all its members (e.g., a model parsed by reflection)
-keep class com.example.model.** { *; }

# Keep classes annotated with a given annotation
-keep @com.example.KeepMe class * { *; }

# Keep fields used by Gson/Moshi (matched by name via reflection)
-keepclassmembers class com.example.dto.** { <fields>; }

# Keep native method names (JNI)
-keepclasseswithmembernames class * { native <methods>; }

# Keep enums' values()/valueOf() (used reflectively)
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}

# Don't warn about an optional dependency that isn't present
-dontwarn org.bouncycastle.**
```

Libraries usually ship their own *consumer* rules (`consumer-rules.pro`) that get merged automatically, so you only add rules for *your* reflection-sensitive code or libraries that lack them. Symptoms of a missing rule: `ClassNotFoundException`/`NoSuchMethodError` only in release builds, or JSON fields silently becoming null.

**📚 Reference:** <https://www.linkedin.com/posts/amit-shekhar-iitbhu_outcomeschool-softwareengineer-tech-activity-7300763237942181888-P_jL>

---

## 11. ProGuard vs R8

| Aspect | ProGuard | R8 |
|--------|----------|----|
| Role | Legacy shrinker/obfuscator | **Default** shrinker in modern AGP |
| Pipeline | Separate shrink → then dexing (D8) | **Single step**: shrink + optimize + obfuscate + dex |
| Config | `.pro` keep rules | **Same** `.pro` keep rules (drop-in compatible) |
| Speed | Slower (extra pass) | Faster (one pass) |
| Optimizations | Good | More aggressive (inlining, class merging, enum-to-int, kotlin-aware) |
| Maintenance | Third-party | Maintained by Google, integrated with AGP |

R8 is Google's replacement for ProGuard and is enabled automatically whenever `isMinifyEnabled = true`. Because it consumes ProGuard-format rules, migrating is essentially free — you keep your `proguard-rules.pro`. R8 has two modes: **compat mode** (default, mirrors ProGuard behavior) and **full mode** (`android.enableR8.fullMode=true`), which optimizes more aggressively and may require additional keep rules.

```properties
# gradle.properties
android.enableR8.fullMode=true
```

Bottom line for interviews: *"R8 is the default; it merges ProGuard's shrinking/obfuscation with D8's dexing into a single, faster pass and reads the same rules."*

**📚 Reference:** <https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineer-tech-activity-7370862013998227456-tAlD> · <https://developer.android.com/build/shrink-code>

---

## 12. Obfuscation vs minification (and shrinking)

These overlapping terms are often confused:

- **Shrinking (code shrinking / tree-shaking)** — *removing* unused classes, methods, and fields. Reduces size and method count.
- **Resource shrinking** — removing unused resources (`isShrinkResources = true`); must be paired with code shrinking.
- **Obfuscation** — *renaming* the surviving classes/methods/fields to short, meaningless names. Does **not** remove code; it makes decompiled output hard to understand and slightly smaller. Produces a `mapping.txt` you must keep to de-obfuscate crash stack traces.
- **Optimization** — rewriting bytecode (inlining, dead-branch removal) for speed/size.

"**Minification**" in the Android/`isMinifyEnabled` sense is an umbrella for shrinking + obfuscation + optimization performed by R8 — note it is *not* the same as JavaScript-style whitespace minification.

Obfuscation is **not** a security boundary (it deters casual reverse engineering but does not encrypt logic). Keep secrets server-side.

```kotlin
buildTypes {
    release {
        isMinifyEnabled = true      // shrink + obfuscate + optimize (R8)
        isShrinkResources = true    // remove unused resources
    }
}
```

```bash
# Retrace an obfuscated stack trace
retrace mapping.txt obfuscated_stacktrace.txt
```

**📚 Reference:** <https://developer.android.com/build/shrink-code>

---

## 13. Things to take care of while using ProGuard/R8

- **Reflection / serialization** — Gson, Moshi, Retrofit models, and anything loaded by name must be kept (see Q10). Otherwise fields go null or you get runtime crashes.
- **Native / JNI** — keep classes and method names referenced from C/C++ (`native <methods>`).
- **Components in the manifest/XML** — Activities, Services, custom Views referenced from XML are usually kept by default consumer rules, but verify.
- **Keep the `mapping.txt`** for every release build and upload it to Play / your crash reporter, or stack traces are unreadable.
- **Test the release build, not just debug** — R8 issues only appear when `isMinifyEnabled = true`. Run instrumented/smoke tests against the minified build.
- **Library consumer rules** — rely on them, but watch for libraries that don't ship any.
- **Don't over-keep** — broad `-keep class ** { *; }` rules defeat shrinking and bloat the APK. Be specific.
- **`-dontwarn` carefully** — silence only known-absent optional deps, not real problems.
- **R8 full mode** may need extra rules (e.g., for generics/`TypeToken`).

```text
-keepattributes Signature, *Annotation*, InnerClasses   # needed by Gson generics, etc.
```

---

## 14. Desugaring in Android

**Desugaring** lets you use newer Java language features and APIs on older Android versions whose runtime doesn't support them. The toolchain rewrites ("de-sugars") the new constructs into bytecode the lower API levels can run.

Two layers:

1. **Language desugaring** (automatic in D8/R8) — lambdas, default/static interface methods, try-with-resources, etc., work down to old `minSdk`.
2. **Core library desugaring** (opt-in) — backports `java.time` (`LocalDate`, `Duration`), `java.util.stream`, `java.util.Optional`, and more to API levels that lack them, by bundling a desugared library.

```kotlin
android {
    compileOptions {
        isCoreLibraryDesugaringEnabled = true
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
}
dependencies {
    coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.1.2")
}
```

Trade-offs: a small APK size increase and a minor build-time cost, in exchange for being able to write modern Java APIs while supporting a low `minSdk`. (Kotlin stdlib generally doesn't need this, but `java.time` usage does.)

**📚 Reference:** <https://www.linkedin.com/posts/amit-shekhar-iitbhu_outcomeschool-softwareengineer-tech-activity-7352908041224118272-JkCu> · <https://developer.android.com/studio/write/java8-support>

---

## 15. annotationProcessor vs kapt vs KSP in Gradle

All three feed **annotation-based code generation** (Room, Dagger/Hilt, Moshi codegen, etc.), but they differ by language and mechanism.

| Tool | For | How it works | Status (2026) |
|------|-----|--------------|---------------|
| `annotationProcessor` | **Java** sources | Standard JSR-269 `javac` processing | Use for pure-Java processors |
| `kapt` (Kotlin Annotation Processing Tool) | Kotlin + Java APs | Generates Java **stubs** from Kotlin so a *Java* processor can run | **Deprecated path** — being superseded by KSP |
| **KSP** (Kotlin Symbol Processing) | Kotlin (and Java) | Reads Kotlin source directly via a lightweight symbol API — **no stub generation** | **Recommended.** KSP2 is stable (KSP 2.0.0, early 2025) and the path forward |

Why KSP wins: kapt must generate Java stubs for the entire module before processing, which is slow. KSP analyzes Kotlin symbols directly and runs **up to ~2x faster**, with better Kotlin fidelity. KSP1 (a K1 compiler plugin) is being deprecated alongside K1; **KSP2** is the path forward. Migrate processors that support KSP (Room, Moshi, Hilt, etc.) and only keep kapt for processors that have no KSP version yet.

```kotlin
plugins {
    alias(libs.plugins.ksp)            // com.google.devtools.ksp
    // alias(libs.plugins.kotlin.kapt) // only if a processor lacks KSP support
}
dependencies {
    implementation(libs.room.runtime)
    ksp(libs.room.compiler)            // KSP instead of kapt(...)

    // Java-only processor example:
    // annotationProcessor(libs.someJavaProcessor)

    // Legacy kapt example (avoid for new code):
    // kapt(libs.someProcessorWithoutKsp)
}
```

**📚 Reference:** <https://www.linkedin.com/posts/amit-shekhar-iitbhu_outcomeschool-softwareengineer-tech-activity-7296566489803866112-DnT8> · <https://kotlinlang.org/docs/ksp-overview.html> · <https://github.com/google/ksp>

---

## 16. APK size reduction

Concrete levers, roughly in order of impact:

- **Ship an AAB** so Play delivers per-device splits (ABI/density/language) instead of a fat APK — often the single biggest win.
- **Enable R8** (`isMinifyEnabled = true`) to shrink and obfuscate, and **`isShrinkResources = true`** to drop unused resources.
- **Use WebP / vector drawables** instead of PNGs; vectors scale across densities with one asset.
- **Remove unused languages/densities** with `resConfigs`.
- **Audit dependencies** — drop heavyweight libraries; prefer lean alternatives.
- **Compress / move large assets** off-device (download on demand, or use Play Asset/Feature Delivery).
- **Avoid duplicate native libs**; split by ABI.
- **Analyze** with Android Studio's **APK Analyzer** (Build > Analyze APK) to see what's actually large.

```kotlin
android {
    defaultConfig {
        resourceConfigurations += listOf("en", "hi")   // keep only these locales
    }
    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
        }
    }
}
```

```bash
# bundletool: estimate per-device download sizes from an AAB
bundletool get-size total --apks=app.apks
```

**📚 Reference:** <https://developer.android.com/topic/performance/reduce-apk-size>

---

## 17. How can you speed up the Gradle build?

Configuration and infrastructure:

```properties
# gradle.properties
org.gradle.jvmargs=-Xmx4g -XX:+UseParallelGC
org.gradle.parallel=true            # build independent modules in parallel
org.gradle.caching=true             # reuse outputs via the build cache
org.gradle.configuration-cache=true # skip the configuration phase when unchanged
android.useAndroidX=true
kotlin.incremental=true
```

Project structure & habits:

- **Modularize** the app so changes recompile only affected modules; prefer **`implementation` over `api`** (Q5) to shrink the recompile blast radius.
- Use the **build cache** (and a remote cache on CI) to reuse task outputs across builds/machines.
- Adopt the **configuration cache** to skip re-running build scripts.
- **Migrate kapt → KSP** (Q15) — annotation processing is often the slowest part.
- Keep **AGP, Gradle, and the Kotlin compiler up to date**; newer versions are consistently faster.
- Avoid dynamic version ranges (`1.+`) — they force network checks and break caching.
- Don't put expensive work in the **configuration phase**; register tasks lazily.
- Profile with `--scan` or `--profile` to find the real bottleneck.

```bash
./gradlew assembleDebug --scan      # generates a shareable build performance report
./gradlew assembleDebug --profile   # local HTML timing report
```

**📚 Reference:** <https://developer.android.com/build/optimize-your-build>

---

## 18. What is ADB?

**ADB (Android Debug Bridge)** is a command-line tool that lets your computer communicate with a device or emulator. It has three parts: a **client** (the `adb` command), a **server** (a background process on your machine routing commands), and a **daemon (`adbd`)** running on the device.

Common commands:

```bash
adb devices                       # list connected devices/emulators
adb install app.apk               # install (or -r to reinstall)
adb uninstall com.example.app
adb logcat                        # stream device logs
adb shell                         # open a shell on the device
adb push local.txt /sdcard/       # copy file to device
adb pull /sdcard/file.txt .       # copy file from device
adb shell am start -n com.example/.MainActivity   # launch an activity
adb shell pm clear com.example.app                # clear app data
adb shell input text "hello"      # inject input
adb shell screencap /sdcard/s.png # screenshot
adb shell dumpsys meminfo com.example.app         # memory snapshot
adb tcpip 5555 && adb connect 192.168.1.5:5555    # wireless debugging
```

ADB is the backbone of testing, debugging, profiling, and CI device automation.

**📚 Reference:** <https://developer.android.com/studio/command-line/adb>

---

## 19. What is Lint and what is it used for?

**Android Lint** is a **static analysis** tool that scans your source, resources, manifest, and Gradle files for problems **without running the app**. It flags issues across categories: **correctness** (likely bugs), **security** (e.g., insecure flags), **performance** (overdraw, unused resources), **accessibility** (missing `contentDescription`), **usability**, **internationalization**, and **API compatibility** (using an API above `minSdk`).

```bash
./gradlew lint                # run lint, produces HTML/XML reports
./gradlew lintDebug
```

```kotlin
android {
    lint {
        abortOnError = true       // fail the build on errors
        warningsAsErrors = false
        checkDependencies = true
        baseline = file("lint-baseline.xml")  // snapshot existing issues
        disable += "TypographyFractions"
    }
}
```

You can suppress per-site with `@SuppressLint("...")` / `tools:ignore`, define a **baseline** to grandfather existing issues, and even write **custom lint rules**. Running lint in CI enforces code quality automatically.

**📚 Reference:** <https://developer.android.com/studio/write/lint>

---

## 20. What is StrictMode?

**StrictMode** is a developer tool that detects accidental violations of good practice on the main thread and other policies — most notably **disk/network I/O on the UI thread** (which causes jank/ANRs) and **resource leaks** (unclosed `Closeable`s, leaked SQLite cursors, leaked Activities).

It has two policy groups:

- **Thread policy** — disk reads/writes, network access, custom slow calls on the main thread.
- **VM policy** — leaked Closeables, leaked Activity instances, leaked SQLite objects, untagged sockets, etc.

Enable it **only in debug builds**, typically in `Application.onCreate()`:

```kotlin
if (BuildConfig.DEBUG) {
    StrictMode.setThreadPolicy(
        StrictMode.ThreadPolicy.Builder()
            .detectAll()
            .penaltyLog()        // or .penaltyDeath() to crash on violation
            .build()
    )
    StrictMode.setVmPolicy(
        StrictMode.VmPolicy.Builder()
            .detectLeakedClosableObjects()
            .detectActivityLeaks()
            .penaltyLog()
            .build()
    )
}
```

It surfaces problems early; you never ship it enabled in release.

**📚 Reference:** <https://outcomeschool.com/blog/strictmode-in-android-development>

---

## 21. How to use the Android Studio Memory Profiler?

The **Memory Profiler** (under Android Studio's **Profiler** / Profileable runs) visualizes your app's memory usage in real time to find leaks and churn.

Workflow:

1. Run the app with profiling (or attach to a debuggable process) and open **Profiler > Memory**.
2. Watch the live graph of Java/Kotlin, native, graphics, stack, and code memory.
3. **Force a GC** and **capture a heap dump** to inspect live objects, instance counts, **retained size**, and reference chains — find what's holding an object that should have been collected.
4. **Record allocations** over a time window to spot allocation churn (frequent short-lived objects causing GC pressure / jank).
5. Look for classic leaks: Activities/Fragments still reachable after destruction, growing collections, static references to `Context`.

Complementary tools: **LeakCanary** (a library that auto-detects leaked Activities/Fragments in debug builds and gives you the leak trace) is often easier for catching leaks day to day; the Memory Profiler is best for deep investigation and allocation analysis.

**📚 Reference:** <https://developer.android.com/studio/profile/memory-profiler>

---

## 22. How to measure method execution time in Android?

Several approaches, from quick-and-dirty to rigorous:

```kotlin
// 1) System.nanoTime — precise wall-clock for a code block
val start = System.nanoTime()
doWork()
val ms = (System.nanoTime() - start) / 1_000_000.0
Log.d("Perf", "doWork took $ms ms")

// 2) Kotlin's measureTimeMillis / measureNanoTime
val took = measureTimeMillis { doWork() }

// 3) measureTime (kotlin.time) — returns a Duration
val d: Duration = measureTime { doWork() }
```

```kotlin
// 4) Trace API — shows the span in the System Trace / Profiler timeline
Trace.beginSection("loadData")
try { loadData() } finally { Trace.endSection() }
// or: androidx.tracing.trace("loadData") { loadData() }
```

For deeper analysis use the **Android Studio CPU Profiler / System Trace** (method tracing, sampling, flame charts) and, for repeatable benchmarks, the **Jetpack Microbenchmark** and **Macrobenchmark** libraries, which handle warmup, run multiple iterations, and report stable medians. Avoid trusting a single `nanoTime` reading on a JIT'd runtime — benchmark libraries give statistically meaningful numbers.

**📚 Reference:** <https://developer.android.com/topic/performance/benchmarking/microbenchmark-overview>

---

## 23. Can you access your SQLite database for debugging?

Yes. Options:

- **Android Studio Database Inspector** (built in) — run a debuggable app on API 26+, open **View > Tool Windows > App Inspection > Database Inspector**, browse tables, and run live SQL queries against the running app's DB.
- **Android-Debug-Database** library — adds a small server in your debug build; open a URL in your browser (printed in Logcat) to view/edit tables, run queries, and inspect SharedPreferences over the same network. Useful when you want a browser UI or to test on a physical device.
- **ADB pull** — copy the DB file off the device and open it in any SQLite client:

```bash
adb shell run-as com.example.app \
  cp databases/app.db /sdcard/app.db
adb pull /sdcard/app.db .
sqlite3 app.db "SELECT * FROM users;"
```

Database Inspector is the default modern choice; the library shines for physical-device and SharedPreferences inspection.

**📚 Reference:** <https://github.com/amitshekhariitbhu/Android-Debug-Database> · <https://developer.android.com/studio/inspect/database>

---

## 24. CI/CD pipeline for Android

A typical pipeline (GitHub Actions, GitLab CI, Bitrise, Jenkins) on every push/PR:

1. **Checkout** + set up JDK and the Android SDK.
2. **Cache** Gradle dependencies and the build cache.
3. **Static checks** — `lint`, `detekt`/`ktlint`.
4. **Unit tests** — `./gradlew test`.
5. **Instrumented tests** — on an emulator or **Firebase Test Lab**.
6. **Build** — `./gradlew bundleRelease` with signing config injected from secrets.
7. **Deploy** — upload the AAB to Play (internal/alpha/beta) via the **Gradle Play Publisher** or `fastlane supply`; distribute test builds via **Firebase App Distribution**.

```yaml
# .github/workflows/android.yml (excerpt)
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: '17' }
      - uses: gradle/actions/setup-gradle@v4
      - run: ./gradlew lint testDebugUnitTest
      - run: ./gradlew bundleRelease
        env:
          SIGNING_KEY: ${{ secrets.SIGNING_KEY }}
```

Keep signing keys and API credentials in CI secrets, never in the repo. Gate merges on green checks.

**📚 Reference:** <https://www.linkedin.com/posts/amit-shekhar-iitbhu_softwareengineer-androiddev-android-activity-7294600775178063872-Q0oY/>

---

## 25. Android app release checklist for production launch

- **Versioning** — bump `versionCode` (must increase) and `versionName`.
- **Build** — `release` build type with `isMinifyEnabled = true`, `isShrinkResources = true`, **debuggable off**, logging stripped/disabled.
- **Signing** — sign with the upload key; enroll in **Play App Signing**; keep the keystore backed up securely.
- **AAB** — publish an **App Bundle**, not an APK.
- **R8 verification** — test the *minified* release build end to end; keep and upload `mapping.txt` for de-obfuscated crashes.
- **Permissions** — remove unused permissions; justify sensitive ones.
- **Secrets/config** — point to production endpoints, production API keys; no debug/staging config leaking in.
- **Compliance** — privacy policy, Play **Data Safety** form, target API level requirement, **16 KB page size** support (Q30), content rating.
- **Crash/analytics** — Crashlytics/analytics wired up.
- **QA** — test on min/target SDK, multiple form factors, offline, upgrade-from-previous-version path.
- **Store listing** — screenshots, description, what's-new, localized assets.
- **Rollout** — use **staged rollout** (start small) and monitor crash-free rate before going to 100%.

**📚 Reference:** <https://www.linkedin.com/posts/amit-shekhar-iitbhu_androiddev-android-activity-7275029928692080640-HIhx>

---

## 26. Git

**Git** is the distributed version control system used to track changes, collaborate, and manage branches. Interview-relevant essentials:

```bash
git checkout -b feature/login     # branch
git add -p && git commit -m "..."  # stage + commit
git pull --rebase origin main      # update, keep history linear
git merge --no-ff feature/login    # merge
git rebase main                    # replay commits onto main
git revert <sha>                   # undo a commit safely (new commit)
git reset --hard <sha>             # move branch pointer (destructive)
git stash                          # shelve work in progress
git cherry-pick <sha>              # apply one commit elsewhere
git log --oneline --graph
```

Concepts to know: **branching strategies** (trunk-based, Git Flow, GitHub Flow), **PR review** workflow, **merge vs rebase** (merge preserves history with merge commits; rebase produces linear history but rewrites commits — don't rebase shared branches), resolving **merge conflicts**, **tags** for releases, and `.gitignore` (exclude `build/`, `local.properties`, `*.keystore`).

**📚 Reference:** <https://git-scm.com/doc>

---

## 27. Firebase

**Firebase** is Google's mobile/web backend platform — a suite of managed services you integrate via the Firebase BoM and Gradle plugins. Commonly used modules:

- **Authentication** — sign-in (email, Google, phone, etc.).
- **Cloud Firestore / Realtime Database** — NoSQL cloud data with offline sync.
- **Cloud Storage** — user file storage.
- **Cloud Messaging (FCM)** — push notifications.
- **Crashlytics** — crash reporting (upload `mapping.txt` for symbolicated traces).
- **Analytics** — event/usage tracking.
- **Remote Config** — change behavior/values without an app update (Q28).
- **App Distribution** — distribute pre-release builds to testers.
- **Test Lab** — run instrumented tests on real/virtual devices in the cloud.

```kotlin
plugins {
    alias(libs.plugins.google.gms)       // applies google-services
    alias(libs.plugins.firebase.crashlytics)
}
dependencies {
    implementation(platform("com.google.firebase:firebase-bom:33.5.0"))
    implementation("com.google.firebase:firebase-analytics")
    implementation("com.google.firebase:firebase-crashlytics")
}
```

**📚 Reference:** <https://firebase.google.com/>

---

## 28. How to change parameters in an app without an app update?

Use **Firebase Remote Config** (or any server-driven config / feature-flag service). You define key/value parameters in the Firebase console; the app fetches and activates them at runtime, overriding bundled defaults — so you can change values, toggle features, and run A/B tests **without shipping a new build**.

```kotlin
val remoteConfig = Firebase.remoteConfig
remoteConfig.setConfigSettingsAsync(
    remoteConfigSettings { minimumFetchIntervalInSeconds = 3600 }
)
remoteConfig.setDefaultsAsync(R.xml.remote_config_defaults)

remoteConfig.fetchAndActivate().addOnCompleteListener {
    val showNewUi = remoteConfig.getBoolean("show_new_ui")
    val maxItems  = remoteConfig.getLong("max_items").toInt()
    // apply values
}
```

Use cases: **feature flags / kill switches**, gradual feature rollout, tuning thresholds, seasonal content, A/B experiments, and forcing config changes after a backend incident. For data that changes constantly, prefer your own backend API; Remote Config is for relatively stable parameters and flags.

**📚 Reference:** <https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineering-tech-activity-7288408601944043520-pHZX> · <https://firebase.google.com/docs/remote-config>

---

## 29. What is Write-Ahead Logging (WAL)?

**WAL** is a SQLite journaling mode (used by Room/SQLite on Android) that changes *how* durability is achieved. Instead of writing changes directly into the main database file and keeping a rollback journal, WAL **appends changes to a separate `-wal` log file first**; the main DB file is updated later during a **checkpoint**.

Why it's used:

- **Concurrency** — readers and a writer can proceed **simultaneously**. In the default rollback-journal mode a writer blocks readers; with WAL, readers see a consistent snapshot while a write is in progress. This reduces `database is locked` contention.
- **Performance** — appending sequentially to the WAL is faster than the rollback-journal's write-then-sync dance, and `fsync` happens mainly at checkpoint time.
- **Crash safety** — if the app crashes, the WAL can be replayed/rolled forward, so committed transactions survive.

Trade-offs: extra files (`-wal`, `-shm`), periodic **checkpointing** to fold the WAL back into the main DB, and only **one writer at a time** (multiple readers + single writer). Android's Room enables WAL by default on supported API levels.

**📚 Reference:** <https://outcomeschool.com/blog/write-ahead-logging> · <https://www.sqlite.org/wal.html>

---

## 30. 16 KB page size for Android apps

Historically Android used a **4 KB memory page size**. Newer 64-bit devices (and the Android 15+ platform direction) support a **16 KB page size**, which improves performance: faster app launches, lower memory-management overhead, quicker camera starts, and faster boot.

What it means for you:

- **Apps with only Kotlin/Java code are already compatible** — nothing to do.
- **Apps with native code (`.so` libraries via NDK or native dependencies)** must be **rebuilt to be 16 KB-aligned**, or they can crash on 16 KB devices. This means compiling/linking with a sufficiently recent NDK and ensuring shared libraries are aligned to 16 KB ELF segment boundaries.

Requirement timeline (verify the latest on the Play policy page): new apps and updates targeting Android 15+ submitted to Google Play must **support 16 KB page sizes** — the requirement began **November 1, 2025**, with an extended compliance window into **2026** announced in Play Console.

```bash
# Check whether your .so libraries are 16 KB aligned
unzip -o app.apk -d out
# inspect alignment of native libs
objdump -p out/lib/arm64-v8a/libfoo.so | grep LOAD
```

Use a recent AGP/NDK and re-test native paths on a 16 KB emulator image.

**📚 Reference:** <https://www.linkedin.com/posts/amit-shekhar-iitbhu_outcomeschool-softwareengineer-tech-activity-7352176600236351489-ACm6> · <https://developer.android.com/guide/practices/page-sizes>

---

## 31. Gradle Version Catalogs (libs.versions.toml)

A **version catalog** is a central, type-safe registry of dependency coordinates and versions, declared in `gradle/libs.versions.toml`. It replaces scattered hard-coded version strings and the older `ext`/`buildSrc` "Versions object" pattern. Since AGP 7.4+/Gradle 7.4 it's **stable and the default** in new Android Studio projects.

```toml
# gradle/libs.versions.toml
[versions]
kotlin = "2.1.0"
coroutines = "1.9.0"
retrofit = "2.11.0"

[libraries]
retrofit = { module = "com.squareup.retrofit2:retrofit", version.ref = "retrofit" }
coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "coroutines" }

[bundles]
networking = ["retrofit", "coroutines-core"]   # group related deps

[plugins]
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
```

Gradle generates a type-safe `libs` accessor used across all modules — with IDE autocomplete and a compile error if a name is wrong:

```kotlin
// module build.gradle.kts
dependencies {
    implementation(libs.retrofit)
    implementation(libs.bundles.networking)   // pulls the whole bundle
}
plugins { alias(libs.plugins.kotlin.android) }
```

Benefits: a **single source of truth** for versions across a multi-module project, no version drift between modules, `bundles` for grouping, easy upgrades, and shareability (a catalog can be published and reused). Compared with `buildSrc`, editing the catalog does **not** invalidate the whole build's configuration the way changing a `buildSrc` Kotlin file does.

**Why it matters:** Version catalogs are now the default in new projects and the expected answer to "how do you manage dependencies across modules?" — knowing the `[versions]/[libraries]/[bundles]/[plugins]` layout and the `libs.*` accessor signals current Gradle fluency.

**📚 Reference:** <https://docs.gradle.org/current/userguide/version_catalogs.html>

---

## 32. Build cache vs configuration cache

Gradle has **two distinct caches** that speed up builds in different ways — a frequent point of confusion:

| | **Build cache** | **Configuration cache** |
| --- | --- | --- |
| Caches | **Task outputs** (the execution phase) | The result of the **configuration phase** (the task graph) |
| Skips | Re-running tasks whose inputs are unchanged | Re-running build scripts to build the task graph |
| Scope | Local **and remote** (shareable across machines/CI) | Local, per set of build parameters |
| Enable | `org.gradle.caching=true` | `org.gradle.configuration-cache=true` |

```properties
# gradle.properties
org.gradle.caching=true                 # reuse task outputs (build cache)
org.gradle.configuration-cache=true     # skip re-configuring the build
org.gradle.parallel=true                # configure/execute modules in parallel
```

A Gradle build runs in phases: **initialization → configuration → execution**. The **configuration cache** short-circuits the configuration phase (no re-evaluating `build.gradle` scripts) — a big win on large multi-module projects where configuration is expensive. The **build cache** short-circuits the execution phase by reusing the outputs of up-to-date tasks; a **remote** build cache lets CI and teammates reuse each other's outputs.

The configuration cache also enforces best practices: tasks must declare inputs/outputs properly and avoid reading mutable global state at execution time, so adopting it often surfaces latent build issues.

**Why it matters:** "How do you speed up Gradle builds?" almost always leads to "what's the difference between the build cache and the configuration cache?" Nailing the phase each one targets (execution vs configuration) and that the build cache can be remote is the discriminating senior answer.

**📚 Reference:** <https://docs.gradle.org/current/userguide/build_cache.html> · <https://docs.gradle.org/current/userguide/configuration_cache.html>

---
