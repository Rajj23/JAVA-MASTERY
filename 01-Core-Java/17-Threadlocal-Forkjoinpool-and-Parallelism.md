# ThreadLocal, ForkJoinPool & Parallelism (Beginner → Strong)


This document completes the core Java concurrency journey.
It explains how Java isolates data per thread, how parallelism is implemented efficiently, and why some “easy” APIs are dangerous in real systems.

### 1. What Problem Are We Solving?

Until now:
*   threads shared data
*   race conditions occurred
*   locks and atomics were used

But many problems require:
*   No sharing at all — each thread should have its own data

**Examples:**
*   request context
*   database session
*   transaction info
*   security user details

Locks are unnecessary here. Java solves this using `ThreadLocal`.

## Part 1 — ThreadLocal

### 2. What Is ThreadLocal?

`ThreadLocal` provides:
*   one value per thread
*   same variable name
*   completely isolated storage

Think of it as:
> “A variable whose value depends on the current thread”

### 3. Simple Example

```java
ThreadLocal<Integer> tl = new ThreadLocal<>();

tl.set(10);
System.out.println(tl.get()); // 10
```

Another thread:

```java
System.out.println(tl.get()); // null
```

Same `ThreadLocal`. Different value per thread.

### 4. Why Normal Variables Cannot Do This

Normal variables:
*   live in heap
*   shared across threads
*   require synchronization

`ThreadLocal` avoids sharing entirely.

### 5. ThreadLocal Internal Working (IMPORTANT)

Internally:

```text
Thread
 └── ThreadLocalMap
       ├── ThreadLocal → value
       ├── ThreadLocal → value
```

**Key facts:**
*   Each `Thread` has its own `ThreadLocalMap`
*   `ThreadLocal` is only the key
*   Data is stored inside the `Thread`

This is a common interview trick question.

### 6. Why ThreadLocal Is Fast

Because:
*   no locks
*   no contention
*   no CAS
*   direct thread-local access

Each thread reads its own map.

### 7. Real-World Usage

`ThreadLocal` is heavily used in:
*   Spring (`RequestContextHolder`)
*   Hibernate (Session management)
*   JDBC (Connection binding)
*   `SecurityContext`
*   Logging frameworks (MDC)

Most backend frameworks rely on `ThreadLocal`.

### 8. InheritableThreadLocal (Parent to Child Propagation)

Standard `ThreadLocal` does **not** pass values to child threads.

```java
ThreadLocal<String> tl = new ThreadLocal<>();
tl.set("Parent");

new Thread(() -> {
    System.out.println(tl.get()); // null
}).start();
```

To solve this, use `InheritableThreadLocal`:

```java
ThreadLocal<String> itl = new InheritableThreadLocal<>();
itl.set("Parent");

new Thread(() -> {
    System.out.println(itl.get()); // Parent
}).start();
```

Use case: Propagating User ID or Transaction ID to async worker threads.

### 9. BIG DANGER — Memory Leaks (CRITICAL)

**Why It Happens**

In thread pools:
*   threads are reused
*   `ThreadLocalMap` lives as long as thread
*   values stay if not removed

If you forget:

```java
threadLocal.remove();
```

Results:
*   memory leaks
*   stale user data
*   security bugs
*   production failures

### 10. Correct Usage Pattern

Always use:

```java
try {
    threadLocal.set(value);
    // business logic
} finally {
    threadLocal.remove();
}
```

This is mandatory, not optional.

### 11. When to Use / Avoid ThreadLocal

**Use when:**
*   per-thread context
*   request-scoped data
*   no data sharing required

**Avoid when:**
*   threads must communicate
*   data must be shared
*   logic depends on execution order

## Part 2 — ForkJoinPool

### 12. Why ThreadPoolExecutor Is Not Enough

`ThreadPoolExecutor` works best for:
*   independent tasks

But not for:
*   recursive algorithms
*   divide-and-conquer logic

**Example:**
*   array splitting
*   recursive computations
*   parallel algorithms

### 13. ForkJoinPool — Core Idea

`ForkJoinPool` is designed for:
*   Recursive parallelism

**Concept:**
*   fork → split task
*   join → combine results
*   repeat until small enough

### 14. Work-Stealing (MOST IMPORTANT)

Each worker thread has its own deque.
*   threads push tasks locally
*   idle threads steal work from others
*   balances load automatically

This is called **work-stealing**.

### 15. Why ForkJoinPool Is Efficient

Because:
*   no central queue
*   reduced contention
*   good CPU cache locality
*   threads rarely idle

### 16. ForkJoinPool vs ThreadPoolExecutor

| Feature | ThreadPoolExecutor | ForkJoinPool |
| :--- | :--- | :--- |
| **Task style** | Independent | Recursive |
| **Queue** | Shared | Per-thread |
| **Load balance** | Central | Work-stealing |
| **Best for** | IO tasks | CPU tasks |

### 17. Common ForkJoinPool Use Cases

*   recursive algorithms
*   parallel computation
*   Java parallel streams
*   array processing

## Part 3 — Parallel Streams

### 18. What Is a Parallel Stream?

```java
list.parallelStream()
    .map(...)
    .forEach(...);
```

Internally:
*   uses `ForkJoinPool.commonPool()`
*   shared across entire JVM

### 19. Configuring Parallelism

By default, the common pool uses `CPU Cores - 1` threads.

To change this globally:

```text
-Djava.util.concurrent.ForkJoinPool.common.parallelism=8
```

> **Warning:** Changing this affects **all** parallel streams in the JVM.

### 20. Why Parallel Streams Are Dangerous

**Problems:**
*   Uses shared common pool
*   Blocking calls cause starvation
*   No control over threads
*   Hard to debug
*   Performance unpredictable

Parallel does not mean faster.

### 21. When Parallel Streams Are Acceptable

**Safe when:**
*   CPU-bound tasks
*   no blocking
*   stateless operations
*   large datasets

**Avoid when:**
*   IO / DB calls
*   synchronized blocks
*   `ThreadLocal` usage
*   small collections

### 22. ThreadLocal + Parallel Streams = Bugs

Because:
*   threads are reused
*   `ThreadLocal` values may leak
*   wrong context appears
*   security issues arise

Frameworks avoid this combination carefully.

### 23. Summary (Slow Revision)

*   `ThreadLocal` isolates data per thread
*   Data stored inside `Thread`
*   Must call `remove()` to avoid leaks
*   `ForkJoinPool` enables recursive parallelism
*   Work-stealing balances load
*   Parallel streams use common pool
*   Parallel ≠ always faster

### 24. Interview Mental Map

*   Per-thread data → `ThreadLocal`
*   Recursive parallelism → `ForkJoinPool`
*   Load balancing → work-stealing
*   Parallel stream risk → common pool
*   ThreadLocal leak → thread reuse