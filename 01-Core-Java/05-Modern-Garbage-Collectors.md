# Modern Garbage Collectors — G1 GC, ZGC, Shenandoah (Deep JVM Internals)


This document explains the three modern garbage collectors used in today’s JVM implementations for large-scale, low-latency Java applications.

Traditional GC algorithms struggle with large heaps and long pause times. Modern GCs solve these issues with region-based memory, concurrent phases, and intelligent compaction.

## 1. Why Modern GC Was Created

Classical GC (Mark-Sweep, Mark-Compact, Copying) works well for small heaps but fails for:
- High-performance backend systems
- Applications using 8GB, 16GB, 32GB+ heap
- Latency-critical apps
- Multi-core servers

### Problems with Old GC:
- Long Stop-The-World pauses
- Slow compaction
- Heap fragmentation
- Poor scaling for large heaps
- Full GC freezes entire application

Therefore, modern GC algorithms were created.

## 2. Modern GC Core Principles (Common Across G1, ZGC, Shenandoah)

Modern GCs use three key ideas:

### 1) Region-Based Heap (Instead of contiguous Young/Old spaces)

Heap is divided into many equal-sized regions (1–32MB).

```
+----+----+----+----+----+----+
| R1 | R2 | R3 | R4 | R5 | R6 |
+----+----+----+----+----+----+
```

Benefits:
- Selective collection
- Predictable memory layout
- Reduced fragmentation
- Faster allocation and compaction

### 2) Concurrent Phases

GC no longer blocks the application for long.

Modern GC does:
- concurrent marking
- concurrent compaction
- concurrent pointer updates

Your app continues running during most GC phases.

### 3) Incremental Compaction

Instead of compacting the entire heap at once (long pause), modern GCs compact small regions incrementally.

## 3. G1 GC (Garbage First) — Deep & Complete

G1 GC became the default GC in Java 9+ for server workloads.

### 3.1 G1 Heap Structure — Region-Based

Heap is divided into ~2048 regions:

```
+----+----+----+----+----+----+
| E  | S  | O  | O  | H  | E  |
+----+----+----+----+----+----+
```
E = Eden, S = Survivor, O = Old, H = Humongous

**Humongous Regions**:
Used for very large objects (> 50% of region size).
These can hurt G1 performance.

### 3.2 How G1 Works — Step-by-Step

#### Step 1 — Young GC (Copying Collection)
- Collects Eden + Survivor regions
- Copies live objects to new Survivor/Old regions
- Fast and efficient (copying algorithm)

#### Step 2 — Concurrent Marking (SATB Algorithm)
- Scans object graph while the application runs
- Uses SATB (Snapshot-At-The-Beginning) to maintain correctness
- Finds live and garbage regions

#### Step 3 — Region Selection (Garbage First Strategy)
G1 chooses regions with most garbage first.

Example:
- R1 → 90% garbage
- R2 → 10%
- R3 → 70%
- R4 → 50%

G1 collects R1, R3, R4 first = maximum benefit per pause.

#### Step 4 — Evacuation Pause
- Moves live objects out of garbage-filled regions
- Frees those regions entirely
- This is the only Stop-the-World pause in G1
- Usually short (goal < 20ms)

### 3.3 Pros & Cons of G1 GC

**Pros:**
- Predictable pause times
- Good for heaps 4GB – 32GB
- Region-based compaction reduces fragmentation
- Default GC for servers today

**Cons:**
- Not ideal for ultra-low latency (< 5ms pause)
- Humongous objects degrade performance
- More memory overhead than older GCs

## 4. ZGC — The Ultra-Low Latency Collector

ZGC aims for <1 millisecond pauses, even with multi-terabyte heaps.
This is the most advanced GC in the JVM.

### 4.1 Key Innovations in ZGC

#### 1) Colored Pointers
Each pointer stores metadata bits:
- Marking state
- Remapping state
- Relocation info

This allows ZGC to update object pointers without pausing the JVM.

#### 2) Load Barriers
- Every reference load is intercepted to ensure it points to the correct object version.
- Enables concurrent relocation of objects.

#### 3) Concurrent Compaction
ZGC moves objects while the application is running.

This is a major breakthrough:
- No long compaction pauses
- No fragmentation
- No heavy Old Gen pauses

### 4.2 ZGC Phases (Simplified)
- Concurrent Mark Start
- Concurrent Marking
- Relocation Set Selection
- Concurrent Relocation
- Concurrent Remapping

