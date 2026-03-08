# 01 — Introduction & Core Concepts

[← Back to Index](./README.md)

---

## Table of Contents
- [What is Hibernate?](#what-is-hibernate)
- [Why Hibernate? (ORM Problem)](#why-hibernate-orm-problem)
- [Hibernate vs JPA](#hibernate-vs-jpa)
- [Hibernate Architecture](#hibernate-architecture)
- [Complete Glossary of Terms](#complete-glossary-of-terms)
- [Hibernate Configuration](#hibernate-configuration)
- [Dialect Reference](#dialect-reference)

---

## What is Hibernate?

Hibernate is an **Object-Relational Mapping (ORM)** framework for Java. It maps Java classes (objects) to database tables, and Java data types to SQL data types, automating the data persistence layer so developers work with objects rather than raw SQL.

```
Java Object  ←─── Hibernate ───→  Database Table
  User.java                         users table
  Order.java                        orders table
```

**Key capabilities:**
- Automatic SQL generation (SELECT, INSERT, UPDATE, DELETE)
- Object-to-table mapping via annotations or XML
- HQL (Hibernate Query Language) — object-oriented queries
- First and second-level caching
- Lazy and eager loading of associations
- Transaction management
- Schema generation and validation

---

## Why Hibernate? (ORM Problem)

Without an ORM, you write repetitive boilerplate:

```java
// Without Hibernate — 20+ lines just to save one object
Connection conn = DriverManager.getConnection(url, user, pass);
PreparedStatement ps = conn.prepareStatement(
    "INSERT INTO users (name, email, age) VALUES (?, ?, ?)");
ps.setString(1, user.getName());
ps.setString(2, user.getEmail());
ps.setInt(3, user.getAge());
ps.executeUpdate();
ps.close();
conn.close();

// With Hibernate — 1 line
session.save(user);
```

The **Object-Relational Impedance Mismatch** is the fundamental problem ORM solves:

| Object World | Relational World |
|-------------|-----------------|
| Objects / Classes | Rows / Tables |
| Inheritance | No natural equivalent |
| Association (references) | Foreign keys |
| Identity (`==` / `equals`) | Primary key |
| Graph navigation | Joins |
| Encapsulation | Column visibility |

---

## Hibernate vs JPA

| Aspect | JPA | Hibernate |
|--------|-----|-----------|
| Type | Specification (API) | Implementation |
| Package (Java EE) | `javax.persistence` | `org.hibernate` |
| Package (Jakarta EE) | `jakarta.persistence` | `org.hibernate` |
| Portability | High (swap implementations) | Vendor-specific features |
| Features | Standard set | Superset of JPA + extras |
| Who defines it | Oracle / Jakarta EE | Red Hat |

> **Rule of thumb:** Use JPA annotations (`@Entity`, `@Id`, etc.) for portability. Use Hibernate-specific annotations (`@DynamicUpdate`, `@BatchSize`, etc.) only when you need the extra capability.

---

## Hibernate Architecture

### Layered Architecture

```
┌─────────────────────────────────────────────────────┐
│                 Application Layer                    │
│         (Your POJOs, Service classes)               │
└─────────────────────────┬───────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────┐
│               Hibernate Layer                        │
│  ┌──────────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ SessionFactory│  │ Session  │  │ Transaction  │  │
│  └──────────────┘  └──────────┘  └──────────────┘  │
│  ┌──────────────┐  ┌──────────┐  ┌──────────────┐  │
│  │  Query/      │  │  Cache   │  │   Criteria   │  │
│  │  HQL         │  │  L1 / L2 │  │   API        │  │
│  └──────────────┘  └──────────┘  └──────────────┘  │
└─────────────────────────┬───────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────┐
│                   JDBC Layer                         │
│         Connection Pool (HikariCP / C3P0)           │
└─────────────────────────┬───────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────┐
│                  Database Layer                      │
│         MySQL / PostgreSQL / Oracle / H2            │
└─────────────────────────────────────────────────────┘
```

### Core Components

| Component | Description |
|-----------|-------------|
| `Configuration` | Reads `hibernate.cfg.xml` or programmatic config; creates `SessionFactory` |
| `SessionFactory` | Thread-safe factory for `Session` objects; built once per application |
| `Session` | Single-threaded unit of work; manages entities and DB operations |
| `Transaction` | Represents a DB transaction; obtained from `Session` |
| `Query` / `HQL` | Object-oriented query language; translated to SQL by Hibernate |
| `Criteria API` | Programmatic, type-safe query building |
| `First-Level Cache` | Per-session entity cache; always enabled |
| `Second-Level Cache` | Per-`SessionFactory` shared cache; optional (EhCache, Redis, etc.) |
| `ConnectionProvider` | Abstracts JDBC connection pooling |
| `Dialect` | Translates HQL/SQL to database-specific SQL syntax |

---

## Complete Glossary of Terms

### A

**Association**  
A relationship between two entity classes. Types: `@OneToOne`, `@OneToMany`, `@ManyToOne`, `@ManyToMany`.

**AuditingEntityListener**  
A JPA lifecycle listener that auto-populates audit fields (`createdAt`, `updatedAt`, etc.).

---

### B

**Batch Fetching**  
An optimization where Hibernate loads collections or proxies in batches instead of one-by-one. Configured with `@BatchSize(size = N)`. Reduces N+1 queries.

**Bidirectional Association**  
An association navigable from both sides. One side is the "owner" (has the foreign key); the other uses `mappedBy`.

```java
// Owner side
@ManyToOne
@JoinColumn(name = "dept_id")
private Department department;

// Inverse side (mappedBy = field name on owner side)
@OneToMany(mappedBy = "department")
private List<Employee> employees;
```

---

### C

**Cascade**  
Propagation of persistence operations from parent to child entities. Types: `PERSIST`, `MERGE`, `REMOVE`, `REFRESH`, `DETACH`, `ALL`.

```java
@OneToMany(cascade = CascadeType.ALL)  // Operations on parent propagate to children
private List<Order> orders;
```

**Criteria API**  
A programmatic, type-safe way to build queries without writing HQL/SQL strings. Uses `CriteriaBuilder`, `CriteriaQuery`, and `Root`.

```java
CriteriaBuilder cb = session.getCriteriaBuilder();
CriteriaQuery<User> cq = cb.createQuery(User.class);
Root<User> root = cq.from(User.class);
cq.where(cb.equal(root.get("status"), "ACTIVE"));
List<User> users = session.createQuery(cq).getResultList();
```

---

### D

**Detached State**  
An entity that was previously associated with a Session, but the Session has been closed. The entity holds an ID but Hibernate no longer tracks its changes. Use `session.merge()` to reattach.

**Dialect**  
A Hibernate class that generates database-specific SQL. E.g., `MySQL8Dialect` generates MySQL-compatible SQL.

**Dirty Checking**  
Hibernate's automatic detection of changes to managed (persistent) entities. At flush time, Hibernate compares the current state with the snapshot taken at load time and generates UPDATE SQL for changed fields.

**`@DynamicInsert`**  
Hibernate annotation that generates INSERT SQL with only non-null columns instead of all columns.

**`@DynamicUpdate`**  
Hibernate annotation that generates UPDATE SQL with only the changed columns instead of all columns.

---

### E

**Eager Loading**  
Loading associated entities/collections immediately when the owning entity is loaded. Uses JOIN queries or secondary SELECTs. Configured with `fetch = FetchType.EAGER`.

**Embeddable**  
A class whose fields are stored in the same table as the owning entity. Marked with `@Embeddable` and used with `@Embedded`. Has no own identity/primary key.

**Entity**  
A Java class mapped to a database table. Instances represent rows. Must have a no-arg constructor and an `@Id` field.

**Entity State**  
An entity's lifecycle state relative to a Hibernate Session: **Transient**, **Persistent**, **Detached**, or **Removed**.

---

### F

**Fetch Strategy**  
How Hibernate loads an association: `SELECT` (separate query), `JOIN` (SQL JOIN), `SUBSELECT`, or `BATCH`.

**First-Level Cache (L1)**  
Session-scoped cache. Every object loaded within a Session is stored here. Subsequent `get()` calls for the same ID return the cached object without a DB hit. Always enabled; cannot be disabled.

**Flush**  
Synchronizing the in-memory state of the Session with the database (executing pending SQL). Triggered automatically before queries, at transaction commit, or manually with `session.flush()`.

**FlushMode**  
Controls when Hibernate flushes: `AUTO` (default — before queries and commit), `COMMIT` (only at commit), `MANUAL` (only when explicitly called), `ALWAYS` (before every query).

---

### G

**Generator Strategy**  
How primary key values are generated. Types: `AUTO`, `IDENTITY`, `SEQUENCE`, `TABLE`, `UUID`, `assigned` (manual).

---

### H

**HQL (Hibernate Query Language)**  
Hibernate's object-oriented query language. Similar to SQL but operates on entity names and fields (not table/column names). Fully portable across databases.

```sql
-- HQL
SELECT u FROM User u WHERE u.email = :email AND u.active = true

-- Translated SQL (MySQL)
SELECT u.id, u.name, u.email, u.active FROM users u WHERE u.email = ? AND u.active = 1
```

**`hibernate.cfg.xml`**  
The main Hibernate configuration file. Contains DB connection details, dialect, and mapping references.

**`hbm.xml` (Hibernate Mapping File)**  
An XML file defining how a Java class maps to a database table. Alternative to annotations.

---

### I

**Impedance Mismatch**  
The conceptual difference between the object model (Java) and the relational model (SQL) that ORM bridges.

**Interceptor**  
A Hibernate extension point for intercepting entity lifecycle events (load, save, update, delete). Implement `EmptyInterceptor` and attach to `SessionFactory` or `Session`.

---

### J

**`@JoinColumn`**  
Specifies the foreign key column for a relationship.

**`@JoinTable`**  
Specifies a join table for `@ManyToMany` or `@OneToMany` relationships.

**JPQL (Java Persistence Query Language)**  
The standard query language defined by the JPA specification. Hibernate's HQL is a superset of JPQL.

---

### L

**Lazy Loading**  
Loading associated data only when it is first accessed. Default for `@OneToMany` and `@ManyToMany`. Hibernate returns a proxy object instead of the real collection, which is populated on first access.

**`LazyInitializationException`**  
Thrown when a lazily loaded association is accessed after the Session is closed.

---

### M

**Managed State (Persistent State)**  
An entity associated with an open Session. Hibernate tracks all changes and syncs them at flush time.

**`mappedBy`**  
Used on the inverse side of a bidirectional relationship to point to the field on the owning side. Tells Hibernate which side owns the foreign key.

**Merge**  
`session.merge(entity)` — copies the state of a detached entity into a new managed instance. Returns the managed copy.

---

### N

**Named Query**  
A pre-defined, reusable HQL/JPQL query declared on an entity with `@NamedQuery`. Parsed and validated at startup.

```java
@NamedQuery(name = "User.findByEmail",
            query = "SELECT u FROM User u WHERE u.email = :email")
@Entity
public class User { }
```

**Native Query**  
A raw SQL query executed via Hibernate. Used when HQL cannot express the needed query.

**N+1 Query Problem**  
Loading N parent entities and then issuing N additional queries to load each parent's association. Fixed with JOIN FETCH, `@EntityGraph`, or `@BatchSize`.

---

### O

**Optimistic Locking**  
A concurrency strategy using a `@Version` field. Fails with `OptimisticLockException` if another transaction modified the same row. No DB-level locks held.

**Orphan Removal**  
Automatically deletes child entities when they are removed from a parent's collection. Enabled with `orphanRemoval = true`.

---

### P

**Persistence Context**  
The set of entity instances that a `Session` (or `EntityManager`) is currently managing. Acts as the first-level cache.

**Persistent State**  
See Managed State.

**Proxy**  
A Hibernate-generated subclass or interface implementation that wraps an entity. Used for lazy loading — the real data is fetched only when a method is called.

---

### Q

**Query Cache**  
Caches the result sets of frequently run queries (not the entities themselves). Must be explicitly enabled per query. Requires second-level cache to be active.

---

### R

**Removed State**  
An entity scheduled for deletion. `session.delete(entity)` marks it removed; the DELETE SQL runs at flush.

---

### S

**Second-Level Cache (L2)**  
A `SessionFactory`-scoped, shared cache for entities, collections, and query results. Optional; requires a provider like EhCache, Infinispan, or Redis. Shared across sessions.

**Session**  
Hibernate's primary interface for DB operations. Represents a unit of work and the first-level cache boundary. Not thread-safe.

**SessionFactory**  
Thread-safe factory that creates `Session` objects. Heavyweight — built once. Holds the second-level cache.

**Snapshot**  
A copy of an entity's state taken when it is first loaded into the session. Used by dirty checking to detect what changed.

---

### T

**Transient State**  
A newly created Java object with no association to any Session and no record in the database.

**Transaction**  
A unit of work where all operations succeed together or all fail (ACID). In Hibernate: `session.beginTransaction()` → do work → `tx.commit()`.

---

### U

**Unidirectional Association**  
An association navigable from only one side. Simpler but may require extra queries in some scenarios.

**`update()` vs `merge()`**  

| `update()` | `merge()` |
|------------|-----------|
| Reattaches a detached entity to the current session | Copies state to a new/existing managed instance |
| Entity must not already be in the session | Safe even if another instance exists in session |
| Returns void | Returns the managed copy |
| Hibernate-specific | JPA standard |

---

### V

**`@Version`**  
Marks a field for optimistic locking. Hibernate automatically increments it on each UPDATE. If the version in the DB doesn't match the in-memory version, `OptimisticLockException` is thrown.

---

### W

**Write-Behind**  
Hibernate buffers SQL statements and sends them to the DB at flush time, not immediately when you call `save()`/`update()`. This allows batching and reordering.

---

## Hibernate Configuration

### `hibernate.cfg.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE hibernate-configuration PUBLIC
    "-//Hibernate/Hibernate Configuration DTD 3.0//EN"
    "http://www.hibernate.org/dtd/hibernate-configuration-3.0.dtd">

<hibernate-configuration>
    <session-factory>

        <!-- ── Database Connection ── -->
        <property name="hibernate.connection.driver_class">com.mysql.cj.jdbc.Driver</property>
        <property name="hibernate.connection.url">jdbc:mysql://localhost:3306/mydb</property>
        <property name="hibernate.connection.username">root</property>
        <property name="hibernate.connection.password">secret</property>

        <!-- ── SQL Dialect ── -->
        <property name="hibernate.dialect">org.hibernate.dialect.MySQL8Dialect</property>

        <!-- ── Connection Pool (built-in — use HikariCP in production) ── -->
        <property name="hibernate.connection.pool_size">10</property>

        <!-- ── Schema Management ── -->
        <!-- none | validate | update | create | create-drop -->
        <property name="hibernate.hbm2ddl.auto">update</property>

        <!-- ── Logging ── -->
        <property name="hibernate.show_sql">true</property>
        <property name="hibernate.format_sql">true</property>
        <property name="hibernate.use_sql_comments">true</property>

        <!-- ── Second-Level Cache ── -->
        <property name="hibernate.cache.use_second_level_cache">true</property>
        <property name="hibernate.cache.region.factory_class">
            org.hibernate.cache.ehcache.EhCacheRegionFactory
        </property>
        <property name="hibernate.cache.use_query_cache">true</property>

        <!-- ── JDBC Batch ── -->
        <property name="hibernate.jdbc.batch_size">30</property>
        <property name="hibernate.order_inserts">true</property>
        <property name="hibernate.order_updates">true</property>

        <!-- ── Mapping files ── -->
        <mapping resource="com/example/User.hbm.xml"/>
        <mapping resource="com/example/Order.hbm.xml"/>

        <!-- Or annotated classes -->
        <mapping class="com.example.entity.Product"/>
        <mapping class="com.example.entity.Category"/>

    </session-factory>
</hibernate-configuration>
```

### Programmatic Configuration

```java
SessionFactory sessionFactory = new Configuration()
    .setProperty("hibernate.connection.url", "jdbc:mysql://localhost:3306/mydb")
    .setProperty("hibernate.connection.username", "root")
    .setProperty("hibernate.connection.password", "secret")
    .setProperty("hibernate.dialect", "org.hibernate.dialect.MySQL8Dialect")
    .setProperty("hibernate.hbm2ddl.auto", "update")
    .setProperty("hibernate.show_sql", "true")
    .addAnnotatedClass(User.class)
    .addAnnotatedClass(Order.class)
    .buildSessionFactory();
```

---

## Dialect Reference

| Database | Dialect Class |
|----------|--------------|
| MySQL 5.x | `org.hibernate.dialect.MySQL5Dialect` |
| MySQL 8.x | `org.hibernate.dialect.MySQL8Dialect` |
| PostgreSQL | `org.hibernate.dialect.PostgreSQLDialect` |
| Oracle 12c | `org.hibernate.dialect.Oracle12cDialect` |
| Microsoft SQL Server | `org.hibernate.dialect.SQLServerDialect` |
| H2 (in-memory) | `org.hibernate.dialect.H2Dialect` |
| HSQLDB | `org.hibernate.dialect.HSQLDialect` |
| SQLite | `org.hibernate.dialect.SQLiteDialect` |
| MariaDB | `org.hibernate.dialect.MariaDB103Dialect` |
| DB2 | `org.hibernate.dialect.DB2Dialect` |

---

[Next → XML-Based Mapping](./02_XML_Based_Mapping.md)
