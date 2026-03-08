# 06 — Session Management

[← Back to Index](./README.md)

---

## Table of Contents
- [Session vs SessionFactory vs EntityManager](#session-vs-sessionfactory-vs-entitymanager)
- [Entity States & Lifecycle](#entity-states--lifecycle)
- [Core Session API](#core-session-api)
- [SessionFactory Configuration](#sessionfactory-configuration)
- [Session Patterns](#session-patterns)
- [Flush Modes](#flush-modes)
- [Evict, Clear & Close](#evict-clear--close)

---

## Session vs SessionFactory vs EntityManager

| | `SessionFactory` | `Session` | `EntityManager` |
|-|-----------------|-----------|-----------------|
| JPA equivalent | `EntityManagerFactory` | `EntityManager` | (is the JPA API) |
| Thread-safe | ✅ Yes | ❌ No | ❌ No |
| Scope | Application lifetime | Unit of work (request/transaction) | Unit of work |
| Creation cost | High (built once) | Low | Low |
| Holds L2 cache | ✅ Yes | No | No |
| HQL support | — | ✅ Yes | Via JPQL |

---

## Entity States & Lifecycle

Every Hibernate entity is always in one of four states:

```
new User()
    │
    │  session.save() / session.persist()
    ▼
┌──────────┐                    ┌──────────┐
│ TRANSIENT│                    │ DETACHED │
│ (new, no │                    │ (session │
│  DB row, │                    │  closed, │
│  no ID)  │                    │  has ID) │
└────┬─────┘                    └────┬─────┘
     │                               │
     │ save/persist                  │ merge/update/lock
     ▼                               ▼
┌─────────────────────────────────────────┐
│              PERSISTENT                  │
│  (associated with open Session)          │
│  (changes tracked — dirty checking)      │
└───────────────────┬─────────────────────┘
                    │
         session.delete()   session.close() / evict()
                    │                │
                    ▼                ▼
              ┌──────────┐    ┌──────────┐
              │ REMOVED  │    │ DETACHED │
              │ (DELETE  │    │          │
              │ pending) │    └──────────┘
              └──────────┘
```

### State Transitions

```java
// TRANSIENT → PERSISTENT
User user = new User("Alice");           // Transient
session.save(user);                      // Persistent (INSERT pending)

// PERSISTENT → DETACHED
session.close();                         // All entities become Detached
session.evict(user);                     // Specific entity Detached

// DETACHED → PERSISTENT
Session s2 = sf.openSession();
User managed = (User) s2.merge(user);    // Returns a NEW managed copy
// OR
s2.update(user);                         // Reattaches (if not already in session)

// PERSISTENT → REMOVED
session.delete(user);                    // DELETE pending at flush
```

---

## Core Session API

### Saving / Persisting

```java
// save() — Hibernate; returns the generated ID; triggers INSERT immediately or at flush
Serializable id = session.save(entity);

// persist() — JPA standard; void return; no guarantee of immediate SQL
session.persist(entity);

// saveOrUpdate() — INSERT if transient, UPDATE if detached
session.saveOrUpdate(entity);

// merge() — copies state of detached entity; returns new managed instance
User managed = session.merge(detachedUser);
```

### Loading / Fetching

```java
// get() — returns null if not found; always hits DB or L1 cache
User user = session.get(User.class, 1L);

// load() — returns proxy (lazy); throws ObjectNotFoundException if not found when accessed
User proxy = session.load(User.class, 1L);

// find() — JPA equivalent of get(); returns null if not found
User user = entityManager.find(User.class, 1L);

// getReference() — JPA equivalent of load(); returns proxy
User proxy = entityManager.getReference(User.class, 1L);

// byId() — Hibernate 5+ fluent API
User user = session.byId(User.class).load(1L);

// byNaturalId() — load by natural business key (uses L2 cache)
User user = session.byNaturalId(User.class)
    .using("email", "alice@example.com")
    .load();
```

### Updating

```java
// No explicit call needed for persistent entities — dirty checking handles it:
User user = session.get(User.class, 1L);
user.setName("Alice Updated");          // Just modify the field
// At flush time → UPDATE users SET name = ? WHERE id = ?

// For detached entities:
session.update(user);       // Reattach (Hibernate)
session.merge(user);        // Copy state (JPA-safe)
```

### Deleting

```java
// delete() — marks entity for deletion
session.delete(entity);

// remove() — JPA standard
entityManager.remove(entity);

// Delete by query — bulk delete
session.createQuery("DELETE FROM User u WHERE u.active = false")
       .executeUpdate();
```

### Querying

```java
// HQL
List<User> users = session.createQuery(
    "FROM User u WHERE u.status = :status", User.class)
    .setParameter("status", Status.ACTIVE)
    .setFirstResult(0)
    .setMaxResults(20)
    .list();

// Named Query
List<User> users = session.getNamedQuery("User.findByEmail")
    .setParameter("email", "alice@example.com")
    .list();

// Native SQL
List<User> users = session.createNativeQuery(
    "SELECT * FROM users WHERE city = :city", User.class)
    .setParameter("city", "London")
    .list();

// Criteria API
CriteriaBuilder cb = session.getCriteriaBuilder();
CriteriaQuery<User> cq = cb.createQuery(User.class);
Root<User> root = cq.from(User.class);
cq.select(root).where(cb.equal(root.get("status"), Status.ACTIVE));
List<User> users = session.createQuery(cq).getResultList();
```

---

## SessionFactory Configuration

```java
// Standard bootstrap
public class HibernateUtil {

    private static final SessionFactory SESSION_FACTORY;

    static {
        try {
            SESSION_FACTORY = new Configuration()
                .configure()              // reads hibernate.cfg.xml
                .buildSessionFactory();
        } catch (Throwable ex) {
            throw new ExceptionInInitializerError(ex);
        }
    }

    public static SessionFactory getSessionFactory() {
        return SESSION_FACTORY;
    }

    public static void shutdown() {
        SESSION_FACTORY.close();
    }
}

// With Spring Boot (auto-configured — just inject)
@Autowired
private SessionFactory sessionFactory;  // or EntityManagerFactory

// Unwrap to get Hibernate Session from JPA EntityManager
Session session = entityManager.unwrap(Session.class);
```

---

## Session Patterns

### Pattern 1: Session per Request (Web)

```java
// Filter — open session at start of request, close at end
public class HibernateSessionFilter implements Filter {

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        Session session = HibernateUtil.getSessionFactory().openSession();
        try {
            session.beginTransaction();
            chain.doFilter(req, res);
            session.getTransaction().commit();
        } catch (Exception e) {
            session.getTransaction().rollback();
            throw e;
        } finally {
            session.close();
        }
    }
}
```

### Pattern 2: Session per Operation (Short-lived)

```java
public User findById(Long id) {
    try (Session session = sessionFactory.openSession()) {
        return session.get(User.class, id);
    }
}
```

### Pattern 3: Current Session (Thread-bound)

```java
// In hibernate.cfg.xml: <property name="hibernate.current_session_context_class">thread</property>
Session session = sessionFactory.getCurrentSession();
// Automatically bound to current thread; closed when transaction ends
```

---

## Flush Modes

| Mode | When SQL is sent to DB |
|------|----------------------|
| `AUTO` (default) | Before queries (to avoid stale reads) + at transaction commit |
| `COMMIT` | Only at transaction commit — may miss changes in same transaction query |
| `ALWAYS` | Before every HQL/Criteria query execution |
| `MANUAL` | Only when `session.flush()` is called explicitly |

```java
session.setHibernateFlushMode(FlushMode.COMMIT);  // Hibernate API
entityManager.setFlushMode(FlushModeType.COMMIT);  // JPA API

// Manual flush
session.flush();   // Force sync to DB within current transaction
```

---

## Evict, Clear & Close

```java
// Evict a specific entity from L1 cache
session.evict(user);          // Hibernate
entityManager.detach(user);   // JPA

// Clear entire L1 cache — all entities become detached
session.clear();

// Check if entity is in session
boolean managed = session.contains(user);

// Refresh from DB — overwrites in-memory changes
session.refresh(user);

// Close session — releases connection, all entities detached
session.close();
```

---

[← Back to Index](./README.md)

---
---

# 07 — Transaction Management

[← Back to Index](./README.md)

---

## Table of Contents
- [What is a Transaction?](#what-is-a-transaction)
- [ACID Properties](#acid-properties)
- [Hibernate Transaction API](#hibernate-transaction-api)
- [Transaction Isolation Levels](#transaction-isolation-levels)
- [Concurrency Issues](#concurrency-issues)
- [Optimistic Locking](#optimistic-locking)
- [Pessimistic Locking](#pessimistic-locking)
- [JTA vs JDBC Transactions](#jta-vs-jdbc-transactions)
- [Spring + Hibernate Transactions](#spring--hibernate-transactions)

---

## What is a Transaction?

A transaction is a unit of work that must execute completely or not at all. In Hibernate, a transaction wraps a sequence of read/write operations against the database.

```java
Transaction tx = null;
try (Session session = sessionFactory.openSession()) {
    tx = session.beginTransaction();

    // all your work here
    User user = session.get(User.class, id);
    user.setBalance(user.getBalance() - amount);

    tx.commit();         // All changes committed atomically
} catch (Exception e) {
    if (tx != null) tx.rollback();    // All changes rolled back
    throw e;
}
```

---

## ACID Properties

| Property | Meaning |
|----------|---------|
| **Atomicity** | All operations succeed or all are rolled back — no partial commits |
| **Consistency** | DB moves from one valid state to another — constraints always satisfied |
| **Isolation** | Concurrent transactions see a consistent view of data |
| **Durability** | Committed transactions survive system failures |

---

## Hibernate Transaction API

```java
// Begin
Transaction tx = session.beginTransaction();

// Check status
boolean isActive = tx.isActive();

// Commit
tx.commit();

// Rollback
tx.rollback();

// Set timeout (seconds)
tx.setTimeout(30);

// Mark for rollback only (cannot commit)
tx.markRollbackOnly();
boolean willRollback = tx.getRollbackOnly();
```

### Full Pattern with Exception Handling

```java
public void transferFunds(Long fromId, Long toId, BigDecimal amount) {
    Transaction tx = null;
    try (Session session = sessionFactory.openSession()) {
        tx = session.beginTransaction();

        Account from = session.get(Account.class, fromId);
        Account to   = session.get(Account.class, toId);

        if (from.getBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException();
        }

        from.setBalance(from.getBalance().subtract(amount));
        to.setBalance(to.getBalance().add(amount));

        // Dirty checking generates UPDATE automatically at commit
        tx.commit();

    } catch (InsufficientFundsException e) {
        if (tx != null) tx.rollback();
        throw e;
    } catch (Exception e) {
        if (tx != null) tx.rollback();
        throw new RuntimeException("Transfer failed", e);
    }
}
```

---

## Transaction Isolation Levels

Isolation levels control what a transaction can see from concurrent transactions:

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|:----------:|:-------------------:|:------------:|
| `READ_UNCOMMITTED` | ✅ Yes | ✅ Yes | ✅ Yes |
| `READ_COMMITTED` | ❌ No | ✅ Yes | ✅ Yes |
| `REPEATABLE_READ` | ❌ No | ❌ No | ✅ Yes |
| `SERIALIZABLE` | ❌ No | ❌ No | ❌ No |

```java
// Set isolation in hibernate.cfg.xml
// <property name="hibernate.connection.isolation">2</property>
// 1=READ_UNCOMMITTED, 2=READ_COMMITTED, 4=REPEATABLE_READ, 8=SERIALIZABLE

// Per session (using JDBC Connection)
session.doWork(connection -> {
    connection.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);
});
```

---

## Concurrency Issues

### Dirty Read
Transaction A reads uncommitted data from Transaction B. B later rolls back. A has read invalid data.

### Non-Repeatable Read
Transaction A reads a row twice. Between reads, Transaction B modifies and commits that row. A gets different values each time.

### Phantom Read
Transaction A queries for rows matching a condition. Between two runs, Transaction B inserts new rows that match. A sees different numbers of rows.

### Lost Update
Both Transaction A and B read the same row. Both modify it and commit. One write overwrites the other.

---

## Optimistic Locking

**Assume conflicts are rare.** No DB lock held. Uses a `@Version` field to detect conflicts at commit time.

```java
@Entity
public class Product {
    @Id @GeneratedValue private Long id;
    private String name;
    private int stock;

    @Version
    private Long version;   // Hibernate increments this on every UPDATE
}

// How it works:
// Thread 1: SELECT product → version = 3
// Thread 2: SELECT product → version = 3
// Thread 1: UPDATE product SET stock=9, version=4 WHERE id=? AND version=3  ✅ (1 row updated)
// Thread 2: UPDATE product SET stock=8, version=4 WHERE id=? AND version=3  ❌ (0 rows — version mismatch)
//           → throws OptimisticLockException
```

```java
// Handling OptimisticLockException
try {
    Product p = session.get(Product.class, productId);
    p.setStock(p.getStock() - quantity);
    tx.commit();
} catch (OptimisticLockException e) {
    tx.rollback();
    // Retry logic or inform user
}
```

---

## Pessimistic Locking

**Assume conflicts are likely.** Acquires a DB-level lock. Other transactions cannot read/write the locked row until the lock is released.

```java
// Acquire a WRITE lock (SELECT ... FOR UPDATE)
Product product = session.get(Product.class, id, LockMode.PESSIMISTIC_WRITE);

// Via query
Product product = session.createQuery("FROM Product p WHERE p.id = :id", Product.class)
    .setParameter("id", id)
    .setLockMode("p", LockMode.PESSIMISTIC_WRITE)
    .uniqueResult();

// JPA-style
Product product = entityManager.find(Product.class, id,
    LockModeType.PESSIMISTIC_WRITE);
```

### Lock Modes

| LockMode | SQL | Description |
|----------|-----|-------------|
| `NONE` | None | No lock (default) |
| `READ` | (version check) | Verify version hasn't changed at commit |
| `OPTIMISTIC` | (version check) | Same as READ |
| `OPTIMISTIC_FORCE_INCREMENT` | (version++) | Force version increment even for read-only access |
| `PESSIMISTIC_READ` | `SELECT ... FOR SHARE` | Others can read but not write |
| `PESSIMISTIC_WRITE` | `SELECT ... FOR UPDATE` | Exclusive — others cannot read or write |
| `PESSIMISTIC_FORCE_INCREMENT` | `SELECT ... FOR UPDATE` + version++ | Exclusive + increments version |
| `UPGRADE` (Hibernate) | `SELECT ... FOR UPDATE` | Same as PESSIMISTIC_WRITE |

---

## JTA vs JDBC Transactions

| | JDBC Transactions | JTA Transactions |
|-|------------------|-----------------|
| Scope | Single data source | Multiple data sources / XA |
| API | `java.sql.Connection` | `javax.transaction.UserTransaction` |
| Spring | `DataSourceTransactionManager` | `JtaTransactionManager` |
| Config | `hibernate.transaction.factory_class=JDBCTransactionFactory` | `hibernate.transaction.factory_class=JTATransactionFactory` |
| Use case | Single DB (most apps) | Distributed transactions |

---

## Spring + Hibernate Transactions

With Spring, use `@Transactional` — Spring manages the transaction lifecycle automatically.

```java
@Service
@Transactional
public class AccountService {

    @Autowired private SessionFactory sessionFactory;

    // Runs in a transaction; auto-commit on success, auto-rollback on RuntimeException
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        Session session = sessionFactory.getCurrentSession();

        Account from = session.get(Account.class, fromId);
        Account to   = session.get(Account.class, toId);

        from.setBalance(from.getBalance().subtract(amount));
        to.setBalance(to.getBalance().add(amount));
        // No explicit save/commit needed — dirty checking + @Transactional handles it
    }

    @Transactional(readOnly = true)           // Optimization — no flush, no dirty check
    public Account getAccount(Long id) {
        return sessionFactory.getCurrentSession().get(Account.class, id);
    }

    @Transactional(rollbackFor = Exception.class)   // Roll back on checked exceptions too
    public void riskyOperation() throws Exception { ... }
}
```

---

[← Association Mappings](./05_Association_Mappings.md) &nbsp;|&nbsp; [Next → Caching](./08_Caching.md)
