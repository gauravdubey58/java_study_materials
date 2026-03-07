# 15 — 100 Interview Questions

[← Back to Index](./README.md)

> **Q1–Q50:** Moderate difficulty — concepts, annotations, Spring Data, Security basics  
> **Q51–Q100:** Tough — internals, edge cases, architecture, production scenarios

---

## 🟡 Moderate Level (Q1 – Q50)

---

**Q1. What is Spring Boot and how does it differ from the Spring Framework?**

Spring Boot is an opinionated extension of the Spring Framework that provides auto-configuration, embedded servers, and starter dependencies to minimize boilerplate. The Spring Framework requires explicit configuration for everything. Spring Boot follows "convention over configuration," allowing developers to start a production-ready app in minutes.

---

**Q2. What does `@SpringBootApplication` do internally?**

It is a meta-annotation that combines three annotations:
- `@Configuration` — Marks the class as a configuration source
- `@EnableAutoConfiguration` — Triggers auto-configuration based on classpath
- `@ComponentScan` — Scans the current package and sub-packages for components

---

**Q3. What is auto-configuration in Spring Boot? How do you debug it?**

Auto-configuration automatically creates and configures Spring beans based on what's on the classpath. Each auto-configuration class uses `@ConditionalOn*` annotations to decide whether to apply. Debug by running with `--debug` flag or `debug=true` property — it prints a **Conditions Evaluation Report**.

---

**Q4. What is the property resolution order in Spring Boot?**

From highest to lowest priority:
1. Command-line arguments (`--server.port=9090`)
2. `SPRING_APPLICATION_JSON` environment variable
3. OS environment variables
4. `application-{profile}.properties`
5. `application.properties`
6. `@PropertySource` annotations
7. Default properties

---

**Q5. What is the difference between `@Component`, `@Service`, `@Repository`, and `@Controller`?**

