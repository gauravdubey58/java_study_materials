# 06 — Spring Data JPA

[← Back to Index](./README.md)

---

## Table of Contents
- [Repository Hierarchy](#repository-hierarchy)
- [Derived Query Methods](#derived-query-methods)
- [Custom JPQL & Native Queries](#custom-jpql--native-queries)
- [Pagination & Sorting](#pagination--sorting)
- [Projections](#projections)
- [Specifications — Dynamic Queries](#specifications--dynamic-queries)
- [The N+1 Problem & Solutions](#the-n1-problem--solutions)
- [Locking — Optimistic & Pessimistic](#locking--optimistic--pessimistic)
- [Soft Delete](#soft-delete)
- [Auditing](#auditing)
- [Flyway — Database Migrations](#flyway--database-migrations)

---

## Repository Hierarchy

```
Repository (marker)
    └── CrudRepository
    │       save(S), findById(ID), findAll(), existsById(), deleteById(), count()
    │
    └── PagingAndSortingRepository
    │       + findAll(Pageable), findAll(Sort)
    │
    └── JpaRepository
            + flush(), saveAndFlush(), deleteAllInBatch()
            + getReferenceById()
```

### When to Use Which

| Interface | Choose When |
|-----------|------------|
| `JpaRepository` | JPA + SQL databases — most common choice |
| `PagingAndSortingRepository` | Non-JPA stores that support paging |
| `CrudRepository` | Minimal interface, non-JPA, simple CRUD |

---

## Derived Query Methods

Spring Data generates SQL from the method name automatically — no `@Query` needed.

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // === Finders ===
    Optional<User> findByEmail(String email);
    List<User> findByStatus(Status status);
    List<User> findByStatusAndCity(Status status, String city);
    List<User> findByNameContainingIgnoreCase(String keyword);
    List<User> findByAgeBetween(int min, int max);
    List<User> findByAgeGreaterThan(int age);
    List<User> findByAgeLessThanEqual(int age);
    List<User> findByCreatedAtAfter(LocalDateTime date);
    List<User> findByNameIn(List<String> names);
    List<User> findByNameNotIn(Collection<String> names);
    List<User> findByAddressIsNull();
    List<User> findByAddressIsNotNull();
    List<User> findByActiveTrue();
    List<User> findByActiveFalse();

    // === Top / First ===
    User findFirstByOrderByCreatedAtDesc();         // Latest created
    List<User> findTop5ByStatusOrderByNameAsc(Status status);

    // === Count / Exists ===
    long countByStatus(Status status);
    boolean existsByEmail(String email);

    // === Delete ===
    long deleteByStatus(Status status);
    void deleteByIdAndStatus(Long id, Status status);
}
```

### Keyword Reference

| Keyword | SQL Equivalent |
|---------|---------------|
| `And` | `AND` |
| `Or` | `OR` |
| `Not` | `!=` |
| `Between` | `BETWEEN x AND y` |
| `LessThan` | `<` |
| `LessThanEqual` | `<=` |
| `GreaterThan` | `>` |
| `GreaterThanEqual` | `>=` |
| `After` | `> date` |
| `Before` | `< date` |
| `IsNull` | `IS NULL` |
| `IsNotNull` | `IS NOT NULL` |
| `Like` | `LIKE '%x%'` |
| `NotLike` | `NOT LIKE` |
| `Containing` | `LIKE '%x%'` |
| `StartingWith` | `LIKE 'x%'` |
| `EndingWith` | `LIKE '%x'` |
| `IgnoreCase` | `UPPER(x) = UPPER(y)` |
| `In` | `IN (...)` |
| `NotIn` | `NOT IN (...)` |
| `True` / `False` | `= true` / `= false` |
| `OrderBy` | `ORDER BY` |
| `Top` / `First` | `LIMIT` |

---

## Custom JPQL & Native Queries

```java
// === JPQL (entity-based, not table-based) ===
@Query("SELECT u FROM User u WHERE u.email = :email AND u.active = true")
Optional<User> findActiveByEmail(@Param("email") String email);

@Query("SELECT u FROM User u WHERE LOWER(u.name) LIKE LOWER(CONCAT('%', :keyword, '%'))")
List<User> searchByName(@Param("keyword") String keyword);

// === Positional Parameters ===
@Query("SELECT u FROM User u WHERE u.status = ?1 AND u.city = ?2")
List<User> findByStatusAndCity(Status status, String city);

// === Native SQL ===
@Query(
    value = "SELECT * FROM users WHERE created_at > :since ORDER BY created_at DESC LIMIT :limit",
    nativeQuery = true
)
List<User> findRecentUsers(@Param("since") LocalDateTime since, @Param("limit") int limit);

// === Modifying Queries (INSERT / UPDATE / DELETE) ===
@Modifying
@Transactional
@Query("UPDATE User u SET u.active = false WHERE u.lastLoginAt < :cutoff")
int deactivateInactiveUsers(@Param("cutoff") LocalDateTime cutoff);

@Modifying(clearAutomatically = true)   // Clears persistence context after — prevents stale data
@Transactional
@Query("DELETE FROM User u WHERE u.status = :status")
void deleteByStatus(@Param("status") Status status);

// === Constructor Expression (DTO projection) ===
@Query("SELECT new com.example.dto.UserSummary(u.id, u.name, u.email) FROM User u WHERE u.active = true")
List<UserSummary> findActiveSummaries();
```

---

## Pagination & Sorting

```java
// Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Page<User>  findByStatus(Status status, Pageable pageable);
    Slice<User> findByCity(String city, Pageable pageable);  // No total count
}

// Controller
@GetMapping("/users")
public ResponseEntity<Page<UserDto>> getUsers(
    @RequestParam(defaultValue = "0")    int page,
    @RequestParam(defaultValue = "20")   int size,
    @RequestParam(defaultValue = "createdAt") String sortBy,
    @RequestParam(defaultValue = "desc") String direction
) {
    Sort sort = direction.equalsIgnoreCase("asc")
        ? Sort.by(sortBy).ascending()
        : Sort.by(sortBy).descending();

    Pageable pageable = PageRequest.of(page, size, sort);
    Page<User> result = userRepository.findByStatus(Status.ACTIVE, pageable);

    return ResponseEntity.ok(result.map(userMapper::toDto));
}
```

### Page vs Slice

| Feature | `Page<T>` | `Slice<T>` |
|---------|----------|-----------|
| Total count query | ✅ Yes (extra SELECT COUNT) | ❌ No |
| Total pages/elements | ✅ Available | ❌ Not available |
| `hasNext()` | ✅ | ✅ |
| Performance | Slower (extra query) | Faster |
| Best for | Pagination with page numbers | Infinite scroll / "Load more" |

---

## Projections

Fetch only the fields you need instead of full entities.

### 1. Interface-Based (Closed) Projection

```java
// Define the interface — only these fields are selected
public interface UserSummary {
    Long getId();
    String getName();
    String getEmail();
}

// Repository
List<UserSummary> findByStatus(Status status);
```

### 2. Interface-Based (Open) Projection — SpEL

```java
public interface UserView {
    String getFirstName();
    String getLastName();

    @Value("#{target.firstName + ' ' + target.lastName}")
    String getFullName();   // Computed from entity fields
}
```

### 3. DTO (Class-Based) Projection

```java
public record UserDto(Long id, String name, String email) {}

// Via constructor expression in @Query
@Query("SELECT new com.example.dto.UserDto(u.id, u.name, u.email) FROM User u")
List<UserDto> findAllAsDto();
```

### 4. Dynamic Projections

```java
// Same method, different projections
<T> List<T> findByStatus(Status status, Class<T> type);

// Usage
List<UserSummary> summaries = repo.findByStatus(Status.ACTIVE, UserSummary.class);
List<User>        full      = repo.findByStatus(Status.ACTIVE, User.class);
```

---

## Specifications — Dynamic Queries

Use when query criteria vary at runtime (search/filter APIs).

```java
// Repository must extend JpaSpecificationExecutor
public interface UserRepository extends JpaRepository<User, Long>,
                                         JpaSpecificationExecutor<User> { }
```

```java
// Define reusable specification predicates
public class UserSpecs {

    public static Specification<User> hasEmail(String email) {
        return (root, query, cb) ->
            email == null ? null : cb.equal(root.get("email"), email);
    }

    public static Specification<User> isActive() {
        return (root, query, cb) -> cb.isTrue(root.get("active"));
    }

    public static Specification<User> ageBetween(Integer min, Integer max) {
        return (root, query, cb) -> {
            if (min == null && max == null) return null;
            if (min == null) return cb.lessThanOrEqualTo(root.get("age"), max);
            if (max == null) return cb.greaterThanOrEqualTo(root.get("age"), min);
            return cb.between(root.get("age"), min, max);
        };
    }

    public static Specification<User> cityIn(List<String> cities) {
        return (root, query, cb) ->
            cities == null || cities.isEmpty() ? null : root.get("city").in(cities);
    }
}
```

```java
// Service — compose specs dynamically
public Page<User> search(UserSearchRequest req, Pageable pageable) {
    Specification<User> spec = Specification
        .where(UserSpecs.isActive())
        .and(UserSpecs.hasEmail(req.getEmail()))
        .and(UserSpecs.ageBetween(req.getMinAge(), req.getMaxAge()))
        .and(UserSpecs.cityIn(req.getCities()));

    return userRepository.findAll(spec, pageable);
}
```

---

## The N+1 Problem & Solutions

### The Problem

```java
// Suppose you load 100 users — each has a lazy-loaded List<Order>
List<User> users = userRepository.findAll();    // 1 query: SELECT * FROM users

for (User user : users) {
    System.out.println(user.getOrders().size()); // 100 queries: SELECT * FROM orders WHERE user_id = ?
}
// Total: 1 + 100 = 101 queries  ← N+1 problem
```

### Solution 1: `JOIN FETCH` in JPQL

```java
@Query("SELECT DISTINCT u FROM User u JOIN FETCH u.orders WHERE u.active = true")
List<User> findActiveWithOrders();
```

### Solution 2: `@EntityGraph`

```java
// On repository method
@EntityGraph(attributePaths = {"orders", "orders.items", "address"})
List<User> findByActive(boolean active);

// Or define named graph on entity
@Entity
@NamedEntityGraph(
    name = "User.withOrders",
    attributeNodes = @NamedAttributeNode("orders")
)
public class User { ... }

@EntityGraph("User.withOrders")
Optional<User> findById(Long id);
```

### Solution 3: `@BatchSize`

```java
@OneToMany(mappedBy = "user")
@BatchSize(size = 20)   // Fetches 20 users' orders in a single IN clause
private List<Order> orders;
```

### Solution 4: DTO Projection

Don't fetch entities at all — fetch only what you need.

```java
@Query("SELECT new com.example.dto.UserOrderCount(u.id, u.name, SIZE(u.orders)) FROM User u GROUP BY u.id, u.name")
List<UserOrderCount> getUserOrderCounts();
```

### Comparison

| Solution | Extra Queries | Cartesian Product Risk | Ease |
|----------|--------------|----------------------|------|
| `JOIN FETCH` | ❌ None | ⚠️ Yes (with multiple collections) | Easy |
| `@EntityGraph` | ❌ None | ⚠️ Yes | Easy |
| `@BatchSize` | ⚠️ Few (batched) | ❌ No | Easy |
| DTO Projection | ❌ None | ❌ No | Medium |

---

## Locking — Optimistic & Pessimistic

### Optimistic Locking (`@Version`)

Best for **low-contention** reads with occasional updates.

```java
@Entity
public class Product {
    @Id @GeneratedValue
    private Long id;

    private String name;
    private int stock;

    @Version               // Automatically incremented on each UPDATE
    private Long version;  // If version mismatch → OptimisticLockException
}
```

```java
// Workflow: read → modify → save
// If another transaction modified the row between read and save → exception thrown
try {
    Product p = productRepository.findById(id).orElseThrow();
    p.setStock(p.getStock() - quantity);
    productRepository.save(p);      // Fails if version changed
} catch (OptimisticLockException e) {
    // Retry or inform user
}
```

### Pessimistic Locking

Best for **high-contention** concurrent writes.

```java
// Acquires a DB-level lock (SELECT ... FOR UPDATE)
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM Product p WHERE p.id = :id")
Optional<Product> findByIdWithLock(@Param("id") Long id);

// Lock modes
// PESSIMISTIC_READ  — shared lock (others can read, not write)
// PESSIMISTIC_WRITE — exclusive lock (others cannot read or write)
// PESSIMISTIC_FORCE_INCREMENT — exclusive lock + increments @Version
```

---

## Soft Delete

Delete records logically (mark as deleted) instead of physically removing them.

```java
@Entity
@SQLDelete(sql = "UPDATE users SET deleted_at = NOW() WHERE id = ?")
@Where(clause = "deleted_at IS NULL")   // Automatically filters all queries
public class User {
    @Id @GeneratedValue
    private Long id;

    private String name;

    @Column(name = "deleted_at")
    private LocalDateTime deletedAt;    // null = active, non-null = soft deleted
}
```

```java
// userRepository.delete(user) → runs UPDATE instead of DELETE
// userRepository.findAll()    → automatically filters deleted_at IS NULL
```

> ⚠️ **Note:** `@Where` is a Hibernate-specific annotation and applies to all queries including JPQL. Native queries bypass it.

---

## Auditing

Automatically populate who created/modified a record and when.

```java
// Enable auditing
@SpringBootApplication
@EnableJpaAuditing
public class MyApp { }

// Provide current user
@Component
public class SpringSecurityAuditorAware implements AuditorAware<String> {
    @Override
    public Optional<String> getCurrentAuditor() {
        return Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
            .filter(auth -> auth.isAuthenticated())
            .map(Authentication::getName);
    }
}

// Use on entity
@Entity
@EntityListeners(AuditingEntityListener.class)
public class Article {
    @CreatedDate   @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @CreatedBy    @Column(updatable = false)
    private String createdBy;

    @LastModifiedBy
    private String updatedBy;
}
```

---

## Flyway — Database Migrations

Versioned, repeatable SQL migrations tracked in a `flyway_schema_history` table.

### Setup

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
```

```properties
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration
spring.jpa.hibernate.ddl-auto=validate   # Let Flyway manage schema, not Hibernate
```

### File Naming Convention

```
src/main/resources/db/migration/
├── V1__create_users_table.sql
├── V2__add_orders_table.sql
├── V3__add_email_index.sql
├── V3.1__add_phone_column.sql
└── R__refresh_user_view.sql     (Repeatable — R__ prefix)
```

```sql
-- V1__create_users_table.sql
CREATE TABLE users (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    email      VARCHAR(100) NOT NULL UNIQUE,
    status     VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE',
    created_at TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

> ⚠️ **Never modify** an already-applied migration file. Create a new version instead.

---

[← Annotations Reference](./05_Annotations_Reference.md) &nbsp;|&nbsp; [Next → Spring Security](./07_Spring_Security.md)
