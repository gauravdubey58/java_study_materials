# 08 — Concurrent Collections

> Covers: Overview of thread-safe collections, ConcurrentHashMap, BlockingQueue variants, CopyOnWriteArrayList, and synchronization barriers (CountDownLatch, CyclicBarrier, Semaphore, Phaser).

---

**Navigation**

[← Future & CompletableFuture](07-future-completablefuture.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Exceptions & Issues](09-exceptions-issues.md)

---

## Table of Contents

- [12.1 Overview](#121-overview)
- [12.2 ConcurrentHashMap](#122-concurrenthashmap)
- [12.3 BlockingQueue — Producer-Consumer](#123-blockingqueue--producer-consumer)
- [12.4 CopyOnWriteArrayList](#124-copyonwritearraylist)
- [12.5 Other Concurrent Collections](#125-other-concurrent-collections)
- [12.6 Synchronization Utilities](#126-synchronization-utilities)
- [Summary](#summary)

---

## 12.1 Overview

| Non-thread-safe | Thread-safe alternative | Notes |
|---|---|---|
| `HashMap` | `ConcurrentHashMap` | Segment/node-level locking; no null keys/values |
| `ArrayList` | `CopyOnWriteArrayList` | Copy on write; best for read-heavy, rare writes |
| `HashSet` | `CopyOnWriteArraySet` | Backed by `CopyOnWriteArrayList` |
| `LinkedList` | `ConcurrentLinkedQueue` | Lock-free FIFO; non-blocking |
| `PriorityQueue` | `PriorityBlockingQueue` | Blocking, unbounded, priority order |
| N/A | `ArrayBlockingQueue` | Blocking, bounded FIFO |
| N/A | `LinkedBlockingQueue` | Blocking, optionally bounded |
| `TreeMap` | `ConcurrentSkipListMap` | Sorted, lock-free, O(log n) |
| `TreeSet` | `ConcurrentSkipListSet` | Sorted set, lock-free |
| N/A | `SynchronousQueue` | Zero-capacity, direct thread-to-thread handoff |
| N/A | `DelayQueue` | Elements released after their delay expires |

> ⚠️ `Collections.synchronizedMap(new HashMap<>())` uses a **single lock** for all operations — much lower throughput than `ConcurrentHashMap`. Avoid in high-concurrency code.

---

## 12.2 ConcurrentHashMap

Java 8+ `ConcurrentHashMap` uses **CAS + per-bucket `synchronized`** — no global lock. Multiple threads can operate on different buckets simultaneously.

### Basic Operations

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

map.put("key", 1);
map.get("key");
map.remove("key");
map.containsKey("key");

// Null keys/values are NOT allowed (unlike HashMap)
// map.put(null, 1);  → NullPointerException
```

### Atomic Compound Operations

```java
// putIfAbsent — atomic check-then-put
map.putIfAbsent("pageViews", 0);

// computeIfAbsent — create only if key absent (lazy initialization)
map.computeIfAbsent("userId", k -> loadFromDatabase(k));

// computeIfPresent — update only if key exists
map.computeIfPresent("counter", (k, v) -> v + 1);

// compute — atomic read-modify-write
map.compute("hits", (k, v) -> v == null ? 1 : v + 1);

// merge — combine existing value with new value
map.merge("total", 10, Integer::sum);       // total = existing + 10
```

### Parallel Bulk Operations (Java 8+)

```java
// forEach in parallel (parallelismThreshold = minimum map size to use parallelism)
map.forEach(2, (k, v) -> processEntry(k, v));

// Parallel reduce
long totalRevenue = map.reduceValues(1, v -> (long) v, Long::sum);

// Parallel search — returns first non-null result
String found = map.search(2, (k, v) -> v > 1000 ? k : null);
```

### mappingCount() vs size()

```java
map.size();           // Returns int — capped at Integer.MAX_VALUE (2.1 billion)
map.mappingCount();   // Returns long — accurate for very large maps
```

### Real-World: Thread-Safe Word Frequency Counter

```java
public class WordFrequency {
    private final ConcurrentHashMap<String, LongAdder> counts = new ConcurrentHashMap<>();

    public void addWord(String word) {
        // computeIfAbsent is atomic — creates LongAdder if absent, then increments
        counts.computeIfAbsent(word, k -> new LongAdder()).increment();
    }

    public long getCount(String word) {
        LongAdder adder = counts.get(word);
        return adder == null ? 0 : adder.sum();
    }

    public Map<String, Long> getTopN(int n) {
        return counts.entrySet().stream()
            .collect(Collectors.toMap(Map.Entry::getKey, e -> e.getValue().sum()))
            .entrySet().stream()
            .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
            .limit(n)
            .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue,
                                      (a, b) -> a, LinkedHashMap::new));
    }
}
```

---

## 12.3 BlockingQueue — Producer-Consumer

`BlockingQueue` is a thread-safe queue that **blocks** producers when full and consumers when empty.

### Blocking Queue Operations

| Method | Queue Full | Queue Empty | Throws |
|---|---|---|---|
| `add(e)` | `IllegalStateException` | — | Yes |
| `offer(e)` | returns `false` | — | No |
| `offer(e, t, u)` | blocks up to timeout | — | `InterruptedException` |
| `put(e)` | **blocks** indefinitely | — | `InterruptedException` |
| `remove()` | — | `NoSuchElementException` | Yes |
| `poll()` | — | returns `null` | No |
| `poll(t, u)` | — | blocks up to timeout | `InterruptedException` |
| `take()` | — | **blocks** indefinitely | `InterruptedException` |
| `peek()` | — | returns `null` | No |

### Log Processing Pipeline

```java
public class AsyncLogPipeline {
    private final BlockingQueue<String> queue = new ArrayBlockingQueue<>(1000);
    private final ExecutorService consumer = Executors.newSingleThreadExecutor();

    public void start() {
        consumer.submit(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                try {
                    String logLine = queue.take();   // Blocks when empty
                    writeToFile(logLine);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    drainAndFlush();                  // Flush remaining before exit
                }
            }
        });
    }

    // Called from multiple threads
    public void log(String message) throws InterruptedException {
        queue.put(message);                          // Blocks when full (back-pressure)
    }

    // For non-blocking logging (drop if full)
    public boolean tryLog(String message) {
        return queue.offer(message);                 // Returns false if full
    }
}
```

### BlockingQueue Implementations

| Class | Capacity | Ordering | Notes |
|---|---|---|---|
| `ArrayBlockingQueue` | Bounded | FIFO | Single lock; predictable memory |
| `LinkedBlockingQueue` | Optional bound | FIFO | Dual lock (head + tail); higher throughput |
| `PriorityBlockingQueue` | Unbounded | Priority | Can grow until OOM; needs `Comparable` |
| `SynchronousQueue` | 0 (handoff only) | N/A | Direct producer→consumer handoff; no buffering |
| `DelayQueue` | Unbounded | Delay expiry | Elements only available after delay expires |

---

## 12.4 CopyOnWriteArrayList

On every **write operation**, `CopyOnWriteArrayList` creates a **fresh copy** of the underlying array. Readers always see a stable, immutable snapshot.

```java
CopyOnWriteArrayList<EventListener> listeners = new CopyOnWriteArrayList<>();

// Thread-safe iteration — even if another thread adds during iteration
// (iterating over a snapshot of the array taken at iterator creation time)
for (EventListener listener : listeners) {
    listener.onEvent(event);       // Safe — no ConcurrentModificationException
}

// Thread-safe modification
listeners.add(newListener);        // Copies entire array — O(n) time + memory
listeners.remove(oldListener);     // Also O(n) copy

// Bulk add is more efficient than individual adds
listeners.addAll(List.of(l1, l2, l3)); // One copy for all three
```

**When to use vs avoid:**

```
✅ Use CopyOnWriteArrayList when:
   - Many reads, rare writes (e.g., event listener registrations)
   - Iteration safety is critical (observers, plugins)
   - List size is small

❌ Avoid when:
   - High write frequency — each write is O(n) time and memory
   - Large list — copying 10,000 elements on each write is expensive
   - Prefer BlockingQueue or ConcurrentLinkedQueue for producer-consumer
```

---

## 12.5 Other Concurrent Collections

### ConcurrentLinkedQueue (Lock-Free FIFO)

```java
Queue<Task> taskQueue = new ConcurrentLinkedQueue<>();

taskQueue.offer(task);          // Non-blocking add (always succeeds — unbounded)
Task t = taskQueue.poll();      // Non-blocking remove (returns null if empty)
Task head = taskQueue.peek();   // Non-blocking peek

// Draining: process all current items
Task item;
while ((item = taskQueue.poll()) != null) {
    process(item);
}
```

### ConcurrentSkipListMap (Sorted + Thread-Safe)

```java
// Sorted by natural order, O(log n), no global lock
ConcurrentSkipListMap<Integer, String> leaderboard = new ConcurrentSkipListMap<>();

leaderboard.put(1500, "Alice");
leaderboard.put(2300, "Bob");
leaderboard.put(1800, "Charlie");

leaderboard.firstKey();         // 1500 (lowest score)
leaderboard.lastKey();          // 2300 (highest score)
leaderboard.headMap(2000);      // All entries with key < 2000
leaderboard.tailMap(1800);      // All entries with key >= 1800
leaderboard.subMap(1500, 2000); // Range query

// Use for leaderboards, range queries, time-series data
```

### DelayQueue

```java
class RetryTask implements Delayed {
    private final Runnable task;
    private final long executeAt;

    RetryTask(Runnable task, long delayMs) {
        this.task = task;
        this.executeAt = System.currentTimeMillis() + delayMs;
    }

    @Override
    public long getDelay(TimeUnit unit) {
        return unit.convert(executeAt - System.currentTimeMillis(), TimeUnit.MILLISECONDS);
    }

    @Override
    public int compareTo(Delayed other) {
        return Long.compare(getDelay(TimeUnit.MILLISECONDS),
                            other.getDelay(TimeUnit.MILLISECONDS));
    }
}

DelayQueue<RetryTask> retryQueue = new DelayQueue<>();
retryQueue.put(new RetryTask(() -> callApi(), 5000)); // Retry after 5 seconds

// Consumer — blocks until an element's delay has expired
RetryTask readyTask = retryQueue.take(); // Blocks until delay passes
readyTask.task.run();
```

---

## 12.6 Synchronization Utilities

### CountDownLatch — One-Time Gate

```java
// Wait for N events to happen before proceeding
CountDownLatch latch = new CountDownLatch(3); // Count = 3

// 3 threads each count down
executor.submit(() -> { initDatabase();  latch.countDown(); });
executor.submit(() -> { initCache();     latch.countDown(); });
executor.submit(() -> { initMessaging(); latch.countDown(); });

// Main thread waits until all 3 have counted down
boolean allReady = latch.await(30, TimeUnit.SECONDS);
if (!allReady) throw new RuntimeException("Init timed out");
System.out.println("All services ready — accepting requests");
```

### CyclicBarrier — Reusable Rendezvous Point

```java
// N threads each wait at a barrier until ALL arrive, then all proceed
CyclicBarrier barrier = new CyclicBarrier(4, () ->
    System.out.println("All workers finished phase — merging results")
);

for (int i = 0; i < 4; i++) {
    executor.submit(() -> {
        processChunk(data);
        barrier.await(); // Wait for all 4 threads to reach here
        // Barrier action runs (once), then all 4 continue

        validateResults();
        barrier.await(); // Reuse: barrier resets automatically
    });
}
```

| | `CountDownLatch` | `CyclicBarrier` |
|---|---|---|
| Reusable | ❌ No | ✅ Yes (auto-resets) |
| Who counts down | Any thread(s) | The waiting parties themselves |
| Barrier action | ❌ No | ✅ Yes (runs once all arrive) |

### Semaphore — Resource Pool / Rate Limiting

```java
// Allow max 10 concurrent database connections
Semaphore semaphore = new Semaphore(10, true); // fair=true → FIFO

public <T> T withConnection(Callable<T> work) throws Exception {
    semaphore.acquire();           // Decrement permit count (blocks if 0)
    try {
        return work.call();
    } finally {
        semaphore.release();       // Increment permit count — wake waiting thread
    }
}

// Try without blocking
if (semaphore.tryAcquire(500, TimeUnit.MILLISECONDS)) {
    try { doWork(); } finally { semaphore.release(); }
} else {
    return "Service busy — try later";
}
```

---

## Summary

| Collection | Locking | Order | Blocking | Best for |
|---|---|---|---|---|
| `ConcurrentHashMap` | Per-bucket CAS | Unordered | No | High-concurrency map operations |
| `CopyOnWriteArrayList` | Copy-on-write | Insertion | No | Read-heavy listener lists |
| `ArrayBlockingQueue` | Single lock | FIFO | Yes | Bounded producer-consumer |
| `LinkedBlockingQueue` | Dual lock | FIFO | Yes | High-throughput producer-consumer |
| `PriorityBlockingQueue` | Single lock | Priority | Yes | Priority task scheduling |
| `ConcurrentLinkedQueue` | Lock-free CAS | FIFO | No | Non-blocking task queue |
| `ConcurrentSkipListMap` | Lock-free CAS | Sorted | No | Sorted concurrent access |
| `DelayQueue` | Single lock | Delay expiry | Yes | Retry queues, scheduling |

---

**Navigation**

[← Future & CompletableFuture](07-future-completablefuture.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Exceptions & Issues](09-exceptions-issues.md)

---

*Last updated: 2026 | Java 21 LTS*
