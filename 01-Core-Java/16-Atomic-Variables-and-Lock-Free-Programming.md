# Atomic Variables & Lock-Free Programming (Beginner → Strong)



This document explains how Java achieves thread safety without using locks.
It focuses on atomic variables, CAS, LongAdder, and performance concepts like false sharing.

Understanding this topic is critical for:
*   high-performance backend systems
*   metrics & counters
*   concurrent libraries
*   JVM-level interviews

### 1. Why Lock-Free Programming Exists

Locks (`synchronized`, `ReentrantLock`) are correct but expensive.

**Problems with locks:**
*   threads block
*   OS context switching
*   reduced CPU utilization
*   scalability issues under high contention

For simple shared variables (counters, stats):
*   Blocking is unnecessary and slow

So Java introduced lock-free programming.

### 2. What Is an Atomic Operation?

An atomic operation:
*   Executes completely or not at all — no partial execution.

Example (NOT atomic):

```java
count++;
```

Internally:
1.  read
2.  increment
3.  write

Multiple steps → race condition.

### 3. CAS — Compare And Swap (Core Concept)

CAS is a CPU-level atomic instruction.

CAS logic:

```java
if (currentValue == expectedValue)
    currentValue = newValue
else
    fail
```

This happens:
*   atomically
*   without locks
*   directly in hardware

### 4. CAS Simple Analogy

Shared value = 10.

Thread says:
> “Change it to 11 only if it’s still 10.”

If someone already changed it → fail → retry.

That retry loop is how CAS works.

### 5. AtomicInteger — What It Really Does

```java
AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();
```

Internally:
1.  Read current value
2.  Try CAS (expected → new)
3.  If CAS fails → retry
4.  Loop until success

This is called a **CAS retry loop**.

### 6. Why AtomicInteger Is Thread-Safe

Atomic variables provide:
*   atomicity → via CAS
*   visibility → via `volatile`
*   ordering → via memory barriers

No locks. No blocking.

### 7. Atomic vs Lock Performance

**With locks:**
*   thread blocks
*   context switch
*   OS overhead

**With CAS:**
*   thread retries
*   stays on CPU
*   no blocking

> **Note:** Low contention → atomic variables are much faster

### 8. The Problem with CAS (Important)

CAS works great until:
*   Many threads update the same variable

Then:
*   CAS fails repeatedly
*   threads spin
*   CPU wasted

This is called **high contention**.

### 9. Why LongAdder Exists

Problem:

```java
AtomicLong counter;
```

Under heavy load:
*   too many CAS retries
*   performance collapses

Solution:
*   Avoid contention by spreading updates

### 10. How LongAdder Works (Conceptual)

Instead of one variable:

```text
counter = base + cell1 + cell2 + cell3 + ...
```

*   threads update different cells
*   almost no contention

Final value = sum of all cells

### 11. AtomicLong vs LongAdder

| Feature | AtomicLong | LongAdder |
| :--- | :--- | :--- |
| **Contention** | High under load | Very low |
| **Memory** | Low | Higher |
| **Exact value always** | Yes | No (during updates) |
| **Best use** | Low contention | High contention |

> Metrics & counters → `LongAdder`

### 12. Why LongAdder Is Not Always Used

Because:
*   more memory usage
*   value not immediately consistent
*   summing cost

So:
*   Use `LongAdder` only when contention is high

### 13. Volatile + CAS (Hidden Detail)

Atomic variables use:
*   `volatile` → visibility
*   CAS → atomicity

This combination gives lock-free correctness.

### 14. ABA Problem (Advanced Concept)

CAS checks:

```java
expected == current
```

But:
*   A → B → A

CAS thinks nothing changed.

This is called the **ABA problem**.

Java provides:
*   `AtomicStampedReference`
*   `AtomicMarkableReference`

(Used in advanced lock-free structures.)

### 15. False Sharing (Performance Killer)

**What Is False Sharing?**
*   two threads update different variables
*   variables lie in same CPU cache line
*   cache invalidation happens

Result:
*   massive slowdown
*   unexpected performance drop

### 16. Why False Sharing Happens

CPU caches operate on cache lines (≈ 64 bytes).

If two variables share a line:
*   updating one invalidates the other

Threads fight indirectly.

### 17. How Java Avoids False Sharing

Java uses:
*   padding (extra unused fields)
*   `@Contended` annotation
*   careful memory layout

`LongAdder` is designed to avoid false sharing.

### 18. When to Use What (Clear Rules)

**Use synchronized / ReentrantLock when:**
*   complex logic
*   multiple shared variables
*   correctness is priority

**Use AtomicInteger / AtomicLong when:**
*   simple counters
*   low contention

**Use LongAdder when:**
*   high-frequency updates
*   metrics & stats
*   many threads

### 19. Summary (Slow Revision)

*   Locks block threads
*   CAS is lock-free
*   Atomic classes use CAS + `volatile`
*   CAS retries under contention
*   `LongAdder` spreads contention
*   False sharing hurts performance
*   Java designs for CPU cache behavior

### 20. Interview Mapping

*   “What is CAS?” → compare & swap
*   “Atomic vs Lock?” → lock-free vs blocking
*   “Why LongAdder?” → reduce contention
*   “False sharing?” → cache line issue
*   “Why Atomic is fast?” → no blocking