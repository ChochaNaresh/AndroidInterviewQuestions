# Kotlin Language — Short Answers (Quick Revision)

> Condensed answers for rapid review before an interview. For full explanations and code see `kotlin_interview.md`.

---

## 1. How does Kotlin work on Android (and the JVM)?

**Kotlin compilation** means the process where Kotlin source code is compiled by `kotlinc` into Java bytecode, which on Android is compiled to DEX bytecode and executed by the Android Runtime (ART).

Kotlin compiles to Java bytecode, which on Android is further compiled to DEX (D8/R8) and run by ART. Because the output is ordinary bytecode, Kotlin is fully interoperable with Java. Other backends exist (Kotlin/JS, Kotlin/Native for iOS), but Android uses the JVM path.

---



## 2. Why use Kotlin? Advantages over Java

**Kotlin's advantages** means the benefit of using Kotlin over Java due to its conciseness, null safety, coroutine support, and 100% interoperability.

- Concise / less boilerplate (data classes, type inference, `when`, default args).
- Null safety built into the type system (`String` vs `String?`).
- 100% Java interoperable; adopt incrementally.
- First-class coroutines for structured concurrency.
- Expressive features: smart casts, extension functions, sealed classes, delegation.
- Google's preferred Android language (Jetpack/Compose are Kotlin-first).

---



## 3. val vs var

**var** means a keyword that declares a mutable variable, whereas **val** means a keyword that declares a read-only variable.

`var` is mutable (reassignable); `val` is read-only (assign-once). `val` fixes the **reference**, not the object — a `val mutableListOf(...)` can still be mutated. Prefer `val` by default.

---



## 4. val vs const (and the advantage of const)

**val** means a read-only reference assigned at runtime, whereas **const val** means a compile-time constant.

Both are immutable. `val` can be assigned at runtime; `const val` is a compile-time constant (primitive/String, top-level or in an object/companion). The compiler inlines a `const` at every call site and it's usable in annotations — use it for fixed config values.

---



## 5. Null safety in Kotlin

**Null safety** means a compiler-enforced system that distinguishes nullable and non-nullable types to prevent NullPointerExceptions.

Kotlin separates non-nullable (`String`) and nullable (`String?`) types at the type level. You must handle null before accessing members (via `?.`, `?:`, `!!`, or a smart-cast check). This moves most `NullPointerException`s to compile time.

---



## 6. Safe call (?.) vs not-null assertion (!!)

**Safe call `?.`** means an operator that invokes a member only if the receiver is non-null, whereas **not-null assertion `!!`** means an operator that forcibly unwraps a value and throws a NullPointerException if it is null.

`?.` returns the result if the receiver is non-null, else `null` (no crash). `!!` forcibly unwraps and throws `NullPointerException` if null. Prefer `?.`; `!!` is a code smell, used only when non-null is guaranteed.

---



## 7. Elvis operator (?:)

**Elvis operator `?:`** means a binary operator that returns its left-hand operand if non-null, and its right-hand operand otherwise.

Returns the left operand if non-null, otherwise the right operand (a default/fallback). The right side can also be `return` or `throw` for early-exit guard clauses.

```kotlin
val length = name?.length ?: -1
```

---



## 8. Is there a ternary operator in Kotlin?

**Ternary operator** means a conditional operator (`? :`) which does not exist in Kotlin because `if-else` is an expression that returns a value.

No. `if`/`else` is an expression that returns a value (`val max = if (a > b) a else b`), filling the same role. For null fallbacks use the Elvis operator `?:`; `when` is also an expression.

---



## 9. Data classes

**Data class** means a class designed to hold data for which the compiler automatically generates utility methods like `equals()`, `hashCode()`, `toString()`, and `copy()`.

A class for holding data; the compiler generates `equals`/`hashCode`/`toString`/`componentN`/`copy` based on primary-constructor properties. The primary constructor needs ≥1 `val`/`var` param; can't be `abstract`/`open`/`sealed`/`inner`. Eliminates boilerplate for DTOs/UI state and gives correct equality for sets/maps and diffing.

