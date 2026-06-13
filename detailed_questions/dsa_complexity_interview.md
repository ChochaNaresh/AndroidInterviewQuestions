# Data Structures, Algorithms & Complexity — Android Interview Guide

A focused reference for the Data Structures and Algorithms (DSA) portion of Android engineering interviews. It covers Big-O time and space complexity, the data structures every Android developer should know (including the JVM collections **and** the Android-specific `ArrayMap`/`SparseArray` family), the complexity of their core operations, and the algorithm patterns that show up most often in coding rounds. Every concept is paired with Kotlin snippets and complexity tables. Accurate as of 2026.

---

## Table of Contents

1. [What is Big-O notation?](#1-what-is-big-o-notation)
2. [Time complexity classes (with Kotlin examples)](#2-time-complexity-classes-with-kotlin-examples)
3. [Space complexity](#3-space-complexity)
4. [Complexity cheat-sheet table](#4-complexity-cheat-sheet-table)
5. [Array vs ArrayList](#5-array-vs-arraylist)
6. [LinkedList](#6-linkedlist)
7. [HashMap and HashSet](#7-hashmap-and-hashset)
8. [TreeMap (and sorted structures)](#8-treemap-and-sorted-structures)
9. [Stack and Queue](#9-stack-and-queue)
10. [Android-specific: ArrayMap & SparseArray](#10-android-specific-arraymap--sparsearray)
11. [Data-structure operation complexity comparison](#11-data-structure-operation-complexity-comparison)
12. [Common algorithm patterns](#12-common-algorithm-patterns)
13. [How to analyze complexity in an interview](#13-how-to-analyze-complexity-in-an-interview)

---

## 1. What is Big-O notation?

**Big-O notation** means a mathematical notation that describes how an algorithm's resource usage (time or space) scales with the input size.

It expresses the **upper bound** — the worst-case growth rate — and ignores constant factors and lower-order terms.

Key points:

- Big-O measures *how performance changes with input size*, not the absolute wall-clock speed. An `O(n)` algorithm can be slower than an `O(n²)` one for small inputs, but it scales better.
- We drop constants and non-dominant terms: `O(2n + 5)` becomes `O(n)`; `O(n² + n)` becomes `O(n²)`.
- Related notations: **Big-O** (worst case / upper bound), **Big-Ω (Omega)** (best case / lower bound), **Big-Θ (Theta)** (tight bound when upper and lower match). Interviews almost always mean Big-O worst case unless they say "average" or "amortized".
- **Amortized complexity** averages the cost of an operation over a sequence of operations (e.g. `ArrayList.add` is *amortized* `O(1)` even though an occasional resize is `O(n)`).

```
Growth rate (best → worst):
O(1) (Constant) < O(log n) (Logarithmic) < O(n) (Linear) < O(n log n) (Linearithmic) < O(n²) (Quadratic) < O(2ⁿ) (Exponential) < O(n!) (Factorial)
```

**📚 Reference:** [Android Developer should know these Data Structures for Next Interview — Outcome School](https://outcomeschool.com/blog/android-developer-should-know-these-data-structures-for-next-interview)

---

## 2. Time complexity classes (with Kotlin examples)

**Time complexity classes** means the mathematical categories (like O(1), O(log N), O(N), O(N log N), and O(N²)) that classify algorithm runtimes as input size grows.

| Class | Name | Rating | Typical source |
|-------|------|--------|----------------|
| `O(1)` | Constant | Excellent | Direct index/hash access |
| `O(log n)` | Logarithmic | Good | Binary search, balanced-tree ops |
| `O(n)` | Linear | Fair | Single loop over input |
| `O(n log n)` | Linearithmic | Acceptable | Efficient sorts (merge/heap/quick) |
| `O(n²)` | Quadratic | Bad | Nested loops over the input |
| `O(2ⁿ)` | Exponential | Horrible | Naive recursion over subsets |
| `O(n!)` | Factorial | Worst | Brute-force permutations |

### Constant Time — O(1)

Runtime is independent of input size.

```kotlin
fun getFirstElement(array: Array<Int>): Int {
    return array[0]
}

fun main() {
    val score = arrayOf(12, 55, 67, 94, 22)
    println(getFirstElement(score)) // 12
}
```

### Logarithmic Time — O(log n)

The work halves on each step. The classic example is **binary search** on a sorted array.

```kotlin
fun binarySearch(array: Array<Int>, target: Int): Int {
    var firstIndex = 0
    var lastIndex = array.size - 1
    while (firstIndex <= lastIndex) {
        val middleIndex = (firstIndex + lastIndex) / 2
        if (array[middleIndex] == target) return middleIndex
        if (array[middleIndex] > target) {
            lastIndex = middleIndex - 1
        } else {
            firstIndex = middleIndex + 1
        }
    }
    return -1
}

fun main() {
    val score = arrayOf(12, 22, 45, 67, 96)
    println(binarySearch(score, 96)) // 4
}
```

### Linear Time — O(n)

Runtime grows in direct proportion to input size — a single pass over `n` elements.

```kotlin
fun calcFactorial(n: Int): Int {
    var factorial = 1
    for (i in 2..n) {
        factorial *= i
    }
    return factorial
}

fun main() {
    println(calcFactorial(5)) // 120
}
```

### Linearithmic Time — O(n log n)

The sweet spot for comparison-based sorting. You do `O(n)` work across `O(log n)` levels. Kotlin's `sorted()`/`sort()` use a tuned merge/insertion hybrid (TimSort) under the hood.

```kotlin
fun main() {
    val data = intArrayOf(5, 2, 9, 1, 7)
    data.sort()              // O(n log n)
    println(data.toList())   // [1, 2, 5, 7, 9]
}
```

### Quadratic Time — O(n²)

A loop inside a loop, each over the input.

```kotlin
fun matchElements(array: Array<String>): String {
    for (i in array.indices) {
        for (j in array.indices) {
            if (i != j && array[i] == array[j]) {
                return "Match found at $i and $j"
            }
        }
    }
    return "No matches found"
}

fun main() {
    val fruit = arrayOf("a", "b", "c", "d", "e", "f", "g", "h", "c", "j")
    println(matchElements(fruit)) // "Match found at 2 and 8"
}
```

### Exponential Time — O(2ⁿ)

Each added input unit doubles the work. Naive recursive Fibonacci recomputes overlapping subproblems.

```kotlin
fun recursiveFibonacci(n: Int): Int {
    return if (n < 2) n
    else recursiveFibonacci(n - 1) + recursiveFibonacci(n - 2)
}

fun main() {
    println(recursiveFibonacci(6)) // 8
}
```

> **Interview tip:** This `O(2ⁿ)` Fibonacci is the canonical lead-in to **memoization / dynamic programming**, which reduces it to `O(n)` time and `O(n)` space by caching results.

```kotlin
fun fib(n: Int, memo: HashMap<Int, Int> = HashMap()): Int {
    if (n < 2) return n
    memo[n]?.let { return it }
    val result = fib(n - 1, memo) + fib(n - 2, memo)
    memo[n] = result
    return result // O(n) time, O(n) space
}
```

---

## 3. Space complexity

**Space complexity** means the measurement of extra memory an algorithm requires during execution relative to the input size.

- An in-place loop that uses a few variables → `O(1)` auxiliary space.
- Building a new list/map proportional to input → `O(n)`.
- **Recursion costs stack space.** A recursion `n` levels deep is `O(n)` space even if each frame is small. Tail-recursive or iterative versions can drop this to `O(1)`.

```kotlin
// O(1) extra space — iterative sum
fun sum(nums: IntArray): Long {
    var total = 0L
    for (x in nums) total += x   // only one accumulator
    return total
}

// O(n) extra space — recursion depth n
fun sumRecursive(nums: IntArray, i: Int = 0): Long {
    if (i == nums.size) return 0
    return nums[i] + sumRecursive(nums, i + 1) // n stack frames
}
```

There is often a **time/space trade-off**: memoization, hash-based lookups, and precomputed indexes spend memory to save time.

---

## 4. Complexity cheat-sheet table

**Data structure operations complexity** means the average and worst-case time complexities for accessing, searching, inserting, and deleting elements in fundamental structures.

| Structure | Access | Search | Insert | Delete | Notes |
|-----------|:------:|:------:|:------:|:------:|-------|
| Array (fixed) | `O(1)` | `O(n)` | `O(n)` | `O(n)` | Insert/delete shift elements |
| `ArrayList` | `O(1)` | `O(n)` | `O(1)`* / `O(n)` | `O(n)` | *Amortized at end; mid-insert shifts |
| `LinkedList` | `O(n)` | `O(n)` | `O(1)`† | `O(1)`† | †At a known node; finding it is `O(n)` |
| `HashMap` / `HashSet` | — | `O(1)` avg, `O(n)` worst | `O(1)` avg | `O(1)` avg | Worst case is `O(log n)` since Java 8 (treeified buckets) |
| `TreeMap` / `TreeSet` | — | `O(log n)` | `O(log n)` | `O(log n)` | Red-black tree, keeps keys sorted |
| Stack (`ArrayDeque`) | `O(n)` | `O(n)` | `O(1)`* | `O(1)` | push/pop at one end |
| Queue (`ArrayDeque`) | `O(n)` | `O(n)` | `O(1)`* | `O(1)` | enqueue/dequeue at ends |
| `ArrayMap` (Android) | `O(log n)` | `O(log n)` | `O(n)` worst | `O(n)` worst | Binary search over sorted hashes |
| `SparseArray` (Android) | `O(log n)` | `O(log n)` | `O(n)` worst / `O(1)` append | `O(n)` worst | int keys, no autoboxing |

### Sorting algorithms

| Algorithm | Best | Average | Worst | Space | Stable |
|-----------|:----:|:-------:|:-----:|:-----:|:------:|
| Bubble sort | `O(n)` | `O(n²)` | `O(n²)` | `O(1)` | Yes |
| Insertion sort | `O(n)` | `O(n²)` | `O(n²)` | `O(1)` | Yes |
| Merge sort | `O(n log n)` | `O(n log n)` | `O(n log n)` | `O(n)` | Yes |
| Quick sort | `O(n log n)` | `O(n log n)` | `O(n²)` | `O(log n)` | No |
| Heap sort | `O(n log n)` | `O(n log n)` | `O(n log n)` | `O(1)` | No |
| TimSort (Kotlin/Java default for objects) | `O(n)` | `O(n log n)` | `O(n log n)` | `O(n)` | Yes |

---

## 5. Array vs ArrayList

**Array vs ArrayList** means the difference between a fixed-size contiguous memory sequence (Array), and a dynamically-resized list wrapper (ArrayList).

**`ArrayList`** (`java.util.ArrayList`, Kotlin's `MutableList` default) is a growable, array-backed list.

| Aspect | Array | ArrayList |
|--------|-------|-----------|
| Size | Fixed at creation | Dynamic (auto-resizes) |
| Index access | `O(1)` | `O(1)` |
| Add at end | n/a (fixed) | `O(1)` amortized |
| Insert/remove in middle | manual shift `O(n)` | `O(n)` (shifts elements) |
| Primitives | `IntArray` stores raw `int` (no boxing) | `List<Int>` boxes to `Integer` |
| Memory | Exact | Over-allocates capacity (~1.5× growth) |

```kotlin
// Array — fixed size, no boxing for primitives
val arr = IntArray(5)        // [0,0,0,0,0]
arr[2] = 42                  // O(1)

// ArrayList — dynamic
val list = arrayListOf(1, 2, 3)
list.add(4)                  // O(1) amortized
list.add(0, 99)              // O(n): shifts everything right
val first = list[0]          // O(1)
```

**Why is `ArrayList.add` "amortized O(1)"?** When the backing array fills, it allocates a larger array (typically 1.5×) and copies all elements — an `O(n)` event. But because resizes happen geometrically rarely, the *average* cost per add across many adds is `O(1)`.

---

## 6. LinkedList

**LinkedList** means a sequential data structure consisting of nodes where each node contains a value and pointers to its neighbouring nodes.

| Operation | Complexity | Why |
|-----------|:----------:|-----|
| Access by index | `O(n)` | Must walk from head/tail |
| Search | `O(n)` | Linear scan |
| Insert/delete at head or tail | `O(1)` | Pointer updates only |
| Insert/delete at known node | `O(1)` | Relink neighbors |
| Insert/delete by index | `O(n)` | `O(n)` to find + `O(1)` to relink |

```kotlin
val ll = java.util.LinkedList<Int>()
ll.addFirst(1)   // O(1)
ll.addLast(2)    // O(1)
ll.add(1, 99)    // O(n) to reach index 1
val x = ll[0]    // O(n) — NOT O(1) like ArrayList
```

**When to use:** frequent insertions/deletions at the ends (queue/deque behaviour). **Avoid** when you need random index access — `ArrayList` wins there. In practice on Android, `ArrayDeque` is usually preferred over `LinkedList` for stack/queue use because it has better cache locality and no per-node object overhead.

---

## 7. HashMap and HashSet

**HashMap vs HashSet** means the distinction between a key-value mapping structure indexed by hash codes (HashMap), and a collection of unique elements backed by a HashMap (HashSet).

**`HashSet`** is a `HashMap` where only keys matter (it wraps a `HashMap` internally).

| Operation | Average | Worst case |
|-----------|:-------:|:----------:|
| `get` / `containsKey` | `O(1)` | `O(log n)` |
| `put` | `O(1)` | `O(log n)` |
| `remove` | `O(1)` | `O(log n)` |
| Iteration | `O(n)` | `O(n)` |

```kotlin
val map = HashMap<String, Int>()
map["apple"] = 3          // O(1) avg
val n = map["apple"]      // O(1) avg
map.remove("apple")       // O(1) avg

val set = HashSet<Int>()
set.add(5)                // O(1) avg
val exists = 5 in set     // O(1) avg
```

**Why "worst case O(log n)" and not O(n)?** Before Java 8, many keys colliding into one bucket degraded to an `O(n)` linked-list scan. Since Java 8 (and on modern Android runtimes), a bucket with too many collisions is **treeified** into a balanced (red-black) tree, capping worst-case lookups at `O(log n)`. With a poor/identical `hashCode()`, behaviour still trends toward that bound.

**Interview must-know:** `HashMap` relies on correct, consistent `hashCode()` **and** `equals()`. If you override one you must override the other, and keys should be immutable in the fields used for hashing. `HashMap`/`HashSet` make **no ordering guarantee** — use `LinkedHashMap` for insertion order, `TreeMap` for sorted order.

---

## 8. TreeMap (and sorted structures)

**TreeMap** means a sorted map implementation backed by a red-black self-balancing binary search tree.

It keeps keys in sorted order and guarantees `O(log n)` for the core operations. **`TreeSet`** is the set equivalent.

| Operation | Complexity |
|-----------|:----------:|
| `get` / `containsKey` | `O(log n)` |
| `put` | `O(log n)` |
| `remove` | `O(log n)` |
| `firstKey` / `lastKey` | `O(log n)` |
| `floorKey` / `ceilingKey` / range views | `O(log n)` |
| Ordered iteration | `O(n)` |

```kotlin
val tm = java.util.TreeMap<Int, String>()
tm[3] = "c"; tm[1] = "a"; tm[2] = "b"
println(tm.firstKey())       // 1  (sorted)
println(tm.ceilingKey(2))    // 2  — smallest key >= 2
for ((k, v) in tm) print("$k$v ")  // a b c — in key order
```

**When to use over HashMap:** you need sorted iteration, nearest-key queries (`floor`/`ceiling`), or range scans. The trade-off is `O(log n)` instead of `O(1)` per operation.

---

## 9. Stack and Queue

**Stack vs Queue** means the difference between a Last-In-First-Out data access structure (Stack), and a First-In-First-Out access structure (Queue).

- **Stack** — LIFO (Last In, First Out): `push`, `pop`, `peek`.
- **Queue** — FIFO (First In, First Out): `enqueue`/`offer`, `dequeue`/`poll`, `peek`.

On the JVM, prefer **`ArrayDeque`** for both (it's faster than the legacy synchronized `Stack` class and avoids `LinkedList`'s node overhead).

| Operation | Stack (`ArrayDeque`) | Queue (`ArrayDeque`) |
|-----------|:--------------------:|:--------------------:|
| push / enqueue | `O(1)` amortized | `O(1)` amortized |
| pop / dequeue | `O(1)` | `O(1)` |
| peek | `O(1)` | `O(1)` |
| search | `O(n)` | `O(n)` |

```kotlin
// Stack (LIFO)
val stack = ArrayDeque<Int>()
stack.addLast(1); stack.addLast(2)   // push
println(stack.removeLast())          // 2  (pop)
println(stack.last())                // 1  (peek)

// Queue (FIFO)
val queue = ArrayDeque<Int>()
queue.addLast(1); queue.addLast(2)   // enqueue
println(queue.removeFirst())         // 1  (dequeue)
```

> Kotlin's stdlib `kotlin.collections.ArrayDeque` and `java.util.ArrayDeque` both work. For a thread-safe queue (e.g. producer/consumer between threads) use `ConcurrentLinkedQueue` or `LinkedBlockingQueue`. For priority-ordered dequeue use `PriorityQueue` (a binary heap: `offer`/`poll` are `O(log n)`, `peek` is `O(1)`).

---

## 10. Android-specific: ArrayMap & SparseArray

**ArrayMap vs SparseArray** means Android-specific memory-efficient collections that use binary search on sorted primitive arrays to avoid the memory overhead of Java's HashMap.

For the small maps common in UI/view code, this fragments memory and increases GC pressure (jank). Android's `androidx.collection` package offers leaner alternatives.

### ArrayMap

`ArrayMap<K, V>` stores entries in **two parallel arrays** — one of sorted key hashes, one of key/value references — and does a **binary search** on the hash array to locate a key.

- **Lookup:** `O(log n)` (binary search), vs `HashMap`'s `O(1)`.
- **Insert/remove:** `O(n)` worst case, because elements may need shifting within the arrays.
- **Memory:** much lower overhead than `HashMap` for small maps — no per-entry `Entry` objects, no large bucket array, better cache locality.
- **Use when:** the map holds a small number of entries (rule of thumb: up to a few hundred / ~1000). For large or high-churn maps, `HashMap` is faster.

```kotlin
import androidx.collection.ArrayMap

val attrs = ArrayMap<String, Int>()
attrs["width"] = 100      // O(n) worst (shift), O(log n) locate
val w = attrs["width"]    // O(log n)
```

### SparseArray family

When **keys are primitives**, the sparse containers avoid autoboxing entirely by storing raw primitive key arrays alongside the value array:

| Class | Key type | Value type |
|-------|----------|------------|
| `SparseArray<E>` | `int` | object `E` |
| `SparseIntArray` | `int` | `int` |
| `SparseLongArray` | `int` | `long` |
| `SparseBooleanArray` | `int` | `boolean` |
| `LongSparseArray<E>` | `long` | object `E` |

- **Lookup:** `O(log n)` (binary search on the sorted int-key array).
- **Insert:** `O(n)` worst case (keep keys sorted); **`append()` is `O(1)`** when you add keys in increasing order.
- **Memory:** no `Integer` boxing, no `Entry` objects — significantly less garbage than `HashMap<Integer, V>`.

```kotlin
import android.util.SparseArray

val views = SparseArray<String>()
views.put(10, "header")   // O(n) worst, O(1) via append() if ascending
views.append(20, "body")  // O(1) — key greater than all existing
val v = views.get(10)     // O(log n), no boxing
```

### Trade-off summary

| | `HashMap` | `ArrayMap` | `SparseArray` |
|---|:---:|:---:|:---:|
| Lookup | `O(1)` avg | `O(log n)` | `O(log n)` |
| Insert | `O(1)` avg | `O(n)` worst | `O(n)` worst / `O(1)` append |
| Memory (small maps) | High | Low | Lowest (no boxing) |
| Key type | any object | any object | primitive (`int`/`long`) |
| Best for | large / high-churn maps | small object-keyed maps | small maps with int/long keys |

**Practical guidance (2026):** the measured difference is small (often <10% in time/energy and <2% memory) for ~100 entries, so the real win from sparse/array containers is in UI and hot paths where reducing **allocations and GC** keeps frames smooth. For very large or write-heavy maps, or when you need general JVM `Map` semantics, stick with `HashMap`/`LinkedHashMap`.

**📚 Reference:**
- [Optimizing Android Performance: When to Use SparseArray, SparseIntArray, and ArrayMap Instead of HashMap — droidcon](https://www.droidcon.com/2025/10/17/optimizing-android-performance-when-to-use-sparsearray-sparseintarray-and-arraymap-instead-of-hashmap/)
- [Android: Should you use HashMap or SparseArray? — Greenspector](https://greenspector.com/en/android-should-you-use-hashmap-or-sparsearray/)
- [Optimising Android app performance with ArrayMap — Medium](https://mzeus.medium.com/optimising-android-app-performance-with-arraymap-9296f4a1f9eb)

---

## 11. Data-structure operation complexity comparison

**Data structure efficiency** means the performance trade-offs between linear structures (lists) and associative structures (trees, maps) for different tasks.

| Need | Best choice | Why |
|------|-------------|-----|
| Fast index access | `Array` / `ArrayList` | `O(1)` access |
| Fast key→value lookup, no order | `HashMap` | `O(1)` average |
| Sorted keys / range queries | `TreeMap` | `O(log n)`, ordered |
| Insertion-order preserved | `LinkedHashMap` | `O(1)` + order |
| Frequent add/remove at ends | `ArrayDeque` | `O(1)` both ends |
| Always pull min/max | `PriorityQueue` | `O(log n)` poll, `O(1)` peek |
| Membership test, no duplicates | `HashSet` | `O(1)` average |
| Small map, int keys, low GC | `SparseArray` | no boxing, low memory |
| Small map, object keys, low GC | `ArrayMap` | low memory overhead |

---

## 12. Common algorithm patterns

**Algorithm design patterns** means reusable algorithmic strategies—such as Sliding Window, Two Pointers, and Fast/Slow Pointers—used to solve coding problems.

Recognizing the pattern is half the battle.

### Two pointers — `O(n)` time, `O(1)` space

Move two indices through a (often sorted) array.

```kotlin
fun isPalindrome(s: String): Boolean {
    var left = 0
    var right = s.length - 1
    while (left < right) {
        if (s[left] != s[right]) return false
        left++; right--
    }
    return true
}
```

### Sliding window — `O(n)` time

Maintain a moving range to answer subarray/substring questions without recomputation.

```kotlin
// Longest substring without repeating characters
fun lengthOfLongestSubstring(s: String): Int {
    val seen = HashMap<Char, Int>()
    var start = 0; var best = 0
    for (end in s.indices) {
        seen[s[end]]?.let { if (it >= start) start = it + 1 }
        seen[s[end]] = end
        best = maxOf(best, end - start + 1)
    }
    return best
}
```

### Hashing for O(1) lookup — trade space for time

```kotlin
// Two Sum — O(n) time, O(n) space
fun twoSum(nums: IntArray, target: Int): IntArray {
    val seen = HashMap<Int, Int>()      // value -> index
    for (i in nums.indices) {
        val need = target - nums[i]
        seen[need]?.let { return intArrayOf(it, i) }
        seen[nums[i]] = i
    }
    return intArrayOf()
}
```

### Binary search — `O(log n)`

Applies to any sorted/monotonic search space, not just arrays (e.g. "search on the answer").

```kotlin
fun lowerBound(a: IntArray, target: Int): Int {
    var lo = 0; var hi = a.size
    while (lo < hi) {
        val mid = (lo + hi) ushr 1   // avoids overflow
        if (a[mid] < target) lo = mid + 1 else hi = mid
    }
    return lo
}
```

### Recursion & Divide and Conquer — often `O(n log n)`

Split the problem, solve subproblems, combine. Merge sort is the archetype.

```kotlin
fun mergeSort(a: IntArray): IntArray {
    if (a.size <= 1) return a
    val mid = a.size / 2
    val left = mergeSort(a.copyOfRange(0, mid))
    val right = mergeSort(a.copyOfRange(mid, a.size))
    return merge(left, right)
}

fun merge(l: IntArray, r: IntArray): IntArray {
    val out = IntArray(l.size + r.size)
    var i = 0; var j = 0; var k = 0
    while (i < l.size && j < r.size) out[k++] = if (l[i] <= r[j]) l[i++] else r[j++]
    while (i < l.size) out[k++] = l[i++]
    while (j < r.size) out[k++] = r[j++]
    return out
}
```

### BFS (Breadth-First Search) / DFS (Depth-First Search) on graphs & trees — `O(V + E)`

```kotlin
// BFS — shortest path in an unweighted graph
fun bfs(graph: Map<Int, List<Int>>, start: Int): List<Int> {
    val visited = HashSet<Int>()
    val queue = ArrayDeque<Int>()
    val order = mutableListOf<Int>()
    queue.addLast(start); visited.add(start)
    while (queue.isNotEmpty()) {
        val node = queue.removeFirst()
        order.add(node)
        for (next in graph[node].orEmpty()) {
            if (visited.add(next)) queue.addLast(next)
        }
    }
    return order
}
```

DFS is the same idea with a stack (or recursion). Use **BFS** for shortest path in unweighted graphs and level-order traversal; **DFS** for path existence, cycle detection, and topological sort.

### Dynamic Programming — turn exponential into polynomial

Identify overlapping subproblems + optimal substructure, then cache (top-down memoization) or build a table (bottom-up). See the Fibonacci memoization in [Section 2](#2-time-complexity-classes-with-kotlin-examples) — `O(2ⁿ)` → `O(n)`.

---

## 13. How to analyze complexity in an interview

**Big-O analysis strategy** means a systematic process of identifying operations, loops, and recursive calls to determine the time and space complexity of code.

1. **Identify the input size(s)** — `n` for one array, `n`/`m` for two, `V`/`E` for a graph.
2. **Count loops & their bounds.** Sequential loops add (`O(n) + O(n) = O(n)`); nested loops multiply (`O(n) × O(n) = O(n²)`).
3. **Account for hidden costs.** `list.contains()` is `O(n)`; calling it inside an `O(n)` loop is `O(n²)`. `String` concatenation in a loop, `list.add(0, x)`, and sorting (`O(n log n)`) are common hidden costs.
4. **Account for recursion depth** for stack-space, and use the recurrence (e.g. `T(n) = 2T(n/2) + O(n)` → `O(n log n)`).
5. **Drop constants and lower-order terms** → report the dominant term.
6. **State both time and space**, and clarify whether it's worst/average/amortized.
7. **Then optimise:** can a `HashMap` turn an `O(n)` inner search into `O(1)`? Can two pointers or a sliding window remove a nested loop? Can memoization collapse exponential recursion?

> **Android-flavored follow-ups** interviewers like: "Why might `ArrayMap`/`SparseArray` be better than `HashMap` here?" (memory/GC on UI thread), "What's the cost of `notifyDataSetChanged` vs `DiffUtil`?" (`DiffUtil` runs an `O(n + m)`-ish Myers diff to compute minimal updates), and "Why is `LinkedList` a poor `RecyclerView` adapter backing store?" (`O(n)` random access during binding).

**📚 Reference:** [Android Developer should know these Data Structures for Next Interview — Outcome School](https://outcomeschool.com/blog/android-developer-should-know-these-data-structures-for-next-interview)
