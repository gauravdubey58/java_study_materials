# 📑 Java Multithreading — Full Index

> Complete table of contents across all sections. Click any topic to jump directly to it.

---

[← Back to README](README.md)

---

## Sections Overview

| # | File | Topics Covered |
|---|---|---|
| 01 | [Thread Fundamentals](01-thread-fundamentals.md) | Process vs Thread, Lifecycle, Creating Threads, ThreadLocal |
| 02 | [Synchronization & Locking](02-synchronization-locking.md) | Race Conditions, synchronized, Intrinsic Locks |
| 03 | [Java Memory Model](03-java-memory-model.md) | JMM, Happens-Before, Volatile |
| 04 | [Atomic Classes & JUC Locks](04-atomic-concurrent-locks.md) | AtomicInteger, CAS, ReentrantLock, ReadWriteLock, StampedLock |
| 05 | [Wait, Notify & Condition](05-wait-notify-condition.md) | Object.wait/notify, Condition Variables, Producer-Consumer |
| 06 | [Executor Framework](06-executor-framework.md) | Thread Pools, ThreadPoolExecutor, ForkJoinPool, ScheduledExecutor |
| 07 | [Future & CompletableFuture](07-future-completablefuture.md) | Future, Callable, Async Pipelines, Error Handling |
| 08 | [Concurrent Collections](08-concurrent-collections.md) | ConcurrentHashMap, BlockingQueue, CopyOnWriteArrayList |
| 09 | [Exceptions & Issues](09-exceptions-issues.md) | Deadlock, Livelock, Starvation, Race Conditions, Exceptions |
| 10 | [Real-World Scenarios](10-real-world-scenarios.md) | 8 Production Patterns with Full Code |
| 11 | [Interview Questions](11-interview-questions.md) | 100 Q&A from Moderate to Complex |
| 12 | [Modern Locking Mechanisms](12-modern-locking-mechanisms.md) | ReentrantLock, ReadWriteLock, StampedLock, Semaphore, CountDownLatch, CyclicBarrier, Phaser, Exchanger |

---

## Detailed Index

