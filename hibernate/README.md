# 🗄️ Hibernate — Complete Study Notes

> **Hibernate Version:** 5.6.x / 6.x | **JPA:** 2.2 / 3.0 | **Java:** 8, 11, 17  
> **Intended For:** Developers · Interview Prep · GitHub Reference

---

## 📂 Repository Structure

| # | File | What's Inside |
|---|------|---------------|
| 01 | [Introduction & Core Concepts](./01_Introduction_Core_Concepts.md) | What is Hibernate, architecture, all key terms & definitions |
| 02 | [XML-Based Mapping](./02_XML_Based_Mapping.md) | `hibernate.cfg.xml`, `hbm.xml` mappings, all element types |
| 03 | [Annotation-Based Mapping](./03_Annotation_Based_Mapping.md) | JPA + Hibernate annotations, entity mapping, relationships |
| 04 | [Inheritance Mapping](./04_Inheritance_Mapping.md) | Single Table, Joined, Table-Per-Class strategies with tradeoffs |
| 05 | [Association Mappings](./05_Association_Mappings.md) | One-to-One, One-to-Many, Many-to-One, Many-to-Many |
| 06 | [Session Management](./06_Session_Management.md) | Session, SessionFactory, lifecycle, states, best practices |
| 07 | [Transaction Management](./07_Transaction_Management.md) | Transactions, propagation, isolation, rollback, JTA vs JDBC |
| 08 | [Caching](./08_Caching.md) | L1 cache, L2 cache (EhCache/Redis), Query Cache, strategies |
| 09 | [Loading Strategies](./09_Loading_Strategies.md) | Lazy vs Eager, fetch joins, batch fetching, N+1 problem |
| 10 | [100 Interview Questions](./10_Interview_Questions.md) | Q1–50 Moderate · Q51–100 Tough |

---

## ⚡ Quick-Start Cheat Sheet

```java
// Standard Hibernate bootstrap (without Spring)
SessionFactory sessionFactory = new Configuration()
    .configure("hibernate.cfg.xml")   // or programmatically
    .addAnnotatedClass(User.class)
    .buildSessionFactory();

// Typical CRUD
try (Session session = sessionFactory.openSession()) {
    Transaction tx = session.beginTransaction();
    session.save(new User("Alice", "alice@example.com"));
    tx.commit();
}

// With Spring Boot — just inject the repository
@Autowired
private UserRepository userRepository;
```

---

## 🏷️ Key Version Notes

| Feature | Hibernate 5.x | Hibernate 6.x |
|---------|--------------|--------------|
| JPA spec | JPA 2.2 | JPA 3.0 (Jakarta) |
| Package | `javax.persistence` | `jakarta.persistence` |
| Query language | HQL / JPQL | HQL (improved) / JPQL |
| UUID generation | Manual | `@GeneratedValue(UUID)` built-in |
| `@Any` mapping | ✅ | ✅ (improved) |
| Criteria API | ✅ | ✅ (type-safe improvements) |

---

## 🗺️ Hibernate Architecture Overview

```
Your Application Code
        │
        ▼
┌───────────────────┐
│  SessionFactory   │  ← One per application (thread-safe, heavyweight)
│   (immutable)     │
└────────┬──────────┘
         │ creates
         ▼
┌───────────────────┐
│     Session       │  ← One per unit of work (not thread-safe, lightweight)
│  (1st level cache)│
└────────┬──────────┘
         │
   ┌─────┴──────┐
   │            │
   ▼            ▼
Transaction   Query
   │            │
   └─────┬──────┘
         │
         ▼
┌───────────────────┐
│  JDBC Connection  │
│  (Connection Pool)│
└────────┬──────────┘
         │
         ▼
   Database (MySQL/PostgreSQL/Oracle/H2)
```

---

> 💡 **Start here:** [Introduction & Core Concepts](./01_Introduction_Core_Concepts.md)  
> 🎯 **Interview prep:** Go straight to [100 Interview Questions](./10_Interview_Questions.md)  
> 🐛 Found an error? PRs are welcome!
