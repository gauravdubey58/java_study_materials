# 05 — Garbage Collectors

> Covers: All 7 JVM garbage collectors — Serial, Parallel, CMS, G1, ZGC, Shenandoah, Epsilon — with phases, diagrams, use cases, and comparison.

---

**Navigation**

[← GC Process](04-gc-process.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → GC Tuning](06-gc-tuning.md)

---

## Table of Contents

- [5.1 Serial GC](#51-serial-gc)
- [5.2 Parallel GC](#52-parallel-gc)
- [5.3 CMS — Concurrent Mark-Sweep](#53-cms--concurrent-mark-sweep)
- [5.4 G1 GC — Garbage First](#54-g1-gc--garbage-first)
- [5.5 ZGC — Z Garbage Collector](#55-zgc--z-garbage-collector)
- [5.6 Shenandoah GC](#56-shenandoah-gc)
- [5.7 Epsilon GC](#57-epsilon-gc)
- [5.8 Quick Comparison Table](#58-quick-comparison-table)
- [5.9 Choosing a Collector](#59-choosing-a-collector)

---

## 5.1 Serial GC

**Flag:** `-XX:+UseSerialGC`
**Default for:** Client-class machines (single CPU, < 2 GB RAM)

### How it Works

- **Single-threaded** for both Minor and Major GC
- **Stop-the-world** for all phases — application completely paused
- Young Gen: mark-copy | Old Gen: mark-compact

```
App Thread:   ████████████│                    │████████
GC Thread:               └────────────────────┘
                           Single thread, full STW
```

### Characteristics

| Property | Value |
|---|---|
| GC threads | 1 |
| STW pauses | Always, full duration |
| Fragmentation | None (compaction) |
| Memory overhead | Minimal |
| CPU overhead | Low |
| Throughput | Low |
| Latency | High |

### When to Use

- Development/testing environments
- Small heaps (< 100 MB)
- Single-core machines or constrained environments (containers with 1 CPU)
- Batch jobs where pause time doesn't matter
- Microcontroller/embedded JVM

```bash
-XX:+UseSerialGC
-Xms64m -Xmx256m
```

---

## 5.2 Parallel GC

**Flag:** `-XX:+UseParallelGC`
**Default for:** Java 8 (server-class machines)
**Also known as:** Throughput Collector

### How it Works

- **Multiple GC threads** run in parallel for both Minor and Major GC
- **Stop-the-world** — application paused, but pause is shorter due to parallelism
- Young Gen: parallel mark-copy | Old Gen: parallel mark-compact

```
App Threads:   ████████████│          │████████████
GC Threads:               └──────────┘
                            N threads in parallel STW
```

### Configuring Thread Count

```bash
-XX:ParallelGCThreads=N        # Default: min(CPU count, (CPU count * 5/8 + 3) for > 8 cores)
                                # e.g., 8 cores → 8 threads; 32 cores → 23 threads
```

### Characteristics

| Property | Value |
|---|---|
| GC threads | N (= CPU cores) |
| STW pauses | Always, but shorter |
| Throughput | Highest among STW collectors |
| Latency | Medium-high |
| Heap size | Medium (1–32 GB) |

### When to Use

- Batch processing jobs (MapReduce, ETL)
- Applications where raw throughput > latency
- Java 8 default — still valid for non-interactive workloads
- CI/CD build servers

```bash
-XX:+UseParallelGC
-XX:ParallelGCThreads=8
-Xms4g -Xmx8g
```

---

## 5.3 CMS — Concurrent Mark-Sweep

**Flag:** `-XX:+UseConcMarkSweepGC`
**Status:** ⚠️ **Deprecated Java 9, Removed Java 14** — use G1 instead

### How it Works

CMS performs most of its work **concurrently** with application threads to reduce pause times. It does **not compact** Old Gen — this eventually causes fragmentation.

### CMS Phases

```
Timeline:
│─STW─│────────concurrent─────────│─STW─│──concurrent──│
  IM         CM                     R         CS
```

| Phase | Type | Description |
|---|---|---|
| **Initial Mark (IM)** | STW | Mark objects directly reachable from GC roots |
| **Concurrent Mark (CM)** | Concurrent | Traverse object graph while app runs |
| **Remark (R)** | STW | Re-mark objects changed during concurrent mark |
| **Concurrent Sweep (CS)** | Concurrent | Reclaim garbage while app runs |

### CMS Problems

```
1. Fragmentation (no compaction):
   After many cycles: [live][free][live][free][live][free]
   Large allocation fails → Concurrent Mode Failure → triggers Full GC (compacting STW!)

2. Concurrent Mode Failure:
   If Old Gen fills up before CMS finishes → fall back to Serial Full GC
   → Very long STW pause (worse than just using Parallel GC)

3. CPU overhead:
   Concurrent phases steal 25-50% CPU from the application
```

### Why CMS Was Removed

G1 GC achieves lower latency **with compaction**, making CMS obsolete. When CMS fails over to Serial Full GC, the resulting pause is often worse than G1 in steady state.

---

## 5.4 G1 GC — Garbage First

**Flag:** `-XX:+UseG1GC`
**Default for:** Java 9+ (server-class machines)

### Key Innovation: Region-Based Heap

G1 divides the heap into equal-sized **regions** (1–32 MB each, always a power of 2). Regions are dynamically assigned to Eden, Survivor, Old, or Humongous roles.

```
G1 Heap Layout (example: 32 regions):
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ E  │ E  │ O  │ S  │ E  │ O  │ Hum│ O  │
├────┼────┼────┼────┼────┼────┼────┼────┤
│ O  │free│ E  │ O  │free│ S  │ O  │free│
├────┼────┼────┼────┼────┼────┼────┼────┤
│free│ E  │ O  │free│ E  │ O  │ E  │ O  │
├────┼────┼────┼────┼────┼────┼────┼────┤
│ O  │free│ O  │free│ O  │free│free│free│
└────┴────┴────┴────┴────┴────┴────┴────┘
E=Eden  S=Survivor  O=Old  Hum=Humongous  free=unassigned
```

### G1 Phases

| Phase | Type | Description |
|---|---|---|
| **Young GC** | STW (parallel) | Evacuate Eden + Survivor regions to new Survivor/Old regions |
| **Concurrent Marking** | Concurrent | Mark live objects in Old regions using SATB |
| **Remark** | Short STW | Finalize marking (SATB buffer processing) |
| **Cleanup** | Short STW | Identify completely empty regions; update accounting |
| **Mixed GC** | STW (parallel) | Collect Young + select Old regions with most garbage |
| **Full GC (fallback)** | STW (single thread) | Emergency when concurrent can't keep up — avoid! |

```
Normal operation cycle:
  [Young GC]→[Young GC]→[Young GC]→[Concurrent Mark]→[Mixed GC]→[Mixed GC]→...
       ↑ frequent, short             ↑ concurrent       ↑ targets high-garbage Old regions
```

### Humongous Objects

Objects larger than **50% of a G1 region size** are allocated directly in Humongous regions (which are contiguous Old Gen regions).

```java
// With default region size ~4 MB: objects > 2 MB are Humongous
byte[] large = new byte[3 * 1024 * 1024]; // → Humongous region
```

**Problems with Humongous objects:**
- Bypass Young Gen → counted as Old Gen immediately → trigger concurrent marking early
- Cause fragmentation if they don't align well with region boundaries
- Must be collected as whole regions

**Fix:** `-XX:G1HeapRegionSize=32m` — larger regions raise the Humongous threshold.

### G1 Pause Target

```bash
-XX:MaxGCPauseMillis=200    # Soft target (default 200ms) — G1 tries to stay within this
                            # G1 selects how many Old regions to collect in Mixed GC
                            # based on this target — NOT a hard guarantee
```

### When to Use G1

- General-purpose server applications (Java 9+ default)
- Heaps 4 GB – 100 GB
- Mixed latency/throughput requirements
- When you're currently using CMS and want to migrate

```bash
-XX:+UseG1GC
-Xms4g -Xmx8g
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=16m     # For large heaps
-XX:G1NewSizePercent=20      # Min Young Gen size
-XX:G1MaxNewSizePercent=40   # Max Young Gen size
```

---

## 5.5 ZGC — Z Garbage Collector

**Flag:** `-XX:+UseZGC`
**Production since:** Java 15 (experimental Java 11)
**Generational ZGC:** Java 21 (generational mode — default in Java 21)

### Key Design Goals

- Pause times **< 1 ms** (sub-millisecond) regardless of heap size
- Supports heaps from 8 MB to **16 TB**
- ~5–15% throughput overhead vs Parallel GC
- Heap compaction done **concurrently**

### Key Innovations

#### 1. Colored Pointers

ZGC encodes GC metadata directly into the 64-bit pointer (using unused high bits on x64):

```
64-bit ZGC pointer:
Bit 63–44: Unused
Bit 43:    Finalizable (object has finalizer)
Bit 42:    Remapped (reference already updated after relocation)
Bit 41:    Marked1 (live in current GC cycle)
Bit 40:    Marked0 (live in previous GC cycle)
Bit 39–0:  Actual object address (1 TB addressable)
```

This allows ZGC to track object state through pointer metadata without touching the objects themselves.

#### 2. Load Barrier

Every reference read executes a tiny load barrier that detects and heals stale references:

```java
// App code:
Object obj = someRef.field;

// ZGC load barrier (JIT-injected):
Object obj = someRef.field;
if (obj.badColor()) {
    obj = ZGC_barrier_slowpath(obj); // Remap to new location
    someRef.field = obj;             // Self-heal for next read
}
```

### ZGC Phases (All < 1ms STW except noted)

```
Timeline:
│STW│────────────concurrent───────────────────────│STW│──concurrent──│STW│
  IM           CM                                   R       Relocate    RR
```

| Phase | Type | Duration |
|---|---|---|
| **Pause Mark Start** | STW | < 1 ms — scan GC roots |
| **Concurrent Mark** | Concurrent | Marks all live objects |
| **Pause Mark End** | STW | < 1 ms — finalize marking |
| **Concurrent Prepare for Reloc.** | Concurrent | Identify relocation sets |
| **Pause Relocate Start** | STW | < 1 ms — scan roots for relocation |
| **Concurrent Relocate** | Concurrent | Move objects, update pointers via load barriers |

### ZGC in Java 21 (Generational ZGC)

Java 21 adds **generational support** to ZGC — separate Young and Old generations with more frequent collection of young objects:

```bash
-XX:+UseZGC                     # Generational ZGC is now the default in Java 21
-XX:+ZGenerational              # Explicit flag (Java 21+)
```

### When to Use ZGC

- Ultra-low latency services (real-time trading, gaming, ad serving)
- Very large heaps (50 GB – 1 TB+)
- Latency SLAs of < 10 ms
- Java 21+ (use Generational ZGC)

```bash
-XX:+UseZGC
-Xms16g -Xmx32g
-XX:SoftMaxHeapSize=28g        # Try to keep heap below this (soft limit)
-XX:ZUncommitDelay=300         # Return memory to OS after 300s idle
```

---

## 5.6 Shenandoah GC

**Flag:** `-XX:+UseShenandoahGC`
**Available since:** Java 12 (OpenJDK only; not in Oracle JDK)
**Developed by:** Red Hat

### Key Differentiator

Like ZGC, Shenandoah performs **concurrent compaction** — but uses a different mechanism called **Brooks forwarding pointers** instead of colored pointers.

### Brooks Forwarding Pointers

Every object has an extra forwarding pointer word in its header. During relocation, the old object's forwarding pointer is updated to point to the new location. All accesses go through the forwarding pointer.

```
Before relocation:              During relocation:
obj_old:                        obj_old:
  [fwdPtr → itself]               [fwdPtr → obj_new] ← updated
  [field1]                        [field1]
  [field2]                        obj_new:
                                    [fwdPtr → itself]
                                    [field1] ← copied
                                    [field2] ← copied
```

### Shenandoah Phases

| Phase | Type | Description |
|---|---|---|
| **Init Mark** | STW | Scan GC roots |
| **Concurrent Mark** | Concurrent | Mark all live objects |
| **Final Mark** | STW | Complete marking |
| **Concurrent Cleanup** | Concurrent | Reclaim immediately reclaimable regions |
| **Concurrent Evacuation** | Concurrent | Copy live objects out of collection set |
| **Init Update Refs** | STW | Prepare for reference updating |
| **Concurrent Update Refs** | Concurrent | Update all references to relocated objects |
| **Final Update Refs** | STW | Update GC root references |

### ZGC vs Shenandoah Comparison

| Feature | ZGC | Shenandoah |
|---|---|---|
| Concurrent compaction | ✅ | ✅ |
| Pause times | < 1 ms | < 10 ms |
| Mechanism | Colored pointers + load barriers | Forwarding pointers + read/write barriers |
| Max heap | 16 TB | No practical limit |
| JDK availability | Oracle JDK + OpenJDK | OpenJDK only |
| Generational support | Java 21 | Limited |
| Memory overhead | Higher (pointer coloring) | Medium (forwarding words) |

### When to Use Shenandoah

- Low-latency requirements with OpenJDK deployment
- When ZGC is unavailable (Oracle JDK restrictions)
- Large heaps needing sub-10ms pauses

```bash
-XX:+UseShenandoahGC
-XX:ShenandoahGCMode=iu        # Incremental Update mode (default)
-XX:ShenandoahGCMode=passive   # STW only (testing)
-XX:ShenandoahGCMode=aggressive # Maximum GC pressure (testing)
```

---

## 5.7 Epsilon GC

**Flag:** `-XX:+UseEpsilonGC`
**Available since:** Java 11

### What it Does

Epsilon is a **no-operation (no-op) GC** — it allocates memory normally but **never collects**. The JVM exits with `OutOfMemoryError` when heap is exhausted.

```
Normal GC:  allocate → GC runs → allocate → GC runs → ...
Epsilon:    allocate → allocate → allocate → ... → OutOfMemoryError
```

### Valid Use Cases

```java
// 1. Performance benchmarking — measure allocation rate without GC interference
//    Clean baseline for allocation pressure tests

// 2. Short-lived command-line tools where heap won't fill up
//    java -XX:+UseEpsilonGC -Xmx1g com.example.BatchImporter input.csv

// 3. Testing GC overhead — compare app performance with vs without GC

// 4. Extremely latency-sensitive, bounded-allocation applications
//    (e.g., real-time control systems with known fixed allocation)
```

> ⚠️ **Never use Epsilon GC in long-running production services.** The heap will eventually fill and the JVM will crash.

```bash
-XX:+UnlockExperimentalVMOptions   # Required to enable
-XX:+UseEpsilonGC
-Xms512m -Xmx512m                  # Set Xms == Xmx (no resize)
```

---

## 5.8 Quick Comparison Table

| Collector | Java Default | STW Pauses | Throughput | Latency | Max Heap | Best For |
|---|---|---|---|---|---|---|
| **Serial** | Client JVM | High (full STW) | Low | High | < 1 GB | Tiny apps, single-core |
| **Parallel** | Java 8 | Medium (parallel STW) | Highest | Medium | ~100 GB | Batch, throughput |
| **CMS** | ⚠️ Removed J14 | Low (concurrent) | Medium | Low | ~50 GB | Legacy low-latency |
| **G1** | Java 9+ | Low-medium | High | Low-medium | ~100 GB | General purpose |
| **ZGC** | Java 21 (Gen) | Sub-ms | Medium-high | Ultra-low | 16 TB | Low-latency, huge heap |
| **Shenandoah** | — | Low (< 10ms) | Medium | Very low | Unlimited | OpenJDK, low-latency |
| **Epsilon** | — | None | N/A | N/A | Bounded | Testing, profiling |

---

## 5.9 Choosing a Collector

```
What is your Java version?
  ├── Java 21+  →  Start with default (Generational ZGC or G1)
  ├── Java 17   →  G1 (default) or ZGC for latency
  └── Java 11   →  G1 (default); ZGC experimental

What is your primary concern?
  ├── Maximum THROUGHPUT (batch, ETL)     →  Parallel GC (-XX:+UseParallelGC)
  ├── Balanced (web services, APIs)       →  G1 GC (-XX:+UseG1GC)
  ├── Low LATENCY (< 100ms SLA)          →  G1 with tuned pause target
  ├── Ultra-low LATENCY (< 10ms SLA)     →  ZGC or Shenandoah
  ├── Huge HEAP (> 100 GB)               →  ZGC (-XX:+UseZGC)
  └── Testing / Benchmarking             →  Epsilon (-XX:+UseEpsilonGC)

What is your heap size?
  ├── < 1 GB      →  Serial or Parallel
  ├── 1–32 GB     →  G1 (default)
  ├── 32–100 GB   →  G1 or ZGC
  └── > 100 GB    →  ZGC (up to 16 TB)
```

---

**Navigation**

[← GC Process](04-gc-process.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → GC Tuning](06-gc-tuning.md)

---

*Last updated: 2026 | Java 21 LTS*
