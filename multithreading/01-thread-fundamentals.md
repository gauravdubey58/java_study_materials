# 01 — Thread Fundamentals

> Covers: What is a thread, process vs thread, thread lifecycle, creating threads, interruption, join, and ThreadLocal.

---

**Navigation**

[🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Synchronization & Locking](02-synchronization-locking.md)

---

## Table of Contents

- [1. Fundamentals of Threads](#1-fundamentals-of-threads)
  - [1.1 What is a Thread?](#11-what-is-a-thread)
  - [1.2 Concurrency vs. Parallelism](#12-concurrency-vs-parallelism)
  - [1.3 Thread Properties](#13-thread-properties)
  - [1.4 Daemon vs. User Threads](#14-daemon-vs-user-threads)
- [2. Thread Lifecycle](#2-thread-lifecycle)
  - [Thread State Transitions](#thread-state-transitions)
- [3. Creating Threads](#3-creating-threads)
  - [3.1 Extending Thread Class](#31-extending-thread-class)
  - [3.2 Implementing Runnable](#32-implementing-runnable-preferred)
  - [3.3 Implementing Callable](#33-implementing-callable-returns-result)
  - [3.4 Thread Interruption](#34-thread-interruption)
  - [3.5 Thread Join](#35-thread-join)
  - [3.6 ThreadLocal](#36-threadlocal)

---

## 1. Fundamentals of Threads

### 1.1 What is a Thread?

A **thread** is the smallest unit of execution within a process. Multiple threads share the same process memory space (heap, code, data) but each maintains its own:

- **Program Counter** — address of the next instruction to execute.
- **Stack** — local variables, method call frames.
- **Thread-local storage** — via `ThreadLocal<T>`.
- **Registers** — CPU register state.

```
Process (JVM)
├── Shared Heap Memory
├── Metaspace (class metadata)
└── Threads
    ├── Thread-1: [Stack | PC | Registers]
    ├── Thread-2: [Stack | PC | Registers]
    └── Thread-3: [Stack | PC | Registers]
```

---

### 1.2 Concurrency vs. Parallelism

| Concept | Definition | Example |
|---|---|---|
| **Concurrency** | Multiple tasks make progress by interleaving execution (single or multi-core). | 2 threads sharing 1 CPU core. |
| **Parallelism** | Multiple tasks execute simultaneously on multiple CPU cores. | 4 threads on 4 cores. |
| **Multithreading** | A form of concurrency using threads within a single process. | Java thread pools. |

---

### 1.3 Thread Properties

```java
Thread t = new Thread(() -> System.out.println("Hello"));
t.setName("worker-1");              // Thread name
t.setPriority(Thread.MAX_PRIORITY); // Priority: 1 (MIN) to 10 (MAX), default 5
t.setDaemon(true);                  // Daemon thread — JVM exits when only daemons remain
t.start();

System.out.println(t.getId());      // Unique thread ID
System.out.println(t.getState());   // Current ThreadState
System.out.println(t.isAlive());    // true if started and not terminated
System.out.println(t.isDaemon());   // true if daemon
```

---

### 1.4 Daemon vs. User Threads

| Aspect | User Thread | Daemon Thread |
|---|---|---|
| JVM exit | JVM waits for all user threads to finish | JVM exits if only daemon threads remain |
| Use case | Business logic, request processing | GC, housekeeping, background monitors |
| Default | Yes (default) | No — must call `setDaemon(true)` before `start()` |
| Example | `main` thread, HTTP handlers | GC thread, JIT compiler thread |

> ⚠️ Daemon threads do **not** have their `finally` blocks guaranteed to run when the JVM shuts down. Never use daemon threads for tasks that must complete (e.g., writing to a database).

---

## 2. Thread Lifecycle

```
                        ┌──────────────────────────────┐
                        │              NEW              │
                        │   Thread t = new Thread(...)  │
                        └──────────────┬───────────────┘
                                       │ t.start()
                        ┌──────────────▼───────────────┐
                   ┌────│           RUNNABLE            │◄────────────┐
                   │    │  (Ready to run OR running)    │             │
                   │    └──────────────┬───────────────┘             │
                   │                   │                              │
          lock not │    synchronized   │ Object.wait()          notify/
          available│    block entry    │ Thread.join()          notifyAll/
                   │                   │ LockSupport.park()     timeout
                   │    ┌──────────────▼───────────────┐             │
                   │    │           WAITING             │─────────────┘
                   │    │      (indefinitely)           │
                   │    └───────────────────────────────┘
                   │
                   │    ┌───────────────────────────────┐
                   │    │        TIMED_WAITING          │
                   │    │  Thread.sleep(ms)             │
                   │    │  Object.wait(timeout)         │
                   │    │  Thread.join(timeout)         │
                   │    └───────────────────────────────┘
                   │
      ┌────────────▼───────────────┐
      │           BLOCKED          │
      │  (waiting for monitor lock)│
      └────────────────────────────┘

      ┌────────────────────────────┐
      │         TERMINATED        │
      │  (run() method completed)  │
      └────────────────────────────┘
```

| State | Description | Triggered By |
|---|---|---|
| `NEW` | Thread created, not started yet. | `new Thread(...)` |
| `RUNNABLE` | Running or ready to run on CPU. | `thread.start()` |
| `BLOCKED` | Waiting to acquire a monitor lock. | Entering `synchronized` block held by another thread |
| `WAITING` | Waiting indefinitely for signal. | `Object.wait()`, `Thread.join()`, `LockSupport.park()` |
| `TIMED_WAITING` | Waiting for a specified duration. | `Thread.sleep(ms)`, `wait(timeout)`, `join(timeout)` |
| `TERMINATED` | Finished execution. | `run()` completes or throws exception |

---

### Thread State Transitions

```java
Thread t = new Thread(() -> {
    try {
        Thread.sleep(1000);    // RUNNABLE → TIMED_WAITING
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
});
// t.getState() == NEW

t.start();
// t.getState() == RUNNABLE or TIMED_WAITING

t.join();
// t.getState() == TERMINATED
```

---

## 3. Creating Threads

### 3.1 Extending Thread Class

```java
public class WorkerThread extends Thread {
    private final int taskId;

    public WorkerThread(int taskId) {
        super("Worker-" + taskId);
        this.taskId = taskId;
    }

    @Override
    public void run() {
        System.out.println(getName() + " processing task " + taskId);
        // task logic here
    }
}

// Usage
WorkerThread w = new WorkerThread(1);
w.start(); // Never call run() directly — that executes on current thread
```

---

### 3.2 Implementing Runnable (Preferred)

```java
public class OrderProcessor implements Runnable {
    private final Order order;

    public OrderProcessor(Order order) {
        this.order = order;
    }

    @Override
    public void run() {
        processOrder(order);
    }
}

// Usage
Thread t = new Thread(new OrderProcessor(order));
t.start();

// Or with lambda
Thread t2 = new Thread(() -> processOrder(order));
t2.start();
```

> **Prefer `Runnable` over extending `Thread`** — keeps task logic separate from the threading mechanism, allows the task to be reused with different executors, and avoids wasting the single-inheritance slot.

---

### 3.3 Implementing Callable (Returns Result)

```java
import java.util.concurrent.Callable;
import java.util.concurrent.FutureTask;

public class PriceCalculator implements Callable<Double> {
    private final String productId;

    public PriceCalculator(String productId) {
        this.productId = productId;
    }

    @Override
    public Double call() throws Exception {
        return fetchPriceFromDB(productId); // Can throw checked exceptions
    }
}

// Usage with FutureTask
FutureTask<Double> task = new FutureTask<>(new PriceCalculator("P001"));
Thread t = new Thread(task);
t.start();
Double price = task.get(); // Blocks until result is available
```

| Interface | Return type | Checked exceptions | Best used with |
|---|---|---|---|
| `Runnable` | `void` | ❌ | `Thread`, `ExecutorService.execute()` |
| `Callable<T>` | `T` | ✅ | `ExecutorService.submit()`, `FutureTask` |

---

### 3.4 Thread Interruption

Interruption is a **cooperative mechanism** — a thread cannot be forcibly stopped.

```java
Thread worker = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        try {
            doWork();
            Thread.sleep(500);
        } catch (InterruptedException e) {
            // IMPORTANT: Restore interrupt flag — sleep() clears it on throw
            Thread.currentThread().interrupt();
            System.out.println("Interrupted — shutting down cleanly");
            break;
        }
    }
});

worker.start();
Thread.sleep(2000);
worker.interrupt(); // Request interruption — does not kill the thread!
```

| Method | Clears interrupt flag? | Static? |
|---|---|---|
| `Thread.currentThread().isInterrupted()` | No | No |
| `Thread.interrupted()` | **Yes** | Yes |
| `thread.interrupt()` | N/A (sets the flag) | No |

> ⚠️ **Never swallow `InterruptedException` silently.** Always either re-throw it or restore the interrupt flag with `Thread.currentThread().interrupt()`. Swallowing it breaks interrupt-based shutdown mechanisms.

---

### 3.5 Thread Join

`join()` causes the **calling thread** to block until the **target thread** completes.

```java
Thread dataLoader = new Thread(() -> loadData());
Thread processor  = new Thread(() -> processData());

dataLoader.start();
dataLoader.join();       // Main thread waits until dataLoader finishes
processor.start();       // Only starts after data is loaded

// With timeout
dataLoader.join(5000);   // Wait max 5 seconds
if (dataLoader.isAlive()) {
    System.out.println("Load timed out — data may be incomplete");
}
```

---

### 3.6 ThreadLocal

`ThreadLocal<T>` provides **per-thread storage** — each thread gets its own independent copy of the variable.

```java
// Example: thread-safe date formatter (SimpleDateFormat is NOT thread-safe)
ThreadLocal<SimpleDateFormat> dateFormat = ThreadLocal.withInitial(
    () -> new SimpleDateFormat("yyyy-MM-dd")
);

// Real-world: per-request user context in a web app
ThreadLocal<UserContext> userContext = new ThreadLocal<>();

// In Servlet filter — set at request start
userContext.set(new UserContext(request.getUserPrincipal()));

// Deep in service layer — no need to pass as parameter
UserContext ctx = userContext.get();

// CRITICAL: Always clean up in finally to prevent memory leaks in thread pools
try {
    handleRequest();
} finally {
    userContext.remove(); // Clears the value for this thread
}
```

> ⚠️ **ThreadLocal memory leak in thread pools:** Thread pool threads are **never destroyed** — they're reused for new requests. If `remove()` is not called, stale `ThreadLocal` values from a previous request persist and are visible to the next request on the same thread. In Java 21+, prefer `ScopedValue` for virtual threads.

---

## Summary

| Concept | Key Takeaway |
|---|---|
| Thread vs Process | Threads share heap; each has own stack, PC, registers |
| Runnable vs Callable | Runnable = void + no checked ex; Callable = T + checked ex |
| `start()` vs `run()` | `start()` creates new thread; `run()` executes on current thread |
| Daemon thread | JVM exits without waiting for them — never use for critical work |
| Interruption | Cooperative — always handle `InterruptedException` properly |
| ThreadLocal | Per-thread storage — always `remove()` in thread pool contexts |

---

**Navigation**

[🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Synchronization & Locking](02-synchronization-locking.md)

---

*Last updated: 2026 | Java 21 LTS*
