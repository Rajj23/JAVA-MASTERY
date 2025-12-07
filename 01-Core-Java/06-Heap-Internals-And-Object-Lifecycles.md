# Heap Internals & Object Lifecycle — Deep JVM Memory Anatomy

This document explains how objects are created, aged, promoted, collected, and managed inside the JVM heap.
An accurate understanding of heap internals is essential for GC tuning, memory optimization, and avoiding performance bottlenecks in backend systems.

## 1. Overview of JVM Heap

The JVM heap is divided into young generation and old generation, each optimized for different object lifetimes.

```
+---------------------------+
|       Old Generation      |
+---------------------------+
|        Young Gen          |
| Eden | Survivor0 | Survivor1 |
+---------------------------+
```

Modern collectors (G1/ZGC/Shenandoah) use regions, but the logical concepts remain the same.

**Key Insight:**
Most objects die young; few objects live long.

This principle shapes the entire heap design.

## 2. Young Generation (Where Objects Are Born)

Young Gen is optimized for fast allocation and quick cleaning.

Contains three spaces:

```
+---------+----------+----------+
|  Eden   | Survivor0| Survivor1|
+---------+----------+----------+
```

### Eden Space
- All new objects are created here.
- Very fast allocations (using TLAB).
- Cleared during every Minor GC.

### Survivor Spaces (S0, S1)
- Store objects that survive Minor GC.
- Used in a copying pattern (From-Space → To-Space).
- Only one survivor space is written to at a time.

## 3. Object Lifecycle Inside the Heap

### Step 1: Object Creation
```java
User u = new User();
```

Goes into Eden (usually inside TLAB for the thread).

### Step 2: Minor GC Runs

When Eden fills, Minor GC begins.

Minor GC:
- Removes dead objects
- Moves live objects from Eden → Survivor space

Example:
Eden → Survivor0 (age = 1)

### Step 3: Aging / Tenuring

Each time an object survives Minor GC, its age increases +1.

```
S0 (age 1) → S1 (age 2)
S1 (age 2) → S0 (age 3)
```

Age stored in object header metadata.

### Step 4: Survivor Space Swapping (Copying GC)

Young Gen uses copying collection:
From-Space → To-Space

Example:
- 1st GC → Eden → S0
- 2nd GC → S0 → S1
- 3rd GC → S1 → S0

This rotation continues until promotion.

### Step 5: Promotion to Old Generation

An object is promoted to Old Gen when:
1. Its age >= MaxTenuringThreshold
   - Default: 15 GC cycles.
2. Survivor space cannot fit the object
   - This triggers premature promotion.

Promotion is expensive and contributes to Old Gen pressure.

## 4. Old Generation (Long-Lived Objects)

Holds:
- Services
- Reusable objects
- Caches
- HTTP clients / DB pools
- Large collections

Collected using:
- Mark-Sweep
- Mark-Compact
- G1's incremental evacuation

Old Gen GC = Major GC
- Slower
- Causes longer STW pauses
- Should be minimized by tuning Young Gen behavior

## 5. Promotion Failures (Very Important)

Promotion failure happens when:
- Old Generation has insufficient space to hold newly tenured objects.

Results in:
- Full GC
- Entire heap scanned
- Mark-compact applied
- Application frozen
- Can take hundreds of ms or seconds

Full GC is the worst-case scenario in production.

## 6. Allocation Failure in Eden or TLAB

- If Eden is full → Minor GC
- If Survivor too small → premature promotion
- If Old Gen full → Major GC → or Full GC
- If TLAB cannot allocate → request new TLAB → may trigger Minor GC

Understanding these flows prevents memory bottlenecks.

## 7. TLAB (Thread-Local Allocation Buffer)

Each thread receives a private slice of Eden:
- Thread 1 → TLAB1
- Thread 2 → TLAB2
- Thread 3 → TLAB3

**Why TLAB exists:**
- Allocation becomes just pointer increment
- No synchronization needed
- Extremely fast (almost C++ level)

