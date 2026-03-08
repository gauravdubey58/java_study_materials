# 10 — 100 Interview Questions

[← Back to Index](./README.md)

> **Q1–Q50:** Moderate — concepts, mapping, session, transactions, caching  
> **Q51–Q100:** Tough — internals, edge cases, performance, architecture

---

## 🟡 Moderate Level (Q1–Q50)

---

**Q1. What is Hibernate and why is it used?**

Hibernate is an ORM (Object-Relational Mapping) framework that maps Java classes to database tables and automates SQL generation. It is used to eliminate repetitive JDBC boilerplate, bridge the object-relational impedance mismatch, provide database-independent HQL queries, and offer built-in features like caching, lazy loading, and dirty checking.

---

**Q2. What is the difference between Hibernate and JPA?**

JPA (Java Persistence API) is a specification — a set of interfaces and rules defined by the Jakarta EE standard. Hibernate is the most popular implementation of JPA. You can write code against JPA interfaces (`EntityManager`, `@Entity`) and swap implementations. Hibernate extends JPA with additional features (`@BatchSize`, `@DynamicUpdate`, `@NaturalId`, etc.) that aren't part of the standard.

---

**Q3. What is ORM? What problem does it solve?**

ORM (Object-Relational Mapping) bridges the mismatch between the object-oriented world (Java classes, inheritance, associations) and the relational world (tables, foreign keys, joins). Without ORM, developers manually convert objects to SQL and results back to objects. ORM automates this conversion, allowing developers to work with Java objects naturally.

---

**Q4. What are the four entity states in Hibernate?**

- **Transient** — New Java object; no ID; no associated Session; not in DB
- **Persistent** — Associated with an open Session; Hibernate tracks changes (dirty checking)
- **Detached** — Was persistent; Session has since been closed or entity was evicted; has ID but no tracking
- **Removed** — Persistent entity marked for deletion; DELETE will be executed at flush

---

**Q5. What is the difference between `session.get()` and `session.load()`?**

- `get()` — Hits the DB (or L1/L2 cache) immediately; returns `null` if entity not found
- `load()` — Returns a proxy without hitting the DB; the DB is only queried when a field (other than ID) is accessed; throws `ObjectNotFoundException` if not found on access

Use `get()` when you're unsure if the entity exists. Use `load()` when you know it exists and want lazy loading.

---

**Q6. What is the `SessionFactory`? Why is it heavyweight?**

`SessionFactory` is the thread-safe factory that creates `Session` objects. It is heavyweight because it reads all mapping metadata (XML or annotations), validates the schema, builds the class-to-table mapping, initializes the second-level cache, and creates the connection pool — all at startup. It should be built once per application and reused.

---

**Q7. What is dirty checking in Hibernate?**

Dirty checking is Hibernate's automatic detection of changes to persistent entities. When an entity enters the persistent state, Hibernate takes a snapshot of its field values. At flush time, it compares the current state with the snapshot. For any changed fields, it generates an UPDATE SQL statement. No explicit `update()` call is needed for persistent entities.

---

**Q8. What is the difference between `save()`, `persist()`, `update()`, and `merge()`?**

- `save()` — Hibernate; returns generated ID; immediately assigns identifier; can be called on transient or detached entities
- `persist()` — JPA; void return; only on transient entities; INSERT may be deferred to flush
- `update()` — Hibernate; reattaches a detached entity to the current session; throws exception if another instance of the same entity already exists in the session
- `merge()` — JPA-standard; copies detached entity state into a managed instance (or creates new managed one); safe if another instance exists; returns the managed copy

---

**Q9. What is HQL? How does it differ from SQL?**

HQL (Hibernate Query Language) is Hibernate's object-oriented query language. It operates on **entity names and field names** (not table/column names). Hibernate translates it to the appropriate SQL for the configured database. Key differences: HQL supports polymorphism (querying a base type returns all subtypes), HQL is database-independent, and HQL uses Java property paths instead of column names.

---

**Q10. What is the first-level cache?**

The first-level cache is a per-`Session` identity map. Every entity loaded or saved within a session is stored here, keyed by class + primary key. Subsequent `get()` calls for the same ID within the same session return the cached object without hitting the DB. It is always enabled and cannot be disabled.

---

**Q11. What is the second-level cache?**

The second-level cache is a `SessionFactory`-scoped, optional, shared cache for entity state (not instances). It persists across sessions. When an entity is loaded, Hibernate first checks L1, then L2, then the DB. Common providers: EhCache, Infinispan, Redis (via Redisson). Must be explicitly enabled and entities marked with `@Cache`.

---

**Q12. What is `@Version` and what problem does it solve?**

`@Version` marks a field (Integer, Long, or Timestamp) for optimistic locking. Hibernate automatically increments this field on every UPDATE. The UPDATE statement includes a `WHERE version = ?` clause. If another transaction modified the row between read and write (incrementing the version), the WHERE clause matches 0 rows and Hibernate throws `OptimisticLockException`. This prevents lost updates without holding DB locks.

---

**Q13. What are the inheritance mapping strategies in Hibernate?**

