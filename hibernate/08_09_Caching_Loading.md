# 08 — Caching

[← Back to Index](./README.md)

---

## Table of Contents
- [Cache Architecture Overview](#cache-architecture-overview)
- [First-Level Cache (L1)](#first-level-cache-l1)
- [Second-Level Cache (L2)](#second-level-cache-l2)
- [Cache Concurrency Strategies](#cache-concurrency-strategies)
- [EhCache Configuration](#ehcache-configuration)
- [Query Cache](#query-cache)
- [Cache Regions](#cache-regions)
- [Cache Statistics & Monitoring](#cache-statistics--monitoring)
- [Redis as L2 Cache](#redis-as-l2-cache)

---

## Cache Architecture Overview

```
Session 1         Session 2
    │                 │
    ▼                 ▼
┌──────────┐     ┌──────────┐
│  L1 Cache│     │  L1 Cache│  ← Per Session, private, always on
│ (Session)│     │ (Session)│
└────┬─────┘     └────┬─────┘
     │                │
     └────────┬────────┘
              │
     ┌────────▼────────┐
     │  L2 Cache       │  ← Per SessionFactory, shared, optional
     │  (EhCache/Redis)│
     └────────┬────────┘
              │
     ┌────────▼────────┐
     │  Query Cache    │  ← Optional, stores result sets
     └────────┬────────┘
              │
              ▼
          Database
```

---

## First-Level Cache (L1)

**Scope:** Single `Session` (unit of work)  
**Enabled:** Always — cannot be disabled  
**Stores:** Entity instances by their primary key  

### How it Works

```java
try (Session session = sessionFactory.openSession()) {
    // First call → hits database
    User alice1 = session.get(User.class, 1L);
    // SELECT * FROM users WHERE id = 1

    // Second call in same session → returns from L1 cache (NO DB query!)
    User alice2 = session.get(User.class, 1L);

    System.out.println(alice1 == alice2);  // true — same object!

    // In a new session, L1 is empty again
    try (Session session2 = sessionFactory.openSession()) {
        User alice3 = session2.get(User.class, 1L);  // hits DB again
        System.out.println(alice1 == alice3);  // false — different sessions
    }
}
```

### L1 Cache Operations

```java
// Remove a specific entity from L1 cache (becomes detached)
session.evict(user);

// Clear entire L1 cache
session.clear();

// Check if entity is in L1 cache
boolean isManaged = session.contains(user);

// Force reload from DB — overwrites any pending changes
session.refresh(user);
```

### L1 Cache and Batch Processing

The L1 cache can become a memory problem during batch operations:

```java
// Problem: Processing 100,000 records — all end up in L1 cache
for (int i = 0; i < 100_000; i++) {
    Record r = new Record(i);
    session.save(r);
    // L1 cache grows until OutOfMemoryError
}

// Fix: Flush and clear periodically
ScrollableResults records = session.createQuery("FROM Record").scroll(ScrollMode.FORWARD_ONLY);
int batchSize = 50;
int count = 0;
while (records.next()) {
    process(records.get(0));
    if (++count % batchSize == 0) {
        session.flush();   // Send buffered SQL to DB
        session.clear();   // Release L1 cache memory
    }
}
```

---

## Second-Level Cache (L2)

**Scope:** `SessionFactory` — shared across all sessions  
**Enabled:** Optional — must be configured  
**Stores:** Entity state (not instances), keyed by entity class + ID  

### Enable L2 Cache

```xml
<!-- hibernate.cfg.xml -->
<property name="hibernate.cache.use_second_level_cache">true</property>
<property name="hibernate.cache.region.factory_class">
    org.hibernate.cache.ehcache.EhCacheRegionFactory
</property>
```

### Mark Entities as Cacheable

```java
// JPA standard
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Category {
    @Id @GeneratedValue private Long id;
    private String name;
}

// Cache collections too
@Entity
public class Product {
    @OneToMany(mappedBy = "product")
    @Cache(usage = CacheConcurrencyStrategy.READ_ONLY)
    private List<Tag> tags;
}
```

### L2 Cache Flow

```
session.get(Product.class, 1L)
            │
            ▼
    Check L1 cache → found? → return object
            │
            ▼ (not in L1)
    Check L2 cache → found? → hydrate into new instance, put in L1, return
            │
            ▼ (not in L2)
    Query Database → put in L2 → put in L1 → return object
```

---

## Cache Concurrency Strategies

| Strategy | Read | Write | Transactions | Use Case |
|----------|------|-------|-------------|---------|
| `READ_ONLY` | ✅ Fast | ❌ No updates | Not needed | Static data (countries, currencies) |
| `NONSTRICT_READ_WRITE` | ✅ | ✅ (no lock) | Optional | Rarely updated; slight stale risk ok |
| `READ_WRITE` | ✅ | ✅ (soft lock) | Required | Frequently read, occasionally updated |
| `TRANSACTIONAL` | ✅ | ✅ (XA lock) | JTA required | Fully serializable (EhCache + JTA only) |

```java
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)          // Static lookup tables
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)         // Most entities
@Cache(usage = CacheConcurrencyStrategy.NONSTRICT_READ_WRITE) // Slightly stale ok
@Cache(usage = CacheConcurrencyStrategy.TRANSACTIONAL)      // XA / JTA environment
```

---

## EhCache Configuration

```xml
<!-- ehcache.xml -->
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

    <!-- Default cache settings (applied when no region-specific config exists) -->
    <defaultCache
        maxEntriesLocalHeap="10000"
        eternal="false"
        timeToIdleSeconds="120"
        timeToLiveSeconds="300"
        memoryStoreEvictionPolicy="LRU"/>

    <!-- Per-entity region -->
    <cache name="com.example.entity.Category"
        maxEntriesLocalHeap="500"
        eternal="true"
        statistics="true"/>

    <cache name="com.example.entity.Product"
        maxEntriesLocalHeap="5000"
        eternal="false"
        timeToLiveSeconds="600"
        timeToIdleSeconds="300"
        memoryStoreEvictionPolicy="LFU"
        maxEntriesLocalDisk="50000"
        diskExpiryThreadIntervalSeconds="120"/>

    <!-- Collection cache region -->
    <cache name="com.example.entity.Product.tags"
        maxEntriesLocalHeap="2000"
        timeToLiveSeconds="300"/>

    <!-- Query cache -->
    <cache name="org.hibernate.cache.internal.StandardQueryCache"
        maxEntriesLocalHeap="5000"
        timeToLiveSeconds="120"/>

    <cache name="org.hibernate.cache.spi.UpdateTimestampsCache"
        maxEntriesLocalHeap="5000"
        eternal="true"/>

</ehcache>
```

### EhCache Eviction Policies

| Policy | Description |
|--------|-------------|
| `LRU` | Least Recently Used — evicts oldest-accessed entries |
| `LFU` | Least Frequently Used — evicts least-accessed entries |
| `FIFO` | First In First Out — evicts oldest-added entries |

---

## Query Cache

Caches the **result set** (list of IDs or scalar values) of frequently executed queries. The actual entities are fetched from L2 or DB.

### Enable

```xml
<property name="hibernate.cache.use_query_cache">true</property>
```

### Usage

```java
// Mark a query as cacheable
List<Category> categories = session.createQuery("FROM Category", Category.class)
    .setCacheable(true)
    .setCacheRegion("category-queries")   // Optional custom region
    .list();

// Named query with cache
@NamedQuery(
    name  = "Category.findAll",
    query = "SELECT c FROM Category c ORDER BY c.name",
    hints = @QueryHint(name = "org.hibernate.cacheable", value = "true")
)
```

### How Query Cache Works

```
First execution:
  Query result → [ID=1, ID=2, ID=3] stored in Query Cache
  Entities → stored in L2 cache (if cacheable)

Second execution (same params):
  Query Cache hit → returns [1, 2, 3]
  Load each entity from L2 cache (or DB if evicted)

Table update (users table):
  UpdateTimestampsCache marks "users" as dirty
  → All query cache entries for User queries are INVALIDATED
```

> ⚠️ The Query Cache is only effective for **frequently run queries with infrequently changing data**. For frequently updated tables, the query cache may cause more overhead than it saves.

---

## Cache Regions

Cache regions allow fine-grained control over what is cached and how.

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE, region = "shortLived")
public class Session { }

@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY, region = "staticData")
public class Country { }
```

### Programmatic Cache Management

```java
// Get cache region
Cache cache = sessionFactory.getCache();

// Evict a specific entity
cache.evictEntityData(User.class, userId);

// Evict all entities of a type
cache.evictEntityData(User.class);

// Evict a collection
cache.evictCollectionData(User.class.getName() + ".orders", userId);

// Evict all query caches
cache.evictDefaultQueryRegion();
cache.evictQueryRegion("category-queries");

// Clear everything
cache.evictAllRegions();
```

---

## Cache Statistics & Monitoring

```xml
<property name="hibernate.generate_statistics">true</property>
```

```java
Statistics stats = sessionFactory.getStatistics();

// L2 hit/miss rates
long l2Hits   = stats.getSecondLevelCacheHitCount();
long l2Misses = stats.getSecondLevelCacheMissCount();
long l2Puts   = stats.getSecondLevelCachePutCount();

// Query cache stats
long qHits   = stats.getQueryCacheHitCount();
long qMisses = stats.getQueryCacheMissCount();

// Entity stats
long loads   = stats.getEntityLoadCount();
long inserts = stats.getEntityInsertCount();
long updates = stats.getEntityUpdateCount();
long deletes = stats.getEntityDeleteCount();

// Connection stats
long transactions   = stats.getTransactionCount();
long connections    = stats.getConnectCount();
long sessionOpens   = stats.getSessionOpenCount();
long sessionCloses  = stats.getSessionCloseCount();

double hitRatio = (double) l2Hits / (l2Hits + l2Misses);
System.out.printf("L2 Cache Hit Ratio: %.2f%%%n", hitRatio * 100);
```

---

## Redis as L2 Cache

For distributed/clustered applications, use Redis or Infinispan instead of EhCache.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-hibernate-6</artifactId>
    <version>3.23.0</version>
</dependency>
```

```yaml
# application.yml (Spring Boot)
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          region:
            factory_class: org.redisson.hibernate.RedissonRegionFactory
          redisson:
            config: classpath:redisson.yaml
```

```yaml
# redisson.yaml
singleServerConfig:
  address: "redis://localhost:6379"
  connectionPoolSize: 64
  idleConnectionTimeout: 10000
  connectTimeout: 10000
```

---

[← Back to Index](./README.md)

---
---

# 09 — Loading Strategies

[← Back to Index](./README.md)

---

## Table of Contents
- [Lazy Loading](#lazy-loading)
- [Eager Loading](#eager-loading)
- [Fetch Strategies](#fetch-strategies)
- [The N+1 Query Problem](#the-n1-query-problem)
- [Solutions to N+1](#solutions-to-n1)
- [Batch Fetching](#batch-fetching)
- [Subselect Fetching](#subselect-fetching)
- [Entity Graphs](#entity-graphs)

---

## Lazy Loading

**Definition:** Associated entities or collections are NOT loaded immediately. Hibernate returns a proxy or empty collection. The real data is fetched from the DB only when a method on the proxy is first accessed.

**Default for:** `@OneToMany`, `@ManyToMany`

```java
@Entity
public class Author {
    @Id private Long id;
    private String name;

    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY) // default
    private List<Book> books;   // This is a Hibernate PersistentList proxy
}

// Usage
try (Session session = sessionFactory.openSession()) {
    Author author = session.get(Author.class, 1L);
    // SELECT * FROM authors WHERE id = 1   ← only this query

    // books not loaded yet...
    List<Book> books = author.getBooks();  // Still a proxy

    int count = books.size();
    // NOW the query fires: SELECT * FROM books WHERE author_id = 1
}
```

### LazyInitializationException

```java
Author author;
try (Session session = sessionFactory.openSession()) {
    author = session.get(Author.class, 1L);
}  // ← session closed here

// DANGER: accessing lazy collection outside session
author.getBooks().size();  // ❌ LazyInitializationException!
                           //    Cannot initialize proxy — no Session
```

**Solutions:**
1. Keep the session open while accessing lazy data
2. Use `JOIN FETCH` to load data within the session
3. Use DTOs — fetch all needed data in service layer before returning
4. Use `@Transactional` to keep session open in service layer

---

## Eager Loading

**Definition:** Associated entities or collections are loaded immediately together with the parent entity, using a JOIN or a secondary SELECT.

**Default for:** `@ManyToOne`, `@OneToOne`

```java
@ManyToOne(fetch = FetchType.EAGER)  // Default — fetches Category in same SQL
private Category category;

@OneToMany(fetch = FetchType.EAGER)  // Override default — fetches all orders immediately
private List<Order> orders;
```

### Problems with Eager Loading

```java
// Problem 1: Cartesian Product
// Eager @OneToMany fires a JOIN — if 1 user has 100 orders → 100 rows returned
// for one user (each with identical user columns)

// Problem 2: Always loaded even when not needed
// Finding a user for authentication? Still JOINs all their orders!

// Problem 3: N+1 in reverse
// "Eager" DOES NOT guarantee a single query — Hibernate may choose secondary SELECTs
```

> ⚠️ **Best practice:** Mark ALL associations as `FetchType.LAZY`. Then selectively fetch eagerly using `JOIN FETCH` in specific queries where the association is needed.

---

## Fetch Strategies

`FetchType` (JPA) controls WHEN to load. `FetchMode` (Hibernate) controls HOW to load.

### FetchMode Options

| Mode | How Hibernate Loads | SQL |
|------|--------------------|----|
| `SELECT` | Separate SELECT statement per collection | `SELECT * FROM orders WHERE user_id = ?` (N times) |
| `JOIN` | LEFT OUTER JOIN in the main query | `SELECT u.*, o.* FROM users u LEFT JOIN orders o ON...` |
| `SUBSELECT` | One SELECT with a subquery for all collections | `SELECT * FROM orders WHERE user_id IN (SELECT id FROM users WHERE...)` |
| `BATCH` | Batches N proxy initializations into one IN clause | `SELECT * FROM orders WHERE user_id IN (1, 2, 3, ..., N)` |

```java
@OneToMany
@Fetch(FetchMode.JOIN)       // Always join — regardless of FetchType
private List<Role> roles;

@OneToMany
@Fetch(FetchMode.SUBSELECT)  // Load all collections in one subselect
private List<Permission> permissions;

@ManyToOne
@Fetch(FetchMode.SELECT)     // Separate SELECT (default for lazy)
private Category category;
```

---

## The N+1 Query Problem

The most common Hibernate performance issue.

### The Problem

```java
// Load 5 users
List<User> users = session.createQuery("FROM User", User.class).list();
// Query 1: SELECT * FROM users   → returns 5 users

for (User user : users) {
    System.out.println(user.getOrders().size());
    // Query 2-6: SELECT * FROM orders WHERE user_id = ?  (one per user!)
}

// Total: 1 + 5 = 6 queries  (with N users, it's N+1)
```

---

## Solutions to N+1

### Solution 1: JOIN FETCH in HQL

```java
List<User> users = session.createQuery(
    "SELECT DISTINCT u FROM User u JOIN FETCH u.orders", User.class
).list();
// Single query: SELECT DISTINCT u.*, o.* FROM users u
//               INNER JOIN orders o ON u.id = o.user_id

// Multiple collections? Use two separate queries (cartesian product risk)
List<User> users = session.createQuery(
    "SELECT DISTINCT u FROM User u JOIN FETCH u.orders", User.class
).list();
session.createQuery(
    "SELECT DISTINCT u FROM User u JOIN FETCH u.tags WHERE u IN :users", User.class
).setParameter("users", users).list();
```

### Solution 2: `@EntityGraph`

```java
// Define on entity
@NamedEntityGraph(
    name = "User.withOrdersAndAddress",
    attributeNodes = {
        @NamedAttributeNode("orders"),
        @NamedAttributeNode("address")
    }
)
@Entity
public class User { ... }

// Use on query
EntityGraph<?> graph = entityManager.getEntityGraph("User.withOrdersAndAddress");
List<User> users = entityManager.createQuery("FROM User", User.class)
    .setHint("javax.persistence.loadgraph", graph)
    .getResultList();
```

### Solution 3: `@BatchSize`

```java
@Entity
@BatchSize(size = 20)       // Loads up to 20 User proxies in one IN clause
public class User { }

// On collection
@OneToMany(mappedBy = "user")
@BatchSize(size = 20)       // When loading any User's orders, load 20 users' orders at once
private List<Order> orders;

// What happens:
// 1. SELECT * FROM users  → 5 users, orders are proxies
// 2. user1.getOrders() triggers: SELECT * FROM orders WHERE user_id IN (1, 2, 3, 4, 5)
//    All 5 users' orders loaded in ONE query instead of 5!
```

### Solution 4: DTO Projection

Fetch only the data you need — no entity loading at all:

```java
public class UserOrderCount {
    private Long userId;
    private String userName;
    private Long orderCount;

    public UserOrderCount(Long userId, String userName, Long orderCount) { ... }
}

List<UserOrderCount> result = session.createQuery(
    "SELECT new com.example.dto.UserOrderCount(u.id, u.name, COUNT(o)) " +
    "FROM User u LEFT JOIN u.orders o GROUP BY u.id, u.name",
    UserOrderCount.class
).list();
// Single query — no N+1 possible
```

### Solution 5: Global Batch Size in Configuration

```xml
<!-- hibernate.cfg.xml — applies globally -->
<property name="hibernate.default_batch_fetch_size">16</property>
```

---

## Batch Fetching

Batch fetching is a middle ground between per-row SELECT and full JOIN FETCH:

```java
@Entity
@BatchSize(size = 10)
public class Product { }
```

```java
// Load 30 order items
List<OrderItem> items = session.createQuery("FROM OrderItem", OrderItem.class).list();
// Query 1: SELECT * FROM order_items

// Access products (lazy, @BatchSize(10))
items.get(0).getProduct().getName();
// Query 2: SELECT * FROM products WHERE id IN (1, 2, 3, ..., 10)  — 10 at once!

items.get(10).getProduct().getName();
// Query 3: SELECT * FROM products WHERE id IN (11, 12, ..., 20)

items.get(20).getProduct().getName();
// Query 4: SELECT * FROM products WHERE id IN (21, 22, ..., 30)

// Total: 4 queries instead of 31 (1 + 30)
```

---

## Subselect Fetching

Loads ALL instances of a collection in a single subquery:

```java
@Entity
public class Department {
    @OneToMany(mappedBy = "department")
    @Fetch(FetchMode.SUBSELECT)
    private List<Employee> employees;
}
```

```sql
-- Query 1: Load departments
SELECT * FROM departments WHERE active = 1;

-- Query 2: Load ALL employees for ALL loaded departments in one shot
SELECT * FROM employees
WHERE dept_id IN (
    SELECT id FROM departments WHERE active = 1
);
-- Total: 2 queries regardless of number of departments
```

| Strategy | Queries (for N parents) | Tradeoff |
|----------|------------------------|---------|
| Lazy SELECT | N+1 | Worst — avoid |
| `@BatchSize(10)` | 1 + ceil(N/10) | Good balance |
| SUBSELECT | 2 | Good — entire result set |
| JOIN FETCH | 1 | Best — but cartesian product risk |
| DTO Projection | 1 | Best when you don't need full entities |

---

## Entity Graphs

Entity graphs provide a flexible, declarative way to specify what to fetch per query.

```java
// Load graph (overrides ALL fetch settings — lazy becomes eager)
EntityGraph<User> loadGraph = entityManager.createEntityGraph(User.class);
loadGraph.addAttributeNodes("orders");
Subgraph<Order> orderGraph = loadGraph.addSubgraph("orders");
orderGraph.addAttributeNodes("items");

List<User> users = entityManager.createQuery("FROM User", User.class)
    .setHint("javax.persistence.loadgraph", loadGraph)
    .getResultList();

// Fetch graph (overrides only what's specified — rest stays as declared)
.setHint("javax.persistence.fetchgraph", loadGraph)
```

### Difference: loadgraph vs fetchgraph

| Hint | Behavior |
|------|---------|
| `loadgraph` | Specified attributes → EAGER; Unspecified → use their declared fetch type |
| `fetchgraph` | Specified attributes → EAGER; Unspecified → LAZY (regardless of declared type) |

---

[← Session & Transaction](./06_07_Session_Transaction.md) &nbsp;|&nbsp; [Next → Interview Questions](./10_Interview_Questions.md)
