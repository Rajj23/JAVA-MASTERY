# Concurrent Collections Internals — ConcurrentHashMap & CopyOnWriteArrayList

Concurrency is where Java becomes truly powerful for backend and system-level programming.
This document explains how Java collections achieve thread safety without sacrificing performance.

---


### HashMap

- Not thread-safe
- Can corrupt internal structure
- In Java 7, resizing could cause infinite loops
- Leads to data loss, CPU spikes, undefined behavior

### Collections.synchronizedMap()

- Uses single global lock
- All threads block each other
- Safe but very slow

**Java needed:** Thread safety + scalability + high throughput

This led to **Concurrent Collections**.

---

## 2. What Makes a Collection "Concurrent"?

A concurrent collection:

- Allows multiple threads to operate safely
- Avoids global locks
- Uses fine-grained locking or lock-free techniques
- Scales well on multi-core CPUs

**Key idea:** Reduce contention, not concurrency

---

# Part 1 — ConcurrentHashMap

---

## 3. ConcurrentHashMap Overview

`ConcurrentHashMap` is a high-performance, thread-safe HashMap.

**Guarantees:**

- No `ConcurrentModificationException`
- High concurrency for reads and writes
- No global lock
- Excellent scalability

---

## 4. Evolution of ConcurrentHashMap

### Java 7 Design

- Map divided into segments
- Each segment had its own lock
- Concurrency level limited by segment count

```
[Segment1][Segment2][Segment3][Segment4]
```

**Problems:**

- Fixed concurrency
- Higher memory usage
- Complex structure

### Java 8+ Design (Important)

Java 8 removed segments and introduced:

- CAS (Compare-And-Swap)
- Volatile reads
- Synchronized blocks at bucket level
- Same data structure as HashMap

**Result:**

- Better performance
- Simpler design
- Unlimited concurrency growth

---

## 5. Internal Structure (Java 8+)

```java
Node<K,V>[] table
```

Each bucket may contain:

- Single node
- Linked list
- Red-black tree

Same as HashMap — with concurrency control.

---

## 6. How put() Works Internally

1. Compute hash
2. Find bucket index
3. If bucket empty → CAS insert
4. If bucket not empty:
   - Synchronize only on that bucket
   - Update or add node
5. Treeify if bucket grows large

**Key Insight:** Locking happens per bucket, not per map

---

## 7. CAS (Compare-And-Swap)

CAS is a CPU-level atomic instruction:

```java
if (memory == expected)
    memory = newValue
else
    retry
```

Used in:

- `ConcurrentHashMap`
- `AtomicInteger` / `AtomicLong`
- `LongAdder`
- `ForkJoinPool`

**CAS enables lock-free concurrency.**

---

## 8. Concurrent Reads

- Reads are lock-free
- Visibility ensured using `volatile`
- Writes synchronize only on affected bucket

This provides extremely fast read performance.

---

## 9. Why ConcurrentHashMap Disallows null

No null keys or values allowed.

**Reason:**

`get()` returning null would be ambiguous:

- Key absent?
- Key present with null value?

**In concurrent systems, ambiguity = bugs.**

---

## 10. Iterators in ConcurrentHashMap

- Weakly consistent
- No `ConcurrentModificationException`
- May or may not reflect latest updates

**Reason:** Strong consistency would require locking → bad performance

---

# Part 2 — CopyOnWriteArrayList

---

## 11. What Is CopyOnWriteArrayList?

A thread-safe List optimized for:

- Many reads
- Very few writes

**Common use cases:**

- Event listeners
- Observer lists
- Configuration snapshots

---

## 12. Internal Working

**On write (add/remove):**

1. Acquire lock
2. Create new array
3. Copy old elements
4. Apply modification
5. Replace array reference

**Reads:**

- No locking
- Direct array access

---

## 13. Performance Characteristics

### Reads

- Extremely fast
- Lock-free

### Writes

- Expensive
- Full array copy

**Use only when:** Reads ≫ Writes

---

## 14. Iterator Behavior

- Iterators work on snapshot
- Never throw `ConcurrentModificationException`
- Safe iteration during concurrent modifications

---

# Part 3 — Comparisons

---

## 15. ConcurrentHashMap vs SynchronizedMap

