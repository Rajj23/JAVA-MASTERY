# Threads, Lifecycle & ThreadPool Internals (Beginner → Strong)

**Day 14 – Core Java Mastery**

This document explains threads and thread pools from absolute basics, step by step.
No assumptions are made about prior knowledge.

If you understand this file properly, you will never be confused about multithreading again.

---

## 1. What Is a Thread? (Very Basic)

A thread is a unit of execution inside a program.

- One program (process) can run multiple threads
- Each thread performs tasks independently
- Threads run in parallel (or appear to)

Every Java program starts with one thread:

```java
public static void main(String[] args) {
    // main thread
}
```

---

## 2. Threads vs Process (Simple Understanding)

| Term    | Description                              |
|---------|------------------------------------------|
| Process | Running application (JVM)                |
| Thread  | Path of execution inside that process    |

```
Process (JVM)
 ├── Thread-1
 ├── Thread-2
 └── Thread-3
```

**Threads share:**
- Heap memory
- Static variables

**Threads do NOT share:**
- Stack memory

---

## 3. What Happens When You Create a Thread?

**Code:**

```java
Thread t = new Thread(() -> {
    System.out.println("Hello");
});
t.start();
```

**Internally (Step-by-Step):**

1. Java creates a Thread object in heap
2. JVM asks the Operating System to create a native thread
3. OS allocates:
   - Native stack (~1MB)
   - Scheduling data
4. JVM links Java thread ↔ OS thread
5. Thread becomes runnable

> **Note:** Threads are OS-level resources, not lightweight Java objects.

---

## 4. Why Creating Threads Is Expensive

Each thread costs:
- Memory (stack)
- CPU scheduling
- Context switching

**Bad code:**

```java
for (int i = 0; i < 10000; i++) {
    new Thread(task).start();
}
```

**Problems:**
- Huge memory usage
- CPU thrashing
- `OutOfMemoryError: unable to create native thread`

---

## 5. Thread Lifecycle (Very Clear)

Thread states in Java:

```
NEW → RUNNABLE → WAITING / BLOCKED → TERMINATED
```

### NEW

```java
Thread t = new Thread(task);
```

- Thread object exists
- OS thread NOT created yet

### RUNNABLE

```java
t.start();
```

- Thread is ready to run
- OS decides when it actually runs

> **Important:** Java has no `RUNNING` state.

### BLOCKED

Thread waits to acquire a lock:

```java
synchronized(lock) {
   // another thread owns lock
}
```

### WAITING

Thread waits indefinitely:

```java
obj.wait();
thread.join();
```

### TIMED_WAITING

Thread waits for fixed time:

```java
Thread.sleep(1000);
obj.wait(1000);
```

### TERMINATED

Thread finished execution.

---

## 6. Why Manual Thread Creation Is a Problem

If every task creates a new thread:
- Too many threads
- Memory waste
- CPU context switching
- Unpredictable performance

**Result:**
- Not scalable
- Not production-safe

---

## 7. Solution: Thread Pool (Core Idea)

**Thread pool** = reusable group of threads

Instead of:

```java
new Thread(task).start();
```

We do:

```
submit task → pool → existing thread executes
```

**Benefits:**
- Threads reused
- Controlled number of threads
- Stable performance

---

## 8. Executor Framework (Why It Exists)

Java provides **Executor Framework** to manage threads safely.

**Key interfaces:**
- `Executor`
- `ExecutorService`

---

## 9. Executor vs ExecutorService

### Executor

```java
void execute(Runnable task);
```

- Just runs task
- No lifecycle control

### ExecutorService

Adds:
- Submit tasks
- Manage shutdown
- Return `Future`
- Reuse threads

> **Note:** Real applications use `ExecutorService`.

---

## 10. ThreadPoolExecutor (Most Important Class)

All thread pools are built on:

```java
ThreadPoolExecutor
```

**Constructor (simplified):**

```java
ThreadPoolExecutor(
  corePoolSize,
  maximumPoolSize,
  keepAliveTime,
  timeUnit,
  workQueue
)
```

Understanding this = understanding thread pools.

---

## 11. corePoolSize

Minimum number of threads always kept alive.

**Example:**

```java
corePoolSize = 2
```

**Meaning:**
- Pool creates 2 threads
- They stay alive even when idle

---

## 12. maximumPoolSize

Maximum threads allowed.

**Example:**

