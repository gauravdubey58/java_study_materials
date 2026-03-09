# 09 — Thread-Related Exceptions & Issues

> Covers: Deadlock, livelock, starvation, race conditions, memory consistency errors, and a complete table of thread-related exceptions with causes and fixes.

---

**Navigation**

[← Concurrent Collections](08-concurrent-collections.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Real-World Scenarios](10-real-world-scenarios.md)

---

## Table of Contents

- [13.1 Deadlock](#131-deadlock)
- [13.2 Livelock](#132-livelock)
- [13.3 Starvation](#133-starvation)
- [13.4 Race Condition](#134-race-condition)
- [13.5 Memory Consistency Errors](#135-memory-consistency-errors)
- [13.6 Common Exceptions](#136-common-exceptions)
- [Summary — Detection & Fix Cheat Sheet](#summary--detection--fix-cheat-sheet)

---

## 13.1 Deadlock

**Definition:** Two or more threads wait for each other's locks indefinitely — none can proceed.

### Classic Deadlock

```java
Object lock1 = new Object();
Object lock2 = new Object();

Thread t1 = new Thread(() -> {
    synchronized (lock1) {
        System.out.println("T1 holds lock1, waiting for lock2");
        try { Thread.sleep(50); } catch (InterruptedException e) {}
        synchronized (lock2) { /* work */ } // ← Waiting for lock2
    }
});

Thread t2 = new Thread(() -> {
    synchronized (lock2) {
        System.out.println("T2 holds lock2, waiting for lock1");
        synchronized (lock1) { /* work */ } // ← Waiting for lock1
    }
});

t1.start();
t2.start();
// Neither thread will ever proceed — deadlock!
```

```
Thread 1: holds lock1 → waiting for lock2
Thread 2: holds lock2 → waiting for lock1
              ↑ circular dependency = deadlock
```

### Deadlock Detection

```bash
# 1. Thread dump via jstack
jstack <pid>
# Look for: "Found one Java-level deadlock:" at the bottom

# 2. Via jcmd
jcmd <pid> Thread.print

# 3. Programmatically
ThreadMXBean bean = ManagementFactory.getThreadMXBean();
long[] deadlocked = bean.findDeadlockedThreads();
if (deadlocked != null) {
    ThreadInfo[] infos = bean.getThreadInfo(deadlocked, true, true);
    for (ThreadInfo info : infos) System.out.println(info);
}
```

### Deadlock Prevention

**Strategy 1: Consistent Lock Ordering**

```java
// Always acquire locks in the same order across all threads
void transfer(Account from, Account to, double amount) {
    Account first  = from.id < to.id ? from : to;   // Lower ID first
    Account second = from.id < to.id ? to   : from;

    synchronized (first) {
        synchronized (second) {
            from.debit(amount);
            to.credit(amount);
        }
    }
}
```

**Strategy 2: tryLock with Timeout and Backoff**

```java
boolean transfer(Account from, Account to, double amount) throws InterruptedException {
    while (true) {
        if (from.lock.tryLock(50, TimeUnit.MILLISECONDS)) {
            try {
                if (to.lock.tryLock(50, TimeUnit.MILLISECONDS)) {
                    try {
                        from.debit(amount);
                        to.credit(amount);
                        return true;
                    } finally {
                        to.lock.unlock();
                    }
                }
            } finally {
                from.lock.unlock();
            }
        }
        Thread.sleep(new Random().nextInt(10)); // Randomized backoff
    }
}
```

**Strategy 3: Single Global Lock (Simple but Coarse)**

```java
// Serialize all transfers through one lock
private static final Object TRANSFER_LOCK = new Object();

void transfer(Account from, Account to, double amount) {
    synchronized (TRANSFER_LOCK) {
        from.debit(amount);
        to.credit(amount);
    }
}
```

---

## 13.2 Livelock

**Definition:** Threads actively change state in response to each other but **make no actual progress** — like two people in a hallway both stepping the same direction to let the other pass.

```java
// Example: Two threads politely yielding to each other forever
class Philosopher {
    private boolean eating = false;
    private Philosopher neighbor;

    public void tryToEat() {
        while (true) {
            // If neighbor is eating, be polite and step back
            while (neighbor.isEating()) {
                eating = false;           // Step back
                Thread.yield();
            }
            eating = true;
            // But neighbor does the same thing → both keep yielding forever
        }
    }
}
```

### Livelock Fix: Randomized Backoff

```java
while (!lock.tryLock()) {
    long backoff = (long)(Math.random() * 100); // Random 0–100ms
    Thread.sleep(backoff);                      // Random delay breaks symmetry
}
```

---

## 13.3 Starvation

**Definition:** A thread is perpetually denied CPU time because other higher-priority or more frequent threads monopolize the lock.

```java
// Without fairness — high-frequency threads can monopolize an unfair lock
ReentrantLock unfairLock = new ReentrantLock(false); // default — barging allowed

// Thread A fires 1000 requests/sec → always gets the lock
// Thread B fires 1 request/sec → may never get the lock!
```

### Starvation Fix: Fair Lock

```java
// Fair lock — threads acquire in FIFO arrival order
ReentrantLock fairLock = new ReentrantLock(true);

// Trade-off: fair lock has lower throughput than unfair lock
// Use when starvation is a real concern (background tasks, lower-priority threads)
```

### Starvation Fix: Separate Thread Pools

```java
// Critical tasks get their own dedicated pool — never starved by others
ExecutorService criticalPool  = Executors.newFixedThreadPool(4);
ExecutorService backgroundPool = Executors.newFixedThreadPool(2);

criticalPool.submit(highPriorityTask);
backgroundPool.submit(lowPriorityTask);
```

---

## 13.4 Race Condition

**Definition:** The program's output depends on the **non-deterministic timing** of threads — results are incorrect under some interleavings.

### Check-Then-Act Race Condition

```java
// BROKEN — check and act are not atomic
if (!map.containsKey(key)) {    // Thread A: key absent
                                // Thread B: key absent too (context switch!)
    map.put(key, createValue()); // Both threads put → one silently overwrites
}

// FIX: atomic operation
map.putIfAbsent(key, value);

// Or for complex initialization:
map.computeIfAbsent(key, k -> createValue(k)); // Atomic in ConcurrentHashMap
```

### Read-Modify-Write Race Condition

```java
// BROKEN — not atomic (3 operations: read, increment, write)
private int counter = 0;
counter++;   // Thread A reads 5, Thread B reads 5, both write 6 → should be 7

// FIX 1: synchronized
synchronized (this) { counter++; }

// FIX 2: AtomicInteger
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();

// FIX 3: LongAdder (for high-throughput counters)
LongAdder adder = new LongAdder();
adder.increment();
```

### Singleton Race Condition (without volatile)

```java
// BROKEN — partial construction visible to other threads (see JMM section)
static Singleton instance;
if (instance == null) instance = new Singleton(); // Race!

// FIX: volatile + double-checked locking, or Holder pattern
```

---

## 13.5 Memory Consistency Errors

**Definition:** Thread B reads a **stale value** written by Thread A because no happens-before relationship was established.

```java
// BROKEN — no synchronization
boolean started = false;
int config = 0;

// Thread A
config = 42;
started = true;      // May be reordered before config = 42!

// Thread B
while (!started) {}  // May loop forever (cached value)
use(config);         // May see 0 (stale) or undefined
```

**Fixes:**

```java
// FIX 1: volatile (for visibility without mutual exclusion)
volatile boolean started = false;
volatile int config = 0;

// FIX 2: synchronized (for visibility + atomicity)
synchronized (lock) {
    config = 42;
    started = true;
}

// FIX 3: AtomicReference for whole state
AtomicReference<Config> configRef = new AtomicReference<>();
configRef.set(new Config(42, true)); // Single atomic publish
```

---

## 13.6 Common Exceptions

| Exception | Cause | Fix |
|---|---|---|
| `ConcurrentModificationException` | Modifying a non-concurrent collection while iterating | Use `CopyOnWriteArrayList`, `ConcurrentHashMap`, or `iterator.remove()` |
| `InterruptedException` | Thread interrupted while blocking (`sleep`, `wait`, `take`) | Restore interrupt flag: `Thread.currentThread().interrupt()` and break/rethrow |
| `RejectedExecutionException` | Thread pool queue is full or executor is shut down | Use `CallerRunsPolicy`; resize pool; check for pool shutdown before submit |
| `TimeoutException` | `Future.get(timeout)` exceeded the limit | Increase timeout; investigate what the task is doing; add alerts |
| `ExecutionException` | The `Callable` threw an exception | Unwrap with `.getCause()`; handle accordingly |
| `IllegalMonitorStateException` | `wait()`/`notify()` called without owning the monitor lock | Always call `wait()`/`notify()` inside `synchronized(obj)` block on the same `obj` |
| `IllegalThreadStateException` | `start()` called on a thread that was already started | Create a new `Thread` instance — threads cannot be restarted |
| `OutOfMemoryError: unable to create native thread` | OS thread limit reached or heap exhausted | Use thread pool; reduce `-Xss` stack size; check for thread leaks |
| `StackOverflowError` | Unbounded recursion in a thread | Add base case; reduce recursion depth; use iterative approach |
| `NullPointerException` from `synchronized(null)` | Synchronizing on a null reference | Always verify the lock object is non-null before acquiring |

### ConcurrentModificationException in Detail

```java
List<String> list = new ArrayList<>(List.of("a", "b", "c"));

// BROKEN — modifying list while iterating
for (String s : list) {
    if (s.equals("b")) list.remove(s); // → ConcurrentModificationException
}

// FIX 1: Use iterator.remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("b")) it.remove(); // Safe
}

// FIX 2: removeIf (Java 8+)
list.removeIf(s -> s.equals("b")); // Safe

// FIX 3: Collect to remove, then remove
List<String> toRemove = list.stream().filter(s -> s.equals("b")).collect(Collectors.toList());
list.removeAll(toRemove);

// FIX 4: CopyOnWriteArrayList for concurrent access
List<String> cowList = new CopyOnWriteArrayList<>(list);
for (String s : cowList) {
    cowList.remove(s); // Safe — iterates snapshot
}
```

---

## Summary — Detection & Fix Cheat Sheet

| Problem | Symptom | Detection | Fix |
|---|---|---|---|
| **Deadlock** | App hangs completely | `jstack` — "Found one Java-level deadlock" | Consistent lock ordering; `tryLock` with timeout |
| **Livelock** | 100% CPU, no progress | Multiple thread dumps — threads change state but loop | Randomized backoff |
| **Starvation** | Some requests never complete | Thread dump — one thread never RUNNABLE | Fair locks; separate thread pools |
| **Race condition** | Inconsistent results, occasional data corruption | Code review + stress testing | `synchronized`, `AtomicX`, or lock-free atomic ops |
| **Memory consistency** | Stale reads, mysterious zero values | JMM analysis | `volatile`, `synchronized`, happens-before establishment |
| **Thread leak** | Growing thread count, eventual OOM | JMX `ThreadCount` metric; periodic thread dump | Use bounded thread pools; ensure threads terminate |
| **False sharing** | Unexpected performance degradation on multi-core | CPU performance profiling | `@Contended`; padding; separate cache lines |

---

**Navigation**

[← Concurrent Collections](08-concurrent-collections.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Real-World Scenarios](10-real-world-scenarios.md)

---

*Last updated: 2026 | Java 21 LTS*
