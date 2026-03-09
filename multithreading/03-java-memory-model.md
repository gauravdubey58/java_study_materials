# 03 — Java Memory Model & Volatile

> Covers: The Java Memory Model (JMM), visibility problems, happens-before guarantees, the volatile keyword, and double-checked locking.

---

**Navigation**

[← Synchronization & Locking](02-synchronization-locking.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Atomic Classes & JUC Locks](04-atomic-concurrent-locks.md)

---

## Table of Contents

- [5. Java Memory Model (JMM)](#5-java-memory-model-jmm)
  - [5.1 The Problem — Visibility & Ordering](#51-the-problem--visibility--ordering)
  - [5.2 Happens-Before Relationship](#52-happens-before-relationship)
- [6. Volatile Keyword](#6-volatile-keyword)
  - [6.1 What Volatile Guarantees](#61-what-volatile-guarantees)
  - [6.2 Volatile vs. Synchronized](#62-volatile-vs-synchronized)
  - [6.3 Double-Checked Locking (Singleton Pattern)](#63-double-checked-locking-singleton-pattern)
- [Summary](#summary)

---

## 5. Java Memory Model (JMM)

### 5.1 The Problem — Visibility & Ordering

Modern CPUs and JIT compilers optimize aggressively. Without explicit synchronization:

- CPUs **cache** values in L1/L2/L3 registers — writes may not be flushed to main memory.
- The JIT compiler may **reorder** instructions (safe in single-thread, unsafe in multi-thread).
- A write in Thread A may **never be seen** by Thread B.

```java
// WITHOUT synchronization — broken!
boolean ready = false;
int value = 0;

// Thread A
value = 42;
ready = true;   // JIT may reorder: this executes BEFORE value = 42

// Thread B
while (!ready) {}            // May loop forever (reads cached ready = false)
System.out.println(value);   // May print 0 (reads cached stale value)
```

**Why this happens:**

```
CPU Architecture:
  Thread A (Core 1)                Thread B (Core 2)
  ┌──────────────────┐             ┌──────────────────┐
  │ L1 Cache: ready=T│             │ L1 Cache: ready=F │ ← stale!
  │ L1 Cache: value=42│            │ L1 Cache: value=0 │ ← stale!
  └────────┬─────────┘             └────────┬─────────┘
           │ (eventual flush)               │ (never invalidated)
  ┌────────▼─────────────────────────────────▼────────┐
  │                  Main Memory                       │
  │             ready = true, value = 42               │
  └────────────────────────────────────────────────────┘
```

---

### 5.2 Happens-Before Relationship

The JMM defines **happens-before (HB)** — a guarantee that if action A happens-before action B, then the result of A is **visible** to B.

| Rule | Description |
|---|---|
| **Program Order** | Every action in a thread HB every subsequent action in the same thread. |
| **Monitor Unlock** | Unlocking a monitor HB every subsequent lock of that same monitor. |
| **Volatile Write** | A write to a `volatile` field HB every subsequent read of that field. |
| **Thread Start** | `Thread.start()` HB any action in the started thread. |
| **Thread Join** | All actions in a thread HB `Thread.join()` returning. |
| **Transitivity** | If A HB B and B HB C, then A HB C. |
| **Object finalization** | End of constructor HB start of `finalize()`. |

```java
// Thread.start() happens-before guarantee
int sharedValue = 0;

sharedValue = 42;                 // (A) write before start

Thread t = new Thread(() -> {
    System.out.println(sharedValue); // (B) guaranteed to see 42
});
t.start();   // start() establishes HB from (A) to all actions in t

// Thread.join() happens-before guarantee
t.join();    // join() establishes HB from all actions in t to here
// Everything t wrote is now visible to this thread
```

---

## 6. Volatile Keyword

### 6.1 What Volatile Guarantees

`volatile` provides **two** key guarantees:

1. **Visibility** — Writes to a `volatile` field are immediately flushed to main memory. Reads always fetch from main memory (bypass CPU cache).
2. **Ordering** — Instructions cannot be reordered across a volatile access (acts as a memory fence).

```java
public class StatusMonitor implements Runnable {
    // Without volatile — worker thread may never see shutdown = true
    private volatile boolean shutdown = false;

    @Override
    public void run() {
        while (!shutdown) {      // Always reads fresh value from main memory
            doWork();
        }
        System.out.println("Shutting down cleanly");
    }

    // Called from a different thread (e.g., shutdown hook)
    public void stop() {
        shutdown = true;         // Immediately flushed to main memory
    }
}
```

**What volatile does NOT guarantee:**

```java
volatile int counter = 0;

// Still a race condition! counter++ is: READ → INCREMENT → WRITE (3 ops)
// Two threads can interleave between the read and write
counter++;  // NOT atomic!

// Fix: use AtomicInteger or synchronized
AtomicInteger safeCounter = new AtomicInteger(0);
safeCounter.incrementAndGet(); // Atomic — one indivisible operation
```

---

### 6.2 Volatile vs. Synchronized

| Aspect | `volatile` | `synchronized` |
|---|---|---|
| Visibility | ✅ Yes | ✅ Yes |
| Atomicity for single read/write | ✅ Yes | ✅ Yes |
| Atomicity for compound ops (`++`) | ❌ No | ✅ Yes |
| Mutual exclusion (one thread at a time) | ❌ No | ✅ Yes |
| Can block threads | ❌ No | ✅ Yes |
| Performance overhead | Low (memory fence) | Higher (lock acquire/release) |

**When `volatile` is sufficient:**

```java
// ✅ Single writer, multiple readers — flag pattern
private volatile boolean initialized = false;

// ✅ Independent writes — no read-modify-write
private volatile long lastAccessTime;
lastAccessTime = System.currentTimeMillis(); // Single write, no dependency on old value

// ❌ NOT sufficient for compound operations
private volatile int hits = 0;
hits++;  // Broken: read-modify-write is not atomic
```

---

### 6.3 Double-Checked Locking (Singleton Pattern)

Without `volatile`, double-checked locking is **broken** due to instruction reordering.

**The broken version (without volatile):**

```java
// BROKEN — do not use!
private static ConfigManager instance;

public static ConfigManager getInstance() {
    if (instance == null) {
        synchronized (ConfigManager.class) {
            if (instance == null) {
                instance = new ConfigManager(); // Problem: partially visible!
            }
        }
    }
    return instance;
}
```

**Why it's broken:**

```
instance = new ConfigManager() compiles to roughly:
  1. Allocate memory for ConfigManager
  2. Write reference to 'instance' field   ← JIT may reorder steps 2 and 3!
  3. Invoke ConfigManager constructor

Thread B reads instance != null after step 2
but BEFORE step 3 (constructor not yet run)
→ Thread B uses a partially-constructed object!
```

**The correct version (with volatile):**

```java
public class ConfigManager {
    // volatile prevents partial construction being visible
    private static volatile ConfigManager instance;

    private ConfigManager() {
        // expensive initialization
    }

    public static ConfigManager getInstance() {
        if (instance == null) {                    // First check (no lock — fast path)
            synchronized (ConfigManager.class) {
                if (instance == null) {            // Second check (with lock — safe)
                    instance = new ConfigManager();
                }
            }
        }
        return instance;
    }
}
```

**Even better — Initialization-on-Demand Holder (no volatile needed):**

```java
public class ConfigManager {
    private ConfigManager() {}

    // Inner class is loaded only when getInstance() is first called
    // JVM class loading is inherently thread-safe
    private static class Holder {
        static final ConfigManager INSTANCE = new ConfigManager();
    }

    public static ConfigManager getInstance() {
        return Holder.INSTANCE; // No synchronization needed
    }
}
```

---

## Summary

| Concept | Key Takeaway |
|---|---|
| JMM visibility problem | CPU caches and JIT reordering can make writes invisible across threads |
| Happens-Before | Formal guarantee that write results are visible — established by sync/volatile/start/join |
| `volatile` guarantees | Visibility + ordering (NOT atomicity for compound ops) |
| `volatile` vs `synchronized` | volatile = visibility only; synchronized = visibility + atomicity + exclusion |
| Double-checked locking | Requires `volatile` on the instance field, or use Holder pattern |

---

**Navigation**

[← Synchronization & Locking](02-synchronization-locking.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Atomic Classes & JUC Locks](04-atomic-concurrent-locks.md)

---

*Last updated: 2026 | Java 21 LTS*