Only root scanning is STW, which lasts microseconds.

### 4.3 ZGC Strengths
- Pause times < 1ms
- Supports multi-terabyte heaps
- Great for low-latency applications
- No fragmentation
- Near pause-less concurrent compaction

Amazing for:
- High-frequency trading
- Real-time analytics
- Online gaming servers
- Financial systems

### 4.4 ZGC Weaknesses
- Slightly lower throughput compared to G1
- Requires newer JVM versions
- More complex GC tuning

## 5. Shenandoah GC — Low Pause Concurrent Collector (RedHat)

Shenandoah is similar to ZGC in design goals but implemented differently.

### 5.1 Key Features
- Region-based heap
- Concurrent marking
- Concurrent compaction (moves objects without pausing threads)
- Precise low-latency GC
- Available on Linux-based systems

### 5.2 How Shenandoah Works
- Concurrent marking (SATB)
- Build relocation set
- Concurrent evacuation (move objects)
- Concurrent reference updates
- Free old regions

Unlike G1, Shenandoah eliminates most evacuation pauses.

## 6. G1 vs ZGC vs Shenandoah — Comparison Table

| Feature | G1 GC | ZGC | Shenandoah |
| :--- | :--- | :--- | :--- |
| Pause Time | 10–20ms | <1ms | <1ms |
| Heap Size | Up to ~32GB | Up to 4TB | Up to 4TB |
| Compaction | Partial | Concurrent | Concurrent |
| Fragmentation | Possible (humongous objs) | None | None |
| Ideal Use | General servers | Ultra-low latency | Low latency servers |
| Throughput | High | Medium | Medium |

## 7. When To Use Which GC?

**Use G1 GC (default):**
- Microservices
- Web servers
- General backend systems
- You want stable ~20ms pauses

**Use ZGC:**
- Ultra-low latency needed
- Heap > 16GB
- Trading, gaming, real-time data systems

**Use Shenandoah:**
- Low-latency workloads on Linux
- Large heaps
- RedHat environment

## 8. GC Tuning Flags

**Enable G1 GC**
```bash
-XX:+UseG1GC
```

**Enable ZGC**
```bash
-XX:+UseZGC
```

**Enable Shenandoah (OpenJDK)**
```bash
-XX:+UseShenandoahGC
```

**Set Max Pause Goal**
```bash
-XX:MaxGCPauseMillis=10
```

**Set Heap Size**
```bash
-Xms4g -Xmx4g
```

## 9. Summary (1-Minute Revision)

- Modern GC replaces old algorithms for large heaps and low latency
- G1 GC = predictable pauses, region-based, default for servers
- ZGC = sub-millisecond pauses, concurrent compaction, supports TB heaps
- Shenandoah = low pause, concurrent compaction
- Region-based memory layout is the foundation of all modern collectors
- ZGC & Shenandoah handle fragmentation perfectly
- GC choice depends on latency vs throughput requirements

## 10. Interview Questions

1. **What problem does G1 solve compared to classical GC?**
   - It solves the problem of long Stop-the-World pauses and fragmentation by using a region-based heap and targeting regions with the most garbage (Garbage First).
2. **What are regions in G1 GC?**
   - The heap is divided into fixed-size memory blocks (1MB-32MB) called regions. This replaces the contiguous Young and Old generation layout.
3. **Explain SATB marking used in G1 and Shenandoah.**
   - **SATB (Snapshot-At-The-Beginning)**: It ensures that the GC sees the object graph as it existed at the start of concurrent marking. Any changes made during marking are recorded to prevent missing live objects.
4. **How does ZGC achieve sub-millisecond pauses?**
   - By performing compaction (relocation) concurrently with the application threads, using load barriers and colored pointers to handle strict correctness.
5. **What are colored pointers in ZGC?**
   - ZGC uses unused bits in the 64-bit object reference address to store metadata (Marked0, Marked1, Remapped), allowing the GC to track object states without checking headers.
6. **Difference between evacuation (G1) and relocation (ZGC/Shenandoah).**
   - **G1 Evacuation**: Moves objects during a Stop-the-World pause.
   - **ZGC/Shenandoah Relocation**: Moves objects *concurrently* while the application is running.
7. **When would you choose ZGC over G1?**
   - When latency is critical (requirement < 1ms pause) or when using very large heaps (multi-terabytes) where G1 pauses would be too long.
8. **Which GC is best for latency-critical systems?**
   - ZGC or Shenandoah, as they guarantee extremely low pause times regardless of heap size.