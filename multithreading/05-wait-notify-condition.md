# 05 — Wait, Notify & Condition Variables

> Covers: Object.wait/notify, the classic producer-consumer pattern, spurious wakeups, and Condition variables with ReentrantLock.

---

**Navigation**

[← Atomic Classes & JUC Locks](04-atomic-concurrent-locks.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Executor Framework](06-executor-framework.md)

---

## Table of Contents

- [9.1 Object.wait() / notify() — Classic Producer-Consumer](#91-objectwait--notify--classic-producer-consumer)
  - [Rules for wait/notify](#rules-for-waitnotify)
  - [Spurious Wakeups](#spurious-wakeups)
  - [notify() vs notifyAll()](#notify-vs-notifyall)
- [9.2 Condition Variables with ReentrantLock](#92-condition-variables-with-reentrantlock)
  - [Why Condition over wait/notify?](#why-condition-over-waitnotify)
- [Summary](#summary)

---

## 9.1 Object.wait() / notify() — Classic Producer-Consumer

`wait()`, `notify()`, and `notifyAll()` are methods on **every Java object** and are the foundation of inter-thread communication.

### How they work

```
Monitor (intrinsic lock)
│
├── Entry Set   — threads blocked trying to acquire the lock (BLOCKED)
│
└── Wait Set    — threads that called wait() inside synchronized (WAITING)
                  → released the lock, waiting for notify()
```

```java
synchronized (lock) {
    wait();       // 1. Releases the lock   2. Suspends this thread   3. Moves to wait set
}
// After notify() + re-acquiring lock → execution resumes here
```

### Bounded Buffer (Classic Producer-Consumer)

```java
public class BoundedBuffer<T> {
    private final Queue<T> queue    = new LinkedList<>();
    private final int      capacity;

    public BoundedBuffer(int capacity) {
        this.capacity = capacity;
    }

    // Producer
    public synchronized void produce(T item) throws InterruptedException {
        while (queue.size() == capacity) { // Use WHILE — not if (spurious wakeups!)
            wait();                        // Release lock, wait for consumer to make space
        }
        queue.offer(item);
        notifyAll();                       // Wake consumers waiting for items
    }

    // Consumer
    public synchronized T consume() throws InterruptedException {
        while (queue.isEmpty()) {          // Use WHILE — not if (spurious wakeups!)
            wait();                        // Release lock, wait for producer to add items
        }
        T item = queue.poll();
        notifyAll();                       // Wake producers waiting for space
        return item;
    }
}
```

---

### Rules for wait/notify

| Rule | Reason |
|---|---|
| Must be called inside `synchronized` block | `wait()`/`notify()` require ownership of the monitor — else `IllegalMonitorStateException` |
| Must call on the object being synchronized on | `synchronized(obj) { obj.wait(); }` — not `this.wait()` inside `synchronized(obj)` |
| Always use `while` loop, not `if` | Spurious wakeups and multiple-producer situations require re-checking the condition |
| Call `notifyAll()` over `notify()` in most cases | `notify()` wakes an arbitrary thread — may wake the wrong one |

```java
// WRONG — using if:
synchronized (lock) {
    if (queue.isEmpty()) {  // Thread wakes spuriously, queue still empty → NPE!
        wait();
    }
    process(queue.poll()); // queue.poll() returns null → crash
}

// CORRECT — using while:
synchronized (lock) {
    while (queue.isEmpty()) { // Re-check after every wakeup
        wait();
    }
    process(queue.poll()); // Safe — guaranteed non-empty
}
```

---

### Spurious Wakeups

A **spurious wakeup** is when a thread wakes from `wait()` without being `notify()`'d. This is allowed by the POSIX thread standard (pthreads) and the JVM exposes this behavior.

```java
// Spurious wakeup protection pattern:
while (conditionNotMet()) {
    wait(); // If woken spuriously, loop re-checks and waits again
}
```

---

### notify() vs notifyAll()

```
notify()     → Wakes ONE arbitrary thread from the wait set
notifyAll()  → Wakes ALL threads from the wait set
```

| Scenario | Use |
|---|---|
| All waiting threads are **identical** and interchangeable | `notify()` is safe and more efficient |
| Waiting threads have **different conditions** (producers + consumers) | `notifyAll()` — otherwise a producer might wake a producer (wrong!) |
| **Default recommendation** | Use `notifyAll()` unless performance profiling proves `notify()` is needed |

```java
// Example of notify() going wrong:
// 3 threads waiting: P1 (producer), C1 (consumer), C2 (consumer)
// Queue is full — C1 and C2 should be woken to consume
// notify() wakes P1 (arbitrary!) — P1 checks queue, still full → waits again
// → C1 and C2 never woken → deadlock!
```

---

## 9.2 Condition Variables with ReentrantLock

`Condition` (from `java.util.concurrent.locks`) is the `ReentrantLock` equivalent of `wait/notify`. It supports **multiple distinct wait sets** per lock — solving the `notifyAll()` thundering herd problem.

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class OrderQueue {
    private final ReentrantLock lock    = new ReentrantLock();
    private final Condition     notFull  = lock.newCondition(); // Producers wait here
    private final Condition     notEmpty = lock.newCondition(); // Consumers wait here
    private final Queue<Order>  queue   = new LinkedList<>();
    private final int           maxSize;

    public OrderQueue(int maxSize) {
        this.maxSize = maxSize;
    }

    public void enqueue(Order order) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == maxSize) {
                notFull.await();        // Producer waits — only producers are here
            }
            queue.offer(order);
            notEmpty.signal();          // Wake exactly ONE consumer (not all!)
        } finally {
            lock.unlock();
        }
    }

    public Order dequeue() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();       // Consumer waits — only consumers are here
            }
            Order order = queue.poll();
            notFull.signal();           // Wake exactly ONE producer (not all!)
            return order;
        } finally {
            lock.unlock();
        }
    }
}
```

### Why Condition over wait/notify?

| Feature | `Object.wait/notify` | `Condition` |
|---|---|---|
| Multiple wait sets per lock | ❌ One wait set per object | ✅ Multiple Conditions per ReentrantLock |
| Targeted signaling | ❌ `notifyAll()` wakes everyone | ✅ `signal()` wakes only the relevant waiting group |
| Timed wait | ✅ `wait(timeout)` | ✅ `await(time, unit)` — more expressive |
| Interruptible vs uninterruptible | ❌ Always interruptible | ✅ `awaitUninterruptibly()` available |
| Integration | Only with `synchronized` | Only with `ReentrantLock` |

### Condition API

```java
Condition condition = lock.newCondition();

condition.await();                           // Like Object.wait()
condition.await(5, TimeUnit.SECONDS);        // Timed wait
condition.awaitUntil(deadline);              // Wait until absolute Date
condition.awaitUninterruptibly();            // Cannot be interrupted while waiting
condition.awaitNanos(nanos);                 // Timed in nanoseconds — returns remaining wait

condition.signal();                          // Like Object.notify()   — wake one
condition.signalAll();                       // Like Object.notifyAll() — wake all
```

### Complete Producer-Consumer with Metrics

```java
public class MeteredQueue<T> {
    private final ReentrantLock lock     = new ReentrantLock();
    private final Condition     notFull  = lock.newCondition();
    private final Condition     notEmpty = lock.newCondition();
    private final Queue<T>      queue    = new LinkedList<>();
    private final int           capacity;

    private long totalEnqueued  = 0;
    private long totalDequeued  = 0;
    private long totalWaitTimeMs = 0;

    public MeteredQueue(int capacity) {
        this.capacity = capacity;
    }

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            long waitStart = System.currentTimeMillis();
            while (queue.size() == capacity) {
                notFull.await();
            }
            totalWaitTimeMs += System.currentTimeMillis() - waitStart;
            queue.offer(item);
            totalEnqueued++;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();
            }
            T item = queue.poll();
            totalDequeued++;
            notFull.signal();
            return item;
        } finally {
            lock.unlock();
        }
    }

    public int size() {
        lock.lock();
        try { return queue.size(); }
        finally { lock.unlock(); }
    }
}
```

---

## Summary

| Concept | Key Takeaway |
|---|---|
| `wait()` | Releases lock + suspends thread — must be in `synchronized` block |
| `notify()` | Wakes one arbitrary thread from wait set |
| `notifyAll()` | Wakes all threads — safer default, use when conditions differ |
| `while` over `if` | Guards against spurious wakeups and multiple-waiter edge cases |
| `Condition` | Multiple wait sets per lock → targeted signaling → fewer unnecessary wakeups |
| `signal()` vs `signalAll()` | `signal()` only when all waiters are equivalent; `signalAll()` when in doubt |

---

**Navigation**

[← Atomic Classes & JUC Locks](04-atomic-concurrent-locks.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Executor Framework](06-executor-framework.md)

---

*Last updated: 2026 | Java 21 LTS*
