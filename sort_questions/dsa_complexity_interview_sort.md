# Data Structures, Algorithms & Complexity — Short Answers (Quick Revision)

> Condensed answers for rapid review before an interview. For full explanations and code see `dsa_complexity_interview.md`.

---

## 1. What is Big-O notation?

**Big-O notation** means a mathematical notation that describes how an algorithm's resource usage (time or space) scales with the input size.

Describes how an algorithm's resource usage (time/memory) grows as input size `n` grows — the worst-case **upper bound**, ignoring constants and lower-order terms. Related: Big-Ω (lower bound), Big-Θ (tight bound), and **amortized** (averaged over a sequence, e.g. `ArrayList.add` is amortized `O(1)`). Order: `O(1)` (Constant) < `O(log n)` (Logarithmic) < `O(n)` (Linear) < `O(n log n)` (Linearithmic) < `O(n²)` (Quadratic) < `O(2ⁿ)` (Exponential) < `O(n!)` (Factorial).

---



## 2. Time complexity classes (with Kotlin examples)

**Time complexity classes** means the mathematical categories (like O(1), O(log N), O(N), O(N log N), and O(N²)) that classify algorithm runtimes as input size grows.

- `O(1)` (Constant) — index/hash access.
- `O(log n)` (Logarithmic) — binary search, balanced-tree ops.
- `O(n)` (Linear) — single loop.
- `O(n log n)` (Linearithmic) — efficient sorts.
- `O(n²)` (Quadratic) — nested loops.
- `O(2ⁿ)` (Exponential) — naive recursion (e.g. recursive Fibonacci).
- `O(n!)` (Factorial) — brute-force permutations.

Tip: naive `O(2ⁿ)` Fibonacci → `O(n)` with memoization/DP (Dynamic Programming).

---



## 3. Space complexity

**Space complexity** means the measurement of extra memory an algorithm requires during execution relative to the input size.

The *extra* (auxiliary) memory an algorithm needs vs input size. In-place loop = `O(1)`; building a new list/map = `O(n)`; **recursion costs stack space** (`n` deep = `O(n)`), reducible via iteration/tail recursion. There's often a time/space trade-off (memoization, hashing spend memory to save time).

---



## 4. Complexity cheat-sheet table

**Data structure operations complexity** means the average and worst-case time complexities for accessing, searching, inserting, and deleting elements in fundamental structures.

- Array/ArrayList: `O(1)` access, `O(n)` search/insert-mid; ArrayList add-at-end amortized `O(1)`.
- LinkedList: `O(n)` access/search, `O(1)` ends.
- HashMap/HashSet: `O(1)` avg, `O(log n)` worst (treeified).
- TreeMap/TreeSet: `O(log n)`, sorted.
- ArrayDeque (stack/queue): `O(1)` ends.

Sorts: merge/heap `O(n log n)`; quick `O(n log n)` avg / `O(n²)` worst; TimSort `O(n log n)` (Java/Kotlin default for objects).

---



## 5. Array vs ArrayList

**Array vs ArrayList** means the difference between a fixed-size contiguous memory sequence (Array), and a dynamically-resized list wrapper (ArrayList).

Array is fixed-size, contiguous, no boxing for primitives (`IntArray`). ArrayList is growable (array-backed, auto-resizes ~1.5×, boxes `Int`→`Integer`). Both `O(1)` index access; mid-insert/remove is `O(n)`. `ArrayList.add` is amortized `O(1)` because geometric resizing makes the occasional `O(n)` copy rare on average.

---



## 6. LinkedList

**LinkedList** means a sequential data structure consisting of nodes where each node contains a value and pointers to its neighbouring nodes.

A doubly linked list: each node has value + prev/next pointers. `O(1)` insert/delete at ends or a known node, but `O(n)` access/search (must walk). Use for frequent end insertions; avoid for random access. On Android, `ArrayDeque` is usually preferred (better cache locality, no per-node overhead).

---



## 7. HashMap and HashSet

**HashMap vs HashSet** means the distinction between a key-value mapping structure indexed by hash codes (HashMap), and a collection of unique elements backed by a HashMap (HashSet).