All are specializations of `@Component` and enable component scanning. Key differences:
- `@Repository` — Enables persistence exception translation (wraps JPA/JDBC exceptions into Spring's `DataAccessException`)
- `@Service` — Semantic marker for business logic; no additional Spring behavior
- `@Controller` — Handles HTTP requests; works with `ViewResolver`
- `@Component` — Generic; use when no specific role fits

---

**Q6. What are Spring Boot Starters? Give five examples.**

Starters are pre-packaged Maven/Gradle dependency descriptors that bring in a set of compatible, commonly used libraries. Examples:
- `spring-boot-starter-web` — MVC + Jackson + Tomcat
- `spring-boot-starter-data-jpa` — JPA + Hibernate + HikariCP
- `spring-boot-starter-security` — Spring Security
- `spring-boot-starter-test` — JUnit5 + Mockito + MockMvc
- `spring-boot-starter-actuator` — Health + metrics endpoints

---

**Q7. How do you create a REST API in Spring Boot?**

1. Add `spring-boot-starter-web`
2. Create a class with `@RestController`
3. Map endpoints with `@GetMapping`, `@PostMapping`, etc.
4. Use `@RequestBody` for request deserialization and `@PathVariable`/`@RequestParam` for URL params

---

**Q8. What is the difference between `@Controller` and `@RestController`?**

`@RestController` = `@Controller` + `@ResponseBody`. In `@Controller`, methods return view names resolved by a `ViewResolver`. In `@RestController`, methods return data objects serialized directly to JSON/XML in the response body.

---

**Q9. What is `ResponseEntity`? When would you use it?**

`ResponseEntity<T>` gives full control over the HTTP response — status code, headers, and body. Use it when you need to set a specific status code (e.g., 201 Created with a Location header), set custom headers, or conditionally return different statuses.

```java
return ResponseEntity.created(URI.create("/api/users/" + id))
    .header("X-Custom", "value")
    .body(created);
```

---

**Q10. What is the difference between `@RequestParam` and `@PathVariable`?**

- `@PathVariable` — Extracts from the URI path: `GET /users/{id}` → `@PathVariable Long id`
- `@RequestParam` — Extracts from query string: `GET /users?page=0` → `@RequestParam int page`

---

**Q11. How does `spring.jpa.hibernate.ddl-auto` work? What are the options?**

Controls how Hibernate manages the database schema at startup:
- `none` — Do nothing (production best practice)
- `validate` — Validate schema against entities; fail if mismatch
- `update` — Update schema to match entities (risky in prod)
- `create` — Drop and create schema on startup
- `create-drop` — Create on start, drop on close (for tests)

---

**Q12. What is the difference between JPA and Hibernate?**

JPA is a specification (API standard) defining how Java objects map to relational databases. Hibernate is the most popular implementation of that specification. Spring Data JPA adds a higher-level abstraction (repository pattern) on top of JPA.

---

**Q13. What is `spring.jpa.open-in-view` and why should it be false?**

When `true` (default), the EntityManager is kept open for the entire HTTP request including view rendering, allowing lazy loading in templates. This is problematic because it causes extra DB queries during rendering, holds connections longer, and hides performance issues. Set it to `false` and load all required data in the service layer.

---

**Q14. What are the three types of dependency injection? Which is preferred?**

- **Constructor injection** — Dependencies in constructor (✅ preferred: immutable, clear, testable)
- **Setter injection** — Via setter method (use for optional dependencies)
- **Field injection** — `@Autowired` on field (❌ avoid: hides deps, can't be `final`, hard to test)

---

**Q15. Explain `@Transactional` and its key attributes.**

Marks a method/class as transactional. Spring creates a proxy that manages transaction boundaries. Key attributes:
- `propagation` — How to handle existing transactions (default: `REQUIRED`)
- `isolation` — Data visibility between concurrent transactions
- `readOnly` — Optimize for reads (no dirty checking, no flush)
- `rollbackFor` — Which exceptions trigger rollback (default: `RuntimeException`)
- `timeout` — Max duration in seconds before rollback

---

**Q16. What is the difference between `REQUIRES_NEW` and `NESTED` propagation?**

- `REQUIRES_NEW` — Always creates a completely independent new transaction. The outer transaction is suspended. If the inner commits, its changes persist even if the outer rolls back.
- `NESTED` — Creates a savepoint within the existing transaction. A rollback of the nested part rolls back to the savepoint only, not the entire outer transaction.

---

**Q17. What is Spring Boot Actuator? Name five endpoints.**

Actuator provides production-ready management features via HTTP endpoints. Key endpoints:
- `/actuator/health` — Application health status
- `/actuator/metrics` — JVM and application metrics
- `/actuator/env` — Environment properties
- `/actuator/loggers` — View/change log levels at runtime
- `/actuator/beans` — All beans in the context

---

**Q18. How do you secure Actuator endpoints in production?**

1. Expose only needed endpoints: `management.endpoints.web.exposure.include=health,info`
2. Run Actuator on a separate internal port: `management.server.port=9090`
3. Bind to localhost only: `management.server.address=127.0.0.1`
4. Protect with Spring Security roles

---

**Q19. What is the difference between `CommandLineRunner` and `ApplicationRunner`?**

Both execute after the Spring context starts. `CommandLineRunner.run(String... args)` receives raw string arguments. `ApplicationRunner.run(ApplicationArguments args)` receives a structured `ApplicationArguments` object with parsed options. Both can be ordered with `@Order`.

---

**Q20. How do you enable caching in Spring Boot?**

1. Add `@EnableCaching` to a configuration class
2. Add a cache provider (Redis, Caffeine, etc.) or use the default in-memory cache
3. Annotate service methods with `@Cacheable`, `@CachePut`, or `@CacheEvict`

---

**Q21. What is the N+1 query problem? How do you fix it?**

When loading N entities, each triggers 1 additional query to load a related association = N+1 total queries. Fix with:
- `JOIN FETCH` in JPQL: `SELECT u FROM User u JOIN FETCH u.orders`
- `@EntityGraph(attributePaths = {"orders"})` on repository method
- `@BatchSize(size = 20)` on the collection field

---

**Q22. What is the difference between `@Value` and `@ConfigurationProperties`?**

- `@Value("${app.name}")` — Injects a single property; supports SpEL; tightly coupled to property name
- `@ConfigurationProperties(prefix = "app")` — Binds a group of related properties to a POJO; supports type safety, validation, relaxed binding, and IDE autocomplete

---

**Q23. How do you implement global exception handling in Spring Boot?**

Use `@RestControllerAdvice` with `@ExceptionHandler` methods. These apply to all controllers globally. This centralizes all error handling and allows returning consistent error response structures.

---

**Q24. What changed in Spring Security 5.7 (Spring Boot 2.7)?**

`WebSecurityConfigurerAdapter` was deprecated. The new approach uses `SecurityFilterChain` and `WebSecurityCustomizer` as beans. `@EnableMethodSecurity` replaces `@EnableGlobalMethodSecurity`. This allows composing multiple security configurations without inheritance.

---

**Q25. What is the difference between authentication and authorization?**

- **Authentication** — Verifying identity ("who are you?") — checking username/password, JWT, certificate
- **Authorization** — Verifying permissions ("what can you do?") — roles, authorities, access control rules

Spring Security handles both: authentication via `AuthenticationManager`, authorization via `AccessDecisionManager`/`AuthorizationFilter`.

---

**Q26. What is `@Async` and how do you enable it?**

`@Async` marks a method to execute asynchronously in a Spring-managed thread pool. Enable with `@EnableAsync`. The calling thread returns immediately without waiting for the method to complete. Return `CompletableFuture<T>` or `void`.

---

**Q27. What are the ways to configure `@Scheduled`?**

- `fixedRate` — Runs every N milliseconds from the start time of the last invocation
- `fixedDelay` — Runs N milliseconds after the last invocation completes
- `cron` — Cron expression (e.g., `"0 0 2 * * ?"` = 2 AM daily)
- `initialDelay` — Delay before first execution

Requires `@EnableScheduling`.

---

**Q28. How do Spring Boot Profiles work?**

Profiles allow environment-specific configuration. Define `application-dev.yml`, `application-prod.yml`, etc. Activate via `spring.profiles.active=prod` property, `SPRING_PROFILES_ACTIVE` env var, or command-line `--spring.profiles.active=prod`. Beans can also be profile-scoped with `@Profile("dev")`.

---

**Q29. What is `@ConditionalOnProperty`?**

Creates a bean only if a specified property has a certain value. Used for feature flags and conditional configuration.

```java
@Bean
@ConditionalOnProperty(name = "feature.payment.v2.enabled", havingValue = "true")
public PaymentServiceV2 paymentServiceV2() { ... }
```

---

**Q30. What is a Spring Boot Fat JAR?**

A self-contained executable JAR built by the `spring-boot-maven-plugin`. It contains the application code, all dependencies, and the embedded server. Run with `java -jar app.jar` on any machine with Java — no external server needed.

---

**Q31. What is the difference between `Page<T>` and `Slice<T>`?**

`Page<T>` executes an additional `SELECT COUNT(*)` to calculate total pages and elements. `Slice<T>` only knows if there's a next page (no count query). Use `Slice` for infinite scroll / "Load more" UI where total count isn't needed.

---

**Q32. What is `@Query` in Spring Data JPA? When do you need it?**

`@Query` defines custom JPQL or native SQL on a repository method. Use it when the derived query method name would be too complex, or when you need specific JOINs, aggregations, or native SQL features.

---

**Q33. How does Spring Boot select which beans to auto-configure?**

It reads all auto-configuration class names from `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`. For each class, it evaluates `@ConditionalOn*` annotations. Only if all conditions pass does the configuration get applied.

---

**Q34. What is the role of `DispatcherServlet` in Spring MVC?**

`DispatcherServlet` is the front controller. It receives all HTTP requests, uses `HandlerMapping` to find the right controller method, delegates execution via `HandlerAdapter`, then uses `ViewResolver` to render a response. All Spring MVC beans (controllers, interceptors) are registered with it.

---

**Q35. What does HTTP 401 vs 403 mean?**

- **401 Unauthorized** — The request lacks valid authentication credentials. The user is not logged in.
- **403 Forbidden** — The user is authenticated but does not have permission to access the resource.

---

**Q36. What is `@CrossOrigin` and when is it needed?**

It enables CORS (Cross-Origin Resource Sharing) for a controller or method. Needed when your frontend (e.g., React on `localhost:3000`) calls your backend API on a different origin. Without it, browsers block the request.

---

**Q37. What is `@DataJpaTest`?**

A test slice annotation that loads only the JPA layer (repositories, entity manager, JPA configuration). Uses an embedded H2 database by default. Does NOT load the full application context — much faster than `@SpringBootTest`. Transactions are rolled back after each test.

---

**Q38. What does `@MockBean` do in Spring tests?**

Creates a Mockito mock and registers it as a bean in the Spring context, replacing any existing bean of that type. Used in `@WebMvcTest` to mock the service layer that the controller depends on.

---

**Q39. What is Flyway? Why use it over `ddl-auto=update`?**

Flyway is a database migration tool that applies versioned SQL scripts in order and tracks them in a `flyway_schema_history` table. Unlike `ddl-auto=update` (which may make unpredictable changes), Flyway migrations are explicit, version-controlled, and safe for production.

---

**Q40. What is the difference between `@NotNull`, `@NotEmpty`, and `@NotBlank`?**

- `@NotNull` — Cannot be null (empty string `""` is allowed)
- `@NotEmpty` — Cannot be null or empty (applies to String, Collection, Map, Array)
- `@NotBlank` — Cannot be null, empty, or whitespace-only (String only)

---

**Q41. How does Spring AOP work under the hood?**

Spring AOP uses proxies. For interfaces, it creates **JDK dynamic proxies**. For concrete classes, it uses **CGLIB** to generate a subclass at runtime. When a bean is requested from the context, Spring returns the proxy. The proxy intercepts method calls and applies the configured advice (before, after, around).

---

**Q42. What is `@Transient` in JPA?**

Marks a field that should NOT be persisted to the database. JPA ignores it for mapping. Useful for computed values, flags, or temporary data that only makes sense in memory.

---

**Q43. How do you configure multiple data sources in Spring Boot?**

Define two `DataSource` beans, mark one with `@Primary`. Define separate `EntityManagerFactory` and `PlatformTransactionManager` for each. Use `@EnableJpaRepositories(basePackages = "...", entityManagerFactoryRef = "...", transactionManagerRef = "...")` to assign repositories to their data sources.

---

**Q44. What is the difference between `save()` and `saveAndFlush()` in `JpaRepository`?**

`save()` marks the entity as managed; the actual SQL may be deferred until the transaction commits or Hibernate decides to flush. `saveAndFlush()` immediately synchronizes the change to the database within the current transaction — useful when you need the result visible immediately (e.g., for subsequent queries in the same transaction).

---

**Q45. What is Spring's Environment abstraction?**

`Environment` is an interface representing the application's runtime environment. It provides access to active profiles and property sources (properties files, environment variables, system properties, etc.). You can inject it or use `@Value` and `@ConfigurationProperties` which use it internally.

---

**Q46. What is HATEOAS?**

HATEOAS (Hypermedia as the Engine of Application State) is a REST constraint where API responses include links to related operations. For example, a GET user response might include links to update or delete that user. Spring Boot supports it via `spring-boot-starter-hateoas`.

---

**Q47. How do you integrate Swagger/OpenAPI documentation?**

Add `springdoc-openapi-ui` dependency. It auto-generates interactive API documentation from Spring MVC annotations and makes it available at `/swagger-ui.html` (or `/swagger-ui/index.html`). Customize with `@Operation`, `@ApiResponse`, `@Schema` annotations from OpenAPI 3.

---

**Q48. What is the difference between `spring.factories` and `AutoConfiguration.imports`?**

Both register auto-configuration classes. In Spring Boot 2.7+, the new `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` file is preferred. The old `spring.factories` file approach is deprecated for auto-configuration but still works. New auto-config classes should also use `@AutoConfiguration` instead of `@Configuration`.

---

**Q49. How does `@ConditionalOnMissingBean` enable user override of auto-configuration?**

Auto-configuration classes use `@ConditionalOnMissingBean` on their `@Bean` methods. If you define a bean of the same type in your code, Spring sees it first and the auto-configured bean is skipped. This allows you to replace any auto-configured bean without touching the library code.

---

**Q50. What does `@EnableJpaAuditing` do?**

Enables Spring Data JPA's auditing feature, which auto-populates `@CreatedDate`, `@LastModifiedDate`, `@CreatedBy`, and `@LastModifiedBy` fields on entities annotated with `@EntityListeners(AuditingEntityListener.class)`. The `AuditorAware<T>` bean provides the current user's identity.

---

## 🔴 Tough Level (Q51 – Q100)

---

**Q51. Walk through Spring Boot's auto-configuration mechanism step by step.**

1. `@SpringBootApplication` includes `@EnableAutoConfiguration`
2. `AutoConfigurationImportSelector` is registered as an `ImportSelector`
3. It reads all class names from `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
4. For each class, it evaluates every `@ConditionalOn*` annotation
5. Classes whose conditions all pass are included in the context as `@Configuration` classes
6. Their `@Bean` methods are invoked, creating beans
7. `@ConditionalOnMissingBean` ensures user-defined beans override auto-configured ones
8. The final result is a fully configured context assembled from both user and auto-configured beans

---

**Q52. Describe the complete Spring Bean lifecycle.**

```
1.  Instantiation           → Constructor called
2.  Property Population     → @Autowired / setter injection
3.  BeanNameAware           → setBeanName() if implemented
4.  BeanFactoryAware        → setBeanFactory() if implemented
5.  ApplicationContextAware → setApplicationContext() if implemented
6.  BeanPostProcessor       → postProcessBeforeInitialization()
7.  InitializingBean        → afterPropertiesSet() if implemented
8.  @PostConstruct          → Custom init logic
9.  BeanPostProcessor       → postProcessAfterInitialization() → AOP proxy created here
    ─── Bean is READY for use ───
10. @PreDestroy             → Custom cleanup
11. DisposableBean          → destroy() if implemented
```

---

**Q53. What is the difference between `BeanPostProcessor` and `BeanFactoryPostProcessor`?**

- `BeanFactoryPostProcessor` — Runs before any beans are created. It receives the `BeanFactory` and can modify bean definitions (metadata). Example: `PropertySourcesPlaceholderConfigurer` resolves `${...}` placeholders in bean definitions.
- `BeanPostProcessor` — Runs after each bean is instantiated. It receives the bean instance and can wrap or modify it. Spring uses this internally for `@Autowired` injection, AOP proxy creation, and `@PostConstruct`.

---

**Q54. Why is field injection with `@Autowired` considered bad practice?**

1. **Hidden dependencies** — You cannot see all dependencies without reading the class body
2. **No immutability** — Fields cannot be `final`
3. **Difficult to test** — Cannot inject mocks via constructor; requires reflection or Spring context
4. **Circular dependency concealment** — Spring may silently handle circular deps that should be caught
5. **SRP encouragement failure** — Easy to add many deps without noticing the class is doing too much

---

**Q55. How does Spring resolve circular dependencies? When does it fail?**

For **singleton** beans with setter or field injection, Spring uses a **three-level cache**:
1. `singletonObjects` — Fully initialized beans
2. `earlySingletonObjects` — Partially initialized beans (exposed early)
3. `singletonFactories` — Factories that can create early bean references

When A depends on B and B depends on A, Spring creates A, puts a factory for it in Level 3, starts creating B, needs A's reference, calls the factory to get an early reference, completes B, finishes A.

**Fails for constructor injection** — Spring cannot create any instance first, so `BeanCurrentlyInCreationException` is thrown. Fix: use `@Lazy` on one constructor parameter or restructure the code.

---

**Q56. Explain Spring's three-level singleton cache.**

The three caches handle circular dependencies for singletons:
- **Level 1 (`singletonObjects`)** — Fully initialized, production-ready beans
- **Level 2 (`earlySingletonObjects`)** — Beans exposed early (as proxies) to break circular deps
- **Level 3 (`singletonFactories`)** — `ObjectFactory` lambdas that, when called, return an early reference

When Spring detects a circular dependency, it calls the factory in Level 3 to get an early (possibly not yet fully initialized) reference of the bean and stores it in Level 2 for the dependent bean to use.

---

**Q57. What are the limitations of Spring AOP compared to AspectJ?**

| Limitation | Spring AOP | AspectJ |
|-----------|------------|---------|
| Self-invocation | ❌ Not intercepted | ✅ Intercepted |
| Private methods | ❌ Cannot intercept | ✅ Can intercept |
| Final classes | ❌ CGLIB cannot proxy | ✅ Can instrument |
| Non-Spring beans | ❌ Not proxied | ✅ Any object |
| Field access | ❌ No | ✅ Yes |
| Weaving type | Runtime proxy | Compile/load-time |

---

**Q58. How do you solve the AOP self-invocation problem?**

When a method in a class calls another method in the same class annotated with `@Transactional` or other AOP annotation, the proxy is bypassed. Solutions:
1. Inject the bean into itself: `@Autowired private MyService self;` (with `@Lazy`)
2. Use `AopContext.currentProxy()`: `((MyService) AopContext.currentProxy()).myMethod();`
3. Restructure the code to avoid self-invocation
4. Use AspectJ compile-time weaving instead

---

**Q59. Why does `@Transactional` on a `private` method not work?**

Spring AOP creates a proxy around the bean. Proxies can only intercept `public` method calls from outside the bean. Private methods are not part of the proxy — they're called directly on the target object, bypassing the proxy entirely. Spring silently ignores `@Transactional` on private/protected methods without error.

---

**Q60. What happens when a `@Transactional` method calls another `@Transactional` method within the same class?**

The inner method call goes to `this` (the real object), not the proxy. The proxy's transaction management is bypassed. The inner method's transaction annotation has no effect — it runs within the outer transaction regardless of propagation settings. This is the self-invocation proxy limitation.

---

**Q61. What is optimistic locking? How is it implemented in JPA?**

Optimistic locking assumes conflicts are rare. It doesn't lock rows in the database. Instead, it uses a `@Version` field (Long or Integer). On every UPDATE, Hibernate adds a `WHERE version = ?` clause. If another transaction modified the row in between (incrementing the version), the update affects 0 rows and Hibernate throws `OptimisticLockException`. The application can retry.

---

**Q62. What is `@EntityGraph` and how does it differ from `JOIN FETCH`?**

Both solve N+1 queries. `@EntityGraph` is a named or ad-hoc graph that specifies which associations to load eagerly for a particular query. It's more reusable and can be applied at the repository method level without changing the JPQL. `JOIN FETCH` is inline in the JPQL query. Both generate a single JOIN query. `@EntityGraph` is cleaner when the same base query needs to be reused with different fetch strategies.

---

**Q63. What is `@DynamicUpdate` and when should you use it?**

`@DynamicUpdate` (Hibernate-specific) generates UPDATE statements with only the columns that actually changed, instead of updating all columns. Use it for entities with many columns where most updates touch only a few fields — reduces unnecessary DB writes and potential lock contention.

---

**Q64. Explain the JPA `@Inheritance` strategies and their tradeoffs.**

| Strategy | Description | Pros | Cons |
|----------|-------------|------|------|
| `SINGLE_TABLE` | All subclasses in one table | Fast queries, no JOINs | Nullable columns, wasted space |
| `JOINED` | Each class has its own table | Normalized | JOIN overhead per query |
| `TABLE_PER_CLASS` | Each concrete class in own table | No JOINs for direct type queries | No polymorphic queries, no shared sequences |

---

**Q65. How does the `@Transactional` rollback rule work for checked exceptions?**

By default, `@Transactional` only rolls back for `RuntimeException` and `Error`. Checked exceptions (e.g., `IOException`) do NOT trigger rollback — the transaction commits even if a checked exception propagates. Fix: `@Transactional(rollbackFor = Exception.class)` to roll back on all exceptions.

---

**Q66. How do you implement a custom Spring Boot auto-configuration for a library?**

1. Create a class annotated with `@AutoConfiguration` and `@ConditionalOn*` annotations
2. Define `@Bean` methods with `@ConditionalOnMissingBean` to allow user override
3. Create `@ConfigurationProperties` for config
4. Register in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
5. Add `@EnableConfigurationProperties` to bind the properties class

---

**Q67. What is the `ApplicationEventPublisher` pattern? Why use events instead of direct calls?**

Events decouple the publisher from consumers. The service publishing an event doesn't know about or depend on the listeners. Benefits: easier to add new behaviors without modifying the publisher, supports async processing, works well with transactions (`@TransactionalEventListener` fires after commit). Downside: harder to trace the flow.

---

**Q68. What is `@TransactionalEventListener` and when should you use it?**

It listens for application events but only processes them after the current transaction phase completes (default: `AFTER_COMMIT`). Use it when you want to perform side effects (send email, call external API) only if the transaction that published the event actually committed — not if it rolled back.

---

**Q69. What is WebClient and how does it differ from RestTemplate?**

`WebClient` is the non-blocking, reactive HTTP client from Spring WebFlux. It supports both reactive (Mono/Flux) and synchronous usage. `RestTemplate` is the blocking, synchronous client — deprecated for new code. `WebClient` handles streaming, back-pressure, and is more efficient for high-concurrency scenarios.

---

**Q70. How do you implement soft delete in Spring Boot?**

```java
@Entity
@SQLDelete(sql = "UPDATE users SET deleted_at = NOW() WHERE id = ?")
@Where(clause = "deleted_at IS NULL")
public class User {
    private LocalDateTime deletedAt;
}
```

`@SQLDelete` overrides DELETE with an UPDATE. `@Where` adds a filter to all JPQL queries so deleted records are invisible. Note: native queries bypass `@Where`.

---

**Q71. What is graceful shutdown in Spring Boot 2.7? How do you configure it?**

Graceful shutdown waits for active requests to complete before shutting down the server. Configure:
```properties
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=20s
```
On receiving a SIGTERM, the server stops accepting new requests and waits up to 20 seconds for in-flight requests to finish.

---

**Q72. How does Spring handle lazy loading outside of a transaction (LazyInitializationException)?**

When accessing a lazy-loaded association after the EntityManager/persistence session is closed, Hibernate throws `LazyInitializationException` because it cannot open a new connection. Solutions:
- Fetch the data within a `@Transactional` method
- Use `JOIN FETCH` or `@EntityGraph` to eagerly load it
- Use DTOs/projections and map needed data before the session closes
- (Avoid) Enable `spring.jpa.open-in-view=true`

---

**Q73. Explain how JWT authentication integrates with Spring Security's filter chain.**

A custom filter (extending `OncePerRequestFilter`) is added before `UsernamePasswordAuthenticationFilter`. It extracts the JWT from the `Authorization: Bearer` header, validates it (signature, expiry), extracts the username, loads `UserDetails`, creates a `UsernamePasswordAuthenticationToken`, and sets it in `SecurityContextHolder`. Subsequent filters (including the authorization filter) see the authenticated user.

---

**Q74. What is method security and what is the difference between `@PreAuthorize` and `@PostAuthorize`?**

Both use SpEL for access control. `@PreAuthorize` is evaluated before the method runs — if it fails, the method is never called. `@PostAuthorize` is evaluated after the method runs and has access to `returnObject` — if it fails, the result is discarded and an exception is thrown. Use `@PostAuthorize` when the access decision depends on the returned data (e.g., owner check).

---

**Q75. What is the difference between `@Cacheable`'s `condition` and `unless`?**

- `condition` — Evaluated before the method. If false, caching is skipped entirely (method always executes, result never cached).
- `unless` — Evaluated after the method. If true, the result is NOT stored in cache (method always executes, but result may be discarded). Use `unless = "#result == null"` to avoid caching null results.

---

**Q76. How do Spring Boot Kubernetes liveness and readiness probes work?**

Spring Boot 2.3+ tracks internal state:
- **Liveness** (`/actuator/health/liveness`) — Is the app running correctly? DOWN = something is broken, restart the pod
- **Readiness** (`/actuator/health/readiness`) — Is the app ready to serve traffic? DOWN = remove from load balancer (e.g., during startup or maintenance)

The app itself can change its readiness state: `applicationContext.publishEvent(new AvailabilityChangeEvent<>(ReadinessState.REFUSING_TRAFFIC, this))`.

---

**Q77. What is `@Retryable` and how does it work internally?**

From Spring Retry (`spring-retry`). `@Retryable` is an AOP-backed annotation that retries a method on specified exceptions. Configure `maxAttempts`, `backoff` (delay/multiplier). Spring creates a proxy that catches the specified exceptions, waits the backoff duration, and retries up to `maxAttempts` times. If all attempts fail, the `@Recover` method is called as a fallback.

---

**Q78. What is the difference between `@SpringBootTest` with `MOCK`, `RANDOM_PORT`, `DEFINED_PORT`, and `NONE`?**

| Mode | Server | Use Case |
|------|--------|---------|
| `MOCK` (default) | Mock servlet env, no real server | MockMvc-based controller tests |
| `RANDOM_PORT` | Real server on random port | Full integration tests with `TestRestTemplate` |
| `DEFINED_PORT` | Real server on `server.port` | When specific port matters |
| `NONE` | No web context at all | Pure service/repository tests |

---

**Q79. How do you implement idempotency for REST POST requests?**

The client generates a unique `Idempotency-Key` UUID and sends it as a header. The server:
1. Checks Redis/DB for the key
2. If found, returns the cached response (no processing)
3. If not found, processes the request, stores the response against the key, returns the response

The key should expire after a reasonable time window (e.g., 24 hours).

---

**Q80. What is `@Modifying(clearAutomatically = true)` and why is it important?**

`@Modifying` is required for UPDATE/DELETE `@Query` methods. Without `clearAutomatically = true`, after a bulk update via JPQL, the first-level cache (persistence context) still holds the old entity state. Subsequent `findById` calls return stale data from the cache, not the updated DB values. Setting `clearAutomatically = true` clears the persistence context after the modifying query, forcing a fresh load.

---

**Q81. How do Spring's conditional annotations interact? What if multiple conditions are on one class?**

All conditions must be satisfied simultaneously (AND logic). If any one `@ConditionalOn*` annotation evaluates to false, the entire configuration class is skipped. To express OR logic, you need to implement a custom `Condition` using the `@Conditional(MyCondition.class)` annotation with a class that implements `Condition`.

---

**Q82. What is Spring Boot's relaxed binding?**

Spring Boot maps property names flexibly. The property `spring.datasource.url` can be set as:
- `spring.datasource.url` (standard)
- `SPRING_DATASOURCE_URL` (OS env var convention)
- `spring.datasource.URL` (different case)
- `spring_datasource_url` (underscores)

All resolve to the same bean property. This makes the app work seamlessly across different platforms (YAML, env vars, Docker, Kubernetes ConfigMaps).

---

**Q83. Explain multi-tenancy approaches in Spring Boot.**

Three strategies:
1. **Separate databases** — One DB per tenant. Switch `DataSource` based on tenant ID in thread-local. Highest isolation.
2. **Separate schemas** — Single DB, one schema per tenant. Hibernate's `MultiTenantConnectionProvider` sets the schema per request.
3. **Shared schema** — All tenants in same tables with a `tenant_id` column. Use Hibernate filters or Spring Data JPA Specifications to filter automatically. Lowest isolation but simplest operations.

Store the tenant identifier in a `ThreadLocal`, populate it from JWT claims or subdomain in a filter.

---

**Q84. What is Spring's `@Lazy` annotation and when should you use it?**

`@Lazy` defers bean creation until the first time it's requested, instead of at startup. Uses:
1. **Circular dependency workaround** — `@Lazy` on a constructor parameter breaks the cycle
2. **Heavy beans** — Delay expensive initialization for beans not always needed
3. **Startup time optimization** — `spring.main.lazy-initialization=true` makes ALL beans lazy globally

---

**Q85. How do you create a custom Actuator health indicator group for Kubernetes?**

```yaml
management:
  endpoint:
    health:
      group:
        liveness:
          include: livenessState, diskSpace
        readiness:
          include: readinessState, db, redis, externalApi
      probes:
        enabled: true
```

Custom health indicators (`implements HealthIndicator`) are automatically included in health groups by name (bean name → health component name).

---

**Q86. What is `@ConfigurationPropertiesBinding`?**

Marks a Spring `Converter<String, T>` or `GenericConverter` to be used for custom type conversion during `@ConfigurationProperties` binding. Needed when you want to convert a property string (e.g., `"10.5.3"`) to a custom type (e.g., `Version` record).

---

**Q87. How do you configure a custom `ThreadPoolTaskExecutor` for `@Async`?**

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
        exec.setCorePoolSize(5);
        exec.setMaxPoolSize(20);
        exec.setQueueCapacity(100);
        exec.setThreadNamePrefix("async-");
        exec.setRejectedExecutionHandler(new CallerRunsPolicy());
        exec.initialize();
        return exec;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) ->
            log.error("Async error in {}: {}", method.getName(), ex.getMessage());
    }
}
```

---

**Q88. How does Spring Boot improve startup time? What are all the options?**

1. `spring.main.lazy-initialization=true` — Don't create beans until first use
2. Exclude unused auto-configurations with `@SpringBootApplication(exclude = ...)`
3. Minimize `@ComponentScan` scope with `scanBasePackages`
4. Use `spring-boot-devtools` in dev (faster restarts via classloader separation)
5. Compile-time indexing with `spring-boot-configuration-processor`
6. GraalVM native image — sub-second startup, low memory
7. JVM tuning: `-XX:TieredStopAtLevel=1` for faster JIT (trade runtime perf for startup speed)

---

**Q89. What is Spring's `@DependsOn` annotation and when is it necessary?**

`@DependsOn("beanA")` tells Spring to initialize `beanA` before the annotated bean, even if there's no explicit injection relationship. Necessary when beans have implicit initialization order dependencies (e.g., a schema initializer must run before a repository that depends on the schema existing).

---

**Q90. Explain how CSRF tokens work and when you should disable CSRF protection.**

CSRF tokens prevent cross-site request forgery. Spring Security generates a token stored in the session (or cookie). State-changing requests (POST/PUT/DELETE) must include this token. The server validates it. Browsers automatically include cookies but not custom headers — a cross-site request won't have the CSRF token.

Disable CSRF when:
- The app is a stateless REST API using JWT in the `Authorization` header (not cookies)
- Mobile apps or server-to-server APIs where there's no browser involved

---

**Q91. What is a Specification in Spring Data JPA? When is it better than `@Query`?**

`Specification<T>` wraps the JPA Criteria API into a composable predicate. Better than `@Query` when:
- Query criteria vary dynamically (optional filters, search APIs)
- You need to combine predicates programmatically: `spec1.and(spec2).or(spec3)`
- You want type-safe queries without string-based JPQL

---

**Q92. What is the difference between `@OneToMany(orphanRemoval = true)` and `CascadeType.REMOVE`?**

`CascadeType.REMOVE` deletes child entities when the parent is deleted. `orphanRemoval = true` additionally deletes child entities when they are removed from the parent's collection — even if the parent is not deleted. It treats removal from the collection as "orphaned" and schedules a DELETE. Use `orphanRemoval = true` when child entities' lifecycle is fully owned by the parent.

---

**Q93. How does Spring Boot's auto-configured `DataSource` get replaced by a user-defined one?**

`DataSourceAutoConfiguration` uses `@ConditionalOnMissingBean(DataSource.class)` on its `@Bean` method. When you define your own `@Bean` of type `DataSource`, Spring registers it first. When the auto-configuration class evaluates, the condition `@ConditionalOnMissingBean` is false (a DataSource bean already exists), so the auto-configured one is skipped.

---

**Q94. How do you profile Spring Boot startup to find slow-initializing beans?**

1. Add startup endpoint: `management.endpoint.startup.enabled=true`
2. After startup: `GET /actuator/startup` returns a JSON timeline of all startup steps with durations
3. Alternatively, implement `ApplicationStartup` and set it: `new SpringApplication(...).setApplicationStartup(BufferingApplicationStartup(2048))`
4. Use Spring Boot Admin or a profiler (JProfiler, YourKit) for deeper analysis

---

**Q95. What is `@Embedded` vs `@ElementCollection` in JPA?**

- `@Embedded` — Embeds an `@Embeddable` POJO's fields into the owning entity's table. No separate table. One-to-one relationship in code, same table in DB.
- `@ElementCollection` — Persists a collection of basic types or `@Embeddable` objects into a separate table. The collection has no identity of its own and is fully owned by the parent entity.

```java
// @ElementCollection example
@ElementCollection
@CollectionTable(name = "user_phones", joinColumns = @JoinColumn(name = "user_id"))
@Column(name = "phone")
private List<String> phoneNumbers;
```

---

**Q96. What happens to Kafka consumer offsets when a `@KafkaListener` throws an exception?**

With `AckMode.MANUAL`, the offset is only committed if `Acknowledgment.acknowledge()` is called. If an exception is thrown before ack, the offset is not committed and the message will be redelivered. With auto-commit mode, the offset is committed on the next poll regardless of exception. Spring Kafka's `DefaultErrorHandler` can be configured with retry policies and a `DeadLetterPublishingRecoverer` to route failed messages to a DLT after exhausted retries.

---

**Q97. How do you implement distributed locking with Redis in Spring Boot?**

Use Redisson's `RLock`:

```java
RLock lock = redissonClient.getLock("lock:order:" + orderId);
boolean acquired = lock.tryLock(5, 10, TimeUnit.SECONDS); // Wait 5s, hold 10s
try {
    if (acquired) {
        // critical section
    }
} finally {
    if (acquired && lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

This prevents duplicate processing in multi-instance deployments.

---

**Q98. What are the tradeoffs between JWT and opaque tokens in Spring Security OAuth2?**

| | JWT (Self-contained) | Opaque Token |
|-|----------------------|-------------|
| Validation | Local (verify signature) | Remote (introspect with auth server) |
| Revocation | Hard (stateless) | Immediate (server controls) |
| Network calls | None per request | One per request |
| Claims | In token (visible) | Hidden from resource server |
| Size | Larger | Small |
| Best for | High-traffic, stateless APIs | When revocation is critical |

---

**Q99. How would you design a Spring Boot application for production on Kubernetes using Boot 2.7 features?**

1. **Probes** — `liveness` and `readiness` probe endpoints via Actuator health groups
2. **Graceful shutdown** — `server.shutdown=graceful` + `spring.lifecycle.timeout-per-shutdown-phase=20s`
3. **Externalized config** — Environment variables from K8s ConfigMaps/Secrets; no hardcoded secrets
4. **Metrics** — Micrometer + Prometheus registry for scraping via `ServiceMonitor`
5. **Structured logging** — JSON logs for aggregation into ELK/Loki
6. **Profiles** — `SPRING_PROFILES_ACTIVE=prod` from the deployment manifest
7. **Health groups** — Separate indicators for liveness vs readiness (DB failing = not ready, not dead)
8. **Low footprint** — GraalVM native image or JVM container awareness (`-XX:MaxRAMPercentage=75`)
9. **Distributed tracing** — Spring Cloud Sleuth + Zipkin/Jaeger for cross-service tracing
10. **Startup events** — `CommandLineRunner` for any pre-flight checks

---

**Q100. How does `@ConditionalOnSingleCandidate` differ from `@ConditionalOnBean`?**

`@ConditionalOnBean` is true if at least one bean of the specified type exists. `@ConditionalOnSingleCandidate` is true only if exactly one bean of the specified type exists, OR if there are multiple but one is marked `@Primary`. This is used for auto-configurations that should only apply when there's no ambiguity — for example, `TransactionAutoConfiguration` uses it to ensure only one `PlatformTransactionManager` is present before auto-configuring `@EnableTransactionManagement`.

---

[← Logging & DevTools](./12_14_Caching_Messaging_Logging.md) &nbsp;|&nbsp; [↑ Back to Index](./README.md)
