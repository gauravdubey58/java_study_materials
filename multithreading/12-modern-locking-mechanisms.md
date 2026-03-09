# 12 — Modern Locking Mechanisms

> Deep dive into Java's modern locking toolkit: ReentrantLock, ReadWriteLock, StampedLock, Semaphore, CountDownLatch, CyclicBarrier, Phaser, and Exchanger — each with real-world production examples.

---

**Navigation**

[← Interview Questions](11-interview-questions.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md)

---

## Table of Contents

- [Overview — Why Modern Locks?](#overview--why-modern-locks)
- [12.1 ReentrantLock](#121-reentrantlock)
- [12.2 ReentrantReadWriteLock](#122-reentrantreadwritelock)
- [12.3 StampedLock](#123-stampedlock)
- [12.4 Semaphore](#124-semaphore)
- [12.5 CountDownLatch](#125-countdownlatch)
- [12.6 CyclicBarrier](#126-cyclicbarrier)
- [12.7 Phaser](#127-phaser)
- [12.8 Exchanger](#128-exchanger)
- [12.9 LockSupport](#129-locksupport)
- [Comparison Table](#comparison-table)
- [Choosing the Right Lock](#choosing-the-right-lock)

---

## Overview — Why Modern Locks?

Java's built-in `synchronized` keyword is simple and JVM-optimized, but it has hard limitations:

| Limitation | Impact |
|---|---|
| Cannot try to acquire without blocking | Thread stuck waiting indefinitely |
| Cannot time out while waiting | No way to avoid indefinite hangs |
| Cannot be interrupted while waiting | Clean shutdown is difficult |
| No fairness guarantee | Low-frequency threads may starve |
| Single wait set per object | `notifyAll()` wakes irrelevant threads |
| No concurrent reads | Read-heavy workloads serialized unnecessarily |

The `java.util.concurrent.locks` package (introduced Java 5, expanded in Java 8) solves all of these.

```
Lock Hierarchy (java.util.concurrent.locks):
  Lock (interface)
    └── ReentrantLock
  ReadWriteLock (interface)
    └── ReentrantReadWriteLock
          ├── ReentrantReadWriteLock.ReadLock
          └── ReentrantReadWriteLock.WriteLock
  StampedLock (Java 8 — does not implement Lock)
```

---

## 12.1 ReentrantLock

### What it is

`ReentrantLock` is a mutual exclusion lock with the same semantics as `synchronized` but with additional capabilities. A thread that already holds the lock can acquire it again (reentrant).

### Core API

```java
ReentrantLock lock = new ReentrantLock();
ReentrantLock fairLock = new ReentrantLock(true); // FIFO ordering

lock.lock();                                    // Block until acquired
lock.lockInterruptibly();                       // Block, but respond to interrupt
boolean acquired = lock.tryLock();             // Immediate: true/false
boolean acquired = lock.tryLock(2, SECONDS);  // Wait up to 2s
lock.unlock();                                  // Always in finally!

lock.isLocked();                               // Any thread holding it?
lock.isHeldByCurrentThread();                  // This thread holding it?
lock.getHoldCount();                           // Re-entry depth for current thread
lock.getQueueLength();                         // Estimate of waiting threads
lock.hasQueuedThread(thread);                  // Is this thread waiting?
```

### Always Unlock in Finally

```java
lock.lock();
try {
    // critical section
} finally {
    lock.unlock(); // Guaranteed even if exception thrown
}
```

---

### Real-World Example 1: Flight Seat Booking System

**Problem:** Airlines process thousands of simultaneous booking requests. A seat must be reserved atomically — no double-booking. If the booking service is busy, callers should fail fast rather than wait indefinitely.

```java
import java.util.concurrent.locks.ReentrantLock;
import java.util.*;
import java.util.concurrent.TimeUnit;

public class FlightBookingService {

    private final Map<String, Set<Integer>> availableSeats = new HashMap<>();
    private final Map<String, Set<Integer>> bookedSeats    = new HashMap<>();
    private final ReentrantLock lock = new ReentrantLock(true); // fair — no starvation

    public void loadFlight(String flightId, int totalSeats) {
        lock.lock();
        try {
            Set<Integer> seats = new HashSet<>();
            for (int i = 1; i <= totalSeats; i++) seats.add(i);
            availableSeats.put(flightId, seats);
            bookedSeats.put(flightId, new HashSet<>());
        } finally {
            lock.unlock();
        }
    }

    /**
     * Try to book a seat. Waits up to 3 seconds for the lock.
     * Returns the seat number, or -1 if unavailable or timeout.
     */
    public int bookNextAvailableSeat(String flightId, String passenger)
            throws InterruptedException {

        // Don't block indefinitely — timeout after 3 seconds
        if (!lock.tryLock(3, TimeUnit.SECONDS)) {
            System.out.println(passenger + ": booking service busy — try again");
            return -1;
        }
        try {
            Set<Integer> available = availableSeats.get(flightId);
            if (available == null || available.isEmpty()) {
                System.out.println(passenger + ": no seats left on " + flightId);
                return -1;
            }
            int seat = available.iterator().next();
            available.remove(seat);
            bookedSeats.get(flightId).add(seat);
            System.out.println(passenger + " booked seat " + seat + " on " + flightId);
            return seat;
        } finally {
            lock.unlock();
        }
    }

    /**
     * Cancel a booking — interruptible so shutdown hooks can cancel pending cancellations.
     */
    public boolean cancelBooking(String flightId, int seat, String passenger)
            throws InterruptedException {

        lock.lockInterruptibly(); // Responds to Thread.interrupt()
        try {
            Set<Integer> booked = bookedSeats.get(flightId);
            if (booked == null || !booked.contains(seat)) {
                System.out.println(passenger + ": seat " + seat + " not found in bookings");
                return false;
            }
            booked.remove(seat);
            availableSeats.get(flightId).add(seat);
            System.out.println(passenger + ": seat " + seat + " cancelled on " + flightId);
            return true;
        } finally {
            lock.unlock();
        }
    }

    public int availableSeatCount(String flightId) {
        lock.lock();
        try {
            Set<Integer> seats = availableSeats.get(flightId);
            return seats == null ? 0 : seats.size();
        } finally {
            lock.unlock();
        }
    }
}
```

---

### Real-World Example 2: Distributed Job Queue with Backoff

**Problem:** A job dispatcher must assign work to workers without duplication. Under high contention, workers should back off and retry rather than spin.

```java
public class JobDispatcher {
    private final Queue<Job>    pendingJobs  = new LinkedList<>();
    private final List<Job>     processingJobs = new ArrayList<>();
    private final ReentrantLock lock = new ReentrantLock();

    public void submitJob(Job job) {
        lock.lock();
        try {
            pendingJobs.offer(job);
        } finally {
            lock.unlock();
        }
    }

    /**
     * Claim the next available job with exponential backoff.
     * Returns null if no job becomes available within maxWaitMs.
     */
    public Job claimNextJob(long maxWaitMs) throws InterruptedException {
        long deadline = System.currentTimeMillis() + maxWaitMs;
        long backoff  = 10; // Start with 10ms backoff

        while (System.currentTimeMillis() < deadline) {
            long remaining = deadline - System.currentTimeMillis();
            if (remaining <= 0) break;

            if (lock.tryLock(Math.min(backoff, remaining), TimeUnit.MILLISECONDS)) {
                try {
                    if (!pendingJobs.isEmpty()) {
                        Job job = pendingJobs.poll();
                        processingJobs.add(job);
                        return job;
                    }
                } finally {
                    lock.unlock();
                }
            }
            // Exponential backoff — reduces lock contention
            backoff = Math.min(backoff * 2, 500);
            Thread.sleep(backoff);
        }
        return null; // Timed out
    }

    public void completeJob(Job job) {
        lock.lock();
        try {
            processingJobs.remove(job);
        } finally {
            lock.unlock();
        }
    }
}
```

---

## 12.2 ReentrantReadWriteLock

### What it is

Allows **multiple readers simultaneously** OR **one exclusive writer**. Dramatically improves throughput for read-heavy data structures where reads vastly outnumber writes.

```
State matrix:
               Read Lock    Write Lock
Read Lock:     ✅ Allowed   ❌ Blocked
Write Lock:    ❌ Blocked   ❌ Blocked
```

### Core API

```java
ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
ReentrantReadWriteLock.ReadLock  readLock  = rwLock.readLock();
ReentrantReadWriteLock.WriteLock writeLock = rwLock.writeLock();

// Read — allows concurrent readers
readLock.lock();
try { /* read data */ }
finally { readLock.unlock(); }

// Write — exclusive
writeLock.lock();
try { /* write data */ }
finally { writeLock.unlock(); }

// Downgrade: write → read (allowed)
writeLock.lock();
try {
    updateData();
    readLock.lock();   // Acquire read BEFORE releasing write
} finally {
    writeLock.unlock(); // Now downgraded to read lock
}
try {
    readUpdatedData();
} finally {
    readLock.unlock();
}

// Upgrade: read → write (NOT directly supported — must release read first)
```

> ⚠️ **Lock upgrade is not supported** in `ReentrantReadWriteLock`. You must release the read lock before acquiring the write lock. Between releasing and re-acquiring, another thread may modify the data — re-check your condition after acquiring the write lock.

---

### Real-World Example 1: Application Configuration Service

**Problem:** A configuration service is read thousands of times per second by every service in the system. Config is updated rarely (deployments, feature flags). Standard locking would serialize all reads unnecessarily.

```java
import java.util.concurrent.locks.ReentrantReadWriteLock;
import java.util.*;

public class ConfigurationService {

    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final ReentrantReadWriteLock.ReadLock  readLock  = rwLock.readLock();
    private final ReentrantReadWriteLock.WriteLock writeLock = rwLock.writeLock();

    private volatile Map<String, String> config = new HashMap<>();
    private volatile long lastUpdatedAt = 0;

    // ─── READ OPERATIONS (concurrent) ───────────────────────────────────────

    public String get(String key) {
        readLock.lock();
        try {
            return config.getOrDefault(key, null);
        } finally {
            readLock.unlock();
        }
    }

    public String get(String key, String defaultValue) {
        readLock.lock();
        try {
            return config.getOrDefault(key, defaultValue);
        } finally {
            readLock.unlock();
        }
    }

    public boolean isFeatureEnabled(String featureFlag) {
        readLock.lock();
        try {
            return "true".equalsIgnoreCase(config.get(featureFlag));
        } finally {
            readLock.unlock();
        }
    }

    public Map<String, String> getAll() {
        readLock.lock();
        try {
            return Collections.unmodifiableMap(new HashMap<>(config)); // Snapshot
        } finally {
            readLock.unlock();
        }
    }

    // ─── WRITE OPERATIONS (exclusive) ───────────────────────────────────────

    /**
     * Hot-reload entire config (e.g., from Consul/etcd on change event).
     * All readers are blocked only during this short swap.
     */
    public void reloadConfig(Map<String, String> newConfig) {
        writeLock.lock();
        try {
            this.config = new HashMap<>(newConfig);
            this.lastUpdatedAt = System.currentTimeMillis();
            System.out.println("Config reloaded: " + newConfig.size() + " entries");
        } finally {
            writeLock.unlock();
        }
    }

    public void set(String key, String value) {
        writeLock.lock();
        try {
            Map<String, String> updated = new HashMap<>(config);
            updated.put(key, value);
            this.config = updated;
            this.lastUpdatedAt = System.currentTimeMillis();
        } finally {
            writeLock.unlock();
        }
    }

    public void delete(String key) {
        writeLock.lock();
        try {
            Map<String, String> updated = new HashMap<>(config);
            updated.remove(key);
            this.config = updated;
        } finally {
            writeLock.unlock();
        }
    }

    public long getLastUpdatedAt() {
        readLock.lock();
        try { return lastUpdatedAt; }
        finally { readLock.unlock(); }
    }
}
```

---

### Real-World Example 2: In-Memory User Session Store

**Problem:** A session store handles millions of session reads (auth checks on every request) and rare writes (login, logout, session update).

```java
public class SessionStore {

    private static class Session {
        final String  userId;
        final String  token;
        final long    createdAt;
        volatile long lastAccessedAt;
        volatile boolean valid = true;

        Session(String userId, String token) {
            this.userId = userId;
            this.token  = token;
            this.createdAt = this.lastAccessedAt = System.currentTimeMillis();
        }
    }

    private final Map<String, Session>   sessions = new HashMap<>();
    private final ReentrantReadWriteLock rwLock   = new ReentrantReadWriteLock();
    private static final long SESSION_TTL_MS = 30 * 60 * 1000; // 30 minutes

    public String createSession(String userId) {
        String token = UUID.randomUUID().toString();
        rwLock.writeLock().lock();
        try {
            sessions.put(token, new Session(userId, token));
            return token;
        } finally {
            rwLock.writeLock().unlock();
        }
    }

    /**
     * Called on every authenticated request — must be fast and concurrent.
     * Uses read lock + volatile write for lastAccessedAt (no write lock needed for the touch).
     */
    public boolean validateSession(String token) {
        rwLock.readLock().lock();
        try {
            Session session = sessions.get(token);
            if (session == null || !session.valid) return false;
            if (System.currentTimeMillis() - session.lastAccessedAt > SESSION_TTL_MS) {
                return false; // Expired — will be cleaned up by background sweeper
            }
            session.lastAccessedAt = System.currentTimeMillis(); // volatile write — visible immediately
            return true;
        } finally {
            rwLock.readLock().unlock();
        }
    }

    public void invalidateSession(String token) {
        rwLock.writeLock().lock();
        try {
            Session session = sessions.get(token);
            if (session != null) session.valid = false;
            sessions.remove(token);
        } finally {
            rwLock.writeLock().unlock();
        }
    }

    /** Background sweeper — removes expired sessions periodically */
    public int purgeExpiredSessions() {
        long now = System.currentTimeMillis();
        rwLock.writeLock().lock();
        try {
            int before = sessions.size();
            sessions.entrySet().removeIf(e ->
                !e.getValue().valid ||
                (now - e.getValue().lastAccessedAt) > SESSION_TTL_MS
            );
            return before - sessions.size();
        } finally {
            rwLock.writeLock().unlock();
        }
    }
}
```

---

## 12.3 StampedLock

### What it is

`StampedLock` (Java 8) is the fastest Java lock for read-heavy workloads. It introduces **optimistic reading** — reading without acquiring any lock at all, then validating that no write occurred.

```
Three modes:
  writeLock()          → exclusive write (like write lock in ReadWriteLock)
  readLock()           → shared read (like read lock in ReadWriteLock)
  tryOptimisticRead()  → NO lock acquired — just a stamp; validate afterwards
```

> ⚠️ `StampedLock` is **NOT reentrant**. A thread holding a read lock that tries to acquire a write lock will **deadlock**. Use carefully.

### Core API

```java
StampedLock lock = new StampedLock();

// Write lock
long stamp = lock.writeLock();
try { /* write */ }
finally { lock.unlockWrite(stamp); }

// Read lock
long stamp = lock.readLock();
try { /* read */ }
finally { lock.unlockRead(stamp); }

// Optimistic read
long stamp = lock.tryOptimisticRead();
// ... read fields ...
if (!lock.validate(stamp)) {
    // A write happened — fall back to real read lock
    stamp = lock.readLock();
    try { /* re-read */ }
    finally { lock.unlockRead(stamp); }
}

// Convert read → write (may fail)
long writeStamp = lock.tryConvertToWriteLock(readStamp);
if (writeStamp == 0) {
    // Conversion failed — release read, acquire write
    lock.unlockRead(readStamp);
    writeStamp = lock.writeLock();
}
```

---

### Real-World Example 1: High-Frequency Trading Price Feed

**Problem:** A market data service receives millions of price updates per second and serves thousands of trading algorithms that read prices in tight loops. Every microsecond of latency matters.

```java
import java.util.concurrent.locks.StampedLock;

public class MarketDataFeed {

    private static class Quote {
        volatile double bid;
        volatile double ask;
        volatile double last;
        volatile long   timestamp;

        Quote(double bid, double ask, double last) {
            this.bid = bid;
            this.ask = ask;
            this.last = last;
            this.timestamp = System.nanoTime();
        }
    }

    private final Map<String, Quote> quotes = new ConcurrentHashMap<>();
    private final StampedLock lock = new StampedLock();

    // ─── WRITE: Market data update (frequent but brief) ──────────────────────

    public void updateQuote(String symbol, double bid, double ask, double last) {
        long stamp = lock.writeLock();
        try {
            Quote q = quotes.computeIfAbsent(symbol, k -> new Quote(0, 0, 0));
            q.bid       = bid;
            q.ask       = ask;
            q.last      = last;
            q.timestamp = System.nanoTime();
        } finally {
            lock.unlockWrite(stamp);
        }
    }

    // ─── READ: Optimistic — zero lock overhead on the hot path ──────────────

    /**
     * Ultra-fast optimistic read for trading algorithms.
     * In the common case (no concurrent write), acquires NO lock.
     * Falls back to read lock only if a write was detected.
     */
    public double[] getBidAsk(String symbol) {
        long stamp = lock.tryOptimisticRead();

        // Read both fields — must be done before validate()
        Quote q = quotes.get(symbol);
        if (q == null) return null;

        double bid = q.bid;
        double ask = q.ask;

        if (!lock.validate(stamp)) {
            // A write occurred while we were reading — fall back to read lock
            stamp = lock.readLock();
            try {
                q   = quotes.get(symbol);
                if (q == null) return null;
                bid = q.bid;
                ask = q.ask;
            } finally {
                lock.unlockRead(stamp);
            }
        }
        return new double[]{bid, ask};
    }

    /**
     * Mid price calculation — optimistic read, convert to write if stale.
     */
    public double getMidPrice(String symbol) {
        long stamp = lock.tryOptimisticRead();
        Quote q = quotes.get(symbol);
        if (q == null) return Double.NaN;
        double mid = (q.bid + q.ask) / 2.0;
        if (lock.validate(stamp)) return mid; // Happy path — no lock needed

        // Fallback: acquire real read lock
        stamp = lock.readLock();
        try {
            q = quotes.get(symbol);
            return q == null ? Double.NaN : (q.bid + q.ask) / 2.0;
        } finally {
            lock.unlockRead(stamp);
        }
    }
}
```

---

### Real-World Example 2: GPS Coordinate Cache with Lock Conversion

**Problem:** A delivery tracking service caches vehicle locations. Most reads are optimistic. Occasionally a stale read must be promoted to a write to trigger a background refresh.

```java
public class VehicleLocationCache {

    private static class Location {
        final double lat, lon;
        final long   updatedAt;
        Location(double lat, double lon) {
            this.lat = lat; this.lon = lon;
            this.updatedAt = System.currentTimeMillis();
        }
    }

    private final Map<String, Location> cache = new HashMap<>();
    private final StampedLock lock = new StampedLock();
    private static final long STALE_THRESHOLD_MS = 5000;

    public void updateLocation(String vehicleId, double lat, double lon) {
        long stamp = lock.writeLock();
        try {
            cache.put(vehicleId, new Location(lat, lon));
        } finally {
            lock.unlockWrite(stamp);
        }
    }

    /**
     * Get location, refreshing from GPS if stale.
     * Demonstrates lock conversion: optimistic → read → write.
     */
    public Location getLocation(String vehicleId) {
        // Stage 1: Optimistic read — fastest path
        long stamp = lock.tryOptimisticRead();
        Location loc = cache.get(vehicleId);
        boolean valid = lock.validate(stamp);

        // Stage 2: Optimistic failed — acquire real read lock
        if (!valid) {
            stamp = lock.readLock();
            try {
                loc = cache.get(vehicleId);
            } finally {
                lock.unlockRead(stamp);
            }
        }

        // Location is fresh — return immediately
        if (loc != null && System.currentTimeMillis() - loc.updatedAt < STALE_THRESHOLD_MS) {
            return loc;
        }

        // Stage 3: Location is stale — need write lock to refresh
        long writeStamp = lock.writeLock();
        try {
            // Double-check: another thread may have refreshed between stage 2 and now
            loc = cache.get(vehicleId);
            if (loc == null || System.currentTimeMillis() - loc.updatedAt >= STALE_THRESHOLD_MS) {
                loc = fetchFromGps(vehicleId); // External GPS call
                cache.put(vehicleId, loc);
            }
            return loc;
        } finally {
            lock.unlockWrite(writeStamp);
        }
    }

    private Location fetchFromGps(String vehicleId) {
        // Simulate GPS fetch
        return new Location(51.5074 + Math.random() * 0.01, -0.1278 + Math.random() * 0.01);
    }
}
```

---

## 12.4 Semaphore

### What it is

A `Semaphore` maintains a count of **permits**. Threads acquire permits before proceeding and release them when done. Unlike mutexes, the releasing thread need not be the acquiring thread.

```java
Semaphore sem = new Semaphore(5);       // 5 permits
Semaphore fair = new Semaphore(5, true); // FIFO ordering

sem.acquire();                    // Acquire 1 permit (block if 0)
sem.acquire(3);                   // Acquire 3 permits atomically
sem.acquireUninterruptibly();     // Cannot be interrupted
sem.tryAcquire();                 // Non-blocking: true/false
sem.tryAcquire(2, SECONDS);       // Wait up to 2s
sem.release();                    // Release 1 permit
sem.release(3);                   // Release 3 permits
sem.availablePermits();           // Current permit count
sem.drainPermits();               // Acquire ALL available permits
```

---

### Real-World Example 1: Database Connection Pool

**Problem:** A service has a pool of 10 database connections. Requests must queue and wait for an available connection rather than fail immediately or create unlimited connections.

```java
import java.util.concurrent.*;

public class DatabaseConnectionPool {

    private final BlockingQueue<Connection> pool;
    private final Semaphore                 permits;
    private final int                       maxSize;

    public DatabaseConnectionPool(int maxConnections) {
        this.maxSize = maxConnections;
        this.pool    = new ArrayBlockingQueue<>(maxConnections);
        this.permits = new Semaphore(maxConnections, true);

        // Pre-populate pool
        for (int i = 0; i < maxConnections; i++) {
            pool.offer(createConnection(i));
        }
    }

    /**
     * Borrow a connection — blocks until one is available.
     * Must be paired with returnConnection() in a finally block.
     */
    public Connection borrow() throws InterruptedException {
        permits.acquire(); // Block if all connections are in use
        return pool.poll(); // Always non-null: permits track available connections
    }

    /**
     * Try to borrow with a timeout. Returns null if unavailable.
     */
    public Connection tryBorrow(long timeout, TimeUnit unit) throws InterruptedException {
        if (!permits.tryAcquire(timeout, unit)) {
            return null; // Connection pool exhausted
        }
        return pool.poll();
    }

    /**
     * Return a connection to the pool.
     */
    public void returnConnection(Connection conn) {
        if (conn != null) {
            if (conn.isValid()) {
                pool.offer(conn);
            } else {
                pool.offer(createConnection(-1)); // Replace broken connection
            }
            permits.release();
        }
    }

    /** Execute work with automatic connection management */
    public <T> T execute(ConnectionWork<T> work) throws Exception {
        Connection conn = borrow();
        try {
            return work.execute(conn);
        } finally {
            returnConnection(conn); // Always return — even if work throws
        }
    }

    public int availableConnections() { return permits.availablePermits(); }
    public int activeConnections()    { return maxSize - permits.availablePermits(); }

    @FunctionalInterface
    public interface ConnectionWork<T> {
        T execute(Connection conn) throws Exception;
    }

    private Connection createConnection(int id) {
        return new Connection("jdbc:postgresql://db:5432/app?conn=" + id);
    }

    // Minimal Connection stub
    public static class Connection {
        private final String url;
        private boolean valid = true;
        Connection(String url) { this.url = url; }
        public boolean isValid() { return valid; }
        public void close() { valid = false; }
    }
}

// Usage
DatabaseConnectionPool pool = new DatabaseConnectionPool(10);

// Option 1: Manual
DatabaseConnectionPool.Connection conn = pool.borrow();
try {
    conn.query("SELECT ...");
} finally {
    pool.returnConnection(conn);
}

// Option 2: Managed
String result = pool.execute(c -> c.query("SELECT name FROM users WHERE id = 1"));
```

---

### Real-World Example 2: API Throttling / Rate Limiting

**Problem:** A microservice exposes an endpoint that calls a paid third-party API with a rate limit of 100 concurrent calls. Internal callers must be throttled.

```java
public class ThrottledApiClient {

    private final Semaphore concurrencyLimit;
    private final AtomicLong totalRequests   = new AtomicLong(0);
    private final AtomicLong rejectedRequests = new AtomicLong(0);
    private final String     apiName;

    public ThrottledApiClient(String apiName, int maxConcurrentCalls) {
        this.apiName          = apiName;
        this.concurrencyLimit = new Semaphore(maxConcurrentCalls, true);
    }

    public <T> T call(Callable<T> apiCall) throws Exception {
        if (!concurrencyLimit.tryAcquire(5, TimeUnit.SECONDS)) {
            rejectedRequests.incrementAndGet();
            throw new RuntimeException(apiName + ": rate limit exceeded (" +
                concurrencyLimit.availablePermits() + " permits remaining)");
        }
        totalRequests.incrementAndGet();
        long start = System.currentTimeMillis();
        try {
            return apiCall.call();
        } finally {
            concurrencyLimit.release();
            long latency = System.currentTimeMillis() - start;
            System.out.printf("[%s] Call completed in %dms | Active: %d%n",
                apiName, latency, getActiveCallCount());
        }
    }

    public int     getActiveCallCount()    { return concurrencyLimit.getMaxPermits() - concurrencyLimit.availablePermits(); }
    public long    getTotalRequests()       { return totalRequests.get(); }
    public long    getRejectedRequests()    { return rejectedRequests.get(); }
    public double  getRejectionRate()       {
        long total = totalRequests.get();
        return total == 0 ? 0 : (double) rejectedRequests.get() / total * 100;
    }
}

// Usage
ThrottledApiClient paymentApi = new ThrottledApiClient("Stripe", 50);
ThrottledApiClient weatherApi = new ThrottledApiClient("OpenWeather", 10);

PaymentResult result = paymentApi.call(() -> stripeClient.chargeCard(cardToken, amount));
```

---

## 12.5 CountDownLatch

### What it is

A one-shot synchronization aid. A count is set at creation; threads call `countDown()` to decrement it. Threads waiting on `await()` are released when the count reaches zero. **Cannot be reset.**

```java
CountDownLatch latch = new CountDownLatch(3);

latch.countDown();              // Decrement count
latch.await();                  // Block until count == 0
latch.await(10, SECONDS);       // Block with timeout
latch.getCount();               // Current count
```

---

### Real-World Example: Microservice Smoke Test on Startup

**Problem:** Before an application starts serving traffic, it must verify that all dependent services (DB, cache, message broker, config server) are healthy. All health checks should run in parallel.

```java
public class StartupHealthChecker {

    private final ExecutorService executor = Executors.newCachedThreadPool();

    public void checkAllOrFail(long timeoutSeconds) throws Exception {
        List<HealthCheck> checks = List.of(
            new HealthCheck("PostgreSQL",      () -> pingDatabase()),
            new HealthCheck("Redis",           () -> pingRedis()),
            new HealthCheck("Kafka",           () -> pingKafka()),
            new HealthCheck("ConfigServer",    () -> pingConfigServer()),
            new HealthCheck("PaymentService",  () -> pingPaymentService())
        );

        CountDownLatch        latch  = new CountDownLatch(checks.size());
        List<String>          errors = new CopyOnWriteArrayList<>();
        List<HealthCheckResult> results = new CopyOnWriteArrayList<>();

        long start = System.currentTimeMillis();

        // Run all checks in parallel
        for (HealthCheck check : checks) {
            executor.submit(() -> {
                try {
                    check.run();
                    results.add(new HealthCheckResult(check.name, true, 0));
                    System.out.println("✅ " + check.name + " — healthy");
                } catch (Exception e) {
                    errors.add(check.name + ": " + e.getMessage());
                    System.err.println("❌ " + check.name + " — FAILED: " + e.getMessage());
                } finally {
                    latch.countDown(); // Always count down — success or failure
                }
            });
        }

        boolean allChecked = latch.await(timeoutSeconds, TimeUnit.SECONDS);
        long elapsed = System.currentTimeMillis() - start;

        if (!allChecked) {
            throw new RuntimeException(
                String.format("Health check timeout after %ds. Pending: %d",
                              timeoutSeconds, latch.getCount()));
        }
        if (!errors.isEmpty()) {
            throw new RuntimeException("Startup health checks failed:\n  - " +
                String.join("\n  - ", errors));
        }
        System.out.printf("All %d health checks passed in %dms — service ready%n",
                          checks.size(), elapsed);
    }

    record HealthCheck(String name, Runnable check) {
        void run() { check.run(); }
    }
    record HealthCheckResult(String name, boolean healthy, long latencyMs) {}

    // Stubs
    private void pingDatabase()      { /* JDBC connection test */ }
    private void pingRedis()         { /* Redis PING command */ }
    private void pingKafka()         { /* Kafka listTopics() */ }
    private void pingConfigServer()  { /* HTTP GET /actuator/health */ }
    private void pingPaymentService(){ /* gRPC health check */ }
}
```

---

## 12.6 CyclicBarrier

### What it is

A reusable synchronization point. All N parties must call `await()` before any of them proceed. The barrier automatically resets after each use. An optional **barrier action** runs once all parties arrive.

```java
CyclicBarrier barrier = new CyclicBarrier(4);          // 4 parties
CyclicBarrier barrier = new CyclicBarrier(4, action);  // With barrier action

barrier.await();                  // Wait for all parties (throws BrokenBarrierException)
barrier.await(5, SECONDS);        // With timeout
barrier.reset();                  // Force reset (waiting threads get BrokenBarrierException)
barrier.isBroken();               // True if barrier was broken (timeout/interrupt/exception)
barrier.getNumberWaiting();       // Parties currently waiting
barrier.getParties();             // Total parties required
```

---

### Real-World Example: Monte Carlo Financial Simulation

**Problem:** A risk engine runs a Monte Carlo simulation to calculate VaR (Value at Risk). Multiple threads simulate market scenarios in phases. Between phases, results must be aggregated before the next phase begins.

```java
import java.util.concurrent.*;
import java.util.concurrent.locks.*;

public class MonteCarloRiskEngine {

    private static final int SIMULATION_THREADS = 8;
    private static final int SCENARIOS_PER_THREAD = 10_000;

    private final double[]   pnlResults     = new double[SIMULATION_THREADS];
    private final double[]   varResults     = new double[SIMULATION_THREADS];
    private volatile double  aggregatedPnl  = 0;
    private volatile double  portfolioVar   = 0;

    // Barrier action: aggregate results from all threads after each phase
    private final CyclicBarrier barrier = new CyclicBarrier(SIMULATION_THREADS, () -> {
        // This runs once after ALL threads complete a phase
        double totalPnl = 0;
        for (double pnl : pnlResults) totalPnl += pnl;
        aggregatedPnl = totalPnl / SIMULATION_THREADS;

        double[] allVars = varResults.clone();
        Arrays.sort(allVars);
        portfolioVar = allVars[(int)(allVars.length * 0.95)]; // 95th percentile

        System.out.printf("Phase complete → Avg P&L: %.2f | 95%% VaR: %.2f%n",
                          aggregatedPnl, portfolioVar);
    });

    public RiskReport runSimulation() throws Exception {
        ExecutorService pool    = Executors.newFixedThreadPool(SIMULATION_THREADS);
        List<Future<?>> futures = new ArrayList<>();

        for (int i = 0; i < SIMULATION_THREADS; i++) {
            final int threadId = i;
            futures.add(pool.submit(() -> runSimulationThread(threadId)));
        }

        for (Future<?> f : futures) f.get(); // Wait for all threads
        pool.shutdown();

        return new RiskReport(aggregatedPnl, portfolioVar);
    }

    private void runSimulationThread(int threadId) throws Exception {
        Random rng = new Random(threadId * 42L);

        // ─── Phase 1: Generate market scenarios ─────────────────────────────
        double[] scenarios = generateScenarios(rng, SCENARIOS_PER_THREAD);
        barrier.await(); // Synchronize — all threads have scenarios ready

        // ─── Phase 2: Price portfolio under each scenario ───────────────────
        double[] portfolioPnl = pricePortfolio(scenarios, rng);
        pnlResults[threadId] = average(portfolioPnl);
        barrier.await(); // Synchronize — barrier action aggregates P&L

        // ─── Phase 3: Calculate VaR contribution ────────────────────────────
        varResults[threadId] = calculateVaR(portfolioPnl, 0.95);
        barrier.await(); // Synchronize — barrier action calculates portfolio VaR

        System.out.printf("Thread %d: P&L=%.2f | VaR=%.2f%n",
                          threadId, pnlResults[threadId], varResults[threadId]);
    }

    private double[] generateScenarios(Random rng, int count) {
        double[] s = new double[count];
        for (int i = 0; i < count; i++) s[i] = rng.nextGaussian() * 0.02;
        return s;
    }

    private double[] pricePortfolio(double[] scenarios, Random rng) {
        double[] pnl = new double[scenarios.length];
        for (int i = 0; i < scenarios.length; i++) pnl[i] = scenarios[i] * 1_000_000;
        return pnl;
    }

    private double calculateVaR(double[] pnl, double confidence) {
        double[] sorted = pnl.clone();
        Arrays.sort(sorted);
        return sorted[(int)(sorted.length * (1 - confidence))];
    }

    private double average(double[] arr) {
        return Arrays.stream(arr).average().orElse(0);
    }

    public record RiskReport(double averagePnl, double var95) {}
}
```

---

## 12.7 Phaser

### What it is

The most flexible synchronization barrier. Supports **dynamic party registration**, **multiple phases**, and optional per-phase actions. Supersedes both `CountDownLatch` and `CyclicBarrier` for complex workflows.

```java
Phaser phaser = new Phaser();          // 0 parties initially
Phaser phaser = new Phaser(3);         // 3 parties
Phaser child  = new Phaser(parent, 3); // Tiered (hierarchical) phaser

phaser.register();              // Dynamically add a party
phaser.bulkRegister(5);         // Add 5 parties at once
phaser.arrive();                // Arrive without waiting
phaser.arriveAndAwaitAdvance(); // Arrive and wait for all others
phaser.arriveAndDeregister();   // Arrive, deregister, and don't wait

phaser.awaitAdvance(phase);     // Wait for phase to complete
phaser.getPhase();              // Current phase number (0, 1, 2, ...)
phaser.getRegisteredParties();  // Registered party count
phaser.getArrivedParties();     // Parties arrived in current phase

// Override to run code between phases
Phaser phaser = new Phaser() {
    @Override protected boolean onAdvance(int phase, int registeredParties) {
        System.out.println("Phase " + phase + " complete");
        return registeredParties == 0; // Return true to terminate the phaser
    }
};
```

---

### Real-World Example: ETL Pipeline with Dynamic Worker Scaling

**Problem:** An ETL pipeline processes data in stages (Extract → Transform → Load). The number of workers for each stage varies. New workers can be added mid-flight if a stage is falling behind.

```java
public class EtlPipeline {

    private final Phaser      phaser  = new Phaser(1); // 1 = coordinator
    private final List<String> errors = new CopyOnWriteArrayList<>();

    public void run(List<DataBatch> batches) throws InterruptedException {
        System.out.println("Starting ETL for " + batches.size() + " batches");

        // ─── EXTRACT phase ───────────────────────────────────────────────────
        List<RawData> rawData = runPhase("EXTRACT", batches, batch -> {
            RawData data = extractFromSource(batch);
            System.out.println("Extracted " + data.recordCount() + " records from " + batch.id());
            return data;
        });

        if (!errors.isEmpty()) throw new RuntimeException("Extract failed: " + errors);

        // ─── TRANSFORM phase (more workers — CPU intensive) ──────────────────
        List<TransformedData> transformed = runPhase("TRANSFORM", rawData, raw -> {
            TransformedData t = applyTransformations(raw);
            System.out.println("Transformed " + t.recordCount() + " records");
            return t;
        });

        if (!errors.isEmpty()) throw new RuntimeException("Transform failed: " + errors);

        // ─── LOAD phase ───────────────────────────────────────────────────────
        runPhase("LOAD", transformed, data -> {
            loadToWarehouse(data);
            System.out.println("Loaded " + data.recordCount() + " records to warehouse");
            return null;
        });

        if (!errors.isEmpty()) throw new RuntimeException("Load failed: " + errors);

        System.out.println("ETL pipeline complete");
        phaser.arriveAndDeregister(); // Coordinator deregisters
    }

    private <I, O> List<O> runPhase(String phaseName, List<I> inputs,
                                     PhaseTask<I, O> task) throws InterruptedException {
        List<O> results = new CopyOnWriteArrayList<>();
        ExecutorService pool = Executors.newFixedThreadPool(
            Math.min(inputs.size(), Runtime.getRuntime().availableProcessors()));

        for (I input : inputs) {
            phaser.register(); // Register each worker dynamically
            pool.submit(() -> {
                try {
                    O result = task.process(input);
                    if (result != null) results.add(result);
                } catch (Exception e) {
                    errors.add(phaseName + ": " + e.getMessage());
                } finally {
                    phaser.arriveAndDeregister(); // Done — signal and deregister
                }
            });
        }

        // Coordinator waits for all workers in this phase to finish
        phaser.arriveAndAwaitAdvance();
        pool.shutdown();

        System.out.printf("[%s] Phase complete: %d inputs → %d outputs%n",
                          phaseName, inputs.size(), results.size());
        return new ArrayList<>(results);
    }

    @FunctionalInterface
    interface PhaseTask<I, O> { O process(I input) throws Exception; }

    // Stubs
    private RawData extractFromSource(DataBatch b)  { return new RawData(b.id(), 1000); }
    private TransformedData applyTransformations(RawData r) { return new TransformedData(r.source(), r.recordCount()); }
    private void loadToWarehouse(TransformedData d) { /* JDBC batch insert */ }

    record DataBatch(String id) {}
    record RawData(String source, int recordCount) {}
    record TransformedData(String source, int recordCount) {}
}
```

---

## 12.8 Exchanger

### What it is

Allows exactly **two threads to swap objects** at a synchronization point. Each calls `exchange(myObject)` and receives the other thread's object. Both threads block until the other arrives.

```java
Exchanger<DataBuffer> exchanger = new Exchanger<>();

// Thread 1
DataBuffer full = fillBuffer();
DataBuffer empty = exchanger.exchange(full);    // Give full, receive empty

// Thread 2
DataBuffer toProcess = exchanger.exchange(empty); // Give empty, receive full
process(toProcess);
```

---

### Real-World Example: High-Throughput Log File Writer (Double Buffering)

**Problem:** A logging system must write logs to disk without slowing down the application. A producer thread fills a buffer; a consumer thread flushes it. Using a single shared buffer would require constant locking. Double buffering allows both to work simultaneously.

```java
import java.util.concurrent.*;

public class DoubleBufferedLogger {

    private static final int BUFFER_SIZE = 10_000;

    private static class LogBuffer {
        private final List<String> entries = new ArrayList<>(BUFFER_SIZE);
        private final String       name;

        LogBuffer(String name) { this.name = name; }

        void add(String entry)    { entries.add(entry); }
        boolean isFull()          { return entries.size() >= BUFFER_SIZE; }
        List<String> getEntries() { return entries; }
        void clear()              { entries.clear(); }

        @Override public String toString() {
            return name + "[" + entries.size() + " entries]";
        }
    }

    private final Exchanger<LogBuffer> exchanger = new Exchanger<>();
    private LogBuffer writeBuffer = new LogBuffer("A");
    private final List<String> flushedLogs = new CopyOnWriteArrayList<>(); // For testing

    public void start() {
        // Consumer thread: receives full buffers, flushes to disk, returns empty buffer
        Thread consumer = new Thread(() -> {
            LogBuffer emptyBuffer = new LogBuffer("B"); // Start with an empty buffer
            while (!Thread.currentThread().isInterrupted()) {
                try {
                    // Hand empty buffer to producer, receive full buffer
                    LogBuffer fullBuffer = exchanger.exchange(emptyBuffer);
                    flushToDisk(fullBuffer);
                    fullBuffer.clear();
                    emptyBuffer = fullBuffer; // This is now empty — reuse it
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }, "log-flusher");
        consumer.setDaemon(true);
        consumer.start();
    }

    /**
     * Called by application threads to log a message.
     * Only blocks when the buffer is full (rare under normal load).
     */
    public synchronized void log(String message) throws InterruptedException {
        writeBuffer.add(System.currentTimeMillis() + " | " + message);
        if (writeBuffer.isFull()) {
            // Buffer full — exchange with consumer thread
            System.out.println("Swapping buffer: " + writeBuffer + " with consumer");
            writeBuffer = exchanger.exchange(writeBuffer); // Block until consumer is ready
        }
    }

    private void flushToDisk(LogBuffer buffer) {
        // Simulate disk write
        System.out.println("Flushing " + buffer.entries.size() + " log entries to disk");
        flushedLogs.addAll(buffer.getEntries());
    }
}

// Usage
DoubleBufferedLogger logger = new DoubleBufferedLogger();
logger.start();

// Application threads
for (int i = 0; i < 50_000; i++) {
    logger.log("Request " + i + " processed in " + (i % 50) + "ms");
}
```

---

## 12.9 LockSupport

### What it is

The lowest-level thread blocking primitive used internally by the JVM's concurrency framework (AQS). Unlike `wait()`/`notify()`, it requires no monitor ownership and supports permit-based signaling.

```java
LockSupport.park();              // Block current thread (permit-based)
LockSupport.park(blocker);       // Block with a blocker object (for diagnostics/thread dumps)
LockSupport.parkNanos(nanos);    // Block up to N nanoseconds
LockSupport.parkUntil(deadline); // Block until absolute time (epoch ms)

LockSupport.unpark(thread);      // Unblock a specific thread (can be called before park!)
```

**Key advantages over `wait()`/`notify()`:**
- No monitor ownership required
- `unpark()` can be called **before** `park()` — permit is stored, next `park()` returns immediately
- `unpark()` targets a **specific thread**, not an arbitrary one from a wait set

---

### Real-World Example: Custom Non-Blocking Handoff Queue

```java
import java.util.concurrent.atomic.*;
import java.util.concurrent.locks.LockSupport;

/**
 * A single-producer, single-consumer non-blocking handoff.
 * Producer parks if consumer hasn't claimed yet; consumer parks if nothing to claim.
 */
public class ThreadHandoff<T> {

    private final AtomicReference<T>      slot           = new AtomicReference<>();
    private final AtomicReference<Thread> waitingConsumer = new AtomicReference<>();
    private final AtomicReference<Thread> waitingProducer = new AtomicReference<>();

    /**
     * Producer: deposit an item and wait until consumer takes it.
     */
    public void deposit(T item) throws InterruptedException {
        if (!slot.compareAndSet(null, item)) {
            throw new IllegalStateException("Slot already occupied — single producer only");
        }

        Thread consumer = waitingConsumer.get();
        if (consumer != null) {
            LockSupport.unpark(consumer); // Consumer is waiting — wake it up
        } else {
            // No consumer yet — park until one arrives
            waitingProducer.set(Thread.currentThread());
            while (slot.get() != null) {
                LockSupport.park(this); // Park with 'this' as blocker for diagnostics
                if (Thread.currentThread().isInterrupted()) {
                    slot.set(null);
                    throw new InterruptedException();
                }
            }
            waitingProducer.set(null);
        }
    }

    /**
     * Consumer: take the deposited item, or wait until one is available.
     */
    public T take() throws InterruptedException {
        T item = slot.getAndSet(null);
        if (item != null) {
            Thread producer = waitingProducer.get();
            if (producer != null) LockSupport.unpark(producer);
            return item;
        }

        // Nothing available — park until producer deposits
        waitingConsumer.set(Thread.currentThread());
        while ((item = slot.getAndSet(null)) == null) {
            LockSupport.park(this);
            if (Thread.currentThread().isInterrupted()) {
                waitingConsumer.set(null);
                throw new InterruptedException();
            }
        }
        waitingConsumer.set(null);
        Thread producer = waitingProducer.get();
        if (producer != null) LockSupport.unpark(producer);
        return item;
    }
}
```

---

## Comparison Table

| Lock / Utility | Mutual Exclusion | Concurrent Readers | Try/Timeout | Interruptible | Fair | Reentrant | Permits > 1 | Reusable | Best For |
|---|---|---|---|---|---|---|---|---|---|
| `synchronized` | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ✅ | Simple critical sections |
| `ReentrantLock` | ✅ | ❌ | ✅ | ✅ | Optional | ✅ | ❌ | ✅ | Advanced locking patterns |
| `ReentrantReadWriteLock` | Write only | ✅ | ✅ | ✅ | Optional | ✅ | ❌ | ✅ | Read-heavy data |
| `StampedLock` | ✅ | ✅ (optimistic) | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | Max read throughput |
| `Semaphore` | Optional (N=1) | ✅ (N > 1) | ✅ | ✅ | Optional | ❌ | ✅ | ✅ | Resource pools, rate limits |
| `CountDownLatch` | ❌ | N/A | ✅ | ✅ | N/A | N/A | N/A | ❌ | One-time startup gate |
| `CyclicBarrier` | ❌ | N/A | ✅ | ✅ | N/A | N/A | N/A | ✅ | Phase synchronization |
| `Phaser` | ❌ | N/A | ✅ | ✅ | N/A | N/A | N/A | ✅ | Dynamic multi-phase |
| `Exchanger` | ❌ | N/A | ✅ | ✅ | N/A | N/A | N/A | ✅ | Two-thread data swap |
| `LockSupport` | ❌ | N/A | ✅ | ✅ | N/A | N/A | N/A | ✅ | Custom synchronizers |

---

## Choosing the Right Lock

```
Do you need mutual exclusion?
  Yes →
    One thread at a time?
      Simple block (no timeout/interrupt) → synchronized
      Need tryLock / timeout / interrupt  → ReentrantLock
      Read-heavy (reads >> writes)         → ReentrantReadWriteLock
      Ultra-high read throughput           → StampedLock (if non-reentrant OK)

  No →
    Limit concurrent access to N resources → Semaphore
    One-time "N events happened" gate      → CountDownLatch
    All N threads rendezvous & repeat      → CyclicBarrier
    Dynamic workers, multi-phase           → Phaser
    Two threads swap data                  → Exchanger
    Build your own synchronizer            → LockSupport + AtomicReference
```

---

**Navigation**

[← Interview Questions](11-interview-questions.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md)

---

*Last updated: 2026 | Java 21 LTS*