---



## 10. @JvmStatic, @JvmField, and @JvmOverloads

**@JvmStatic, @JvmField, and @JvmOverloads** means annotations used to control how Kotlin declarations are generated and consumed in Java code.

Improve Java consumption of Kotlin code: `@JvmStatic` emits a real static method (call `MyClass.foo()`), `@JvmField` exposes a property as a plain field (no getter/setter), `@JvmOverloads` generates Java overloads for functions with default parameters. Key for libraries/custom views used from Java.

---



## 11. Primitive types in Kotlin

**Primitive types** means basic data types represented as objects in Kotlin code but compiled to JVM primitives for performance where possible.

Everything is an object in source (`Int`, `Double`, `Boolean`) — no primitive keywords. The compiler maps these to JVM primitives where possible, boxing only when needed (nullable `Int?` or generic `List<Int>`). Boxing matters in hot loops.

---



## 12. String interpolation

**String interpolation** means embedding variables or expressions directly inside a string literal using the `$` symbol.

String templates embed variables/expressions with `$`: `$name` for a simple identifier, `${expr}` for any expression. Escape a literal dollar with `\$`. Cleaner than Java `+` concatenation.

---



## 13. Destructuring declarations

**Destructuring declaration** means a syntax that unpacks an object into multiple variables by calling its component functions.

Unpacks an object into multiple variables via `component1()`, `component2()`, … (auto-generated by data classes). Works with maps, lists, and pairs; use `_` to ignore a value.

```kotlin
val (name, age) = employee
```

---



## 14. lateinit keyword

**lateinit** means a modifier that allows declaring a non-null mutable property without initialising it in the constructor.

Lets you declare a non-null `var` without initialising it in the constructor, promising to assign before first use (DI, `onCreate`, test setup). Only on `var`, non-null, non-primitive; check with `::prop.isInitialized`. Accessing before init throws `UninitializedPropertyAccessException`.

---



## 15. lateinit vs lazy

**lateinit** means a property modifier for manual initialisation later, whereas **lazy** means a delegate that computes a read-only value on first access.

`lateinit` is for `var`, set manually anytime before use, not thread-managed — value comes from outside (DI/lifecycle). `lazy` is for `val`, computed on first access via a lambda, thread-safe by default — for an expensive value built once on demand.

---



## 16. == vs === (structural vs referential equality)

**==** means structural equality that compares values using `equals()`, whereas **===** means referential equality that checks if two references point to the same object.

`==` is structural equality (calls `equals()`, null-safe). `===` is referential equality (same object in memory). Java devs often misuse `==`; in Kotlin `==` is the safe value comparison.

---



## 17. forEach

**forEach** means a higher-order function that executes a lambda block sequentially for each element in an iterable collection.

A higher-order function running a lambda for each element (`it` is the element). A bare `return` inside is non-local (returns from the enclosing function because the lambda is inlined); use `return@forEach` to skip an iteration.

---



## 18. Companion objects

**Companion object** means a singleton object declared inside a class that allows accessing its members using the class name.

A singleton object tied to its enclosing class, letting you call members without an instance — Kotlin's closest equivalent to Java statics. One per class; can hold constants, factory methods, implement interfaces, and use `@JvmStatic`/`@JvmField`. An unnamed companion is accessible as `Class.Companion`; if named (e.g. `companion object Factory`) you reference it as `Class.Factory`.

---



## 19. Equivalent of Java static methods

**Static equivalents** means alternatives to Java's static members in Kotlin including top-level functions, companion objects, or object declarations.

No `static` keyword. Use top-level functions (stateless utilities), a `companion object` (static-like members of a class), or an `object` declaration (singleton). Add `@JvmStatic` for true Java statics. Prefer top-level functions for general utilities.

---



## 20. map vs flatMap

**map** means a transformation operator that maps each element to a new value, whereas **flatMap** means an operator that maps each element to a collection and flattens the results.