- **Single Table** — All subclasses in one table; discriminator column identifies type. Best performance; poor normalization.
- **Joined** — One table per class; subclass tables share the parent PK as FK; queries use JOINs. Best normalization; slower queries.
- **Table Per Concrete Class** — One table per concrete class; base class fields duplicated; polymorphic queries use UNION ALL.
- **MappedSuperclass** — No table for the base class; shared fields only; not a JPA entity; no polymorphic queries.

---

**Q14. What is the difference between `@OneToMany` and `@ManyToOne`?**

`@ManyToOne` is on the "many" side (child); the FK column lives in this table. It is the **owner** of the relationship. `@OneToMany` is on the "one" side (parent); it uses `mappedBy` to reference the `@ManyToOne` field name on the child, making it the **inverse** side with no FK column.

---

**Q15. What is `mappedBy` and why is it important?**

`mappedBy` on the inverse side of a bidirectional association tells Hibernate which field on the other side owns the relationship (holds the FK column). Without `mappedBy`, Hibernate would create two separate FK columns or a join table. `mappedBy` also means that changes made only on the inverse side are NOT persisted — changes must be made on the owner side.

---

**Q16. What is `CascadeType.ALL` and what operations does it cover?**

`CascadeType.ALL` is shorthand for `{PERSIST, MERGE, REMOVE, REFRESH, DETACH}`. When applied to an association, all of these operations are propagated from parent to child. For example, saving a parent also saves all children, and deleting the parent deletes all children.

---

**Q17. What is the difference between `orphanRemoval = true` and `CascadeType.REMOVE`?**

`CascadeType.REMOVE` propagates DELETE from parent to children. `orphanRemoval = true` does this AND also deletes a child if it is removed from the parent's collection (disassociated), even without deleting the parent. Use `orphanRemoval` for entities whose lifecycle is fully owned by the parent.

---

**Q18. What is lazy loading? What is `LazyInitializationException`?**

Lazy loading defers the loading of associated entities until they are first accessed. Hibernate returns a proxy object. `LazyInitializationException` is thrown when you try to access a lazily loaded association after the `Session` is closed, because there is no active session to execute the required SQL.

---

**Q19. What is the N+1 query problem?**

When loading N entities and accessing a lazy-loaded association on each, N additional queries are triggered — total N+1 queries. Example: loading 50 users then calling `user.getOrders()` in a loop = 1 + 50 = 51 queries.

---

**Q20. What are the ways to solve the N+1 problem?**

1. `JOIN FETCH` in HQL: `SELECT DISTINCT u FROM User u JOIN FETCH u.orders`
2. `@EntityGraph` to specify attributes to fetch
3. `@BatchSize(size = N)` on entity or collection
4. `@Fetch(FetchMode.SUBSELECT)` for collections
5. DTO projections — fetch only needed data in a single query

---

**Q21. What is eager loading? What are its drawbacks?**

Eager loading fetches associated entities immediately with the parent. Drawbacks: unnecessary data loaded even when the association isn't needed, potential cartesian product explosion with multiple eagerly-loaded collections, and worse performance for queries where the association is irrelevant.

---

**Q22. What is a Hibernate dialect?**

A dialect is a class that knows how to generate SQL for a specific database (MySQL, PostgreSQL, Oracle, etc.). It handles differences in data types, pagination syntax (`LIMIT`, `ROWNUM`, `FETCH FIRST`), sequence support, and function names. Example: `org.hibernate.dialect.MySQL8Dialect`.

---

**Q23. What is `hbm2ddl.auto`? What are its values?**

`hbm2ddl.auto` controls Hibernate's schema management:
- `none` — Do nothing (production default)
- `validate` — Validate schema matches entity mapping; fail if mismatch
- `update` — Modify schema to match entities (may not drop columns)
- `create` — Drop and recreate schema on startup
- `create-drop` — Create on startup, drop on `SessionFactory` close (for tests)

---

**Q24. What is the difference between the Criteria API and HQL?**

HQL is a string-based query language (type-unsafe, validated at runtime). The Criteria API is a programmatic, type-safe Java API that prevents SQL injection and catches errors at compile time. The Criteria API is more verbose but better for dynamic query construction.

---

**Q25. What is `@Embeddable` and when do you use it?**

`@Embeddable` marks a class as a value object whose fields are stored in the same table as the owning entity. It has no separate identity or table. Use it to group related columns (like an Address with street, city, zip) into a reusable class. Multiple instances can be embedded with different column names using `@AttributeOverride`.

---

**Q26. What is optimistic vs pessimistic locking?**

Optimistic locking (`@Version`) doesn't acquire DB locks; it detects conflicts at commit time. Good for low-contention scenarios. Pessimistic locking (`SELECT FOR UPDATE`) acquires exclusive DB locks at read time. Good for high-contention scenarios but reduces throughput.

---

**Q27. How does Hibernate handle transactions?**

Hibernate wraps the underlying JDBC or JTA transaction. With JDBC: `session.beginTransaction()` → `tx.commit()` / `tx.rollback()`. With Spring: `@Transactional` manages the transaction boundary automatically, using `SessionFactory.getCurrentSession()` to bind the session to the thread.

---

**Q28. What are transaction isolation levels?**

`READ_UNCOMMITTED` (can see uncommitted data), `READ_COMMITTED` (default for most DBs — sees only committed data), `REPEATABLE_READ` (same read returns same data within transaction), `SERIALIZABLE` (fully serialized — no phantom reads). Higher isolation = better consistency but lower concurrency.

