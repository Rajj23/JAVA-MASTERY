# Java Multithreading & Memory Visibility — JMM in Practice

For more depth, visit [Java Threading & Multithreading](https://github.com/BenGJ10/Java-Backend-Mastery/tree/main/Advance%20Java/Threads)

Multithreading bugs are among the hardest problems in backend systems.
They occur not because code is "wrong", but because memory visibility and ordering are misunderstood.

This document explains how threads interact with memory, how the Java Memory Model (JMM) works, and why keywords like `volatile` and `synchronized` are necessary.

---

## 1. What Is a Thread (JVM Reality)

A Java thread:

- Has its own stack
- Shares the same heap with other threads

```
Shared Heap
 ├── Objects
 ├── Static fields
 └── Class metadata

Thread A Stack        Thread B Stack
 ├── Frame A1         ├── Frame B1
 ├── Frame A2         ├── Frame B2
```

**Key Insight:**

- Stacks are thread-private
- Heap is shared

This single fact explains most concurrency problems.

---

## 2. The Real Problem in Multithreading

The biggest problem is not locking. It is **memory visibility**.

Just because one thread writes a value does not mean another thread immediately sees it.

**Reason:** CPU caches + JVM optimizations.

---

## 3. CPU Cache Hierarchy (Why Visibility Breaks)

Modern CPUs use caches:

```
Main Memory (RAM)
       ↑
   L3 Cache
       ↑
   L2 Cache
       ↑
L1 Cache (per CPU core)
```

Each core:

- May cache variables locally
- May delay writing back to RAM
- May not reload updated values

**Example:**

```java
boolean flag = false;

// Thread A
flag = true;

// Thread B
while (!flag) {}
```

Thread B may loop forever.

---

## 4. Java Memory Model (JMM)

The Java Memory Model defines:

- When writes by one thread become visible to another
- What instruction reorderings are allowed
- What synchronization guarantees exist

**JMM answers:** "When is shared data safe to read?"

---

## 5. Happens-Before Relationship (Most Important)

**Definition:**

If A happens-before B, then:

- A's effects are visible to B
- A executes logically before B

Without happens-before: **No visibility guarantee**

---

## 6. Happens-Before Rules

### Program Order Rule

Within one thread:

```java
a = 1;
b = 2;
```

`a = 1` happens-before `b = 2`

### Monitor Lock Rule (synchronized)

```java
synchronized(lock) {
    x = 10;
}
```

Unlocking `lock` happens-before any subsequent locking of `lock`.

### Volatile Rule

```java
volatile boolean flag;
```

Write to volatile → happens-before → subsequent read of same volatile

### Thread Start Rule

```java
t.start();
```

Everything before `start()` happens-before code in the new thread.

### Thread Join Rule

```java
t.join();
```

Everything in the thread happens-before `join()` returns.

---

## 7. The volatile Keyword (Truth vs Myth)

### What volatile DOES:

- Guarantees visibility
- Prevents instruction reordering

### What volatile DOES NOT:

- Does NOT guarantee atomicity

### Correct Use

```java
volatile boolean running = true;

while (running) {
    // work
}

// Thread A:
running = false;
```

Loop exits correctly.

### Incorrect Use

```java
volatile int count = 0;
count++; // NOT atomic
```

Expanded operations:

```
read → increment → write
```

Race condition still exists.

---

## 8. Why synchronized Works

`synchronized` provides:

### Mutual Exclusion

Only one thread enters critical section.

### Memory Visibility

- Flushes writes on exit
- Invalidates caches on entry

So `synchronized` guarantees:

- Atomicity
- Visibility
- Ordering

**Stronger than volatile.**

---

## 9. Instruction Reordering (Critical Concept)

Compilers and CPUs may reorder instructions if single-thread semantics remain correct.

**Example:**

```java
a = 1;
b = 2;
```

May execute as:

```java
b = 2;
a = 1;
```

- Single thread → fine
- Multi-thread → dangerous

---

## 10. Classic Reordering Bug

```java
int x = 0, y = 0;

// Thread A:        // Thread B:
x = 1;              y = 1;
r1 = y;             r2 = x;
```

**Possible outcome:**

```
r1 = 0
r2 = 0
```

This surprises most developers — but it is legal under JMM.

---

## 11. How volatile Prevents Reordering

- Writes before volatile write cannot move after it
- Reads after volatile read cannot move before it

`volatile` acts as a **memory fence**.

---

## 12. Atomic Classes (Brief Introduction)

```java
AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();
```

Internally uses:

- CAS (Compare-And-Swap)
- Volatile
- Retry loops

Provides:

- Atomicity
- Visibility
- Lock-free performance

---

## 13. Bugs You Can Now Explain

After understanding JMM, you can explain:

- Infinite loops with flags
- Stale reads
- Broken double-checked locking
- Production-only concurrency bugs
- Why volatile is necessary
- Why synchronized fixes visibility issues

---

## 14. Summary (1-Minute Revision)

| Concept | Key Point |
|---------|-----------|
| Thread memory | Private stacks, shared heap |
| CPU caches | Break visibility |
| JMM | Defines visibility & ordering |
| Happens-before | Visibility guarantee |
| volatile | Visibility + ordering |
| synchronized | Atomicity + visibility + ordering |
| Instruction reordering | Is real and causes bugs |
| Atomic classes | Use CAS for lock-free operations |

---

## 15. Interview Questions

### What is the Java Memory Model?

The Java Memory Model (JMM) is a specification that defines how threads interact with memory. It specifies:

1. **Visibility rules** — when writes by one thread become visible to other threads
2. **Ordering rules** — what instruction reorderings are permitted
3. **Synchronization semantics** — guarantees provided by `synchronized`, `volatile`, and other constructs

Without JMM, program behavior would depend on CPU architecture, cache implementation, and compiler optimizations, making portable concurrent programming impossible.

### What is happens-before?

**Happens-before** is a relationship between two actions where:

- If action A happens-before action B, then A's effects are guaranteed to be visible to B
- It establishes a partial ordering of memory operations

Key happens-before rules:
- Program order within a thread
- Unlocking a monitor → locking the same monitor
- Writing to volatile → reading the same volatile
- `Thread.start()` → first action in new thread
- Last action in thread → `Thread.join()` returning

### Why is volatile needed?

`volatile` is needed because:

1. **CPU caches** — each core may cache variables locally, not seeing updates from other cores
2. **Compiler optimizations** — the compiler may cache values in registers or reorder reads/writes
3. **No automatic visibility** — without synchronization, writes may never become visible to other threads

`volatile` solves this by:
- Forcing reads from main memory
- Forcing writes to main memory immediately
- Preventing instruction reordering around volatile accesses

### Difference between volatile and synchronized?

| Aspect | volatile | synchronized |
|--------|----------|--------------|
| Atomicity | No | Yes |
| Visibility | Yes | Yes |
| Ordering | Yes | Yes |
| Mutual exclusion | No | Yes |
| Use case | Simple flags, state | Compound actions |
| Performance | Faster | Slower (lock overhead) |

**volatile** is suitable for simple read/write of single variables.
**synchronized** is required when multiple operations must be atomic.

### Can instruction reordering cause bugs?

Yes, instruction reordering can cause serious bugs in multithreaded code.

**Example — Double-checked locking (broken):**

```java
if (instance == null) {
    synchronized(lock) {
        if (instance == null) {
            instance = new Singleton(); // Can be reordered!
        }
    }
}
```

Object construction involves:
1. Allocate memory
2. Initialize fields
3. Assign reference

Reordering may assign reference before initialization, causing another thread to see a partially constructed object.

**Fix:** Make `instance` volatile.

### Why does code work locally but fail in production?

Several factors cause local/production differences:

| Factor | Local | Production |
|--------|-------|------------|
| CPU cores | 1-4 | 16+ |
| Load | Light | Heavy |
| JIT optimization | Minimal | Aggressive |
| Timing | Predictable | Variable |
| Cache behavior | Simple | Complex |

**Root causes:**
- More cores = more parallel execution = more visibility issues
- JIT optimization increases with runtime = more reordering
- Higher load = more context switches = more stale reads
- Race conditions have narrow windows that rarely trigger locally

**Solution:** Always use proper synchronization, never rely on timing.