`map` transforms each element to exactly one element (same size). `flatMap` transforms each element to a collection and flattens all results into one list. Use `flatMap` when each item yields multiple items.

---



## 21. List vs Array

**Array** means a fixed-size, mutable JVM array structure, whereas **List** means a read-only or mutable collection interface.

- `Array<T>`: class, fixed size, invariant, JVM array, has primitive specializations (`IntArray`, no boxing).
- `List<T>`: interface, read-only (`MutableList` resizable), covariant, boxes primitives.
- Use `List`/`MutableList` for general code; `IntArray`/`Array` for fixed-size, perf-sensitive numeric work.

---



## 22. Visibility modifiers

**Visibility modifiers** means keywords that restrict access to classes, interfaces, constructors, methods, and properties.

`public` (default, everywhere), `private` (declaring class or, top-level, the file), `protected` (class + subclasses, not top-level), `internal` (same module — no Java equivalent). `internal` is key for multi-module/library design.

---



## 23. init blocks

**init block** means an initializer block executed in sequence during class construction to perform initialisation logic.

Run during construction, after primary-constructor params are available, in declaration order interleaved with property initializers. Since the primary constructor can't hold code, `init` is where validation and derived-property setup go. Multiple `init` blocks run top-to-bottom.

---



## 24. Constructors (primary & secondary)

**Primary constructor** means the constructor declared in the class header, whereas **secondary constructor** means an additional constructor declared in the class body.

The primary constructor is in the class header (can declare `val`/`var` props, no code body — use `init`). Secondary constructors use the `constructor` keyword and must delegate to the primary via `this(...)`. Most classes need only a primary constructor with default params.

---



## 25. open keyword (and open vs public)

**open keyword** means a modifier that marks classes or members as inheritable or overridable since they are final by default.

Classes/members are `final` by default; `open` opts in to subclassing/overriding. `open` (inheritance — can it be overridden?) and `public` (visibility — who can see it?) are orthogonal. "Final by default" is a deliberate design.

---



## 26. Lambda expressions

**Lambda expression** means an anonymous function that can be passed as a value or argument.

An anonymous function treated as a value: `{ params -> body }`, single param implicitly `it`. Its type is a function type like `(Int, Int) -> Int`. A trailing lambda (last argument) can be moved outside the parentheses. Foundation of Kotlin's functional style and DSLs.

---



## 27. Higher-order functions (and returning a function)

**Higher-order function** means a function that accepts one or more functions as parameters or returns a function.

A function that takes and/or returns functions, enabling abstraction over behaviour. You can pass `::funcRef` or return a lambda.

```kotlin
fun multiplier(factor: Int): (Int) -> Int = { x -> x * factor }
```

Powers the stdlib (`map`, `filter`), callbacks, and reusable behaviour.

---



## 28. Extension functions

**Extension function** means a mechanism to add new functions to an existing class without inheriting from it.

Add a function to an existing type without inheriting/modifying it; the receiver is `this`. Resolved statically (compiled to a static method taking the receiver), can't access privates, and a member function with the same signature wins. Common for `Context`/`View`/`Flow` helpers.

---



## 29. Infix functions

**Infix function** means a single-parameter member or extension function called without using dots or parentheses.

The `infix` modifier lets a single-parameter member/extension function be called without dot/parentheses (`op minus 8`). Requires exactly one parameter (no default/vararg). Examples: `1 to "one"`, `x until y`. Use sparingly for readability.

---



## 30. Inline functions

**Inline function** means a function whose body and lambda arguments are copied directly into the call site by the compiler to eliminate allocation overhead.

`inline` copies the function body and its lambda arguments into the call site, eliminating lambda object allocation and call overhead. Also enables non-local returns from lambdas and is required for `reified`. Trade-off: larger bytecode — best for small higher-order functions.

---



## 31. noinline

**noinline** means a modifier used on inline function lambda parameters to prevent them from being inlined.

