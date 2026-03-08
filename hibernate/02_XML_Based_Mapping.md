# 02 — XML-Based Mapping

[← Back to Index](./README.md)

---

## Table of Contents
- [What is XML-Based Mapping?](#what-is-xml-based-mapping)
- [Basic hbm.xml Structure](#basic-hbmxml-structure)
- [All hbm.xml Elements & Attributes](#all-hbmxml-elements--attributes)
- [Primary Key Generators](#primary-key-generators)
- [Property Types & Column Mapping](#property-types--column-mapping)
- [Component Mapping](#component-mapping)
- [Collection Mappings](#collection-mappings)
- [Relationship Mappings](#relationship-mappings)
- [Inheritance Mapping in XML](#inheritance-mapping-in-xml)
- [Named Queries in XML](#named-queries-in-xml)
- [Complete Working Example](#complete-working-example)

---

## What is XML-Based Mapping?

XML-based mapping uses `.hbm.xml` files to describe how Java classes map to database tables — without touching the Java source code. This was the original Hibernate approach before annotations were introduced.

**When to use XML mapping today:**
- Legacy applications built before JPA annotations existed
- When you cannot modify the source class (third-party library)
- When you want mapping configuration fully separated from source code
- When different deployments need different mappings for the same class

---

## Basic hbm.xml Structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE hibernate-mapping PUBLIC
    "-//Hibernate/Hibernate Mapping DTD 3.0//EN"
    "http://www.hibernate.org/dtd/hibernate-mapping-3.0.dtd">

<hibernate-mapping
    package="com.example.entity"
    schema="public"
    catalog="mydb"
    default-lazy="true"
    default-cascade="none"
    default-access="field">

    <class name="User" table="users" dynamic-update="true" dynamic-insert="false">

        <!-- Primary Key -->
        <id name="id" column="id" type="long">
            <generator class="identity"/>
        </id>

        <!-- Version for optimistic locking -->
        <version name="version" column="version" type="integer"/>

        <!-- Simple properties -->
        <property name="name"      column="name"       type="string"    not-null="true" length="100"/>
        <property name="email"     column="email"       type="string"    not-null="true" unique="true"/>
        <property name="age"       column="age"         type="integer"/>
        <property name="active"    column="active"      type="boolean"   not-null="true"/>
        <property name="createdAt" column="created_at"  type="timestamp" update="false"/>

    </class>

</hibernate-mapping>
```

---

## All hbm.xml Elements & Attributes

### `<hibernate-mapping>` — Root Element

| Attribute | Description | Default |
|-----------|-------------|---------|
| `package` | Default package for class names | — |
| `schema` | Default DB schema | — |
| `catalog` | Default DB catalog | — |
| `default-lazy` | Lazy loading default for all collections | `true` |
| `default-cascade` | Default cascade for all associations | `none` |
| `default-access` | Property access: `field` or `property` | `property` |
| `auto-import` | Auto-import class names for HQL | `true` |

---

### `<class>` — Maps a Java Class to a Table

| Attribute | Description |
|-----------|-------------|
| `name` | Fully qualified class name |
| `table` | Database table name |
| `schema` | Override schema for this class |
| `lazy` | Enable proxy-based lazy loading |
| `dynamic-update` | Generate UPDATE only for changed columns |
| `dynamic-insert` | Generate INSERT only for non-null columns |
| `select-before-update` | SELECT before UPDATE to detect actual changes |
| `polymorphism` | `implicit` (default) or `explicit` |
| `where` | Append SQL WHERE to all queries (like soft delete filter) |
| `mutable` | If `false`, object is read-only (no UPDATE ever) |
| `batch-size` | Batch size for lazy loading this entity |
| `optimistic-lock` | `version` (default) \| `dirty` \| `all` \| `none` |
| `entity-name` | Logical name if multiple mappings for same class |

---

### `<id>` — Primary Key

```xml
<id name="id" column="id" type="long" access="field">
    <column name="id" not-null="true"/>
    <generator class="identity"/>
</id>
```

| Attribute | Description |
|-----------|-------------|
| `name` | Java field name |
| `column` | DB column name |
| `type` | Hibernate type |
| `access` | `field` or `property` |
| `unsaved-value` | Value that means "not yet saved" (default: `null` for objects, `0` for primitives) |

---

### `<composite-id>` — Composite Primary Key

```xml
<composite-id name="id" class="OrderItemId">
    <key-property name="orderId"   column="order_id"   type="long"/>
    <key-property name="productId" column="product_id" type="long"/>
</composite-id>
```

---

### `<version>` — Optimistic Locking

```xml
<version name="version" column="version" type="integer"
         access="field" unsaved-value="null"/>
```

---

### `<timestamp>` — Timestamp-Based Versioning

```xml
<timestamp name="lastUpdated" column="last_updated"
           source="db"/>   <!-- 'db' uses DB time, 'vm' uses JVM time -->
```

---

### `<property>` — Regular Column Mapping

```xml
<property name="email"
          column="email_address"
          type="string"
          not-null="true"
          unique="true"
          length="255"
          update="true"
          insert="true"
          formula="upper(email)"
          access="field"
          lazy="false"
          generated="never"/>   <!-- never | insert | always -->
```

| Attribute | Description |
|-----------|-------------|
| `name` | Java field name |
| `column` | DB column (defaults to field name) |
| `type` | Hibernate type (see below) |
| `not-null` | Add NOT NULL constraint |
| `unique` | Add UNIQUE constraint |
| `length` | Column length for VARCHAR |
| `precision` / `scale` | For numeric types |
| `update` | Include in UPDATEs |
| `insert` | Include in INSERTs |
| `formula` | Derive value from SQL expression (read-only) |
| `lazy` | Lazy-load this column (requires bytecode enhancement) |
| `generated` | `never`, `insert`, `always` — DB-generated values |

---

### `<many-to-one>` — Many-to-One / Foreign Key

```xml
<many-to-one name="department"
             class="Department"
             column="dept_id"
             not-null="true"
             fetch="join"
             cascade="none"
             lazy="proxy"
             foreign-key="fk_emp_dept"/>
```

---

### `<one-to-one>` — One-to-One Relationship

```xml
<!-- Primary key shared (same PK) -->
<one-to-one name="profile"
            class="UserProfile"
            cascade="all"
            constrained="true"/>

<!-- Foreign key in this table -->
<many-to-one name="profile"
             class="UserProfile"
             column="profile_id"
             unique="true"
             cascade="all"/>
```

---

### `<set>`, `<list>`, `<bag>`, `<map>` — Collections

```xml
<!-- Set (no duplicates, no order) -->
<set name="orders"
     table="orders"
     cascade="all-delete-orphan"
     lazy="true"
     inverse="true"
     batch-size="10"
     fetch="select">
    <key column="user_id"/>
    <one-to-many class="Order"/>
</set>

<!-- List (ordered, allows duplicates) -->
<list name="tags" table="product_tags" lazy="true">
    <key column="product_id"/>
    <list-index column="position"/>
    <element type="string" column="tag"/>
</list>

<!-- Bag (unordered, allows duplicates) -->
<bag name="comments" table="comments" cascade="all">
    <key column="post_id"/>
    <one-to-many class="Comment"/>
</bag>

<!-- Map -->
<map name="attributes" table="product_attributes">
    <key column="product_id"/>
    <map-key type="string" column="attr_key"/>
    <element type="string" column="attr_value"/>
</map>
```

---

### `<many-to-many>` Mapping

```xml
<set name="roles" table="user_roles" lazy="true">
    <key column="user_id"/>
    <many-to-many class="Role" column="role_id"/>
</set>
```

---

### `<component>` — Embeddable / Value Object

```xml
<component name="address" class="Address" access="field">
    <property name="street"  column="street"   type="string"/>
    <property name="city"    column="city"      type="string"/>
    <property name="country" column="country"   type="string"/>
    <property name="zipCode" column="zip_code"  type="string"/>
</component>
```

---

### `<join>` — Map One Class Across Multiple Tables

```xml
<class name="Employee" table="employees">
    <id name="id"><generator class="identity"/></id>
    <property name="name" column="name"/>

    <!-- Additional columns from a second table -->
    <join table="employee_details" optional="true" fetch="join">
        <key column="employee_id"/>
        <property name="bio"    column="bio"    type="text"/>
        <property name="photo"  column="photo"  type="binary"/>
    </join>
</class>
```

---

### `<subclass>`, `<joined-subclass>`, `<union-subclass>` — Inheritance

```xml
<!-- Single Table Inheritance -->
<class name="Animal" table="animals" discriminator-value="ANIMAL">
    <id name="id"><generator class="identity"/></id>
    <discriminator column="animal_type" type="string"/>
    <property name="name"/>
    <subclass name="Dog" discriminator-value="DOG">
        <property name="breed" column="breed"/>
    </subclass>
    <subclass name="Cat" discriminator-value="CAT">
        <property name="indoor" column="is_indoor"/>
    </subclass>
</class>

<!-- Joined (Table-Per-Subclass) -->
<class name="Vehicle" table="vehicles">
    <id name="id"><generator class="identity"/></id>
    <property name="make"/>
    <joined-subclass name="Car" table="cars">
        <key column="vehicle_id"/>
        <property name="numDoors" column="num_doors"/>
    </joined-subclass>
    <joined-subclass name="Truck" table="trucks">
        <key column="vehicle_id"/>
        <property name="payload" column="payload_tons"/>
    </joined-subclass>
</class>

<!-- Union / Table-Per-Class -->
<class name="Shape" abstract="true">
    <id name="id"><generator class="hilo"/></id>
    <union-subclass name="Circle" table="circles">
        <property name="radius"/>
    </union-subclass>
    <union-subclass name="Rectangle" table="rectangles">
        <property name="width"/>
        <property name="height"/>
    </union-subclass>
</class>
```

---

## Primary Key Generators

| Generator | Description | Use Case |
|-----------|-------------|---------|
| `identity` | DB auto-increment (`AUTO_INCREMENT`) | MySQL, PostgreSQL SERIAL |
| `sequence` | DB sequence | PostgreSQL, Oracle |
| `hilo` | High-Low algorithm (Hibernate internal) | Legacy; avoid in new code |
| `seqhilo` | Sequence + high-low | Legacy |
| `uuid` | 128-bit UUID as string | Distributed systems |
| `uuid2` | RFC 4122 compliant UUID | Preferred UUID option |
| `guid` | Database `GUID()` / `newid()` | MSSQL, MySQL |
| `native` | Uses `identity`, `sequence`, or `hilo` based on DB | Portable |
| `assigned` | Application sets the ID manually | Natural/composite keys |
| `select` | Uses a DB trigger to generate key | Legacy trigger-based schemas |
| `increment` | Hibernate increments in-memory | Single-instance only; not safe for clusters |
| `foreign` | Uses the PK of an associated entity | `@OneToOne` shared PK |

```xml
<!-- Sequence with custom parameters -->
<id name="id" type="long">
    <generator class="sequence">
        <param name="sequence_name">user_seq</param>
        <param name="initial_value">1</param>
        <param name="increment_size">50</param>
    </generator>
</id>
```

---

## Property Types & Column Mapping

### Hibernate Basic Types

| Hibernate Type | Java Type | SQL Type |
|---------------|-----------|---------|
| `string` | `String` | `VARCHAR` |
| `integer` / `int` | `Integer` / `int` | `INTEGER` |
| `long` | `Long` / `long` | `BIGINT` |
| `short` | `Short` / `short` | `SMALLINT` |
| `double` | `Double` / `double` | `DOUBLE` |
| `float` | `Float` / `float` | `FLOAT` |
| `boolean` | `Boolean` / `boolean` | `BIT` / `BOOLEAN` |
| `byte` | `Byte` / `byte` | `TINYINT` |
| `character` | `Character` / `char` | `CHAR(1)` |
| `date` | `java.util.Date` | `DATE` |
| `time` | `java.util.Date` | `TIME` |
| `timestamp` | `java.util.Date` | `TIMESTAMP` |
| `calendar` | `java.util.Calendar` | `TIMESTAMP` |
| `big_decimal` | `java.math.BigDecimal` | `NUMERIC` |
| `big_integer` | `java.math.BigInteger` | `NUMERIC` |
| `locale` | `java.util.Locale` | `VARCHAR` |
| `text` | `String` | `CLOB` / `TEXT` |
| `blob` | `java.sql.Blob` | `BLOB` |
| `clob` | `java.sql.Clob` | `CLOB` |
| `binary` | `byte[]` | `VARBINARY` |
| `uuid-char` | `java.util.UUID` | `CHAR(36)` |
| `uuid-binary` | `java.util.UUID` | `BINARY(16)` |

---

## Component Mapping

A component (embeddable) is a value object whose fields are stored in the same table as the owning entity.

```xml
<class name="com.example.Person" table="persons">
    <id name="id"><generator class="identity"/></id>
    <property name="firstName" column="first_name"/>
    <property name="lastName"  column="last_name"/>

    <!-- Embedded address -->
    <component name="homeAddress" class="com.example.Address">
        <property name="street"  column="home_street"/>
        <property name="city"    column="home_city"/>
        <property name="zipCode" column="home_zip"/>
    </component>

    <!-- Another embedded address -->
    <component name="workAddress" class="com.example.Address">
        <property name="street"  column="work_street"/>
        <property name="city"    column="work_city"/>
        <property name="zipCode" column="work_zip"/>
    </component>
</class>
```

---

## Named Queries in XML

```xml
<!-- HQL Named Query -->
<query name="User.findByEmail">
    <![CDATA[
        SELECT u FROM User u WHERE u.email = :email AND u.active = true
    ]]>
</query>

<!-- Native SQL Named Query -->
<sql-query name="User.findActiveRaw">
    <return alias="u" class="User"/>
    <![CDATA[
        SELECT {u.*} FROM users u WHERE u.active = 1 ORDER BY u.name
    ]]>
</sql-query>
```

Usage:
```java
List<User> users = session.getNamedQuery("User.findByEmail")
    .setParameter("email", "alice@example.com")
    .list();
```

---

## Complete Working Example

### Java Classes

```java
public class Department {
    private Long id;
    private String name;
    private List<Employee> employees;
    // getters + setters
}

public class Employee {
    private Long id;
    private String name;
    private String email;
    private Address address;   // Component
    private Department department;
    // getters + setters
}

public class Address {
    private String street;
    private String city;
    private String zipCode;
    // getters + setters
}
```

### Department.hbm.xml

```xml
<?xml version="1.0"?>
<!DOCTYPE hibernate-mapping PUBLIC
    "-//Hibernate/Hibernate Mapping DTD 3.0//EN"
    "http://www.hibernate.org/dtd/hibernate-mapping-3.0.dtd">

<hibernate-mapping package="com.example.entity">
    <class name="Department" table="departments">

        <id name="id" column="id" type="long">
            <generator class="identity"/>
        </id>

        <property name="name" column="name" type="string" not-null="true" length="100"/>

        <set name="employees"
             table="employees"
             cascade="all-delete-orphan"
             lazy="true"
             inverse="true"
             batch-size="20">
            <key column="dept_id"/>
            <one-to-many class="Employee"/>
        </set>

    </class>
</hibernate-mapping>
```

### Employee.hbm.xml

```xml
<?xml version="1.0"?>
<!DOCTYPE hibernate-mapping PUBLIC
    "-//Hibernate/Hibernate Mapping DTD 3.0//EN"
    "http://www.hibernate.org/dtd/hibernate-mapping-3.0.dtd">

<hibernate-mapping package="com.example.entity">
    <class name="Employee" table="employees" dynamic-update="true">

        <id name="id" column="id" type="long">
            <generator class="identity"/>
        </id>

        <version name="version" column="version" type="integer"/>

        <property name="name"  column="name"  type="string" not-null="true"/>
        <property name="email" column="email" type="string" not-null="true" unique="true"/>

        <!-- Embedded address component -->
        <component name="address" class="Address">
            <property name="street"  column="street"   type="string"/>
            <property name="city"    column="city"      type="string"/>
            <property name="zipCode" column="zip_code"  type="string"/>
        </component>

        <!-- Many-to-one (FK in employees table) -->
        <many-to-one name="department"
                     class="Department"
                     column="dept_id"
                     not-null="true"
                     fetch="join"
                     lazy="proxy"/>

    </class>
</hibernate-mapping>
```

---

[← Introduction](./01_Introduction_Core_Concepts.md) &nbsp;|&nbsp; [Next → Annotation-Based Mapping](./03_Annotation_Based_Mapping.md)
