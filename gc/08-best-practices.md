# 08 — Best Practices

> Covers: Heap configuration rules, collector selection, GC-friendly coding, common pitfalls, memory leak patterns, and production checklists.

---

**Navigation**

[← GC Monitoring Tools](07-gc-monitoring-tools.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Interview Questions](09-interview-questions.md)

---

## Table of Contents

- [8.1 Heap Configuration](#81-heap-configuration)
- [8.2 Collector Selection](#82-collector-selection)
- [8.3 Allocation Patterns](#83-allocation-patterns)
- [8.4 Common Pitfalls](#84-common-pitfalls)
- [8.5 GC-Friendly Code Patterns](#85-gc-friendly-code-patterns)
- [8.6 Memory Leak Patterns](#86-memory-leak-patterns)
- [8.7 Production Readiness Checklist](#87-production-readiness-checklist)

---

## 8.1 Heap Configuration

### Always Set `-Xms == -Xmx` in Production

```bash
# BAD — heap resizing pauses
-Xms512m -Xmx4g

# GOOD — heap committed at startup, no resize pauses
-Xms4g -Xmx4g
```

**Why:** When the heap grows, the JVM requests more memory from the OS, which takes time. When it shrinks, memory is returned. This causes unpredictable pause spikes, especially under burst traffic.

### Size Based on Live Data

```
Rule of thumb:
  Xmx ≥ 3× live data set size  (for G1/ZGC)
  Xmx ≥ 4× live data set size  (for Parallel GC)

Live data = heap size right after a Full GC

Example: live data = 1.5 GB
  G1: -Xmx4g (3× = 4.5 → round to 4 GB)
  ZGC: -Xmx6g (4× = 6 GB — ZGC needs more headroom)
```

### Cap Metaspace

```bash
# Always set! Without it, a class-loading leak exhausts native memory
-XX:MaxMetaspaceSize=512m

# For apps with many dynamic class loaders (Groovy, CGLIB, Spring AOP)
-XX:MaxMetaspaceSize=1g
```

### Pre-Touch Heap Pages in Production

```bash
-XX:+AlwaysPreTouch    # Fault all heap pages at JVM startup
                       # Eliminates page-fault pauses during warm-up
                       # Increases startup time but makes GC pauses more predictable
```

---

## 8.2 Collector Selection

### Quick Reference

| Scenario | Recommended Collector |
|---|---|
| Java 9+ web service (default) | G1 GC |
| Batch / ETL / throughput-critical | Parallel GC |
| Latency SLA < 10ms | ZGC (Java 15+) |
| Very large heap (> 100 GB) | ZGC |
| OpenJDK, low latency | Shenandoah |
| Benchmarking allocation | Epsilon GC |
| Legacy Java 8 | Parallel GC (default) or G1 |

### Avoid These Combinations

```bash
# DON'T: CMS is removed in Java 14
-XX:+UseConcMarkSweepGC   # Use G1 instead

# DON'T: Mixing conflicting flags
-XX:+UseG1GC -XX:+UseParallelGC  # Undefined behavior

# DON'T: Epsilon in production
-XX:+UseEpsilonGC  # JVM will OOM when heap fills!
```

---

## 8.3 Allocation Patterns

### Reduce Short-Lived Object Creation

```java
// BAD: Creates a new StringBuilder each iteration
for (int i = 0; i < 1_000_000; i++) {
    String result = new StringBuilder()
        .append("user-").append(i).toString();
    process(result);
}

// BETTER: Reuse a single StringBuilder
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1_000_000; i++) {
    sb.setLength(0);   // Reset without allocation
    sb.append("user-").append(i);
    process(sb.toString());
}
```

### Object Pooling for Expensive Objects

```java
// Pool reusable, expensive objects (connections, buffers)
// Apache Commons Pool2 example:
GenericObjectPool<ExpensiveResource> pool =
    new GenericObjectPool<>(new ExpensiveResourceFactory());
pool.setMaxTotal(10);
pool.setMaxIdle(5);

ExpensiveResource resource = pool.borrowObject();
try {
    resource.process(data);
} finally {
    pool.returnObject(resource);
}
```

### Prefer Primitives Over Boxed Types

```java
// BAD: Autoboxing creates Integer objects on every loop iteration
List<Integer> nums = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    nums.add(i);         // Autoboxes int → Integer → allocation
}

// BETTER: Use primitive-specialized collections (e.g., Eclipse Collections, Trove)
IntArrayList nums = new IntArrayList(1_000_000);
for (int i = 0; i < 1_000_000; i++) {
    nums.add(i);         // No boxing
}

// Or: use arrays for numeric data
int[] nums = new int[1_000_000];
```

### Avoid String Concatenation in Loops

```java
// BAD: O(n²) — creates a new String on every +
String result = "";
for (String part : parts) {
    result += part;     // Each += creates a new String object
}

// GOOD: O(n)
StringBuilder sb = new StringBuilder();
for (String part : parts) {
    sb.append(part);
}
String result = sb.toString();

// Also good for simple cases:
String result = String.join("", parts);
```

### Use `String.intern()` Carefully

```java
// String.intern() stores string in the interning pool (Metaspace)
// Saves heap memory if the same string appears many times

// GOOD for: high-cardinality repeated strings (usernames, product codes)
String internedKey = key.intern();  // Only one copy in memory

// BAD for: unique strings (UUIDs, timestamps) — wastes Metaspace
String uuid = UUID.randomUUID().toString().intern(); // Never released!
```

---

## 8.4 Common Pitfalls

### Pitfall 1: Calling `System.gc()` in Application Code

```java
// DON'T DO THIS:
System.gc();   // Hint to JVM — may trigger a Full GC pause in production
               // Can cause multi-second pauses
               // Never rely on it for correctness

// Exceptions:
// - Before heap dump capture in diagnostic tools (controlled environment)
// - Before benchmarks to establish a clean baseline

// In production: remove all System.gc() calls
// If library calls it: -XX:+DisableExplicitGC to ignore it completely
// Or: -XX:+ExplicitGCInvokesConcurrent (G1/ZGC: runs concurrent GC instead of Full GC)
```

### Pitfall 2: Forgetting `MaxMetaspaceSize`

```java
// Without -XX:MaxMetaspaceSize:
// App uses CGLIB proxies, Groovy scripts, or ClassLoaders in a loop
// → Each creates new classes → Metaspace grows unbounded
// → Eventually: java.lang.OutOfMemoryError: Metaspace
// → OS may also run out of native memory → JVM crash

// Always add:
// -XX:MaxMetaspaceSize=512m
```

### Pitfall 3: Humongous Allocations Bypassing Young Gen

```java
// Objects > 50% of G1 region size go directly to Old Gen
// With -XX:G1HeapRegionSize=4m → threshold = 2 MB

byte[] large = new byte[3 * 1024 * 1024]; // 3 MB → Humongous!

// Symptoms:
// - GC log: "Pause Young (Concurrent Start) (G1 Humongous Allocation)"
// - Old Gen fills up faster than expected
// - Frequent concurrent marking cycles

// Fix 1: Increase region size
// -XX:G1HeapRegionSize=32m → threshold = 16 MB

// Fix 2: Avoid large allocations (reuse buffers, use pooling)
// ByteBuffer.allocateDirect() for large I/O buffers
```

### Pitfall 4: Heap Too Large for the GC to Handle

```java
// Common mistake: "more heap = better"
// With Parallel/Serial GC: larger heap = LONGER STW pauses for Old Gen collection

// 32 GB heap with Parallel GC → Full GC can pause for 30+ seconds
// Fix: switch to G1 or ZGC for large heaps

// Rule of thumb:
// Parallel GC: keep heap ≤ 8–16 GB
// G1:          up to ~100 GB
// ZGC:         up to 16 TB
```

### Pitfall 5: Not Monitoring GC in Production

```bash
# Always enable GC logging — it has < 1% overhead
# Without logs, you're flying blind when production has GC issues

# Minimum production GC logging:
-Xlog:gc*:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=10,filesize=20m
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/dumps/
```

### Pitfall 6: Small Young Gen with High Allocation Rate

```java
// Web services with many short-lived objects need a large Young Gen
// If Young Gen is too small → Minor GC runs constantly → GC overhead > 10%

// Diagnosis: jstat -gcutil shows E% filling every 1-2 seconds
// Fix:
// -XX:NewRatio=1   → Young Gen = 50% of heap (default is 33%)
// or
// -Xmn4g           → fixed Young Gen size
```

---

## 8.5 GC-Friendly Code Patterns

### Pattern 1: Immutable Objects

```java
// Immutable objects can't have cross-generational references
// → No write barrier overhead; card table stays clean

// Use immutable value objects:
public final class Money {
    private final long amount;
    private final Currency currency;

    public Money(long amount, Currency currency) {
        this.amount = amount;
        this.currency = currency;
    }
    // No setters — immutable
}
```

### Pattern 2: Object Reuse with Pooling

```java
// For frequently created/destroyed heavy objects
public class BufferPool {
    private final ArrayDeque<byte[]> pool = new ArrayDeque<>();
    private final int bufferSize;

    public BufferPool(int bufferSize) {
        this.bufferSize = bufferSize;
    }

    public byte[] borrow() {
        byte[] buf = pool.pollFirst();
        return buf != null ? buf : new byte[bufferSize];
    }

    public void release(byte[] buf) {
        // Clear sensitive data before returning
        Arrays.fill(buf, (byte) 0);
        pool.addFirst(buf);
    }
}
```

### Pattern 3: Avoid Long-Lived Caches Without Eviction

```java
// BAD: Unbounded cache — never released → Old Gen memory leak
static Map<String, Object> cache = new HashMap<>();
cache.put(key, value); // Grows forever!

// GOOD: Use Caffeine or Guava Cache with size + TTL limits
Cache<String, Object> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(30, TimeUnit.MINUTES)
    .build();
```

### Pattern 4: Use `try-with-resources` for Resource Objects

```java
// Resources closed promptly → objects eligible for GC sooner
// No relying on finalize() for cleanup

try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql);
     ResultSet rs = ps.executeQuery()) {
    while (rs.next()) {
        process(rs);
    }
} // conn, ps, rs closed automatically in reverse order
```

---

## 8.6 Memory Leak Patterns

### Pattern 1: Static Collection Growing Unbounded

```java
// LEAK: Static map holds references forever
public class EventRegistry {
    private static final Map<String, EventListener> listeners = new HashMap<>();

    public static void register(String key, EventListener listener) {
        listeners.put(key, listener);   // Never removed!
    }
    // Missing: deregister() method
}

// FIX: Add explicit removal, or use WeakHashMap
private static final Map<String, EventListener> listeners =
    Collections.synchronizedMap(new WeakHashMap<>());
```

### Pattern 2: Inner Class Holding Outer Reference

```java
// LEAK: Anonymous Runnable holds reference to enclosing MyService
public class MyService {
    private byte[] largeData = new byte[100 * 1024 * 1024]; // 100 MB

    public void scheduleWork() {
        executor.submit(new Runnable() {
            @Override
            public void run() {
                // Even if we don't use largeData, this Runnable holds a reference to
                // MyService, which holds largeData → 100 MB can't be GC'd
            }
        });
    }

    // FIX: Use static inner class or lambda with captured local variables only
    public void scheduleWorkFixed() {
        byte[] snapshot = Arrays.copyOf(largeData, 100); // Only capture what's needed
        executor.submit(() -> process(snapshot));
    }
}
```

### Pattern 3: ThreadLocal Not Removed in Thread Pools

```java
// LEAK: ThreadLocal values persist across requests in thread pools
ThreadLocal<UserContext> userContext = new ThreadLocal<>();

// In servlet/filter:
userContext.set(context); // Set at request start
handleRequest();
// If remove() not called → stays in thread's ThreadLocal map forever
// → next request on this thread sees stale context

// FIX: Always in finally block
try {
    userContext.set(context);
    handleRequest();
} finally {
    userContext.remove(); // Critical!
}
```

### Pattern 4: Listeners/Observers Not Deregistered

```java
// LEAK: Every subscribe() adds a listener; unsubscribe() never called
eventBus.subscribe(myListener);   // myListener held by eventBus
// myListener object can never be GC'd while eventBus is alive

// FIX: Always unsubscribe in cleanup/destroy
try {
    eventBus.subscribe(myListener);
    doWork();
} finally {
    eventBus.unsubscribe(myListener);
}

// Or: use weak references in the event bus implementation
```

---

## 8.7 Production Readiness Checklist

```
GC Configuration:
  ☐ -Xms == -Xmx (fixed heap, no resize pauses)
  ☐ -XX:MaxMetaspaceSize set (prevent unbounded growth)
  ☐ Appropriate collector chosen for workload
  ☐ -XX:+AlwaysPreTouch (eliminate page-fault pauses)
  ☐ -XX:+HeapDumpOnOutOfMemoryError (capture OOM evidence)
  ☐ -XX:HeapDumpPath=/dumps/ (writable directory with enough disk)

GC Logging (always enabled):
  ☐ -Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags:filecount=10,filesize=20m
  ☐ Log rotation configured (filecount + filesize)
  ☐ Log directory has sufficient disk space

Monitoring:
  ☐ JMX exposed for heap/GC metrics (via Prometheus JMX exporter or similar)
  ☐ Alerts on: Old Gen > 80%, Full GC count increasing, GC overhead > 10%
  ☐ Heap usage dashboard configured

Code Review:
  ☐ No System.gc() calls in application code
  ☐ No finalizer() methods (use Cleaner or try-with-resources)
  ☐ Static collections have eviction policies (Caffeine, size limits)
  ☐ ThreadLocal.remove() called in thread pool contexts
  ☐ Event listeners deregistered when no longer needed

Load Testing:
  ☐ Run GC logs through GCEasy after load test
  ☐ p99 GC pause within SLA
  ☐ No Full GC during steady-state load
  ☐ Old Gen stable (not continuously growing)
  ☐ GC overhead < 5% of total CPU time
```

---

**Navigation**

[← GC Monitoring Tools](07-gc-monitoring-tools.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Interview Questions](09-interview-questions.md)

---

*Last updated: 2026 | Java 21 LTS*
