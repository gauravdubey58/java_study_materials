# 04 — Inheritance Mapping

[← Back to Index](./README.md)

---

## Table of Contents
- [Overview of Strategies](#overview-of-strategies)
- [1. Single Table Inheritance (STI)](#1-single-table-inheritance-sti)
- [2. Joined Table Inheritance (Table-Per-Subclass)](#2-joined-table-inheritance-table-per-subclass)
- [3. Table Per Concrete Class (Union)](#3-table-per-concrete-class-union)
- [4. MappedSuperclass](#4-mappedsuperclass)
- [Strategy Comparison](#strategy-comparison)
- [Polymorphic Queries](#polymorphic-queries)
- [XML-Based Inheritance Mapping](#xml-based-inheritance-mapping)

---

## Overview of Strategies

JPA/Hibernate supports four approaches to mapping an inheritance hierarchy to relational tables:

```
        Animal (abstract/concrete base)
       /       \
    Dog        Cat
   /
 Poodle
```

| Strategy | Tables | JPA Annotation |
|----------|--------|---------------|
| Single Table | 1 table for entire hierarchy | `@Inheritance(strategy = InheritanceType.SINGLE_TABLE)` |
| Joined | 1 table per class (JOINed) | `@Inheritance(strategy = InheritanceType.JOINED)` |
| Table Per Concrete Class | 1 table per concrete class | `@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)` |
| MappedSuperclass | Not a JPA entity — shared fields only | `@MappedSuperclass` |

---

## 1. Single Table Inheritance (STI)

**All classes in the hierarchy share one table.** A discriminator column identifies the type of each row.

### Schema

```sql
CREATE TABLE vehicles (
    id             BIGINT PRIMARY KEY AUTO_INCREMENT,
    vehicle_type   VARCHAR(31) NOT NULL,   -- discriminator
    make           VARCHAR(100),
    model          VARCHAR(100),
    num_doors      INT,          -- Car only (NULL for trucks)
    payload_tons   DOUBLE,       -- Truck only (NULL for cars)
    engine_cc      INT           -- Car only
);
```

### Annotation-Based

```java
@Entity
@Table(name = "vehicles")
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(
    name            = "vehicle_type",
    discriminatorType = DiscriminatorType.STRING,
    length          = 31
)
@DiscriminatorValue("VEHICLE")
public abstract class Vehicle {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String make;

    @Column(nullable = false)
    private String model;

    // getters + setters
}

@Entity
@DiscriminatorValue("CAR")
public class Car extends Vehicle {

    @Column(name = "num_doors")
    private int numberOfDoors;

    @Column(name = "engine_cc")
    private int engineCc;
    // getters + setters
}

@Entity
@DiscriminatorValue("TRUCK")
public class Truck extends Vehicle {

    @Column(name = "payload_tons")
    private double payloadTons;
    // getters + setters
}

@Entity
@DiscriminatorValue("ELECTRIC_CAR")
public class ElectricCar extends Car {

    @Column(name = "battery_kwh")
    private double batteryKwh;
}
```

### XML-Based

```xml
<class name="Vehicle" table="vehicles" discriminator-value="VEHICLE">
    <id name="id"><generator class="identity"/></id>
    <discriminator column="vehicle_type" type="string"/>
    <property name="make"  column="make"/>
    <property name="model" column="model"/>

    <subclass name="Car" discriminator-value="CAR">
        <property name="numberOfDoors" column="num_doors"/>
        <property name="engineCc"      column="engine_cc"/>

        <subclass name="ElectricCar" discriminator-value="ELECTRIC_CAR">
            <property name="batteryKwh" column="battery_kwh"/>
        </subclass>
    </subclass>

    <subclass name="Truck" discriminator-value="TRUCK">
        <property name="payloadTons" column="payload_tons"/>
    </subclass>
</class>
```

### Characteristics

| Aspect | Detail |
|--------|--------|
| Tables created | 1 |
| Query performance | ✅ Best — no JOINs |
| Storage | ❌ Many nullable columns |
| Normalization | ❌ Poor |
| Polymorphic query | ✅ Simple — single table scan |
| NOT NULL constraints | ❌ Cannot enforce on subclass columns |
| Best for | Shallow hierarchies with few subclass-specific columns |

---

## 2. Joined Table Inheritance (Table-Per-Subclass)

**Each class has its own table.** Subclass tables share the same primary key as the base table. Queries use JOINs.

### Schema

```sql
CREATE TABLE payments (
    id           BIGINT PRIMARY KEY AUTO_INCREMENT,
    amount       DECIMAL(10,2) NOT NULL,
    currency     VARCHAR(3),
    created_at   TIMESTAMP
);

CREATE TABLE credit_card_payments (
    id           BIGINT PRIMARY KEY,     -- FK to payments.id
    card_number  VARCHAR(16),
    card_type    VARCHAR(20),
    FOREIGN KEY (id) REFERENCES payments(id)
);

CREATE TABLE bank_transfer_payments (
    id           BIGINT PRIMARY KEY,
    account_no   VARCHAR(20),
    bank_code    VARCHAR(10),
    FOREIGN KEY (id) REFERENCES payments(id)
);
```

### Annotation-Based

```java
@Entity
@Table(name = "payments")
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class Payment {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal amount;

    @Column(length = 3)
    private String currency;

    private LocalDateTime createdAt;

    @PrePersist void onCreate() { createdAt = LocalDateTime.now(); }
}

@Entity
@Table(name = "credit_card_payments")
@PrimaryKeyJoinColumn(name = "id")   // Optional — default is same as parent PK name
public class CreditCardPayment extends Payment {

    @Column(name = "card_number", length = 16)
    private String cardNumber;

    @Column(name = "card_type", length = 20)
    private String cardType;
}

@Entity
@Table(name = "bank_transfer_payments")
public class BankTransferPayment extends Payment {

    @Column(name = "account_no", length = 20)
    private String accountNo;

    @Column(name = "bank_code", length = 10)
    private String bankCode;
}
```

### XML-Based

```xml
<class name="Payment" table="payments">
    <id name="id"><generator class="identity"/></id>
    <property name="amount"    column="amount"    type="big_decimal"/>
    <property name="currency"  column="currency"  type="string"/>
    <property name="createdAt" column="created_at" type="timestamp"/>

    <joined-subclass name="CreditCardPayment" table="credit_card_payments">
        <key column="id"/>
        <property name="cardNumber" column="card_number"/>
        <property name="cardType"   column="card_type"/>
    </joined-subclass>

    <joined-subclass name="BankTransferPayment" table="bank_transfer_payments">
        <key column="id"/>
        <property name="accountNo" column="account_no"/>
        <property name="bankCode"  column="bank_code"/>
    </joined-subclass>
</class>
```

### Characteristics

| Aspect | Detail |
|--------|--------|
| Tables created | 1 per class in hierarchy |
| Query performance | ⚠️ JOIN per subclass level |
| Storage | ✅ Normalized, no nulls |
| Normalization | ✅ Best |
| Polymorphic query | ⚠️ Requires LEFT OUTER JOINs |
| NOT NULL constraints | ✅ Can enforce on subclass columns |
| Best for | Deep hierarchies; data integrity important; complex subclass schemas |

---

## 3. Table Per Concrete Class (Union)

**Each concrete (non-abstract) class has its own complete table.** The base class fields are duplicated in each subclass table.

### Schema

```sql
CREATE TABLE dogs (
    id       BIGINT PRIMARY KEY,
    name     VARCHAR(100),
    breed    VARCHAR(50)
);

CREATE TABLE cats (
    id       BIGINT PRIMARY KEY,
    name     VARCHAR(100),
    indoor   BOOLEAN
);
-- Note: 'name' column is repeated in both tables
```

### Annotation-Based

```java
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Animal {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)   // TABLE or SEQUENCE — not IDENTITY
    private Long id;

    private String name;
}

@Entity
@Table(name = "dogs")
public class Dog extends Animal {
    private String breed;
}

@Entity
@Table(name = "cats")
public class Cat extends Animal {
    private boolean indoor;
}
```

> ⚠️ **`IDENTITY` generation strategy does NOT work** with `TABLE_PER_CLASS` because identity generation is per-table. Use `AUTO`, `SEQUENCE`, or `TABLE`.

### XML-Based

```xml
<class name="Animal" abstract="true">
    <id name="id">
        <generator class="hilo"/>   <!-- or sequence — NOT identity -->
    </id>
    <property name="name" column="name"/>

    <union-subclass name="Dog" table="dogs">
        <property name="breed" column="breed"/>
    </union-subclass>

    <union-subclass name="Cat" table="cats">
        <property name="indoor" column="indoor"/>
    </union-subclass>
</class>
```

### Characteristics

| Aspect | Detail |
|--------|--------|
| Tables created | 1 per concrete class |
| Query performance | ⚠️ UNION ALL for polymorphic queries |
| Storage | ❌ Column duplication across tables |
| Normalization | ❌ Poor — base fields duplicated |
| Polymorphic query | ❌ Slow — UNION ALL across tables |
| Foreign keys | ❌ Cannot reference abstract base |
| Best for | Rarely — when no polymorphic queries are needed |

---

## 4. MappedSuperclass

**Not a JPA entity.** No table created for the base class. Subclasses get all fields. Cannot be queried as a type. Used purely to share common fields/mappings.

### Annotation-Based

```java
@MappedSuperclass
public abstract class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @Version
    private Long version;

    @PrePersist
    void prePersist() { createdAt = updatedAt = LocalDateTime.now(); }

    @PreUpdate
    void preUpdate()  { updatedAt = LocalDateTime.now(); }

    // getters + setters
}

@Entity
@Table(name = "users")
public class User extends BaseEntity {
    private String name;
    private String email;
}

@Entity
@Table(name = "products")
public class Product extends BaseEntity {
    private String sku;
    private BigDecimal price;
}
```

The `users` and `products` tables both get `id`, `created_at`, `updated_at`, `version` columns automatically.

### Characteristics

| Aspect | Detail |
|--------|--------|
| Creates its own table | ❌ No |
| Queryable as base type | ❌ No |
| Field inheritance | ✅ All fields propagate |
| Polymorphism | ❌ Not supported |
| Best for | Sharing audit fields, version, or ID strategy across unrelated entities |

---

## Strategy Comparison

| Feature | Single Table | Joined | Table Per Class | MappedSuperclass |
|---------|:----------:|:------:|:---------------:|:----------------:|
| Tables | 1 | N (per class) | N (per concrete) | 0 (no own table) |
| Nullable columns | ❌ Many | ✅ None | ❌ Duplicated | N/A |
| Polymorphic query | ✅ Fast | ⚠️ JOINs | ❌ UNION | ❌ Not supported |
| NOT NULL on subclass | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes |
| FK to base type | ✅ Yes | ✅ Yes | ❌ No | ❌ No |
| Normalization | ❌ Poor | ✅ Best | ❌ Poor | N/A |
| IDENTITY strategy | ✅ Yes | ✅ Yes | ❌ No | ✅ Yes |
| Deep hierarchy cost | ✅ Same | ❌ More JOINs | ❌ More UNIONs | ✅ Same |
| **Recommended** | Shallow/few subclass columns | Data integrity + deep | Avoid | Shared audit fields |

---

## Polymorphic Queries

All strategies support polymorphic HQL queries (except `MappedSuperclass`):

```java
// Retrieves ALL vehicles regardless of subtype (Car, Truck, ElectricCar)
List<Vehicle> all = session.createQuery("FROM Vehicle", Vehicle.class).list();

// Filter to a specific subtype
List<Car> cars = session.createQuery("FROM Car", Car.class).list();

// Polymorphic parameter
Query<Vehicle> q = session.createQuery(
    "FROM Vehicle v WHERE TYPE(v) IN (Car, ElectricCar)", Vehicle.class);
```

**Generated SQL per strategy:**

```sql
-- Single Table: one SELECT with discriminator filter
SELECT * FROM vehicles WHERE vehicle_type IN ('CAR', 'ELECTRIC_CAR');

-- Joined: LEFT OUTER JOINs
SELECT v.*, c.*, e.* FROM vehicles v
LEFT OUTER JOIN cars c ON v.id = c.id
LEFT OUTER JOIN electric_cars e ON c.id = e.id;

-- Table Per Class: UNION ALL
SELECT id, name, num_doors AS numDoors, NULL AS batteryKwh, 'CAR' AS clazz FROM cars
UNION ALL
SELECT id, name, num_doors, battery_kwh, 'ELECTRIC_CAR' FROM electric_cars;
```

---

[← Annotation-Based Mapping](./03_Annotation_Based_Mapping.md) &nbsp;|&nbsp; [Next → Association Mappings](./05_Association_Mappings.md)
