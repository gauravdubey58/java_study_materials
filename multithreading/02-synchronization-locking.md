# 02 — Thread Synchronization & Locking

> Covers: Race conditions, synchronized methods and blocks, static synchronization, reentrant locking, and intrinsic locks.

---

**Navigation**

[← Thread Fundamentals](01-thread-fundamentals.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Java Memory Model](03-java-memory-model.md)

---

## Table of Contents

- [4.1 The Problem — Race Conditions](#41-the-problem--race-conditions)
- [4.2 Synchronized Method](#42-synchronized-method)
- [4.3 Synchronized Block (Finer Granularity)](#43-synchronized-block-finer-granularity)
- [4.4 Static Synchronized](#44-static-synchronized)
- [4.5 Reentrant Locking](#45-reentrant-locking)
- [Summary](#summary)

---

## 4.1 The Problem — Race Conditions

A **race condition** occurs when two or more threads access shared data concurrently and the outcome depends on the order of their execution.

```java
public class BankAccount {
    private int balance = 1000;

    // NOT thread-safe — race condition!
    public void withdraw(int amount) {
        if (balance >= amount) {
            // Thread A reads balance = 1000 ─┐
            // Thread B reads balance = 1000  │ Both pass the check
            //                                ↓
            balance -= amount; // Both withdraw → balance can go negative!
        }
    }
}
```

**Execution interleaving that causes the bug:**

```
Time  Thread A                   Thread B
  1   reads balance = 1000
  2                              reads balance = 1000
  3   balance -= 800  → 200
  4                              balance -= 800  → 200 (wrong! should be -600)
```

The **check-then-act** sequence is not atomic — another thread can interpose between the check and the action.

---

## 4.2 Synchronized Method

The `synchronized` keyword on an instance method acquires the **intrinsic lock** (monitor) on `this` before entering and releases it on exit.

```java
public class BankAccount {
    private int balance = 1000;

    // Acquires lock on 'this' — only one thread in any synchronized method at a time
    public synchronized void withdraw(int amount) {
        if (balance >= amount) {
            balance -= amount;
        }
    }

    public synchronized void deposit(int amount) {
        balance += amount;
    }

    // Reads also need synchronization if concurrent writes exist
    public synchronized int getBalance() {
        return balance;
    }
}
```

> **Key rule:** All `synchronized` instance methods on the **same object** share the same intrinsic lock. Only one thread can be inside any of them at a time.

---

## 4.3 Synchronized Block (Finer Granularity)

Synchronized blocks allow you to lock only the critical section rather than the entire method, and to use **custom lock objects** for independent sections.

```java
public class OrderService {
    private final Object inventoryLock = new Object();
    private final Object paymentLock   = new Object();
    private int inventory = 100;
    private double totalRevenue = 0;

    public void placeOrder(int qty, double price) {

        // --- Critical section 1: inventory (uses inventoryLock) ---
        synchronized (inventoryLock) {
            if (inventory < qty) throw new RuntimeException("Out of stock");
            inventory -= qty;
        }
        // Payment and inventory use different locks →
        // multiple threads can process payments and check inventory concurrently

        // --- Critical section 2: payment (uses paymentLock) ---
        synchronized (paymentLock) {
            totalRevenue += price * qty;
        }
    }
}
```

**Why this is better:**

| Approach | Concurrency | Lock scope |
|---|---|---|
| `synchronized` on whole method | Only 1 thread in entire method | Entire method |
| `synchronized` on `this` block | Only 1 thread in critical section | Specific block |
| Separate lock objects | Different sections run concurrently | Independent sections |

---

## 4.4 Static Synchronized

A `synchronized` keyword on a **static method** acquires the lock on the **`Class` object** (e.g., `ConnectionPool.class`), not on an instance.

```java
public class ConnectionPool {
    private static int connectionCount = 0;

    // Locks on ConnectionPool.class — affects ALL instances
    public static synchronized int getConnectionCount() {
        return connectionCount;
    }

    public static synchronized void increment() {
        connectionCount++;
    }
}
```

**Object lock vs. Class lock:**

```
Instance methods (synchronized):   lock = this            (per-object)
Static methods (synchronized):     lock = MyClass.class   (shared across all instances)
```

> ⚠️ Never mix static and instance synchronized methods to protect the **same** shared state — they use **different locks** and provide no mutual exclusion between them.

---

## 4.5 Reentrant Locking

Java's intrinsic locks (and `ReentrantLock`) are **reentrant** — a thread can re-acquire a lock it already holds without deadlocking itself.

```java
public class RecursiveLogger {

    public synchronized void logWarning(String msg) {
        System.out.println("[WARN] " + msg);
        logInternal(msg); // Calls another synchronized method — safe! (reentrant)
    }

    public synchronized void logInternal(String msg) {
        // This thread already holds the lock on 'this'
        // Reentrant lock: same thread re-acquires → increments hold count
        System.out.println("[INTERNAL] " + msg);
    }
}
```

**How reentrancy works internally:**

```
Thread A calls logWarning()
  → Acquires lock on 'this' (hold count = 1)
  → Calls logInternal()
    → Re-acquires lock on 'this' (hold count = 2)  ← same thread, allowed
    → Returns (hold count drops to 1)
  → Returns (hold count drops to 0 → lock released)
```

Without reentrancy, calling `logInternal()` from `logWarning()` would **deadlock** (thread waiting for a lock it already holds).

---

## How Intrinsic Locks Work Internally

Every Java object has an associated **monitor** (a.k.a. intrinsic lock). The JVM uses the object's **mark word** (in the object header) to track the lock state.

```
Object Header Mark Word States:
  Unlocked:       [ hashcode | age | 0 | 01 ]
  Biased locked:  [ thread ID | epoch | age | 1 | 01 ]
  Thin locked:    [ pointer to stack lock record | 00 ]
  Fat/inflated:   [ pointer to monitor object | 10 ]
  Marked for GC:  [ forwarding pointer | 11 ]
```

The JVM applies progressive optimizations:
1. **Biased locking** — assume single-thread access; no CAS needed (deprecated Java 15+).
2. **Thin lock (stack lock)** — use CAS on mark word; fast for low contention.
3. **Fat lock (inflated)** — fall back to OS mutex when contention is detected.

---

## Summary

| Construct | Lock target | Scope | Use case |
|---|---|---|---|
| `synchronized` method | `this` (instance) | Whole method | Simple thread-safe methods |
| `synchronized(this)` block | `this` | Part of method | Reduce lock scope |
| `synchronized(customLock)` block | Custom object | Independent sections | Decouple unrelated critical sections |
| `static synchronized` method | `ClassName.class` | Whole static method | Protect static/class-level state |

**Best practices:**
- Keep synchronized blocks **as small as possible** — minimize time holding the lock.
- Never call unknown external code (e.g., listener callbacks) while holding a lock — risk of deadlock.
- Prefer `java.util.concurrent` utilities over raw `synchronized` for complex scenarios.

---

**Navigation**

[← Thread Fundamentals](01-thread-fundamentals.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Java Memory Model](03-java-memory-model.md)

---

*Last updated: 2026 | Java 21 LTS*