In an `inline` function with multiple lambda params, `noinline` opts one lambda out of inlining so it becomes a real object you can store, pass on, or return. Needed because inlined lambdas can't be assigned to variables or passed onward.

---



## 32. crossinline

**crossinline** means a modifier on inline function lambda parameters that forbids non-local returns when called from another context.

Marks an inlined lambda that must **not** allow non-local returns, used when the lambda is invoked from a different context inside the function (e.g. inside another lambda or a `Runnable`). Inlined, but `return` from it is forbidden (use `return@label`).

---



## 33. Reified type parameters

**Reified type parameter** means a type parameter in an inline function whose actual class type is preserved at runtime.

On an `inline` function, a `reified` type parameter preserves the actual type at runtime (defeating JVM type erasure), so you can use `T::class`, `is T`, `as T`. Enables clean generic APIs like `gson.fromJson<User>(json)`. Only works on `inline` functions.

---



## 34. Scope functions: let, run, with, also, apply

**Scope function** means standard library functions (`let`, `run`, `with`, `also`, `apply`) that execute a code block within the context of an object.

Execute a block in an object's context, differing by reference (`it` vs `this`) and return value (object vs lambda result).

- `let` — `it`, returns result — null-checks/transform.
- `run` — `this`, returns result — compute from members.
- `with` — `this`, returns result — group calls (non-extension).
- `also` — `it`, returns object — side effects.
- `apply` — `this`, returns object — configure/build.

---



## 35. apply vs with

**apply** means an extension function returning the receiver object, whereas **with** means a non-extension function returning the lambda result.

Both reference the object as `this`. `apply` is an extension that returns the receiver — ideal for configuring and returning an object (builder style). `with` is a non-extension that returns the lambda result — ideal for computing a derived value from a non-null object.

---



## 36. Pair and Triple

**Pair and Triple** means simple generic data holders used to return two or three values from a function.

Simple generic holders for returning two/three values without a dedicated class; access via `.first`/`.second`/`.third`, support destructuring. `a to b` is the idiomatic way to build a `Pair` (used in `mapOf`). For meaningful data, prefer a `data class`.

---



## 37. Labels

**Label** means an identifier followed by `@` used to target specific loops, lambdas, or outer class instances.

An identifier + `@` (e.g. `loop@`) names a loop or lambda so you can target it with `break`, `continue`, or `return` (`break@loop`, `return@forEach`). Also enables qualified `this@OuterClass`. Essential for nested loops and local returns inside lambdas.

---



## 38. Sealed classes (and vs enum)

**Sealed class** means a class that restricts its inheritance hierarchy to a closed set of subclasses known at compile time.

A sealed class/interface defines a restricted, closed set of known subtypes (compiler enforces exhaustive `when`, no `else`). vs enum: enum constants are single instances with the same fields; sealed subclasses are different types, each carrying its own state. Sealed classes are "power enums" — ideal for UI state/network results.

---



## 39. Collections overview

**Kotlin collections** means interfaces representing lists, sets, and maps that come in read-only and mutable variants.

Read-only (`listOf`, `setOf`, `mapOf`) and mutable (`mutableListOf`, …) flavors — read-only interfaces just hide mutators, not deeply immutable. List (ordered, duplicates), Set (unique), Map (key→value). Rich functional API (`map`, `filter`, `groupBy`, `fold`); use `asSequence()` for lazy pipelines.

---



## 40. Inline (value) classes

**Inline value class** means a class wrapping a single value that the compiler optimises to avoid runtime allocations.

`@JvmInline value class` wraps a single value to add type safety with little/no runtime overhead (the wrapper is usually erased). Exactly one `val` in the primary constructor; can have methods/interfaces. Prevents mixing same-underlying-type values (e.g. `UserId` vs `OrderId`) without allocation cost.

---



## 41. Delegates (delegated properties & class delegation)

**Delegation** means a pattern where property access or class implementation is forwarded to another object using the `by` keyword.

