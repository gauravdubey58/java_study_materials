# 11 — 100 Interview Questions

> 100 Java Multithreading interview questions with complete answers. Difficulty: 🟡 Moderate | 🔴 Complex.

---

**Navigation**

[← Real-World Scenarios](10-real-world-scenarios.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md)

---

## Table of Contents

- [Core Threading Concepts (Q1–Q15)](#core-threading-concepts-q1q15)
- [Synchronization & Locking (Q16–Q35)](#synchronization--locking-q16q35)
- [Java Memory Model & Visibility (Q36–Q45)](#java-memory-model--visibility-q36q45)
- [Executor Framework & CompletableFuture (Q46–Q60)](#executor-framework--completablefuture-q46q60)
- [Concurrent Collections (Q61–Q70)](#concurrent-collections-q61q70)
- [Common Problems & Debugging (Q71–Q85)](#common-problems--debugging-q71q85)
- [Advanced & Architecture (Q86–Q100)](#advanced--architecture-q86q100)

---

## Core Threading Concepts (Q1–Q15)

---

**Q1 🟡** What is the difference between a `process` and a `thread`? What resources does a thread share vs. own privately?

> **Answer:** A process is an isolated program with its own memory space. Threads within a process **share** heap memory, code, data segment, file handles, and open sockets. Each thread **privately owns** its stack (local variables, call frames), program counter (PC), and CPU registers. Thread creation is cheaper than process creation because no new memory space is allocated.

---

**Q2 🟡** What is the difference between `Runnable` and `Callable`? When would you use each?

> **Answer:** `Runnable.run()` returns `void` and cannot throw checked exceptions. `Callable<V>.call()` returns a typed result `V` and can throw checked exceptions. Use `Runnable` for fire-and-forget tasks. Use `Callable` when you need a result or need to propagate checked exceptions via a `Future<V>`.

---

**Q3 🟡** Why should you never call `thread.run()` directly instead of `thread.start()`?

> **Answer:** `thread.run()` executes the `run()` method on the **calling thread** — no new thread is created; it's just a regular method call. `thread.start()` creates a new OS-level thread and schedules `run()` on it. Calling `run()` directly defeats the purpose of multithreading.

---

**Q4 🟡** What happens when you call `start()` on an already-started thread?

> **Answer:** An `IllegalThreadStateException` is thrown immediately. A `Thread` object can only be started once. To re-run the task, create a new `Thread` instance.

---

**Q5 🟡** Explain the difference between `Thread.sleep()` and `Object.wait()`.

> **Answer:** `Thread.sleep(ms)` pauses the current thread for a duration **without releasing any locks** it holds. `Object.wait()` must be called within a `synchronized` block — it **releases the monitor lock** and suspends the thread until `notify()`/`notifyAll()` is called or a timeout occurs. `sleep()` is a static `Thread` method; `wait()` is an instance method on every `Object`.

---

**Q6 🟡** What is a daemon thread? What happens to daemon threads when the last user thread finishes?

> **Answer:** A daemon thread is a background service thread (e.g., GC, JIT compiler). When all user (non-daemon) threads finish, the JVM **exits immediately**, killing all daemon threads — their `finally` blocks may not run. Set with `setDaemon(true)` before calling `start()`.

---

**Q7 🟡** What is thread priority? Is it guaranteed to be respected by the JVM?

> **Answer:** Thread priority (1–10, default 5) is a **hint** to the OS scheduler. The JVM maps it to OS-level priorities, but the OS may ignore or re-interpret them. Priority-based scheduling is **not guaranteed** in Java — never rely on it for program correctness.

---

**Q8 🟡** What is the difference between `notify()` and `notifyAll()`?

> **Answer:** `notify()` wakes **one arbitrary** thread from the wait set. `notifyAll()` wakes **all** waiting threads, who then compete for the lock. Use `notifyAll()` when threads wait on different conditions (e.g., producers and consumers share one monitor) to prevent missing a relevant wake-up. Use `notify()` only when all waiting threads are identical and interchangeable.

---

**Q9 🟡** Why must `wait()` always be called inside a `while` loop instead of an `if`?

> **Answer:** Due to **spurious wakeups** — a thread can wake from `wait()` without being explicitly notified (allowed by POSIX thread standard). Using `while` re-checks the condition after every wakeup, ensuring the thread only proceeds when the condition is truly satisfied. Using `if` can cause the thread to proceed on a spurious wakeup when the condition is still false.

---

**Q10 🟡** What is `Thread.join()`? What exception does it throw and why?

> **Answer:** `join()` causes the calling thread to block until the target thread completes (or a timeout expires). It throws `InterruptedException` because the calling thread could be interrupted while waiting, and all blocking Java operations must be interruptible to support clean shutdown patterns.

---

**Q11 🟡** What is `ThreadLocal`? Give a real-world use case.

> **Answer:** `ThreadLocal<T>` provides **per-thread storage** — each thread gets its own independent copy of the variable, invisible to other threads. Real-world use case: storing the current HTTP request's `UserContext` or `SecurityContext` in a web server so service-layer code can access it without the context being passed as a parameter through every method call.

---

**Q12 🟡** How does `ThreadLocal` cause memory leaks in thread pools? How do you prevent it?

> **Answer:** Thread pool threads are **never destroyed** — they're reused for new requests. If a request sets a `ThreadLocal` value but never calls `remove()`, the value remains associated with that thread and is visible to the next request handled by the same thread. Prevention: always call `threadLocal.remove()` in a `finally` block at the end of the request/task lifecycle (e.g., in a Servlet filter's finally block).

---

**Q13 🔴** Explain the happens-before guarantee in the Java Memory Model. Why is it important?

> **Answer:** The JMM defines **happens-before (HB)** — if action A happens-before action B, then the result of A is guaranteed to be **visible** to B. Without HB, CPUs and the JIT compiler can reorder instructions and cache values, making writes in one thread invisible to another. Key HB rules: unlock HB next lock on same monitor; volatile write HB next read; `Thread.start()` HB started thread's actions; `Thread.join()` HB caller after join returns. Correct concurrent code must establish HB relationships between conflicting accesses.

---

**Q14 🔴** What is a spurious wakeup? How does the JVM allow it and how do you defend against it?

> **Answer:** A spurious wakeup is when a thread wakes from `Object.wait()` or `LockSupport.park()` **without being explicitly notified**. POSIX thread standards allow OS implementations to spuriously wake blocked threads for efficiency reasons. The JVM exposes this behavior to developers rather than masking it. Defense: always wrap `wait()` in a `while (conditionNotMet()) { wait(); }` loop — re-check the condition on every wakeup.

---

**Q15 🔴** What is the difference between `Thread.interrupted()` and `Thread.currentThread().isInterrupted()`?

> **Answer:** `Thread.interrupted()` is a **static method** that returns the interrupt status of the current thread **and clears it**. `isInterrupted()` is an **instance method** that returns the flag **without clearing it**. Using `Thread.interrupted()` in a loop condition inadvertently clears the flag, potentially causing the interrupted signal to be lost. For checking only (without side effects), always use `isInterrupted()`.

---

## Synchronization & Locking (Q16–Q35)

---

**Q16 🟡** What is a race condition? Give a code example and fix.

> **Answer:** A race condition occurs when the outcome depends on the interleaving of thread execution. Classic example: `count++` is not atomic (read, modify, write). Two threads read `count = 5` simultaneously and both write `6` — result should be `7`. Fix: `AtomicInteger.incrementAndGet()`, `synchronized` block, or `volatile` for single independent writes.

---

**Q17 🟡** What is the difference between object-level and class-level locking?

> **Answer:** `synchronized` on an instance method locks on `this` — each object instance has its own lock. `synchronized` on a `static` method locks on the `Class` object (`MyClass.class`) — shared across all instances. Mixing both for the same shared state is dangerous — they use **different locks** and provide no mutual exclusion between them.

---

**Q18 🟡** Can two threads execute different `synchronized` methods on the same object simultaneously?

> **Answer:** No. All `synchronized` instance methods on the same object share the **same intrinsic lock (`this`)**. Only one thread can hold it at a time, so only one thread can be inside any synchronized instance method on that object at a time. To allow concurrency between independent methods, use synchronized blocks with **separate lock objects**.

---

**Q19 🟡** What is reentrant locking? Why does Java support it?

> **Answer:** Reentrant locking allows a thread to **re-acquire a lock it already holds** without deadlocking itself. Java supports it because synchronized methods commonly call other synchronized methods on the same object (e.g., a public method calling a private synchronized helper). Without reentrancy, such calls would cause the thread to wait for a lock it will never release — instant deadlock.

---

**Q20 🟡** What are the key advantages of `ReentrantLock` over `synchronized`?

> **Answer:** `ReentrantLock` provides: `tryLock()` (non-blocking attempt), `tryLock(timeout)` (timed acquisition), `lockInterruptibly()` (interruptible wait), optional **fairness** (FIFO ordering to prevent starvation), `newCondition()` for multiple wait sets per lock, and `getQueueLength()` for monitoring. `synchronized` provides none of these.

---

**Q21 🟡** Why must `ReentrantLock.unlock()` always be in a `finally` block?

> **Answer:** If an exception occurs between `lock()` and `unlock()`, the lock is **never released** — all threads waiting for it hang forever, effectively a deadlock. Unlike `synchronized` (where the JVM automatically releases the lock on exception), `ReentrantLock` is explicitly managed. The `finally` block guarantees `unlock()` runs regardless of exceptions.

---

**Q22 🟡** What is `ReadWriteLock`? When would you use it over a regular lock?

> **Answer:** `ReadWriteLock` allows **multiple concurrent readers** OR **one exclusive writer**, but not both simultaneously. Use it for data structures that are read far more often than written — configuration caches, product catalogs, static reference data. A regular lock would unnecessarily block readers from each other, degrading throughput for primarily read workloads.

---

**Q23 🟡** What is the difference between `ReentrantLock` with `fair=true` vs `fair=false`?

> **Answer:** **Fair (`true`)**: Threads acquire the lock in FIFO arrival order — no starvation but lower throughput due to queue management overhead. **Non-fair (`false`, default)**: The next thread may "barge" ahead of waiting threads — higher throughput but possible starvation for some threads. Use fair locking when starvation of lower-frequency threads is a real concern.

---

**Q24 🔴** What is `StampedLock`? How does optimistic reading work?

> **Answer:** `StampedLock` (Java 8) adds an **optimistic read mode** on top of read/write modes. `tryOptimisticRead()` returns a stamp **without acquiring any lock**. After reading, call `validate(stamp)` — if no write occurred during the read, the stamp is valid and the data is safe. If invalid, fall back to a real `readLock()`. This avoids lock acquisition entirely for successful reads in low-write scenarios, maximizing throughput.

---

**Q25 🔴** What is a `Semaphore`? How does it differ from a mutex?

> **Answer:** A `Semaphore` maintains a **count of permits** (N ≥ 1). `acquire()` decrements the count (blocks if 0); `release()` increments it. A **mutex** is a semaphore with count = 1 — only one thread at a time. Semaphores are used for resource pools (max 10 DB connections), rate limiting, and signaling. Unlike mutexes/locks, the releasing thread does **not need to be the same** as the acquiring thread.

---

**Q26 🔴** What is the ABA problem in CAS operations? How is it addressed in Java?

> **Answer:** Thread 1 reads value A. Thread 2 changes A→B→A. Thread 1's CAS succeeds (sees A as expected) even though the data was changed and restored. This can cause bugs in algorithms that rely on the value never having changed. Java addresses it with `AtomicStampedReference<V>` (value + integer stamp/version) and `AtomicMarkableReference<V>` (value + boolean mark) — CAS fails if either the value **or** the stamp has changed.

---

**Q27 🔴** What is lock striping? How does `ConcurrentHashMap` use it?

> **Answer:** Lock striping divides a data structure into independent segments, each with its own lock, reducing contention. Java 7's `ConcurrentHashMap` used 16 `ReentrantLock` segments. Java 8+ refined to **node-level locking** — `synchronized` only on the specific hash bucket's head node, plus CAS for insertions. This allows near-independent concurrent access for threads operating on different keys.

---

**Q28 🔴** Explain the difference between intrinsic locks (`synchronized`) and explicit locks (`ReentrantLock`) in terms of JVM implementation.

> **Answer:** Intrinsic locks use `monitorenter`/`monitorexit` JVM bytecodes and are stored in the object's **mark word** (header). The JVM applies biased locking, thin locks, and inflated (OS mutex) progressively. `ReentrantLock` is implemented **entirely in Java** on top of `AbstractQueuedSynchronizer (AQS)` — a CLH spin-then-block queue using CAS on an `int` state field. AQS provides more flexibility but doesn't benefit from JVM's monitorenter-specific optimizations.

---

**Q29 🔴** What is `AbstractQueuedSynchronizer (AQS)`? Which classes are built on it?

> **Answer:** AQS is the internal framework for building synchronization primitives. It maintains an `int` **state** and a **FIFO queue of waiting threads**. Subclasses define `tryAcquire()`/`tryRelease()` semantics using the state via CAS. Classes built on AQS include: `ReentrantLock`, `Semaphore`, `CountDownLatch`, `ReentrantReadWriteLock`, `FutureTask`, and `ThreadPoolExecutor` (for worker state management).

---

**Q30 🔴** Can `synchronized(null)` cause a deadlock?

> **Answer:** No — `synchronized(null)` throws `NullPointerException` immediately. It cannot cause a deadlock. However, it can crash the thread handling a request, potentially leaving shared state in an inconsistent state if the NPE is not properly caught. Always verify lock objects are non-null before synchronizing.

---

**Q31 🔴** What is biased locking and how does it optimize single-threaded synchronized code?

> **Answer:** Biased locking (JVM optimization) assumes that if a monitor is always acquired by the same thread, it's likely to remain uncontested. The JVM records the **thread ID** in the object header. Subsequent acquisitions by the same thread require **no CAS or synchronization** — just a pointer comparison. If a different thread requests the lock, bias is **revoked** (requires a STW pause). Deprecated in Java 15, disabled by default in Java 21 due to high revocation cost in modern applications.

---

**Q32 🔴** How would you implement a thread-safe Singleton without synchronizing `getInstance()`?

> **Answer:** Use the **Initialization-on-Demand Holder** (Bill Pugh) idiom:
```java
public class Singleton {
    private Singleton() {}
    private static class Holder {
        static final Singleton INSTANCE = new Singleton();
    }
    public static Singleton getInstance() {
        return Holder.INSTANCE; // JVM class loading is inherently thread-safe
    }
}
```
The `Holder` class is only loaded when `getInstance()` is first called. Class initialization is guaranteed by the JVM to be thread-safe — no explicit synchronization needed.

---

**Q33 🔴** What is `LockSupport.park()` and `LockSupport.unpark()`? How do they differ from `wait()`/`notify()`?

> **Answer:** `LockSupport` is the low-level primitive used by AQS. `park()` blocks the current thread; `unpark(thread)` unblocks a specific thread. Key differences from `wait()`/`notify()`: (1) No monitor ownership required. (2) `unpark()` can be called **before** `park()` — the next `park()` returns immediately (permit-based). (3) `unpark(thread)` targets a **specific thread**, not an arbitrary one from a set. (4) No `IllegalMonitorStateException` risk.

---

**Q34 🔴** What is spin-waiting (busy-waiting)? When is it appropriate vs. harmful?

> **Answer:** Spin-waiting has a thread loop and repeatedly check a condition without yielding. **Appropriate when**: lock hold time is very short (nanoseconds) and OS context switch cost (~10μs) would dominate. **Harmful when**: wait time is long — wastes CPU cycles, prevents other threads from using the core, can cause cache invalidation storms. The JVM automatically spins briefly before inflating a contended lock from thin (CAS) to fat (OS mutex).

---

**Q35 🔴** Explain "false sharing" in CPU caches and how it affects concurrent Java code.

> **Answer:** CPUs load memory in **cache lines** (~64 bytes). If two threads write to different variables that share the same cache line, each write **invalidates the other CPU's entire cache line** — forcing a full reload even though they're touching different variables. This can degrade performance by 10–100x on write-heavy workloads. Java 8+ fix: `@Contended` annotation (requires `-XX:-RestrictContended`) pads the field to its own cache line. Manual fix: add `long[] padding = new long[7]` between hot fields.

---

## Java Memory Model & Visibility (Q36–Q45)

---

**Q36 🟡** What does `volatile` guarantee and what does it NOT guarantee?

> **Answer:** **Guarantees**: visibility (writes immediately flushed to main memory, reads always from main memory) and ordering (prevents reordering across the volatile access — acts as memory fence). **Does NOT guarantee**: atomicity for compound operations. `volatile counter++` is still three non-atomic steps (read, increment, write) and can be interleaved by another thread.

---

**Q37 🟡** Can `volatile` replace `synchronized`? When is it sufficient?

> **Answer:** `volatile` is sufficient only when: a single thread writes and others only read, or updates are completely independent of the current value (e.g., `flag = true`). It is **NOT sufficient** for: check-then-act, read-modify-write (`count++`), or coordinating sequences of multiple reads/writes as a unit. For compound operations, use `synchronized` or `AtomicX`.

---

**Q38 🟡** What is the difference between `AtomicInteger` and `volatile int`?

> **Answer:** Both guarantee **visibility**. `AtomicInteger` additionally provides **atomic compound operations** — `incrementAndGet()`, `compareAndSet()`, `getAndUpdate()` — via hardware CAS instructions. `volatile int` makes individual reads/writes visible but compound operations like `++` remain non-atomic (three separate non-atomic steps).

---

**Q39 🔴** Why are `long` and `double` writes not guaranteed atomic on 32-bit JVMs?

> **Answer:** `long` and `double` are 64-bit. On a 32-bit JVM, writing a 64-bit value requires **two separate 32-bit operations**. A thread can read the high 32 bits from one write and the low 32 bits from another write — a **torn value** that was never actually stored. Declaring `volatile long` or using `synchronized` provides 64-bit atomicity. 64-bit JVMs provide native 64-bit atomic writes.

---

**Q40 🔴** What is instruction reordering and how can it break concurrent code even with atomic operations?

> **Answer:** CPUs and the JIT compiler reorder instructions for performance while maintaining **single-thread correctness**. Example: a reference can be published to a shared field **before the object's constructor finishes** (reference assigned before fields initialized). Another thread reading the reference may see an incompletely constructed object. `volatile` and `synchronized` insert **memory fences** that prevent reordering across them. This is the core reason double-checked locking requires `volatile`.

---

**Q41 🔴** Explain the "out-of-thin-air" safety guarantee in the JMM.

> **Answer:** The JMM guarantees that values read from a variable are always **values that were actually stored** — no completely fabricated or arbitrary values. This is a minimal baseline even without synchronization. However, it does **not** prevent reading **stale** values (old but previously written values). Out-of-thin-air safety prevents values like `-1` appearing in a counter that was only ever incremented from 0.

---

**Q42 🔴** What is a "data race" per the JMM? How does it differ from a logical "race condition"?

> **Answer:** A **data race** (JMM): two threads access the same variable concurrently, at least one is a write, and there is no happens-before between them. Data races give the JMM permission to return arbitrary values for the read. A **race condition** is a broader logical correctness bug — the program produces wrong results based on scheduling, even with proper HB (e.g., check-then-act patterns). All data races are bugs; not all race conditions are data races.

---

**Q43 🔴** How does the `final` keyword help with thread safety in object publication?

> **Answer:** The JMM guarantees that after a constructor completes, all threads will see the correctly initialized values of **`final` fields** without any synchronization — even if the object reference was shared unsafely (e.g., via a data race on the reference). This is achieved via a **freeze action** at end of constructor. Non-final fields still require proper synchronization for safe visibility after publication.

---

**Q44 🔴** What is "safe publication"? List 4 ways to safely publish an object.

> **Answer:** Safe publication ensures another thread sees the object in a **fully initialized** state. Four safe ways: (1) Store in a `static final` field (class initialization guarantee). (2) Store in a `volatile` field. (3) Store via properly `synchronized` write (HB to any subsequent read). (4) Store in a concurrent collection like `ConcurrentHashMap` (put HB get for the same key). Unsafe publication: storing in a plain field without synchronization — receiver may see partial construction.

---

**Q45 🔴** Why is this singleton broken in a multithreaded environment?
```java
static Config instance;
public static Config getInstance() {
    if (instance == null) instance = new Config();
    return instance;
}
```

> **Answer:** Three separate problems: (1) **Visibility** — writes to `instance` may not be flushed to main memory without `volatile` or `synchronized`. (2) **Atomicity** — two threads can simultaneously see `instance == null` and both create a `Config`. (3) **Partial construction** — due to reordering, `instance` can be assigned **before the constructor completes**, so another thread may receive a reference to an incompletely initialized `Config`. Fix: Holder pattern or `volatile` + double-checked locking.

---

## Executor Framework & CompletableFuture (Q46–Q60)

---

**Q46 🟡** What are the 4 built-in thread pool types from `Executors`? What is the danger of `newCachedThreadPool()`?

> **Answer:** `newFixedThreadPool(n)`, `newCachedThreadPool()`, `newSingleThreadExecutor()`, `newScheduledThreadPool(n)`. Danger of `newCachedThreadPool()`: uses an unbounded `SynchronousQueue` — under high load, creates **unlimited threads**, causing `OutOfMemoryError` or OS thread exhaustion. In production, prefer `ThreadPoolExecutor` with bounded queue and explicit `maximumPoolSize`.

---

**Q47 🟡** What is the difference between `shutdown()` and `shutdownNow()` on an `ExecutorService`?

> **Answer:** `shutdown()`: graceful — stops accepting new tasks; allows queued and running tasks to **complete normally**. `shutdownNow()`: forceful — attempts to stop running tasks by calling `interrupt()`, drains the queue, and returns a list of unstarted tasks. Best practice: call `shutdown()`, then `awaitTermination(timeout)`, then `shutdownNow()` if timeout expires.

---

**Q48 🟡** What is the difference between `submit()` and `execute()` on an executor?

> **Answer:** `execute(Runnable)` is fire-and-forget — returns void; exceptions go to `UncaughtExceptionHandler` and are easily lost. `submit(Callable/Runnable)` returns a `Future<T>` — allows result retrieval, status checking, cancellation, and exception handling via `future.get()` (wraps exceptions in `ExecutionException`).

---

**Q49 🟡** What is `CountDownLatch`? How is it different from `CyclicBarrier`?

> **Answer:** `CountDownLatch` is a **one-time gate** — any number of threads call `countDown()`; waiting threads release when count hits 0. It **cannot be reset**. `CyclicBarrier` requires exactly N threads to all call `await()` simultaneously before any proceed — it **auto-resets** for reuse. `CountDownLatch` suits "wait for N events"; `CyclicBarrier` suits "all N workers rendezvous at a checkpoint".

---

**Q50 🟡** What does `Future.get()` throw? How do you properly handle its exceptions?

> **Answer:** `Future.get()` throws: `ExecutionException` (wrapping the `Callable`'s exception), `InterruptedException` (calling thread interrupted while waiting), `TimeoutException` (`get(timeout)` version only), and `CancellationException` (task was cancelled). Proper handling: catch `ExecutionException` and extract with `.getCause()`; catch `InterruptedException` and restore the interrupt flag with `Thread.currentThread().interrupt()`.

---

**Q51 🟡** What is the difference between `thenApply()` and `thenCompose()` in `CompletableFuture`?

> **Answer:** `thenApply(T → U)` applies a **synchronous** function returning `U` — like `Stream.map()`. `thenCompose(T → CompletableFuture<U>)` is for **asynchronous** functions returning a `CompletableFuture` — like `Stream.flatMap()`, it flattens the nested future. If you used `thenApply` with an async function, you'd get `CompletableFuture<CompletableFuture<U>>` (double-wrapped). Use `thenCompose` to avoid nesting.

---

**Q52 🟡** What is `CompletableFuture.allOf()` vs `anyOf()`?

> **Answer:** `allOf(futures...)` returns a `CompletableFuture<Void>` that completes when **all** input futures complete (complete or exceptionally). `anyOf(futures...)` returns a `CompletableFuture<Object>` that completes with the result of **whichever future finishes first**. Use `allOf` to parallel-fetch and merge; use `anyOf` for redundant requests where the first response wins.

---

**Q53 🔴** What is the difference between `thenApply()` and `thenApplyAsync()`?

> **Answer:** `thenApply(fn)` executes in the **same thread** that completed the previous stage (or the calling thread if already done). `thenApplyAsync(fn)` submits to `ForkJoinPool.commonPool()` (or a provided executor). Use `thenApplyAsync` for long or blocking work — otherwise you may block a Netty I/O thread or ForkJoin worker, stalling the entire pipeline.

---

**Q54 🔴** What is the difference between `exceptionally()` and `handle()` in `CompletableFuture`?

> **Answer:** `exceptionally(fn)` runs **only on failure** — receives the exception and returns a recovery value; no-op on success. `handle(biFunction)` runs **always** — receives both result and exception (one will be null), can transform success or recover from failure in a single step. Use `handle` when you need to act on both outcomes; `exceptionally` for pure error recovery.

---

**Q55 🔴** What are the 4 rejection policies for `ThreadPoolExecutor`? Which is safest for critical tasks?

> **Answer:** `AbortPolicy` (default): throws `RejectedExecutionException`. `CallerRunsPolicy`: the **submitting thread** runs the task — provides back-pressure, never loses a task. `DiscardPolicy`: silently drops the task. `DiscardOldestPolicy`: drops the oldest queued task and retries. **Safest for critical tasks**: `CallerRunsPolicy` — no task is ever lost and it naturally slows down producers when the pool is overwhelmed.

---

**Q56 🔴** How does `ForkJoinPool` work-stealing work? Why is it better than a regular pool for recursive tasks?

> **Answer:** Each `ForkJoinPool` worker has its own **double-ended deque (deque)**. Threads push/pop from the **front** (LIFO) — cache-friendly for their own tasks. Idle threads **steal from the back** (FIFO) of other threads' deques — minimizing contention at the steal point. This keeps all threads busy when work is uneven. A regular shared queue creates a bottleneck at the queue; deque-per-thread with stealing is far more scalable for divide-and-conquer workloads.

---

**Q57 🔴** What happens to a `scheduleAtFixedRate` task that takes longer than its period?

> **Answer:** The task is **not run concurrently** with itself — the pool waits for the task to finish, then immediately starts the next execution (no delay). The schedule effectively becomes "as fast as possible" with no gap between runs. If this is unwanted, use `scheduleWithFixedDelay` which enforces a gap between the **end of one run** and the **start of the next**.

---

**Q58 🔴** What is `Phaser`? How does it improve on `CountDownLatch` and `CyclicBarrier`?

> **Answer:** `Phaser` is the most flexible synchronization barrier. It supports: **dynamic party registration** (`register()` / `arriveAndDeregister()` at runtime), **multiple automatic phases** (advances through phase 0, 1, 2... automatically), and a **tree structure** for scalability. `CountDownLatch` is single-use with a fixed count; `CyclicBarrier` requires a fixed party count at construction. `Phaser` handles dynamic membership and multi-phase workflows neither can support.

---

**Q59 🔴** What is `Exchanger`? Describe a use case.

> **Answer:** `Exchanger<V>` allows exactly **two threads to swap objects** at a synchronization point — each calls `exchange(myObject)` and receives the other's object. Use case: **double-buffering** I/O — Thread A fills a write buffer while Thread B drains a read buffer; they exchange when both are ready. Both threads stay productive with minimal synchronization overhead compared to a shared queue.

---

**Q60 🔴** How would you implement a thread-safe, bounded object pool using `Semaphore`?

```java
public class ObjectPool<T> {
    private final Semaphore semaphore;
    private final ConcurrentLinkedQueue<T> pool;

    public ObjectPool(List<T> objects) {
        this.pool = new ConcurrentLinkedQueue<>(objects);
        this.semaphore = new Semaphore(objects.size(), true); // fair
    }

    public T borrow() throws InterruptedException {
        semaphore.acquire();   // Block if all objects are borrowed
        return pool.poll();    // Always non-null: semaphore ensures count matches pool size
    }

    public void returnObject(T obj) {
        pool.offer(obj);
        semaphore.release();   // Signal one more object available
    }
}
```

---

## Concurrent Collections (Q61–Q70)

---

**Q61 🟡** Why is `HashMap` not thread-safe? What can go wrong under concurrent access?

> **Answer:** `HashMap` is unsynchronized. Concurrent `put()` during a resize can cause an **infinite loop** (Java 6 — circular linked list) or **data loss** (Java 8+). Concurrent reads during writes can return `null` for existing keys. Two threads computing the same hash bucket can silently overwrite each other's entries. Use `ConcurrentHashMap`.

---

**Q62 🟡** What is the difference between `ConcurrentHashMap` and `Collections.synchronizedMap()`?

> **Answer:** `Collections.synchronizedMap()` wraps a `HashMap` with a **single mutex** — every operation serializes on one lock, creating a throughput bottleneck. `ConcurrentHashMap` uses **per-bucket CAS + synchronized** — multiple threads can operate on different buckets simultaneously. Also: `ConcurrentHashMap` provides atomic `compute`/`merge` operations; `synchronizedMap` requires external locking for compound operations.

---

**Q63 🟡** When would you use `CopyOnWriteArrayList` over `ArrayList`?

> **Answer:** `CopyOnWriteArrayList` is ideal when **reads vastly outnumber writes** and iteration must be safe without external locking (no `ConcurrentModificationException`). Use cases: event listener lists, observer patterns, plugin registries. Avoid when writes are frequent — each write creates a full O(n) array copy in both time and memory.

---

**Q64 🟡** What is `BlockingQueue`? What are the differences between `put()`, `offer()`, and `add()`?

> **Answer:** `BlockingQueue` is a thread-safe queue with optional blocking. `add(e)`: throws `IllegalStateException` if full. `offer(e)`: returns `false` if full (non-blocking). `offer(e, timeout, unit)`: waits up to timeout. `put(e)`: **blocks indefinitely** until space available. For producer-consumer, `put()`/`take()` provide the correct blocking semantics with `InterruptedException` support.

---

**Q65 🔴** How does `ConcurrentHashMap.compute()` work atomically? What is it useful for?

> **Answer:** `compute(key, BiFunction)` holds the **bucket lock** for the entire read-apply-write sequence — guaranteeing atomicity. The function receives the current value (or null if absent) and returns the new value (null removes the key). Useful for: atomic counters (`(k,v) -> v == null ? 1 : v + 1`), accumulating values, lazy initialization. Cannot be replicated with separate `get()`/`put()` without external synchronization.

---

**Q66 🔴** What is `SynchronousQueue`? How is it used in the cached thread pool?

> **Answer:** `SynchronousQueue` is a **zero-capacity queue** — a `put()` blocks until another thread is ready to `take()`, and vice versa. It's a direct producer-to-consumer **handoff** with no buffering. `newCachedThreadPool()` uses it internally: when a task is submitted, it tries to hand off directly to an idle thread; if no idle thread exists, a new one is created immediately (no task buffering).

---

**Q67 🔴** What is `DelayQueue`? Give a real-world use case.

> **Answer:** `DelayQueue<E extends Delayed>` is a `BlockingQueue` where elements can only be dequeued after their **delay has expired**. `take()` blocks until the element with the smallest remaining delay becomes available. Real-world uses: retry queues (failed requests requeued with a 5-second delay), session expiry management, scheduled task execution, time-based cache eviction.

---

**Q68 🔴** What does `ConcurrentSkipListMap` offer over `ConcurrentHashMap`?

> **Answer:** `ConcurrentSkipListMap` is a **sorted, lock-free `NavigableMap`** backed by a skip list. It supports `firstKey()`, `lastKey()`, `headMap()`, `tailMap()`, `subMap()` — all thread-safe. `ConcurrentHashMap` is O(1) but **unordered**. Use `ConcurrentSkipListMap` for leaderboards, time-series data, range queries, or any use case requiring sorted concurrent access.

---

**Q69 🔴** How can `ConcurrentModificationException` occur with `Collections.synchronizedMap()` even with the wrapper?

> **Answer:** `synchronizedMap` wraps each individual method call in a synchronized block but does **not** synchronize multi-step compound operations. If you iterate a `synchronizedMap` in a for-each loop, the iteration involves multiple `hasNext()`/`next()` calls — not protected as a unit. Another thread modifying the map between calls triggers `CME`. Fix: manually synchronize the entire iteration: `synchronized(syncMap) { for (...) {} }`.

---

**Q70 🔴** Compare `ArrayBlockingQueue`, `LinkedBlockingQueue`, and `PriorityBlockingQueue`.

> **Answer:** `ArrayBlockingQueue`: **fixed capacity**, backed by array, single lock for head + tail — predictable memory, moderate throughput. `LinkedBlockingQueue`: **optionally bounded**, backed by linked nodes, **two separate locks** (head and tail) — higher throughput for producer-consumer. `PriorityBlockingQueue`: **unbounded**, elements dequeued in priority order — can grow until OOM; requires `Comparable` or `Comparator`. Best throughput: `LinkedBlockingQueue` (dual-lock). Memory control: `ArrayBlockingQueue`. Priority processing: `PriorityBlockingQueue`.

---

## Common Problems & Debugging (Q71–Q85)

---

**Q71 🟡** What is a deadlock? How do you detect it in a running JVM?

> **Answer:** Deadlock is a circular lock dependency where two or more threads wait forever. Detection: (1) `jstack <pid>` — look for "Found one Java-level deadlock:" section. (2) `jcmd <pid> Thread.print`. (3) `ThreadMXBean.findDeadlockedThreads()` programmatically. (4) JConsole "Detect Deadlock" button. (5) VisualVM Threads tab.

---

**Q72 🟡** What is thread starvation? How can `ReentrantLock`'s fairness setting help?

> **Answer:** Starvation is when a thread **never gets to run** because other threads always acquire the lock first. With non-fair `ReentrantLock`, a high-frequency thread can monopolize the lock, leaving a low-frequency thread waiting indefinitely. `new ReentrantLock(true)` (fair mode) enforces **FIFO lock acquisition** — every waiting thread is guaranteed to eventually get the lock, at the cost of some throughput.

---

**Q73 🟡** How do you diagnose `OutOfMemoryError: unable to create native thread`?

> **Answer:** Cause: OS cannot create more threads. Diagnosis: (1) Take a thread dump — check total thread count. (2) `ulimit -u` — check per-user thread limit. (3) Monitor thread count via JMX `ThreadCount`. Root causes: unbounded `newCachedThreadPool()`, thread leak (threads never terminated), stack too large (`-Xss` too high). Fix: bounded pools, reduce `-Xss256k`, fix leaking code.

---

**Q74 🟡** What is the risk of blocking inside `CompletableFuture.thenApply()`?

> **Answer:** `thenApply()` may run on the `ForkJoinPool.commonPool()`, which is shared JVM-wide for all `CompletableFuture` pipelines and parallel streams. Blocking I/O inside `thenApply()` consumes a `ForkJoin` thread indefinitely, **starving other async operations** across the entire application. Fix: use `thenApplyAsync(fn, dedicatedIOExecutor)` with a separate executor for any I/O or blocking work.

---

**Q75 🔴** How would you diagnose a Java application with 100% CPU usage?

> **Answer:** (1) `top -H -p <pid>` — identify the high-CPU OS thread ID (decimal). (2) Convert to hex: `printf '%x\n' <tid>`. (3) Take thread dump: `jstack <pid>`. (4) Find thread with matching `nid=0x<hex>`. (5) Inspect stack trace. (6) Take 3 dumps with 5-second gaps — same stack frame across all = genuinely stuck in infinite loop. Common causes: tight retry loop, non-terminating regex on pathological input, log message toString() in hot path.

---

**Q76 🔴** What is a "thread leak"? How does it manifest and how do you detect it?

> **Answer:** A thread leak is when threads are created and started but **never terminate** — count grows monotonically. Symptoms: growing thread count in monitoring, eventual `OutOfMemoryError: unable to create native thread`. Detection: JMX `ThreadCount` metric over time; periodic `jstack | grep "Thread" | wc -l`; VisualVM thread count graph. Root causes: `newCachedThreadPool()` without rate control, daemon threads with infinite loops that should terminate, creating raw threads per request.

---

**Q77 🔴** What is a "lock convoy" and how does it reduce throughput?

> **Answer:** A lock convoy forms when many threads repeatedly block on the **same lock**, forming a queue. Each lock release causes a context switch to wake the next thread. Even with very short critical sections, the **OS context switch overhead** (~10μs each) dominates. Overall throughput collapses. Solutions: narrow critical sections, use lock-free structures (`ConcurrentHashMap`, `AtomicX`), use `ReadWriteLock` for read-heavy access, use `StampedLock` optimistic reads.

---

**Q78 🔴** How do you use `Thread.setUncaughtExceptionHandler()` in production?

```java
// Global handler for unhandled exceptions in all threads
Thread.setDefaultUncaughtExceptionHandler((thread, throwable) -> {
    log.error("Uncaught exception in thread: {}", thread.getName(), throwable);
    metricsClient.increment("thread.uncaught.exception");
});

// Per-thread-pool via ThreadFactory
ThreadFactory factory = r -> {
    Thread t = new Thread(r);
    t.setUncaughtExceptionHandler((thread, ex) ->
        log.error("Worker thread {} crashed", thread.getName(), ex));
    return t;
};
```
> ⚠️ Handler is only called for tasks submitted via `execute()`. Tasks via `submit()` have their exceptions captured in the `Future` — handler is **not** invoked.

---

**Q79 🔴** What is the difference in exception handling for `submit(Callable)` vs `execute(Runnable)`?

> **Answer:** `submit(Callable)` wraps exceptions inside `ExecutionException` — accessible via `future.get().getCause()`. If you never call `get()`, the exception is **silently lost**. `execute(Runnable)` routes uncaught exceptions to the thread's `UncaughtExceptionHandler`. Key pitfall: fire-and-forget `submit()` without calling `get()` silently swallows all exceptions.

---

**Q80 🔴** Explain a scenario where `notifyAll()` causes a performance problem.

> **Answer:** In a producer-consumer setup using a single monitor (one synchronized object), `notifyAll()` wakes **both** producers and consumers. Most wake up only to re-check the condition and go back to `wait()` — wasted context switches and lock contention. Fix: use `ReentrantLock` with two `Condition` objects (`notFull` for producers, `notEmpty` for consumers) — `signal()` only the relevant group, eliminating the thundering herd entirely.

---

**Q81 🔴** What happens-before guarantees do `Thread.start()` and `Thread.join()` provide?

> **Answer:** **`start()`**: All actions in the calling thread **before** `t.start()` happen-before any action in thread `t`. The new thread sees all writes made by the parent before `start()` — no synchronization needed. **`join()`**: All actions in thread `t` happen-before `t.join()` returning. The caller sees everything `t` wrote without needing additional synchronization.

---

**Q82 🔴** Can `synchronized` instance methods on the same object deadlock within that single object?

> **Answer:** Not via reentrancy — all instance `synchronized` methods share one lock which is reentrant for the same thread. But inter-object deadlock is possible: Method A on `obj1` locks `obj1` then tries to lock `obj2`, while another thread in Method B on `obj2` holds `obj2`'s lock and tries to acquire `obj1`'s lock. Deadlock = circular dependency **across objects**, not within one.

---

**Q83 🔴** How do virtual threads (Java 21 Project Loom) change the Java threading model?

> **Answer:** Virtual threads are **lightweight, JVM-managed threads** — millions can exist simultaneously (vs. thousands of platform threads). When a virtual thread blocks on I/O, the JVM **unmounts it from its carrier (platform) thread**, freeing the carrier to run another virtual thread — no OS context switch. This eliminates the need for async/reactive programming for I/O-bound workloads: write simple blocking code, get async throughput. Create with `Thread.ofVirtual().start(task)` or `Executors.newVirtualThreadPerTaskExecutor()`.

---

**Q84 🔴** What is the risk of `ThreadLocal` with virtual threads (Java 21)?

> **Answer:** With millions of virtual threads, a `ThreadLocal` per thread means **millions of independent value copies** — significant heap pressure. Virtual thread `ThreadLocal` values are also not automatically cleaned up (unlike pooled platform threads that get `remove()` called). Java 21 introduces **`ScopedValue`** as the preferred alternative: immutable, scoped to a thread's execution and inherited by child threads, with no per-thread storage overhead.

---

**Q85 🔴** How do you implement a timeout for `CompletableFuture` in Java 9+?

```java
// orTimeout — completes exceptionally with TimeoutException
CompletableFuture<String> result = CompletableFuture
    .supplyAsync(() -> slowCall())
    .orTimeout(5, TimeUnit.SECONDS)
    .exceptionally(ex ->
        ex instanceof TimeoutException ? "timeout-fallback" : null
    );

// completeOnTimeout — completes with a default value (no exception)
CompletableFuture<String> result2 = CompletableFuture
    .supplyAsync(() -> slowCall())
    .completeOnTimeout("default-value", 5, TimeUnit.SECONDS);
```

---

## Advanced & Architecture (Q86–Q100)

---

**Q86 🔴** What is the difference between optimistic and pessimistic locking? Where does each fit?

> **Answer:** **Pessimistic**: assumes conflicts are likely — acquires lock before accessing data (e.g., `synchronized`, `ReentrantLock`). Best for high contention. **Optimistic**: assumes conflicts are rare — reads without locking, validates at write time (CAS / version check), retries on conflict (e.g., `AtomicInteger.compareAndSet()`). Best for low-contention, read-heavy workloads. Database analogy: `SELECT FOR UPDATE` (pessimistic) vs version column (optimistic).

---

**Q87 🔴** Explain the Producer-Consumer pattern. What is the optimal buffer size strategy?

> **Answer:** Producers add items to a bounded buffer; consumers remove and process them. If producer rate > consumer rate, buffer fills → producers block (back-pressure). Buffer sizing: too small → frequent blocking (low throughput); too large → high memory use + long queues (high latency). Optimal size depends on acceptable latency: `buffer_size ≈ production_rate × max_acceptable_latency`. Tune empirically under load. For burst handling: larger buffer; for strict latency SLAs: small buffer + more consumers.

---

**Q88 🔴** How would you design a thread-safe LRU cache for high concurrency?

> **Answer:** Use `ConcurrentHashMap<K, Node>` for O(1) lookup + a doubly linked list for LRU ordering, all protected by a `ReentrantReadWriteLock`. On `get()`: read lock, promote node to head. On `put()`: write lock, insert at head, evict tail if over capacity. For production: use **Caffeine** cache — uses `ConcurrentHashMap` + W-TinyLFU eviction with async cleanup via `StripedBuffer` (CAS-based, no global lock). Caffeine benchmarks ~10x faster than `LinkedHashMap` behind synchronization.

---

**Q89 🔴** How do you implement a token bucket `RateLimiter` using `ScheduledExecutorService`?

```java
public class TokenBucketRateLimiter {
    private final int capacity;
    private final AtomicInteger tokens;
    private final ScheduledExecutorService scheduler =
        Executors.newSingleThreadScheduledExecutor();

    public TokenBucketRateLimiter(int capacity, int refillPerSecond) {
        this.capacity = capacity;
        this.tokens = new AtomicInteger(capacity);
        scheduler.scheduleAtFixedRate(
            () -> tokens.updateAndGet(t -> Math.min(capacity, t + refillPerSecond)),
            1, 1, TimeUnit.SECONDS
        );
    }

    public boolean tryAcquire() {
        int current;
        do {
            current = tokens.get();
            if (current == 0) return false;
        } while (!tokens.compareAndSet(current, current - 1));
        return true;
    }
}
```

---

**Q90 🔴** What is the Reactor pattern and how does it relate to Java's NIO and `CompletableFuture`?

> **Answer:** The Reactor pattern uses a **single-threaded event loop** (e.g., Java NIO's `Selector`) to demultiplex I/O events and dispatch to handlers — no thread-per-connection overhead. `CompletableFuture` enables **chaining async callbacks** without blocking. Combined (e.g., Netty + CompletableFuture): the Netty event loop receives I/O, triggers `CompletableFuture` stages on separate executor pools — fully asynchronous pipeline where no thread ever blocks on network I/O.

---

**Q91 🔴** What are the dangers of `ForkJoinPool.commonPool()`?

> **Answer:** `commonPool()` is **JVM-wide shared** — used by default for `CompletableFuture.supplyAsync()` (no explicit executor), parallel streams, and `ForkJoin` tasks. Dangers: (1) Blocking I/O starves **all** async operations system-wide. (2) Pool size = `availableProcessors - 1` — may be too small. (3) `ThreadLocal` values from outer context don't transfer. Always provide a **dedicated executor** for I/O-bound `CompletableFuture` chains to isolate the workload.

---

**Q92 🔴** How do you implement thread-safe lazy initialization without `synchronized` on the getter?

> **Answer:** Initialization-on-Demand Holder idiom:
```java
public class HeavyService {
    private HeavyService() {}
    private static class Holder {
        static final HeavyService INSTANCE = new HeavyService();
    }
    public static HeavyService getInstance() {
        return Holder.INSTANCE; // Thread-safe: JVM class loading guarantee
    }
}
```
Alternative using `AtomicReference` + CAS for non-singleton lazy fields:
```java
private final AtomicReference<Config> ref = new AtomicReference<>();
public Config getConfig() {
    Config c = ref.get();
    if (c != null) return c;
    Config fresh = loadConfig();
    ref.compareAndSet(null, fresh); // Only first CAS wins; discard others
    return ref.get();
}
```

---

**Q93 🔴** What are structured concurrency and `ScopedValue` introduced in Java 21?

> **Answer:** **Structured concurrency** (JEP 453): treats concurrent tasks as a structured unit — if the enclosing scope exits, all child tasks are automatically cancelled. `StructuredTaskScope` ensures failures in subtasks propagate cleanly and all resources are released. Eliminates the "orphaned thread" problem in ad-hoc async code. **`ScopedValue`** (JEP 446): immutable alternative to `ThreadLocal` — a value is bound for a scope's duration and automatically unbound when the scope exits. Inherited by child threads without per-thread copies — ideal for passing request context to virtual threads at scale.

---

**Q94 🔴** How does Java handle the "thundering herd" problem in `notifyAll()`?

> **Answer:** `notifyAll()` wakes all waiting threads — they all contend for the lock, only one wins, the rest return to `BLOCKED`. This is the thundering herd: N threads wake up but N-1 immediately block again — N-1 unnecessary context switches. Java mitigation: (1) `ReentrantLock` with separate `Condition` objects — `signal()` only the relevant waiting group. (2) Use `notify()` only when all waiting threads are equivalent. (3) `BlockingQueue` implementations handle this internally with dual-condition design.

---

**Q95 🔴** What is the difference between `ConcurrentHashMap.size()` and `mappingCount()`?

> **Answer:** `size()` returns an `int` — overflows for maps with more than `Integer.MAX_VALUE` (~2.1 billion) entries. `mappingCount()` returns a `long` — accurate for arbitrarily large maps. Internally (Java 8+), the counter uses a `LongAdder`-like `CounterCell[]` array for high-concurrency counting, making both methods O(1) amortized. Use `mappingCount()` for large maps to avoid integer overflow surprises.

---

**Q96 🔴** Design a multi-producer, multi-consumer order processor ensuring at-most-once delivery.

> **Answer:** Use a bounded `ArrayBlockingQueue<Order>` between producers and consumers. Producers: `queue.put(order)` — blocks on full (back-pressure). Consumers: `queue.take()` — blocks on empty. **At-most-once**: order is removed from queue **before** processing — `take()` is the dequeue point; if consumer crashes mid-process, the order is lost (not requeued). Use `AtomicLong` for order ID generation. Separate `ThreadPoolExecutor` for producers and consumers. Drain remaining queue on `shutdown()` + `awaitTermination()`.

---

**Q97 🔴** What happens to daemon threads when `shutdownNow()` is called on their executor?

> **Answer:** `shutdownNow()` calls `interrupt()` on **all** pool threads (daemon or not). Threads that check `isInterrupted()` or call a blocking operation will see/throw `InterruptedException`. If task code ignores interruption (never checks `isInterrupted()`), it continues until the task finishes — `shutdownNow()` only **requests** interruption; it cannot force-terminate threads. Returns a list of unstarted tasks but cannot guarantee running tasks stop.

---

**Q98 🔴** How do you implement a distributed lock with Redis and what Java threading primitives support it locally?

> **Answer:** Redis distributed lock (Redisson): use `SET key uuid NX PX <ttl>` — atomic set-if-not-exists with expiry. Release: Lua script `if get(key)==uuid then del(key)`. Local JVM backing: `ReentrantLock` to avoid per-request Redis calls for threads on the same JVM contending the same key. Add a **watchdog thread** to extend TTL while work is in progress. Handle failures: TTL expires before task completes (extend via heartbeat); Redis node failure (Redlock algorithm across 5 nodes for fault tolerance).

---

**Q99 🔴** When using `CompletableFuture`, how do you ensure exceptions in the async chain are never silently lost?

> **Answer:** Four strategies: (1) Always call `future.get()` or `future.join()` somewhere — unchecked exceptions surface. (2) Attach `exceptionally()` or `handle()` to **every terminal stage**. (3) Use `whenComplete((result, ex) -> { if (ex != null) log.error(ex); })` as a safety net on every future. (4) Set a global completion handler for unhandled `CompletableFuture` exceptions in your framework (e.g., Spring's `AsyncUncaughtExceptionHandler`). The most dangerous pattern: fire-and-forget `supplyAsync()` with no `.get()` and no `.exceptionally()`.

---

**Q100 🔴** Compare `synchronized`, `ReentrantLock`, `StampedLock`, and `ConcurrentHashMap` for a high-throughput read/write shared map.

| Approach | Read Throughput | Write Throughput | Complexity | Correctness Risk |
|---|---|---|---|---|
| `synchronized HashMap` | Low (global lock) | Low | Simple | Low |
| `ReentrantLock` + `HashMap` | Low (single lock) | Low | Medium | Low |
| `ReadWriteLock` + `HashMap` | High (concurrent reads) | Medium | Medium | Medium |
| `StampedLock` + `HashMap` | Highest (optimistic) | High | High | High (validate stamp) |
| `ConcurrentHashMap` | Highest (no read lock) | High (bucket-level) | Low (built-in) | Low |

> **Recommendation:** Use `ConcurrentHashMap` — best balance of throughput, safety, and simplicity. Use `ReadWriteLock` only for custom data structures. Use `StampedLock` only if profiling proves `ReadWriteLock` is the specific bottleneck.

---

## Quick Cheat Sheet

```
Thread Creation:      Runnable > Thread extension | Callable for results
Visibility only:      volatile
Atomicity + Vis:      synchronized / AtomicX / ReentrantLock
Read-heavy map:       ConcurrentHashMap
Read-heavy list:      CopyOnWriteArrayList
Producer-Consumer:    ArrayBlockingQueue / LinkedBlockingQueue
One-time gate:        CountDownLatch
Reusable barrier:     CyclicBarrier
Dynamic barrier:      Phaser
Resource pool:        Semaphore
CPU-bound tasks:      ForkJoinPool / parallelStream
I/O-bound tasks:      Fixed ThreadPoolExecutor (dedicated pool)
Async pipelines:      CompletableFuture
Deadlock detection:   jstack / jcmd Thread.print
Memory leaks:         Heap dump + Eclipse MAT
High CPU:             jstack + top -H nid correlation
```

---

**Navigation**

[← Real-World Scenarios](10-real-world-scenarios.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md)

---

*Last updated: 2026 | Java 21 LTS*