```java
maximumPoolSize = 5
```

**Meaning:**
- Pool can grow from 2 → 5 threads
- Beyond that → no new threads allowed

---

## 13. workQueue (Task Waiting Area)

Queue stores tasks when threads are busy.

Think: "Waiting line for tasks"

**Example:**

```java
BlockingQueue<Runnable>
```

---

## 14. Task Submission Flow (VERY IMPORTANT)

When you submit a task:

```java
executor.execute(task);
```

JVM follows strict steps:

| Step | Condition                              | Action              |
|------|----------------------------------------|---------------------|
| 1    | Running threads < corePoolSize         | Create new thread   |
| 2    | Queue NOT full                         | Put task in queue   |
| 3    | Threads < maximumPoolSize              | Create new thread   |
| 4    | None of the above                      | Reject task         |

This logic explains everything about thread pool behavior.

---

## 15. keepAliveTime

Extra threads (beyond core) should not live forever.

```java
keepAliveTime = 60 seconds
```

**Meaning:**
- Extra threads die after 60s of idle time

---

## 16. Why Executors Utility Methods Are Dangerous

```java
Executors.newFixedThreadPool(10);
```

**Internally:**
- Uses unbounded queue

**If tasks come fast:**
- Queue grows forever
- Memory leak
- `OutOfMemoryError`

> **Best Practice:** Always prefer custom `ThreadPoolExecutor`.

---

## 17. Simple Mental Model

```
Tasks → Queue → Threads → CPU
```

**You control:**
- Number of threads
- Queue size
- Rejection behavior

---

## 18. Summary (Slow Revision)

| Concept                  | Key Point                               |
|--------------------------|-----------------------------------------|
| Threads                  | OS-level and expensive                  |
| Many threads             | Dangerous                               |
| Thread pools             | Reuse threads                           |
| ThreadPoolExecutor       | Controls behavior                       |
| corePoolSize             | Minimum threads                         |
| maxPoolSize              | Maximum threads                         |
| Queue                    | Holds waiting tasks                     |
| Submission               | Follows 4 strict steps                  |

---

## 19. Interview Questions

### Q1: Why are threads expensive?

**Answer:** Threads are expensive because each thread requires:
- **Memory allocation:** ~1MB stack space per thread
- **OS resources:** Native thread creation involves system calls
- **CPU scheduling overhead:** More threads mean more context switching
- **Synchronization costs:** Managing shared resources across threads

### Q2: What is the thread lifecycle?

**Answer:** A thread in Java goes through these states:
1. **NEW:** Thread object created but `start()` not called
2. **RUNNABLE:** Thread is ready to run (includes both ready and running)
3. **BLOCKED:** Waiting to acquire a monitor lock
4. **WAITING:** Waiting indefinitely for another thread (`wait()`, `join()`)
5. **TIMED_WAITING:** Waiting for a specified time (`sleep()`, `wait(timeout)`)
6. **TERMINATED:** Thread has completed execution

### Q3: Why does Java not have RUNNING state?

**Answer:** Java combines "ready to run" and "actually running" into a single `RUNNABLE` state because:
- The JVM cannot reliably distinguish between them
- The OS scheduler controls when a thread actually executes
- A thread can switch between ready and running many times per second
- This abstraction keeps the API simpler and platform-independent

### Q4: Why do we need thread pools?

**Answer:** Thread pools are needed because:
- **Resource control:** Limit the number of concurrent threads
- **Performance:** Avoid overhead of creating/destroying threads repeatedly
- **Stability:** Prevent system overload from too many threads
- **Queue management:** Handle task overflow gracefully
- **Lifecycle management:** Proper shutdown and cleanup

### Q5: Explain task submission flow in ThreadPoolExecutor

**Answer:** When a task is submitted:
1. If `runningThreads < corePoolSize` → Create new thread
2. Else if queue is not full → Add task to queue
3. Else if `runningThreads < maximumPoolSize` → Create new thread
4. Else → Reject the task using RejectedExecutionHandler

### Q6: Why are Executors utility methods dangerous?

**Answer:** Methods like `Executors.newFixedThreadPool()` are dangerous because:
- They use **unbounded queues** (`LinkedBlockingQueue`)
- If tasks arrive faster than processing, queue grows indefinitely
- This leads to **memory exhaustion** and `OutOfMemoryError`
- **Best practice:** Use `ThreadPoolExecutor` directly with bounded queues