### [01 — Thread Fundamentals](01-thread-fundamentals.md)
- [1.1 What is a Thread?](01-thread-fundamentals.md#11-what-is-a-thread)
- [1.2 Concurrency vs. Parallelism](01-thread-fundamentals.md#12-concurrency-vs-parallelism)
- [1.3 Thread Properties](01-thread-fundamentals.md#13-thread-properties)
- [1.4 Daemon vs. User Threads](01-thread-fundamentals.md#14-daemon-vs-user-threads)
- [2.1 Thread Lifecycle Diagram](01-thread-fundamentals.md#2-thread-lifecycle)
- [2.2 Thread State Transitions](01-thread-fundamentals.md#thread-state-transitions)
- [3.1 Extending Thread Class](01-thread-fundamentals.md#31-extending-thread-class)
- [3.2 Implementing Runnable](01-thread-fundamentals.md#32-implementing-runnable-preferred)
- [3.3 Implementing Callable](01-thread-fundamentals.md#33-implementing-callable-returns-result)
- [3.4 Thread Interruption](01-thread-fundamentals.md#34-thread-interruption)
- [3.5 Thread Join](01-thread-fundamentals.md#35-thread-join)
- [3.6 ThreadLocal](01-thread-fundamentals.md#36-threadlocal)

### [02 — Synchronization & Locking](02-synchronization-locking.md)
- [4.1 The Problem — Race Conditions](02-synchronization-locking.md#41-the-problem--race-conditions)
- [4.2 Synchronized Method](02-synchronization-locking.md#42-synchronized-method)
- [4.3 Synchronized Block](02-synchronization-locking.md#43-synchronized-block-finer-granularity)
- [4.4 Static Synchronized](02-synchronization-locking.md#44-static-synchronized)
- [4.5 Reentrant Locking](02-synchronization-locking.md#45-reentrant-locking)

### [03 — Java Memory Model](03-java-memory-model.md)
- [5.1 The Problem — Visibility & Ordering](03-java-memory-model.md#51-the-problem--visibility--ordering)
- [5.2 Happens-Before Relationship](03-java-memory-model.md#52-happens-before-relationship)
- [6.1 What Volatile Guarantees](03-java-memory-model.md#61-what-volatile-guarantees)
- [6.2 Volatile vs. Synchronized](03-java-memory-model.md#62-volatile-vs-synchronized)
- [6.3 Double-Checked Locking](03-java-memory-model.md#63-double-checked-locking-singleton-pattern)

### [04 — Atomic Classes & JUC Locks](04-atomic-concurrent-locks.md)
- [7.1 java.util.concurrent.atomic Package](04-atomic-concurrent-locks.md#71-javautilconcurrentatomic-package)
- [7.2 Thread-Safe Request Counter](04-atomic-concurrent-locks.md#72-real-world-thread-safe-request-counter)
- [7.3 AtomicReference — Lock-Free Stack](04-atomic-concurrent-locks.md#73-atomicreference--lock-free-stack)
- [7.4 LongAdder vs. AtomicLong](04-atomic-concurrent-locks.md#74-longadder-vs-atomiclong)
- [8.1 ReentrantLock](04-atomic-concurrent-locks.md#81-reentrantlock)
- [8.2 ReadWriteLock](04-atomic-concurrent-locks.md#82-readwritelock)
- [8.3 StampedLock](04-atomic-concurrent-locks.md#83-stampedlock-java-8)
- [8.4 Lock Comparison Table](04-atomic-concurrent-locks.md#84-lock-comparison)

### [05 — Wait, Notify & Condition](05-wait-notify-condition.md)
- [9.1 Object.wait() / notify() — Classic Producer-Consumer](05-wait-notify-condition.md#91-objectwait--notify--classic-producer-consumer)
- [9.2 Condition Variables with ReentrantLock](05-wait-notify-condition.md#92-condition-variables-with-reentrantlock)

### [06 — Executor Framework](06-executor-framework.md)
- [10.1 Why Executors?](06-executor-framework.md#101-why-executors)
- [10.2 Executor Hierarchy](06-executor-framework.md#102-executor-hierarchy)
- [10.3 Creating Thread Pools](06-executor-framework.md#103-creating-thread-pools)
- [10.4 ThreadPoolExecutor — Full Control](06-executor-framework.md#104-threadpoolexecutor--full-control)
- [10.5 Thread Pool Sizing Formula](06-executor-framework.md#105-thread-pool-sizing-formula)
- [10.6 Scheduled Executor](06-executor-framework.md#106-scheduled-executor)
- [10.7 ForkJoinPool — Divide and Conquer](06-executor-framework.md#107-forkjoinpool--divide-and-conquer)

### [07 — Future & CompletableFuture](07-future-completablefuture.md)
- [11.1 Future<T>](07-future-completablefuture.md#111-futuret)
- [11.2 CompletableFuture — Async Pipelines](07-future-completablefuture.md#112-completablefuture--async-pipelines)
- [11.3 Combining Futures](07-future-completablefuture.md#113-completablefuture--combining-futures)
- [11.4 Error Handling](07-future-completablefuture.md#114-completablefuture--error-handling)

### [08 — Concurrent Collections](08-concurrent-collections.md)
- [12.1 Overview Table](08-concurrent-collections.md#121-overview)
- [12.2 ConcurrentHashMap](08-concurrent-collections.md#122-concurrenthashmap)
- [12.3 BlockingQueue — Producer-Consumer](08-concurrent-collections.md#123-blockingqueue--producer-consumer)
- [12.4 CopyOnWriteArrayList](08-concurrent-collections.md#124-copyonwritearraylist)

### [09 — Exceptions & Issues](09-exceptions-issues.md)
- [13.1 Deadlock](09-exceptions-issues.md#131-deadlock)
- [13.2 Livelock](09-exceptions-issues.md#132-livelock)
- [13.3 Starvation](09-exceptions-issues.md#133-starvation)
- [13.4 Race Condition](09-exceptions-issues.md#134-race-condition)
- [13.5 Memory Consistency Errors](09-exceptions-issues.md#135-memory-consistency-errors)
- [13.6 Common Exceptions Table](09-exceptions-issues.md#136-common-exceptions)

### [10 — Real-World Scenarios](10-real-world-scenarios.md)
- [14.1 E-Commerce — Concurrent Inventory Management](10-real-world-scenarios.md#141-scenario-e-commerce--concurrent-inventory-management)
- [14.2 API Rate Limiter](10-real-world-scenarios.md#142-scenario-rate-limiter)
- [14.3 Parallel Data Processing Pipeline](10-real-world-scenarios.md#143-scenario-parallel-data-processing-pipeline)
- [14.4 Circuit Breaker Pattern](10-real-world-scenarios.md#144-scenario-circuit-breaker-pattern)
- [14.5 Thread-Safe Cache with Expiry](10-real-world-scenarios.md#145-scenario-thread-safe-cache-with-expiry)
- [14.6 CountDownLatch — Service Initialization](10-real-world-scenarios.md#146-scenario-countdown-latch--parallel-service-initialization)
- [14.7 CyclicBarrier — Batch Processing](10-real-world-scenarios.md#147-scenario-cyclicbarrier--parallel-batch-processing)
- [14.8 Phaser — Dynamic Task Coordination](10-real-world-scenarios.md#148-scenario-phaser--dynamic-task-coordination)

### [11 — 100 Interview Questions](11-interview-questions.md)
- [Q1–Q15: Core Threading Concepts](11-interview-questions.md#core-threading-concepts-q1q15)
- [Q16–Q35: Synchronization & Locking](11-interview-questions.md#synchronization--locking-q16q35)
- [Q36–Q45: Java Memory Model & Visibility](11-interview-questions.md#java-memory-model--visibility-q36q45)
- [Q46–Q60: Executor Framework & CompletableFuture](11-interview-questions.md#executor-framework--completablefuture-q46q60)
- [Q61–Q70: Concurrent Collections](11-interview-questions.md#concurrent-collections-q61q70)
- [Q71–Q85: Common Problems & Debugging](11-interview-questions.md#common-problems--debugging-q71q85)
- [Q86–Q100: Advanced & Architecture](11-interview-questions.md#advanced--architecture-q86q100)

---

[← Back to README](README.md) | [Start Reading → Thread Fundamentals](01-thread-fundamentals.md)

---

*Last updated: 2026 | Java 21 LTS*

### [12 — Modern Locking Mechanisms](12-modern-locking-mechanisms.md)
- [Overview — Why Modern Locks?](12-modern-locking-mechanisms.md#overview--why-modern-locks)
- [12.1 ReentrantLock](12-modern-locking-mechanisms.md#121-reentrantlock)
  - [Real-World: Flight Seat Booking System](12-modern-locking-mechanisms.md#real-world-example-1-flight-seat-booking-system)
  - [Real-World: Job Queue with Backoff](12-modern-locking-mechanisms.md#real-world-example-2-distributed-job-queue-with-backoff)
- [12.2 ReentrantReadWriteLock](12-modern-locking-mechanisms.md#122-reentrantreadwritelock)
  - [Real-World: Application Configuration Service](12-modern-locking-mechanisms.md#real-world-example-1-application-configuration-service)
  - [Real-World: In-Memory Session Store](12-modern-locking-mechanisms.md#real-world-example-2-in-memory-user-session-store)
- [12.3 StampedLock](12-modern-locking-mechanisms.md#123-stampedlock)
  - [Real-World: High-Frequency Trading Price Feed](12-modern-locking-mechanisms.md#real-world-example-1-high-frequency-trading-price-feed)
  - [Real-World: GPS Location Cache](12-modern-locking-mechanisms.md#real-world-example-2-gps-coordinate-cache-with-lock-conversion)
- [12.4 Semaphore](12-modern-locking-mechanisms.md#124-semaphore)
  - [Real-World: Database Connection Pool](12-modern-locking-mechanisms.md#real-world-example-1-database-connection-pool)
  - [Real-World: API Throttling](12-modern-locking-mechanisms.md#real-world-example-2-api-throttling--rate-limiting)
- [12.5 CountDownLatch](12-modern-locking-mechanisms.md#125-countdownlatch)
  - [Real-World: Microservice Smoke Test on Startup](12-modern-locking-mechanisms.md#real-world-example-microservice-smoke-test-on-startup)
- [12.6 CyclicBarrier](12-modern-locking-mechanisms.md#126-cyclicbarrier)
  - [Real-World: Monte Carlo Financial Simulation](12-modern-locking-mechanisms.md#real-world-example-monte-carlo-financial-simulation)
- [12.7 Phaser](12-modern-locking-mechanisms.md#127-phaser)
  - [Real-World: ETL Pipeline with Dynamic Workers](12-modern-locking-mechanisms.md#real-world-example-etl-pipeline-with-dynamic-worker-scaling)
- [12.8 Exchanger](12-modern-locking-mechanisms.md#128-exchanger)
  - [Real-World: Double-Buffered Log Writer](12-modern-locking-mechanisms.md#real-world-example-high-throughput-log-file-writer-double-buffering)
- [12.9 LockSupport](12-modern-locking-mechanisms.md#129-locksupport)
- [Comparison Table](12-modern-locking-mechanisms.md#comparison-table)
- [Choosing the Right Lock](12-modern-locking-mechanisms.md#choosing-the-right-lock)