---

**Q29. What is the Query Cache?**

The Query Cache stores the result set (list of IDs) of queries marked as cacheable. On subsequent executions with the same parameters, the IDs are returned from cache and the entities loaded from L2 cache. Useful for queries on rarely changing data. Invalidated whenever the queried table is modified.

---

**Q30. What is `@NaturalId` in Hibernate?**

`@NaturalId` marks a business identifier (like email or SSN) as a natural key. Hibernate provides optimized loading by natural ID (`session.byNaturalId(User.class).using("email", e).load()`) which leverages the second-level cache. Unlike `@Id`, it does not have to be the primary key.

---

**Q31. What is the difference between `@Table` and `@SecondaryTable`?**

`@Table` specifies the primary table for an entity. `@SecondaryTable` maps some fields of the entity to a second table. Both tables share the same primary key. Used to split one logical entity across two physical tables.

---

**Q32. What is `@Formula`?**

A Hibernate annotation that maps a field to a read-only SQL expression:

```java
@Formula("(SELECT COUNT(*) FROM orders o WHERE o.user_id = id)")
private int orderCount;
```

The SQL is included in the SELECT but not in INSERT/UPDATE. Useful for derived/calculated fields.

---

**Q33. What are named queries and what is the advantage of using them?**

Named queries are pre-defined HQL or SQL queries declared with `@NamedQuery` on an entity or in `hbm.xml`. They are parsed and validated at startup (not at query time), providing early detection of errors. They are also reusable across the codebase by name.

---

**Q34. What is `session.flush()` and when is it called automatically?**

`flush()` synchronizes the in-memory state of the session with the database — it sends all pending SQL to the DB but does NOT commit the transaction. It is called automatically before query execution (in AUTO mode) and at transaction commit.

---

**Q35. What is the difference between `session.clear()` and `session.evict()`?**

`evict(entity)` removes one specific entity from the L1 cache (it becomes detached). `clear()` removes all entities from the L1 cache — the entire session is cleared. Both are useful in batch processing to prevent out-of-memory issues.

---

**Q36. What is `@BatchSize` and how does it improve performance?**

`@BatchSize(size = N)` tells Hibernate to load up to N uninitialized proxies or collections in a single `IN` clause instead of N individual queries. For example, `@BatchSize(20)` on a collection reduces N+1 queries to roughly `N/20 + 1` queries.

---

**Q37. What is `@Immutable` in Hibernate?**

`@Immutable` marks an entity or collection as read-only. Hibernate will never generate UPDATE SQL for it. Useful for audit log tables, lookup tables, or any data that should never change after creation. Improves performance by skipping dirty checking.

---

**Q38. What is `@Where` in Hibernate?**

`@Where(clause = "deleted_at IS NULL")` adds a SQL fragment to all queries for that entity or collection. Commonly used for soft delete — ensuring deleted records are never returned. It also applies to HQL queries but NOT to native SQL queries.

---

**Q39. What is the difference between `@Entity` and `@MappedSuperclass`?**

`@Entity` creates a full JPA entity with its own table and can be queried directly. `@MappedSuperclass` is not an entity — it has no table and cannot be queried as a type. It only provides shared field/method inheritance to concrete subclasses. Cannot be used as a polymorphic target in associations.

---

**Q40. How do you map a composite primary key in JPA?**

Two approaches:
1. `@IdClass` — Declare the composite PK as a separate serializable class; annotate each PK field in the entity with `@Id`
2. `@EmbeddedId` — The composite PK class is `@Embeddable`; use `@EmbeddedId` on a single field in the entity

---

**Q41. What is `@DynamicUpdate`?**

A Hibernate annotation that generates UPDATE SQL with only the changed columns instead of all columns. Useful for wide tables where most columns are rarely changed — reduces unnecessary column updates and potential lock contention.

---

**Q42. What is `@SQLDelete`?**

`@SQLDelete(sql = "...")` overrides the default DELETE SQL Hibernate would generate. Commonly used for soft delete: instead of `DELETE FROM users WHERE id = ?`, it executes `UPDATE users SET deleted_at = NOW() WHERE id = ?`.

---

**Q43. What is a Hibernate Interceptor?**

An extension point for intercepting entity lifecycle events (load, save, update, delete). Implement `EmptyInterceptor` and override desired methods. Used for auditing, logging, or modifying data before persistence. Can be attached to `SessionFactory` (global) or individual `Session` (local).

---

**Q44. What is `@Filter` and how does it differ from `@Where`?**

`@Where` is a static, always-applied SQL condition. `@Filter` is configurable per session — you can enable or disable it and pass parameters. Useful for multi-tenancy (filter by tenant_id) or toggling visibility (show only active records when needed).

---

**Q45. What is the difference between JPQL `JOIN` and `JOIN FETCH`?**

