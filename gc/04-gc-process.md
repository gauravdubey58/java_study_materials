# 04 — The GC Process

> Covers: Marking phase, sweeping vs copying vs compaction, stop-the-world pauses, concurrent vs parallel GC, and write barriers.

---

**Navigation**

[← Generational GC](03-generational-gc.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Garbage Collectors](05-garbage-collectors.md)

---

## Table of Contents

- [4.1 Phase 1: Marking](#41-phase-1-marking)
- [4.2 Phase 2: Sweeping / Copying](#42-phase-2-sweeping--copying)
- [4.3 Phase 3: Compaction](#43-phase-3-compaction)
- [4.4 Stop-The-World (STW) Pauses](#44-stop-the-world-stw-pauses)
- [4.5 Concurrent vs Parallel GC](#45-concurrent-vs-parallel-gc)
- [4.6 Write Barriers](#46-write-barriers)
- [Summary](#summary)

---

## 4.1 Phase 1: Marking

Marking identifies which objects are **live** (reachable) and which are **dead** (garbage).

### Tri-Color Marking Algorithm

Modern GCs use **tri-color marking** to track the state of each object during traversal:

```
White  = Not yet visited (initially all objects are white)
Grey   = Visited but children not yet scanned
Black  = Visited and all children scanned (live — will not be collected)

Algorithm:
  1. Start: all objects White, GC roots → Grey
  2. Pick any Grey object:
       Mark it Black
       For each referenced child:
         if White → make Grey (add to work queue)
  3. Repeat until no Grey objects remain
  4. All remaining White objects = garbage
```

```
Initial state:
  Root → [A:grey] → [B:white] → [C:white]
                 → [D:white]

After scanning A:
  Root → [A:black] → [B:grey] → [C:white]
                  → [D:grey]

After scanning B:
  Root → [A:black] → [B:black] → [C:grey]
                  → [D:grey]

After scanning C, D:
  Root → [A:black] → [B:black] → [C:black]
                  → [D:black]
  [E:white] [F:white]  ← no path from root → GARBAGE
```

### Marking in Practice

```bash
# What GC traces from (GC Roots):
  - Local variables in active thread stacks
  - Static fields of loaded classes
  - Active monitor objects (synchronized)
  - JNI global references

# Example: finding a memory leak
# If a static List holds thousands of objects → all are GC roots → never collected
static List<UserSession> activeSessions = new ArrayList<>();
// If sessions are never removed → memory leak → Old Gen fills → Full GC loops
```

### Concurrent Marking Challenge

When marking runs concurrently with the application, the application can **modify the object graph** during marking — creating the risk of **floating garbage** or worse, **missing live objects** (premature collection of live data).

The **Snapshot-At-The-Beginning (SATB)** algorithm (used by G1) and **incremental update** (used by CMS) solve this with write barriers (see §4.6).

---

## 4.2 Phase 2: Sweeping / Copying

After marking, the GC reclaims memory from dead (white) objects. Three strategies exist:

### Strategy 1: Mark-Sweep

```
Before:  [A:live][B:dead][C:live][D:dead][E:live][F:dead]
After:   [A:live][     ][C:live][      ][E:live][      ]
                  ↑ free hole    ↑ free hole      ↑ free hole
```

- **Fast** — just update a free list with dead object locations
- **Fragmentation** — free space is scattered; large allocations may fail even with enough total free memory
- Used by: CMS (for Old Gen)

### Strategy 2: Mark-Copy (Copying Collection)

```
From-space (Eden + S0):         To-space (S1):
[A:live][B:dead][C:live]   →   [A:live][C:live]
[D:dead][E:live][F:dead]        [E:live]
         ↑ Compacted, no gaps
```

- **No fragmentation** — allocation is just a pointer bump after copy
- **Fast allocation** — pointer bump allocation is ~1 ns
- **Requires 2× space** — half of memory is always empty (for the destination)
- Used by: Young Generation in all collectors

### Strategy 3: Mark-Compact

```
Before:  [A:live][B:dead][C:live][D:dead][E:live]
After:   [A:live][C:live][E:live][          free         ]
                                   ↑ All free space is contiguous
```

- **No fragmentation** — all live objects moved to one end
- **Pointer updates required** — every reference to moved objects must be updated
- **Expensive** — O(n) for live objects; updating all references takes time
- Used by: Old Gen in Serial, Parallel, G1 (mixed GC), Full GC fallback

### Comparison

| Strategy | Fragmentation | Speed | Space overhead | Used in |
|---|---|---|---|---|
| Mark-Sweep | High | Fast | Low | CMS Old Gen |
| Mark-Copy | None | Fastest | High (50%) | Young Gen (all collectors) |
| Mark-Compact | None | Slow | Low | Old Gen (Serial, Parallel, G1) |

---

## 4.3 Phase 3: Compaction

Compaction moves all live objects to one end of the heap, creating a single large contiguous free space.

```
Fragmented heap (after many mark-sweep cycles):
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ A  │free│ B  │free│free│ C  │free│ D  │
└────┴────┴────┴────┴────┴────┴────┴────┘

After compaction:
┌────┬────┬────┬────┬────────────────────┐
│ A  │ B  │ C  │ D  │    free            │
└────┴────┴────┴────┴────────────────────┘
                      ↑ large contiguous block
```

### Why Compaction Matters

Without compaction, even with plenty of total free memory, a large object allocation can fail:

```
Total heap: 1 GB
Live objects: 400 MB
Free: 600 MB (but scattered in 10 MB holes)
New request: 50 MB object

Without compaction: FAIL — no single contiguous 50 MB block
With compaction:    SUCCESS — 600 MB contiguous block available
```

### Compaction Cost

- Must update **every reference** to every moved object (pointer fixup pass)
- For G1 Full GC: can take **seconds** for multi-GB heaps
- ZGC/Shenandoah do this **concurrently** using load barriers — no STW compaction

---

## 4.4 Stop-The-World (STW) Pauses

A **stop-the-world pause** is when all application threads are halted for GC work. No requests are served during this time.

### Why STW is Necessary

Without STW, the GC could be marking an object as live while the application thread is simultaneously nulling the last reference to it — causing incorrect behavior. The most conservative approach is to freeze the world.

```
Application threads:  ████████████│         │████████████
GC thread:                        │███████  │
                                  ↑ STW     ↑ Resume
                         All app threads paused here
```

### Safepoints

The JVM doesn't stop threads instantly — threads continue until they reach a **safepoint** (a point in code where GC can safely inspect/modify thread state).

```java
// Common safepoint locations:
  - Method calls (especially in loops)
  - Backward branches (loop back edges)
  - JNI calls

// PROBLEM: A tight loop with no safepoints can delay STW
for (long i = 0; i < Long.MAX_VALUE; i++) {
    // No method calls, no backward branches (counted loop optimization)
    // Thread may never reach a safepoint → other threads wait → "time to safepoint" latency
}
```

```bash
# Diagnose long time-to-safepoint:
-XX:+SafepointTimeout
-XX:SafepointTimeoutDelay=1000   # Log if safepoint takes > 1 second
-Xlog:safepoint*                 # Full safepoint logging (Java 9+)
```

### STW Pause Sources

| Event | Typical Pause | Collector |
|---|---|---|
| Minor GC | 1–50 ms | All |
| G1 Mixed GC | 50–200 ms | G1 |
| G1 Full GC (fallback) | 1–30 s | G1 |
| ZGC (initial mark) | < 1 ms | ZGC |
| ZGC (remark) | < 1 ms | ZGC |
| Parallel GC | 100 ms–2 s | Parallel |

---

## 4.5 Concurrent vs Parallel GC

Two axes of GC work parallelism — often confused:

| Term | Meaning | Application threads? |
|---|---|---|
| **Parallel GC** | Multiple GC threads work simultaneously | Stopped (STW) |
| **Concurrent GC** | GC threads work alongside application threads | Running |
| **Serial GC** | Single GC thread | Stopped (STW) |

```
Serial GC:
App:  ████████████│              │████████
GC:              └─────────────┘
                  1 thread, STW

Parallel GC:
App:  ████████████│       │████████
GC:              └───────┘
                  N threads, STW (shorter pause)

Concurrent GC (e.g., ZGC marking):
App:  ████████████████████████████████████
GC:         ──────────────────────────
             GC runs concurrently (no pause)
```

### Trade-offs

| Approach | STW Pause | CPU Overhead | Throughput |
|---|---|---|---|
| Serial | Long | Low | Low |
| Parallel STW | Medium | Medium | High |
| Mostly Concurrent | Very short | High (GC steals CPU) | Medium-High |

---

## 4.6 Write Barriers

A **write barrier** is a small piece of code injected by the JIT compiler at every reference write, allowing the GC to track cross-generational or concurrent marking changes.

### Card Table Write Barrier (Minor GC)

```java
// Application code:
oldGenObject.field = youngGenObject;

// JIT-generated code (pseudocode):
oldGenObject.field = youngGenObject;
int cardIndex = (address_of(oldGenObject) >> 9);
cardTable[cardIndex] = DIRTY;   // Mark this 512-byte card as modified
```

During Minor GC, only **dirty cards** in Old Gen need to be scanned for references to Young Gen objects — avoiding a full Old Gen scan.

### SATB Write Barrier (Concurrent Marking)

G1 uses **Snapshot-At-The-Beginning (SATB)** to ensure that objects live at the START of concurrent marking are not incorrectly collected:

```java
// Application thread updates a reference during concurrent marking:
obj.field = newRef;

// SATB barrier (pseudocode):
Object oldRef = obj.field;        // Save old reference
if (oldRef != null) {
    satbQueue.enqueue(oldRef);    // Remember what was there before
}
obj.field = newRef;               // Do the actual update
```

The old reference is added to a SATB buffer — re-marked as live during the remark phase, ensuring no live object is ever missed.

### Load Barrier (ZGC/Shenandoah)

ZGC injects a barrier on **every reference READ** (not write) to handle concurrent relocation:

```java
// Application code:
Object ref = obj.field;

// ZGC load barrier (pseudocode):
Object ref = obj.field;
if (ref.colorBits != expected_color) {
    ref = slowPath_fixup(ref);    // Remap if object has been relocated
    obj.field = ref;              // Self-heal the reference
}
return ref;
```

This allows ZGC to **relocate objects while the application runs** — the next read fixes up stale references automatically.

---

## Summary

| Phase | What Happens | STW? |
|---|---|---|
| **Marking** | Traverse object graph from roots; mark live objects | Sometimes (partial or full) |
| **Sweeping** | Reclaim dead objects (creates free list holes) | STW or concurrent |
| **Copying** | Copy live objects to empty region (no fragmentation) | STW |
| **Compaction** | Move all live objects to one end; reclaim contiguous free space | STW or concurrent (ZGC) |

---

**Navigation**

[← Generational GC](03-generational-gc.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Garbage Collectors](05-garbage-collectors.md)

---

*Last updated: 2026 | Java 21 LTS*
