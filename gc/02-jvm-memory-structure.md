# 02 — JVM Memory Structure

> Covers: Full JVM memory layout, heap regions (Eden, Survivor, Old Gen), Metaspace, non-heap areas, stack memory, and TLAB.

---

**Navigation**

[← What is GC?](01-what-is-gc.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Generational GC](03-generational-gc.md)

---

## Table of Contents

- [2.1 Heap Overview](#21-heap-overview)
- [2.2 Young Generation](#22-young-generation)
- [2.3 Old Generation (Tenured Space)](#23-old-generation-tenured-space)
- [2.4 Metaspace](#24-metaspace)
- [2.5 Non-Heap Memory](#25-non-heap-memory)
- [2.6 TLAB — Thread-Local Allocation Buffer](#26-tlab--thread-local-allocation-buffer)
- [Summary](#summary)

---

## 2.1 Heap Overview

The JVM heap is where all Java objects live. It is divided into regions optimized for the **Weak Generational Hypothesis** — most objects die young.

```
┌───────────────────────────────────────────────────────────────┐
│                         JVM PROCESS                           │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                       HEAP                           │    │
│  │                                                      │    │
│  │  ┌────────────────────────┐  ┌──────────────────┐   │    │
│  │  │    Young Generation    │  │  Old Generation  │   │    │
│  │  │                        │  │  (Tenured Space) │   │    │
│  │  │  ┌───────┐ ┌───┐ ┌───┐│  │                  │   │    │
│  │  │  │ Eden  │ │ S0│ │ S1││  │  Long-lived objs │   │    │
│  │  │  │(~80%) │ │8% │ │8% ││  │                  │   │    │
│  │  │  └───────┘ └───┘ └───┘│  │                  │   │    │
│  │  └────────────────────────┘  └──────────────────┘   │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              Metaspace (off-heap)                    │    │
│  │       Class metadata, method bytecodes, CP           │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
│  ┌──────────────┐  ┌─────────────┐  ┌──────────────────┐    │
│  │ Thread Stack │  │  Code Cache │  │  Direct Buffers  │    │
│  │ (per thread) │  │  (JIT code) │  │  (ByteBuffer)    │    │
│  └──────────────┘  └─────────────┘  └──────────────────┘    │
└───────────────────────────────────────────────────────────────┘
```

### Default Size Proportions

| Region | Default proportion |
|---|---|
| Young Generation | ~33% of heap (`-XX:NewRatio=2` → Old:Young = 2:1) |
| Eden (within Young) | ~80% of Young Gen (`-XX:SurvivorRatio=8`) |
| Each Survivor (S0/S1) | ~10% of Young Gen each |
| Old Generation | ~67% of heap |

---

## 2.2 Young Generation

The Young Generation is where **all new objects** are allocated. It is designed to be collected frequently and quickly.

### Eden Space

- **All** new object allocations go here (via TLAB — see §2.6)
- When Eden fills up → Minor GC is triggered
- After Minor GC → Eden is completely empty

### Survivor Spaces (S0 and S1)

- Two equal-sized spaces — **only one is active at any time**
- The inactive one is always empty (used as the copy destination during the next Minor GC)
- Objects surviving Minor GC are copied here with age incremented
- Objects reaching max age (`MaxTenuringThreshold`, default 15) are promoted to Old Gen

```
Before Minor GC:            After Minor GC:
┌────────┐ ┌────┐ ┌────┐   ┌────────┐ ┌────┐ ┌────┐
│ Eden   │ │ S0 │ │ S1 │   │ Eden   │ │ S0 │ │ S1 │
│(full)  │ │(8) │ │    │   │(empty) │ │    │ │(10)│
└────────┘ └────┘ └────┘   └────────┘ └────┘ └────┘
                     ↑                          ↑
              was active                  now active
              (numbers = object ages)
```

### Why Two Survivor Spaces?

The **copy collection** algorithm requires a destination space that is empty. After copying live objects from Eden + active Survivor to the other Survivor space, the source spaces are cleared. The "ping-pong" between S0 and S1 achieves this without fragmentation.

---

## 2.3 Old Generation (Tenured Space)

The Old Generation holds **long-lived objects** that have been promoted from the Young Generation.

### Characteristics

- Much **larger** than Young Gen (default ~2× Young Gen size)
- Collected by **Major GC** or **Full GC** — less frequently but more expensively
- Uses **mark-sweep-compact** (not copy) — live objects are moved to one end to eliminate gaps
- **Large objects** may be allocated directly here (bypassing Eden) to avoid expensive copying

### What lives in Old Gen?

```java
// Objects that survive many Minor GCs
static Map<String, Config> CONFIG_CACHE = new HashMap<>();  // Long-lived → Old Gen

// Very large objects (varies by collector)
byte[] largeArray = new byte[10 * 1024 * 1024];  // May go directly to Old Gen

// Class instances held for the life of the application
private final ConnectionPool pool = new ConnectionPool(10); // → Old Gen
```

### Promotion Triggers

| Trigger | Description |
|---|---|
| Age threshold reached | Object age ≥ `MaxTenuringThreshold` (default 15) |
| Dynamic tenuring | JVM lowers threshold if Survivor space > 50% full |
| Survivor space overflow | Live objects don't fit in Survivor → forced promotion |
| Large object allocation | Objects larger than a threshold skip Young Gen entirely |

---

## 2.4 Metaspace

Metaspace (introduced in Java 8, replacing PermGen) stores **class-level metadata**.

### What Metaspace Stores

| Content | Examples |
|---|---|
| Class metadata | Class name, superclass, interfaces, access flags |
| Method bytecodes | Compiled `.class` file bytecode |
| Constant pool | String literals, class/method/field references |
| Static variables | `static` field values (since Java 8) |
| JIT-compiled code | NOT here — in Code Cache |

### Key Difference from PermGen (Java 7 and earlier)

| Feature | PermGen | Metaspace |
|---|---|---|
| Location | On heap (JVM heap) | Off heap (native OS memory) |
| Default size | Fixed (often 64–256 MB) | **Unlimited** (grows as needed) |
| `OutOfMemoryError` | `java.lang.OutOfMemoryError: PermGen space` | `java.lang.OutOfMemoryError: Metaspace` |
| GC trigger | At Full GC | When usage exceeds `MetaspaceSize` threshold |
| Configuring cap | `-XX:MaxPermSize` | `-XX:MaxMetaspaceSize` |

### Metaspace Flags

```bash
-XX:MetaspaceSize=128m        # Initial commit size (also triggers first GC)
-XX:MaxMetaspaceSize=512m     # Hard cap — prevents unbounded native memory growth
-XX:MinMetaspaceFreeRatio=40  # Minimum % free to avoid expanding
-XX:MaxMetaspaceFreeRatio=70  # Maximum % free (shrink back if exceeded)
```

> ⚠️ Always set `-XX:MaxMetaspaceSize` in production. Without it, a class-loading leak (e.g., in OSGi, reflection-heavy code, or Groovy scripts) can exhaust native memory and crash the OS.

### Metaspace GC Trigger

```
Metaspace usage > MetaspaceSize threshold
       ↓
Full GC triggered (includes Metaspace cleanup)
       ↓
Unloaded classes and their metadata are freed
       ↓
MetaspaceSize threshold re-evaluated
```

Classes are unloaded only when their **ClassLoader** is GC'd (i.e., no more strong references to the ClassLoader).

---

## 2.5 Non-Heap Memory

Memory areas outside the GC-managed heap:

### Thread Stack Memory

- Each thread has its own **stack** (~512 KB – 1 MB by default)
- Stores: method call frames, local variables (primitives and object references), operand stack
- Stack overflows → `StackOverflowError`
- Size controlled by `-Xss256k` (reducing this allows more threads)

```
Thread Stack:
  Frame: main()
    └─ Frame: processOrder()
         └─ Frame: validateInventory()   ← current frame
              local: quantity = 5
              local: productRef → [heap object]
```

### Code Cache (JIT Compiled Code)

- Stores native machine code generated by the JIT compiler (C1 + C2)
- Default size: ~240 MB (Java 9+)
- When full: JIT compilation disabled → performance degrades
- Controlled by `-XX:ReservedCodeCacheSize=256m`
- Monitor: `jcmd <pid> Compiler.codecache`

### Direct Buffers (Off-Heap)

```java
// Allocated outside the GC heap — not collected by GC directly
ByteBuffer direct = ByteBuffer.allocateDirect(100 * 1024 * 1024); // 100 MB off-heap

// Freed when the ByteBuffer object is GC'd AND Cleaner runs
// Or: explicitly using sun.misc.Unsafe (internal)
```

- Used by NIO channels for zero-copy I/O
- Not subject to GC pressure but can cause `OutOfMemoryError: Direct buffer memory`
- Monitor total direct memory: `-XX:MaxDirectMemorySize=512m`

---

## 2.6 TLAB — Thread-Local Allocation Buffer

TLABs are **private Eden sub-regions** given to each thread for object allocation — no synchronization needed.

```
Eden Space:
┌──────────────────────────────────────────────────┐
│ TLAB-Thread1 │ TLAB-Thread2 │ TLAB-Thread3 │ ... │
│  [obj1][obj2]│  [obj3][obj4]│  [obj5]      │     │
└──────────────────────────────────────────────────┘
```

### How TLAB Works

```
Thread wants to allocate a new object:
  1. Check TLAB has space? → Bump pointer forward → Done (no lock!)
  2. TLAB full? → Request new TLAB from Eden → Bump pointer → Done
  3. Eden full? → Trigger Minor GC
```

### Benefits

- **Lock-free allocation** — no synchronization between threads for typical object creation
- **Cache-friendly** — objects allocated by the same thread are in the same TLAB → same cache line
- **High throughput** — allocation is essentially a pointer increment (O(1), ~1 ns)

### TLAB Flags

```bash
-XX:+UseTLAB              # Enable TLABs (on by default)
-XX:TLABSize=512k         # Fixed TLAB size (default: dynamic, ~1% of Eden)
-XX:+PrintTLAB            # Log TLAB stats (useful for diagnosis)
-XX:TLABWasteTargetPercent=1  # Max % of TLAB to waste before requesting new one
```

---

## Summary

| Memory Area | Managed By GC? | Holds | OOM Message |
|---|---|---|---|
| Eden | ✅ Yes (Minor GC) | New object allocations | `Java heap space` |
| Survivor (S0/S1) | ✅ Yes (Minor GC) | Short-lived survivors | `Java heap space` |
| Old Generation | ✅ Yes (Major/Full GC) | Long-lived objects | `Java heap space` |
| Metaspace | ✅ Yes (Full GC) | Class metadata | `Metaspace` |
| Thread Stack | ❌ No | Call frames, local vars | `StackOverflowError` |
| Code Cache | ❌ No | JIT-compiled machine code | Disabled JIT (no OOM) |
| Direct Buffers | Partial (via Cleaner) | Off-heap NIO buffers | `Direct buffer memory` |
| TLAB | ✅ Yes (part of Eden) | Per-thread allocation chunk | `Java heap space` |

---

**Navigation**

[← What is GC?](01-what-is-gc.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Generational GC](03-generational-gc.md)

---

*Last updated: 2026 | Java 21 LTS*
