# 03 — Annotation-Based Mapping

[← Back to Index](./README.md)

---

## Table of Contents
- [JPA Annotations](#jpa-annotations)
- [Hibernate-Specific Annotations](#hibernate-specific-annotations)
- [Column & Table Mapping](#column--table-mapping)
- [Primary Key & Generation](#primary-key--generation)
- [Embeddable Types](#embeddable-types)
- [Enum Mapping](#enum-mapping)
- [Date & Time Mapping](#date--time-mapping)
- [LOB Mapping](#lob-mapping)
- [Formula (Derived Properties)](#formula-derived-properties)
- [Complete Entity Example](#complete-entity-example)

---

## JPA Annotations

### `@Entity`
Marks a class as a JPA entity mapped to a database table.

```java
@Entity
@Table(name = "users",
    schema = "public",
    indexes = {
        @Index(name = "idx_email",      columnList = "email"),
        @Index(name = "idx_created_at", columnList = "created_at DESC")
    },
    uniqueConstraints = {
        @UniqueConstraint(name = "uq_email", columnNames = {"email"})
    }
)
public class User {
    // must have @Id field and no-arg constructor
}
```

---

### `@Id`
Marks the primary key field.

```java
@Id
private Long id;
```

---

### `@GeneratedValue`
Specifies the primary key generation strategy.

| Strategy | SQL Behavior | Best For |
|----------|-------------|---------|
| `AUTO` | Provider chooses | Prototyping |
| `IDENTITY` | DB auto-increment | MySQL, PostgreSQL SERIAL |
| `SEQUENCE` | DB sequence | PostgreSQL, Oracle |
| `TABLE` | Simulated sequence via table | Portability (avoid) |

```java
// IDENTITY
@Id @GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

// SEQUENCE with custom sequence
@Id @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "user_seq")
@SequenceGenerator(name = "user_seq", sequenceName = "user_id_seq",
                   allocationSize = 50, initialValue = 1)
private Long id;

// UUID (Hibernate 6)
@Id @GeneratedValue(strategy = GenerationType.UUID)
private UUID id;
```

---

### `@Column`
Maps a field to a specific column with constraints.

```java
@Column(
    name            = "email_address",
    nullable        = false,
    unique          = true,
    length          = 255,
    precision       = 10,    // for BigDecimal
    scale           = 2,     // for BigDecimal
    insertable      = true,
    updatable       = false, // Immutable after insert (e.g., createdAt)
    columnDefinition = "VARCHAR(255) DEFAULT 'active'"
)
private String email;
```

---

### `@Transient`
Excludes a field from all DB operations.

```java
@Transient
private String fullName;  // computed from firstName + lastName; not persisted
```

---

### `@Version`
Enables optimistic locking via automatic version increment.

```java
@Version
private Long version;  // auto-incremented on every UPDATE
```

---

### `@Lob`
Maps a field to a large object column (CLOB or BLOB).

```java
@Lob
@Column(name = "content")
private String content;  // → CLOB (TEXT)

@Lob
@Column(name = "thumbnail")
private byte[] thumbnail;  // → BLOB
```

---

### `@Enumerated`
Controls how enum values are stored.

```java
@Enumerated(EnumType.STRING)   // ✅ Stores "ACTIVE", "INACTIVE" — readable, safe
private Status status;

@Enumerated(EnumType.ORDINAL)  // ❌ Stores 0, 1 — fragile if enum order changes
private Priority priority;
```

---

### `@Temporal`
Specifies how `java.util.Date` / `Calendar` fields are mapped (not needed for `java.time` types).

```java
@Temporal(TemporalType.DATE)       // → DATE
private Date birthDate;

@Temporal(TemporalType.TIME)       // → TIME
private Date meetingTime;

@Temporal(TemporalType.TIMESTAMP)  // → TIMESTAMP
private Date createdAt;

// Modern (no @Temporal needed):
private LocalDate    birthDate;    // → DATE
private LocalTime    meetingTime;  // → TIME
private LocalDateTime createdAt;  // → TIMESTAMP
```

---

### `@Basic`
Controls basic type loading behavior.

```java
@Basic(fetch = FetchType.LAZY, optional = false)
@Lob
private byte[] largeFile;  // Loaded lazily — requires bytecode enhancement
```

---

### `@Access`
Specifies field vs property access at the class or field level.

```java
@Entity
@Access(AccessType.FIELD)  // Hibernate accesses fields directly (not getters)
public class User {
    @Id private Long id;
    private String name;
}
```

---

### `@ElementCollection`
Maps a collection of basic types or embeddables to a separate table.

```java
@ElementCollection
@CollectionTable(name = "user_phones",
    joinColumns = @JoinColumn(name = "user_id"))
@Column(name = "phone_number")
private List<String> phoneNumbers;

// Ordered
@ElementCollection
@OrderColumn(name = "list_order")
private List<String> tags;

// Map
@ElementCollection
@MapKeyColumn(name = "label")
@Column(name = "url")
private Map<String, String> socialLinks;
```

---

### `@Embedded` & `@Embeddable`

```java
@Embeddable
public class Address {
    private String street;
    private String city;
    @Column(name = "zip_code")
    private String zipCode;
}

@Entity
public class Employee {
    @Embedded
    private Address homeAddress;

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "street",  column = @Column(name = "work_street")),
        @AttributeOverride(name = "city",    column = @Column(name = "work_city")),
        @AttributeOverride(name = "zipCode", column = @Column(name = "work_zip"))
    })
    private Address workAddress;   // Reuse embeddable with different column names
}
```

---

### `@IdClass` — Composite Primary Key (Option 1)

```java
// The ID class (must implement Serializable)
public class OrderItemId implements Serializable {
    private Long orderId;
    private Long productId;
    // equals() + hashCode() + no-arg constructor required
}

@Entity
@IdClass(OrderItemId.class)
public class OrderItem {
    @Id private Long orderId;
    @Id private Long productId;
    private int quantity;
}
```

---

### `@EmbeddedId` — Composite Primary Key (Option 2)

```java
@Embeddable
public class OrderItemId implements Serializable {
    private Long orderId;
    private Long productId;
    // equals() + hashCode() required
}

@Entity
public class OrderItem {
    @EmbeddedId
    private OrderItemId id;
    private int quantity;
}
```

---

### `@NamedQuery` & `@NamedNativeQuery`

```java
@NamedQueries({
    @NamedQuery(
        name  = "User.findByEmail",
        query = "SELECT u FROM User u WHERE u.email = :email AND u.active = true"
    ),
    @NamedQuery(
        name  = "User.findAllActive",
        query = "SELECT u FROM User u WHERE u.active = true ORDER BY u.name"
    )
})
@NamedNativeQuery(
    name            = "User.countByCity",
    query           = "SELECT city, COUNT(*) FROM users GROUP BY city",
    resultSetMapping = "CityCountMapping"
)
@Entity
public class User { }
```

---

### `@EntityListeners` — Lifecycle Callbacks

```java
@Entity
@EntityListeners(AuditingEntityListener.class)
public class Article {

    @CreatedDate  @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @CreatedBy   @Column(updatable = false)
    private String createdBy;

    @LastModifiedBy
    private String updatedBy;

    // JPA Lifecycle callbacks on the entity itself
    @PrePersist
    void prePersist() { this.createdAt = LocalDateTime.now(); }

    @PreUpdate
    void preUpdate()  { this.updatedAt = LocalDateTime.now(); }

    @PostLoad
    void postLoad()   { /* called after entity loaded from DB */ }

    @PostPersist  void postPersist()  { /* after INSERT */ }
    @PostUpdate   void postUpdate()   { /* after UPDATE */ }
    @PreRemove    void preRemove()    { /* before DELETE */ }
    @PostRemove   void postRemove()   { /* after DELETE */ }
}
```

---

## Hibernate-Specific Annotations

### `@DynamicUpdate` / `@DynamicInsert`

```java
@Entity
@DynamicUpdate   // UPDATE sets only changed columns
@DynamicInsert   // INSERT sets only non-null columns
public class Product {
    @Id @GeneratedValue private Long id;
    private String name;
    private BigDecimal price;
    private String description;  // often unchanged — won't appear in most UPDATEs
}
```

---

### `@SelectBeforeUpdate`

```java
@Entity
@SelectBeforeUpdate   // SELECT entity before UPDATE to detect actual changes
public class Config { }
```

---

### `@BatchSize`

```java
@Entity
@BatchSize(size = 20)  // Load up to 20 entities in one IN clause
public class Product { }

// Also on collections
@OneToMany
@BatchSize(size = 10)  // Load up to 10 collections at once
private List<Review> reviews;
```

---

### `@Fetch`

```java
@OneToMany
@Fetch(FetchMode.SUBSELECT)   // Load all collections in one subselect
private List<Order> orders;

@ManyToOne
@Fetch(FetchMode.JOIN)        // Always JOIN fetch
private Category category;
```

---

### `@Where`

```java
@Entity
@Where(clause = "deleted_at IS NULL")  // Applied to all queries on this entity
public class User { }

// On collection
@OneToMany
@Where(clause = "status = 'ACTIVE'")
private List<Order> activeOrders;
```

---

### `@Filter` / `@FilterDef`

More flexible than `@Where` — can be toggled on/off per session.

```java
@FilterDef(name = "activeFilter",
           parameters = @ParamDef(name = "active", type = "boolean"))
@Filter(name = "activeFilter", condition = "active = :active")
@Entity
public class User { }

// Enable per session
session.enableFilter("activeFilter").setParameter("active", true);
```

---

### `@Immutable`

```java
@Entity
@Immutable   // Never generates UPDATE SQL — entity is read-only
public class AuditLog { }
```

---

### `@NaturalId`

```java
@Entity
public class User {
    @Id @GeneratedValue private Long id;

    @NaturalId   // Business key — Hibernate caches and queries by this
    @Column(unique = true)
    private String email;
}

// Load by natural ID (cached, no full table scan)
User user = session.byNaturalId(User.class)
    .using("email", "alice@example.com")
    .load();
```

---

### `@SQLDelete` / `@SQLInsert` / `@SQLUpdate`

Override the default generated SQL:

```java
@Entity
@SQLDelete(sql = "UPDATE users SET deleted_at = NOW() WHERE id = ?")
@Where(clause = "deleted_at IS NULL")
public class User {
    private LocalDateTime deletedAt;
}
```

---

### `@OrderBy` / `@SortNatural` / `@SortComparator`

```java
@OneToMany
@OrderBy("createdAt DESC, price ASC")   // SQL ORDER BY
private List<Review> reviews;

@ManyToMany
@SortNatural   // Java-side sorting (Comparable)
private SortedSet<Tag> tags;
```

---

### `@Cache` (Second-Level Cache)

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE, region = "users")
public class User { }

@OneToMany
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)
private List<Permission> permissions;
```

---

## Complete Entity Example

```java
@Entity
@Table(name = "products",
    indexes = @Index(name = "idx_sku", columnList = "sku"),
    uniqueConstraints = @UniqueConstraint(columnNames = "sku")
)
@DynamicUpdate
@BatchSize(size = 25)
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
@SQLDelete(sql = "UPDATE products SET deleted_at = NOW() WHERE id = ? AND version = ?")
@Where(clause = "deleted_at IS NULL")
@NamedQuery(name = "Product.findBySku",
            query = "SELECT p FROM Product p WHERE p.sku = :sku")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "product_seq")
    @SequenceGenerator(name = "product_seq", sequenceName = "product_id_seq", allocationSize = 10)
    private Long id;

    @Version
    private Long version;

    @Column(name = "sku", nullable = false, length = 50)
    @NaturalId
    private String sku;

    @Column(nullable = false, length = 200)
    private String name;

    @Column(precision = 10, scale = 2)
    private BigDecimal price;

    @Lob
    @Column(name = "description")
    private String description;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private ProductStatus status;

    @Embedded
    private Dimensions dimensions;

    @ElementCollection
    @CollectionTable(name = "product_images",
                     joinColumns = @JoinColumn(name = "product_id"))
    @Column(name = "image_url")
    @OrderColumn(name = "display_order")
    private List<String> imageUrls = new ArrayList<>();

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private Category category;

    @OneToMany(mappedBy = "product", cascade = CascadeType.ALL, orphanRemoval = true)
    @BatchSize(size = 20)
    private List<Review> reviews = new ArrayList<>();

    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @Column(name = "deleted_at")
    private LocalDateTime deletedAt;

    @PrePersist
    void onCreate() { createdAt = updatedAt = LocalDateTime.now(); }

    @PreUpdate
    void onUpdate() { updatedAt = LocalDateTime.now(); }
}
```

---

[← XML-Based Mapping](./02_XML_Based_Mapping.md) &nbsp;|&nbsp; [Next → Inheritance Mapping](./04_Inheritance_Mapping.md)