`JOIN` creates a SQL JOIN but does NOT load the associated entity into memory (it's just used for filtering). `JOIN FETCH` creates a SQL JOIN AND loads the associated entity into memory as part of the parent. Use `JOIN FETCH` to solve N+1.

---

**Q46. What is `CacheConcurrencyStrategy.READ_WRITE` vs `READ_ONLY`?**

`READ_ONLY` is for data that never changes after creation — fastest, no synchronization needed. `READ_WRITE` uses soft locks to ensure cache consistency when data is updated — valid for frequently updated entities. `READ_ONLY` entities throw an exception if you try to update them.

---

**Q47. What is `FetchMode.SUBSELECT`?**

When a collection is marked with `@Fetch(FetchMode.SUBSELECT)`, Hibernate loads all collections for all loaded parent entities in a single SQL using a subquery (`WHERE user_id IN (SELECT id FROM users WHERE ...)`). Results in exactly 2 queries instead of N+1.

---

**Q48. What is the use of `@OrderBy` vs `@SortNatural`?**

`@OrderBy("field ASC")` adds an `ORDER BY` clause to the SQL query when loading the collection — sorting is done by the database. `@SortNatural` sorts the collection in Java memory using the element's natural ordering (`Comparable`). `@OrderBy` is more efficient (database-side); `@SortNatural` works in memory.

---

**Q49. What is `@ElementCollection`?**

Maps a collection of simple types (String, Integer, Enum) or `@Embeddable` objects to a separate collection table. The collection has no separate entity identity. The collection table has a FK pointing back to the owner entity. Lifecyle is fully owned by the parent.

---

**Q50. What is `GenerationType.SEQUENCE` vs `IDENTITY`?**

`SEQUENCE` uses a DB sequence object — Hibernate can pre-allocate a batch of IDs (via `allocationSize`) reducing round trips. Works before INSERT. `IDENTITY` relies on DB auto-increment — the ID is only available after INSERT. This prevents Hibernate from batching INSERTs (`hibernate.jdbc.batch_size` is ignored for IDENTITY-keyed entities).

---

## 🔴 Tough Level (Q51–Q100)

---

**Q51. Explain how Hibernate's dirty checking works internally.**

When an entity becomes persistent, Hibernate stores a **snapshot** of its field values (a `Object[]` array) in the persistence context. At flush time, `DefaultFlushEntityEventListener` iterates all persistent entities, compares current field values with their snapshots using deep equality, identifies changed fields, and generates UPDATE SQL for those fields. This comparison happens for every persistent entity at every flush — which is why limiting the number of persistent entities in a session matters for performance.

---

**Q52. What is the write-behind mechanism in Hibernate?**

Hibernate buffers SQL operations in memory and sends them to the DB at flush time (not when you call `save()`/`update()`). This write-behind allows Hibernate to reorder and batch SQL for efficiency — for example, all INSERTs first, then UPDATEs, then DELETEs; and similar statements can be batched in JDBC batch updates. The actual SQL execution order may differ from the order you called Hibernate methods.

---

**Q53. How does Hibernate manage the identity map (first-level cache) and what are its implications?**

The identity map (L1 cache) stores one object per entity type per primary key within a session. If you call `session.get(User.class, 1L)` twice in the same session, you get the exact same Java object reference. This guarantees identity consistency but has implications: long-running sessions or batch operations that load many entities can exhaust heap memory. The solution is `session.evict()` or `session.clear()` combined with `session.flush()` in loops.

---

**Q54. What is the difference between `load()` returning a proxy and an actual entity? When does the DB query fire?**

`load()` returns a CGLIB or Javassist subclass proxy. The proxy holds only the ID. No SQL fires until you call any method other than `getId()` (because the proxy already knows its ID). When any other method is called (e.g., `getName()`), the proxy's `LazyInitializer` fires a SELECT, populates the real entity's fields, and delegates the call. If the session is closed at this point, `LazyInitializationException` is thrown.

---

**Q55. Explain Hibernate's batch insert behavior and why `IDENTITY` generation breaks it.**

When `hibernate.jdbc.batch_size` is set (e.g., 30), Hibernate groups SQL statements and sends them as a JDBC batch for efficiency. However, `GenerationType.IDENTITY` requires the DB to auto-generate the ID. Hibernate must perform a single INSERT immediately after each `session.save()` call to get the generated ID back (for L1 cache keying). This makes batching impossible for IDENTITY-keyed entities. Use `SEQUENCE` with `allocationSize > 1` to enable batching.

---

**Q56. How does the second-level cache invalidation work with `READ_WRITE` strategy?**

When a `READ_WRITE` entity is updated, Hibernate places a soft lock in the cache entry before the DB update. Other sessions that try to read this entry during the update see the lock and fall back to the DB. After commit, the cache entry is replaced with the new value and the lock is removed. This two-phase approach ensures no session reads stale data during an update.

---

**Q57. What is the `UpdateTimestampsCache` and how does it affect the Query Cache?**

The `UpdateTimestampsCache` tracks the last modification timestamp for each table. When a query cache entry is requested, Hibernate compares the query's timestamp with the table's last-modified timestamp. If the table was modified more recently than the query was cached, the cache entry is invalidated. This is why any INSERT/UPDATE/DELETE on a table invalidates all query cache entries for that table.

---

**Q58. What is the cartesian product problem with multiple `JOIN FETCH` on a single query?**

When you `JOIN FETCH` two `@OneToMany` collections in a single HQL query:

```java
"SELECT u FROM User u JOIN FETCH u.orders JOIN FETCH u.tags"
```

The SQL result set is a cartesian product of orders × tags per user. If a user has 10 orders and 5 tags, you get 50 result rows for one user. Hibernate de-duplicates them in memory, but the network transfer and result set processing are expensive. Solution: use `DISTINCT` + `@QueryHints(HibernateHints.PASS_DISTINCT_THROUGH, false)`, or load each collection in separate queries.

---

**Q59. How do `@Filter` and multi-tenancy work together in Hibernate?**

For shared-schema multi-tenancy, define a `@FilterDef` with a `tenant_id` parameter and apply it to all tenant-specific entities. In a `SessionFactory` interceptor or request filter, enable the filter for every new session with the current tenant's ID. This ensures all queries automatically append `WHERE tenant_id = ?` without modifying any business logic.

---

**Q60. What is the open-session-in-view anti-pattern?**

Open-session-in-view keeps the Hibernate session open for the entire HTTP request, including the view rendering phase. This allows lazy loading to work in templates but is an anti-pattern because: it moves DB access out of the service/transactional layer (breaking concerns), queries are executed without transactional guarantees, it holds DB connections longer, and performance issues in the view layer are harder to diagnose. Use proper DTOs or eager loading in the service layer instead.

---

**Q61. Explain the difference between `@OneToOne` with `@MapsId` and a regular `@OneToOne` with `@JoinColumn`.**

With `@MapsId`, the child entity's PK is the same value as the parent's PK (shared primary key). The child table has no extra FK column — the PK IS the FK. With `@JoinColumn`, the child has its own PK and a separate FK column pointing to the parent. `@MapsId` guarantees a 1:1 relationship at the DB level through the PK constraint; `@JoinColumn` requires a UNIQUE constraint to enforce it.

---

**Q62. How does Hibernate's `@BatchSize` work internally for proxy initialization?**

When a proxy (lazy entity) is initialized, Hibernate checks the persistence context for other uninitialized proxies of the same type. It collects up to `batchSize` proxy IDs and fires a single `SELECT WHERE id IN (1, 2, ..., N)` to load them all. The batcher is keyed per entity type and session. This reduces N individual selects to `ceil(N / batchSize)` batched selects.

---

**Q63. What is `session.doWork()` and when would you use it?**

`doWork(Work work)` provides direct access to the underlying JDBC `Connection`. Use it when you need to execute raw JDBC operations within the current Hibernate transaction — setting transaction isolation, executing stored procedures, or calling JDBC-only features. It ensures the connection is obtained from the same pool/session context.

---

**Q64. What happens to the second-level cache during a `session.merge()` operation?**

`merge()` copies the detached entity's state into a new managed entity. If a L2 cache entry exists for the entity, Hibernate uses it as the baseline for the merge. After a successful commit, Hibernate updates the L2 cache entry with the new state. If another session invalidated the L2 entry between the merge and commit, Hibernate falls back to the DB.

---

**Q65. How do you implement soft delete in Hibernate and what are the limitations?**

Use `@SQLDelete` to override DELETE with `UPDATE ... SET deleted_at = NOW()`, and `@Where(clause = "deleted_at IS NULL")` to filter all queries. Limitations: `@Where` is bypassed by native queries; bulk DELETE HQL (`DELETE FROM Entity WHERE ...`) bypasses `@SQLDelete`; foreign key constraints may prevent the "delete" if dependent records exist; queries `SELECT ... WHERE id = ?` are also filtered, so you cannot load a soft-deleted record via normal means.

---

**Q66. What is the role of `@SequenceGenerator`'s `allocationSize` in performance?**

`allocationSize` defines how many sequence values Hibernate pre-allocates per DB round trip. With `allocationSize = 50`, Hibernate fetches one DB sequence value, then assigns IDs locally from `value` to `value + 49` without hitting the DB. This dramatically reduces sequence-related DB calls during bulk inserts. The DB sequence must use the same `INCREMENT BY` value to avoid gaps or collisions.

---

**Q67. Explain how Hibernate handles `@ManyToMany` with `Set` vs `List`.**

Using a `List` for `@ManyToMany` causes Hibernate to use "bag" semantics — it deletes all join table rows and re-inserts them on any collection change. This is extremely inefficient. Using a `Set` uses proper element comparison — only the changed entries are deleted/inserted. Always use `Set<T>` for `@ManyToMany` associations, and implement proper `equals()`/`hashCode()` based on a natural business key.

---

**Q68. What is `EntityGraph` (load graph vs fetch graph)?**

Both override which associations are eagerly loaded per query. `loadgraph` — specified paths are EAGER; unspecified paths use their declared fetch type. `fetchgraph` — specified paths are EAGER; ALL unspecified paths are LAZY regardless of their declared type. `fetchgraph` is more aggressive — it overrides even `FetchType.EAGER` declarations to lazy.

---

**Q69. How does optimistic locking fail silently and how do you prevent it?**

If `@Transactional(rollbackFor = ...)` is not properly configured, or if the `OptimisticLockException` is caught and swallowed somewhere in the call stack, the concurrent update is lost without notification. Prevent it by: propagating the exception to the caller, implementing retry logic, or using `@Retryable(OptimisticLockException.class)` (Spring Retry). Also, always annotate the exception handler to ensure the transaction is rolled back.

---

**Q70. What is the "DISTINCT" behavior difference between SQL and HQL with JOIN FETCH?**

In SQL, `DISTINCT` removes duplicate rows. In HQL with `JOIN FETCH`, `DISTINCT` tells Hibernate to deduplicate the result in Java memory (since the JOIN produces multiple rows per entity). Without `DISTINCT`, you get N copies of the same entity (one per joined row). In Hibernate 5.2+, use `query.setHint(QueryHints.PASS_DISTINCT_THROUGH, false)` with HQL `DISTINCT` to avoid passing `DISTINCT` to SQL (which would be redundant and potentially slower).

---

**Q71. Explain the difference between `CascadeType.PERSIST` and `CascadeType.MERGE`. When does each apply?**

`PERSIST` propagates when `session.persist()` / `save()` is called — used when inserting new entities. `MERGE` propagates when `session.merge()` is called — used when reattaching detached entities. They are distinct because you may want to save a new parent with new children (`PERSIST`) but not allow detached children to be merged back when only the parent is merged (`MERGE`). Incorrect cascade can lead to unintended INSERT or UPDATE of associated entities.

---

**Q72. How does Hibernate determine the SQL update order to avoid constraint violations?**

By default, Hibernate processes inserts first (parents before children, alphabetically by entity name), then updates, then deletes. This default ordering can cause FK constraint violations in some cases. Set `hibernate.order_inserts=true` and `hibernate.order_updates=true` to reorder SQL within each category. For FK dependencies that can't be resolved by ordering, use `@DeferredConstraint` or flush between saves.

---

**Q73. What is the persistence context and how does it differ from the EntityManager?**

The persistence context is the set of entity instances currently managed (the identity map + dirty tracking state). The `EntityManager` (or `Session`) is the API through which you interact with the persistence context. One `EntityManager` manages one persistence context. The persistence context is the in-memory first-level cache, while the `EntityManager` is the API object.

---

**Q74. How does `@Fetch(FetchMode.JOIN)` differ from `JOIN FETCH` in HQL?**

`@Fetch(FetchMode.JOIN)` on a mapping annotation affects all loading for that association regardless of how the entity was loaded — even `session.get()` will use a JOIN. `JOIN FETCH` in HQL is query-specific — it only affects that particular query execution. `@Fetch(FetchMode.JOIN)` is global for the association; `JOIN FETCH` is per-query. Prefer `JOIN FETCH` in queries for explicit control.

---

**Q75. What are the pros and cons of using `@MappedSuperclass` vs `@Inheritance(SINGLE_TABLE)` for shared fields?**

`@MappedSuperclass` has no shared table — each subclass has its own full table. Simpler, no polymorphic queries, best for unrelated entities sharing common fields (like `BaseEntity` with id/timestamps). `SINGLE_TABLE` puts all types in one table — supports polymorphic queries and associations to the base type, but requires nullable columns for subclass fields. Use `@MappedSuperclass` for shared fields across unrelated entities; use `SINGLE_TABLE` when you need polymorphism.

---

**Q76. How would you tune Hibernate for inserting 1 million records efficiently?**

1. Use `SEQUENCE` generator with `allocationSize` matching `jdbc.batch_size`
2. Set `hibernate.jdbc.batch_size = 50`
3. Set `hibernate.order_inserts = true`
4. Use `StatelessSession` (bypasses L1 cache, dirty checking, interceptors)
5. Flush and clear every `batch_size` rows
6. Disable L2 cache for the import
7. Use `hibernate.jdbc.batch_versioned_data = true`
8. Run in a single transaction (avoid per-row commit overhead)

---

**Q77. What is `StatelessSession` and when should you use it?**

`StatelessSession` is a stripped-down Hibernate session that bypasses the first-level cache, dirty checking, lifecycle callbacks, and interceptors. It fires SQL immediately without buffering. Use it for bulk ETL operations where performance is critical and you don't need entity tracking. It has no cascade, no lazy loading, and no second-level cache interaction.

---

**Q78. What is the `ScrollableResults` API and when is it useful?**

`ScrollableResults` is Hibernate's database cursor wrapper. Instead of loading all results into a `List`, it allows you to iterate through results one-by-one (or in batches), keeping only the current row in memory. Useful for processing very large result sets without loading them all into heap. Requires `ScrollMode.FORWARD_ONLY` for most databases.

---

**Q79. How does Hibernate handle timestamp-based optimistic locking compared to version-based?**

With `@Version` using a `Timestamp`, Hibernate compares the timestamp in the WHERE clause instead of an integer counter. The risk: timestamps have limited precision (milliseconds), so two concurrent transactions within the same millisecond may not detect the conflict. Integer `@Version` is always strictly incremented and has no such precision issue. Prefer integer/long versions.

---

**Q80. What is the N+1 problem with `@OneToOne(fetch = LAZY)`?**

Even when declared `LAZY`, `@OneToOne` on the non-owning side (inverse, with `mappedBy`) cannot be proxied by Hibernate without bytecode enhancement. Hibernate must issue a SELECT to determine if the associated entity exists (NULL or not) — otherwise it cannot decide whether to return `null` or a proxy. This causes N queries for N parents. Solutions: bytecode enhancement, `@LazyToOne(LazyToOneOption.NO_PROXY)`, restructure to `@ManyToOne` (owner side), or fetch with `JOIN FETCH`.

---

**Q81. What is the difference between `session.lock()` and `session.buildLockRequest()`?**

`session.lock(entity, LockMode.PESSIMISTIC_WRITE)` acquires a lock on an already-loaded entity (reissues SQL with FOR UPDATE). `buildLockRequest(LockOptions)` is the newer fluent API for the same purpose, supporting more options like timeout and scope. Both require the entity to be in the persistent state.

---

**Q82. How do you map a legacy schema where column names don't follow Java naming conventions?**

Use `@Column(name = "LEGACY_COL_NAME")` for individual fields, `@Table(name = "LEGACY_TABLE")` for the entity, and globally configure a `PhysicalNamingStrategy` to apply transformations (e.g., convert camelCase to UPPER_UNDERSCORE). Hibernate 5 provides `SpringPhysicalNamingStrategy` (via Spring Boot) and `PhysicalNamingStrategyStandardImpl`.

---

**Q83. Explain Hibernate's `@Any` mapping.**

`@Any` maps a polymorphic association where the target entity type is not fixed — it can reference any entity class. The actual type is stored in a discriminator column alongside the FK. Useful for audit logs or tagging systems where a single table can reference multiple entity types.

```java
@Any(metaColumn = @Column(name = "target_type"))
@AnyMetaDef(idType = "long", metaType = "string",
    metaValues = {
        @MetaValue(value = "USER",    targetEntity = User.class),
        @MetaValue(value = "PRODUCT", targetEntity = Product.class)
    })
@JoinColumn(name = "target_id")
private Object target;
```

---

**Q84. What is the `@Loader` annotation?**

`@Loader(namedQuery = "...")` replaces the default SELECT query Hibernate uses to load an entity with a named native SQL query. Useful when the default generated SELECT is not suitable (e.g., must call a stored procedure, use hints, or apply DB-specific optimizations).

---

**Q85. How does `@Cache` interact with `session.refresh()`?**

`session.refresh(entity)` always bypasses both L1 and L2 caches and reloads the entity state from the DB. After refreshing, the L2 cache entry is updated with the fresh data. Use `refresh()` when you need to ensure you have the absolute latest state from the DB, for example after an external process modifies the row.

---

**Q86. What is `FlushMode.COMMIT` and when is it beneficial?**

With `FlushMode.COMMIT`, Hibernate only flushes at transaction commit, not before query execution. Benefit: fewer round trips to the DB during a complex transaction with many queries. Risk: queries within the transaction may not see the changes made in the same transaction. Use it in read-heavy transactions or when you control query ordering and know changes don't affect your queries.

---

**Q87. How do you implement a read-only repository with Hibernate for maximum performance?**

1. `@Transactional(readOnly = true)` — tells Hibernate/Spring to skip dirty checking and flushing
2. `session.setDefaultReadOnly(true)` — marks all loaded entities as read-only (no snapshot taken)
3. `query.setReadOnly(true)` — marks all entities from this query as read-only
4. Consider `@Immutable` on the entity class
5. Use `StatelessSession` for bulk reads

These eliminate snapshot creation, dirty checking overhead, and unnecessary flush operations.

---

**Q88. Explain the difference between `@CollectionId` and `@OrderColumn`.**

`@CollectionId` (Hibernate-specific) adds a surrogate key column to the collection table, making it a `idbag` — the collection has a unique row identifier distinct from the element's identity. `@OrderColumn` adds a position/index column for ordered `List` collections — Hibernate uses it to maintain list order in the DB and retrieve elements in order.

---

**Q89. How does Hibernate's schema validation (`validate`) work?**

With `hbm2ddl.auto=validate`, on `SessionFactory` build, Hibernate queries the DB schema and compares every mapped table, column (type, length, nullable), and constraint against the mapping metadata. If any discrepancy is found (missing table, wrong type, missing column), it throws a `SchemaManagementException` and refuses to start. Useful in production to detect migration mistakes early.

---

**Q90. What is the `@Synchronize` annotation?**

`@Synchronize` on an entity tells Hibernate which tables must be flushed before queries on this entity are executed. Used with `@Subselect` (view-mapped entities) where the entity reads from a computed view that depends on other tables:

```java
@Entity
@Subselect("SELECT u.id, COUNT(o.id) as order_count FROM users u LEFT JOIN orders o ON u.id = o.user_id GROUP BY u.id")
@Synchronize({"users", "orders"})   // Flush these tables before querying this view
public class UserOrderStats { }
```

---

**Q91. How does the "bag" semantics differ from "set" and "list" in Hibernate collections?**

- **Bag** — Unordered, allows duplicates; mapped as `List` without `@OrderColumn`; Hibernate deletes all rows and re-inserts on modification (worst for performance)
- **Set** — Unordered, no duplicates; mapped as `Set`; uses element equality; efficient modification
- **List** — Ordered with position index (`@OrderColumn`); allows duplicates; efficient by index; modification requires shifting indices
- **idbag** — Bag with a surrogate PK column; allows efficient element removal without full delete/re-insert

---

**Q92. What is the purpose of `hibernate.connection.provider_class`?**

It specifies the connection pool implementation. Hibernate's built-in pool is for development only. In production, set it to a proper pool:
- `org.hibernate.hikaricp.internal.HikariCPConnectionProvider` — HikariCP
- `org.hibernate.connection.C3P0ConnectionProvider` — C3P0
- `org.hibernate.connection.ProxoolConnectionProvider` — Proxool
- Or use Spring's `DataSource` which supersedes this in Spring applications.

---

**Q93. What is `@SelectBeforeUpdate` and when is it necessary?**

`@SelectBeforeUpdate` tells Hibernate to perform a SELECT before any UPDATE to check if the data actually changed. Without it, Hibernate generates an UPDATE for any detached entity passed to `merge()` or `update()`, even if nothing changed (causing unnecessary DB writes and version increments). Use it for detached entities that may or may not have changed, to avoid phantom updates.

---

**Q94. How does Hibernate handle circular references in bidirectional relationships during serialization?**

Hibernate proxies and bidirectional relationships cause infinite recursion in JSON/XML serialization. Solutions:
- Jackson: `@JsonIgnore` on one side, `@JsonManagedReference`/`@JsonBackReference`, or `@JsonIdentityInfo`
- Use DTOs to break the cycle before serialization
- `@Transient` on the inverse side (loses the inverse navigation)
- Use projections that don't include the circular reference

---

**Q95. What is the performance impact of `@Inheritance(JOINED)` for deep hierarchies?**

Each level of inheritance adds one JOIN per query. A 3-level hierarchy (`Vehicle → Car → ElectricCar`) requires 2 LEFT OUTER JOINs per query. For polymorphic queries, Hibernate must LEFT OUTER JOIN to all subclass tables. With a very wide hierarchy (10+ subtypes), this generates enormous JOIN chains. Consider flattening deep hierarchies or using SINGLE_TABLE for frequently queried root types.

---

**Q96. What are Hibernate events and listeners? How do they differ from JPA lifecycle callbacks?**

Hibernate events (`PreInsertEvent`, `PostUpdateEvent`, `PreDeleteEvent`, etc.) are Hibernate-internal events fired before/after each persistence operation. You register an `EventListener` with the `EventListenerRegistry`. They are more granular than JPA callbacks (`@PrePersist` etc.) and provide access to the persistence context, session, and entity state. JPA callbacks are simpler (annotations on the entity), Hibernate event listeners are more powerful (global interceptors, access to change set).

---

**Q97. Explain the concept of "phantom flush" and how to diagnose it.**

A phantom flush is an unexpected flush that fires during a query, triggered by `FlushMode.AUTO` when Hibernate detects a dirty entity whose table might be queried. It causes unexpected SQL execution order and performance issues. Diagnose with `hibernate.show_sql=true` and by enabling flush event logging. Solutions: use `FlushMode.COMMIT` for read-heavy operations, limit the number of persistent entities in the session, or use read-only sessions.

---

**Q98. How does the `@Persister` annotation work?**

`@Persister(impl = CustomEntityPersister.class)` allows replacing Hibernate's default entity persistence strategy with a custom implementation. Useful for non-standard storage (e.g., storing entities in a key-value store while still using Hibernate's query capabilities) or implementing custom SQL generation logic. Advanced use case; rarely needed in standard applications.

