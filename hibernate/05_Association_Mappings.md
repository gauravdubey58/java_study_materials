# 05 — Association Mappings

[← Back to Index](./README.md)

---

## Table of Contents
- [Key Concepts](#key-concepts)
- [One-to-One](#one-to-one)
- [One-to-Many / Many-to-One](#one-to-many--many-to-one)
- [Many-to-Many](#many-to-many)
- [Cascade Types](#cascade-types)
- [Fetch Types](#fetch-types)
- [Bidirectional Sync Helper Methods](#bidirectional-sync-helper-methods)
- [Common Mistakes & Best Practices](#common-mistakes--best-practices)

---

## Key Concepts

| Term | Definition |
|------|-----------|
| **Owner side** | The side that holds the foreign key column in the DB. Always `@ManyToOne` or the side without `mappedBy`. |
| **Inverse side** | The side with `mappedBy = "fieldName"`. Does NOT have a FK column. Hibernate ignores changes made only on this side. |
| **Unidirectional** | Association navigable from one side only |
| **Bidirectional** | Association navigable from both sides. Requires `mappedBy` on inverse side. |
| **Cascade** | Propagates operations (save, delete, etc.) from parent to child |
| **orphanRemoval** | Deletes children that are removed from the parent's collection |

---

## One-to-One

One record in Table A relates to exactly one record in Table B.

### Strategy 1: Shared Primary Key

Both entities share the same primary key value. The child's PK is also an FK to the parent.

```java
@Entity
@Table(name = "users")
public class User {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY,
              optional = false)
    private UserProfile profile;
}

@Entity
@Table(name = "user_profiles")
public class UserProfile {

    @Id
    private Long id;     // Same value as User.id

    @OneToOne(fetch = FetchType.LAZY)
    @MapsId              // Maps PK to the User's PK
    @JoinColumn(name = "id")
    private User user;

    private String bio;
    private String avatarUrl;
}
```

**Schema:**
```sql
users         (id PK)
user_profiles (id PK FK → users.id)
```

---

### Strategy 2: Foreign Key in Child Table

More common. The child table has a unique FK column pointing to the parent.

```java
@Entity
@Table(name = "employees")
public class Employee {

    @Id @GeneratedValue private Long id;
    private String name;

    @OneToOne(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @JoinColumn(name = "parking_spot_id", unique = true)
    private ParkingSpot parkingSpot;
}

@Entity
@Table(name = "parking_spots")
public class ParkingSpot {

    @Id @GeneratedValue private Long id;
    private String spotNumber;

    @OneToOne(mappedBy = "parkingSpot")  // Inverse side
    private Employee employee;
}
```

**Schema:**
```sql
employees     (id PK, parking_spot_id UNIQUE FK → parking_spots.id)
parking_spots (id PK)
```

---

### XML Mapping

```xml
<!-- Employee.hbm.xml -->
<many-to-one name="parkingSpot" class="ParkingSpot"
             column="parking_spot_id" unique="true"
             cascade="all" fetch="join" lazy="proxy"/>

<!-- ParkingSpot.hbm.xml (inverse side) -->
<one-to-one name="employee" class="Employee"
            property-ref="parkingSpot"
            cascade="none"/>
```

---

## One-to-Many / Many-to-One

One parent has many children. The FK lives in the child's table.

### Bidirectional (Recommended)

```java
@Entity
@Table(name = "departments")
public class Department {

    @Id @GeneratedValue private Long id;
    private String name;

    // Inverse side — owns no FK column
    @OneToMany(
        mappedBy     = "department",
        cascade      = CascadeType.ALL,
        orphanRemoval = true,
        fetch        = FetchType.LAZY
    )
    @OrderBy("name ASC")
    private List<Employee> employees = new ArrayList<>();

    // Helper methods to keep both sides in sync
    public void addEmployee(Employee e) {
        employees.add(e);
        e.setDepartment(this);
    }

    public void removeEmployee(Employee e) {
        employees.remove(e);
        e.setDepartment(null);
    }
}

@Entity
@Table(name = "employees")
public class Employee {

    @Id @GeneratedValue private Long id;
    private String name;

    // Owner side — holds the FK column
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "dept_id", nullable = false)
    private Department department;
}
```

**Schema:**
```sql
departments (id PK, name)
employees   (id PK, name, dept_id FK → departments.id)
```

---

### Unidirectional One-to-Many (Avoid)

Navigable only from parent. Hibernate creates a **join table** by default — inefficient.

```java
@Entity
public class Post {
    @OneToMany(cascade = CascadeType.ALL)
    // ← No mappedBy → creates join table: post_comments (post_id, comments_id)
    private List<Comment> comments;
}
```

To avoid the join table, specify `@JoinColumn`:

```java
@OneToMany(cascade = CascadeType.ALL)
@JoinColumn(name = "post_id")  // FK in comments table — no join table
private List<Comment> comments;
```

---

### XML Mapping

```xml
<!-- Department.hbm.xml -->
<set name="employees" table="employees"
     cascade="all-delete-orphan" lazy="true" inverse="true" batch-size="20">
    <key column="dept_id" not-null="true"/>
    <one-to-many class="Employee"/>
</set>

<!-- Employee.hbm.xml -->
<many-to-one name="department" class="Department"
             column="dept_id" not-null="true"
             fetch="join" lazy="proxy"/>
```

---

## Many-to-Many

Both sides can have multiple records of the other. Requires a **join table**.

### Annotation-Based

```java
@Entity
@Table(name = "students")
public class Student {

    @Id @GeneratedValue private Long id;
    private String name;

    // Owner side — defines the join table
    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE},
                fetch = FetchType.LAZY)
    @JoinTable(
        name               = "student_courses",
        joinColumns        = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();

    public void enroll(Course c)  { courses.add(c); c.getStudents().add(this); }
    public void withdraw(Course c){ courses.remove(c); c.getStudents().remove(this); }
}

@Entity
@Table(name = "courses")
public class Course {

    @Id @GeneratedValue private Long id;
    private String title;

    @ManyToMany(mappedBy = "courses")   // Inverse side
    private Set<Student> students = new HashSet<>();
}
```

**Schema:**
```sql
students        (id PK, name)
courses         (id PK, title)
student_courses (student_id FK, course_id FK, PK(student_id, course_id))
```

---

### Many-to-Many with Extra Columns (Join Entity)

When the join table has extra columns (e.g., enrollment date, grade), replace `@ManyToMany` with a join entity:

```java
@Entity
@Table(name = "enrollments")
public class Enrollment {

    @EmbeddedId
    private EnrollmentId id;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("studentId")
    private Student student;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("courseId")
    private Course course;

    private LocalDate enrolledAt;

    @Enumerated(EnumType.STRING)
    private Grade grade;
}

@Embeddable
public class EnrollmentId implements Serializable {
    private Long studentId;
    private Long courseId;
    // equals + hashCode
}

// In Student:
@OneToMany(mappedBy = "student", cascade = CascadeType.ALL)
private List<Enrollment> enrollments = new ArrayList<>();
```

---

### XML Mapping

```xml
<!-- Student.hbm.xml -->
<set name="courses" table="student_courses" lazy="true" cascade="save-update,merge">
    <key column="student_id"/>
    <many-to-many class="Course" column="course_id"/>
</set>

<!-- Course.hbm.xml (inverse) -->
<set name="students" table="student_courses" lazy="true" inverse="true">
    <key column="course_id"/>
    <many-to-many class="Student" column="student_id"/>
</set>
```

---

## Cascade Types

Cascade propagates operations from parent to children.

| Type | Propagates | JPA | Hibernate |
|------|-----------|-----|-----------|
| `PERSIST` | `session.persist()` / `save()` | ✅ | ✅ |
| `MERGE` | `session.merge()` | ✅ | ✅ |
| `REMOVE` | `session.remove()` / `delete()` | ✅ | ✅ |
| `REFRESH` | `session.refresh()` | ✅ | ✅ |
| `DETACH` | `session.detach()` / `evict()` | ✅ | ✅ |
| `ALL` | All above | ✅ | ✅ |
| `REPLICATE` | `session.replicate()` | ❌ | ✅ Hibernate-only |
| `SAVE_UPDATE` | `session.saveOrUpdate()` | ❌ | ✅ Hibernate-only |
| `LOCK` | `session.lock()` | ❌ | ✅ Hibernate-only |
| `DELETE` | Alias for REMOVE | ❌ | ✅ Hibernate-only |

### `orphanRemoval` vs `CascadeType.REMOVE`

```java
// CascadeType.REMOVE: delete children when parent is deleted
@OneToMany(cascade = CascadeType.REMOVE)
private List<Comment> comments;

// orphanRemoval: ALSO delete children when removed from collection
@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
private List<Comment> comments;

// With orphanRemoval:
post.getComments().remove(comment);  // ← This DELETES the comment from DB
// Without orphanRemoval: just removes the FK reference
```

---

## Fetch Types

| Type | Behavior | Default For |
|------|---------|-------------|
| `EAGER` | Load immediately with parent | `@ManyToOne`, `@OneToOne` |
| `LAZY` | Load on first access (proxy) | `@OneToMany`, `@ManyToMany` |

```java
@ManyToOne(fetch = FetchType.LAZY)     // Override EAGER default — always recommended
private Department department;

@OneToMany(fetch = FetchType.LAZY)     // Default — fine
private List<Order> orders;

@OneToMany(fetch = FetchType.EAGER)    // Override LAZY — use carefully
private List<Permission> permissions;
```

> ⚠️ **Best practice:** Always use `LAZY` on all associations. Fetch eagerly only when needed via `JOIN FETCH` in queries.

---

## Bidirectional Sync Helper Methods

Both sides of a bidirectional association must be kept in sync manually in Java memory. Helper methods enforce this:

```java
@Entity
public class Post {

    @OneToMany(mappedBy = "post", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Comment> comments = new ArrayList<>();

    // Always add via this method — keeps both sides in sync
    public void addComment(Comment comment) {
        comments.add(comment);
        comment.setPost(this);
    }

    public void removeComment(Comment comment) {
        comments.remove(comment);
        comment.setPost(null);
    }
}

@Entity
public class Comment {

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "post_id")
    private Post post;

    // equals() and hashCode() based on natural key, NOT id
    // Using id breaks logic before the entity is persisted (id = null)
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Comment)) return false;
        Comment c = (Comment) o;
        return Objects.equals(content, c.content);  // Use business key
    }

    @Override
    public int hashCode() {
        return Objects.hashCode(content);
    }
}
```

---

## Common Mistakes & Best Practices

| Mistake | Fix |
|---------|-----|
| Using `@ManyToMany` with extra attributes | Model as a separate entity (`@OneToMany` + `@ManyToOne`) |
| Not setting `mappedBy` on inverse side | Results in two FKs or a join table when only one is needed |
| Using `FetchType.EAGER` everywhere | Causes unnecessary JOINs on every query; use `LAZY` + JOIN FETCH when needed |
| Not providing helper methods for bidirectional | One side gets out of sync in memory |
| Using `List` with `@ManyToMany` | Causes `HibernateException` (bag semantics with join table); use `Set` |
| `orphanRemoval = true` without `CascadeType.ALL` | Removing a child from the collection won't delete it (needs PERSIST + MERGE at minimum) |
| `equals()`/`hashCode()` based on `id` in `@ManyToMany` Set | Before saving, id is null — two unsaved entities are "equal"; use business key |
| Cascade `REMOVE` on `@ManyToMany` | Deletes the target entity entirely, not just the join row |

---

[← Inheritance Mapping](./04_Inheritance_Mapping.md) &nbsp;|&nbsp; [Next → Session Management](./06_Session_Management.md)
