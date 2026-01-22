# Collectors, Reduce & Mutable Reduction (Deep Internals)

**Topic:** Core Java (Streams API Internals)
**Related API:** `java.util.stream.Stream`, `java.util.stream.Collectors`

This document explores how Java Streams produce results, focusing on the critical distinction between `reduce()` (immutable reduction) and `collect()` (mutable reduction). It delves into the internal mechanics of these terminal operations, which is where most real-world stream bugs and performance issues occur.

---

## 1. Why Terminal Operations Matter

Streams are designed as lazy pipelines. Intermediate operations (like `map`, `filter`) only define the processing steps but do not execute them. Execution is triggered only when a **Terminal Operation** is invoked.

Common terminal operations include:
*   `forEach`: Iterates over elements (side-effects).
*   `reduce`: Combines elements into a single immutable value.
*   `collect`: Accumulates elements into a mutable container.

---

## 2. Immutable Reduction: `reduce()`

`reduce()` is designed to combine a sequence of elements into a single result by repeatedly applying a combining operation.

### 2.1 Conceptual Execution
If you have a stream of numbers `[e1, e2, e3]` and a reduction operation `+` (sum), `reduce` executes as:
`(((0 + e1) + e2) + e3) ...`

This process creates a new value at each step. It is **Immutable Reduction**.

### 2.2 The Three Forms of `reduce()`

#### 1. `reduce(BinaryOperator)`
```java
Optional<Integer> result = list.stream().reduce(Integer::sum);
```
*   **No Initial Identity**: If the stream is empty, the result is hidden in an `Optional`.
*   **Return Type**: `Optional<T>`

#### 2. `reduce(identity, BinaryOperator)`
```java
int sum = list.stream().reduce(0, Integer::sum);
```
*   **Identity**: Proxies as the initial value.
*   **Return Type**: `T` (Always returns a value; 0 if stream is empty).

#### 3. `reduce(identity, accumulator, combiner)`
```java
int sum = list.parallelStream().reduce(
    0,
    (a, b) -> a + b,  // Accumulator: (int, int) -> int
    Integer::sum      // Combiner: (int, int) -> int
);
```
*   **Combiner**: Essential for parallel streams to merge partial results from different threads.

### 2.3 Identity Value Rules (Critical)
The identity value must be an identity for the accumulator function. Mathematically:
`identity ⊕ x = x`

*   **Sum**: 0 (`0 + x = x`)
*   **Product**: 1 (`1 * x = x`)
*   **Min**: `Integer.MAX_VALUE`
*   **Max**: `Integer.MIN_VALUE`

**Common Pitfall**: Using a non-identity value (e.g., initialized to 10 for a sum) will lead to incorrect results in parallel streams because the identity is applied to *each split* of the stream.

---

## 3. Why `reduce()` Is Bad for Collections

Attempting to use `reduce()` to build a `List` or `Map` is a performance anti-pattern and often buggy.

**Bad Example (Do not do this):**
```java
List<Integer> list = stream.reduce(
    new ArrayList<>(),
    (acc, e) -> { acc.add(e); return acc; }, // Mutating accumulator!
    (a, b) -> { a.addAll(b); return a; }
);
```
**Problems:**
1.  **Performance**: `reduce` expects the accumulator to return a new value. If you return a new list every time ($O(N^2)$), it's slow. If you mutate the passed list, you break functional purity.
2.  **Parallel Unsafety**: Shared mutable state in `reduce` without synchronization causes race conditions.

**Rule:** Never use `reduce` to accomplish what `collect` is designed for.

---

## 4. Mutable Reduction: `collect()`

`collect()` is a terminal operation specifically designed for **Mutable Reduction**. It accumulates elements into a mutable container (like `ArrayList`, `HashMap`, or `StringBuilder`) rather than combining them into a single immutable value.

### 4.1 Internal Mechanics of `collect()`
`collect` requires three components:
1.  **Supplier**: Creates a new, empty result container (e.g., `() -> new ArrayList<>()`).
2.  **Accumulator**: Adds a new element to a result container (e.g., `List::add`).
3.  **Combiner**: Merges two result containers (e.g., `List::addAll`).