If TLAB fills → thread requests a new one → may trigger Minor GC.

## 8. Write Barriers & Card Tables

GC must track references between generations.

Example:
OldGenObject → EdenObject

To track this, JVM uses:

### Write Barrier
- Runs on every reference assignment.
- Ensures GC metadata is updated.

### Card Table
- Divides heap into small “cards”; marks those with cross-generational references.

Used by:
- G1 GC
- CMS
- Parallel GC

## 9. Heap Tuning Concepts

### MaxTenuringThreshold
Controls how many GC cycles an object survives before promotion.

```java
-XX:MaxTenuringThreshold=10
```

- Lower value → faster promotion
- Higher value → more time in Survivor spaces

### SurvivorRatio
Controls survivor space size relative to Eden.

```java
-XX:SurvivorRatio=8
```

Means:
- Eden : Survivor = 8 : 1 : 1

## 10. Full Object Lifecycle Diagram

```
[1] Allocate in Eden  
      ↓  
[2] Minor GC → move to S0 (age 1)  
      ↓  
[3] Next Minor GC → move to S1 (age 2)  
      ↓  
[4] Swap (S1 → S0) with age increment  
      ↓  
[5] If age >= threshold → promote to Old Gen  
      ↓  
[6] If Old Gen cannot accept → Full GC  
```

This diagram explains almost all GC + heap-related behavior.

## 11. Why Heap Internals Matter in Backend Engineering

Understanding heap internals allows you to:
- Diagnose memory leaks
- Avoid unnecessary promotions
- Tune GC for:
  - microservices
  - low-latency applications
  - high-memory workloads
- Avoid Full GC
- Interpret GC logs correctly
- Optimize object allocations in code
- Understand why OutOfMemoryError happens
- Predict application memory behavior

This is essential for production-grade Java engineering.

## 12. Summary (1-Minute Revision)

- Young Gen = Eden + Two Survivor Spaces
- Most objects die in Eden
- Survivor spaces hold objects that survive Minor GC
- Objects age until tenuring threshold
- Promotion to Old Gen happens after several cycles or space exhaustion
- Old Gen contains long-lived objects
- Promotion failure can trigger Full GC
- TLAB speeds up allocation
- Write barriers + card tables track references across generations

## 13. Interview Questions

1. **Explain the lifecycle of a Java object in the heap.**
   - Created in Eden -> Survives Minor GC -> Moved to Survivor Space (S0/S1) -> Ages per GC cycle -> Promoted to Old Gen if age > threshold or Survivor space full -> Collected by Major/Full GC if unreachable.
2. **What are Survivor spaces and why do we need two of them?**
   - They are buffers in Young Gen to hold objects that survived Minor GC but are not yet old enough for Old Gen. Two are used to enable Copying GC (swapping From/To roles) which prevents fragmentation in Young Gen.
3. **What is tenuring?**
   - The process of aging an object. Each time an object survives a Minor GC, its age (tenure) increments.
4. **What triggers promotion to Old Gen?**
   - When an object's age exceeds `MaxTenuringThreshold` (default 15), or if Survivor spaces are full (premature promotion).
5. **What is a promotion failure?**
   - When objects need to be promoted from Young to Old Gen, but Old Gen doesn't have enough contiguous space to hold them. This triggers a Full GC.
6. **How does TLAB improve allocation speed?**
   - TLAB (Thread-Local Allocation Buffer) gives each thread a private chunk of Eden. This allows object allocation to happen via a simple pointer bump without thread synchronization (locks).
7. **Why are write barriers required in the JVM?**
   - To track references from Old Gen to Young Gen. When an Old Gen object is modified to point to a Young Gen object, the write barrier updates the Card Table so GC can find these references without scanning the entire Old Gen.
8. **What is SurvivorRatio and why does it matter?**
   - The ratio between Eden size and one Survivor space size. Example `SurvivorRatio=8` means Eden is 8x larger than S0. It matters because it determines how much space is available for new objects vs. surviving objects.