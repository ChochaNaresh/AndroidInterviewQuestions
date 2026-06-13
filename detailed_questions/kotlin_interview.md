# Kotlin Language Interview Questions & Answers

A comprehensive, interview-ready guide to the Kotlin **language** — the core constructs that come up again and again in Android and backend interviews. This file deliberately focuses on language fundamentals (null safety, classes, scope functions, generics, delegates, idioms). Coroutines and Flow are covered in their own dedicated files.

Every answer includes a clear explanation, idiomatic Kotlin code, trade-offs, and a short "why it matters" note. Accurate as of 2026 (Kotlin 2.x).

---

## Table of Contents

1. [How does Kotlin work on Android (and the JVM)?](#1-how-does-kotlin-work-on-android-and-the-jvm)
2. [Why use Kotlin? Advantages over Java](#2-why-use-kotlin-advantages-over-java)
3. [val vs var](#3-val-vs-var)
4. [val vs const](#4-val-vs-const-and-the-advantage-of-const)
5. [Null safety in Kotlin](#5-null-safety-in-kotlin)
6. [Safe call (?.) vs not-null assertion (!!)](#6-safe-call--vs-not-null-assertion-)
7. [Elvis operator (?:)](#7-elvis-operator-)
8. [Is there a ternary operator?](#8-is-there-a-ternary-operator-in-kotlin)
9. [Data classes](#9-data-classes)
10. [@JvmStatic, @JvmField, @JvmOverloads](#10-jvmstatic-jvmfield-and-jvmoverloads)
11. [Primitive types in Kotlin](#11-primitive-types-in-kotlin)
12. [String interpolation](#12-string-interpolation)
13. [Destructuring declarations](#13-destructuring-declarations)
14. [lateinit keyword](#14-lateinit-keyword)
15. [lateinit vs lazy](#15-lateinit-vs-lazy)
16. [== vs ===](#16--vs--structural-vs-referential-equality)
17. [forEach](#17-foreach)
18. [Companion objects](#18-companion-objects)
19. [Equivalent of Java static methods](#19-equivalent-of-java-static-methods)
20. [map vs flatMap](#20-map-vs-flatmap)
21. [List vs Array](#21-list-vs-array)
22. [Visibility modifiers](#22-visibility-modifiers)
23. [init blocks](#23-init-blocks)
24. [Constructors (primary & secondary)](#24-constructors-primary--secondary)
25. [open keyword](#25-open-keyword-and-open-vs-public)
26. [Lambda expressions](#26-lambda-expressions)
27. [Higher-order functions (and returning a function)](#27-higher-order-functions-and-returning-a-function)
28. [Extension functions](#28-extension-functions)
29. [Infix functions](#29-infix-functions)
30. [Inline functions](#30-inline-functions)
31. [noinline](#31-noinline)
32. [crossinline](#32-crossinline)
33. [Reified type parameters](#33-reified-type-parameters)
34. [Scope functions: let, run, with, also, apply](#34-scope-functions-let-run-with-also-apply)
35. [apply vs with](#35-apply-vs-with)
36. [Pair and Triple](#36-pair-and-triple)
37. [Labels](#37-labels)
38. [Sealed classes (and vs enum)](#38-sealed-classes-and-vs-enum)
39. [Collections overview](#39-collections-overview)
40. [Inline (value) classes](#40-inline-value-classes)
41. [Delegates (delegated properties & class delegation)](#41-delegates-delegated-properties--class-delegation)
42. [Singleton in Kotlin](#42-singleton-in-kotlin)
43. [String vs StringBuffer vs StringBuilder](#43-string-vs-stringbuffer-vs-stringbuilder)
44. [partition](#44-partition)
45. [associateBy (List to Map)](#45-associateby-list-to-map)
46. [Remove duplicates from an array/list](#46-remove-duplicates-from-an-arraylist)
47. [Kotlin Multiplatform (KMP)](#47-kotlin-multiplatform-kmp)
48. [Generics & variance (in / out / star projection)](#48-generics--variance-in--out--star-projection)
49. [typealias](#49-typealias)
50. [Operator overloading](#50-operator-overloading)

---

## 1. How does Kotlin work on Android (and the JVM)?

Kotlin source is compiled by the Kotlin compiler (`kotlinc`) into **Java bytecode** (`.class` files), which runs on the **JVM**. On Android, that bytecode is further compiled to DEX by D8/R8 and executed by the **ART** (Android Runtime). Because the output is ordinary bytecode, Kotlin is fully **interoperable** with Java — you can call Java from Kotlin and vice versa within the same project.

Kotlin also has other backends: **Kotlin/JS** (compiles to JavaScript) and **Kotlin/Native** (compiles to native binaries via LLVM, used for iOS in Kotlin Multiplatform). On Android specifically, the JVM path is what's used.

**Why it matters:** Understanding that Kotlin compiles to the same bytecode as Java explains both its seamless interop and why language features are often implemented as compiler tricks (e.g., extension functions become static methods, `data class` generates `equals`/`hashCode`).

**📚 Reference:** https://kotlinlang.org/docs/server-overview.html

---

## 2. Why use Kotlin? Advantages over Java

- **Concise / boilerplate-free** — data classes, type inference, no semicolons, `when`, default arguments.
- **Null-safe by design** — nullability is part of the type system (`String` vs `String?`), eliminating most `NullPointerException`s at compile time.
- **100% interoperable** with Java — adopt incrementally in an existing codebase.
- **First-class coroutines** for structured concurrency (vs. callbacks/threads).
- **Functional features** — higher-order functions, lambdas, immutability by default with `val`.
- **Smart casts, extension functions, sealed classes, delegation** — expressive modeling.
- **Officially preferred on Android** — Google announced Kotlin-first in 2019; Jetpack and Compose are Kotlin-first.

**Why it matters:** Interviewers want to know you can articulate concrete engineering benefits (safety, conciseness, interop), not just "it's modern."

**📚 Reference:** https://kotlinlang.org/docs/comparison-to-java.html

---

## 3. val vs var

- `var` declares a **mutable** variable — you can reassign it.
- `val` declares a **read-only** (assign-once) reference — you cannot reassign it after initialization.

```kotlin
var counter = 0
counter = 1          // OK

val name = "Naresh"
// name = "Other"    // Compile error
```

Important nuance: `val` makes the **reference** immutable, **not the object**. A `val` list can still have its contents mutated if the underlying type is mutable:

```kotlin
val list = mutableListOf(1, 2)
list.add(3)          // OK — the reference is fixed, the object is mutable
```

**Why it matters:** Prefer `val` by default for safer, more predictable code; reach for `var` only when reassignment is genuinely needed.

**📚 Reference:** https://stackoverflow.com/questions/44200075/val-and-var-in-kotlin

---

## 4. val vs const (and the advantage of const)

Both are immutable, but they differ in **when** the value is known:

- `val` — value can be assigned at **runtime** (e.g., from a function call).
- `const val` — a **compile-time constant**. Value must be known at compile time, must be a primitive or `String`, and must be a top-level declaration or a member of an `object`/`companion object`.

```kotlin
const val API_VERSION = "v2"          // baked in at compile time

val currentTime = System.currentTimeMillis()  // runtime value, can't be const
```

**Advantage of `const`:** The compiler **inlines** the value directly at every call site (no field access at runtime), and it's usable in annotations and `when` constants. It signals a true constant.

**Why it matters:** Use `const` for fixed configuration values (keys, tags, versions) for clarity and a tiny performance edge; use `val` for immutable values computed at runtime.

**📚 Reference:** https://outcomeschool.com/blog/const-in-kotlin · https://www.youtube.com/watch?v=3G49ivVxfkU

---

## 5. Null safety in Kotlin

Kotlin distinguishes **non-nullable** and **nullable** types at the type level. A plain `String` can never hold `null`; only `String?` can.

```kotlin
var a: String = "hi"
// a = null               // Compile error

var b: String? = "hi"
b = null                  // OK
```

To access members of a nullable type you must handle the null case — via safe call `?.`, the Elvis operator `?:`, the `!!` assertion, or a smart-cast null check (`if (b != null) b.length`).

```kotlin
val len = b?.length          // Int? — null if b is null
```

**Why it matters:** This moves an entire class of runtime crashes (`NullPointerException`) to compile time, the single most-cited Kotlin advantage.

**📚 Reference:** https://kotlinlang.org/docs/null-safety.html

---

## 6. Safe call (?.) vs not-null assertion (!!)

- **Safe call `?.`** returns the property/result if the receiver is non-null, otherwise returns `null` — no crash.
- **Not-null assertion `!!`** forcibly unwraps; if the value is `null` it throws a `NullPointerException`.

```kotlin
var name: String? = "Some string"
println(name?.length)   // 11
name = null
println(name?.length)   // null   (safe call)
println(name!!.length)  // throws NullPointerException  (assertion)
```

**Why it matters:** `?.` is the safe, idiomatic choice. `!!` is a code smell — only use it when you can guarantee non-null and want a fast failure; overusing it throws away Kotlin's null safety.

**📚 Reference:** https://kotlinlang.org/docs/null-safety.html

---

## 7. Elvis operator (?:)

The Elvis operator `?:` returns its left operand if it is **non-null**, otherwise it returns the right operand (a default/fallback).

```kotlin
val name: String? = null
val length = name?.length ?: -1   // -1 because name is null
```

It pairs naturally with safe calls, and the right side can also be `return` or `throw` for early exit:

```kotlin
fun greet(user: User?) {
    val u = user ?: return            // bail out early
    val email = u.email ?: throw IllegalStateException("no email")
}
```

**Why it matters:** It's Kotlin's concise way to provide defaults and do guard clauses, replacing verbose `if (x != null) ... else ...` blocks.

**📚 Reference:** https://kotlinlang.org/docs/null-safety.html#elvis-operator

---

## 8. Is there a ternary operator in Kotlin?

**No.** Kotlin has no `cond ? a : b`. Instead, `if`/`else` is an **expression** that returns a value, which fills the same role:

```kotlin
val max = if (a > b) a else b
```

For null fallbacks, the Elvis operator `?:` is the idiomatic ternary-like form. `when` is also an expression.

**Why it matters:** Because `if`/`when` are expressions, Kotlin doesn't need a separate ternary; this is a common "gotcha" question for Java developers.

**📚 Reference:** https://kotlinlang.org/docs/control-flow.html#if-expression

---

## 9. Data classes

A `data class` is a class whose main purpose is to hold data. The compiler auto-generates:

- `equals()` / `hashCode()` (based on properties in the primary constructor)
- `toString()` (readable form)
- `componentN()` functions (for destructuring)
- `copy()` (create a modified copy)

```kotlin
data class Employee(val name: String, val age: Int)

val e = Employee("Asha", 30)
val older = e.copy(age = 31)          // copy with one change
val (n, a) = e                        // destructuring
println(e)                            // Employee(name=Asha, age=30)
```

**Requirements:** the primary constructor must have at least one parameter, all of them must be `val`/`var`, and a data class can't be `abstract`, `open`, `sealed`, or `inner`. Note: generated `equals`/`hashCode`/`toString` only consider properties **declared in the primary constructor**.

**Why it matters:** Eliminates the boilerplate Java requires for value objects (DTOs, UI state, API models) — and correct `equals`/`hashCode` is essential for use in sets/maps and for state comparison in Compose/RecyclerView diffing.

**📚 Reference:** https://outcomeschool.com/blog/data-class-in-kotlin · https://kotlinlang.org/docs/data-classes.html

---

## 10. @JvmStatic, @JvmField, and @JvmOverloads

These annotations improve how Kotlin code is consumed from **Java**:

- **`@JvmStatic`** — emits a real Java `static` method for a function in a `companion object`/`object`, so Java can call `MyClass.foo()` instead of `MyClass.Companion.foo()`.
- **`@JvmField`** — exposes a property as a public Java **field** (no getter/setter), so Java accesses it directly.
- **`@JvmOverloads`** — generates Java overloads for a function that has **default parameter values**, so Java callers (which don't understand Kotlin defaults) can omit arguments.

```kotlin
class Config {
    companion object {
        @JvmStatic fun create() = Config()
        @JvmField val DEFAULT = "x"
    }

    @JvmOverloads
    fun setup(timeout: Int = 30, retries: Int = 3) { /* ... */ }
}
```

**Why it matters:** Essential when writing Kotlin libraries or custom views consumed by Java callers (e.g., a custom `View` with `@JvmOverloads` constructors).

**📚 Reference:** https://outcomeschool.com/blog/jvmstatic-annotation-in-kotlin · https://outcomeschool.com/blog/jvmfield-annotation-in-kotlin · https://outcomeschool.com/blog/jvmoverloads-annotation-in-kotlin

---

## 11. Primitive types in Kotlin

In Kotlin **everything is an object** at the language level — there are no primitive keywords; you use `Int`, `Double`, `Boolean`, etc. However, the compiler is smart: where possible it maps these to **JVM primitives** (`int`, `double`) in the generated bytecode for performance, and uses boxed wrappers (`Integer`) only when needed — e.g., when the type is nullable (`Int?`) or used as a generic type argument (`List<Int>`).

```kotlin
val x: Int = 5      // compiles to primitive int
val y: Int? = 5     // boxed (java.lang.Integer) because nullable
```

**Why it matters:** You get a clean, unified type system in source, with no manual boxing — but you should know that nullable/generic numeric types incur boxing, which matters in hot loops.

**📚 Reference:** https://kotlinlang.org/docs/basic-types.html

---

## 12. String interpolation

String templates let you embed variables and expressions directly inside a string using `$`:

```kotlin
val name = "Mohsen"
println("Hello! My name is $name")        // simple variable
println("Next year I'm ${age + 1}")        // expression in braces
```

Use `$var` for a simple identifier and `${...}` for any expression or property access. To print a literal `$`, escape it: `\$`.

**Why it matters:** Cleaner and less error-prone than Java's `+` concatenation or `String.format`.

**📚 Reference:** https://kotlinlang.org/docs/strings.html#string-templates

---

## 13. Destructuring declarations

Destructuring lets you unpack an object into multiple variables in one statement. It works by calling `component1()`, `component2()`, … which data classes generate automatically.

```kotlin
data class Employee(val name: String, val age: Int)

val (name, age) = employee
println(name)
println(age)
```

Common with maps and lists, and to ignore values with `_`:

```kotlin
for ((key, value) in map) { /* ... */ }
val (_, second) = pair
```

**Why it matters:** Concise extraction of multiple return values; works with any class that declares `componentN` operators.

**📚 Reference:** https://kotlinlang.org/docs/destructuring-declarations.html

---

## 14. lateinit keyword

`lateinit` lets you declare a **non-null** `var` property without initializing it in the constructor, promising the compiler you'll assign it before first use. It's for cases where the value is set later — dependency injection, `onCreate`, test `@Before` setup.

```kotlin
lateinit var repository: UserRepository

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    repository = UserRepository()   // initialize before use
}

// Check before access:
if (::repository.isInitialized) repository.load()
```

**Constraints:** only on `var`; only non-null types; not on primitives (`Int`, `Boolean`); not in the primary constructor. Accessing it before initialization throws `UninitializedPropertyAccessException`.

**Why it matters:** Avoids making properties nullable just because they're set after construction — keeps call sites clean (no `?.`).

**📚 Reference:** https://outcomeschool.com/blog/lateinit-vs-lazy-in-kotlin

---

## 15. lateinit vs lazy

| | `lateinit` | `lazy` |
|---|---|---|
| Applies to | `var` only | `val` only |
| Initialization | Set manually, anytime before use | Computed on first access via lambda |
| Thread safety | Not managed | Thread-safe by default (`SYNCHRONIZED`) |
| Use case | Value comes from outside (DI, lifecycle) | Expensive value computed once, lazily |

```kotlin
val database: Database by lazy {        // computed on first access
    Database.create(context)
}

lateinit var presenter: Presenter        // injected later
```

**Why it matters:** Pick `lazy` when the object can build itself the first time it's needed; pick `lateinit` when something external provides the value after construction.

**📚 Reference:** https://outcomeschool.com/blog/lateinit-vs-lazy-in-kotlin

---

## 16. == vs === (structural vs referential equality)

- **`==`** checks **structural** equality — it calls `equals()`. (`a == b` translates to `a?.equals(b) ?: (b === null)`.)
- **`===`** checks **referential** equality — whether two references point to the **same object** in memory.

```kotlin
val a = "kt".plus("lin")   // builds a new String "ktlin"
val b = "ktlin"
println(a == b)            // true  — same contents
println(a === b)           // false — different objects (typically)
```

For primitives represented as JVM primitives, `===` effectively compares values.

**Why it matters:** Java developers habitually misuse `==`. In Kotlin `==` is the safe value comparison (no manual `.equals` null checks), and `===` is the explicit identity check.

**📚 Reference:** https://outcomeschool.com/blog/structural-and-referential-equality-in-kotlin · https://www.youtube.com/watch?v=lJtgxT2OIgQ

---

## 17. forEach

`forEach` is a higher-order function on iterables/collections that runs a lambda for each element — the functional equivalent of a Java for-each loop.

```kotlin
val nums = listOf(1, 2, 3)
nums.forEach { println(it) }              // 'it' is each element
nums.forEachIndexed { i, v -> println("$i:$v") }
```

Note: a plain `return` inside `forEach` returns from the enclosing function (non-local return, because the lambda is inlined). To skip an iteration, use `return@forEach`.

**Why it matters:** Idiomatic iteration; but know the difference between `forEach` (side effects) and `map` (transformation), and the label-return behavior.

**📚 Reference:** https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/for-each.html

---

## 18. Companion objects

A `companion object` is a singleton object tied to its enclosing class, letting you call members **without an instance** — Kotlin's closest equivalent to Java `static` members. There's one companion per class.

```kotlin
class User private constructor(val id: Int) {
    companion object Factory {
        const val TAG = "User"
        fun create(id: Int) = User(id)
    }
}

val u = User.create(1)     // called on the class
```

Companions can implement interfaces and hold state, and their members can be annotated with `@JvmStatic`/`@JvmField` for Java callers. Under the hood the companion is a nested singleton object — accessible as `User.Companion` when unnamed, or by its given name (here `User.Factory`) when named.

**Why it matters:** Factory methods, constants, and "static-like" utilities — and a common idiom for `newInstance()` Fragment factories.

**📚 Reference:** https://outcomeschool.com/blog/companion-object-in-kotlin · https://kotlinlang.org/docs/object-declarations.html#companion-objects

---

## 19. Equivalent of Java static methods

Kotlin has no `static` keyword. To achieve static-like behavior, use:

- **Top-level (package-level) functions** — declared outside any class. Best for stateless utilities.
- **`companion object`** — for "static" members associated with a specific class.
- **`object` declaration** — a singleton; its members are accessed on the object name.

```kotlin
// Top-level
fun formatPrice(p: Double) = "$$p"

// object singleton
object MathUtils { fun square(x: Int) = x * x }

// companion
class Repo { companion object { fun create() = Repo() } }
```

Add `@JvmStatic` if you need true Java `static` methods.

**Why it matters:** Prefer top-level functions for general utilities (no need for a `Utils` class); use companions when the function logically belongs to a class.

**📚 Reference:** https://stackoverflow.com/questions/40352684/what-is-the-equivalent-of-java-static-methods-in-kotlin

---

## 20. map vs flatMap

- **`map`** transforms each element into exactly one element → produces a collection of the same size.
- **`flatMap`** transforms each element into a **collection**, then **flattens** all those collections into one.

```kotlin
val nested = listOf(listOf(1, 2), listOf(3, 4))

nested.map { it.sum() }       // [3, 7]            — one result per element
nested.flatMap { it }         // [1, 2, 3, 4]      — flattened

val words = listOf("ab", "cd")
words.flatMap { it.toList() } // [a, b, c, d]
```

**Why it matters:** `flatMap` is the go-to when each item yields multiple items (e.g., each order has many line items) and you want a single flat list.

**📚 Reference:** https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/flat-map.html

---

## 21. List vs Array

| Aspect | `Array<T>` | `List<T>` |
|---|---|---|
| Nature | Class, fixed size | Interface (`ArrayList`, `LinkedList`, …) |
| Size | Fixed at creation | `List` read-only; `MutableList` resizable |
| Mutability | Elements mutable, size fixed | `List` immutable view; `MutableList` mutable |
| Variance | **Invariant** (`Array<Int>` ≠ `Array<Number>`) | **Covariant** (`List<Int>` is a `List<Number>`) |
| Primitives | Specialized `IntArray`, `DoubleArray` (no boxing) | No primitive specialization (boxes) |
| Compiles to | JVM array | JVM collection |

```kotlin
val arr = arrayOf(1, 2, 3)          // Array<Int>
val prim = intArrayOf(1, 2, 3)      // IntArray — no boxing
val list = listOf(1, 2, 3)          // read-only List
val ml = mutableListOf(1, 2, 3)     // MutableList
```

**Why it matters:** Use `List`/`MutableList` for general app code (flexible, covariant, idiomatic). Reach for `IntArray`/`Array` for fixed-size, performance-sensitive numeric work to avoid boxing.

**📚 Reference:** https://stackoverflow.com/questions/36262305/difference-between-list-and-array-types-in-kotlin

---

## 22. Visibility modifiers

Kotlin has four visibility modifiers (default is **public**):

- **`public`** (default) — visible everywhere.
- **`private`** — visible inside the declaring class, or (for top-level declarations) inside the same **file**.
- **`protected`** — visible in the declaring class and its **subclasses** (not on top-level declarations).
- **`internal`** — visible everywhere within the same **module** (a compilation unit, e.g., a Gradle module). This has no Java equivalent.

```kotlin
class Account {
    private val pin = 1234            // class-only
    protected open val balance = 0    // class + subclasses
    internal val accountType = "X"    // same module
    val owner = "Asha"                // public (default)
}
```

**Why it matters:** `internal` is key for library/multi-module design — exposing APIs publicly while hiding implementation details across module boundaries.

**📚 Reference:** https://kotlinlang.org/docs/visibility-modifiers.html · https://youtu.be/wOHpuf74-cI

---

## 23. init blocks

An `init` block runs as part of object construction, **after** the primary constructor's parameters are available, executing **in declaration order** interleaved with property initializers. Since the primary constructor can't contain code, `init` is where primary-constructor logic goes.

```kotlin
class User(name: String) {
    val displayName: String
    init {
        require(name.isNotBlank()) { "name required" }
        displayName = name.trim().uppercase()
        println("User created")
    }
}
```

A class can have multiple `init` blocks; they run top-to-bottom together with property initializers, all before any secondary constructor body.

**Why it matters:** It's the canonical place for validation and derived-property setup tied to the primary constructor.

**📚 Reference:** https://outcomeschool.com/blog/init-block-in-kotlin · https://kotlinlang.org/docs/classes.html#constructors

---

## 24. Constructors (primary & secondary)

- **Primary constructor** — part of the class header. It can declare properties directly (`val`/`var`) but cannot contain code; use `init` blocks for initialization logic.
- **Secondary constructor(s)** — declared in the body with the `constructor` keyword. Each must delegate to the primary constructor (directly or via another secondary) using `this(...)`. You can have multiple.

```kotlin
class Person(val name: String, val age: Int) {   // primary
    init { println("primary") }

    constructor(name: String) : this(name, 0) {   // secondary → delegates
        println("secondary")
    }
}
```

If a class has a primary constructor, every secondary constructor must chain to it. The primary `init` blocks and property initializers run before the secondary constructor body.

**Why it matters:** Most Kotlin classes need only a primary constructor (often with default parameter values instead of overloads); secondary constructors are mainly for interop or alternate construction paths.

**📚 Reference:** https://kotlinlang.org/docs/classes.html#constructors

---

## 25. open keyword (and open vs public)

By default, Kotlin classes and members are **`final`** — they cannot be subclassed or overridden. The **`open`** keyword opts in to inheritance/overriding.

```kotlin
open class Animal {          // can be subclassed
    open fun sound() {}      // can be overridden
    fun name() {}            // final — cannot be overridden
}

class Dog : Animal() {
    override fun sound() {}
}
```

**`open` vs `public`** — they answer different questions and are orthogonal:

- `public` is a **visibility** modifier — *who can see/access* the member.
- `open` is an **inheritance** modifier — *whether it can be subclassed/overridden*.

A member can be `public` but `final` (visible everywhere, not overridable), or `open` — these combine independently.

**Why it matters:** Kotlin's "final by default" is a deliberate design (effective-Java "design for inheritance or prohibit it"). Don't confuse visibility with overridability — a common interview trap.

**📚 Reference:** https://outcomeschool.com/blog/open-keyword-in-kotlin

---

## 26. Lambda expressions

A lambda is an **anonymous function** treated as a value — you can store it in a variable, pass it as an argument, or return it. Syntax: `{ params -> body }`. A single parameter is implicitly named `it`.

```kotlin
val sum = { a: Int, b: Int -> a + b }
println(sum(2, 3))                 // 5

listOf(1, 2, 3).filter { it > 1 }  // 'it' = each element

// Trailing lambda convention:
button.setOnClickListener { onClick() }
```

The type of a lambda is a function type, e.g., `(Int, Int) -> Int`. If a lambda is the last argument, it can be moved outside the parentheses (trailing lambda).

**Why it matters:** Lambdas are the foundation of Kotlin's functional style and DSLs (Compose, Gradle Kotlin DSL, collection operations).

**📚 Reference:** https://outcomeschool.com/blog/higher-order-functions-and-lambdas-in-kotlin · https://kotlinlang.org/docs/lambdas.html

---

## 27. Higher-order functions (and returning a function)

A **higher-order function** takes one or more functions as parameters and/or returns a function. This enables abstraction over behavior.

```kotlin
// Takes a function:
fun passMeFunction(action: () -> Unit) {
    action()
}

fun add(a: Int, b: Int): Int = a + b

// Returns a function:
fun returnMeAddFunction(): (Int, Int) -> Int {
    return ::add                  // function reference
}

val adder = returnMeAddFunction()
val result = adder(2, 2)          // 4
```

You can also return a lambda directly:

```kotlin
fun multiplier(factor: Int): (Int) -> Int = { x -> x * factor }
val triple = multiplier(3)
println(triple(5))                // 15
```

**Why it matters:** Higher-order functions power the standard library (`map`, `filter`, scope functions), callbacks, and reusable behavior (e.g., a `retry(times) { block() }` utility).

**📚 Reference:** https://outcomeschool.com/blog/higher-order-functions-and-lambdas-in-kotlin · https://x.com/amitiitbhu/status/1862721662208155800

---

## 28. Extension functions

Extension functions let you add a function to an existing class **without inheriting or modifying it**. You declare the receiver type before the function name; inside, `this` refers to the receiver.

```kotlin
fun View.show() { visibility = View.VISIBLE }
fun View.hide() { visibility = View.GONE }

toolbar.hide()

fun String.isEmail() = contains("@")
"a@b.com".isEmail()   // true
```

How it works: extensions are resolved **statically** (compiled to static methods taking the receiver as the first parameter). They don't actually modify the class and can't access its `private` members; if a member function with the same signature exists, the **member wins**.

**Why it matters:** Cleaner, discoverable utility APIs (very common in Android for `Context`, `View`, `Flow` helpers) without subclassing or `Utils` classes. Extension *properties* exist too (no backing field).

**📚 Reference:** https://outcomeschool.com/blog/extension-function-in-kotlin · https://kotlinlang.org/docs/extensions.html

---

## 29. Infix functions

The `infix` modifier lets you call a single-parameter member or extension function **without the dot and parentheses**, reading like an operator.

```kotlin
class Operations {
    var x = 10
    infix fun minus(num: Int) { x -= num }
}

val op = Operations()
op minus 8        // same as op.minus(8)
println(op.x)     // 2
```

Requirements: must be a member or extension function, have exactly **one** parameter (no default value, not vararg). Standard-library examples: `1 to "one"`, `a shl 2`, `x until y`, `"abc" in list`.

**Why it matters:** Improves readability for DSL-like APIs and pair creation (`"key" to value`), but should be used sparingly where it genuinely reads better.

**📚 Reference:** https://outcomeschool.com/blog/infix-notation-in-kotlin

---

## 30. Inline functions

`inline` tells the compiler to **copy the function's body (and its lambda arguments' bodies) directly into the call site** instead of making a real function call. This is primarily an optimization for **higher-order functions**: without inlining, each lambda becomes a `Function` object allocation; inlining removes that overhead.

```kotlin
inline fun measure(block: () -> Unit) {
    val start = System.nanoTime()
    block()
    println("Took ${System.nanoTime() - start} ns")
}
```

Benefits:
- No lambda object allocation / no call overhead.
- Enables **non-local returns** from the lambda (a `return` inside the lambda returns from the caller).
- Required to support **`reified`** type parameters.

Trade-offs: increases bytecode size if the function is large or called in many places; best for small functions taking lambdas. Use `noinline`/`crossinline` to fine-tune lambda behavior.

**Why it matters:** Understanding inlining explains why stdlib functions like `let`, `forEach`, `filter` are cheap, and why `reified` requires `inline`.

**📚 Reference:** https://outcomeschool.com/blog/inline-function-in-kotlin · https://www.youtube.com/watch?v=GLLI8h67ryo

---

## 31. noinline

In an `inline` function with multiple lambda parameters, `noinline` marks a specific lambda that should **not** be inlined. You need this when you want to treat that lambda as a real object — e.g., store it, pass it to another (non-inline) function, or return it.

```kotlin
inline fun doSomething(abc: () -> Unit, noinline xyz: () -> Unit) {
    abc()            // inlined
    saveForLater(xyz) // xyz is a real object — can be stored/passed
}
```

**Why it matters:** Inlined lambdas can't be assigned to variables or passed onward; `noinline` selectively opts a lambda out so you regain that flexibility.

**📚 Reference:** https://outcomeschool.com/blog/noinline-in-kotlin

---

## 32. crossinline

`crossinline` marks an inlined lambda that must **not allow non-local returns**, while still being inlined. You need it when the lambda is invoked from a different execution context inside the inline function — e.g., inside another lambda, an object, or a `Runnable` — where a non-local `return` would be illegal.

```kotlin
inline fun runOnUi(crossinline block: () -> Unit) {
    Handler(Looper.getMainLooper()).post {
        block()      // invoked inside another lambda → crossinline required
    }
}
```

Contrast:
- normal inline lambda → can do non-local `return`.
- `crossinline` → inlined, but `return` from it is forbidden (must use `return@label`).
- `noinline` → not inlined at all.

**Why it matters:** Without `crossinline`, the compiler rejects calling the lambda from a nested context; it's a common need when wrapping callbacks/posting to handlers.

**📚 Reference:** https://outcomeschool.com/blog/crossinline-in-kotlin

---

## 33. Reified type parameters

Due to JVM **type erasure**, a generic type argument `T` isn't available at runtime — you normally can't do `T::class` or `is T`. Marking a type parameter **`reified`** on an **`inline`** function preserves the actual type at the call site (because the body is inlined, the concrete type is known there).

```kotlin
inline fun <reified T> Gson.fromJson(json: String): T =
    fromJson(json, T::class.java)

inline fun <reified T> List<*>.filterIsType(): List<T> =
    filterIsInstance<T>()   // uses 'is T' under the hood

val user: User = gson.fromJson(jsonString)   // no need to pass User::class.java
```

Requirements: only works on `inline` functions. With reified you can call `T::class`, use `is T` / `as T`, and create arrays of `T`.

**Why it matters:** Powers clean generic APIs (JSON parsing, DI lookups like `inject<T>()`, type-filtering) without passing `Class<T>` tokens around.

**📚 Reference:** https://www.youtube.com/watch?v=kD2T84FnTck · https://kotlinlang.org/docs/inline-functions.html#reified-type-parameters

---

## 34. Scope functions: let, run, with, also, apply

Scope functions execute a block in the context of an object. They differ in two ways: how the object is **referenced** (`it` vs `this`) and what they **return** (the object vs the lambda result).

| Function | Object as | Returns | Typical use |
|---|---|---|---|
| `let` | `it` | lambda result | null-checks, transform a value |
| `run` | `this` | lambda result | compute a result from an object's members |
| `with` | `this` | lambda result | group calls on one object (non-extension) |
| `also` | `it` | the object | side effects (logging) without breaking the chain |
| `apply` | `this` | the object | configure/build an object |

```kotlin
// let — null check + transform
val len = name?.let { println(it); it.length }

// run — compute result using members
val area = rectangle.run { width * height }

// with — call many methods on one object
with(numbers) { println("size=$size, first=${first()}") }

// also — side effect, returns receiver
val list = mutableListOf(1, 2).also { println("created $it") }.apply { add(3) }

// apply — configure, returns receiver
val person = Person().apply { name = "John"; age = 25 }
```

**Quick rule of thumb:** return the configured object → `apply`/`also`; need a computed result → `let`/`run`/`with`. Reference by `it` → `let`/`also`; by `this` → `run`/`with`/`apply`.

**Why it matters:** They make object handling concise and reduce temporary variables — a staple of idiomatic Kotlin and a frequent interview discussion.

**📚 Reference:** https://kotlinlang.org/docs/scope-functions.html · https://stackoverflow.com/questions/45977011/example-of-when-should-we-use-run-let-apply-also-and-with-on-kotlin

---

## 35. apply vs with

Both reference the object as **`this`**, but:

- **`apply`** is an **extension** function and **returns the receiver object** — ideal for configuring and returning an object (builder style).
- **`with`** is a **regular** (non-extension) function that takes the object as an argument and **returns the lambda result** — ideal when you want to operate on an object and get a computed value, and when the object is non-null.

```kotlin
// apply — returns the object (good for construction)
val intent = Intent().apply {
    putExtra("id", 1)
    action = Intent.ACTION_VIEW
}

// with — returns a result
val summary = with(user) {
    "$name ($age)"        // returns this string
}
```

Practical guidance: use `apply` when the final value you want **is the object** (configuring a view, intent, builder); use `with` when you want a **derived result** and already have a non-null reference. (`apply` is null-safe via `?.apply { }`; `with` is not, hence `run` is often preferred for nullable receivers.)

**Why it matters:** Choosing the right one keeps chains clean and signals intent (configuration vs computation).

**📚 Reference:** https://kotlinlang.org/docs/scope-functions.html

---

## 36. Pair and Triple

`Pair` and `Triple` are simple generic data holders for returning **two** or **three** values from a function without defining a dedicated class. Members are accessed via `.first`, `.second`, `.third` and they support destructuring.

```kotlin
fun minMax(list: List<Int>): Pair<Int, Int> = list.min() to list.max()

val (lo, hi) = minMax(listOf(3, 1, 9))
val triple = Triple("a", 1, true)
println(triple.second)            // 1
```

`a to b` is the idiomatic infix way to create a `Pair` (used heavily for `mapOf("k" to v)`).

**Why it matters:** Handy for quick multi-value returns, but for anything with meaning, a `data class` with named properties is clearer and more maintainable than `Pair`/`Triple`.

**📚 Reference:** https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-pair/

---

## 37. Labels

A label is an identifier followed by `@` (e.g., `loop@`, `outer@`) that names an expression — most often a loop or a lambda — so you can target it with `break`, `continue`, or `return`.

```kotlin
loop@ for (i in 1..3) {
    for (j in 1..3) {
        if (i + j == 4) break@loop   // breaks the outer loop
    }
}

listOf(1, 2, 3).forEach lit@{
    if (it == 2) return@lit          // continue-like skip
    println(it)
}
```

Labels also enable qualified `this` (`this@OuterClass`) and qualified returns from lambdas (`return@forEach`).

**Why it matters:** Essential for controlling flow in nested loops and for doing local returns inside lambdas (especially in `forEach`/`let` where a bare `return` would exit the whole function).

**📚 Reference:** https://kotlinlang.org/docs/returns.html

---

## 38. Sealed classes (and vs enum)

A **sealed class** (or `sealed interface`) defines a **restricted, closed set of subtypes** known at compile time — all direct subclasses must be in the same module/package. They model "this can be exactly one of these known shapes."

```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val message: String) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

fun render(state: Result<User>) = when (state) {   // no 'else' needed
    is Result.Success -> show(state.data)
    is Result.Error   -> showError(state.message)
    Result.Loading    -> showSpinner()
}
```

**Sealed class vs enum:**

| | `enum` | `sealed class` |
|---|---|---|
| Instances | Fixed set of **single** instances | Multiple **instances**, each can carry different state |
| State | Same fields for all constants | Each subclass has its **own** properties/types |
| Subtypes | Not really subclassable | Different subclass **types** (data class, object, etc.) |

Sealed classes are "power enums": when an enum's cases would need different data, use a sealed class.

**Why it matters:** Exhaustive `when` (compiler enforces all cases handled, no `else`) makes them ideal for modeling UI state, network results, and events safely.

**📚 Reference:** https://kotlinlang.org/docs/sealed-classes.html

---

## 39. Collections overview

Kotlin's collections come in **read-only** and **mutable** flavors (the read-only interfaces don't expose mutators; they're not deeply immutable, just a read-only view).

- **List** — ordered, indexed, allows duplicates. `listOf` / `mutableListOf`.
- **Set** — unordered, unique elements. `setOf` / `mutableSetOf`.
- **Map** — key→value, unique keys. `mapOf` / `mutableMapOf`.

```kotlin
val nums = listOf(1, 2, 3, 4)          // List<Int> (read-only)
println(nums[2])                        // 3

val fruits = setOf("apple", "banana")   // Set
val capitals = mapOf("DE" to "Berlin")  // Map
println(capitals["DE"])                 // Berlin

val mutable = mutableListOf(1, 2)
mutable.add(3)
```

The standard library provides a rich functional API: `map`, `filter`, `groupBy`, `associateBy`, `partition`, `fold`/`reduce`, `flatMap`, `sortedBy`, etc. Use `Sequence` (`asSequence()`) for lazy evaluation over large/expensive pipelines to avoid intermediate collections.

**Why it matters:** Prefer read-only types in APIs to communicate immutability of intent; know the functional operators to write expressive, concise data transformations.

**📚 Reference:** https://kotlinlang.org/docs/collections-overview.html

---

## 40. Inline (value) classes

An inline/value class wraps a single value to add **type safety with zero (or minimal) runtime overhead** — at runtime the wrapper is usually erased and the underlying value is used directly. Declared with `@JvmInline value class`.

```kotlin
@JvmInline
value class UserId(val value: String)

@JvmInline
value class Password(val value: String)

fun login(id: UserId, pw: Password) { /* ... */ }

// login(Password("x"), UserId("y"))  // Compile error — types don't mix!
```

Rules: exactly **one** property in the primary constructor (`val`), no backing fields for other state, can have methods and implement interfaces. The wrapper is boxed only when needed (e.g., used as a nullable type or a generic argument).

**Why it matters:** Prevents mixing semantically different values of the same underlying type (e.g., `UserId` vs `OrderId`, both `String`) without the allocation cost of a normal wrapper class — domain modeling with no performance penalty.

**📚 Reference:** https://kotlinlang.org/docs/inline-classes.html

---

## 41. Delegates (delegated properties & class delegation)

Kotlin's `by` keyword delegates work to another object. Two forms:

**1. Delegated properties** — a property's `get`/`set` is handled by a delegate object that implements `getValue`/`setValue`. Built-in delegates:

```kotlin
val lazyValue: String by lazy { computeExpensive() }   // lazy init

var name: String by Delegates.observable("init") { _, old, new ->
    println("$old -> $new")                            // react to changes
}

val config: Map<String, Any> = mapOf("port" to 8080)
val port: Int by config                                // map-backed property
```

You can write custom delegates by implementing `ReadOnlyProperty`/`ReadWriteProperty` or the `getValue`/`setValue` operators.

**2. Class delegation** — implement an interface by delegating to another instance, avoiding boilerplate forwarding:

```kotlin
interface Repository { fun load(): String }

class RealRepo : Repository { override fun load() = "data" }

class LoggingRepo(repo: Repository) : Repository by repo {  // forwards to repo
    override fun load(): String {
        println("loading")
        return "data"   // can selectively override
    }
}
```

**Why it matters:** Delegation enables the favor-composition-over-inheritance pattern, reusable property behaviors (`lazy`, `observable`, `vetoable`), and powers libraries (Compose `remember`, `viewModels()`, preferences).

**📚 Reference:** https://kotlinlang.org/docs/delegated-properties.html · https://kotlinlang.org/docs/delegation.html

---

## 42. Singleton in Kotlin

Kotlin has a built-in singleton: the **`object` declaration**. It's instantiated lazily and thread-safely on first access — no need for the double-checked-locking boilerplate Java requires.

```kotlin
object AnalyticsTracker {
    private var count = 0
    fun track(event: String) { count++ /* ... */ }
}

AnalyticsTracker.track("open")   // same single instance everywhere
```

For a singleton that needs **constructor parameters** (e.g., a `Context`), use a class with a companion-object factory and double-checked locking, or better, a DI framework (Hilt `@Singleton`):

```kotlin
class Database private constructor(context: Context) {
    companion object {
        @Volatile private var instance: Database? = null
        fun get(context: Context): Database =
            instance ?: synchronized(this) {
                instance ?: Database(context.applicationContext).also { instance = it }
            }
    }
}
```

**Why it matters:** `object` is the idiomatic, safe singleton. Know the parameterized variant for Android (Room, Retrofit clients) and that DI is usually the cleaner solution.

**📚 Reference:** https://kotlinlang.org/docs/object-declarations.html

---

## 43. String vs StringBuffer vs StringBuilder

- **`String`** — **immutable**. Every modification creates a new object; concatenating in a loop is costly (many temporary objects).
- **`StringBuilder`** — **mutable**, **not** thread-safe. Fast for building/modifying strings in a single thread.
- **`StringBuffer`** — **mutable**, **thread-safe** (methods are `synchronized`), so slightly slower due to locking.

```kotlin
val sb = StringBuilder()
for (i in 1..1000) sb.append(i)
val result = sb.toString()
```

| | Mutable | Thread-safe | Performance |
|---|---|---|---|
| String | No | N/A (immutable) | Poor for repeated concatenation |
| StringBuilder | Yes | No | Fastest single-thread |
| StringBuffer | Yes | Yes | Slower (synchronized) |

In Kotlin, simple concatenation/templates are fine for a few values; use `StringBuilder` (or the `buildString { }` helper) for heavy/loop concatenation. Use `StringBuffer` only when multiple threads truly share one builder (rare).

**Why it matters:** Choosing `StringBuilder` over `+` in loops is a classic performance question; thread-safety vs speed is the key distinction between the two builders.

**📚 Reference:** https://outcomeschool.com/blog/string-vs-stringbuffer-vs-stringbuilder

---

## 44. partition

`partition` is a filtering function that splits a collection into **two lists** based on a predicate: a `Pair` where `first` holds elements that satisfy the predicate and `second` holds those that don't.

```kotlin
val numbers = listOf(1, 2, 3, 4, 5, 6)
val (even, odd) = numbers.partition { it % 2 == 0 }
println(even)   // [2, 4, 6]
println(odd)    // [1, 3, 5]
```

It's more efficient and readable than calling `filter` twice (one pass instead of two), and destructuring gives both results cleanly.

**Why it matters:** Single-pass split into "matches / non-matches" — handy for separating valid vs invalid items, active vs inactive, etc.

**📚 Reference:** https://outcomeschool.com/blog/partition-filtering-function-in-kotlin

---

## 45. associateBy (List to Map)

`associateBy` converts a list into a **Map**, using a selector to choose the key (and optionally a value transform). It's the idiomatic way to build a lookup table from a list.

```kotlin
data class User(val id: Int, val name: String)
val users = listOf(User(1, "A"), User(2, "B"))

val byId: Map<Int, User> = users.associateBy { it.id }
println(byId[2])                       // User(id=2, name=B)

// With value transform:
val idToName = users.associateBy({ it.id }, { it.name })  // {1=A, 2=B}
```

If two elements produce the same key, the **last one wins**. Related: `associateWith` (keys from elements, values from selector) and `groupBy` (one key → list of elements).

**Why it matters:** Turning a `List` fetched from a DB/API into an O(1) lookup `Map` by ID is extremely common; `associateBy` does it in one expressive call.

**📚 Reference:** https://outcomeschool.com/blog/associateby-list-to-map-in-kotlin

---

## 46. Remove duplicates from an array/list

The simplest idiomatic way is `distinct()`, which returns a list of unique elements (preserving order). Converting to a `Set` also removes duplicates.

```kotlin
val nums = listOf(1, 2, 2, 3, 3, 3)
val unique = nums.distinct()              // [1, 2, 3]
val asSet = nums.toSet()                   // {1, 2, 3} (Set)

// Dedup by a property:
val users = listOf(User(1, "A"), User(1, "B"))
val byId = users.distinctBy { it.id }      // keeps first per id

// Arrays:
val arr = arrayOf(1, 1, 2).distinct()      // works on arrays too
```

`distinct` uses `equals`/`hashCode` (so data classes dedupe by content); `distinctBy` dedupes by a derived key, keeping the first occurrence.

**Why it matters:** Shows command of the collections API; `distinctBy` is the answer when you need to dedupe objects by a specific field rather than full equality.

**📚 Reference:** https://outcomeschool.com/blog/remove-duplicates-from-an-array-in-kotlin

---

## 47. Kotlin Multiplatform (KMP)

Kotlin Multiplatform lets you write **shared code once** and compile it for multiple targets — Android (JVM), iOS (Native), desktop, web (JS/Wasm), and server. You share business logic (networking, data, domain, view models) while keeping platform-specific UI and APIs native.

Key mechanics:
- **`commonMain`** holds shared, platform-agnostic code; platform source sets (`androidMain`, `iosMain`) hold platform-specific implementations.
- **`expect`/`actual`** — declare an `expect`ed API in common code and provide the `actual` implementation per platform.

```kotlin
// commonMain
expect fun platformName(): String

// androidMain
actual fun platformName() = "Android"

// iosMain
actual fun platformName() = "iOS"
```

Each target compiles via its backend: Android → JVM bytecode, iOS → native (LLVM via Kotlin/Native), web → JS/Wasm. **Compose Multiplatform** extends this to share UI as well. As of 2026, KMP is **Stable** for sharing logic.

**Why it matters:** Maximizes code reuse across platforms while preserving native performance and UX — a major reason teams adopt Kotlin beyond Android.

**📚 Reference:** https://kotlinlang.org/docs/multiplatform.html · https://youtu.be/nwfNh6Kd5hI

---

## 48. Generics & variance (in / out / star projection)

Generics let you write type-safe code that works over many types (`List<T>`, `Box<T>`). The hard interview question is **variance**: if `Dog` is a subtype of `Animal`, is `Box<Dog>` a subtype of `Box<Animal>`? By default in Kotlin, **no** — generics are *invariant*. Variance annotations relax this safely.

**Declaration-site variance** — annotate the type parameter where the class is declared:

```kotlin
// out T  → COVARIANT: T only appears in "out" positions (return values, producers).
//          Producer<Dog> IS-A Producer<Animal>.
interface Producer<out T> {
    fun produce(): T            // OK: T is returned
    // fun consume(item: T)     // COMPILE ERROR: T cannot be a parameter
}

// in T   → CONTRAVARIANT: T only appears in "in" positions (parameters, consumers).
//          Consumer<Animal> IS-A Consumer<Dog>.
interface Consumer<in T> {
    fun consume(item: T)        // OK: T is consumed
    // fun produce(): T         // COMPILE ERROR: T cannot be returned
}

val animals: Producer<Animal> = object : Producer<Dog> { override fun produce() = Dog() } // OK (out)
val dogEater: Consumer<Dog>   = object : Consumer<Animal> { override fun consume(i: Animal) {} } // OK (in)
```

A handy mnemonic is **PECS** (from Java): *Producer-Extends, Consumer-Super* → Kotlin's `out` = produce, `in` = consume. This is why `List<out E>` is covariant (read-only producer) while `MutableList<E>` is invariant (it also consumes).

**Use-site variance (type projection)** — when the class itself is invariant, annotate at the use site:

```kotlin
fun copy(from: Array<out Any>, to: Array<Any>) { /* from is read-only here */ }
copy(arrayOf("a", "b"), arrayOfNulls<Any>(2))   // Array<String> accepted via out-projection
```

**Star projection (`*`)** — when you don't know or care about the type argument but want type safety:

```kotlin
fun printSize(list: List<*>) = println(list.size)   // can read as Any?, cannot add non-null items
```

`Foo<*>` for `Foo<out T : TUpper>` is equivalent to `Foo<out TUpper>` for reads and `Foo<in Nothing>` for writes — so you can read (as the upper bound) but not safely write.

**Why it matters:** Variance is one of the most common "senior Kotlin" questions. Being able to say *"`out` = covariant producer, `in` = contravariant consumer, default is invariant, and `*` is a safe unknown"* — with the PECS rationale — signals real depth.

**📚 Reference:** https://kotlinlang.org/docs/generics.html

---

## 49. typealias

A `typealias` introduces an alternative name for an existing type. It does **not** create a new type — it's pure source-level sugar, erased by the compiler — so it's fully interchangeable with the original and adds no runtime overhead.

```kotlin
// Tame long generic / functional types:
typealias ClickHandler = (View, Int) -> Unit
typealias UserCache = MutableMap<UserId, List<User>>

// Disambiguate names via import alias relatives:
typealias AndroidColor = android.graphics.Color

fun setListener(handler: ClickHandler) { /* ... */ }
val cache: UserCache = mutableMapOf()
```

Contrast with an **inline (value) class**: a value class *is* a distinct type (gives you type safety, e.g. preventing a `UserId` being passed where an `OrderId` is expected), whereas a `typealias` is the *same* type with a friendlier name and gives no extra safety.

**Why it matters:** Knowing `typealias` is a readability tool — not a new type — and contrasting it with value classes shows you understand Kotlin's type system rather than just its syntax.

**📚 Reference:** https://kotlinlang.org/docs/type-aliases.html

---

## 50. Operator overloading

Kotlin maps operators (`+`, `[]`, `in`, `==`, `()`, etc.) to functions with reserved names marked `operator`. You can provide these on your own types (or via extension functions) to make them read naturally.

```kotlin
data class Vec(val x: Int, val y: Int) {
    operator fun plus(o: Vec) = Vec(x + o.x, y + o.y)   // a + b
    operator fun get(i: Int) = if (i == 0) x else y      // v[0]
    operator fun invoke() = "($x, $y)"                    // v()
}

val c = Vec(1, 2) + Vec(3, 4)   // Vec(4, 6)  -> calls plus
val first = c[0]                 // 4          -> calls get
```

Common mappings: `+ - * / %` → `plus/minus/times/div/rem`; `+= ` → `plusAssign`; `[]` → `get`/`set`; `in` → `contains`; `..` → `rangeTo`; `()` → `invoke`; `==` → `equals`; `< > <= >=` → `compareTo`; `++ --` → `inc`/`dec`.

Guidance: only overload when the operator's meaning is **unambiguous** for your type (math types, DSLs, collection-like wrappers). Note `==` already calls `equals` (null-safe), so for data classes you rarely write it yourself.

**Why it matters:** It powers idiomatic APIs (`list[i]`, `range in bounds`, Compose's `Modifier`, coroutine `delay` ranges) and DSLs; interviewers use it to check you won't abuse operators into unreadable code.

**📚 Reference:** https://kotlinlang.org/docs/operator-overloading.html

---

*End of Kotlin language guide. Coroutines, Flow, and concurrency topics are covered in their dedicated files.*
