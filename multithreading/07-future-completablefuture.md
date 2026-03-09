# 07 — Future, Callable & CompletableFuture

> Covers: Future<T>, Callable, blocking result retrieval, CompletableFuture async pipelines, combining futures, and error handling.

---

**Navigation**

[← Executor Framework](06-executor-framework.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Concurrent Collections](08-concurrent-collections.md)

---

## Table of Contents

- [11.1 Future\<T\>](#111-futuret)
- [11.2 CompletableFuture — Async Pipelines](#112-completablefuture--async-pipelines)
- [11.3 CompletableFuture — Combining Futures](#113-completablefuture--combining-futures)
- [11.4 CompletableFuture — Error Handling](#114-completablefuture--error-handling)
- [11.5 CompletableFuture — Timeouts (Java 9+)](#115-completablefuture--timeouts-java-9)
- [11.6 thenApply vs thenCompose vs thenCombine](#116-thenapply-vs-thencompose-vs-thencombine)
- [Summary](#summary)

---

## 11.1 Future\<T\>

`Future<T>` represents the result of an **asynchronous computation** that may not yet be complete.

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

// Submit a Callable — returns a Future
Future<String> future = executor.submit(() -> {
    Thread.sleep(2000);
    return fetchUserProfile("user-123");
});

// Do other work while the task runs in the background
doOtherWork();

// Retrieve the result (blocks until ready)
String profile = future.get();                         // Blocks indefinitely
String profile2 = future.get(5, TimeUnit.SECONDS);    // Max 5 seconds → TimeoutException

// Check status without blocking
boolean done      = future.isDone();      // true if complete, cancelled, or exceptioned
boolean cancelled = future.isCancelled(); // true if cancelled

// Cancel the task
future.cancel(true);  // true = interrupt if currently running
                      // false = don't interrupt, just prevent from starting
```

### Handling Future Exceptions

```java
Future<String> future = executor.submit(() -> {
    if (someCondition) throw new IOException("Service unavailable");
    return "result";
});

try {
    String result = future.get();
} catch (ExecutionException e) {
    Throwable cause = e.getCause();          // The actual exception from Callable
    if (cause instanceof IOException) {
        handleIOFailure((IOException) cause);
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();      // Restore interrupt flag
} catch (TimeoutException e) {
    future.cancel(true);
    handleTimeout();
}
```

### Limitations of Future

| Problem | Description |
|---|---|
| Blocking `get()` | No way to attach a callback — must block the calling thread |
| No chaining | Cannot automatically trigger the next step when result is ready |
| No combining | Difficult to wait for multiple futures and merge results |
| No exception propagation | Exceptions are buried inside `ExecutionException` |
| No timeout without blocking | `get(timeout)` still blocks the calling thread |

These limitations are solved by `CompletableFuture`.

---

## 11.2 CompletableFuture — Async Pipelines

`CompletableFuture<T>` extends `Future<T>` with a rich API for building **non-blocking async pipelines**.

### Creating

```java
// Async task with no return value
CompletableFuture<Void> cf1 = CompletableFuture.runAsync(() -> sendEmail());

// Async task with return value
CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> fetchUser("u1"));

// Async with custom executor (recommended for I/O)
ExecutorService ioPool = Executors.newFixedThreadPool(20);
CompletableFuture<String> cf3 = CompletableFuture.supplyAsync(
    () -> callExternalApi(), ioPool
);

// Already completed
CompletableFuture<String> done = CompletableFuture.completedFuture("cached-result");

// Already failed
CompletableFuture<String> failed = CompletableFuture.failedFuture(
    new RuntimeException("Service down")
);
```

### Building Pipelines

```java
CompletableFuture<String> pipeline = CompletableFuture

    // Step 1: Fetch user asynchronously
    .supplyAsync(() -> fetchUser("user-1"), ioPool)

    // Step 2: Transform result (runs in same thread as step 1 or completing thread)
    .thenApply(user -> enrichUserData(user))

    // Step 3: Another async transformation (submitted to ioPool)
    .thenApplyAsync(user -> fetchRecommendations(user), ioPool)

    // Step 4: Side effect (log) — doesn't change the result
    .thenApply(recs -> {
        log.info("Fetched {} recommendations", recs.size());
        return recs;
    })

    // Step 5: Terminal transformation
    .thenApply(recs -> formatAsJson(recs));
```

### Sync vs Async variants

```java
cf.thenApply(fn)          // Runs in the thread that completed the previous stage
cf.thenApplyAsync(fn)     // Runs in ForkJoinPool.commonPool()
cf.thenApplyAsync(fn, ex) // Runs in provided executor 'ex'
```

> **Rule of thumb:** For I/O or heavy computation, always use `...Async(fn, executor)` with a dedicated pool to avoid blocking the thread that completed the prior stage.

---

## 11.3 CompletableFuture — Combining Futures

### thenCombine — Wait for Both, Merge

```java
CompletableFuture<User>        userFuture    = CompletableFuture.supplyAsync(() -> fetchUser(id),   ioPool);
CompletableFuture<List<Order>> ordersFuture  = CompletableFuture.supplyAsync(() -> fetchOrders(id), ioPool);

// Both run in parallel; combined when BOTH complete
CompletableFuture<UserDashboard> dashboard =
    userFuture.thenCombine(ordersFuture, (user, orders) ->
        new UserDashboard(user, orders)
    );
```

### allOf — Wait for All

```java
List<String> productIds = List.of("P1", "P2", "P3", "P4");

List<CompletableFuture<Product>> futures = productIds.stream()
    .map(id -> CompletableFuture.supplyAsync(() -> fetchProduct(id), ioPool))
    .collect(Collectors.toList());

// Wait for ALL futures to complete
CompletableFuture<Void> allDone =
    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]));

// Collect results after all complete
List<Product> products = allDone.thenApply(v ->
    futures.stream()
           .map(CompletableFuture::join) // join() doesn't throw checked exceptions
           .collect(Collectors.toList())
).get();
```

### anyOf — First to Complete Wins

```java
// Redundant requests to two endpoints — use whichever responds first
CompletableFuture<Object> fastest = CompletableFuture.anyOf(
    CompletableFuture.supplyAsync(() -> fetchFromPrimaryDC()),
    CompletableFuture.supplyAsync(() -> fetchFromSecondaryDC())
);

String result = (String) fastest.get();
```

### thenAcceptBoth — Consume Both, No Result

```java
userFuture.thenAcceptBoth(ordersFuture, (user, orders) -> {
    auditLog.record(user, orders); // Side effect — no return value
});
```

---

## 11.4 CompletableFuture — Error Handling

### exceptionally — Recover on Failure

```java
CompletableFuture<String> result = CompletableFuture
    .supplyAsync(() -> fetchFromRemoteService())
    .exceptionally(ex -> {
        log.error("Remote call failed", ex);
        return "fallback-response";          // Recovery value
    });
// If supplyAsync succeeds → exceptionally is a no-op, result passes through
// If supplyAsync fails  → exceptionally runs, returns fallback
```

### handle — Always Runs (Success or Failure)

```java
CompletableFuture<String> result = CompletableFuture
    .supplyAsync(() -> fetchData())
    .handle((data, ex) -> {
        if (ex != null) {
            log.error("Failed: {}", ex.getMessage());
            return "default-data";
        }
        return data.toUpperCase();
    });
// handle() always runs — receives (result, null) on success or (null, exception) on failure
```

### whenComplete — Side Effect on Completion

```java
CompletableFuture<String> result = CompletableFuture
    .supplyAsync(() -> fetchData())
    .whenComplete((data, ex) -> {
        if (ex != null) metricsClient.increment("fetch.error");
        else            metricsClient.increment("fetch.success");
        // Does NOT change the result — just a side effect
    });
```

### Comparing Error Handlers

| Method | Runs when | Can change result | Can recover from exception |
|---|---|---|---|
| `exceptionally(fn)` | Only on failure | ✅ Yes (recovery value) | ✅ Yes |
| `handle(biFunc)` | Always | ✅ Yes | ✅ Yes |
| `whenComplete(biConsumer)` | Always | ❌ No (side effect only) | ❌ No |

---

## 11.5 CompletableFuture — Timeouts (Java 9+)

```java
// orTimeout — completes exceptionally with TimeoutException after deadline
CompletableFuture<String> result = CompletableFuture
    .supplyAsync(() -> slowExternalCall())
    .orTimeout(5, TimeUnit.SECONDS)
    .exceptionally(ex -> {
        if (ex instanceof TimeoutException) return "timeout-fallback";
        throw new CompletionException(ex);
    });

// completeOnTimeout — completes with a default value instead of exception
CompletableFuture<String> result2 = CompletableFuture
    .supplyAsync(() -> slowExternalCall())
    .completeOnTimeout("default-value", 5, TimeUnit.SECONDS);
```

---

## 11.6 thenApply vs thenCompose vs thenCombine

| Method | Input function | Output | Analogy |
|---|---|---|---|
| `thenApply(T → U)` | Sync function returning `U` | `CF<U>` | `Stream.map()` |
| `thenCompose(T → CF<U>)` | Async function returning `CF<U>` | `CF<U>` (flattened) | `Stream.flatMap()` |
| `thenCombine(CF<U>, BiFunc)` | Two futures + merge function | `CF<V>` | Zip two streams |

```java
// thenApply — sync transform
cf.thenApply(str -> str.toUpperCase())     // CF<String> → CF<String>

// thenCompose — async transform (avoids CF<CF<T>> nesting)
cf.thenCompose(userId -> fetchUserAsync(userId))  // CF<String> → CF<User>
// NOT: cf.thenApply(userId -> fetchUserAsync(userId)) → CF<CF<User>> ← WRONG

// thenCombine — parallel then merge
userCF.thenCombine(ordersCF, (u, o) -> new Dashboard(u, o))
```

---

## Summary

| API | Use case |
|---|---|
| `Future.get()` | Simple blocking result retrieval |
| `CompletableFuture.supplyAsync()` | Start an async computation |
| `thenApply()` | Sync transform in the pipeline |
| `thenApplyAsync()` | Async transform on separate thread |
| `thenCompose()` | Chain async stages (flatMap) |
| `thenCombine()` | Merge two independent parallel futures |
| `allOf()` | Wait for all futures to complete |
| `anyOf()` | Wait for the first future to complete |
| `exceptionally()` | Recover from failure with fallback |
| `handle()` | Handle both success and failure in one step |
| `orTimeout()` | Fail with TimeoutException after deadline |
| `completeOnTimeout()` | Return default value after deadline |

---

**Navigation**

[← Executor Framework](06-executor-framework.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Concurrent Collections](08-concurrent-collections.md)

---

*Last updated: 2026 | Java 21 LTS*
