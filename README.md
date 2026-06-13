# Android Interview Preparation — Complete Guide

A comprehensive, code-first set of interview-preparation guides covering everything you need for a modern Android engineering interview: the Kotlin and Java languages, coroutines and Flow, the Android framework and Jetpack, UI (both the classic View system and Jetpack Compose), architecture, libraries, performance, testing, build tooling, DSA, system design, and the soft-skill / mock-interview material that rounds out a real loop.

Every guide is written as **question → answer** with clear explanations, idiomatic Kotlin/Java code, trade-offs, and "why it matters" notes. All content is kept accurate as of **2026** (Kotlin 2.x, Compose BOM 2025.x, R8/KSP defaults, Android 14/15-era APIs, Java 21+ LTS).

---

## Folder layout

- **`detailed_questions/`** — the full guides with detailed explanations, code, and trade-offs. Linked from the tables below.
- **`sort_questions/`** — condensed **quick-revision** companions (one per guide, named `<guide>_sort.md`) with 1–4 sentence answers for last-minute review. Each mirrors the full guide's question numbering.

---

## How to use this guide

- **Short on time?** Skim the language fundamentals (Kotlin, Java, OOP/SOLID), then Coroutines/Flow and Android core components — these come up in almost every interview.
- **Targeting a senior role?** Spend extra time on Architecture, System Design, Performance/Memory, and Build Tooling.
- **Practicing end to end?** Use the Mock Interview and Preparation Tips sections in the General & Misc guide to rehearse.

---

## Topics

### Languages & Fundamentals

| Guide (Detailed / Short) | What it covers |
| --- | --- |
| [Kotlin Language](./detailed_questions/kotlin_interview.md) / [Short](./sort_questions/kotlin_interview_sort.md) | Null safety, classes, scope functions, generics, delegates, idioms |
| [Kotlin Coroutines](./detailed_questions/kotlin_coroutines_interview.md) / [Short](./sort_questions/kotlin_coroutines_interview_sort.md) | Structured concurrency, scopes, dispatchers, cancellation, pitfalls |
| [Kotlin Flow](./detailed_questions/kotlin_flow_interview.md) / [Short](./sort_questions/kotlin_flow_interview_sort.md) | Cold/hot flows, operators, StateFlow/SharedFlow, backpressure |
| [Java](./detailed_questions/java_interview.md) / [Short](./sort_questions/java_interview_sort.md) | String pool, JMM & GC, concurrency, equality, generics, collections |
| [OOP & SOLID](./detailed_questions/oop_solid_interview.md) / [Short](./sort_questions/oop_solid_interview_sort.md) | OOP fundamentals and the SOLID design principles in Kotlin/Java |
| [DSA & Complexity](./detailed_questions/dsa_complexity_interview.md) / [Short](./sort_questions/dsa_complexity_interview_sort.md) | Big-O, JVM & Android collections, algorithm patterns for coding rounds |

### Android Framework

| Guide (Detailed / Short) | What it covers |
| --- | --- |
| [Core Components & Lifecycle](./detailed_questions/android_components_interview.md) / [Short](./sort_questions/android_components_interview_sort.md) | `Context`, Activity/Fragment lifecycle, Intents, Services, IPC |
| [Jetpack & Core Android](./detailed_questions/android_jetpack_interview.md) / [Short](./sort_questions/android_jetpack_interview_sort.md) | ViewModel, LiveData/StateFlow, WorkManager, DataStore, Navigation |
| [Performance, Memory & Internals](./detailed_questions/android_performance_memory_interview.md) / [Short](./sort_questions/android_performance_memory_interview_sort.md) | Threading, memory/battery optimization, storage, system internals |

### UI

| Guide (Detailed / Short) | What it covers |
| --- | --- |
| [UI / View System](./detailed_questions/android_ui_interview.md) / [Short](./sort_questions/android_ui_interview_sort.md) | Views, layouts, custom views, Canvas, RecyclerView, theming |
| [Jetpack Compose](./detailed_questions/jetpack_compose_interview.md) / [Short](./sort_questions/jetpack_compose_interview_sort.md) | Composition, recomposition, state, side effects, performance |

### Architecture, Design & Libraries

| Guide (Detailed / Short) | What it covers |
| --- | --- |
| [Architecture & Design Patterns](./detailed_questions/android_architecture_design_patterns_interview.md) / [Short](./sort_questions/android_architecture_design_patterns_interview_sort.md) | MVC/MVP/MVVM/MVI, Clean Architecture, multi-module, GoF patterns |
| [Libraries](./detailed_questions/android_libraries_interview.md) / [Short](./sort_questions/android_libraries_interview_sort.md) | Retrofit/OkHttp, Dagger/Hilt, RxJava, Glide/Coil/Fresco |
| [System Design](./detailed_questions/android_system_design_interview.md) / [Short](./sort_questions/android_system_design_interview_sort.md) | Client-focused design (image loader, WhatsApp, caching, offline) |

### Quality & Delivery

| Guide (Detailed / Short) | What it covers |
| --- | --- |
| [Testing](./detailed_questions/android_testing_interview.md) / [Short](./sort_questions/android_testing_interview_sort.md) | Testing pyramid, JUnit, Mockito/MockK, Espresso, Robolectric, Turbine |
| [Build Tools & Gradle](./detailed_questions/android_tools_gradle_interview.md) / [Short](./sort_questions/android_tools_gradle_interview_sort.md) | Gradle (Kotlin DSL), R8/ProGuard, KSP/kapt, ADB, Lint, CI/CD |

### Everything Else

| Guide (Detailed / Short) | What it covers |
| --- | --- |
| [General & Miscellaneous](./detailed_questions/general_misc_interview.md) / [Short](./sort_questions/general_misc_interview_sort.md) | Persistence, security, FCM, dark theme, HR questions, **prep tips & mock interview** |

---

## At a glance

- **17 topic guides**, ~16,000 lines of Q&A content.
- Code-first: idiomatic Kotlin throughout, with Java where the topic demands it.
- Trade-off oriented: every answer is written to survive interview follow-ups, not just recite a definition.

> Currency note: Where the platform has moved on, the guides call it out explicitly — e.g. `StateFlow` over `LiveData`, DataStore over `SharedPreferences`, Compose over the View system for greenfield UI, R8 over ProGuard, and KSP over kapt.