| Feature | ConcurrentHashMap | SynchronizedMap |
|---------|-------------------|-----------------|
| Locking | Bucket-level | Whole map |
| Read concurrency | High | Low |
| Write concurrency | High | Low |
| Null allowed | No | Yes |
| Performance | Excellent | Poor |

---

## 16. CopyOnWriteArrayList vs SynchronizedList

| Feature | CopyOnWriteArrayList | SynchronizedList |
|---------|----------------------|------------------|
| Reads | Lock-free | Locked |
| Writes | Expensive | Moderate |
| Iterators | Safe (snapshot) | Fail-fast |
| Best for | Read-heavy | Balanced |

---

## 17. When to Use What

### Use ConcurrentHashMap when:

- High read/write concurrency
- Caching, counters, session storage
- Multi-core scalability required

### Use CopyOnWriteArrayList when:

- Reads dominate writes
- Listeners, observers, config data

### Avoid:

- `HashMap` in multithreading
- `CopyOnWriteArrayList` for frequent writes
- Synchronized collections for high concurrency

---

## 18. JVM & JIT Perspective

- CAS avoids blocking
- JIT optimizes uncontended synchronized blocks
- Fine-grained locking improves CPU cache locality
- Scales well with increasing CPU cores

---

## 19. Summary (1-Minute Revision)

| Concept | Key Point |
|---------|-----------|
| Normal collections | Break in concurrency |
| ConcurrentHashMap | Scales via bucket-level locking |
| Java 8 changes | Removed segments, uses CAS |
| CAS | Core concurrency primitive |
| Null handling | ConcurrentHashMap forbids null |
| Iterators | Weakly consistent |
| CopyOnWriteArrayList | Copies array on every write |
| Best use case | Read-heavy workloads |

---

## 20. Interview Questions

### How does ConcurrentHashMap work internally (Java 8)?

In Java 8+, `ConcurrentHashMap` uses a `Node<K,V>[]` array similar to `HashMap`. For insertions, it uses **CAS (Compare-And-Swap)** for empty buckets and **synchronized blocks at the bucket level** for non-empty buckets. Reads are lock-free using volatile semantics. This design eliminates the segment-based approach of Java 7 and provides unlimited concurrency scalability.

### Why is ConcurrentHashMap faster than synchronizedMap?

| Aspect | ConcurrentHashMap | synchronizedMap |
|--------|-------------------|-----------------|
| Locking granularity | Per bucket | Entire map |
| Read operations | Lock-free | Requires lock |
| Write operations | Lock only affected bucket | Blocks all operations |

Multiple threads can read and write to different buckets simultaneously in `ConcurrentHashMap`, while `synchronizedMap` serializes all access.

### What is CAS and why is it important?

**CAS (Compare-And-Swap)** is a CPU-level atomic instruction with this logic:

```java
if (currentValue == expectedValue) {
    currentValue = newValue;  // atomic
    return true;
}
return false;  // retry needed
```

**Importance:**
- Enables lock-free concurrency
- No thread blocking or context switching
- Used in `AtomicInteger`, `ConcurrentHashMap`, and other concurrent utilities
- Foundation of non-blocking algorithms

### Why are null keys not allowed in ConcurrentHashMap?

In concurrent environments, `get()` returning `null` is ambiguous:
- Does the key not exist?
- Does the key exist with a null value?

In single-threaded `HashMap`, you can use `containsKey()` to disambiguate. In concurrent scenarios, the state can change between `containsKey()` and `get()` calls, making this check unreliable. Disallowing null eliminates this ambiguity.

### When should you use CopyOnWriteArrayList?

Use `CopyOnWriteArrayList` when:

- **Reads vastly outnumber writes** (e.g., 100:1 ratio or more)
- **Safe iteration is critical** (no `ConcurrentModificationException`)
- **Use cases:** Event listeners, observer patterns, configuration data

Avoid when:
- Frequent modifications occur
- List is large (copying becomes expensive)
- Memory is constrained

### Why don't concurrent iterators throw ConcurrentModificationException?

Concurrent collection iterators are **weakly consistent**:

- They work on a snapshot or allow concurrent modifications
- They may or may not reflect changes made after iterator creation
- No fail-fast mechanism is needed

**Trade-off:** Sacrificing strong consistency for better performance and no blocking. This is acceptable because:
1. Strong consistency would require global locking
2. Most concurrent use cases can tolerate slightly stale data
3. The alternative (fail-fast) is worse in concurrent scenarios