`HashMap` stores key→value in buckets by `hashCode()`; `HashSet` wraps one. `get`/`put`/`remove` are `O(1)` average, `O(log n)` worst (buckets treeify into red-black trees since Java 8). Requires consistent `hashCode()` + `equals()` (override both; immutable keys). No ordering — use `LinkedHashMap` (insertion order) or `TreeMap` (sorted).

---



## 8. TreeMap (and sorted structures)

**TreeMap** means a sorted map implementation backed by a red-black self-balancing binary search tree.

A red-black self-balancing BST (Binary Search Tree) keeping keys sorted with `O(log n)` core ops, plus `firstKey`/`floorKey`/`ceilingKey`/range views. `TreeSet` is the set form. Use over HashMap when you need sorted iteration, nearest-key queries, or range scans — trading `O(1)` for `O(log n)`.

---



## 9. Stack and Queue

**Stack vs Queue** means the difference between a Last-In-First-Out data access structure (Stack), and a First-In-First-Out access structure (Queue).

Abstract data types by access discipline: **Stack** = LIFO (Last-In, First-Out) (push/pop/peek), **Queue** = FIFO (First-In, First-Out) (enqueue/dequeue/peek). Prefer `ArrayDeque` for both (faster than legacy `Stack`, no node overhead) — `O(1)` push/pop/peek. Use `PriorityQueue` (binary heap) for priority order, concurrent queues for thread safety.

---



## 10. Android-specific: ArrayMap & SparseArray

**ArrayMap vs SparseArray** means Android-specific memory-efficient collections that use binary search on sorted primitive arrays to avoid the memory overhead of Java's HashMap.

`HashMap` is memory-hungry (per-entry `Entry` objects, autoboxed keys). **`ArrayMap`** uses two parallel arrays with binary search: `O(log n)` lookup, `O(n)` insert/remove, low memory — good for small object-keyed maps. **`SparseArray`** family uses primitive `int`/`long` keys (no boxing): `O(log n)` lookup, `O(n)` insert (`append()` is `O(1)` ascending). Win is reduced allocations/GC on UI hot paths; use HashMap for large/write-heavy maps.

---



## 11. Data-structure operation complexity comparison

**Data structure efficiency** means the performance trade-offs between linear structures (lists) and associative structures (trees, maps) for different tasks.

Pick by need: fast index access → Array/ArrayList; key→value lookup → HashMap; sorted/range → TreeMap; insertion order → LinkedHashMap; add/remove at ends → ArrayDeque; min/max → PriorityQueue; membership → HashSet; small int-keyed low-GC map → SparseArray; small object-keyed → ArrayMap.

---



## 12. Common algorithm patterns

**Algorithm design patterns** means reusable algorithmic strategies—such as Sliding Window, Two Pointers, and Fast/Slow Pointers—used to solve coding problems.

- **Two pointers** — `O(n)`/`O(1)` on (often sorted) arrays (e.g. palindrome).
- **Sliding window** — `O(n)` subarray/substring problems.
- **Hashing** — trade space for `O(1)` lookup (Two Sum).
- **Binary search** — `O(log n)` on any monotonic search space.
- **Divide & conquer / recursion** — often `O(n log n)` (merge sort).
- **BFS (Breadth-First Search) / DFS (Depth-First Search)** — `O(V+E)` graph/tree traversal (BFS for shortest path, DFS for cycles/topo sort).
- **Dynamic programming** — overlapping subproblems + memoization/tabulation turns exponential into polynomial.

---



## 13. How to analyze complexity in an interview

**Big-O analysis strategy** means a systematic process of identifying operations, loops, and recursive calls to determine the time and space complexity of code.

1. Identify input size(s). 2. Count loops (sequential add, nested multiply). 3. Account for hidden costs (`contains` is `O(n)`, sorting `O(n log n)`). 4. Account for recursion depth (use the recurrence). 5. Drop constants/lower terms. 6. State both time and space (worst/avg/amortized). 7. Then optimise — HashMap to remove an inner search, two pointers/sliding window to remove a nested loop, memoization to collapse recursion.

