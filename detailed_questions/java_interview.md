# Java Interview Questions

A comprehensive, code-driven reference covering the core Java topics that come up in Android and backend interviews: the `String` class and string pool, primitives vs. wrapper types, pass-by-value semantics, the Java Memory Model and garbage collection, concurrency primitives (`synchronized`, `volatile`, locks, the atomic package, `ThreadPoolExecutor`), exception handling, object copying and serialization, equality, the `final`/`finally`/`finalize` trio, `static`, reflection, the `String`/`StringBuffer`/`StringBuilder` family, iterators, generics, and the collection types. Answers reflect Java behavior as of 2026 (Java 21+ LTS era, though the semantics described are stable across modern versions).

---

## Table of Contents

1. [How is the String class implemented and why is it immutable?](#1-how-is-the-string-class-implemented-and-why-is-it-immutable)
2. [What is the String pool?](#2-what-is-the-string-pool)
3. [Difference between Integer and int (and autoboxing)](#3-difference-between-integer-and-int-and-autoboxing)
4. [What are the 8 primitive types in Java?](#4-what-are-the-8-primitive-types-in-java)
5. [Are objects passed by reference or by value?](#5-are-objects-passed-by-reference-or-by-value)
6. [What is the Garbage Collector and how does it work?](#6-what-is-the-garbage-collector-and-how-does-it-work)
7. [The Java Memory Model and happens-before](#7-the-java-memory-model-and-happens-before)
8. [What does synchronized mean?](#8-what-does-synchronized-mean)
9. [What is the volatile modifier?](#9-what-is-the-volatile-modifier)
10. [Object-level lock vs class-level lock](#10-object-level-lock-vs-class-level-lock)
11. [Monitor and synchronization](#11-monitor-and-synchronization)
12. [What is a ThreadPoolExecutor?](#12-what-is-a-threadpoolexecutor)
13. [Concurrency vs Parallelism](#13-concurrency-vs-parallelism)
14. [The atomic package: get, set, lazySet, compareAndSet, weakCompareAndSet](#14-the-atomic-package-get-set-lazyset-compareandset-weakcompareandset)
15. [How do try / catch / finally work?](#15-how-do-try--catch--finally-work)
16. [Checked vs Unchecked exceptions](#16-checked-vs-unchecked-exceptions)
17. [throw vs throws](#17-throw-vs-throws)
18. [Shallow vs Deep copy](#18-shallow-vs-deep-copy)
19. [Serialization and Deserialization](#19-serialization-and-deserialization)
20. [The transient modifier](#20-the-transient-modifier)
21. [What are anonymous classes?](#21-what-are-anonymous-classes)
22. [== vs equals()](#22--vs-equals)
23. [The hashCode() and equals() contract](#23-the-hashcode-and-equals-contract)
24. [final, finally, and finalize](#24-final-finally-and-finalize)
25. [The static keyword (variables, methods, blocks)](#25-the-static-keyword-variables-methods-blocks)
26. [Can a static method be overridden?](#26-can-a-static-method-be-overridden)
27. [What is Reflection?](#27-what-is-reflection)
28. [String vs StringBuffer vs StringBuilder](#28-string-vs-stringbuffer-vs-stringbuilder)
29. [Fail-fast vs fail-safe iterators](#29-fail-fast-vs-fail-safe-iterators)
30. [Generics in Java](#30-generics-in-java)
31. [Arrays vs ArrayList](#31-arrays-vs-arraylist)
32. [HashSet vs TreeSet](#32-hashset-vs-treeset)
33. [HashMap vs HashSet](#33-hashmap-vs-hashset)

---

## 1. How is the String class implemented and why is it immutable?

`String` is a final class. Internally it wraps a character store that is declared `final` and is never exposed for mutation. (Historically this was a `char[]`; since Java 9, with Compact Strings, it is a `byte[]` plus a one-byte `coder` flag that records LATIN-1 vs UTF-16 to save memory.) Because the backing array reference is `final` and never handed out, a `String` object's value can never change after construction. There is no primitive `String` type — every string is an object.

Any method that "modifies" a string actually returns a **new** `String`:

```java
String hello = "Hello, World!";
String upper = hello.toUpperCase(); // returns a NEW String
System.out.println(hello); // still "Hello, World!"
System.out.println(upper); // "HELLO, WORLD!"
```

Why immutability was chosen:

- **Security** — strings are used for file paths, network connections, class loading, and credentials. If a string could be mutated after a security check, a caller could change it between the check and the use (a TOCTOU attack).
- **Thread safety** — immutable objects can be shared freely across threads without synchronization.
- **String pool / interning** — immutability is what makes pooling and literal sharing safe (see Q2).
- **hashCode caching** — because the value never changes, `String` caches its hash code on first computation, making it an ideal `HashMap` key.

> Note: "immutable" here means immutable through the normal language. Reflection can forcibly mutate the internal array, but doing so corrupts the JVM's invariants and is never legitimate.

**📚 Reference:** <https://docs.oracle.com/javase/tutorial/java/data/strings.html>

---

## 2. What is the String pool?

The **String pool** (a.k.a. the string intern pool) is a region of the heap where the JVM stores one canonical instance of each distinct string literal. When the compiler encounters a literal, it records it in the constant pool; at runtime that literal resolves to a single shared `String` instance in the pool. Two identical literals therefore refer to the *same* object.

```java
String a = "java";
String b = "java";
System.out.println(a == b); // true — same pooled object

String c = new String("java"); // 'new' forces a fresh heap object
System.out.println(a == c); // false — different reference
System.out.println(a.equals(c)); // true — same content

String d = c.intern(); // returns the canonical pooled instance
System.out.println(a == d); // true
```

Key points:

- `new String("...")` always creates a new object, bypassing the pool.
- `String.intern()` returns the pooled instance, adding the string to the pool if absent.
- Compile-time constant concatenations (`"ja" + "va"`) are folded into a single literal and pooled. Concatenation involving a runtime variable produces a new, non-pooled object.

Interning saves memory and makes `==` comparisons of literals work, but interning large numbers of dynamic strings can pressure memory — prefer `.equals()` for content comparison.

**📚 Reference:** <https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineer-tech-activity-7354122537204666368-HwxH>

---

## 3. Difference between Integer and int (and autoboxing)

`int` is a **primitive** — a raw 32-bit value stored directly (on the stack for locals, inline in objects for fields), with no methods and no `null`. `Integer` is a **wrapper class** that boxes an `int` inside an object on the heap, adds utility methods (`parseInt`, `compareTo`, `MAX_VALUE`, etc.), and can be `null`.

```java
int x = 5;            // primitive, cannot be null
Integer y = 5;        // autoboxing: Integer.valueOf(5)
int z = y;            // unboxing: y.intValue()
Integer n = null;     // legal
// int bad = n;       // NullPointerException on unboxing
```

Use wrappers when you need `null` (e.g., "no value yet"), generics (`List<Integer>` — collections cannot hold primitives), or wrapper methods. Use primitives in hot loops and large arrays for performance and lower memory.

The notorious gotcha — wrapper identity caching:

```java
Integer a = 127, b = 127;
System.out.println(a == b); // true  — values in [-128,127] are cached
Integer c = 128, d = 128;
System.out.println(c == d); // false — outside cache, distinct objects
// Always compare wrapper values with .equals() or unbox first.
```

**📚 Reference:** <https://docs.oracle.com/javase/tutorial/java/data/numberclasses.html>

---

## 4. What are the 8 primitive types in Java?

| Type | Size | Default | Range / Notes |
|---|---|---|---|
| `byte` | 8-bit | `0` | -128 to 127 |
| `short` | 16-bit | `0` | -32,768 to 32,767 |
| `int` | 32-bit | `0` | ~ ±2.1 billion |
| `long` | 64-bit | `0L` | ~ ±9.2 quintillion |
| `float` | 32-bit | `0.0f` | IEEE-754 single precision |
| `double` | 64-bit | `0.0d` | IEEE-754 double precision |
| `char` | 16-bit | `'\u0000'` | unsigned UTF-16 code unit |
| `boolean` | JVM-defined | `false` | `true` / `false` |

Each has a corresponding wrapper: `Byte`, `Short`, `Integer`, `Long`, `Float`, `Double`, `Character`, `Boolean`. (`void` has `Void` but is not a primitive value type.)

---

## 5. Are objects passed by reference or by value?

**Java is always pass-by-value.** What is copied depends on the parameter type:

- For a **primitive**, the value itself is copied. Changes inside the method do not affect the caller.
- For an **object**, the *reference* (the pointer) is copied by value. Both the original and the copy point to the same object, so mutating the object's fields is visible to the caller — but **reassigning the parameter** to a new object does not affect the caller's variable.

```java
void mutate(StringBuilder sb) { sb.append(" world"); } // visible to caller
void reassign(StringBuilder sb) { sb = new StringBuilder("x"); } // NOT visible

StringBuilder s = new StringBuilder("hello");
mutate(s);    System.out.println(s); // "hello world"
reassign(s);  System.out.println(s); // still "hello world"
```

This is why people mistakenly say objects are passed by reference — the *object* is shared, but the *reference variable* is a copy.

---

## 6. What is the Garbage Collector and how does it work?

The **Garbage Collector (GC)** automatically reclaims heap memory occupied by objects that are no longer reachable, freeing the developer from manual `free()`/`delete`. An object is **alive** as long as it is reachable through a chain of references from a *GC root* (active thread stacks, static fields, JNI references). When no such chain exists, the object is eligible for collection.

How modern (generational) GC works:

1. **Reachability via roots** — the collector traces from GC roots; anything not reached is garbage. This handles **cyclic references** correctly: two objects that reference only each other but have no external live reference are *both* collected (Java does not use reference counting).
2. **Generational hypothesis** — most objects die young. The heap is split into a **Young generation** (Eden + two Survivor spaces) and an **Old/Tenured generation**.
3. **Minor GC** — frequent, cheap collections of the young generation; survivors are aged and eventually **promoted** to the old generation.
4. **Major / Full GC** — less frequent collection of the old generation; more expensive.
5. **Mark–Sweep–Compact** — mark reachable objects, sweep the rest, compact to reduce fragmentation.

Common collectors: **G1** (default since Java 9, region-based, low-pause), **ZGC** and **Shenandoah** (sub-millisecond pauses for large heaps), and the older **Parallel** and (removed) CMS collectors. On Android, the runtime is **ART**, which uses a concurrent copying collector tuned for mobile.

You cannot force GC; `System.gc()` is only a hint. Reference strengths influence collection:

- **Strong** (default) — never collected while reachable.
- **Soft** (`SoftReference`) — collected only under memory pressure; good for caches.
- **Weak** (`WeakReference`) — collected at the next GC once only weakly reachable.
- **Phantom** (`PhantomReference`) — for post-mortem cleanup, used with a `ReferenceQueue`.

**📚 Reference:** <https://www.linkedin.com/posts/amit-shekhar-iitbhu_java-tech-softwareengineer-activity-7308111597581799425-qZN0/>

---

## 7. The Java Memory Model and happens-before

The **Java Memory Model (JMM)** defines when a write by one thread becomes visible to a read by another, and what reorderings the compiler/CPU may perform. Without synchronization, each thread may cache variables in registers or CPU caches, so one thread can fail to see another's updates indefinitely.

The core concept is **happens-before**: if action A happens-before action B, then A's memory effects are visible to B. Rules that establish it:

- **Program order** within a single thread.
- **Monitor lock** — unlocking a monitor happens-before every later lock of the same monitor (`synchronized`).
- **volatile** — a write to a volatile field happens-before every later read of it.
- **Thread start/join** — `Thread.start()` happens-before the thread's actions; a thread's actions happen-before another thread returning from its `join()`.
- **Final fields** — values set in a constructor are visible to other threads once the object is safely published.

Without a happens-before edge, the JMM permits stale reads and surprising reorderings, which is the root cause of most concurrency bugs.

---

## 8. What does synchronized mean?

`synchronized` provides **mutual exclusion** and **visibility** by acquiring an object's intrinsic lock (monitor). Only one thread can hold a given monitor at a time, so synchronized blocks guarded by the same monitor cannot run concurrently. Entering the block establishes a happens-before relationship with the previous release, guaranteeing memory visibility.

```java
class Counter {
    private int count = 0;

    public synchronized void increment() { count++; }      // locks 'this'

    public void incrementBlock() {
        synchronized (this) { count++; }                    // explicit, finer-grained
    }

    private static int total;
    public static synchronized void bump() { total++; }     // locks Counter.class
}
```

- A `synchronized` **instance** method locks on `this`.
- A `synchronized` **static** method locks on the `Class` object.
- A `synchronized` **block** locks on whatever object you name — useful for finer granularity and for locking on a dedicated private lock object (a common best practice to avoid external interference).

`synchronized` guarantees both atomicity (of the guarded region) and visibility, but overuse causes contention. It is reentrant: a thread already holding a monitor can re-acquire it.

---

## 9. What is the volatile modifier?

`volatile` marks a field as always read from and written to **main memory**, never a thread-local cache. It guarantees **visibility** and prevents instruction reordering across the access, but it does **not** provide atomicity for compound operations.

```java
class Worker {
    private volatile boolean running = true; // visibility across threads

    public void stop()      { running = false; } // seen immediately by run()
    public void run() { while (running) { /* work */ } }
}
```

Key semantics:

- A write to a volatile happens-before any subsequent read of it (JMM happens-before edge).
- Reads/writes of a single volatile are atomic (including `long`/`double`, which are otherwise not guaranteed atomic).
- It does **not** make `count++` thread-safe — that is read-modify-write (three operations). For that, use `synchronized` or `AtomicInteger`.

Use `volatile` for simple flags and the "double-checked locking" singleton idiom; use atomics or locks when you need atomic compound updates.

---

## 10. Object-level lock vs class-level lock

Both use `synchronized`, but they protect different scopes:

- An **object-level lock** is acquired on a specific instance (`this` or any instance object). It serializes threads operating on the **same instance**; different instances have independent locks and run concurrently.
- A **class-level lock** is acquired on the `Class` object (one per class per classloader). It serializes threads across **all instances** of the class — used to guard static state.

```java
class Demo {
    // Object-level: locks 'this'
    public synchronized void instanceMethod() { /* ... */ }
    public void instanceBlock() { synchronized (this) { /* ... */ } }

    // Class-level: locks Demo.class
    public static synchronized void staticMethod() { /* ... */ }
    public void classBlock() { synchronized (Demo.class) { /* ... */ } }
}
```

A thread holding the object lock of an instance does **not** block another thread acquiring the class lock (they are different monitors), so a static-synchronized and an instance-synchronized method on the same class can run at the same time.

**📚 Reference:** <https://x.com/amitiitbhu/status/1818156936413778332>

---

## 11. Monitor and synchronization

A **monitor** is the synchronization construct that backs `synchronized`. Every Java object has an associated intrinsic lock (the monitor). The JVM implements `synchronized` with the `monitorenter` / `monitorexit` bytecodes: entering acquires the monitor, exiting (including via exception) releases it.

A monitor also provides a **wait set** used by the inter-thread coordination methods, which must be called while holding the monitor:

- `wait()` — releases the monitor and parks the thread until notified (or timeout).
- `notify()` / `notifyAll()` — wake one / all threads waiting on this monitor; they re-contend for the lock.

```java
class BoundedBuffer {
    private final Queue<Integer> q = new LinkedList<>();
    private final int capacity = 10;

    public synchronized void put(int x) throws InterruptedException {
        while (q.size() == capacity) wait();   // always guard with a loop
        q.add(x);
        notifyAll();
    }
    public synchronized int take() throws InterruptedException {
        while (q.isEmpty()) wait();
        int x = q.remove();
        notifyAll();
        return x;
    }
}
```

Always re-check the condition in a `while` loop (not `if`) to defend against spurious wakeups and lost-wakeup races. Monitors are reentrant. For more flexible coordination, `java.util.concurrent.locks.ReentrantLock` + `Condition` offers the same model with extra features (tryLock, fairness, multiple condition queues).

---

## 12. What is a ThreadPoolExecutor?

`ThreadPoolExecutor` is the workhorse implementation of `ExecutorService`. Instead of spawning a fresh `Thread` per task (expensive in memory and scheduling), it maintains a reusable pool of worker threads that pull tasks from a queue. This caps concurrency, reuses threads, and smooths out bursty workloads.

Constructor parameters:

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    2,                              // corePoolSize: threads kept alive even when idle
    4,                              // maximumPoolSize: hard cap on threads
    60L, TimeUnit.SECONDS,          // keepAliveTime: idle timeout for non-core threads
    new LinkedBlockingQueue<>(100), // workQueue: holds tasks awaiting a thread
    Executors.defaultThreadFactory(),       // threadFactory: names/configures threads
    new ThreadPoolExecutor.AbortPolicy()    // rejectedExecutionHandler
);
executor.execute(() -> doWork());
executor.shutdown();
```

How tasks are dispatched (the order is important and a frequent interview point):

1. If running threads < `corePoolSize`, start a new thread for the task.
2. Else enqueue the task in `workQueue`.
3. If the queue is full and threads < `maximumPoolSize`, start a new (non-core) thread.
4. If the queue is full and threads == `maximumPoolSize`, the task is **rejected** via the `RejectedExecutionHandler`.

Built-in rejection policies: `AbortPolicy` (throws `RejectedExecutionException`, the default), `CallerRunsPolicy` (runs on the submitting thread, providing backpressure), `DiscardPolicy` (silently drops), `DiscardOldestPolicy` (drops the oldest queued task).

The `Executors` factory provides presets (`newFixedThreadPool`, `newCachedThreadPool`, `newSingleThreadExecutor`), but they hide these parameters — production code often configures `ThreadPoolExecutor` directly with a bounded queue to avoid unbounded memory growth. On Android, `AsyncTask` historically used one internally; today prefer Kotlin coroutines/`Dispatchers` or a custom executor.

**📚 Reference:** <https://outcomeschool.com/blog/threadpoolexecutor-in-android>

---

## 13. Concurrency vs Parallelism

- **Concurrency** is about *dealing with* many tasks at once — structuring a program so multiple tasks make progress in overlapping time periods. On a single CPU core this is achieved by interleaving (time-slicing); the tasks are *in progress* together but not necessarily *executing* at the same instant.
- **Parallelism** is about *doing* many tasks at the same instant — literally executing on multiple cores/CPUs simultaneously. Parallelism requires hardware with multiple execution units.

```
Concurrency (1 core, interleaved):  A--  B--  A--  B--  A--
Parallelism (2 cores, simultaneous): A-------------------  (core 0)
                                     B-------------------  (core 1)
```

Concurrency is a structuring/design concern; parallelism is an execution/hardware concern. You can have concurrency without parallelism (cooperative tasks on one core) and parallelism is one way to *run* concurrent tasks. In Java, an `ExecutorService` expresses concurrency; whether it runs in parallel depends on the number of available cores. Rob Pike's summary: "Concurrency is about structure; parallelism is about execution."

**📚 Reference:** <https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineering-activity-7370752695130914816-1mxl>

---

## 14. The atomic package: get, set, lazySet, compareAndSet, weakCompareAndSet

`java.util.concurrent.atomic` (e.g. `AtomicInteger`, `AtomicLong`, `AtomicReference`) provides lock-free, thread-safe single-variable operations built on hardware **Compare-And-Swap (CAS)** instructions. The shared method set:

- **`get()`** — reads the current value with **volatile read** semantics (always the latest value).
- **`set(v)`** — writes a new value with **volatile write** semantics; the write is immediately visible to other threads and acts as a happens-before edge.
- **`lazySet(v)`** — like `set` but with weaker (release) ordering: it does not eagerly flush a visibility barrier, so other threads may observe the write *later*. It guarantees prior writes won't be reordered after it, but it is **not** a full happens-before edge. Slightly cheaper on some architectures; used to null out references to aid GC or in single-writer scenarios where immediate cross-thread visibility isn't required.
- **`compareAndSet(expect, update)`** (CAS) — atomically sets the value to `update` **only if** it currently equals `expect`, returning `true`/`false`. Has full volatile read+write memory effects. This is the strong form and only fails when the value genuinely differs.
- **`weakCompareAndSet(expect, update)`** — same atomic conditional write, but **may fail spuriously** (return `false` even when the value matches) and provides **no happens-before ordering** with respect to other variables. Cheaper on some platforms; must be used in a retry loop. (In modern JDKs this is `weakCompareAndSetPlain`; the plain `weakCompareAndSet` is now an alias of the stronger form.)

```java
AtomicInteger counter = new AtomicInteger(0);

counter.set(5);                 // volatile write
int v = counter.get();          // volatile read

// Lock-free increment via CAS retry loop:
int cur;
do {
    cur = counter.get();
} while (!counter.compareAndSet(cur, cur + 1));

// Or the convenience method built on CAS:
counter.incrementAndGet();
```

CAS-based atomics scale better than locks under moderate contention because there is no blocking, but under very high contention the retry loop can waste cycles (consider `LongAdder` for hot counters).

**📚 References:**
<https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/package-summary.html>,
<https://www.baeldung.com/java-atomic-set-vs-lazyset>

---

## 15. How do try / catch / finally work?

`try` wraps code that may throw; `catch` handles matching exceptions; `finally` always runs (whether or not an exception occurred or was caught), making it ideal for cleanup.

```java
public int read() {
    Resource r = null;
    try {
        r = open();
        return r.value();        // even with a return, finally still runs
    } catch (IOException e) {
        log(e);
        return -1;
    } finally {
        if (r != null) r.close(); // always executed
    }
}
```

Important rules:

- `finally` runs even if `try` or `catch` executes a `return`, `break`, or `continue`. **Yes — the `finally` block runs even when there is a `return` inside the `try`.**
- If `finally` itself `return`s or throws, it **overrides** any value/exception from `try`/`catch` — avoid returning from `finally`.
- The only ways `finally` is skipped: `System.exit()`, JVM crash, the thread being killed, or an infinite loop.
- Multiple `catch` blocks are checked top-to-bottom; more specific exceptions must precede broader ones. Multi-catch (`catch (IOException | SQLException e)`) handles several types in one block.
- **try-with-resources** auto-closes anything implementing `AutoCloseable`, replacing most manual `finally` cleanup:

```java
try (BufferedReader br = new BufferedReader(new FileReader("f.txt"))) {
    return br.readLine();
} // br.close() called automatically, even on exception
```

---

## 16. Checked vs Unchecked exceptions

All exceptions descend from `Throwable` (`Error` and `Exception`):

- **Checked exceptions** extend `Exception` (but not `RuntimeException`). The compiler **forces** you to either catch them or declare them with `throws`. They model recoverable, expected failure conditions — e.g. `IOException`, `SQLException`, `ClassNotFoundException`.
- **Unchecked exceptions** extend `RuntimeException`. The compiler does **not** require handling. They typically signal programming bugs — e.g. `NullPointerException`, `IllegalArgumentException`, `ArrayIndexOutOfBoundsException`, `ArithmeticException`.
- **Errors** extend `Error` and represent serious, usually unrecoverable JVM conditions — `OutOfMemoryError`, `StackOverflowError`. Don't catch these in normal code.

```java
// Checked: must declare or catch
void load() throws IOException {
    Files.readAllBytes(Path.of("config.txt"));
}

// Unchecked: no declaration required
int divide(int a, int b) {
    return a / b; // may throw ArithmeticException at runtime
}
```

Guideline: throw checked exceptions for conditions a well-written caller can reasonably recover from; throw unchecked for programming errors. Many modern frameworks favor unchecked exceptions to avoid `throws` clutter.

---

## 17. throw vs throws

- **`throw`** is a statement that actually *raises* an exception instance at runtime — from inside a method or block.
- **`throws`** is part of a method *signature* declaring which checked exceptions the method may propagate, shifting the handling obligation to callers.

```java
// 'throws' declares; 'throw' raises
public void withdraw(int amount) throws InsufficientFundsException {
    if (amount > balance) {
        throw new InsufficientFundsException("Balance too low"); // raises
    }
    balance -= amount;
}
```

You can `throw` one exception object; you can `throws` multiple types (comma-separated). `throw` is followed by an instance; `throws` is followed by class names.

---

## 18. Shallow vs Deep copy

When copying an object that holds references to other (mutable) objects, the question is whether you copy the references or the referenced objects too.

- **Shallow copy** — duplicates the top-level object but copies the **references** to nested objects. The copy and the original share the same nested objects, so mutating a nested object through one is visible through the other. `Object.clone()` is shallow by default.
- **Deep copy** — recursively duplicates the object *and* every object it references, producing a fully independent graph.

```java
class Address { String city; Address(String c){ city = c; } }

class Person implements Cloneable {
    String name;
    Address address;
    Person(String n, Address a) { name = n; address = a; }

    // Shallow: shares the same Address instance
    @Override protected Person clone() throws CloneNotSupportedException {
        return (Person) super.clone();
    }

    // Deep: clones the nested Address too
    Person deepCopy() {
        return new Person(this.name, new Address(this.address.city));
    }
}

Person p1 = new Person("A", new Address("NYC"));
Person shallow = p1.clone();
shallow.address.city = "LA";
System.out.println(p1.address.city); // "LA" — shared object mutated!

Person deep = p1.deepCopy();
deep.address.city = "SF";
System.out.println(p1.address.city); // unchanged
```

Trade-offs: shallow copies are fast and cheap but risk unintended shared-state mutation; deep copies are safe and independent but costlier and must handle cycles. Common deep-copy techniques: manual copy constructors (preferred), serialization round-trip, or copy libraries.

**📚 Reference:** <https://www.linkedin.com/posts/amit-shekhar-iitbhu_outcomeschool-softwareengineering-activity-7224635014641016834-j8X1>

---

## 19. Serialization and Deserialization

**Serialization** converts an object's state into a byte stream so it can be persisted to disk or sent over a network. **Deserialization** reconstructs the object from that byte stream. In core Java this is enabled by implementing the marker interface `java.io.Serializable`.

```java
class User implements Serializable {
    private static final long serialVersionUID = 1L; // version control
    private String name;
    private transient String password; // excluded from serialization
    private static int count;           // static is NOT serialized (class-level)
    User(String n, String p) { name = n; password = p; }
}

// Serialize
try (ObjectOutputStream out =
         new ObjectOutputStream(new FileOutputStream("user.ser"))) {
    out.writeObject(new User("Alice", "secret"));
}

// Deserialize
try (ObjectInputStream in =
         new ObjectInputStream(new FileInputStream("user.ser"))) {
    User u = (User) in.readObject(); // password will be null (transient)
}
```

Key points:

- **`serialVersionUID`** identifies the class version. If you change a class without updating compatibly, a mismatched UID causes `InvalidClassException` on deserialization. Always declare it explicitly.
- `transient` fields and `static` fields are not serialized; on deserialization, transient fields get default values (`null`/`0`).
- The constructor is **not** called during deserialization; the object is rebuilt from the stream.
- You can customize the process with `writeObject`/`readObject` private methods or use `Externalizable` for full control.
- Native Java serialization has well-known security and performance pitfalls (deserialization-of-untrusted-data attacks). On Android, `Parcelable` is faster for IPC, and for general data interchange JSON (Gson/Moshi/kotlinx.serialization) or protobuf is usually preferred.

**📚 Reference:** <https://www.linkedin.com/posts/amit-shekhar-iitbhu_outcomeschool-softwareengineer-tech-activity-7371542771142373376-NbFD>

---

## 20. The transient modifier

`transient` marks an instance field to be **excluded from serialization**. When the object is serialized, transient fields are skipped; when deserialized, they are restored to their default value (`null` for references, `0`/`false` for primitives).

```java
class Session implements Serializable {
    private String userId;
    private transient String authToken;   // sensitive, not persisted
    private transient Connection conn;     // not meaningfully serializable
}
```

Use it for: sensitive data (passwords, tokens) you don't want written out, fields that can be recomputed/derived, or non-serializable resources like live connections, threads, or streams. It applies only to serialization — it has no effect at runtime otherwise.

---

## 21. What are anonymous classes?

An **anonymous class** is a class without a name, declared and instantiated in a single expression. It is used to provide a one-off implementation of an interface or subclass on the spot — typically for listeners and callbacks.

```java
// Implementing an interface inline
Runnable r = new Runnable() {
    @Override public void run() {
        System.out.println("running");
    }
};

// Subclassing inline / Android click listener
button.setOnClickListener(new View.OnClickListener() {
    @Override public void onClick(View v) {
        handleClick();
    }
});
```

Characteristics:

- Defined and instantiated at once; cannot have a constructor (no name) but can use an instance initializer block.
- Can access `final` or **effectively final** local variables of the enclosing scope, plus enclosing instance fields.
- Holds an implicit reference to the enclosing instance — a classic Android memory-leak source (e.g. an anonymous `Handler`/listener outliving its `Activity`).
- For a single-abstract-method (functional) interface, a **lambda** is the concise modern replacement: `Runnable r = () -> System.out.println("running");`. Lambdas do not create a new class file per instance and (unlike anonymous classes) do not capture `this` of the enclosing class unless they reference it.

---

## 22. == vs equals()

- **`==`** compares **references** for objects (do both operands point to the same instance?) and raw values for primitives.
- **`equals()`** compares **logical equality** as defined by the class. The default `Object.equals()` is just `==`, but classes like `String`, `Integer`, and collections override it for value comparison.

```java
String a = new String("hi");
String b = new String("hi");
System.out.println(a == b);       // false — different objects
System.out.println(a.equals(b));  // true  — same content

Integer x = 1000, y = 1000;
System.out.println(x == y);       // false — outside Integer cache
System.out.println(x.equals(y));  // true
```

Rule of thumb: use `==` for primitives and for true identity checks (including `== null`); use `.equals()` for content comparison of objects. For `null`-safe comparison use `Objects.equals(a, b)`.

---

## 23. The hashCode() and equals() contract

`equals()` defines logical equality; `hashCode()` returns an int bucket used by hash-based collections (`HashMap`, `HashSet`, `Hashtable`). They are bound by a contract that **must** be honored, especially when used as keys:

1. **Consistency** — repeated calls return the same result if the object isn't modified.
2. **equals → hashCode** — if `a.equals(b)` is `true`, then `a.hashCode() == b.hashCode()` **must** hold.
3. The reverse is **not** required — equal hash codes do not imply `equals` (a hash *collision* is allowed).
4. `equals` must be reflexive, symmetric, transitive, and `x.equals(null)` is `false`.

Why override both together: a `HashMap` first locates the bucket by `hashCode()`, then uses `equals()` to find the exact key within that bucket. If you override only `equals`, two "equal" objects may land in different buckets and lookups fail. If you override only `hashCode`, equal-by-content objects won't be recognized as the same key.

```java
class Point {
    final int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }

    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Point)) return false;
        Point p = (Point) o;
        return x == p.x && y == p.y;
    }
    @Override public int hashCode() {
        return Objects.hash(x, y); // consistent with equals
    }
}
```

(Java `record` types generate a correct `equals`/`hashCode`/`toString` automatically.)

---

## 24. final, finally, and finalize

Three unrelated things with confusingly similar names:

- **`final`** — a modifier for immutability/non-extensibility:
  - `final` variable → assign once (a constant or, for objects, a fixed *reference*; the object itself can still mutate).
  - `final` method → cannot be overridden.
  - `final` class → cannot be subclassed (e.g. `String`).
  - Make a value `final` when it should never change after construction, to express intent, enable safe sharing across threads (final-field publication guarantees), and allow capture in lambdas/anonymous classes.

- **`finally`** — the block in `try/catch/finally` that always executes for cleanup (see Q15).

- **`finalize()`** — a deprecated `Object` method the GC *might* call before reclaiming an object. It has been **deprecated since Java 9 and marked deprecated *for removal* in Java 18 (JEP 421)**; it is not yet removed as of Java 21, but the finalization mechanism can be disabled and is slated to be dropped in a future release. It is unreliable (no guarantee it runs, when, or at all) and a performance hazard. Use `try-with-resources`/`AutoCloseable` or `java.lang.ref.Cleaner` instead.

```java
final int MAX = 100;             // constant
final List<String> list = new ArrayList<>();
list.add("ok");                  // allowed — object mutates
// list = new ArrayList<>();     // compile error — reference is final
```

---

## 25. The static keyword (variables, methods, blocks)

`static` binds a member to the **class itself** rather than to any instance:

- **Static variable** — a single copy shared across all instances of the class. Changing it through one instance affects all (it belongs to the class, not the object). Good for constants and shared counters.
- **Static method** — invoked without an instance (`Math.max(...)`). It cannot use `this` or access non-static (instance) members directly, because there's no instance context. (So: a static method *cannot* use non-static members.)
- **Static block** — runs **once**, when the class is first loaded/initialized (the first time you create an instance or access a static member), used for one-time setup of static state. Multiple static blocks run in source order.

```java
class Config {
    static int instances = 0;          // shared across all instances
    static final String VERSION;       // constant

    static {                           // static initializer block
        VERSION = loadVersion();       // runs once at class init
    }

    Config() { instances++; }          // affects the single shared counter

    static int count() { return instances; } // callable as Config.count()
}
```

Static members are initialized at class-load time, before any instance exists. Static imports let you reference them unqualified.

---

## 26. Can a static method be overridden?

**No — static methods cannot be overridden; they are *hidden*.** Overriding is based on runtime polymorphism via the object's actual type (dynamic dispatch). Static methods belong to the class and are resolved at **compile time** based on the *declared* (reference) type, not the runtime object — this is **method hiding**, not overriding.

```java
class Parent {
    static void greet() { System.out.println("Parent static"); }
    void hello()        { System.out.println("Parent instance"); }
}
class Child extends Parent {
    static void greet() { System.out.println("Child static"); }  // hides
    @Override void hello() { System.out.println("Child instance"); } // overrides
}

Parent p = new Child();
p.greet();  // "Parent static"   — resolved by reference type (hiding)
p.hello();  // "Child instance"  — resolved by runtime type (overriding)
```

A child can declare a static method with the same signature (covariant return allowed), but both methods remain independently accessible via their respective class names. You also cannot override a static method with an instance method or vice versa — that's a compile error.

---

## 27. What is Reflection?

**Reflection** (`java.lang.reflect`) lets a program inspect and manipulate classes, fields, methods, and constructors **at runtime**, even ones unknown at compile time. You can discover type metadata, instantiate objects, invoke methods, and read/write fields dynamically — including private members (by calling `setAccessible(true)`).

```java
Class<?> clazz = Class.forName("com.example.User");
Object obj = clazz.getDeclaredConstructor().newInstance();

Method m = clazz.getDeclaredMethod("setName", String.class);
m.invoke(obj, "Alice");

Field f = clazz.getDeclaredField("password");
f.setAccessible(true);          // bypass private access
f.set(obj, "secret");
```

Uses: dependency-injection frameworks (Spring, Dagger/Hilt), serialization libraries (Gson/Jackson), JUnit test discovery, ORMs, and IDE/tooling. Trade-offs: it is **slower** than direct calls, **bypasses compile-time type safety** and encapsulation, can break under the Java Platform Module System / `setAccessible` restrictions, and complicates obfuscation/optimization (on Android, reflected members must be kept in ProGuard/R8 rules). Use it sparingly and prefer compile-time alternatives (annotation processing / code generation) where possible.

**📚 Reference:** <https://x.com/amitiitbhu/status/1819234916812341567>

---

## 28. String vs StringBuffer vs StringBuilder

| | `String` | `StringBuilder` | `StringBuffer` |
|---|---|---|---|
| Mutability | **Immutable** | Mutable | Mutable |
| Thread-safe | Yes (immutable) | **No** | **Yes** (synchronized) |
| Performance | Slow for concatenation | **Fastest** | Slower (sync overhead) |
| Since | 1.0 | Java 5 | 1.0 |

Because `String` is immutable, building a string by repeated `+=` in a loop creates a new object each iteration — O(n²) garbage. Use a mutable builder instead.

```java
// BAD: creates many intermediate String objects
String s = "";
for (int i = 0; i < 1000; i++) s += i;

// GOOD: single mutable buffer
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) sb.append(i);
String result = sb.toString();
```

- Use **`StringBuilder`** by default for single-threaded string building (the common case) — it's the fastest.
- Use **`StringBuffer`** only when the same builder instance is shared and mutated across multiple threads (its methods are `synchronized`).
- Use **`String`** for fixed/constant text and when you need a safe shared/key value.

(Note: simple `a + b` concatenations the compiler optimizes; the concern is concatenation in loops.)

**📚 Reference:** <https://outcomeschool.com/blog/string-vs-stringbuffer-vs-stringbuilder>

---

## 29. Fail-fast vs fail-safe iterators

- **Fail-fast** iterators throw `ConcurrentModificationException` if the underlying collection is **structurally modified** (add/remove) during iteration by anything other than the iterator's own `remove()`. They detect modification via an internal `modCount` counter and fail immediately rather than risk undefined behavior. Examples: `ArrayList`, `HashMap`, `HashSet`, `Vector` iterators. They iterate the live collection directly.

- **Fail-safe** (more precisely *weakly consistent*) iterators operate over a **snapshot or a tolerant view**, so concurrent modification does not throw. They never see a `ConcurrentModificationException` but may not reflect modifications made after the iterator was created. Examples: `CopyOnWriteArrayList` (iterates a snapshot copy), `ConcurrentHashMap` (weakly consistent view).

```java
// Fail-fast: throws ConcurrentModificationException
List<Integer> list = new ArrayList<>(List.of(1, 2, 3));
for (Integer n : list) {
    if (n == 2) list.remove(n);   // structural change during iteration -> CME
}

// Safe removal during iteration via the iterator itself:
Iterator<Integer> it = list.iterator();
while (it.hasNext()) {
    if (it.next() == 2) it.remove(); // OK
}

// Fail-safe: no exception (iterates a snapshot)
List<Integer> cow = new CopyOnWriteArrayList<>(List.of(1, 2, 3));
for (Integer n : cow) { cow.remove(n); } // safe, but won't see the change
```

Trade-offs: fail-fast gives early bug detection at the cost of throwing on concurrent access; fail-safe avoids exceptions but uses extra memory (copies) and may show stale data.

---

## 30. Generics in Java

**Generics** parameterize types, enabling **compile-time type safety** and eliminating most casts. Instead of storing `Object` and casting, you declare the element type once; the compiler enforces it and inserts casts for you.

```java
List<String> names = new ArrayList<>();
names.add("Alice");
// names.add(42);          // compile error — type-safe
String first = names.get(0); // no cast needed

// Generic method
<T> T firstOrNull(List<T> list) {
    return list.isEmpty() ? null : list.get(0);
}

// Bounded type parameter
<T extends Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;
}
```

Wildcards control variance:

- `List<? extends Number>` — a producer you can **read** `Number` from (upper bound).
- `List<? super Integer>` — a consumer you can **write** `Integer` into (lower bound).
- PECS mnemonic: *Producer Extends, Consumer Super*.

**Type erasure**: generics are a compile-time feature. The compiler checks types then erases them, replacing type parameters with their bounds (or `Object`). Consequences: no `new T[]`, no `instanceof List<String>`, generic type info isn't available at runtime (except via reflection on declared signatures), and you can't have two overloads that differ only by generic parameter. The benefit is backward compatibility with pre-generics code.

---

## 31. Arrays vs ArrayList

| | Array | `ArrayList` |
|---|---|---|
| Size | **Fixed** at creation | **Dynamic** (auto-resizes) |
| Type | Primitives **or** objects | Objects only (primitives autoboxed) |
| Dimensions | Multi-dimensional supported | Single-dimensional (nest lists) |
| Length / size | `array.length` (field) | `list.size()` (method) |
| API | Minimal (use `Arrays` utility) | Rich (`add`, `remove`, `contains`, ...) |
| Performance | Faster, less memory, no boxing | Slight overhead, resizing cost |
| Type safety | Covariant (runtime `ArrayStoreException`) | Generic, compile-time checked |

```java
int[] arr = new int[3];          // fixed size, holds primitives directly
arr[0] = 10;

List<Integer> list = new ArrayList<>();
list.add(10);                    // autoboxed to Integer; grows dynamically
list.add(20);
```

Use an **array** when the size is known and fixed, you need maximum performance, you store primitives, or you need multiple dimensions. Use an **`ArrayList`** when the size varies, you want convenient operations, and you work with objects. `ArrayList` is backed by an array internally and grows by allocating a larger array and copying (~1.5× growth), giving amortized O(1) append and O(1) random access.

**📚 Reference:** <https://stackoverflow.com/questions/32020000/what-is-the-difference-between-an-array-arraylist-and-a-list/32020072>

---

## 32. HashSet vs TreeSet

Both implement `Set` (no duplicates), but differ in ordering and performance:

| | `HashSet` | `TreeSet` |
|---|---|---|
| Backing structure | Hash table (`HashMap`) | Red-black tree (`TreeMap`) |
| Ordering | **None** (unordered) | **Sorted** (natural or `Comparator`) |
| add/remove/contains | **O(1)** average | **O(log n)** |
| Null | One `null` allowed | **No** null (natural ordering throws NPE) |
| Extra API | Basic `Set` | Navigation: `first`, `last`, `ceiling`, `floor`, `headSet`, `tailSet` |
| Requirement | Good `hashCode`/`equals` | Elements `Comparable` or a `Comparator` |

```java
Set<String> hs = new HashSet<>();
hs.add("banana"); hs.add("apple"); hs.add("cherry");
// iteration order undefined

Set<String> ts = new TreeSet<>();
ts.add("banana"); ts.add("apple"); ts.add("cherry");
// iterates sorted: apple, banana, cherry
```

Use **`HashSet`** when you only need uniqueness and fast membership tests (the common choice). Use **`TreeSet`** when you need elements kept in sorted order or need range/navigation queries. (`LinkedHashSet` is a third option that preserves insertion order with near-`HashSet` performance.)

**📚 Reference:** <https://stackoverflow.com/questions/25602382/java-hashset-vs-treeset-when-should-i-use-over-other>

---

## 33. HashMap vs HashSet

They serve different purposes and one is built on the other:

- **`HashMap<K,V>`** stores **key→value** pairs. Keys are unique; you look up a value by its key in O(1) average. Allows one `null` key and multiple `null` values.
- **`HashSet<E>`** stores **a set of unique elements** (no values). It is internally backed by a `HashMap` — each element is stored as a key mapped to a shared dummy value (`PRESENT`). So a `HashSet` is essentially "the key set of a HashMap."

| | `HashMap` | `HashSet` |
|---|---|---|
| Stores | Key-value pairs | Unique elements only |
| Interface | `Map` | `Set` |
| Insert | `put(key, value)` | `add(element)` |
| Lookup | `get(key)` | `contains(element)` |
| Duplicates | Unique keys (values may repeat) | No duplicate elements |
| Backed by | Hash table | A `HashMap` internally |

```java
HashMap<String, Integer> ages = new HashMap<>();
ages.put("Alice", 30);
ages.put("Bob", 25);
System.out.println(ages.get("Alice")); // 30

HashSet<String> names = new HashSet<>();
names.add("Alice");
names.add("Alice"); // ignored — duplicate
System.out.println(names.contains("Alice")); // true
System.out.println(names.size());            // 1
```

Use a **`HashMap`** when you need to associate values with keys; use a **`HashSet`** when you only care about membership/uniqueness. Both are **not thread-safe** — for concurrent use, prefer `ConcurrentHashMap` / `ConcurrentHashMap.newKeySet()` or wrap with `Collections.synchronizedMap`/`synchronizedSet`.

**📚 Reference:** <https://stackoverflow.com/questions/2773824/difference-between-hashset-and-hashmap>

---
