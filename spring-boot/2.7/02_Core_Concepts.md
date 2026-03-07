# 02 — Core Concepts & Terminology

[← Back to Index](./README.md)

---

## Table of Contents
- [IoC — Inversion of Control](#1-ioc--inversion-of-control)
- [Dependency Injection (DI)](#2-dependency-injection-di)
- [Spring Bean & Scopes](#3-spring-bean--scopes)
- [ApplicationContext vs BeanFactory](#4-applicationcontext-vs-beanfactory)
- [AOP — Aspect-Oriented Programming](#5-aop--aspect-oriented-programming)
- [Transaction Management](#6-transaction-management)
- [Spring MVC & DispatcherServlet](#7-spring-mvc--dispatcherservlet)
- [Spring Data Concepts](#8-spring-data-concepts)
- [Externalized Configuration](#9-externalized-configuration)
- [Other Important Terms](#10-other-important-terms)

---

## 1. IoC — Inversion of Control

**Definition:** A design principle where the control of object creation and lifecycle is transferred from the application code to a **container** (Spring IoC Container).

**Use Case:** Instead of manually instantiating objects with `new`, Spring manages their lifecycle, making code loosely coupled and easier to test.

```java
// ❌ Without IoC — tight coupling
UserService userService = new UserService(new UserRepository(new DataSource()));

// ✅ With IoC — Spring manages it
@Autowired
private UserService userService;
```

---

## 2. Dependency Injection (DI)

**Definition:** A specific implementation of IoC where dependencies are "injected" into a class rather than the class creating them.

### Three Types of DI

| Type | How | Recommended? | Why |
|------|-----|-------------|-----|
| **Constructor Injection** | Via constructor | ✅ Yes | Immutable, testable, mandatory deps clear |
| **Setter Injection** | Via setter method | ⚠️ Sometimes | For optional dependencies |
| **Field Injection** | Via `@Autowired` on field | ❌ Avoid | Hides deps, hard to test, no immutability |

```java
// ✅ Constructor Injection (Preferred)
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final EmailService emailService;

    // @Autowired is optional when there's only one constructor (Spring 4.3+)
    public OrderService(OrderRepository orderRepository, EmailService emailService) {
        this.orderRepository = orderRepository;
        this.emailService = emailService;
    }
}

// ⚠️ Setter Injection (for optional deps)
@Service
public class ReportService {
    private NotificationService notificationService;

    @Autowired(required = false)
    public void setNotificationService(NotificationService ns) {
        this.notificationService = ns;
    }
}

// ❌ Field Injection (avoid)
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository; // Cannot be final, hard to test
}
```

---

## 3. Spring Bean & Scopes

**Definition:** Any object managed by the Spring IoC Container is called a **Bean**. Beans are created, configured, and managed by Spring based on configuration metadata.

### Bean Scopes

| Scope | One instance per... | Use Case |
|-------|---------------------|----------|
| `singleton` | Spring context (default) | Stateless services, repositories |
| `prototype` | Request to the container | Stateful objects, command objects |
| `request` | HTTP request | Web request-specific data |
| `session` | HTTP session | User session data |
| `application` | `ServletContext` | Application-wide shared state |
| `websocket` | WebSocket session | WebSocket communication data |

```java
@Bean
@Scope("prototype")  // New instance every time it's requested
public ShoppingCart shoppingCart() {
    return new ShoppingCart();
}
```

### Bean Lifecycle Callbacks

```java
@Component
public class DatabaseConnectionPool {

    @PostConstruct          // Called after all dependencies are injected
    public void initialize() {
        System.out.println("Initializing connection pool...");
    }

    @PreDestroy             // Called before the bean is destroyed
    public void cleanup() {
        System.out.println("Closing all connections...");
    }
}
```

---

## 4. ApplicationContext vs BeanFactory

| Feature | `BeanFactory` | `ApplicationContext` |
|---------|--------------|---------------------|
| Bean initialization | Lazy (on request) | Eager (at startup) |
| Annotation support | Limited | ✅ Full |
| AOP integration | Limited | ✅ Full |
| Event publishing | ❌ No | ✅ Yes |
| Internationalization (i18n) | ❌ No | ✅ Yes |
| `@Autowired`, `@Value` | ❌ No | ✅ Yes |
| Recommended for | Low-memory unit tests | All production use |

### ApplicationContext Implementations

| Class | Used When |
|-------|-----------|
| `AnnotationConfigApplicationContext` | Java-based configuration |
| `ClassPathXmlApplicationContext` | XML-based configuration |
| `WebApplicationContext` | Web applications |
| `SpringApplication` (Boot) | Spring Boot apps (wraps all of the above) |

---

## 5. AOP — Aspect-Oriented Programming

**Definition:** A programming paradigm that separates **cross-cutting concerns** (logging, security, transactions, metrics) from core business logic.

### Key AOP Terms

| Term | Definition | Example |
|------|-----------|---------|
| **Aspect** | Module for a cross-cutting concern | `LoggingAspect`, `SecurityAspect` |
| **Advice** | The action taken at a join point | Log a message, check permissions |
| **Join Point** | A specific point in execution | Method call, exception thrown |
| **Pointcut** | Expression matching join points | "All methods in `service` package" |
| **Weaving** | Linking aspects to target objects | Done by Spring at runtime |

### Advice Types

| Advice | When It Runs | Use Case |
|--------|-------------|----------|
| `@Before` | Before method executes | Input validation, logging |
| `@After` | After method (always) | Resource cleanup |
| `@AfterReturning` | After successful return | Log result, transform output |
| `@AfterThrowing` | After exception is thrown | Error logging, alerting |
| `@Around` | Wraps the method | Timing, retry, caching, transactions |

```java
@Aspect
@Component
public class PerformanceAspect {

    // Reusable pointcut
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() {}

    // @Around is the most powerful advice
    @Around("serviceLayer()")
    public Object measureTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = pjp.proceed();   // Call the actual method
        long elapsed = System.currentTimeMillis() - start;
        log.info("{} took {} ms", pjp.getSignature().getName(), elapsed);
        return result;
    }
}
```

### ⚠️ AOP Limitations in Spring

- **Self-invocation is not proxied** — calling a `@Transactional` method from the same class bypasses the proxy
- **Final methods/classes** — cannot be proxied by CGLIB
- **Private methods** — cannot be intercepted
- **Only Spring beans** — non-Spring objects are not proxied

---

## 6. Transaction Management

**Definition:** Spring provides **declarative** transaction management via `@Transactional`, abstracting away the underlying API (JDBC, JPA, Hibernate).

### Propagation Behaviours

| Propagation | Existing Transaction? | Behaviour |
|-------------|----------------------|-----------|
| `REQUIRED` *(default)* | Yes | Joins it |
| `REQUIRED` | No | Creates new |
| `REQUIRES_NEW` | Yes | Suspends it, creates new |
| `REQUIRES_NEW` | No | Creates new |
| `NESTED` | Yes | Creates savepoint inside it |
| `SUPPORTS` | Yes | Joins it |
| `SUPPORTS` | No | Runs non-transactionally |
| `MANDATORY` | No | Throws exception |
| `NOT_SUPPORTED` | Yes | Suspends it, runs non-transactionally |
| `NEVER` | Yes | Throws exception |

### Isolation Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|-----------|---------------------|--------------|
| `READ_UNCOMMITTED` | ✅ Possible | ✅ Possible | ✅ Possible |
| `READ_COMMITTED` | ❌ Prevented | ✅ Possible | ✅ Possible |
| `REPEATABLE_READ` | ❌ Prevented | ❌ Prevented | ✅ Possible |
| `SERIALIZABLE` | ❌ Prevented | ❌ Prevented | ❌ Prevented |

```java
@Service
public class PaymentService {

    // Full transaction configuration
    @Transactional(
        propagation  = Propagation.REQUIRED,
        isolation    = Isolation.READ_COMMITTED,
        readOnly     = false,
        rollbackFor  = Exception.class,
        noRollbackFor = ValidationException.class,
        timeout      = 30
    )
    public void processPayment(Payment payment) { ... }

    // Read-only optimization (no flush, faster)
    @Transactional(readOnly = true)
    public List<Payment> getHistory(Long userId) { ... }
}
```

> ⚠️ **Common Pitfall:** `@Transactional` only rolls back on `RuntimeException` by default. Checked exceptions do NOT trigger rollback unless you add `rollbackFor = Exception.class`.

---

## 7. Spring MVC & DispatcherServlet

**Spring MVC** is a web framework implementing the Model-View-Controller pattern, built on the Servlet API.

### DispatcherServlet Flow

```
HTTP Request
     │
     ▼
DispatcherServlet  (Front Controller — single entry point)
     │
     ▼
HandlerMapping     (finds the correct @Controller method)
     │
     ▼
HandlerAdapter     (invokes the method, handles @RequestBody etc.)
     │
     ▼
@Controller / @RestController method executes
     │
     ▼
ViewResolver       (for @Controller — resolves view name to template)
     │             (for @RestController — skipped; response written directly)
     ▼
HTTP Response
```

### MVC vs REST

| Feature | `@Controller` | `@RestController` |
|---------|--------------|------------------|
| Return type | View name (String) | Object → JSON/XML |
| Includes `@ResponseBody` | ❌ No | ✅ Yes |
| Used for | Server-side rendered pages | REST APIs |

---

## 8. Spring Data Concepts

### Repository Hierarchy

```
Repository  (marker interface)
    └── CrudRepository       — save, findById, findAll, delete
          └── PagingAndSortingRepository  — + findAll(Pageable), findAll(Sort)
                └── JpaRepository        — + flush, saveAndFlush, deleteInBatch
```

### Key Terms

| Term | Definition |
|------|-----------|
| **Entity** | Java class mapped to a DB table via `@Entity` |
| **Repository** | Interface for data access; Spring generates implementation |
| **JPQL** | JPA's object-oriented query language (operates on entities, not tables) |
| **Native Query** | Raw SQL query; set `nativeQuery = true` on `@Query` |
| **Projection** | Fetching only specific fields instead of full entities |
| **Specification** | Dynamic query builder using the Criteria API |
| **Auditing** | Auto-populating `createdAt`, `updatedAt`, `createdBy` fields |

---

## 9. Externalized Configuration

**Definition:** Spring Boot's ability to configure an application from outside the packaged JAR using property files, environment variables, or command-line arguments.

### Priority Order (Highest → Lowest)

```
1. Command-line arguments           --server.port=9090
2. SPRING_APPLICATION_JSON          (env var with inline JSON)
3. OS environment variables         SERVER_PORT=9090
4. application-{profile}.properties
5. application.properties / .yml
6. @PropertySource annotations
7. Default properties               SpringApplication.setDefaultProperties()
```

> The **higher** in the list, the higher the priority — command-line args **always win**.

---

## 10. Other Important Terms

| Term | Definition |
|------|-----------|
| **Fat JAR / Uber JAR** | Self-contained executable JAR with app + deps + embedded server |
| **Auto-Configuration** | Spring Boot's mechanism to configure beans based on classpath contents |
| **Starter POM** | Pre-packaged dependency set for a feature (e.g., `spring-boot-starter-web`) |
| **Profile** | Named environment configuration (dev, test, prod) |
| **Actuator** | Sub-project for production-ready monitoring and management endpoints |
| **SpEL** | Spring Expression Language — used in `@Value`, `@ConditionalOnExpression`, security rules |
| **HATEOAS** | Hypermedia links in REST responses; supported via `spring-boot-starter-hateoas` |
| **Flyway / Liquibase** | Database schema version control tools |
| **DevTools** | Developer tooling for hot reload and LiveReload |
| **Embedded Server** | Web server (Tomcat, Jetty, Undertow) packaged inside the JAR |

---

[← Introduction](./01_Introduction.md) &nbsp;|&nbsp; [Next → Architecture & Auto-Configuration](./03_Architecture_AutoConfiguration.md)
