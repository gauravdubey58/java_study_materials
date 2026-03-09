# 06 — Executor Framework

> Covers: Why executors, executor hierarchy, thread pool types, ThreadPoolExecutor, pool sizing, rejection policies, ScheduledExecutor, and ForkJoinPool.

---

**Navigation**

[← Wait, Notify & Condition](05-wait-notify-condition.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Future & CompletableFuture](07-future-completablefuture.md)

---

## Table of Contents

- [10.1 Why Executors?](#101-why-executors)
- [10.2 Executor Hierarchy](#102-executor-hierarchy)
- [10.3 Creating Thread Pools](#103-creating-thread-pools)
- [10.4 ThreadPoolExecutor — Full Control](#104-threadpoolexecutor--full-control)
- [10.5 Thread Pool Sizing Formula](#105-thread-pool-sizing-formula)
- [10.6 Scheduled Executor](#106-scheduled-executor)
- [10.7 ForkJoinPool — Divide and Conquer](#107-forkjoinpool--divide-and-conquer)
- [Summary](#summary)

---

## 10.1 Why Executors?

| Problem with raw `new Thread(...)` | Executor solution |
|---|---|
| Thread creation is expensive (OS-level syscall, ~1ms) | Thread **reuse** via pooling |
| No upper bound on thread count → OOM | Configurable **pool sizes** and bounded queues |
| No lifecycle management | Graceful **shutdown** support |
| No result retrieval mechanism | `Future<T>` and `CompletableFuture<T>` |
| No error/exception handling hooks | `UncaughtExceptionHandler`, `afterExecute()` |

```java
// Bad: a new thread per request under load → thousands of threads → OOM
void handleRequest(Request req) {
    new Thread(() -> process(req)).start(); // DON'T DO THIS
}

// Good: submit to a bounded pool
ExecutorService pool = Executors.newFixedThreadPool(10);
void handleRequest(Request req) {
    pool.submit(() -> process(req));
}
```

---

## 10.2 Executor Hierarchy

```
Executor                         (interface) — execute(Runnable)
  └── ExecutorService            (interface) — submit(), shutdown(), invokeAll()
        ├── AbstractExecutorService
        │     └── ThreadPoolExecutor     ← primary implementation
        │           └── ScheduledThreadPoolExecutor
        └── ForkJoinPool                 ← work-stealing parallel pool
```

---

## 10.3 Creating Thread Pools

```java
import java.util.concurrent.*;

// --- Fixed pool ---
// N threads always alive. Queue is unbounded (LinkedBlockingQueue).
// Best for: steady workloads, CPU-bound tasks.
ExecutorService fixed = Executors.newFixedThreadPool(4);

// --- Cached pool ---
// Grows/shrinks dynamically. Idle threads die after 60 seconds.
// ⚠️ WARNING: Unbounded — can create unlimited threads under heavy load → OOM
// Best for: short-lived async I/O tasks with bursty traffic.
ExecutorService cached = Executors.newCachedThreadPool();

// --- Single thread ---
// Sequential execution, tasks run one at a time, in submission order.
// If the thread dies, a replacement is created automatically.
// Best for: ordered event processing, single-writer to a resource.
ExecutorService single = Executors.newSingleThreadExecutor();

// --- Scheduled pool ---
// Supports delayed and periodic task execution.
// Best for: cron-like tasks, cache refresh, health checks.
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(2);

// --- Work-stealing pool ---
// Backed by ForkJoinPool. Threads steal tasks from each other.
// Best for: recursive, divide-and-conquer, CPU-bound parallel tasks.
ExecutorService workStealing = Executors.newWorkStealingPool();
// Or: Executors.newWorkStealingPool(parallelism)
```

> ⚠️ **Production warning:** `Executors.newFixedThreadPool` and `newSingleThreadExecutor` use an **unbounded `LinkedBlockingQueue`** — tasks can queue up indefinitely, causing memory exhaustion. Prefer `ThreadPoolExecutor` with a **bounded queue** in production.

---

## 10.4 ThreadPoolExecutor — Full Control

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4,                                        // corePoolSize: min threads always alive
    10,                                       // maximumPoolSize: max threads under load
    60, TimeUnit.SECONDS,                     // keepAliveTime: idle non-core threads die after this
    new ArrayBlockingQueue<>(500),            // workQueue: bounded queue (blocks/rejects when full)
    new ThreadFactory() {                     // threadFactory: customize thread names/daemon status
        private final AtomicInteger count = new AtomicInteger(1);
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r, "order-worker-" + count.getAndIncrement());
            t.setDaemon(true);
            return t;
        }
    },
    new ThreadPoolExecutor.CallerRunsPolicy() // rejectedExecutionHandler
);

// Optional: allow core threads to time out too
executor.allowCoreThreadTimeOut(true);
```

### How ThreadPoolExecutor Decides Thread Count

```
Submit task
    │
    ├─ threads < corePoolSize?      → Create new thread (even if idle threads exist)
    │
    ├─ queue not full?              → Add to queue (don't create new thread)
    │
    ├─ threads < maximumPoolSize?   → Create new thread (queue is full)
    │
    └─ queue full + max threads?    → Apply rejection policy
```

### Rejection Policies

| Policy | Behavior | Use when |
|---|---|---|
| `AbortPolicy` (default) | Throws `RejectedExecutionException` | Caller must handle overload |
| `CallerRunsPolicy` | Caller thread runs the task | Natural back-pressure — slows producers |
| `DiscardPolicy` | Silently **drops** the task | Metrics/logging — losing some is OK |
| `DiscardOldestPolicy` | Drops the **oldest** queued task, retries submission | Fresh data is more important |

```java
// Custom rejection handler — log + alert
executor.setRejectedExecutionHandler((task, pool) -> {
    log.error("Task rejected! Pool size: {}, Queue size: {}",
              pool.getPoolSize(), pool.getQueue().size());
    metricsClient.increment("threadpool.rejected");
    // Optionally save task to DB for retry
});
```

### Graceful Shutdown

```java
executor.shutdown();                         // Stop accepting new tasks
boolean done = executor.awaitTermination(30, TimeUnit.SECONDS);
if (!done) {
    List<Runnable> dropped = executor.shutdownNow(); // Interrupt running tasks
    log.warn("{} tasks were not completed", dropped.size());
}
```

### Monitoring

```java
executor.getPoolSize();          // Current number of threads
executor.getActiveCount();       // Threads currently executing tasks
executor.getCorePoolSize();      // Core pool size
executor.getMaximumPoolSize();   // Max pool size
executor.getQueue().size();      // Current queue depth
executor.getCompletedTaskCount(); // Total tasks completed since start
executor.getTaskCount();         // Total tasks submitted
```

---

## 10.5 Thread Pool Sizing Formula

### CPU-Bound Tasks

Tasks that spend most time computing (sorting, encryption, compression):

```
Pool size = Number of CPU cores + 1

Rationale: +1 extra thread to keep the CPU busy during occasional
           page faults or minor blocking pauses.

Example: 8-core machine → pool size = 9
```

### I/O-Bound Tasks

Tasks that spend most time waiting (DB queries, HTTP calls, file I/O):

```
Pool size = Number of CPU cores × (1 + Wait time / Compute time)

Example: 8 cores, 90% wait time (wait/compute = 9)
         Pool size = 8 × (1 + 9) = 80 threads

Rationale: While N threads wait for I/O, N more can run on CPU.
```

### Mixed Workloads

```
Use SEPARATE thread pools for CPU-bound and I/O-bound tasks.

ExecutorService cpuPool = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors() + 1);

ExecutorService ioPool = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors() * 10);
```

> **Always validate with load testing** — formulas are starting points, not gospel.

---

## 10.6 Scheduled Executor

```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

// Run ONCE after a delay
scheduler.schedule(
    () -> sendAlertEmail(),
    5, TimeUnit.SECONDS           // Run once after 5 seconds
);

// Run REPEATEDLY at a fixed rate
// Next run starts X time after the LAST START
// If task takes longer than period → next run starts immediately after task finishes
scheduler.scheduleAtFixedRate(
    () -> refreshProductCache(),
    0,                            // Initial delay
    10, TimeUnit.SECONDS          // Period: run every 10 seconds
);

// Run REPEATEDLY with a fixed delay
// Next run starts X time after the LAST FINISH
// Safer if task duration is variable — no overlap possible
scheduler.scheduleWithFixedDelay(
    () -> pollExternalQueue(),
    0,                            // Initial delay
    10, TimeUnit.SECONDS          // Delay between end of one run and start of next
);

// Cancel a scheduled task
ScheduledFuture<?> task = scheduler.scheduleAtFixedRate(...);
task.cancel(false); // false = don't interrupt if running; true = interrupt

// Graceful shutdown
scheduler.shutdown();
if (!scheduler.awaitTermination(30, TimeUnit.SECONDS)) {
    scheduler.shutdownNow();
}
```

**scheduleAtFixedRate vs scheduleWithFixedDelay:**

```
scheduleAtFixedRate (period = 5s):
  Task A:  [==2s==]
                    [==gap 3s==]
                               Task B: [==2s==]
           |←────── 5s ──────→|

scheduleWithFixedDelay (delay = 5s):
  Task A:  [==2s==]
                   [══════ 5s delay ══════]
                                          Task B: [==2s==]
```

---

## 10.7 ForkJoinPool — Divide and Conquer

`ForkJoinPool` is designed for **recursive, divide-and-conquer** tasks using **work-stealing** for efficient load balancing.

### Work-Stealing Architecture

```
Worker Thread Deques:
  Thread 1: [TaskA1][TaskA2][TaskA3] ← pushes/pops from front (LIFO)
  Thread 2: [TaskB1][TaskB2]         ← idle → steals from BACK of Thread 1 (FIFO)
  Thread 3: [      ]                 ← idle → steals from BACK of Thread 2
```

### RecursiveTask (Returns Result)

```java
public class ParallelArraySum extends RecursiveTask<Long> {
    private static final int THRESHOLD = 1000; // Split if chunk > 1000 elements
    private final int[] array;
    private final int start, end;

    public ParallelArraySum(int[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end   = end;
    }

    @Override
    protected Long compute() {
        if (end - start <= THRESHOLD) {
            // Base case: compute directly (no more splitting)
            long sum = 0;
            for (int i = start; i < end; i++) sum += array[i];
            return sum;
        }
        // Split into two halves
        int mid = (start + end) / 2;
        ParallelArraySum left  = new ParallelArraySum(array, start, mid);
        ParallelArraySum right = new ParallelArraySum(array, mid,   end);

        left.fork();                         // Submit left to pool (async)
        long rightResult = right.compute();  // Compute right in current thread
        long leftResult  = left.join();      // Wait for left result

        return leftResult + rightResult;
    }
}

// Usage
int[] data = new int[10_000_000];
ForkJoinPool pool = ForkJoinPool.commonPool(); // Shared JVM-wide pool
long total = pool.invoke(new ParallelArraySum(data, 0, data.length));
```

### RecursiveAction (No Return Value)

```java
public class ParallelSort extends RecursiveAction {
    private final int[] array;
    private final int start, end;
    private static final int THRESHOLD = 500;

    @Override
    protected void compute() {
        if (end - start <= THRESHOLD) {
            Arrays.sort(array, start, end);
            return;
        }
        int mid = (start + end) / 2;
        ParallelSort left  = new ParallelSort(array, start, mid);
        ParallelSort right = new ParallelSort(array, mid,   end);
        invokeAll(left, right); // Fork both and wait for both to finish
        merge(array, start, mid, end);
    }
}
```

### Common Pool Warning

```java
// ForkJoinPool.commonPool() is SHARED across the entire JVM
// Used by: CompletableFuture.supplyAsync(), parallel streams, ForkJoin tasks

// DANGER: blocking I/O in commonPool starves all async operations system-wide
CompletableFuture.supplyAsync(() -> {
    dbConnection.query("SELECT ...");  // Blocks a commonPool thread!
    return result;
});

// FIX: provide a dedicated executor for I/O
CompletableFuture.supplyAsync(() -> dbConnection.query("..."), dedicatedIoExecutor);
```

---

## Summary

| Pool Type | Threads | Queue | Best for |
|---|---|---|---|
| `newFixedThreadPool(n)` | Fixed N | Unbounded | Steady CPU workloads |
| `newCachedThreadPool()` | Unlimited | SynchronousQueue | Short bursty I/O |
| `newSingleThreadExecutor()` | 1 | Unbounded | Sequential ordering |
| `newScheduledThreadPool(n)` | Fixed N | DelayQueue | Cron jobs, periodic tasks |
| `ThreadPoolExecutor` (custom) | Core–Max | Bounded | Production services |
| `ForkJoinPool` | CPU cores | Work-stealing deques | Recursive parallel algorithms |

---

**Navigation**

[← Wait, Notify & Condition](05-wait-notify-condition.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Future & CompletableFuture](07-future-completablefuture.md)

---

*Last updated: 2026 | Java 21 LTS*
