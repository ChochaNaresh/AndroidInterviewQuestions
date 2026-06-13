# OOP & SOLID — Short Answers (Quick Revision)

> Condensed answers for rapid review before an interview. For full explanations and code see `oop_solid_interview.md`.

---

## 1. What is OOP and its four pillars?

OOP models software as interacting **objects** (state + behaviour). The four pillars: **Abstraction** (expose what, hide how), **Encapsulation** (bundle data with methods, restrict access), **Inheritance** (derive/specialize, "is-a"), **Polymorphism** (one interface, many implementations).

---

## 2. Encapsulation

Bundling data and the methods that operate on it into a class, restricting direct access and exposing a controlled API — protecting invariants. In Kotlin, tune visibility with `private set`; in Java, private fields + getters/setters. It's the basis for ViewModels exposing read-only `StateFlow` over a private `MutableStateFlow`.

---

## 3. Inheritance

A subclass derives fields/behaviour from a superclass, modelling "is-a" and enabling reuse/specialization. Kotlin classes are `final` by default — mark `open` to allow it. Prefer it only for genuine "is-a"; otherwise favour composition.

---

## 4. Polymorphism

A single reference type can refer to different concrete types, with the actual method resolved by real type. Two kinds: **runtime/subtype** (overriding, dynamic dispatch) and **compile-time/ad-hoc** (overloading). It lets you code against an abstraction and swap implementations; it underpins LSP and DIP.

---

## 5. Abstraction

Exposing only essential high-level behaviour while hiding implementation, via `interface`/`abstract class`. **Abstraction vs Encapsulation:** abstraction is about design (hide complexity behind a contract); encapsulation is about implementation (restrict access to state). Callers depend on the contract, enabling testing and module boundaries.

---

## 6. Abstract class vs Interface

Abstract class: can hold state, constructors, and concrete + abstract methods; cannot be instantiated; only **one** can be extended. Interface: a contract (with default methods) describing capabilities; no mutable state/constructor; a class can implement **many**. Use an abstract class to share state/behaviour among related types; an interface to express a capability across unrelated types. Diamond conflict → override and use `super<Interface>.method()`.

---

## 7. Method overloading vs overriding

