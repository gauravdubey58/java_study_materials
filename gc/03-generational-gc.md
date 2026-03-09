# 03 — Generational Garbage Collection

> Covers: Weak Generational Hypothesis, Minor GC flow, Major GC, Full GC, Metaspace GC, object promotion, and tenuring.

---

**Navigation**

[← JVM Memory Structure](02-jvm-memory-structure.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → GC Process](04-gc-process.md)

---

## Table of Contents

- [3.1 Weak Generational Hypothesis](#31-weak-generational-hypothesis)
- [3.2 Minor GC (Young Generation)](#32-minor-gc-young-generation)
- [3.3 Major GC / Full GC](#33-major-gc--full-gc)
- [3.4 Metaspace GC](#34-metaspace-gc)
- [3.5 Object Promotion & Tenuring](#35-object-promotion--tenuring)
- [Summary](#summary)

---

## 3.1 Weak Generational Hypothesis

Generational GC is built on a well-observed empirical fact about object lifetimes:

> **"Most objects die young."**

Studies of real Java applications show that 80–98% of objects become unreachable within a very short time of allocation (often within the same method call).

```
Object Lifetime Distribution (typical Java app):
 
Survival  100% ┤█
Rate           ├█
            80%├█
               ├█ █
            60%├█ █
               ├█ █
            40%├█ █ █
               ├█ █ █ █
            20%├█ █ █ █ ██
             0%└──────────────────────────►
               1  2  3  4  5  ... GC cycles
 
Most objects (dark bars) die before or at their first GC.
```

### Why This Matters for GC Design

- Collecting **only the Young Generation** (Minor GC) cleans up ~80–98% of garbage very quickly
- Minor GC operates on a small memory region → fast and frequent
- Old Generation grows slowly → collected rarely but more expensively

---

## 3.2 Minor GC (Young Generation)

### Trigger Conditions

- Eden space is **full** — new object cannot be allocated
- Explicit `System.gc()` call (usually triggers Full GC)
- Some GC implementations may trigger proactively based on allocation rate

### Step-by-Step Minor GC Flow

```
Step 1: Eden fills up
┌──────────────┐  ┌────────────┐  ┌────────────┐
│  Eden (FULL) │  │ S0 (active)│  │ S1 (empty) │
│ obj1 obj2    │  │ obj5 (age2)│  │            │
│ obj3 obj4    │  │ obj6 (age1)│  │            │
└──────────────┘  └────────────┘  └────────────┘

Step 2: Mark live objects in Eden + S0
- Trace from GC roots
- Mark obj1, obj3, obj5 as live (obj2, obj4, obj6 are dead)

Step 3: Copy live objects to S1, increment ages
┌──────────────┐  ┌────────────┐  ┌────────────┐
│  Eden        │  │ S0         │  │ S1 (active)│
│  (cleared)   │  │ (cleared)  │  │ obj1 (age1)│
│              │  │            │  │ obj3 (age1)│
└──────────────┘  └────────────┘  │ obj5 (age3)│
                                   └────────────┘

Step 4: Objects with age ≥ threshold promoted to Old Gen
- obj5 (age 3) → promoted if MaxTenuringThreshold = 3
```

### Characteristics of Minor GC

| Property | Value |
|---|---|
| Algorithm | Stop-The-World (STW) |
| Threads | Parallel (multiple GC threads) for most collectors |
| Duration | Very short: 1–50 ms typically |
| Frequency | Very frequent: every few seconds under load |
| Memory freed | ~80–98% of Young Gen per collection |
| Fragmentation | None — uses copy collection |

### Card Table — Tracking Old→Young References

A problem: during Minor GC, how do we know which Young Gen objects are referenced by Old Gen objects (without scanning all of Old Gen)?

```
Solution: Card Table

Old Generation memory is divided into 512-byte "cards".
Whenever an Old Gen object's reference field is updated to point to
a Young Gen object, that card is marked "dirty".

During Minor GC: scan only dirty cards (a tiny fraction of Old Gen)
→ Fast identification of cross-generational references
```

```bash
# Enable card table verification (debugging only)
-XX:+VerifyBeforeGC
```

---

## 3.3 Major GC / Full GC

### Major GC

Collects the **Old Generation** (and usually Young Gen too). Triggered when:

- Old Generation is full or near-full
- Promotion failure — no room in Old Gen for objects being promoted from Young Gen
- Concurrent GC cannot keep up with the allocation rate (concurrent mode failure in CMS/G1)

### Full GC

Collects **all generations**: Young + Old + Metaspace. The most expensive GC event.

**Full GC triggers:**
```
1. Explicit System.gc() call
2. Metaspace exhausted
3. Direct buffer allocation failure
4. JVM determines heap is critically full
5. Concurrent GC fallback (when G1/CMS can't keep up)
6. Heap dump requested (jmap -dump)
7. RMI distributed GC
```

### Full GC Phases (G1 Fallback — Single Threaded)

```
Full GC Timeline:
│────────────── STW PAUSE ─────────────────│
│ Mark Young │ Mark Old │ Compact │ Verify │
│────────────┼──────────┼─────────┼────────│
  Fast         Slow       Slowest   Fast
```

> ⚠️ **Full GC pauses can last seconds to minutes** for large heaps. In production, frequent Full GCs are a red flag indicating memory pressure, allocation rate issues, or GC misconfiguration.

### Identifying Full GC in Logs

```
# Java 9+ GC log format:
[2024-01-15T10:30:45.123+0000][gc] GC(42) Pause Full (G1 Compaction Pause)
[2024-01-15T10:30:45.123+0000][gc] GC(42)   Eden regions: 200->0(200)
[2024-01-15T10:30:45.123+0000][gc] GC(42)   Survivor regions: 20->0(20)
[2024-01-15T10:30:45.123+0000][gc] GC(42)   Old regions: 800->600(800)
[2024-01-15T10:30:48.456+0000][gc] GC(42) Pause Full (G1 Compaction Pause) 7168M->5120M(8192M) 3333ms
#                                                                                            ^^^^^^ 3.3 second pause!
```

---

## 3.4 Metaspace GC

Metaspace is collected as part of Full GC when class metadata can be unloaded.

### When Can a Class Be Unloaded?

A class (and its metadata in Metaspace) can be unloaded **only when all three conditions are true**:

```
1. No instances of the class exist on the heap
        AND
2. No Class object for this class is reachable (no static refs to Class<?>)
        AND
3. The ClassLoader that loaded the class is eligible for GC
         (all references to the ClassLoader are gone)
```

### Metaspace GC in Practice

```java
// Standard classes (loaded by bootstrap/system ClassLoader) → NEVER unloaded
// JVM lives longer than any single ClassLoader chain

// Dynamic class loading (e.g., Groovy, CGLIB, hot-reload frameworks)
// → Custom ClassLoaders → CAN be unloaded when loader is GC'd
URLClassLoader loader = new URLClassLoader(urls);
Class<?> dynamicClass = loader.loadClass("com.example.DynamicClass");
// ...
loader.close();
loader = null;        // ClassLoader eligible for GC
dynamicClass = null;  // Class object eligible for GC
// → Metaspace entry for DynamicClass freed at next Full GC
```

### Common Metaspace Leak

```java
// PROBLEM: Generating classes at runtime without releasing ClassLoaders
// Common in: reflection proxies, CGLIB (Spring AOP), Groovy scripts,
//            JRuby, Clojure hot reload

// Each call creates a new ClassLoader + class → Metaspace grows forever
for (int i = 0; i < 10000; i++) {
    ProxyFactory factory = new ProxyFactory(target);
    factory.addAdvice(advice);
    factory.getProxy(); // Creates new proxy class each time! (if not cached)
}
// → OutOfMemoryError: Metaspace
```

**Fix:** Cache generated proxies/classes, use a shared ClassLoader, or limit `-XX:MaxMetaspaceSize`.

---

## 3.5 Object Promotion & Tenuring

### Age Counter

Every object in the Young Generation has an **age counter** stored in its object header (4 bits → max age 15).

```
Object Header (64-bit JVM, compressed oops):
┌────────────────────────────────────────────────────────┐
│ Mark Word (8 bytes)                                    │
│   [hashCode:25][age:4][biasedLock:1][lockState:2]      │
│                  ↑ age counter (0–15)                  │
├────────────────────────────────────────────────────────┤
│ Class Pointer (4 bytes, compressed)                    │
└────────────────────────────────────────────────────────┘
```

### Promotion Rules

```
Each Minor GC:
  if (object.age < MaxTenuringThreshold AND fits in Survivor):
    → Copy to Survivor, age++

  if (object.age >= MaxTenuringThreshold):
    → Promote to Old Generation (age counter no longer needed)

  if (Survivor space is > 50% full after copying):
    → JVM dynamically lowers tenuring threshold (may promote objects earlier)

  if (Survivor space is completely full):
    → All remaining live objects go directly to Old Gen (premature promotion)
```

### Tenuring Configuration

```bash
-XX:MaxTenuringThreshold=15    # Max age (default 15; 1–15)
                               # Lower value → objects promoted faster → less time in Young Gen
                               # Higher value → objects stay in Young Gen longer

-XX:+PrintTenuringDistribution # Log age distribution of objects in Survivor spaces
```

### Reading Tenuring Distribution Output

```
Desired survivor size 134217728 bytes, new threshold 7 (max 15)
- age   1:   45600000 bytes,   45600000 total    ← 45 MB survived 1 GC
- age   2:   12800000 bytes,   58400000 total    ← 12 MB survived 2 GCs
- age   3:    8200000 bytes,   66600000 total
- age   4:    5100000 bytes,   71700000 total
- age   5:    3800000 bytes,   75500000 total
- age   6:    3200000 bytes,   78700000 total
- age   7:    2900000 bytes,   81600000 total
                                      ↑ 81 MB > 50% of 134 MB → JVM lowered threshold to 7
```

### Premature Promotion Problem

**Symptom:** Old Gen fills up quickly → frequent Full GCs even though objects are actually short-lived.

**Root cause:** Survivor spaces are too small → objects forced to Old Gen before they die.

```
Diagnosis:
  jstat -gcutil <pid> 1000
  → Watch O (Old Gen %) growing steadily
  → Minor GC frequency high but Old Gen still growing

Fix:
  -XX:SurvivorRatio=6    # Larger Survivor spaces (default 8 → Eden:S = 8:1:1)
  -Xmn1g                 # Increase total Young Gen size
  -XX:MaxTenuringThreshold=10  # Keep objects in Young Gen longer
```

---

## Summary

| GC Type | Region | Trigger | Pause | Frequency |
|---|---|---|---|---|
| **Minor GC** | Young Generation | Eden full | Short (1–50ms) | Very frequent |
| **Major GC** | Old Generation | Old Gen full | Medium-Long (100ms–2s) | Occasional |
| **Full GC** | All generations + Metaspace | Memory pressure, explicit call | Long (seconds) | Rare (should be) |
| **Metaspace GC** | Metaspace | Usage > threshold | Part of Full GC | Rare |

---

**Navigation**

[← JVM Memory Structure](02-jvm-memory-structure.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → GC Process](04-gc-process.md)

---

*Last updated: 2026 | Java 21 LTS*