The `by` keyword delegates work to another object. (1) Delegated properties — `get`/`set` handled by a delegate (`by lazy`, `Delegates.observable`, map-backed). (2) Class delegation — implement an interface by forwarding to an instance (`class A(b: B) : B by b`). Enables composition over inheritance and powers `viewModels()`, Compose `remember`.

---



## 42. Singleton in Kotlin

**Singleton** means a design pattern that restricts a class to a single instance, implemented natively in Kotlin using the `object` keyword.

The `object` declaration is a built-in, lazily and thread-safely initialized singleton — no double-checked-locking boilerplate. For a singleton needing constructor params (e.g. `Context`), use a class with a companion factory + double-checked locking, or a DI framework (Hilt `@Singleton`).

---



## 43. String vs StringBuffer vs StringBuilder

**String** means an immutable character sequence, **StringBuilder** means a mutable non-thread-safe sequence, and **StringBuffer** means a mutable thread-safe sequence.

- `String` — immutable; concatenation in a loop creates many objects.
- `StringBuilder` — mutable, not thread-safe, fastest single-thread.
- `StringBuffer` — mutable, thread-safe (synchronized), slower.

Use `StringBuilder` (or `buildString { }`) for heavy/loop concatenation.

---



## 44. partition

**partition** means a collection function that splits elements into a Pair of two lists based on a predicate.

Splits a collection into a `Pair` of two lists by a predicate (`first` = matches, `second` = rest) in a single pass — better than two `filter` calls.

```kotlin
val (even, odd) = numbers.partition { it % 2 == 0 }
```

---



## 45. associateBy (List to Map)

**associateBy** means a collection transformation that builds a Map using a key selector function.

Converts a list into a Map using a selector for the key (optionally a value transform) — the idiomatic way to build an O(1) lookup table. On duplicate keys, the last wins. Related: `associateWith`, `groupBy`.

```kotlin
val byId = users.associateBy { it.id }
```

---



## 46. Remove duplicates from an array/list

**Deduplication** means removing duplicate elements from a collection using functions like `distinct()` or `toSet()`.

Use `distinct()` (unique elements, order preserved) or `toSet()`. `distinctBy { it.id }` dedupes by a derived key, keeping the first occurrence. `distinct` uses `equals`/`hashCode`, so data classes dedupe by content.

---



## 47. Kotlin Multiplatform (KMP)

**Kotlin Multiplatform (KMP)** means a framework that allows sharing Kotlin code across multiple platforms while keeping native UIs.

Write shared code once (business logic, networking, data, view models) and compile for Android/iOS/desktop/web/server while keeping UI native. `commonMain` holds shared code; `expect`/`actual` declare an API in common and implement it per platform. Stable as of 2026; Compose Multiplatform extends sharing to UI.

---



## 48. Generics & variance (in / out / star projection)

**Generics** means type parameterisation that provides type safety, where **variance** (`in`/`out`) defines how subtyping of template arguments affects the subtyping of the outer class.

Generics are invariant by default. `out T` = covariant (producer, T only in out positions) → `Producer<Dog>` is a `Producer<Animal>`. `in T` = contravariant (consumer, T only in in positions) → `Consumer<Animal>` is a `Consumer<Dog>`. Mnemonic PECS: Producer-Extends/`out`, Consumer-Super/`in`. `*` (star projection) = safe unknown type (read as upper bound, can't write).

---



## 49. typealias

**typealias** means a declaration that provides an alternative name for an existing type without creating a new type.

An alternative name for an existing type — pure source sugar, not a new type, zero runtime cost, fully interchangeable. Tames long generic/functional types. Unlike a value class (which is a distinct type giving extra safety), a typealias adds no type safety.

---



## 50. Operator overloading

**Operator overloading** means defining custom implementations for predefined operators by implementing functions with reserved names.

Operators map to functions with reserved names marked `operator` (`+` → `plus`, `[]` → `get`/`set`, `in` → `contains`, `()` → `invoke`, `==` → `equals`, comparisons → `compareTo`). Only overload when the meaning is unambiguous (math types, DSLs, collection-like wrappers).