**Overloading** — same name, different parameter list, same class, resolved at **compile time** (return type alone doesn't distinguish). **Overriding** — subclass redefines an inherited method with the same signature, resolved at **runtime** (dynamic dispatch). Overriding powers runtime polymorphism; overloading is caller convenience.

---

## 8. Access modifiers

- **Java:** `private` (class) < default/package-private < `protected` (subclass + package) < `public`.
- **Kotlin:** `private` (class/file), `protected` (class + subclasses, no package), `internal` (same module), `public` (default).

Kotlin defaults to `public`, has no package-private (replaced by `internal`), and `protected` doesn't grant package access. `internal` is key for hiding implementation across Gradle modules.

---

## 9. Can an interface implement (extend) another interface?

**Yes** — an interface can extend one or more interfaces (Java `extends`, Kotlin `:`), enlarging the contract; it can add members but not remove inherited ones. A class implements many interfaces but extends only one (abstract) class. This supports the Interface Segregation Principle.

---

## 10. Composition vs Inheritance

Inheritance = "is-a", tight coupling to the superclass. Composition = "has-a", a class holds and delegates to other objects, looser coupling and swappable parts (Kotlin's `by` makes it first-class). Guideline: **favour composition over inheritance** to avoid fragile, rigid hierarchies. Decision test: cat *is an* animal vs person *has a* job.

---

## 11. static, static method overriding, constructor inheritance

- **`static`** members belong to the class, not an instance; Kotlin uses `companion object`/top-level/`object` instead.
- **Static methods can't be overridden** — they use static dispatch; a same-signature subclass method is **hiding**, not overriding.
- **Constructors aren't inherited** — only members are; a subclass constructor must call `super(...)`/`: Super(...)`.

Key insight: polymorphism/inheritance apply to instance members, not statics or constructors.

---

## 12. The String Pool in Java

A JVM heap area holding one shared copy of each distinct **string literal** (possible because `String` is immutable). Equal literals reuse the same reference; `new String("...")` forces a new object (bypasses the pool); `.intern()` returns the pooled reference. Use `equals()` (content) not `==` (reference). Kotlin's `==` maps to `equals()`, `===` is reference equality.

---

## 13. SOLID overview

Five OO design principles: **S**ingle Responsibility (one reason to change), **O**pen/Closed (open for extension, closed for modification), **L**iskov Substitution (subtypes substitutable for base types), **I**nterface Segregation (many small interfaces over one fat one), **D**ependency Inversion (depend on abstractions). They justify Clean Architecture, MVVM, DI, and Repository.

---

## 14. S — Single Responsibility Principle (SRP)

A class should have only one reason to change — one responsibility. Split a class that fetches, parses, and persists into separate `Api`, `Parser`, and `Dao` classes composed by a repository. Result: each changes for one reason and is independently testable. SRP is why Android separates ViewModel, Repository, and DataSource.

---

## 15. O — Open/Closed Principle (OCP)

Entities should be open for extension but closed for modification — add behaviour by adding code, not editing tested code. Instead of an `AreaCalculator` with a `when` over shape types, define a `Shape` interface; new shapes implement it without touching the calculator. Powers polymorphic `RecyclerView` view types and strategy interfaces.

---

## 16. L — Liskov Substitution Principle (LSP)

Subtype objects must be replaceable for their base type without breaking correctness (no strengthened preconditions, weakened postconditions, or surprising exceptions). Classic violation: `Square` extending `Rectangle` breaks area expectations. Fix: model them as independent `Shape`s. A `ReadOnlyList` that throws on `add()` also violates LSP.

---

## 17. I — Interface Segregation Principle (ISP)

Clients shouldn't depend on methods they don't use — prefer many small, role-specific interfaces over one fat one. A `MultiFunctionDevice` forcing a simple printer to stub `scan`/`fax` is a smell; split into `Printer`, `Scanner`, `Fax`. Maps to small Android listeners (`OnClickListener` vs `OnLongClickListener`).

---

## 18. D — Dependency Inversion Principle (DIP)

High-level and low-level modules should both depend on **abstractions**, not each other; abstractions shouldn't depend on details. Instead of a ViewModel instantiating a concrete API, define a `UserDataSource` interface and inject the implementation — allowing fakes in tests. DIP is the foundation of dependency injection (Hilt/Dagger/Koin); DI is the technique that implements the principle.

---

## 19. Aggregation vs Composition

**Composition** is a strong "has-a" relationship (child dies with parent). **Aggregation** is a weak "has-a" relationship (child can outlive parent). Example: A Car owns its Engine (Composition), but merely references a Driver (Aggregation).

---

## 20. Cohesion vs Coupling

**Cohesion** measures how focused a single class's responsibilities are (high cohesion = good, aligns with SRP). **Coupling** measures how dependent classes are on each other (low coupling = good, changing one doesn't break others).

---

## 21. The Law of Demeter (Principle of Least Knowledge)

An object should only talk to its immediate friends, not strangers. Avoid "train wrecks" like `user.getAddress().getCity().getZipCode()`. Instead, `user.getZipCode()` should internally delegate. This prevents deep coupling to internal structures.

---

## 22. What is an Anemic Domain Model?

An anti-pattern where domain objects are just bags of data (getters/setters) and all business logic is in external "Service" classes. A **Rich Domain Model** encapsulates both state and the behaviour that modifies it, protecting invariants.

---

## 23. Abstraction vs Encapsulation

**Abstraction** hides complexity by providing a simple interface (focuses on *what* an object does). **Encapsulation** bundles data with methods and restricts access to internal state (focuses on protecting *how* data is managed). Analogy: Abstraction is using a steering wheel to drive; Encapsulation is the hood protecting the engine from being tampered with.

---

## 24. Static vs Dynamic Binding (Early vs Late Binding)

**Static binding** resolves method calls at compile-time (used for static, private, final methods, and overloading). **Dynamic binding** resolves method calls at runtime based on the actual object type (used for overridden methods, powering polymorphism). 

---

## 25. Upcasting vs Downcasting

**Upcasting** is casting a child object to a parent reference; it is implicit and always safe. **Downcasting** is casting a parent reference back to a child type; it is explicit, potentially unsafe, and requires a type check (e.g., `is` or `instanceof`) to avoid `ClassCastException`.

---

## 26. Stateful vs Stateless Objects

**Stateful objects** hold mutable data that can change over time; they require synchronization in multithreading. **Stateless objects** hold no mutable data and return results based solely on input arguments; they are inherently thread-safe and easy to test.

---

## 27. Is-A vs Has-A Relationships

**Is-A** represents inheritance (a `Dog` is an `Animal`), creating tight coupling. **Has-A** represents composition/aggregation (a `Car` has an `Engine`), delegating work to another object. The golden rule is to favour "Has-A" (composition) over "Is-A" (inheritance) for flexible design.
