# 04 — Atomic Classes & JUC Locks

> Covers: Atomic classes, CAS operations, LongAdder, ReentrantLock, ReadWriteLock, StampedLock, and lock comparison.

---

**Navigation**

[← Java Memory Model](03-java-memory-model.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Wait, Notify & Condition](05-wait-notify-condition.md)

---

## Table of Contents

- [7. Atomic Classes](#7-atomic-classes)
  - [7.1 java.util.concurrent.atomic Package](#71-javautilconcurrentatomic-package)
  - [7.2 Real-World: Thread-Safe Request Counter](#72-real-world-thread-safe-request-counter)
  - [7.3 AtomicReference — Lock-Free Stack](#73-atomicreference--lock-free-stack)
  - [7.4 LongAdder vs. AtomicLong](#74-longadder-vs-atomiclong)
- [8. java.util.concurrent Locks](#8-javautilconcurrent-locks)
  - [8.1 ReentrantLock](#81-reentrantlock)
  - [8.2 ReadWriteLock](#82-readwritelock)
  - [8.3 StampedLock (Java 8+)](#83-stampedlock-java-8)
  - [8.4 Lock Comparison](#84-lock-comparison)
- [Summary](#summary)

---

## 7. Atomic Classes

### 7.1 java.util.concurrent.atomic Package

Atomic classes use **CAS (Compare-And-Swap)** CPU instructions to perform lock-free, thread-safe operations. CAS is a single atomic CPU instruction:

```
CAS(address, expected, newValue):
  if (*address == expected):
    *address = newValue
    return true
  else:
    return false   ← retry loop (spin)
```

**Available atomic classes:**

```java
import java.util.concurrent.atomic.*;

// Integer and Long
AtomicInteger  counter = new AtomicInteger(0);
AtomicLong     longCounter = new AtomicLong(0L);

// Boolean
AtomicBoolean  flag = new AtomicBoolean(false);

// Object reference
AtomicReference<String>  ref = new AtomicReference<>("initial");

// Array elements
AtomicIntegerArray  arr = new AtomicIntegerArray(10);

// Reference + version stamp (prevents ABA problem)
AtomicStampedReference<String> stamped = new AtomicStampedReference<>("v1", 0);
```

**Common operations:**

```java
AtomicInteger counter = new AtomicInteger(0);

counter.get();                        // Read current value
counter.set(10);                      // Write (not atomic relative to others, but visible)
counter.getAndSet(20);                // Atomically set and return old value

counter.incrementAndGet();            // Atomic ++counter
counter.decrementAndGet();            // Atomic --counter
counter.addAndGet(5);                 // Atomic counter += 5
counter.getAndIncrement();            // Atomic counter++ (returns old value)

// CAS — the building block
counter.compareAndSet(10, 20);        // If current==10, set to 20 → returns boolean

// Lambda-based (Java 8+)
counter.updateAndGet(x -> x * 2);    // Atomic lambda update
counter.accumulateAndGet(5, Integer::sum); // Atomic accumulation
```

---

### 7.2 Real-World: Thread-Safe Request Counter

```java
public class MetricsCollector {
    private final AtomicLong requestCount  = new AtomicLong(0);
    private final AtomicLong errorCount    = new AtomicLong(0);
    private final AtomicLong totalLatencyMs = new AtomicLong(0);

    // Called from multiple threads on every request
    public void recordRequest(long latencyMs, boolean isError) {
        requestCount.incrementAndGet();
        totalLatencyMs.addAndGet(latencyMs);
        if (isError) {
            errorCount.incrementAndGet();
        }
    }

    public double getAverageLatency() {
        long count = requestCount.get();
        return count == 0 ? 0 : (double) totalLatencyMs.get() / count;
    }

    public double getErrorRate() {
        long count = requestCount.get();
        return count == 0 ? 0 : (double) errorCount.get() / count;
    }
}
```

---

### 7.3 AtomicReference — Lock-Free Stack

```java
public class LockFreeStack<T> {
    private final AtomicReference<Node<T>> top = new AtomicReference<>();

    public void push(T value) {
        Node<T> newNode = new Node<>(value);
        Node<T> currentTop;
        do {
            currentTop = top.get();
            newNode.next = currentTop;
        } while (!top.compareAndSet(currentTop, newNode)); // Retry if another thread pushed first
    }

    public T pop() {
        Node<T> currentTop;
        Node<T> newTop;
        do {
            currentTop = top.get();
            if (currentTop == null) return null;   // Stack is empty
            newTop = currentTop.next;
        } while (!top.compareAndSet(currentTop, newTop)); // Retry if another thread popped first
        return currentTop.value;
    }

    private static class Node<T> {
        final T value;
        Node<T> next;
        Node(T value) { this.value = value; }
    }
}
```

---

### 7.4 LongAdder vs. AtomicLong

Under **high concurrency**, many threads CAS-retrying on the same `AtomicLong` causes significant contention.

`LongAdder` solves this by maintaining multiple internal cells:

```
AtomicLong:                         LongAdder:
                                    ┌─────────┐
  ┌────────────┐                    │ cell[0] │ ← Thread 1, Thread 2
  │   value    │ ← ALL threads      │ cell[1] │ ← Thread 3, Thread 4
  └────────────┘    contend here    │ cell[2] │ ← Thread 5, Thread 6
                                    │  base   │ ← Thread 7
                                    └─────────┘
                                    sum() = base + cell[0] + cell[1] + cell[2]
```

```java
// AtomicLong — high contention under many threads
AtomicLong atomicCounter = new AtomicLong(0);
atomicCounter.incrementAndGet();

// LongAdder — better throughput, eventual consistency on sum()
LongAdder adder = new LongAdder();
adder.increment();
adder.add(5);
long total = adder.sum();       // Merges all cells — may not reflect concurrent increments
adder.reset();                  // Reset all cells to 0
long totalAndReset = adder.sumThenReset();
```

| | `AtomicLong` | `LongAdder` |
|---|---|---|
| Throughput under high concurrency | Lower (single-cell CAS contention) | Higher (distributed cells) |
| `compareAndSet` (CAS) | ✅ Yes | ❌ No |
| Memory | One cell | Multiple cells |
| Best for | Low contention + CAS semantics | High-frequency counting (metrics, hit counters) |

---

## 8. java.util.concurrent Locks

### 8.1 ReentrantLock

`ReentrantLock` is more flexible than `synchronized`. It must be **manually unlocked**, always in a `finally` block.

```java
import java.util.concurrent.locks.ReentrantLock;

public class TicketBookingService {
    private final ReentrantLock lock = new ReentrantLock(true); // fair=true → FIFO ordering
    private int availableSeats = 100;

    // Standard lock/unlock
    public boolean bookSeat(String customer) {
        lock.lock();
        try {
            if (availableSeats <= 0) return false;
            availableSeats--;
            System.out.println(customer + " booked. Remaining: " + availableSeats);
            return true;
        } finally {
            lock.unlock(); // ALWAYS in finally — even if exception thrown
        }
    }

    // Non-blocking: try to acquire, return immediately if unavailable
    public boolean tryBookSeat(String customer) {
        if (lock.tryLock()) {           // Returns false immediately if locked
            try {
                return bookSeat(customer);
            } finally {
                lock.unlock();
            }
        }
        return false; // Could not acquire lock — skip
    }

    // Timed: try for up to 3 seconds
    public boolean bookWithTimeout(String customer) throws InterruptedException {
        if (lock.tryLock(3, TimeUnit.SECONDS)) {
            try {
                return bookSeat(customer);
            } finally {
                lock.unlock();
            }
        }
        throw new RuntimeException("Booking service timed out after 3s");
    }

    // Interruptible: can be cancelled while waiting
    public boolean bookInterruptible(String customer) throws InterruptedException {
        lock.lockInterruptibly(); // Throws InterruptedException if interrupted while waiting
        try {
            return bookSeat(customer);
        } finally {
            lock.unlock();
        }
    }
}
```

**ReentrantLock extras:**

```java
lock.getHoldCount();        // How many times current thread holds this lock
lock.isHeldByCurrentThread(); // Is current thread the owner?
lock.isLocked();            // Is any thread holding this lock?
lock.getQueueLength();      // Estimate of threads waiting for this lock
lock.hasQueuedThreads();    // Any threads waiting?
```

---

### 8.2 ReadWriteLock

Allows **multiple concurrent readers** OR **one exclusive writer**. Ideal for read-heavy workloads.

```java
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class ProductCatalog {
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Map<String, Product> catalog = new HashMap<>();

    // Multiple threads can call this simultaneously
    public Product getProduct(String id) {
        rwLock.readLock().lock();
        try {
            return catalog.get(id);
        } finally {
            rwLock.readLock().unlock();
        }
    }

    // Only one thread at a time — all readers blocked during write
    public void updateProduct(String id, Product product) {
        rwLock.writeLock().lock();
        try {
            catalog.put(id, product);
        } finally {
            rwLock.writeLock().unlock();
        }
    }

    // Reading + writing — use write lock for entire block
    public Product getOrCreate(String id, Supplier<Product> factory) {
        // Optimistic: try read lock first
        rwLock.readLock().lock();
        try {
            Product p = catalog.get(id);
            if (p != null) return p;
        } finally {
            rwLock.readLock().unlock();
        }
        // Must upgrade to write lock (ReadWriteLock doesn't support upgrade directly)
        rwLock.writeLock().lock();
        try {
            return catalog.computeIfAbsent(id, k -> factory.get());
        } finally {
            rwLock.writeLock().unlock();
        }
    }
}
```

**ReadWriteLock rules:**

```
State matrix:
               Read Lock    Write Lock
Read Lock:     ✅ Allowed   ❌ Blocked
Write Lock:    ❌ Blocked   ❌ Blocked
```

---

### 8.3 StampedLock (Java 8+)

`StampedLock` offers three modes, including **optimistic reading** — the fastest option for read-heavy scenarios with infrequent writes.

```java
import java.util.concurrent.locks.StampedLock;

public class Point {
    private double x, y;
    private final StampedLock lock = new StampedLock();

    // Mode 1: Optimistic read — no actual lock acquired!
    public double distanceFromOrigin() {
        long stamp = lock.tryOptimisticRead();  // Returns a stamp (non-zero = no write in progress)
        double currentX = x;
        double currentY = y;

        if (!lock.validate(stamp)) {    // Did a write occur during our read?
            // Fall back to a real read lock
            stamp = lock.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                lock.unlockRead(stamp);
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }

    // Mode 2: Read lock — blocks writers, allows concurrent readers
    public double getX() {
        long stamp = lock.readLock();
        try {
            return x;
        } finally {
            lock.unlockRead(stamp);
        }
    }

    // Mode 3: Write lock — exclusive
    public void move(double deltaX, double deltaY) {
        long stamp = lock.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            lock.unlockWrite(stamp);
        }
    }
}
```

> ⚠️ `StampedLock` is **not reentrant** — a thread that tries to acquire a write lock while holding a read lock will **deadlock**. Use with care.

---

### 8.4 Lock Comparison

| Lock Type | Multiple Readers | Try-Lock | Timed Lock | Interruptible | Fair | Reentrant | Performance |
|---|---|---|---|---|---|---|---|
| `synchronized` | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | JVM-optimized |
| `ReentrantLock` | ❌ | ✅ | ✅ | ✅ | Optional | ✅ | Moderate |
| `ReentrantReadWriteLock` | ✅ | ✅ | ✅ | ✅ | Optional | ✅ | High (read-heavy) |
| `StampedLock` | ✅ (optimistic) | ✅ | ✅ | ✅ | ❌ | ❌ | Highest |

**When to choose what:**

```
Low contention, simple methods     → synchronized
Need tryLock/timeout/interruptible → ReentrantLock
Read-heavy data structures         → ReentrantReadWriteLock
Ultra-high-read throughput         → StampedLock (if non-reentrant is OK)
High-frequency counters            → AtomicLong / LongAdder
Low-contention updates             → AtomicReference + CAS
```

---

## Summary

| Class | Mechanism | Best for |
|---|---|---|
| `AtomicInteger/Long` | CAS instruction | Counters, flags, CAS-based algorithms |
| `LongAdder` | Distributed cells | High-concurrency increment-only counters |
| `AtomicReference` | CAS on object reference | Lock-free data structures |
| `ReentrantLock` | AQS-based mutex | Complex locking (try, timeout, interruptible) |
| `ReadWriteLock` | AQS read/write | Read-heavy shared data (caches, catalogs) |
| `StampedLock` | Optimistic + stamp | Extremely read-heavy, non-reentrant access |

---

**Navigation**

[← Java Memory Model](03-java-memory-model.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Wait, Notify & Condition](05-wait-notify-condition.md)

---

*Last updated: 2026 | Java 21 LTS*
