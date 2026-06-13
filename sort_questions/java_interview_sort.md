# Java — Short Answers (Quick Revision)

> Condensed answers for rapid review before an interview. For full explanations and code see `java_interview.md`.

---

## 1. How is the String class implemented and why is it immutable?

`String` is a final class wrapping a `final` backing array (a `byte[]` + coder flag since Java 9's Compact Strings) that's never exposed, so the value can't change after construction; "mutating" methods return a new `String`. Immutable for security (TOCTOU safety), thread-safety, safe pooling/interning, and `hashCode` caching (ideal map key).

---

## 2. What is the String pool?

A heap region storing one canonical instance of each distinct string literal, so identical literals refer to the same object (`==` is true). `new String(...)` bypasses the pool; `intern()` returns the pooled instance. Compile-time constant concatenations are folded and pooled; runtime concatenation isn't. Prefer `.equals()` for content comparison.

---

## 3. Difference between Integer and int (and autoboxing)

`int` is a primitive (raw value, no methods, can't be null); `Integer` is a heap wrapper object (utility methods, nullable). Autoboxing converts between them. Use wrappers for null/generics/methods, primitives for performance. Gotcha: `Integer` caches `[-128,127]`, so `==` is true there but false outside — always compare with `.equals()`.

---

## 4. What are the 8 primitive types in Java?

`byte` (8-bit), `short` (16-bit), `int` (32-bit), `long` (64-bit), `float` (32-bit), `double` (64-bit), `char` (16-bit UTF-16 unit), `boolean`. Each has a wrapper (`Byte`, `Short`, `Integer`, `Long`, `Float`, `Double`, `Character`, `Boolean`).

---

## 5. Are objects passed by reference or by value?

Java is always pass-by-value. For primitives the value is copied; for objects the reference is copied by value — both point to the same object, so mutating its fields is visible to the caller, but reassigning the parameter to a new object is not.

---

## 6. What is the Garbage Collector and how does it work?

The GC automatically reclaims heap memory from objects unreachable from GC roots (thread stacks, statics, JNI). It traces reachability (handles cycles correctly — no reference counting), uses the generational hypothesis (Young: Eden+Survivors, Old), does cheap Minor GCs / expensive Major GCs, and Mark–Sweep–Compact. Collectors: G1 (default), ZGC, Shenandoah; ART on Android. `System.gc()` is only a hint. Reference strengths: strong, soft (caches), weak, phantom.

---

## 7. The Java Memory Model and happens-before

The JMM defines when one thread's write becomes visible to another and what reorderings are allowed. The core concept is **happens-before**: if A happens-before B, A's effects are visible to B. Established by: program order, monitor unlock→lock, volatile write→read, `Thread.start()`/`join()`, and final fields after safe publication. Without it, stale reads/reorderings occur.

---

## 8. What does synchronized mean?

Provides mutual exclusion and visibility by acquiring an object's intrinsic lock (monitor); only one thread holds it, and entering establishes happens-before with the prior release. Instance method locks `this`; static method locks the `Class`; a block locks the named object. Reentrant; overuse causes contention.

---

## 9. What is the volatile modifier?

Marks a field as always read/written from main memory (never a thread-local cache), guaranteeing visibility and preventing reordering, but **not** atomicity for compound ops. A volatile write happens-before later reads; single reads/writes (incl. `long`/`double`) are atomic. `count++` is still unsafe — use `synchronized`/`AtomicInteger`. Good for flags and double-checked locking.

---

## 10. Object-level lock vs class-level lock

Object-level lock (on an instance, `this`) serializes threads on the same instance; different instances run concurrently. Class-level lock (on the `Class` object) serializes across all instances — guards static state. A thread holding an instance lock doesn't block one acquiring the class lock (different monitors).

---

## 11. Monitor and synchronization

A monitor is the intrinsic lock backing `synchronized`, implemented via `monitorenter`/`monitorexit` bytecodes. It also has a wait set used by `wait()` (releases lock, parks), `notify()`/`notifyAll()` (wake waiters) — all called while holding the monitor. Always re-check conditions in a `while` loop (spurious wakeups). Reentrant; `ReentrantLock` + `Condition` is the flexible alternative.

---

## 12. What is a ThreadPoolExecutor?

The main `ExecutorService` implementation — a reusable pool of worker threads pulling tasks from a queue, capping concurrency and reusing threads. Params: corePoolSize, maximumPoolSize, keepAliveTime, workQueue, threadFactory, rejectedExecutionHandler. Dispatch order: core threads → queue → non-core threads (up to max) → reject. Rejection policies: Abort (default), CallerRuns, Discard, DiscardOldest. Prefer a bounded queue.

---

## 13. Concurrency vs Parallelism

Concurrency is dealing with many tasks in overlapping time (structuring; achievable by interleaving on one core). Parallelism is doing many tasks at the same instant (execution; requires multiple cores). Concurrency is a design concern, parallelism an execution concern — "Concurrency is about structure; parallelism is about execution."

---

## 14. The atomic package: get, set, lazySet, compareAndSet, weakCompareAndSet

Lock-free thread-safe single-variable ops built on CAS. `get()`/`set()` have volatile read/write semantics. `lazySet()` uses weaker release ordering (cheaper, no full happens-before). `compareAndSet(expect, update)` atomically sets only if equal to expect, with full memory effects. `weakCompareAndSet` may fail spuriously and gives no ordering — use in a retry loop. Consider `LongAdder` under high contention.

---

## 15. How do try / catch / finally work?

`try` wraps risky code, `catch` handles matching exceptions, `finally` always runs (even on `return`/`break`/`continue`) — ideal for cleanup. A `return`/`throw` in `finally` overrides the try/catch result (avoid). Only skipped by `System.exit()`, JVM crash, or thread kill. Prefer try-with-resources for `AutoCloseable`.

---

## 16. Checked vs Unchecked exceptions

Checked exceptions extend `Exception` (not `RuntimeException`); the compiler forces catch-or-declare — for recoverable conditions (`IOException`). Unchecked extend `RuntimeException`; no handling required — signal bugs (`NullPointerException`). Errors extend `Error` — serious, unrecoverable (`OutOfMemoryError`), don't catch.

---

## 17. throw vs throws

`throw` is a statement that raises an exception instance at runtime. `throws` is part of the method signature declaring which checked exceptions may propagate, shifting handling to callers. You `throw` one instance; you `throws` multiple class names.

---

## 18. Shallow vs Deep copy

Shallow copy duplicates the top-level object but shares references to nested objects (mutations visible through both); `Object.clone()` is shallow. Deep copy recursively duplicates the whole graph, producing independence. Shallow is cheap but risks shared-state bugs; deep is safe but costlier and must handle cycles (copy constructors preferred).

---

## 19. Serialization and Deserialization

Serialization converts an object's state to a byte stream (persist/send); deserialization rebuilds it — enabled by implementing `Serializable`. `serialVersionUID` versions the class (mismatch → `InvalidClassException`). `transient`/`static` fields aren't serialized; the constructor isn't called on deserialization. Native serialization has security pitfalls — prefer `Parcelable` (Android IPC) or JSON/protobuf.

---

## 20. The transient modifier

Marks an instance field to be excluded from serialization; on deserialization it's restored to its default (`null`/`0`/`false`). Use for sensitive data (passwords/tokens), derived/recomputable fields, or non-serializable resources (connections, threads). No effect outside serialization.

---

## 21. What are anonymous classes?

A class without a name, declared and instantiated in one expression — a one-off implementation of an interface/subclass, typically for listeners/callbacks. No constructor (use an init block); captures effectively-final locals and enclosing instance (implicit reference → Android leak source). For functional interfaces, a lambda is the concise modern replacement.

---

## 22. == vs equals()

`==` compares references for objects (same instance?) and raw values for primitives. `equals()` compares logical equality as defined by the class (default `Object.equals` is `==`, but `String`/`Integer`/collections override it). Use `==` for primitives/identity/`null`, `.equals()` for content; `Objects.equals(a, b)` for null-safety.

---

## 23. The hashCode() and equals() contract

If `a.equals(b)` then `a.hashCode() == b.hashCode()` must hold (reverse not required — collisions allowed); both must be consistent, and `equals` reflexive/symmetric/transitive. Hash collections locate the bucket via `hashCode()` then find the key via `equals()` — override both together or lookups fail. (`record` types generate them automatically.)

---

## 24. final, finally, and finalize

`final` — assign-once variable / non-overridable method / non-subclassable class. `finally` — the always-executing cleanup block in try/catch. `finalize()` — a deprecated (since Java 9; deprecated for removal in Java 18 via JEP 421, but not yet removed as of Java 21), unreliable GC callback; use try-with-resources/`AutoCloseable` or `Cleaner` instead.

---

## 25. The static keyword (variables, methods, blocks)

Binds a member to the class, not an instance. Static variable — one shared copy across all instances. Static method — invoked without an instance, can't use `this` or non-static members. Static block — runs once at class load, for one-time static setup (multiple run in source order). Initialized at class-load time before any instance.

---

## 26. Can a static method be overridden?

No — static methods are **hidden**, not overridden. Overriding uses runtime polymorphism (dynamic dispatch); statics belong to the class and resolve at compile time by the declared reference type (method hiding). You also can't mix a static and instance method with the same signature (compile error).

---

## 27. What is Reflection?

`java.lang.reflect` lets a program inspect/manipulate classes, fields, methods, and constructors at runtime, even unknown ones — including private members via `setAccessible(true)`. Used by DI frameworks, serializers (Gson), JUnit, ORMs. Trade-offs: slower, bypasses type safety/encapsulation, module/ProGuard issues — use sparingly, prefer code generation.

---

## 28. String vs StringBuffer vs StringBuilder

`String` is immutable (concatenation in a loop is O(n²) garbage). `StringBuilder` is mutable, not thread-safe, fastest — the default for single-threaded building. `StringBuffer` is mutable and thread-safe (synchronized), slower — only when a builder is shared across threads.

---

## 29. Fail-fast vs fail-safe iterators

Fail-fast iterators throw `ConcurrentModificationException` on structural modification during iteration (detected via `modCount`) — `ArrayList`, `HashMap`, `HashSet`; iterate the live collection. Fail-safe (weakly consistent) iterate a snapshot/tolerant view and don't throw but may show stale data — `CopyOnWriteArrayList`, `ConcurrentHashMap`. (Use the iterator's own `remove()` to modify safely.)

---

## 30. Generics in Java

Parameterize types for compile-time type safety and to eliminate casts. Support generic methods and bounded type parameters (`<T extends Comparable<T>>`). Wildcards control variance: `? extends Number` (producer, read), `? super Integer` (consumer, write) — PECS. **Type erasure**: generics are compile-time only (erased to bounds/Object), so no `new T[]`, no `instanceof List<String>`, no runtime type info.

---

## 31. Arrays vs ArrayList

Array: fixed size, holds primitives or objects, multi-dimensional, `.length`, faster/no boxing, covariant (runtime `ArrayStoreException`). `ArrayList`: dynamic (auto-resizes ~1.5×, amortized O(1) append), objects only (autoboxes), rich API, `.size()`, generic compile-time safety. Use arrays for fixed-size/perf/primitives; `ArrayList` for varying size/convenience.

---

## 32. HashSet vs TreeSet

`HashSet` — hash table, unordered, O(1) add/remove/contains, one null allowed, needs good `hashCode`/`equals`. `TreeSet` — red-black tree, sorted (natural or `Comparator`), O(log n), no null, adds navigation API (`first`, `ceiling`, `headSet`). Use `HashSet` for fast uniqueness; `TreeSet` for sorted/range queries. (`LinkedHashSet` keeps insertion order.)

---

## 33. HashMap vs HashSet

`HashMap<K,V>` stores key→value pairs (unique keys, O(1) lookup by key, one null key). `HashSet<E>` stores unique elements only and is internally backed by a `HashMap` (each element is a key mapped to a dummy value). Use `HashMap` to associate values with keys; `HashSet` for membership/uniqueness. Both are not thread-safe (use `ConcurrentHashMap`).
