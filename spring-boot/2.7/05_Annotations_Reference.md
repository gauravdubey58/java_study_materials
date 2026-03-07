# 05 — Complete Annotations Reference

[← Back to Index](./README.md)

> Every annotation used in Spring Boot 2.7 — with definition, use case, and code snippet.

---

## Table of Contents

- [A. Core / Stereotype](#a-core--stereotype-annotations)
- [B. Dependency Injection](#b-dependency-injection-annotations)
- [C. Request Mapping](#c-request-mapping-annotations)
- [D. Request / Response Handling](#d-request--response-handling-annotations)
- [E. Bean Validation (JSR-380)](#e-bean-validation-jsr-380-annotations)
- [F. JPA / Data](#f-jpa--data-annotations)
- [G. Transaction](#g-transaction-annotations)
- [H. Scheduling](#h-scheduling-annotations)
- [I. Caching](#i-caching-annotations)
- [J. Security](#j-security-annotations)
- [K. AOP](#k-aop-annotations)
- [L. Events & Async](#l-events--async-annotations)
- [M. Profile & Conditional](#m-profile--conditional-annotations)
- [N. Actuator](#n-actuator-annotations)
- [O. Testing](#o-testing-annotations)
- [P. Miscellaneous](#p-miscellaneous-annotations)

---

## A. Core / Stereotype Annotations

### `@SpringBootApplication`
**Definition:** Meta-annotation combining `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`.  
**Use Case:** Applied to the main class of every Spring Boot application.

```java
@SpringBootApplication(
    scanBasePackages = {"com.example.app", "com.example.shared"},
    exclude = {DataSourceAutoConfiguration.class}
)
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

---

### `@Configuration`
**Definition:** Marks a class as a source of `@Bean` definitions; processed by Spring as a configuration class.  
**Use Case:** Java-based configuration; replacing XML `<beans>` config.

```java
@Configuration
public class AppConfig {
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }
}
```

---

### `@Bean`
**Definition:** Method-level annotation; the return value is registered as a Spring bean.  
**Use Case:** Integrating third-party libraries, or complex object construction.

```java
@Bean(name = "myCache", initMethod = "start", destroyMethod = "stop")
@Scope("singleton")
public CacheManager cacheManager() {
    return new ConcurrentMapCacheManager("users", "products");
}
```

---

### `@Component`
**Definition:** Generic stereotype annotation. Spring detects it during component scan and registers the class as a bean.  
**Use Case:** Utility classes, helpers, validators that don't fit `@Service` or `@Repository`.

```java
@Component
public class CurrencyConverter {
    public BigDecimal convert(BigDecimal amount, String from, String to) { ... }
}
```

---

### `@Service`
**Definition:** Specialization of `@Component` for the service layer.  
**Use Case:** Business logic classes.

```java
@Service
public class UserService {
    public User createUser(CreateUserRequest request) { ... }
}
```

---

### `@Repository`
**Definition:** Specialization of `@Component` for the data access layer. Also enables **automatic persistence exception translation** (converts JPA/Hibernate/JDBC exceptions to Spring's `DataAccessException`).  
**Use Case:** Custom DAO implementations.

```java
@Repository
public class UserDaoImpl implements UserDao {
    public User findByUsername(String username) { ... }
}
```

---

### `@Controller`
**Definition:** Specialization of `@Component` for MVC web controllers that return view names.  
**Use Case:** Server-side rendered web apps (Thymeleaf, JSP).

```java
@Controller
public class HomeController {
    @GetMapping("/home")
    public String home(Model model) {
        model.addAttribute("greeting", "Welcome!");
        return "home"; // resolves to templates/home.html
    }
}
```

---

### `@RestController`
**Definition:** `@Controller` + `@ResponseBody`. Every handler method serializes its return value directly to the response body (JSON/XML).  
**Use Case:** RESTful APIs.

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {
    @GetMapping("/{id}")
    public ResponseEntity<UserDto> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }
}
```

---

## B. Dependency Injection Annotations

### `@Autowired`
**Definition:** Marks a constructor, field, or setter for automatic dependency injection.  
**Use Case:** Wiring beans together (prefer constructor injection).

```java
@Service
public class OrderService {
    private final OrderRepository repo;

    @Autowired  // Optional on single constructor (Spring 4.3+)
    public OrderService(OrderRepository repo) {
        this.repo = repo;
    }
}
```

---

### `@Qualifier`
**Definition:** Specifies which bean to inject when multiple candidates of the same type exist.

```java
@Autowired
@Qualifier("redisCache")
private CacheManager cacheManager;

// Bean definition side
@Bean("redisCache")
public CacheManager redisCache() { ... }

@Bean("caffeinCache")
public CacheManager caffeinCache() { ... }
```

---

### `@Primary`
**Definition:** Marks a bean as the preferred candidate when multiple beans of the same type exist.

```java
@Bean
@Primary
public DataSource primaryDataSource() { ... }

@Bean
public DataSource secondaryDataSource() { ... }
```

---

### `@Value`
**Definition:** Injects a value from properties, environment variables, or SpEL expressions.

```java
@Value("${app.name:DefaultApp}")            // Property with default
private String appName;

@Value("${server.port}")                    // Required property
private int port;

@Value("#{systemProperties['java.home']}")  // SpEL — system property
private String javaHome;

@Value("#{2 * T(Math).PI}")                 // SpEL — expression
private double twoPi;

@Value("${allowed.origins:}".split(","))    // List from property
private String[] allowedOrigins;
```

---

### `@Resource`
**Definition:** JSR-250 annotation. Injects by **name** first, then by type. Part of Jakarta EE.  
**Use Case:** When you need to inject a specific bean by its name.

```java
@Resource(name = "primaryDataSource")
private DataSource dataSource;
```

---

### `@Inject`
**Definition:** JSR-330 standard equivalent of `@Autowired`. Requires `jakarta.inject` dependency.  
**Use Case:** Portable code across DI frameworks.

---

## C. Request Mapping Annotations

### `@RequestMapping`
**Definition:** Maps HTTP requests to handler methods. Supports HTTP method, path, headers, params, content type.

```java
@RequestMapping(
    value    = "/api/users",
    method   = RequestMethod.GET,
    produces = MediaType.APPLICATION_JSON_VALUE,
    headers  = "X-API-Version=2",
    params   = "active=true"
)
public List<User> getActiveUsers() { ... }
```

---

### `@GetMapping` / `@PostMapping` / `@PutMapping` / `@PatchMapping` / `@DeleteMapping`
**Definition:** HTTP method-specific shortcuts for `@RequestMapping`.

```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) { ... }

@PostMapping("/users")
public ResponseEntity<User> createUser(@Valid @RequestBody CreateUserRequest req) { ... }

@PutMapping("/users/{id}")
public User replaceUser(@PathVariable Long id, @Valid @RequestBody UpdateUserRequest req) { ... }

@PatchMapping("/users/{id}")
public User patchUser(@PathVariable Long id, @RequestBody Map<String, Object> fields) { ... }

@DeleteMapping("/users/{id}")
@ResponseStatus(HttpStatus.NO_CONTENT)
public void deleteUser(@PathVariable Long id) { ... }
```

---

## D. Request / Response Handling Annotations

### `@RequestBody`
**Definition:** Deserializes the HTTP request body (JSON/XML) into a Java object using `HttpMessageConverter`.

```java
@PostMapping("/orders")
public Order placeOrder(@RequestBody @Valid PlaceOrderRequest request) { ... }
```

---

### `@PathVariable`
**Definition:** Extracts values from URI template variables.

```java
@GetMapping("/shops/{shopId}/products/{productId}")
public Product getProduct(
    @PathVariable Long shopId,
    @PathVariable("productId") Long pid
) { ... }
```

---

### `@RequestParam`
**Definition:** Extracts query string or form parameters from the URL.

```java
@GetMapping("/users")
public Page<User> search(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size,
    @RequestParam(required = false) String query,
    @RequestParam(name = "sort_by", defaultValue = "createdAt") String sortBy
) { ... }
```

---

### `@RequestHeader`
**Definition:** Extracts an HTTP header value.

```java
@GetMapping("/secure")
public String secure(
    @RequestHeader("Authorization") String authHeader,
    @RequestHeader(value = "X-Request-ID", required = false) String requestId
) { ... }
```

---

### `@CookieValue`
**Definition:** Extracts a cookie value.

```java
@GetMapping("/profile")
public String profile(@CookieValue(name = "SESSION", required = false) String sessionId) { ... }
```

---

### `@ResponseStatus`
**Definition:** Sets the HTTP status code for a response or marks an exception class with a default status.

```java
@PostMapping("/users")
@ResponseStatus(HttpStatus.CREATED)
public User createUser(@RequestBody User user) { ... }

// On exception class
@ResponseStatus(value = HttpStatus.NOT_FOUND, reason = "User not found")
public class UserNotFoundException extends RuntimeException { }
```

---

### `@ExceptionHandler`
**Definition:** Handles exceptions thrown by handler methods. Scoped to the controller unless used in `@ControllerAdvice`.

```java
@ExceptionHandler({UserNotFoundException.class, ProductNotFoundException.class})
public ResponseEntity<ErrorResponse> handleNotFound(RuntimeException ex) {
    return ResponseEntity.status(HttpStatus.NOT_FOUND)
        .body(new ErrorResponse(ex.getMessage()));
}
```

---

### `@ControllerAdvice` / `@RestControllerAdvice`
**Definition:** Global exception handling, model attributes, and data binding across all controllers.  
`@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Map<String, String> handleValidationErrors(MethodArgumentNotValidException ex) {
        return ex.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                FieldError::getDefaultMessage
            ));
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleAll(Exception ex) {
        return new ErrorResponse("INTERNAL_ERROR", ex.getMessage());
    }
}
```

---

### `@CrossOrigin`
**Definition:** Enables CORS for a controller or method.

```java
@CrossOrigin(
    origins = {"http://localhost:3000", "https://myapp.com"},
    methods = {RequestMethod.GET, RequestMethod.POST},
    allowedHeaders = "*",
    maxAge = 3600
)
@RestController
public class ApiController { ... }
```

---

## E. Bean Validation (JSR-380) Annotations

### `@Valid` vs `@Validated`
- `@Valid` — JSR-303 standard; triggers validation; no group support
- `@Validated` — Spring variant; supports validation groups

```java
@PostMapping("/users")
public User create(@Valid @RequestBody CreateUserRequest request) { ... }
```

### All Validation Constraints

```java
public class UserRequest {
    @NotNull                                    // Cannot be null
    @NotEmpty                                   // Cannot be null or empty
    @NotBlank                                   // Cannot be null, empty, or whitespace
    @Size(min = 2, max = 50)                    // String/Collection length
    private String name;

    @Email(message = "Must be a valid email")   // Email format
    @Pattern(regexp = "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,6}$")
    private String email;

    @Min(18) @Max(120)                          // Numeric min/max
    @Positive                                   // Must be > 0
    @PositiveOrZero                             // Must be >= 0
    @Negative                                   // Must be < 0
    @NegativeOrZero                             // Must be <= 0
    private int age;

    @Digits(integer = 10, fraction = 2)         // Numeric digits
    @DecimalMin("0.00") @DecimalMax("9999.99")
    private BigDecimal price;

    @Future                                     // Must be in the future
    @FutureOrPresent
    private LocalDate joiningDate;

    @Past                                       // Must be in the past
    @PastOrPresent
    private LocalDate birthDate;

    @AssertTrue                                 // Must be true
    private boolean termsAccepted;

    @AssertFalse                                // Must be false
    private boolean deleted;

    @NotEmpty
    private List<@NotBlank String> roles;       // Validate collection elements

    @Valid                                      // Cascade validation to nested object
    private Address address;
}
```

---

## F. JPA / Data Annotations

### Entity Mapping

```java
@Entity
@Table(name = "users",
    indexes = {
        @Index(name = "idx_email", columnList = "email"),
        @Index(name = "idx_created_at", columnList = "created_at")
    },
    uniqueConstraints = @UniqueConstraint(columnNames = {"email"})
)
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "full_name", nullable = false, length = 100)
    private String name;

    @Column(unique = true, nullable = false)
    private String email;

    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @Transient                              // Not persisted
    private String fullDisplayName;

    @Lob                                    // Large Object (CLOB/BLOB)
    private String biography;

    @Enumerated(EnumType.STRING)            // Store as string, not ordinal
    private Status status;

    @Embedded                              // Inline another class's fields
    private Address address;

    @Version                               // Optimistic locking
    private Long version;
}
```

### GenerationType Strategies

| Strategy | Behaviour | Best For |
|----------|-----------|---------|
| `IDENTITY` | DB auto-increment | MySQL, PostgreSQL SERIAL |
| `SEQUENCE` | DB sequence object | PostgreSQL, Oracle |
| `TABLE` | Simulates sequence with a table | Portability (avoid in prod) |
| `AUTO` | JPA picks (usually SEQUENCE) | Prototyping |
| `UUID` | Generates UUID (Hibernate 6) | Distributed systems |

### Relationships

```java
// One-to-One
@OneToOne(cascade = CascadeType.ALL, fetch = FetchType.LAZY, optional = false)
@JoinColumn(name = "profile_id")
private UserProfile profile;

// One-to-Many (parent side)
@OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
@OrderBy("createdAt DESC")
private List<Order> orders = new ArrayList<>();

// Many-to-One (child side)
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "user_id", nullable = false)
private User user;

// Many-to-Many
@ManyToMany
@JoinTable(
    name = "user_roles",
    joinColumns        = @JoinColumn(name = "user_id"),
    inverseJoinColumns = @JoinColumn(name = "role_id")
)
private Set<Role> roles = new HashSet<>();
```

### Embeddable

```java
@Embeddable
public class Address {
    @Column(nullable = false)
    private String street;
    private String city;
    private String country;
    @Column(name = "zip_code")
    private String zipCode;
}
```

### JPA Auditing

```java
@Entity
@EntityListeners(AuditingEntityListener.class)
public class Article {

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @CreatedBy
    @Column(updatable = false)
    private String createdBy;

    @LastModifiedBy
    private String updatedBy;
}

// Enable in main class or config
@SpringBootApplication
@EnableJpaAuditing(auditorAwareRef = "currentUserAuditor")
public class MyApp { }

// Provide current user
@Component("currentUserAuditor")
public class CurrentUserAuditor implements AuditorAware<String> {
    @Override
    public Optional<String> getCurrentAuditor() {
        return Optional.ofNullable(SecurityContextHolder.getContext())
            .map(ctx -> ctx.getAuthentication())
            .filter(auth -> auth != null && auth.isAuthenticated())
            .map(Authentication::getName);
    }
}
```

### Lifecycle Callbacks

```java
@PrePersist   public void prePersist()  { this.createdAt = LocalDateTime.now(); }
@PreUpdate    public void preUpdate()   { this.updatedAt = LocalDateTime.now(); }
@PostLoad     public void postLoad()    { /* after entity loaded from DB */ }
@PostPersist  public void postPersist() { /* after INSERT */ }
@PostUpdate   public void postUpdate()  { /* after UPDATE */ }
@PreRemove    public void preRemove()   { /* before DELETE */ }
@PostRemove   public void postRemove()  { /* after DELETE */ }
```

---

## G. Transaction Annotations

### `@Transactional`

```java
@Service
public class TransferService {

    @Transactional(
        propagation  = Propagation.REQUIRED,        // default
        isolation    = Isolation.READ_COMMITTED,
        readOnly     = false,
        rollbackFor  = {Exception.class},           // roll back on checked too
        noRollbackFor = {InsufficientFundsException.class},
        timeout      = 30                           // seconds
    )
    public void transfer(Long fromId, Long toId, BigDecimal amount) { ... }

    @Transactional(readOnly = true)  // Optimization: no flush, skip dirty checking
    public List<Account> getAll() { ... }
}
```

### `@EnableTransactionManagement`

```java
@Configuration
@EnableTransactionManagement
public class TxConfig { }
// (auto-enabled by @SpringBootApplication via auto-config)
```

---

## H. Scheduling Annotations

### `@Scheduled`

```java
@EnableScheduling  // Required — on @Configuration or main class
@SpringBootApplication
public class MyApp { }

@Component
public class Jobs {

    @Scheduled(fixedRate = 10_000)                   // Every 10s from start
    public void pollExternalApi() { ... }

    @Scheduled(fixedDelay = 5_000)                   // 5s after last execution ends
    public void processQueue() { ... }

    @Scheduled(initialDelay = 1_000, fixedRate = 60_000)
    public void withDelay() { ... }

    // Cron: second minute hour day-of-month month day-of-week
    @Scheduled(cron = "0 0 2 * * MON-FRI")           // 2 AM on weekdays
    public void nightly() { ... }

    @Scheduled(cron = "${app.jobs.cleanup.cron:0 0 3 * * *}")  // From property
    public void cleanup() { ... }
}
```

**Cron Quick Reference:**

| Expression | Meaning |
|------------|---------|
| `0 * * * * *` | Every minute |
| `0 0 * * * *` | Every hour |
| `0 0 12 * * *` | Every day at noon |
| `0 0 0 * * MON` | Every Monday at midnight |
| `0 0 0 1 * *` | First day of every month |
| `0 0 0 L * *` | Last day of every month |

---

## I. Caching Annotations

### Setup

```java
@SpringBootApplication
@EnableCaching
public class MyApp { }
```

### `@Cacheable` — Read through cache

```java
@Cacheable(
    value     = "products",
    key       = "#id",
    condition = "#id > 0",           // Don't cache if condition false
    unless    = "#result == null"    // Don't cache null results
)
public Product findById(Long id) { ... }
```

### `@CachePut` — Always execute + update cache

```java
@CachePut(value = "products", key = "#product.id")
public Product update(Product product) { ... }
```

### `@CacheEvict` — Remove from cache

```java
@CacheEvict(value = "products", key = "#id")
public void delete(Long id) { ... }

@CacheEvict(value = "products", allEntries = true)  // Clear entire cache
@Scheduled(cron = "0 0 3 * * *")
public void clearProductCache() { }
```

### `@Caching` — Multiple cache operations

```java
@Caching(
    cacheable = { @Cacheable("products") },
    evict     = { @CacheEvict("productList"), @CacheEvict("categories") }
)
public Product getAndRefresh(Long id) { ... }
```

### `@CacheConfig` — Class-level defaults

```java
@CacheConfig(cacheNames = "products", cacheManager = "redisCacheManager")
@Service
public class ProductService {
    @Cacheable   // Uses class-level cache name
    public Product findById(Long id) { ... }
}
```

---

## J. Security Annotations

### `@EnableWebSecurity`

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .antMatchers("/public/**").permitAll()
                .antMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
        return http.build();
    }
}
```

### Method Security (Boot 2.7 / Security 5.7+)

```java
// Replaces @EnableGlobalMethodSecurity
@EnableMethodSecurity(
    prePostEnabled  = true,   // @PreAuthorize, @PostAuthorize, @PreFilter, @PostFilter
    securedEnabled  = true,   // @Secured
    jsr250Enabled   = true    // @RolesAllowed, @PermitAll, @DenyAll
)
```

### Security Annotations on Methods

```java
// Before method executes
@PreAuthorize("hasRole('ADMIN')")
public void adminAction() { ... }

@PreAuthorize("hasRole('USER') and #userId == authentication.principal.id")
public User getProfile(Long userId) { ... }

// After method executes (has access to returnObject)
@PostAuthorize("returnObject.owner == authentication.name")
public Document getDocument(Long id) { ... }

// Filter collection BEFORE method runs
@PreFilter("filterObject.owner == authentication.name")
public void deleteFiles(List<File> files) { ... }

// Filter collection AFTER method runs
@PostFilter("filterObject.confidential == false")
public List<Report> getReports() { ... }

// Simple role check
@Secured({"ROLE_ADMIN", "ROLE_SUPERUSER"})
public void sensitiveAction() { ... }

// JSR-250
@RolesAllowed("MANAGER")
public void managerOnlyAction() { ... }
```

---

## K. AOP Annotations

```java
@Aspect
@Component
@Order(1)   // Multiple aspects — lower = higher priority
public class LoggingAspect {

    // Reusable pointcut definition
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() {}

    @Pointcut("@annotation(com.example.annotation.Auditable)")
    public void auditableMethods() {}

    @Before("serviceLayer()")
    public void logEntry(JoinPoint jp) {
        log.info(">> {} called with args: {}", jp.getSignature(), jp.getArgs());
    }

    @After("serviceLayer()")
    public void logExit(JoinPoint jp) {
        log.info("<< {} finished", jp.getSignature().getName());
    }

    @AfterReturning(pointcut = "serviceLayer()", returning = "result")
    public void logResult(JoinPoint jp, Object result) {
        log.info("Result: {}", result);
    }

    @AfterThrowing(pointcut = "serviceLayer()", throwing = "ex")
    public void logError(JoinPoint jp, Throwable ex) {
        log.error("Exception in {}: {}", jp.getSignature(), ex.getMessage());
    }

    @Around("auditableMethods()")
    public Object audit(ProceedingJoinPoint pjp) throws Throwable {
        log.info("AUDIT: {} started", pjp.getSignature().getName());
        Object result = pjp.proceed();   // Execute actual method
        log.info("AUDIT: {} completed", pjp.getSignature().getName());
        return result;
    }
}
```

---

## L. Events & Async Annotations

### `@EventListener`

```java
// Define event
public class UserRegisteredEvent {
    private final User user;
    public UserRegisteredEvent(User user) { this.user = user; }
    public User getUser() { return user; }
}

// Publish event
@Service
public class UserService {
    @Autowired
    private ApplicationEventPublisher publisher;

    public User register(CreateUserRequest req) {
        User user = userRepository.save(toEntity(req));
        publisher.publishEvent(new UserRegisteredEvent(user));
        return user;
    }
}

// Listen to event
@Component
public class UserEventListener {

    @EventListener
    public void onRegistered(UserRegisteredEvent event) {
        emailService.sendWelcomeEmail(event.getUser());
    }

    // Only fires when Spring condition is met
    @EventListener(condition = "#event.user.premium == true")
    public void onPremiumRegistered(UserRegisteredEvent event) {
        giftService.sendWelcomeGift(event.getUser());
    }
}
```

### `@TransactionalEventListener`

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onOrderPlaced(OrderPlacedEvent event) {
    // Only runs AFTER the transaction that published this event commits
    emailService.sendOrderConfirmation(event.getOrder());
}
```

### `@Async`

```java
@EnableAsync   // Required on a @Configuration class
@SpringBootApplication
public class MyApp { }

@Service
public class NotificationService {

    @Async                              // Runs in Spring's task executor thread pool
    public void sendEmail(String to, String body) {
        // Non-blocking; caller returns immediately
        emailClient.send(to, body);
    }

    @Async("customExecutor")           // Named executor bean
    public CompletableFuture<String> fetchData(Long id) {
        String data = externalApi.fetch(id);
        return CompletableFuture.completedFuture(data);
    }
}
```

---

## M. Profile & Conditional Annotations

### `@Profile`

```java
@Configuration
@Profile("dev")
public class DevDatabaseConfig {
    @Bean public DataSource h2() { ... }
}

@Configuration
@Profile("!dev")            // All profiles EXCEPT dev
public class ProdDatabaseConfig {
    @Bean public DataSource mysql() { ... }
}

@Configuration
@Profile({"staging", "prod"})   // Multiple profiles
public class CloudConfig { ... }
```

### `@ConditionalOnProperty` (most common)

```java
@Bean
@ConditionalOnProperty(
    name         = "feature.email.enabled",
    havingValue  = "true",
    matchIfMissing = false   // Default: don't create if property missing
)
public EmailService emailService() { ... }
```

---

## N. Actuator Annotations

### Custom Endpoints

```java
@Component
@Endpoint(id = "app-info")   // Accessible at /actuator/app-info
public class AppInfoEndpoint {

    @ReadOperation               // GET /actuator/app-info
    public Map<String, Object> info() {
        return Map.of(
            "environment", System.getenv("APP_ENV"),
            "uptime", ManagementFactory.getRuntimeMXBean().getUptime()
        );
    }

    @WriteOperation              // POST /actuator/app-info with body
    public void updateConfig(@Selector String key, String value) {
        configService.set(key, value);
    }

    @DeleteOperation             // DELETE /actuator/app-info/{key}
    public void clearConfig(@Selector String key) {
        configService.remove(key);
    }
}

// Expose only via HTTP
@WebEndpoint(id = "http-only-endpoint")
public class HttpOnlyEndpoint { ... }

// Expose only via JMX
@JmxEndpoint(id = "jmx-endpoint")
public class JmxEndpoint { ... }
```

---

## O. Testing Annotations

| Annotation | Purpose | Context Loaded |
|------------|---------|---------------|
| `@SpringBootTest` | Full integration test | Full context |
| `@WebMvcTest` | Controller layer test | Web layer only |
| `@DataJpaTest` | Repository layer test | JPA + embedded DB |
| `@DataMongoTest` | MongoDB repository test | Mongo context |
| `@WebFluxTest` | Reactive controller test | WebFlux layer |
| `@JsonTest` | JSON serialization test | Jackson context |
| `@RestClientTest` | REST client test | RestTemplate context |

```java
// Full integration test
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class FullIntegrationTest {
    @Autowired TestRestTemplate restTemplate;
    @MockBean  ExternalPaymentService paymentService;
}

// Slice: web layer only
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean  UserService userService;    // Mock the service layer

    @Test
    void getUser_returnsOk() throws Exception {
        given(userService.findById(1L)).willReturn(new UserDto(1L, "Alice"));
        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("Alice"));
    }
}

// Slice: JPA layer only
@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE)  // Use real DB (not H2)
class UserRepositoryTest {
    @Autowired UserRepository repo;

    @Test
    void findByEmail_returnsUser() {
        repo.save(new User("Alice", "alice@test.com"));
        assertThat(repo.findByEmail("alice@test.com")).isPresent();
    }
}
```

---

## P. Miscellaneous Annotations

| Annotation | Description |
|------------|-------------|
| `@Lazy` | Defer bean creation until first use |
| `@DependsOn("beanA")` | Force beanA to initialize first |
| `@Order(1)` | Control bean/advice initialization order |
| `@Scope("prototype")` | Override bean scope |
| `@PropertySource("classpath:custom.properties")` | Load additional properties file |
| `@Import(OtherConfig.class)` | Import another `@Configuration` class |
| `@ImportResource("classpath:legacy.xml")` | Import XML configuration |
| `@ComponentScan(basePackages = "com.example")` | Configure component scanning |
| `@EnableConfigurationProperties(MyProps.class)` | Bind `@ConfigurationProperties` POJO |
| `@JsonIgnore` | Exclude field from JSON serialization |
| `@JsonProperty("user_name")` | Rename field in JSON |
| `@JsonAlias({"fname", "firstName"})` | Accept multiple JSON names during deserialization |

---

[← Starters & Config](./04_Starters_and_Configuration.md) &nbsp;|&nbsp; [Next → Spring Data JPA](./06_Spring_Data_JPA.md)
