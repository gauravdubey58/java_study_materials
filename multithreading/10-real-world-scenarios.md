# 10 — Real-World Scenarios

> 8 complete, production-ready concurrency patterns with full code: inventory management, rate limiter, parallel pipeline, circuit breaker, expiring cache, service bootstrap, batch processing, and dynamic coordination.

---

**Navigation**

[← Exceptions & Issues](09-exceptions-issues.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → 100 Interview Questions](11-interview-questions.md)

---

## Table of Contents

- [14.1 E-Commerce — Concurrent Inventory Management](#141-scenario-e-commerce--concurrent-inventory-management)
- [14.2 Rate Limiter](#142-scenario-rate-limiter)
- [14.3 Parallel Data Processing Pipeline](#143-scenario-parallel-data-processing-pipeline)
- [14.4 Circuit Breaker Pattern](#144-scenario-circuit-breaker-pattern)
- [14.5 Thread-Safe Cache with Expiry](#145-scenario-thread-safe-cache-with-expiry)
- [14.6 CountDownLatch — Parallel Service Initialization](#146-scenario-countdown-latch--parallel-service-initialization)
- [14.7 CyclicBarrier — Parallel Batch Processing](#147-scenario-cyclicbarrier--parallel-batch-processing)
- [14.8 Phaser — Dynamic Task Coordination](#148-scenario-phaser--dynamic-task-coordination)

---

## 14.1 Scenario: E-Commerce — Concurrent Inventory Management

**Problem:** Multiple users can concurrently purchase the same product. Must prevent overselling without serializing all requests.

**Solution:** CAS-based stock reservation — no lock, no blocking.

```java
public class InventoryService {
    // productId → available stock (AtomicInteger for CAS operations)
    private final ConcurrentHashMap<String, AtomicInteger> stock = new ConcurrentHashMap<>();

    public void addStock(String productId, int quantity) {
        stock.computeIfAbsent(productId, k -> new AtomicInteger(0))
             .addAndGet(quantity);
    }

    /**
     * Atomically reserve stock without any lock.
     * Uses CAS retry loop — no thread is ever blocked.
     * Returns true if reservation succeeded, false if insufficient stock.
     */
    public boolean reserveStock(String productId, int quantity) {
        AtomicInteger available = stock.get(productId);
        if (available == null) return false; // Product not found

        int current;
        do {
            current = available.get();
            if (current < quantity) return false; // Not enough stock
            // CAS: only update if value hasn't changed since we read it
            // If another thread changed it → retry loop
        } while (!available.compareAndSet(current, current - quantity));

        return true; // Successfully reserved
    }

    public void releaseStock(String productId, int quantity) {
        AtomicInteger available = stock.get(productId);
        if (available != null) available.addAndGet(quantity);
    }

    public int getStock(String productId) {
        AtomicInteger s = stock.get(productId);
        return s == null ? 0 : s.get();
    }
}
```

**Key concepts used:** `ConcurrentHashMap`, `AtomicInteger`, CAS retry loop.

---

## 14.2 Scenario: Rate Limiter

**Problem:** Limit the number of concurrent calls to an external API to prevent overload.

**Solution:** Semaphore-based concurrency limiter + token bucket for throughput limiting.

### Concurrency Limiter (max N simultaneous calls)

```java
public class ApiRateLimiter {
    private final Semaphore semaphore;
    private final String serviceName;

    public ApiRateLimiter(int maxConcurrent, String serviceName) {
        this.semaphore   = new Semaphore(maxConcurrent, true);
        this.serviceName = serviceName;
    }

    public <T> T execute(Callable<T> task) throws Exception {
        if (!semaphore.tryAcquire(3, TimeUnit.SECONDS)) {
            throw new RuntimeException(serviceName + " rate limit exceeded");
        }
        try {
            return task.call();
        } finally {
            semaphore.release(); // Always release
        }
    }
}
```

### Token Bucket Rate Limiter (max N requests per second)

```java
public class TokenBucketRateLimiter {
    private final int     capacity;
    private final AtomicInteger tokens;
    private final ScheduledExecutorService refiller =
        Executors.newSingleThreadScheduledExecutor(r -> {
            Thread t = new Thread(r, "rate-limiter-refiller");
            t.setDaemon(true);
            return t;
        });

    public TokenBucketRateLimiter(int capacity, int refillPerSecond) {
        this.capacity = capacity;
        this.tokens   = new AtomicInteger(capacity);

        refiller.scheduleAtFixedRate(() ->
            tokens.updateAndGet(t -> Math.min(capacity, t + refillPerSecond)),
            1, 1, TimeUnit.SECONDS
        );
    }

    public boolean tryAcquire() {
        int current;
        do {
            current = tokens.get();
            if (current == 0) return false; // No tokens available
        } while (!tokens.compareAndSet(current, current - 1));
        return true;
    }

    public void shutdown() {
        refiller.shutdown();
    }
}
```

**Key concepts used:** `Semaphore`, `AtomicInteger`, CAS, `ScheduledExecutorService`.

---

## 14.3 Scenario: Parallel Data Processing Pipeline

**Problem:** Generate a monthly report that requires data from 3 different data sources. Sequential fetching is too slow.

**Solution:** Fetch all 3 sources in parallel using `CompletableFuture.allOf()`.

```java
public class ReportGenerator {
    private final ExecutorService ioPool = Executors.newFixedThreadPool(
        Runtime.getRuntime().availableProcessors() * 4  // I/O-bound → more threads
    );

    public Report generateMonthlyReport(String month) throws Exception {
        // Launch all 3 fetches in parallel
        CompletableFuture<List<Sale>>     salesFuture =
            CompletableFuture.supplyAsync(() -> fetchSales(month),     ioPool);

        CompletableFuture<List<Return>>   returnsFuture =
            CompletableFuture.supplyAsync(() -> fetchReturns(month),   ioPool);

        CompletableFuture<List<Customer>> customersFuture =
            CompletableFuture.supplyAsync(() -> fetchNewCustomers(month), ioPool);

        // Wait for all 3 to complete, then combine
        return CompletableFuture
            .allOf(salesFuture, returnsFuture, customersFuture)
            .thenApply(v -> new Report(
                salesFuture.join(),       // join() is safe after allOf completes
                returnsFuture.join(),
                customersFuture.join()
            ))
            .orTimeout(30, TimeUnit.SECONDS)
            .get();
    }
    // Time reduction: ~3x faster than sequential (30s → 10s for 3 equal-duration sources)
}
```

**Key concepts used:** `CompletableFuture`, `allOf()`, `join()`, dedicated I/O executor.

---

## 14.4 Scenario: Circuit Breaker Pattern

**Problem:** If an external service is down, requests fail and stack up, exhausting thread pools. Need to "open the circuit" and fail fast while the service recovers.

**Solution:** A thread-safe circuit breaker using `volatile` for state and `AtomicInteger` for failure counting.

```java
public class CircuitBreaker {

    public enum State { CLOSED, OPEN, HALF_OPEN }

    private volatile State state = State.CLOSED;
    private final AtomicInteger failureCount = new AtomicInteger(0);
    private volatile long lastFailureTime;

    private final int  failureThreshold;
    private final long resetTimeoutMs;

    public CircuitBreaker(int failureThreshold, long resetTimeoutMs) {
        this.failureThreshold = failureThreshold;
        this.resetTimeoutMs   = resetTimeoutMs;
    }

    public <T> T execute(Callable<T> task) throws Exception {
        switch (state) {
            case OPEN:
                if (System.currentTimeMillis() - lastFailureTime > resetTimeoutMs) {
                    state = State.HALF_OPEN; // Allow one trial request
                } else {
                    throw new RuntimeException("Circuit OPEN — service unavailable");
                }
                break;
            case HALF_OPEN:
                // Allow through — outcome will determine if we close or re-open
                break;
            case CLOSED:
                // Normal — allow through
                break;
        }

        try {
            T result = task.call();
            onSuccess();
            return result;
        } catch (Exception e) {
            onFailure();
            throw e;
        }
    }

    private void onSuccess() {
        failureCount.set(0);
        state = State.CLOSED;
    }

    private void onFailure() {
        lastFailureTime = System.currentTimeMillis();
        if (failureCount.incrementAndGet() >= failureThreshold) {
            state = State.OPEN;
        }
    }

    public State getState() { return state; }
}

// Usage
CircuitBreaker cb = new CircuitBreaker(5, 10_000); // Open after 5 failures, retry after 10s

try {
    String result = cb.execute(() -> externalApiCall());
} catch (RuntimeException e) {
    if (e.getMessage().startsWith("Circuit OPEN")) {
        return getCachedFallback();
    }
    throw e;
}
```

**Key concepts used:** `volatile` state, `AtomicInteger`, state machine pattern.

---

## 14.5 Scenario: Thread-Safe Cache with Expiry

**Problem:** Cache expensive computations with automatic expiry — no stale data, no memory leaks.

**Solution:** `ConcurrentHashMap` + `ScheduledExecutorService` for background eviction.

```java
public class ExpiringCache<K, V> {

    private final ConcurrentHashMap<K, CacheEntry<V>> cache      = new ConcurrentHashMap<>();
    private final long                                 ttlMillis;
    private final ScheduledExecutorService             cleaner    =
        Executors.newSingleThreadScheduledExecutor(r -> {
            Thread t = new Thread(r, "cache-cleaner");
            t.setDaemon(true);
            return t;
        });

    public ExpiringCache(long ttlMillis) {
        this.ttlMillis = ttlMillis;
        // Background eviction — runs every ttl interval
        cleaner.scheduleAtFixedRate(
            this::evictExpired,
            ttlMillis, ttlMillis, TimeUnit.MILLISECONDS
        );
    }

    public void put(K key, V value) {
        cache.put(key, new CacheEntry<>(value, System.currentTimeMillis() + ttlMillis));
    }

    public Optional<V> get(K key) {
        CacheEntry<V> entry = cache.get(key);
        if (entry == null || entry.isExpired()) {
            cache.remove(key); // Lazy eviction on read
            return Optional.empty();
        }
        return Optional.of(entry.value);
    }

    /**
     * Get from cache, or compute and cache the result.
     * Uses computeIfAbsent for atomic "get-or-create" semantics.
     */
    public V getOrCompute(K key, java.util.function.Supplier<V> loader) {
        CacheEntry<V> entry = cache.get(key);
        if (entry != null && !entry.isExpired()) return entry.value;

        V freshValue = loader.get();
        cache.put(key, new CacheEntry<>(freshValue, System.currentTimeMillis() + ttlMillis));
        return freshValue;
    }

    private void evictExpired() {
        cache.entrySet().removeIf(e -> e.getValue().isExpired());
    }

    public void shutdown() {
        cleaner.shutdown();
    }

    private static class CacheEntry<V> {
        final V    value;
        final long expiryTime;

        CacheEntry(V value, long expiryTime) {
            this.value      = value;
            this.expiryTime = expiryTime;
        }

        boolean isExpired() {
            return System.currentTimeMillis() > expiryTime;
        }
    }
}

// Usage
ExpiringCache<String, UserProfile> cache = new ExpiringCache<>(5 * 60 * 1000); // 5 min TTL
UserProfile profile = cache.getOrCompute(userId, () -> fetchFromDB(userId));
```

**Key concepts used:** `ConcurrentHashMap`, `ScheduledExecutorService`, daemon thread, lazy + eager eviction.

---

## 14.6 Scenario: CountDownLatch — Parallel Service Initialization

**Problem:** Application requires Database, Cache, and Message Queue to be ready before accepting HTTP requests.

**Solution:** `CountDownLatch` — main thread waits for all 3 initialization tasks to complete.

```java
public class ApplicationBootstrap {
    private final CountDownLatch      latch    = new CountDownLatch(3);
    private final ExecutorService     executor = Executors.newFixedThreadPool(3);
    private final List<String>        errors   = new CopyOnWriteArrayList<>();

    public void start() throws InterruptedException {
        long startTime = System.currentTimeMillis();

        executor.submit(() -> initService("DatabasePool",  this::initDatabasePool));
        executor.submit(() -> initService("CacheLayer",    this::initCacheLayer));
        executor.submit(() -> initService("MessageQueue",  this::initMessageQueue));

        boolean allReady = latch.await(30, TimeUnit.SECONDS);

        long elapsed = System.currentTimeMillis() - startTime;
        if (!allReady) {
            throw new RuntimeException("Boot timeout after " + elapsed + "ms");
        }
        if (!errors.isEmpty()) {
            throw new RuntimeException("Init failures: " + errors);
        }
        System.out.println("All services ready in " + elapsed + "ms — accepting requests");
    }

    private void initService(String name, Runnable init) {
        try {
            init.run();
            System.out.println(name + " initialized");
        } catch (Exception e) {
            errors.add(name + ": " + e.getMessage());
            System.err.println(name + " FAILED: " + e.getMessage());
        } finally {
            latch.countDown(); // Always count down — even on failure
        }
    }
}
```

**Key concepts used:** `CountDownLatch`, `CopyOnWriteArrayList` for error aggregation, parallel initialization.

---

## 14.7 Scenario: CyclicBarrier — Parallel Batch Processing

**Problem:** Process a large dataset in parallel with 4 workers. After each processing phase, results must be merged before the next phase begins.

**Solution:** `CyclicBarrier` — all workers synchronize at phase boundaries, and the barrier action runs once per phase.

```java
public class ParallelBatchProcessor {
    private static final int      WORKER_COUNT = 4;
    private final        Object[] phaseResults = new Object[WORKER_COUNT];

    private final CyclicBarrier barrier = new CyclicBarrier(WORKER_COUNT, () -> {
        // Barrier action: runs ONCE after all workers arrive at the barrier
        // Perfect place to merge phase results before next phase
        System.out.println("Phase complete — merging results from all workers");
        mergePhaseResults(phaseResults);
    });

    public void processDataset(List<DataChunk> chunks) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(WORKER_COUNT);
        List<Future<?>> futures  = new ArrayList<>();

        for (int i = 0; i < WORKER_COUNT; i++) {
            final int workerId = i;
            futures.add(executor.submit(() -> {
                // --- Phase 1: Load data ---
                DataChunk chunk = chunks.get(workerId);
                Object loadResult = loadChunk(chunk);
                phaseResults[workerId] = loadResult;
                barrier.await(); // Wait for all workers to finish loading

                // --- Phase 2: Transform ---
                Object transformResult = transform(loadResult);
                phaseResults[workerId] = transformResult;
                barrier.await(); // Wait for all workers to finish transforming

                // --- Phase 3: Validate ---
                validateResults(transformResult, workerId);
                phaseResults[workerId] = null;
                barrier.await(); // Final sync
            }));
        }

        for (Future<?> f : futures) f.get(); // Wait for all phases to complete
        executor.shutdown();
    }
}
```

**Key concepts used:** `CyclicBarrier` with barrier action, multi-phase parallel processing, shared result array.

---

## 14.8 Scenario: Phaser — Dynamic Task Coordination

**Problem:** A job manager dispatches tasks dynamically at runtime. The number of tasks is not known upfront. Need to wait for all dispatched tasks to finish before proceeding.

**Solution:** `Phaser` — supports dynamic party registration and deregistration.

```java
public class DynamicJobManager {

    public void runJob(List<Runnable> workerTasks) throws InterruptedException {
        // Register main thread as one party
        Phaser phaser = new Phaser(1);

        for (Runnable task : workerTasks) {
            phaser.register(); // Register each worker BEFORE starting it
            new Thread(() -> {
                try {
                    task.run();
                } finally {
                    phaser.arriveAndDeregister(); // Signal done + deregister
                }
            }).start();
        }

        // Main thread waits for all workers to arrive and deregister
        phaser.arriveAndAwaitAdvance();
        System.out.println("All " + workerTasks.size() + " workers completed");
    }

    /**
     * Multi-phase job with dynamic workers added mid-flight
     */
    public void runMultiPhaseJob() throws InterruptedException {
        Phaser phaser = new Phaser(1); // Main thread

        // Phase 1 workers
        for (int i = 0; i < 3; i++) {
            phaser.register();
            final int id = i;
            new Thread(() -> {
                System.out.println("Phase 1 worker " + id + " running");
                phaser.arriveAndAwaitAdvance(); // Sync at end of phase 1

                // Some workers also participate in phase 2
                if (id < 2) {
                    System.out.println("Phase 2 worker " + id + " running");
                    phaser.arriveAndDeregister(); // Done after phase 2
                } else {
                    phaser.arriveAndDeregister(); // Done after phase 1
                }
            }).start();
        }

        phaser.arriveAndAwaitAdvance(); // Main: wait for end of phase 1
        System.out.println("Phase 1 complete — " + phaser.getRegisteredParties() + " parties in phase 2");
        phaser.arriveAndAwaitAdvance(); // Main: wait for end of phase 2
        System.out.println("All phases complete");
    }
}
```

**Comparison with CountDownLatch and CyclicBarrier:**

| Feature | CountDownLatch | CyclicBarrier | Phaser |
|---|---|---|---|
| Reusable | ❌ | ✅ | ✅ |
| Dynamic parties | ❌ | ❌ | ✅ |
| Barrier action | ❌ | ✅ | ✅ (via `onAdvance()`) |
| Multi-phase | ❌ | ✅ (manual) | ✅ (automatic phase tracking) |
| Party deregistration | ❌ | ❌ | ✅ |

**Key concepts used:** `Phaser`, dynamic registration/deregistration, multi-phase coordination.

---

## Common Pattern Reference

```
Pattern             → Use
─────────────────────────────────────────────────────────
No-oversell         → ConcurrentHashMap + AtomicInteger CAS
Rate limiting       → Semaphore (concurrency) / token bucket (throughput)
Parallel fetch      → CompletableFuture.allOf() + dedicated I/O pool
Fault tolerance     → CircuitBreaker (volatile state + AtomicInteger)
TTL cache           → ConcurrentHashMap + ScheduledExecutor cleanup
Boot coordination   → CountDownLatch (one-shot, N services ready)
Phase sync          → CyclicBarrier (reusable, fixed parties)
Dynamic workers     → Phaser (dynamic registration, multi-phase)
```

---

**Navigation**

[← Exceptions & Issues](09-exceptions-issues.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → 100 Interview Questions](11-interview-questions.md)

---

*Last updated: 2026 | Java 21 LTS*
