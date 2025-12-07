# Garbage Collection (GC) — Classical Algorithms (Mark-Sweep, Mark-Compact, Copying, Generational GC)



Garbage Collection (GC) is the backbone of Java’s automatic memory management.
This document covers every classical GC algorithm used in JVMs before modern collectors like G1, ZGC, and Shenandoah.

Understanding these foundations is essential before learning modern GC behavior.

## 1. Introduction

In Java, all objects are created in the heap, and memory is reclaimed automatically by the Garbage Collector.

GC’s main responsibilities:
- Identify unreachable objects
- Reclaim memory
- Prevent memory leaks
- Keep the JVM stable

GC solves the problem of manual memory management seen in C/C++.

But GC is not magic—it follows specific algorithms.

## 2. When Is an Object Eligible for Garbage Collection?

An object is eligible for GC when:
- No strong reference points to it

Example:
```java
User u = new User();
u = null; // object eligible for GC
```

Indirect case:
```java
User u1 = new User();
User u2 = new User();
u1 = u2; // old object loses reference → eligible for GC
```

GC does not collect:
- referenced objects
- objects reachable from GC roots

## 3. GC Roots (Starting Point of Reachability)

Garbage Collection does reachability analysis starting at GC Roots.

GC Roots include:
- Local variables in the stack
- Active threads
- Static fields
- JNI references (native code)
- Method parameters

Anything reachable from GC Roots = alive.

Not reachable = garbage.

## 4. Classical Garbage Collection Algorithms

These are the foundation of all JVM GCs.

The three core algorithms are:
- Mark & Sweep
- Mark & Compact
- Copying Collection

Let’s study each deeply.

## 5. Algorithm 1 — Mark & Sweep

This is the simplest and oldest GC algorithm.

### Step 1: Mark Phase

- Start from GC Roots
- Traverse all references
- Mark every reachable (alive) object

“Marking” means tagging objects as alive.

### Step 2: Sweep Phase

- Scan heap from start to end
- Delete all unmarked objects
- Reclaim their memory

### Issues with Mark & Sweep

#### 1. Heap Fragmentation

After sweeping, the heap may look like:

```
|live|   |live|     |free|    |live|   |free|
```

Small free gaps appear → fragmented heap.

Effects:
- New large object cannot fit
- Even if total free memory is enough
- Can cause OutOfMemoryError

#### 2. Stop-the-World Pause

Both marking and sweeping require pausing the application.

During GC:
- all threads STOP
- JVM freezes
- UI stutters, server delays appear

## 6. Algorithm 2 — Mark & Compact (Improved Mark & Sweep)

This solves fragmentation.

### Step 1: Mark Phase

Same as before.

### Step 2: Compact Phase

All live objects are moved to one side of the heap:

```
|live|live|live|live|free|free|free|
```

This removes fragmented holes.

### Benefits of Mark & Compact
- No fragmentation
- Allocating new memory is faster
- Large objects fit easily

### Drawback

Because compaction moves objects:
- More CPU work
- Longer Stop-the-World pauses

This algorithm is commonly used in Old Generation GC.

## 7. Algorithm 3 — Copying Collection (Young Generation GC)

Fastest classical GC algorithm.

Used heavily in Young Generation (Eden + Survivor spaces).

Heap is divided into:
- From-Space
- To-Space

### Step 1: Allocate new objects in From-Space

### Step 2: At GC:

- Copy all live objects from From-Space → To-Space
- Clear entire From-Space
- Swap the roles

This is called semispace copying.

### Benefits
- No fragmentation
- Extremely fast
- Ideal for objects that die young

Which is why it’s perfect for young generation.

### Drawback
- Doubles memory usage

You need To-Space and From-Space.

## 8. Generational Garbage Collection (Very Important)

This builds on the Generational Hypothesis:
- Most objects die young.
- Objects that survive become long-living.

Thus, JVM divides heap into:

### 8.1 Young Generation (Minor GC)

Consists of:
- Eden
- Survivor S0
- Survivor S1

Minor GC uses Copying Algorithm

Steps:
- New objects go to Eden
- Minor GC occurs when Eden fills
- Live objects copied to Survivor spaces
- Objects surviving multiple GCs → promoted to Old Gen

Characteristics of Minor GC:
- Very fast
- Low pause time
- Happens frequently

### 8.2 Old Generation (Major GC)

Used for long-lived objects.

Algorithms:
- Mark & Sweep
- Mark & Compact

Characteristics of Major GC:
- Slow
- Long Stop-the-World pauses
- Happens rarely

## 9. Stop-the-World Explained

During GC:
- All application threads pause
- JVM stops running your program
- Only GC thread runs

Why?

Because:
- Object graph must remain stable
- No thread should modify pointers while GC runs
- Moving objects requires updating references safely

Even modern collectors have brief pauses.

## 10. Comparison (Mark-Sweep vs Mark-Compact vs Copying)

| Algorithm | Speed | Fragmentation | Memory Use | Best For |
| :--- | :--- | :--- | :--- | :--- |
| Mark & Sweep | Medium | Yes | Low | Old Gen |
| Mark & Compact | Slow | No | Low | Old Gen |
| Copying | Fastest | No | High | Young Gen |

## 11. Why Modern GC Exists (G1, ZGC, Shenandoah)

Classical GC has limitations:
- Long pauses
- Fragmentation issues
- Slow compaction
- Cannot use multi-core CPUs efficiently

Modern GC solves:
- pause time issues
- large heap performance
- concurrent cycles
- region-based compaction


## 12. Summary (1-Minute Revision)

- GC removes objects unreachable from GC Roots
- Classical algorithms: Mark-Sweep, Mark-Compact, Copying
- Copying GC is used in young generation (fast)
- Mark-Compact used in old generation (no fragmentation)
- Mark-Sweep causes fragmentation
- Minor GC = fast
- Major GC = slow
- Stop-the-world happens in all classical GCs
- Generational GC = young + old generation strategy

## 13. Interview Questions

1. **Explain Mark & Sweep algorithm.**
   - It is a two-phase GC algorithm. **Mark phase**: Identify alive objects starting from GC Roots. **Sweep phase**: Reclaim memory by deleting unmarked objects.
2. **Why does Mark & Sweep cause fragmentation?**
   - It clears dead objects in place, leaving gaps (holes) between surviving objects, which scatters free memory across the heap.
3. **What is the benefit of Mark & Compact?**
   - It eliminates fragmentation by moving all live objects to one end of the heap, making free memory contiguous.
4. **Why is copying GC used for young generation?**
   - Because most objects in Young Generation die quickly (Generational Hypothesis). Copying live objects (which are few) to a survivor space is extremely fast compared to scanning the whole heap for compaction.
5. **What is Minor GC vs Major GC?**
   - **Minor GC**: Collects Young Generation (Eden + Survivor), happens frequently, and is fast.
   - **Major GC**: Collects Old Generation, happens rarely, is slower, and involves Stop-the-World pauses.
6. **What is the Generational Hypothesis?**
   - The observation that "most objects die young" and "few objects survive for a long time," which justifies splitting the heap into Young and Old generations.
7. **What are GC Roots?**
   - The starting points for reachability analysis. They include local variables (stack), active threads, static variables, and JNI references.
8. **Why do all GC algorithms use Stop-the-World?**
   - To ensure the object graph is consistent and no threads are modifying references while the GC is marking or moving objects.