**Equivalent Logic:**
```java
collect(
  () -> new ArrayList<>(), // Supplier
  (list, e) -> list.add(e), // Accumulator
  (l1, l2) -> l1.addAll(l2) // Combiner
)
```

### 4.2 Why `collect()` Works Well in Parallel
In a parallel stream:
1.  Multiple threads process different segments of the stream.
2.  Each thread calls the **Supplier** to create its *own local container*.
3.  Each thread accumulates elements into its local container (no locking needed).
4.  Finally, the **Combiner** merges these local containers.

This "Divide, Accumulate, Merge" strategy ensures thread safety without explicit synchronization overhead on every element.

---

## 5. The Collectors Utility Class

`java.util.stream.Collectors` provides factory methods for common collectors.

### 5.1 Common Collectors
*   `toList()`, `toSet()`: Standard collections.
*   `toCollection(Supplier)`: Specific collection (e.g., `TreeSet`, `LinkedList`).
*   `joining()`: Concatenates strings.

### 5.2 groupingBy() Internals
```java
Map<String, List<User>> map = users.stream()
    .collect(Collectors.groupingBy(User::getCity));
```
**Internals:**
1.  Creates a `HashMap`.
2.  For each element, computes the classification key (`User::getCity`).
3.  Fetches the list for that key (or creates one).
4.  Adds the element to the list.

### 5.3 partitioningBy() vs groupingBy()
*   **`partitioningBy(Predicate)`**: Always returns a `Map<Boolean, List<T>>`. It is optimized because the map size is known to be exactly 2 (true and false buckets).
*   **`groupingBy(Function)`**: Key can be any type. Result map size is dynamic.

### 5.4 Downstream Collectors
`groupingBy` allows a second collector (downstream) to value processing.

```java
// Count users per city
Map<String, Long> countByCity = users.stream()
    .collect(Collectors.groupingBy(User::getCity, Collectors.counting()));
```
This avoids creating intermediate lists if you only need the count.

---

## 6. `toMap()` Pitfalls (Common Production Bug)

**The Bug:**
```java
Map<Integer, String> map = users.stream()
    .collect(Collectors.toMap(User::getId, User::getName));
```
If the stream contains two users with the *same ID*, `toMap` throws an `IllegalStateException: Duplicate key`.

**The Fix:** Always provide a merge function.
```java
Collectors.toMap(
    User::getId,
    User::getName,
    (existing, replacement) -> existing // Strategy: Keep existing
);
```

---

## 7. Comparison: `reduce` vs `collect`

| Feature | `reduce` | `collect` |
| :--- | :--- | :--- |
| **Concept** | **Immutable Reduction** | **Mutable Reduction** |
| **Mechanism** | Combines values to produce a new value (`T + T -> T`) | Adds values to a container (`Container.add(T)`) |
| **Best For** | Sum, Min, Max, String Concatenation | Lists, Sets, Maps, String Builders |
| **Performance** | $O(N)$ for primitives (very fast) | $O(N)$ for object accumulation |
| **Parallelism** | Requires careful identity & associativity | Safe by design (local containers) |

**Rule of Thumb:**
*   Evaluating a single value? Use `reduce`.
*   Building a data structure? Use `collect`.

---

## 8. Summary regarding Internals

*   **Streams are push-based**: Elements are pushed through the pipeline sinks to the collector.
*   **`reduce`** relies on the properties of the binary operator (associativity is key for parallel).
*   **`collect`** relies on independent container creation (Supplier) to enable contention-free parallel accumulation.

---

## 9. Interview Questions

1.  **What is the difference between `reduce()` and `collect()`?**
    *   `reduce` performs immutable reduction (combining values), while `collect` performs mutable reduction (accumulating into a container). Using `reduce` for lists behaves like $O(N^2)$ or mutates state unsafely.

2.  **Why shouldn't you use `reduce` to add elements to a List?**
    *   Because `reduce` is conceptually designed to return a *new* value each time. Mutating the passed list violates functional purity and is not thread-safe in parallel streams.

3.  **How does `collect` handle parallelism without synchronization?**
    *   It gives each thread its own result container (created via the Supplier). Threads accumulate locally, and then containers are merged using the Combiner.

4.  **What happens if `toMap` encounters duplicate keys?**
    *   It throws `IllegalStateException`. You must provide a merge function `(v1, v2) -> v1` to handle collisions.