---

**Q99. How would you handle a scenario where you need to bulk-delete millions of rows without loading them into memory?**

1. **HQL bulk delete:** `DELETE FROM User u WHERE u.status = 'INACTIVE'` — direct SQL, no entity loading, bypasses `@SQLDelete`
2. **Native SQL:** `session.createNativeQuery("DELETE FROM users WHERE status = 'INACTIVE'").executeUpdate()`
3. **Chunk deletion:** Delete in batches to avoid long lock holds and transaction log bloat
4. **TRUNCATE via native query:** For full table clears
5. Important: bulk deletes bypass L2 cache invalidation for specific entities — call `cache.evictEntityData(User.class)` after

---

**Q100. What are the most important Hibernate performance tuning knobs and what does each do?**

| Property | Effect |
|----------|--------|
| `hibernate.jdbc.batch_size = 30` | Batch INSERT/UPDATE/DELETE into single JDBC calls |
| `hibernate.order_inserts = true` | Group same-type INSERTs for batching |
| `hibernate.order_updates = true` | Group same-type UPDATEs for batching |
| `hibernate.cache.use_second_level_cache = true` | Enable entity caching across sessions |
| `hibernate.cache.use_query_cache = true` | Cache query result sets |
| `hibernate.generate_statistics = true` | Monitor cache hit ratios and query counts |
| `hibernate.default_batch_fetch_size = 16` | Global batch size for proxy initialization |
| `hibernate.jdbc.fetch_size = 50` | JDBC ResultSet fetch size (rows fetched per round trip) |
| `@DynamicUpdate` | UPDATE only changed columns |
| `@Immutable` | Skip dirty checking and updates for static data |
| `@BatchSize(N)` | Per-entity/collection batch fetching |
| `FlushMode.COMMIT` | Reduce mid-transaction flushes |
| `StatelessSession` | Bypass L1 cache for bulk operations |

---

[← Caching & Loading](./08_09_Caching_Loading.md) &nbsp;|&nbsp; [↑ Back to Index](./README.md)
