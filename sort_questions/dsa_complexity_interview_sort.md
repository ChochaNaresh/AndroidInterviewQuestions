# Data Structures, Algorithms & Complexity — Short Answers (Quick Revision)

> Condensed answers for rapid review before an interview. For full explanations and code see `dsa_complexity_interview.md`.

---

## 1. What is Big-O notation?

Describes how an algorithm's resource usage (time/memory) grows as input size `n` grows — the worst-case **upper bound**, ignoring constants and lower-order terms. Related: Big-Ω (lower bound), Big-Θ (tight bound), and **amortized** (averaged over a sequence, e.g. `ArrayList.add` is amortized `O(1)`). Order: `O(1) < O(log n) < O(n) < O(n log n) < O(n²) < O(2ⁿ) < O(n!)`.

---

## 2. Time complexity classes (with Kotlin examples)

- `O(1)` — index/hash access.
- `O(log n)` — binary search, balanced-tree ops.
- `O(n)` — single loop.
- `O(n log n)` — efficient sorts.
- `O(n²)` — nested loops.
- `O(2ⁿ)` — naive recursion (e.g. recursive Fibonacci).
- `O(n!)` — brute-force permutations.

Tip: naive `O(2ⁿ)` Fibonacci → `O(n)` with memoization/DP.

---

## 3. Space complexity

The *extra* (auxiliary) memory an algorithm needs vs input size. In-place loop = `O(1)`; building a new list/map = `O(n)`; **recursion costs stack space** (`n` deep = `O(n)`), reducible via iteration/tail recursion. There's often a time/space trade-off (memoization, hashing spend memory to save time).

---

## 4. Complexity cheat-sheet table

- Array/ArrayList: `O(1)` access, `O(n)` search/insert-mid; ArrayList add-at-end amortized `O(1)`.
- LinkedList: `O(n)` access/search, `O(1)` ends.
- HashMap/HashSet: `O(1)` avg, `O(log n)` worst (treeified).
- TreeMap/TreeSet: `O(log n)`, sorted.
- ArrayDeque (stack/queue): `O(1)` ends.

Sorts: merge/heap `O(n log n)`; quick `O(n log n)` avg / `O(n²)` worst; TimSort `O(n log n)` (Java/Kotlin default for objects).

---

## 5. Array vs ArrayList

Array is fixed-size, contiguous, no boxing for primitives (`IntArray`). ArrayList is growable (array-backed, auto-resizes ~1.5×, boxes `Int`→`Integer`). Both `O(1)` index access; mid-insert/remove is `O(n)`. `ArrayList.add` is amortized `O(1)` because geometric resizing makes the occasional `O(n)` copy rare on average.

---

## 6. LinkedList

A doubly linked list: each node has value + prev/next pointers. `O(1)` insert/delete at ends or a known node, but `O(n)` access/search (must walk). Use for frequent end insertions; avoid for random access. On Android, `ArrayDeque` is usually preferred (better cache locality, no per-node overhead).

---

## 7. HashMap and HashSet

`HashMap` stores key→value in buckets by `hashCode()`; `HashSet` wraps one. `get`/`put`/`remove` are `O(1)` average, `O(log n)` worst (buckets treeify into red-black trees since Java 8). Requires consistent `hashCode()` + `equals()` (override both; immutable keys). No ordering — use `LinkedHashMap` (insertion order) or `TreeMap` (sorted).

---

## 8. TreeMap (and sorted structures)

A red-black self-balancing BST keeping keys sorted with `O(log n)` core ops, plus `firstKey`/`floorKey`/`ceilingKey`/range views. `TreeSet` is the set form. Use over HashMap when you need sorted iteration, nearest-key queries, or range scans — trading `O(1)` for `O(log n)`.

---

## 9. Stack and Queue

Abstract data types by access discipline: **Stack** = LIFO (push/pop/peek), **Queue** = FIFO (enqueue/dequeue/peek). Prefer `ArrayDeque` for both (faster than legacy `Stack`, no node overhead) — `O(1)` push/pop/peek. Use `PriorityQueue` (binary heap) for priority order, concurrent queues for thread safety.

---

## 10. Android-specific: ArrayMap & SparseArray

`HashMap` is memory-hungry (per-entry `Entry` objects, autoboxed keys). **`ArrayMap`** uses two parallel arrays with binary search: `O(log n)` lookup, `O(n)` insert/remove, low memory — good for small object-keyed maps. **`SparseArray`** family uses primitive `int`/`long` keys (no boxing): `O(log n)` lookup, `O(n)` insert (`append()` is `O(1)` ascending). Win is reduced allocations/GC on UI hot paths; use HashMap for large/write-heavy maps.

---

## 11. Data-structure operation complexity comparison

Pick by need: fast index access → Array/ArrayList; key→value lookup → HashMap; sorted/range → TreeMap; insertion order → LinkedHashMap; add/remove at ends → ArrayDeque; min/max → PriorityQueue; membership → HashSet; small int-keyed low-GC map → SparseArray; small object-keyed → ArrayMap.

---

## 12. Common algorithm patterns

- **Two pointers** — `O(n)`/`O(1)` on (often sorted) arrays (e.g. palindrome).
- **Sliding window** — `O(n)` subarray/substring problems.
- **Hashing** — trade space for `O(1)` lookup (Two Sum).
- **Binary search** — `O(log n)` on any monotonic search space.
- **Divide & conquer / recursion** — often `O(n log n)` (merge sort).
- **BFS/DFS** — `O(V+E)` graph/tree traversal (BFS for shortest path, DFS for cycles/topo sort).
- **Dynamic programming** — overlapping subproblems + memoization/tabulation turns exponential into polynomial.

---

## 13. How to analyze complexity in an interview

1. Identify input size(s). 2. Count loops (sequential add, nested multiply). 3. Account for hidden costs (`contains` is `O(n)`, sorting `O(n log n)`). 4. Account for recursion depth (use the recurrence). 5. Drop constants/lower terms. 6. State both time and space (worst/avg/amortized). 7. Then optimize — HashMap to remove an inner search, two pointers/sliding window to remove a nested loop, memoization to collapse recursion.
