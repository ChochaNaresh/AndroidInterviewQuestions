# OOP & SOLID Principles — Android Interview Guide

A comprehensive, interview-ready reference for Object-Oriented Programming fundamentals and the SOLID design principles, written from an Android (Kotlin/Java) perspective. Each answer includes a clear definition, code, and a short "why it matters" so you can both recite the concept and defend it in a follow-up. Accurate as of 2026.

---

## Table of Contents

**OOP Fundamentals**
1. [What is OOP and its four pillars?](#1-what-is-oop-and-its-four-pillars)
2. [Encapsulation](#2-encapsulation)
3. [Inheritance](#3-inheritance)
4. [Polymorphism](#4-polymorphism)
5. [Abstraction](#5-abstraction)
6. [Abstract class vs Interface](#6-abstract-class-vs-interface)
7. [Method overloading vs overriding](#7-method-overloading-vs-overriding)
8. [Access modifiers](#8-access-modifiers)
9. [Can an interface implement (extend) another interface?](#9-can-an-interface-implement-extend-another-interface)
10. [Composition vs Inheritance](#10-composition-vs-inheritance)
11. [static, static method overriding, constructor inheritance](#11-static-static-method-overriding-constructor-inheritance)
12. [The String Pool in Java](#12-the-string-pool-in-java)

**SOLID Principles**
13. [SOLID overview](#13-solid-overview)
14. [S — Single Responsibility Principle (SRP)](#14-s--single-responsibility-principle-srp)
15. [O — Open/Closed Principle (OCP)](#15-o--openclosed-principle-ocp)
16. [L — Liskov Substitution Principle (LSP)](#16-l--liskov-substitution-principle-lsp)
17. [I — Interface Segregation Principle (ISP)](#17-i--interface-segregation-principle-isp)
18. [D — Dependency Inversion Principle (DIP)](#18-d--dependency-inversion-principle-dip)

**Advanced OOP Concepts**
19. [Aggregation vs Composition](#19-aggregation-vs-composition)
20. [Cohesion vs Coupling](#20-cohesion-vs-coupling)
21. [The Law of Demeter (Principle of Least Knowledge)](#21-the-law-of-demeter-principle-of-least-knowledge)
22. [What is an Anemic Domain Model?](#22-what-is-an-anemic-domain-model)
23. [Abstraction vs Encapsulation](#23-abstraction-vs-encapsulation)

---

## 1. What is OOP and its four pillars?

**Object-Oriented Programming (OOP)** is a paradigm that models software as a collection of **objects** — bundles of state (fields) and behaviour (methods) — that interact with each other. It is the foundation of Android development: `Activity`, `View`, `ViewModel`, and your domain models are all classes/objects.

The four pillars:

| Pillar | One-line definition |
|---|---|
| **Abstraction** | Expose *what* an object does, hide *how* it does it. |
| **Encapsulation** | Bundle data with the methods that operate on it, and restrict direct access. |
| **Inheritance** | Derive a new class from an existing one to reuse and specialize behaviour ("is-a"). |
| **Polymorphism** | One interface, many implementations; the same call behaves differently per type. |

**Why it matters:** Interviewers use this as a warm-up. Name all four, then be ready to drill into any one with a code example (the next questions).

**📚 Reference:** [GoF / Design Patterns](https://en.wikipedia.org/wiki/Design_Patterns)

---

## 2. Encapsulation

**Definition:** Encapsulation bundles data (fields) and the methods that operate on that data into a single unit (a class) and restricts direct access to the internal state, exposing it only through a controlled API (getters/setters or methods). This protects invariants — the object stays in a valid state.

**Kotlin:**

```kotlin
class BankAccount(initialBalance: Long) {
    // Private backing field — cannot be set to an invalid value from outside
    var balance: Long = initialBalance
        private set            // public getter, private setter

    fun deposit(amount: Long) {
        require(amount > 0) { "Amount must be positive" }
        balance += amount
    }

    fun withdraw(amount: Long) {
        require(amount in 1..balance) { "Invalid withdrawal" }
        balance -= amount
    }
}

val account = BankAccount(100)
account.deposit(50)        // OK
// account.balance = -999  // Compile error: setter is private — invariant protected
```

In Kotlin, `val`/`var` properties generate getters/setters automatically; you tune visibility with `private set`. In Java you write explicit private fields plus public getters/setters.

**Why it matters:** Without encapsulation, any caller could put an object into an illegal state (negative balance, half-initialized model). It is also the basis for `private` fields in `ViewModel`s exposing read-only `StateFlow`/`LiveData` while keeping the mutable backing field private.

---

## 3. Inheritance

**Definition:** Inheritance lets a subclass derive fields and behaviour from a superclass, modelling an **"is-a"** relationship and enabling code reuse and specialization.

**Kotlin** (classes are `final` by default — you must mark them `open`):

```kotlin
open class Animal(val name: String) {
    open fun sound(): String = "..."
}

class Dog(name: String) : Animal(name) {
    override fun sound(): String = "Woof"   // specialize
}

val a: Animal = Dog("Rex")
println(a.sound())   // Woof
```

**Java:**

```java
class Animal {
    protected final String name;
    Animal(String name) { this.name = name; }
    String sound() { return "..."; }
}
class Dog extends Animal {
    Dog(String name) { super(name); }
    @Override String sound() { return "Woof"; }
}
```

**Why it matters:** Inheritance is powerful but easily abused. Prefer it only for genuine "is-a" relationships; otherwise favour **composition** (see Q10). Kotlin's `final`-by-default encourages this — you opt into inheritance deliberately with `open`.

---

## 4. Polymorphism

**Definition:** Polymorphism ("many forms") means a single interface or reference type can refer to objects of different concrete types, and the actual method invoked is resolved by the object's real type. Two kinds:

- **Runtime / subtype polymorphism** — method **overriding**, resolved via dynamic dispatch at runtime.
- **Compile-time / ad-hoc polymorphism** — method **overloading**, resolved by the compiler from the argument signature.

```kotlin
interface Shape { fun area(): Double }
class Circle(val r: Double) : Shape { override fun area() = Math.PI * r * r }
class Square(val s: Double) : Shape { override fun area() = s * s }

fun printArea(shape: Shape) = println(shape.area())  // works for any Shape

printArea(Circle(2.0))   // 12.566...
printArea(Square(3.0))   // 9.0
```

**Why it matters:** It is what lets you write code against an abstraction (`Shape`, `Repository`, `RecyclerView.Adapter`) and swap implementations without changing callers. It underpins LSP, DIP, and most of Android's framework design.

---

## 5. Abstraction

**Definition:** Abstraction exposes only the essential, high-level behaviour of an object while hiding implementation details. In Java/Kotlin it is expressed via `interface` and `abstract class`.

```kotlin
abstract class Repository<T> {
    abstract suspend fun getById(id: String): T   // what — no how

    // Shared concrete behaviour
    open suspend fun exists(id: String): Boolean = runCatching { getById(id) }.isSuccess
}

class UserRepository(private val api: Api) : Repository<User>() {
    override suspend fun getById(id: String): User = api.fetchUser(id)
}
```

**Abstraction vs Encapsulation (common follow-up):** Abstraction is about *design* — hiding complexity behind a simple contract (what to expose). Encapsulation is about *implementation* — restricting access to internal state (how to protect it). They complement each other.

**Why it matters:** Callers depend on the abstract contract, not the concrete class, which is exactly what makes testing (fakes/mocks) and module boundaries possible.

---

## 6. Abstract class vs Interface

**Definitions:**

- An **abstract class** is a class that may contain both concrete (implemented) and abstract (unimplemented) methods, can hold state (fields), can have constructors, and **cannot be instantiated** — it must be extended. A class can extend **only one** abstract class.
- An **interface** is a contract: a set of method/property signatures (and, since Java 8 / always in Kotlin, default method implementations) describing what implementers must provide. It cannot hold mutable state (Kotlin interface properties have no backing field). A class can implement **many** interfaces.

| Aspect | Abstract class | Interface |
|---|---|---|
| Instantiable | No | No |
| Multiple inheritance | One only | Many |
| State / backing fields | Yes | No (Kotlin: properties without backing field) |
| Constructor | Yes | No |
| Default method bodies | Yes | Yes (Java 8+, Kotlin always) |
| Use when | Share state + base behaviour among closely related types | Define a capability/role multiple unrelated types can fulfil |

```kotlin
// Interface — a capability; default method allowed
interface Clickable {
    fun onClick()
    fun showTooltip() = println("Default tooltip")   // default body
}

// Abstract class — shared state + partial implementation
abstract class UiComponent(val id: String) {
    abstract fun render()
    fun log() = println("Rendering $id")              // concrete, uses state
}

class Button(id: String) : UiComponent(id), Clickable {
    override fun render() = log()
    override fun onClick() = println("Clicked $id")
}
```

**Diamond conflict (Kotlin follow-up):** if two interfaces provide the same default method, the implementer must override and disambiguate with `super<InterfaceName>.method()`.

**Why it matters:** Choose an interface to express "can do X" across unrelated types, and an abstract class to share real implementation/state among a family of related types. Kotlin favours interfaces (with defaults) for flexibility.

**📚 Reference:** README lines 1015–1021, OOP.md Q14

---

## 7. Method overloading vs overriding

**Definitions:**

- **Overloading** — multiple methods in the *same* class share the same name but differ in parameter list (number/type/order). Resolved at **compile time** (static/ad-hoc polymorphism). Return type alone does **not** distinguish overloads.
- **Overriding** — a subclass provides a new implementation of a method inherited from a superclass/interface with the *same signature*. Resolved at **runtime** (dynamic dispatch).

```kotlin
class Printer {
    // Overloading — same name, different signatures
    fun print(value: Int) = println("Int: $value")
    fun print(value: String) = println("String: $value")
    fun print(a: Int, b: Int) = println("Sum: ${a + b}")
}

open class Base { open fun greet() = "Hello from Base" }
class Derived : Base() {
    override fun greet() = "Hello from Derived"   // Overriding
}

val b: Base = Derived()
println(b.greet())   // "Hello from Derived" — resolved at runtime
```

| | Overloading | Overriding |
|---|---|---|
| Where | Same class | Subclass vs superclass |
| Signature | Must differ | Must match |
| Binding | Compile time | Runtime |
| Keyword (Kotlin) | none | `override` (+ `open`/`abstract` on parent) |

**Why it matters:** Overriding is the engine of runtime polymorphism (LSP/DIP rely on it). Overloading is purely a convenience for callers. Mislabeling these is a classic red flag in interviews.

**📚 Reference:** README line 1023, OOP.md Q3

---

## 8. Access modifiers

**Definition:** Access modifiers control the visibility/scope of classes, members, and constructors.

**Java** (strictest → most lenient):

| Modifier | Visible from |
|---|---|
| `private` | Only the declaring class. |
| *default* (package-private, no keyword) | Same package. |
| `protected` | Subclasses + same package. |
| `public` | Everywhere. |

**Kotlin** (default is `public`):

| Modifier | Visible from |
|---|---|
| `private` | Same class (or same file for top-level declarations). |
| `protected` | The declaring class and subclasses (not package — Kotlin has no package-level visibility). |
| `internal` | Anywhere in the same **module** (Gradle module/compilation unit). |
| `public` (default) | Everywhere. |

Key differences from Java: Kotlin defaults to `public` (Java member default is package-private); Kotlin has **no** package-private — it replaces it with `internal` (module-scoped); and Kotlin `protected` does **not** grant package access.

```kotlin
class Config {
    private val secret = "key"          // only inside Config
    internal val buildType = "debug"    // anywhere in this module
    protected open val region = "US"    // Config + subclasses
    val version = "1.0"                 // public by default
}
```

**Why it matters:** Visibility is your primary tool for encapsulation and for defining clean module boundaries. `internal` is heavily used in multi-module Android apps to hide implementation classes from other Gradle modules while keeping them accessible within the module.

**📚 Reference:** README lines 1027–1032, OOP.md Q2

---

## 9. Can an interface implement (extend) another interface?

**Yes.** An interface can extend one or more other interfaces, inheriting their abstract members. In **Java** the keyword is `extends` (an interface uses `extends`, not `implements`, because it is not providing implementations — only enlarging the contract). In **Kotlin** there are no `extends`/`implements` keywords at all; you use a colon `:` for both class inheritance and interface conformance. You can add new members to the sub-interface freely but cannot remove inherited ones.

**Java:**

```java
interface Readable  { String read(); }
interface Writable  { void write(String data); }
interface ReadWrite extends Readable, Writable {   // interface extends interfaces
    void flush();
}
```

**Kotlin:**

```kotlin
interface Readable { fun read(): String }
interface Writable { fun write(data: String) }
interface ReadWrite : Readable, Writable {          // colon, multiple parents
    fun flush()
}

class File : ReadWrite {
    override fun read() = "data"
    override fun write(data: String) {}
    override fun flush() {}
}
```

A **class** can also implement multiple interfaces but extend only one (abstract) class.

**Why it matters:** Interface composition lets you build small focused contracts and combine them — directly supporting the Interface Segregation Principle (Q17).

**📚 Reference:** README lines 1034–1035, OOP.md Q4

---

## 10. Composition vs Inheritance

**Definitions:**

- **Inheritance** ("is-a") — a class extends another, inheriting its API and behaviour. Tight coupling: the subclass depends on the superclass's internals.
- **Composition** ("has-a") — a class holds instances of other classes and delegates work to them. Looser coupling, swappable parts.

```kotlin
// Composition — Car HAS-A Engine; swap engines without touching Car's hierarchy
interface Engine { fun start(): String }
class PetrolEngine : Engine { override fun start() = "Vroom" }
class ElectricEngine : Engine { override fun start() = "Hum" }

class Car(private val engine: Engine) {      // delegates to the composed object
    fun start() = engine.start()
}

Car(ElectricEngine()).start()   // "Hum"
```

Kotlin's **delegation** (`by`) makes composition first-class:

```kotlin
interface Logger { fun log(msg: String) }
class ConsoleLogger : Logger { override fun log(msg: String) = println(msg) }

class Service(logger: Logger) : Logger by logger   // delegates Logger to the field
```

**Decision rule:** Use the "is-a" vs "has-a" test. A cat *is an* animal (inheritance); a person *has a* job (composition). The widely cited guideline is **"favour composition over inheritance"** because composition avoids fragile base-class problems and rigid hierarchies.

**Why it matters:** Over-using inheritance creates deep, brittle hierarchies. Composition keeps designs flexible and is essential to OCP and DIP.

**📚 Reference:** OOP.md Q11

---

## 11. static, static method overriding, constructor inheritance

- **`static` (Java):** static members belong to the **class**, not an instance — shared across all instances and accessible without one. Kotlin has no `static` keyword; you use `companion object`, top-level declarations, or `object` (singleton) instead.

  ```kotlin
  class MathUtils {
      companion object {
          fun square(x: Int) = x * x   // call as MathUtils.square(4)
      }
  }
  ```

- **Can a static method be overridden?** **No.** Static methods are bound at compile time to the declaring class (static dispatch), so they are not polymorphic. A subclass can declare a static method with the same signature, but that is **hiding** (method hiding), not overriding. In Kotlin, companion-object functions likewise are not overridable the way instance methods are.

- **Can a constructor be inherited?** **No.** Constructors are not members of a class, and only members are inherited. A subclass constructor must call a superclass constructor (`super(...)` in Java, `: Super(...)` in Kotlin), but the constructor itself is not inherited.

**Why it matters:** These are common "gotcha" questions. The key insight is that polymorphism and inheritance apply to **instance members**, not to statics or constructors.

**📚 Reference:** OOP.md Q5, Q6, Q7

---

## 12. The String Pool in Java

**Definition:** The **String Pool** (a.k.a. string intern pool) is a special storage area in the JVM heap where the runtime keeps a single shared copy of each distinct **string literal**. When you write a literal, the JVM checks the pool: if an equal string already exists it returns that reference; otherwise it adds the new one. This is possible because `String` is **immutable**.

```java
String a = "hello";            // created in the pool
String b = "hello";            // reuses the same pooled instance
System.out.println(a == b);    // true  — same reference

String c = new String("hello"); // forces a new heap object, NOT pooled
System.out.println(a == c);     // false — different reference
System.out.println(a.equals(c));// true  — same content

String d = c.intern();          // returns the pooled reference
System.out.println(a == d);     // true
```

- `==` compares **references**; `equals()` compares **content**. Always use `equals()` for string comparison.
- `new String("...")` deliberately bypasses the pool; `.intern()` puts/finds the value in the pool.
- Since Java 7 the pool lives in the regular heap (not PermGen), so it is garbage-collected.

**Kotlin note:** Kotlin string literals compile to the same JVM strings and share the pool. Kotlin's `==` maps to `.equals()` (structural equality) and `===` is reference equality.

**Why it matters:** It explains the classic `==` vs `equals()` trap and why string literals are memory-efficient. Interning is a memory optimization but overusing `intern()` on dynamic strings can bloat the pool.

**📚 Reference:** README line 1025

---

## 13. SOLID overview

**SOLID** is a set of five object-oriented design principles (popularized by Robert C. Martin) that make software easier to maintain, extend, and test:

| Letter | Principle | Core idea |
|---|---|---|
| **S** | Single Responsibility | A class should have one reason to change. |
| **O** | Open/Closed | Open for extension, closed for modification. |
| **L** | Liskov Substitution | Subtypes must be substitutable for their base types. |
| **I** | Interface Segregation | Many small client-specific interfaces beat one fat one. |
| **D** | Dependency Inversion | Depend on abstractions, not concretions. |

**Why it matters:** SOLID is the most common architecture question in mid/senior Android interviews. Clean Architecture, MVVM, Hilt/Dagger DI, and Repository patterns are all justified by these principles. Know each by name, a memorable analogy, and a before/after code example.

**📚 References (Outcome School posts):**
[SRP](https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineering-tech-activity-7234425076308164611-sY57) ·
[OCP](https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineering-tech-activity-7287026079851061248-bjdM) ·
[LSP](https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineering-tech-activity-7300845161725448193-iCnO) ·
[ISP](https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineer-tech-activity-7301459048292397056--9bL) ·
[DIP](https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineer-tech-activity-7303987022493270016-VrZL)

---

## 14. S — Single Responsibility Principle (SRP)

**Definition:** A class should have **only one reason to change** — i.e. one responsibility. *"One chef cannot run the whole restaurant."*

**Violation** — a class doing fetching, parsing, and persisting:

```kotlin
// BAD: three reasons to change (network format, JSON schema, DB schema)
class UserManager {
    fun fetchAndSaveUser(id: String) {
        val json = HttpClient().get("/users/$id")   // networking
        val user = parseJson(json)                   // parsing
        Database.insert(user)                        // persistence
    }
    private fun parseJson(json: String): User = /* ... */ TODO()
}
```

**Fixed** — split responsibilities, compose them:

```kotlin
class UserApi(private val http: HttpClient) {
    fun fetch(id: String): String = http.get("/users/$id")
}
class UserParser {
    fun parse(json: String): User = /* ... */ TODO()
}
class UserDao(private val db: Database) {
    fun save(user: User) = db.insert(user)
}

class UserRepository(
    private val api: UserApi,
    private val parser: UserParser,
    private val dao: UserDao,
) {
    fun refresh(id: String) = dao.save(parser.parse(api.fetch(id)))
}
```

**Why it matters:** Each class now changes for exactly one reason; you can unit-test parsing without a network, swap the DB without touching the parser. SRP is why Android architecture separates `ViewModel`, `Repository`, and `DataSource`.

**📚 Reference:** README line 1005, OOP.md Q12

---

## 15. O — Open/Closed Principle (OCP)

**Definition:** Software entities (classes, modules, functions) should be **open for extension but closed for modification** — add new behaviour by adding code, not by editing existing, tested code. *"Trying new shoes doesn't require you to saw your feet off."*

**Violation** — adding a shape means editing `AreaCalculator` every time:

```kotlin
// BAD: must modify this when/if a new shape is introduced
class AreaCalculator {
    fun area(shape: Any): Double = when (shape) {
        is Circle -> Math.PI * shape.r * shape.r
        is Square -> shape.s * shape.s
        else -> throw IllegalArgumentException()
    }
}
```

**Fixed** — abstract over an interface; new shapes extend without modifying the calculator:

```kotlin
interface Shape { fun area(): Double }
class Circle(val r: Double) : Shape { override fun area() = Math.PI * r * r }
class Square(val s: Double) : Shape { override fun area() = s * s }
class Triangle(val b: Double, val h: Double) : Shape { override fun area() = 0.5 * b * h }  // NEW, no edits elsewhere

class AreaCalculator {
    fun totalArea(shapes: List<Shape>): Double = shapes.sumOf { it.area() }
}
```

**Why it matters:** You extend behaviour by adding `Triangle` — `AreaCalculator` stays untouched and unbroken. This is why Android uses polymorphic `RecyclerView.Adapter` view types, strategy interfaces, and sealed-class-driven handlers.

**📚 Reference:** README line 1006, OOP.md Q12

---

## 16. L — Liskov Substitution Principle (LSP)

**Definition:** Objects of a supertype should be **replaceable with objects of a subtype without breaking** the correctness of the program. A subtype must honour the behavioural contract of its parent (no strengthening preconditions, no weakening postconditions, no surprising exceptions).

**Violation** — the classic Rectangle/Square problem:

```kotlin
open class Rectangle(var width: Int, var height: Int) {
    open fun setW(w: Int) { width = w }
    open fun setH(h: Int) { height = h }
    fun area() = width * height
}

// Square "is-a" Rectangle? It breaks the contract:
class Square(side: Int) : Rectangle(side, side) {
    override fun setW(w: Int) { width = w; height = w }
    override fun setH(h: Int) { width = h; height = h }
}

fun resizeAndCheck(r: Rectangle) {
    r.setW(5); r.setH(4)
    check(r.area() == 20) { "Expected 20, got ${r.area()}" }
}
resizeAndCheck(Rectangle(0, 0))  // OK -> 20
resizeAndCheck(Square(0))        // FAILS -> area is 16: substitution broke the program
```

**Fixed** — don't force an "is-a" that isn't behaviourally true; model them as independent shapes:

```kotlin
interface Shape { fun area(): Int }
class Rectangle(val width: Int, val height: Int) : Shape { override fun area() = width * height }
class Square(val side: Int) : Shape { override fun area() = side * side }
// No subtype claims to be substitutable for the other; no broken expectations.
```

**Why it matters:** LSP keeps polymorphism trustworthy — if a subtype can silently violate caller assumptions, every `Base x = subtype` becomes a landmine. A common Android example: a `ReadOnlyList` subtype that throws on `add()` violates LSP if callers expect a mutable `List`.

**📚 Reference:** README line 1007, OOP.md Q12

---

## 17. I — Interface Segregation Principle (ISP)

**Definition:** **Clients should not be forced to depend on methods they do not use.** Prefer many small, role-specific interfaces over one large general-purpose one.

**Violation** — a fat interface forces unrelated implementations to stub methods:

```kotlin
// BAD: a simple printer is forced to implement scan/fax it can't do
interface MultiFunctionDevice {
    fun print(doc: String)
    fun scan(doc: String)
    fun fax(doc: String)
}

class OldPrinter : MultiFunctionDevice {
    override fun print(doc: String) = println("Printing")
    override fun scan(doc: String) = throw UnsupportedOperationException()  // smell
    override fun fax(doc: String) = throw UnsupportedOperationException()   // smell
}
```

**Fixed** — split into focused capabilities; implementers pick only what they support:

```kotlin
interface Printer { fun print(doc: String) }
interface Scanner { fun scan(doc: String) }
interface Fax     { fun fax(doc: String) }

class SimplePrinter : Printer {
    override fun print(doc: String) = println("Printing")
}
class AllInOne : Printer, Scanner, Fax {
    override fun print(doc: String) = println("Printing")
    override fun scan(doc: String) = println("Scanning")
    override fun fax(doc: String) = println("Faxing")
}
```

**Why it matters:** Clients depend only on what they need, so changes to `Scanner` never ripple into `SimplePrinter`. In Android this maps to small listener interfaces (e.g. separate `OnClickListener` vs `OnLongClickListener`) rather than one bloated callback.

**📚 Reference:** README line 1008, OOP.md Q12

---

## 18. D — Dependency Inversion Principle (DIP)

**Definition:** High-level modules should not depend on low-level modules; **both should depend on abstractions**. And abstractions should not depend on details — details depend on abstractions. *"Program to an interface, not an implementation. You wouldn't wire a lamp directly to your house."*

**Violation** — `ViewModel` (high-level) directly instantiates a concrete data source (low-level):

```kotlin
// BAD: hard dependency on a concrete class; impossible to swap or fake in tests
class RemoteUserApi {
    fun load(): User = /* real network call */ TODO()
}
class UserViewModel {
    private val api = RemoteUserApi()       // tightly coupled
    fun user() = api.load()
}
```

**Fixed** — depend on an abstraction, inject the implementation:

```kotlin
interface UserDataSource { fun load(): User }                 // abstraction

class RemoteUserApi : UserDataSource { override fun load(): User = TODO() }
class FakeUserApi   : UserDataSource { override fun load() = User("Test") }  // for tests

class UserViewModel(private val source: UserDataSource) {     // depends on abstraction
    fun user() = source.load()
}

// Production
UserViewModel(RemoteUserApi())
// Unit test
UserViewModel(FakeUserApi())
```

**Why it matters:** DIP is the foundation of **dependency injection** (Hilt/Dagger/Koin) and testability in Android. By inverting the dependency, `UserViewModel` no longer knows or cares whether data comes from network, cache, or a fake — you swap implementations at the composition root. Note: *dependency inversion* (the principle) is what *dependency injection* (the technique) implements.

**📚 Reference:** README line 1009, OOP.md Q12

---

## 19. Aggregation vs Composition

**Definition:** Both are forms of association ("has-a" relationships), but they differ in **lifecycle dependency**.

- **Composition:** A strong "has-a" relationship where the child object's lifecycle is bound to the parent. If the parent is destroyed, the child is destroyed. (e.g., A `House` and a `Room`).
- **Aggregation:** A weak "has-a" relationship where the child object can exist independently of the parent. (e.g., A `Department` and a `Teacher`).

**Kotlin:**

```kotlin
// Composition: Engine is created by Car and dies with Car
class Car {
    private val engine = Engine() 
}

// Aggregation: Teacher is passed into Department and can exist outside of it
class Department(val teachers: List<Teacher>)
```

**Why it matters:** Understanding this distinction helps in designing proper object lifecycles and garbage collection. In Android, a `Fragment` has a composition relationship with its Views, but an aggregation relationship with its `ViewModel`.

---

## 20. Cohesion vs Coupling

**Definitions:**

- **Cohesion:** Measures how closely related and focused the responsibilities of a single module/class are. **High cohesion** is good — a class should do one thing well (aligns with SRP).
- **Coupling:** Measures the degree of interdependence between different modules/classes. **Low coupling** is good — changing one class shouldn't require changing others.

**Why it matters:** You want **high cohesion and low coupling**. Highly cohesive classes are easier to understand and test. Loosely coupled classes are easier to change and reuse without causing ripple effects across the codebase.

---

## 21. The Law of Demeter (Principle of Least Knowledge)

**Definition:** A design guideline stating that an object should only talk to its immediate friends and not to strangers. "Don't talk to strangers." 

Specifically, a method `M` of object `O` should only invoke methods of:
1. `O` itself.
2. Objects passed as arguments to `M`.
3. Objects instantiated within `M`.
4. Direct component objects of `O`.

**Violation (Train Wreck):**
```kotlin
val zipCode = user.getAddress().getCity().getZipCode() // Bad: deep coupling
```

**Fixed:**
```kotlin
val zipCode = user.getZipCode() // Good: User delegates internally
```

**Why it matters:** It reduces tight coupling. If the internal structure of `Address` changes, you don't want `User`'s callers to break. It promotes encapsulation and delegation.

---

## 22. What is an Anemic Domain Model?

**Definition:** An **Anemic Domain Model** is an anti-pattern where domain objects are just bags of getters and setters (data structures) with no business logic or behaviour. All the logic is stripped out and placed in external "Service" or "Manager" classes.

**Violation:**
```kotlin
// Anemic model: just data
class User {
    var isPremium: Boolean = false
}

// Logic outside the object
class UserService {
    fun upgradeUser(user: User) {
        user.isPremium = true
    }
}
```

**Rich Domain Model (Fixed):**
```kotlin
// Rich model: state + behaviour encapsulated
class User(var isPremium: Boolean = false) {
    fun upgrade() {
        this.isPremium = true
    }
}
```

**Why it matters:** Anemic models violate encapsulation — the state is exposed and manipulated from the outside. A Rich Domain Model bundles the state and the behaviour that mutates it, making the object responsible for its own invariants.

---

## 23. Abstraction vs Encapsulation

**Definitions & Differences:**
While both concepts help in hiding details, they solve different problems at different stages of design.

- **Abstraction** is about **design** (focusing on the *outside* view). It hides the complexity of an operation by providing a simple interface. You expose *what* the object does, but hide *how* it does it.
- **Encapsulation** is about **implementation** (focusing on the *inside* view). It bundles data and methods together and restricts direct access to the internal state. You protect the integrity of the data.

**Analogy:**
- **Abstraction:** When you drive a car, you use the steering wheel and pedals. You don't need to know how the combustion engine works to drive it.
- **Encapsulation:** The engine has a physical hood over it. You cannot reach in and manually mix the fuel and air while it's running; the car protects its internal moving parts from external interference.

**Why it matters:** They complement each other. Abstraction allows clients to use your classes easily without needing to understand their internal complexity, while encapsulation ensures that clients cannot misuse or break the internal state of those classes.
