# Locks & Synchronizers (Beginner → Internal Understanding)


Concurrency is not only about creating threads — it is about controlling how threads access shared data.

This document explains why locks exist, how they work internally, and when to use which synchronization tool.

### 1. The Core Problem: Shared Mutable Data

Consider this code:

```java
int count = 0;

void increment() {
    count++;
}
```

When two threads execute `increment()` simultaneously, the result may be wrong.

**Why?**

Because `count++` is not a single operation. It expands to:

1.  read count
2.  increment
3.  write count

Threads can interleave these steps, causing a race condition.

### 2. Why volatile Is NOT Enough

```java
volatile int count;
```

This does not fix the problem.

**Why?**
*   `volatile` ensures visibility
*   but does **NOT** ensure atomicity

So we need a stronger mechanism.

### 3. Mutual Exclusion (The Core Idea)

We need:
*   Only one thread at a time should modify shared data

This concept is called **mutual exclusion**, and Java provides multiple tools to achieve it.

## Part 1 — synchronized

### 4. What Is synchronized?

`synchronized` is a Java keyword that:
*   Allows only one thread inside a critical section
*   Guarantees visibility and ordering

Example:

```java
synchronized void increment() {
    count++;
}
```

### 5. What Is the Lock in synchronized?

Every Java object has an intrinsic lock (monitor).

```java
synchronized(obj) {
    // critical section
}
```

*   Thread must acquire `obj`’s monitor
*   Other threads block until it is released

### 6. Method vs Block Synchronization

**Method-level**

```java
synchronized void method() { }
```
*   Lock = `this`

**Block-level**

```java
synchronized(lockObj) { }
```
*   Lock = `lockObj`

Block-level synchronization is more flexible and recommended.

### 7. JVM Internals of synchronized

The JVM uses bytecode instructions:
*   `monitorenter`
*   `monitorexit`

These instructions:
*   acquire/release locks
*   act as memory barriers

This is why `synchronized` fixes visibility issues too.

### 8. Reentrancy

Java locks are reentrant.

This means:
*   same thread can acquire same lock multiple times
*   avoids self-deadlock

### 9. Limitations of synchronized

*   No try-lock
*   No fairness control
*   Cannot interrupt waiting threads
*   Single condition per lock

For advanced use cases → use `ReentrantLock`.

## Part 2 — ReentrantLock

### 10. What Is ReentrantLock?

`ReentrantLock` is a lock implementation that provides:
*   same safety as `synchronized`
*   more control and flexibility

Basic usage:

```java
Lock lock = new ReentrantLock();

lock.lock();
try {
    count++;
} finally {
    lock.unlock();
}
```

### 11. Why try–finally Is Mandatory

If `unlock()` is not called:
*   lock is never released
*   system deadlocks

**Always use try–finally.**

### 12. Advanced Features of ReentrantLock

**tryLock()**

```java
if (lock.tryLock()) {
    // safe attempt
}
```

**Interruptible Lock**

```java
lock.lockInterruptibly();
```

**Fair Locks**

```java
new ReentrantLock(true);
```

*   Fair = first-come-first-served
*   Unfair (default) = better performance

### 13. When to Use ReentrantLock

Use when:
*   you need try-lock
*   fairness is required
*   interruption support is needed
*   multiple condition queues are needed

## Part 3 — ReadWriteLock

### 14. Problem It Solves

In read-heavy systems:
*   many threads read data
*   few threads write data

Using a single lock:
*   readers block readers (Inefficient)

### 15. ReadWriteLock Concept

*   Multiple readers allowed
*   Only one writer allowed
*   Writers block readers

```java
ReadWriteLock rw = new ReentrantReadWriteLock();
```

Usage:

```java
rw.readLock().lock();
rw.writeLock().lock();
```

### 16. When to Use ReadWriteLock

Use when:
*   reads ≫ writes
*   shared data is mostly read
*   performance matters

## Part 4 — Semaphore

### 17. What Is a Semaphore?

A Semaphore controls how many threads can access a resource.

Example:
*   only 3 DB connections allowed

```java
Semaphore sem = new Semaphore(3);
```

### 18. How Semaphore Works

```java
sem.acquire(); // take permit
// critical section
sem.release(); // return permit
```

If no permit is available:
*   thread blocks

### 19. Semaphore vs Lock

*   Lock → one thread at a time
*   Semaphore → N threads at a time

## Part 5 — CountDownLatch

### 20. Problem It Solves

Wait until N tasks complete, then proceed.

Example:
*   wait for multiple services to start

### 21. How CountDownLatch Works

```java
CountDownLatch latch = new CountDownLatch(3);
```

**Worker threads:**

```java
latch.countDown();
```

**Main thread:**

```java
latch.await();
```

When count reaches zero → main thread continues.

### 22. Limitation of CountDownLatch

*   Cannot be reused
*   One-time synchronization only

## Part 6 — CyclicBarrier

### 23. What Is CyclicBarrier?

All threads must reach a point together.

Example:
*   all players ready → start game

### 24. How CyclicBarrier Works

```java
CyclicBarrier barrier = new CyclicBarrier(3);
```

**Each thread:**

```java
barrier.await();
```

When all arrive:
*   threads proceed together
*   barrier resets automatically

### 25. CountDownLatch vs CyclicBarrier

| Feature | CountDownLatch | CyclicBarrier |
| :--- | :--- | :--- |
| **Reusable** | No | Yes |
| **Direction** | Workers → main | All threads |
| **Use case** | One-time wait | Repeated phases |

### 26. Summary (Slow Revision)

*   **Race conditions** come from shared data
*   `volatile` ≠ atomicity
*   `synchronized` provides safety + visibility
*   `ReentrantLock` adds flexibility
*   `ReadWriteLock` improves read-heavy performance
*   `Semaphore` limits concurrency
*   `CountDownLatch` waits for tasks
*   `CyclicBarrier` syncs phases

### 27. Interview Mapping

*   One thread at a time → `synchronized` / `ReentrantLock`
*   Many readers → `ReadWriteLock`
*   Limit access → `Semaphore`
*   Wait for tasks → `CountDownLatch`
*   Sync phases → `CyclicBarrier`