# ☕ Java 8 (March 2014) — The Revolution

> Java 8 is the most transformative release in Java's history. If you only master one version deeply, make it this one — the foundations introduced here underpin everything in modern Java.

📘 [← Back to Index](./index.md)

---

## 1. Lambda Expressions

### What It Is
A lambda is an anonymous function — a block of code passed as data. It implements a **functional interface** (an interface with exactly one abstract method).

### Syntax
```java
// Before Java 8 — Anonymous class
Runnable r = new Runnable() {
    @Override
    public void run() {
        System.out.println("Running");
    }
};

// Java 8 — Lambda
Runnable r = () -> System.out.println("Running");

// With parameters
Comparator<String> byLength = (s1, s2) -> s1.length() - s2.length();

// With block body
Comparator<String> byLength = (s1, s2) -> {
    int diff = s1.length() - s2.length();
    return diff;
};
```

### Built-in Functional Interfaces (java.util.function)

| Interface | Signature | Use Case |
|-----------|-----------|---------|
| `Predicate<T>` | `T → boolean` | Filtering |
| `Function<T,R>` | `T → R` | Transforming |
| `Consumer<T>` | `T → void` | Consuming/side effects |
| `Supplier<T>` | `() → T` | Lazy creation |
| `BiFunction<T,U,R>` | `T, U → R` | Two inputs, one output |
| `UnaryOperator<T>` | `T → T` | Transform same type |
| `BinaryOperator<T>` | `T, T → T` | Combine two of same type |

### Real-World Example: Payment Processing Pipeline

```java
public class PaymentProcessor {

    // Predicate — is payment eligible for processing?
    private final Predicate<Payment> isEligible = payment ->
        payment.getAmount().compareTo(BigDecimal.ZERO) > 0
        && payment.getStatus() == PaymentStatus.PENDING
        && !payment.isExpired();

    // Function — enrich payment with fee
    private final Function<Payment, Payment> addProcessingFee = payment -> {
        BigDecimal fee = payment.getAmount().multiply(new BigDecimal("0.025"));
        return payment.withFee(fee);
    };

    // Consumer — audit log
    private final Consumer<Payment> auditLog = payment ->
        auditService.record(payment.getId(), "PROCESSED", LocalDateTime.now());

    // Supplier — generate transaction ID
    private final Supplier<String> transactionIdGenerator =
        () -> "TXN-" + UUID.randomUUID().toString().toUpperCase();

    public void process(List<Payment> payments) {
        payments.stream()
            .filter(isEligible)                          // Predicate
            .map(addProcessingFee)                       // Function
            .peek(p -> p.setTransactionId(transactionIdGenerator.get())) // Supplier + Consumer
            .forEach(auditLog.andThen(this::sendToGateway)); // Consumer chaining
    }
}
```

### Corner Cases and Gotchas

**1. Variable capture — must be effectively final**
```java
int discount = 10; // effectively final — OK
// discount = 20; // Uncommenting this line would break the lambda below

List<Integer> prices = Arrays.asList(100, 200, 300);
prices.forEach(p -> System.out.println(p - discount)); // OK

// WRONG — modifying a captured variable
int[] counter = {0}; // Workaround using array (hack — avoid in production)
prices.forEach(p -> counter[0]++); // Compiles but is not thread-safe
```

**2. `this` inside a lambda refers to the enclosing class, not the lambda**
```java
public class Service {
    private String name = "Service";

    public void demo() {
        Runnable r = () -> System.out.println(this.name); // 'this' = Service instance
        Runnable r2 = new Runnable() {
            public void run() {
                // Here 'this' = the anonymous Runnable instance, NOT Service
                // System.out.println(this.name); // COMPILE ERROR
                System.out.println(Service.this.name); // Use outer class reference
            }
        };
    }
}
```

**3. Checked exceptions in lambdas**
```java
// PROBLEM — lambdas can't throw checked exceptions directly
List<String> urls = List.of("http://example.com", "http://bad url");

// This WON'T COMPILE — new URL() throws MalformedURLException (checked)
// urls.stream().map(url -> new URL(url)).collect(toList());

// SOLUTION 1 — wrap in a helper
@FunctionalInterface
public interface ThrowingFunction<T, R> {
    R apply(T t) throws Exception;
}

public static <T, R> Function<T, R> wrap(ThrowingFunction<T, R> f) {
    return t -> {
        try { return f.apply(t); }
        catch (Exception e) { throw new RuntimeException(e); }
    };
}

urls.stream().map(wrap(url -> new URL(url))).collect(toList()); // Now compiles
```

---

## 2. Method References

Shorthand for lambdas that simply call an existing method.

```java
// Four types of method references
// 1. Static method reference
Function<String, Integer> parse = Integer::parseInt;

// 2. Instance method reference (on a specific instance)
String prefix = "Hello";
Predicate<String> startsWith = prefix::startsWith;

// 3. Instance method reference (on an arbitrary instance of a type)
Function<String, String> toUpper = String::toUpperCase;
// Equivalent to: s -> s.toUpperCase()

// 4. Constructor reference
Supplier<ArrayList<String>> listFactory = ArrayList::new;
Function<String, StringBuilder> sbFactory = StringBuilder::new;
```

### Real-World: Repository Pattern

```java
public class CustomerService {
    private final CustomerRepository repo;

    public List<String> getActiveCustomerEmails() {
        return repo.findAll().stream()
            .filter(Customer::isActive)              // instance method ref on arbitrary instance
            .map(Customer::getEmail)                 // instance method ref
            .filter(Objects::nonNull)                // static method ref
            .map(String::toLowerCase)                // instance method ref
            .sorted(String::compareTo)               // instance method ref as Comparator
            .collect(Collectors.toList());
    }

    public List<Customer> createCustomersFromNames(List<String> names) {
        return names.stream()
            .map(Customer::new)   // Constructor reference — calls Customer(String name)
            .collect(Collectors.toList());
    }
}
```

---

## 3. Streams API

The Streams API enables declarative, pipeline-based data processing — the most important Java 8 feature for day-to-day development.

### Stream Pipeline Structure

```
Source → Intermediate Operations (lazy) → Terminal Operation (triggers execution)
```

### Creating Streams

```java
// From collection
Stream<String> s1 = list.stream();
Stream<String> s2 = list.parallelStream();

// From array
Stream<String> s3 = Arrays.stream(new String[]{"a","b","c"});

// From values
Stream<String> s4 = Stream.of("a", "b", "c");

// Infinite streams
Stream<Integer> naturals = Stream.iterate(0, n -> n + 1);
Stream<Double> randoms   = Stream.generate(Math::random);

// Range (IntStream)
IntStream range = IntStream.rangeClosed(1, 100);
```

### Key Intermediate Operations

```java
List<Order> orders = orderRepository.findAll();

// filter — keep elements matching predicate
orders.stream().filter(o -> o.getAmount().compareTo(new BigDecimal("1000")) > 0)

// map — transform each element
orders.stream().map(Order::getCustomer)

// flatMap — flatten nested collections
orders.stream().flatMap(o -> o.getItems().stream()) // Stream<OrderItem>

// distinct — remove duplicates (uses equals/hashCode)
orders.stream().map(Order::getCustomerId).distinct()

// sorted — natural order or custom comparator
orders.stream().sorted(Comparator.comparing(Order::getCreatedAt).reversed())

// peek — side effects without consuming (good for debugging)
orders.stream().peek(o -> log.debug("Processing: {}", o.getId()))

// limit / skip — pagination
orders.stream().skip(20).limit(10) // Page 3, size 10
```

### Key Terminal Operations

```java
// collect — most powerful terminal op
List<Order> list = stream.collect(Collectors.toList());
Set<String> set  = stream.collect(Collectors.toSet());
Map<Long, Order> byId = stream.collect(Collectors.toMap(Order::getId, o -> o));

// Grouping — most important for real-world use
Map<OrderStatus, List<Order>> byStatus =
    orders.stream().collect(Collectors.groupingBy(Order::getStatus));

// Counting per group
Map<OrderStatus, Long> countByStatus =
    orders.stream().collect(Collectors.groupingBy(Order::getStatus, Collectors.counting()));

// Sum per group
Map<String, BigDecimal> revenueByCustomer = orders.stream().collect(
    Collectors.groupingBy(
        Order::getCustomerId,
        Collectors.reducing(BigDecimal.ZERO, Order::getAmount, BigDecimal::add)
    )
);

// reduce — fold to single value
Optional<BigDecimal> total = orders.stream()
    .map(Order::getAmount)
    .reduce(BigDecimal::add);

// count, sum, average, min, max
long count = orders.stream().filter(Order::isPaid).count();
OptionalDouble avg = orders.stream().mapToDouble(o -> o.getAmount().doubleValue()).average();
Optional<Order> largest = orders.stream().max(Comparator.comparing(Order::getAmount));

// anyMatch, allMatch, noneMatch — short-circuit
boolean hasOverdue = orders.stream().anyMatch(Order::isOverdue);
boolean allPaid    = orders.stream().allMatch(Order::isPaid);

// findFirst, findAny
Optional<Order> first = orders.stream().filter(Order::isUrgent).findFirst();
```

### Real-World: Invoice Report Generation

```java
public class InvoiceReportService {

    public InvoiceReport generateMonthlyReport(List<Invoice> invoices, YearMonth month) {
        List<Invoice> monthInvoices = invoices.stream()
            .filter(inv -> YearMonth.from(inv.getDate()).equals(month))
            .collect(Collectors.toList());

        // Revenue by category
        Map<String, BigDecimal> revenueByCategory = monthInvoices.stream()
            .collect(Collectors.groupingBy(
                Invoice::getCategory,
                Collectors.reducing(BigDecimal.ZERO, Invoice::getAmount, BigDecimal::add)
            ));

        // Top 5 customers by spend
        List<String> topCustomers = monthInvoices.stream()
            .collect(Collectors.groupingBy(Invoice::getCustomerId,
                Collectors.reducing(BigDecimal.ZERO, Invoice::getAmount, BigDecimal::add)))
            .entrySet().stream()
            .sorted(Map.Entry.<String, BigDecimal>comparingByValue().reversed())
            .limit(5)
            .map(Map.Entry::getKey)
            .collect(Collectors.toList());

        // Overdue invoices
        List<Invoice> overdue = monthInvoices.stream()
            .filter(inv -> inv.getDueDate().isBefore(LocalDate.now()) && !inv.isPaid())
            .sorted(Comparator.comparing(Invoice::getDueDate))
            .collect(Collectors.toList());

        // Total
        BigDecimal total = monthInvoices.stream()
            .map(Invoice::getAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        return new InvoiceReport(month, total, revenueByCategory, topCustomers, overdue);
    }
}
```

### Corner Cases and Gotchas

**1. Streams are single-use**
```java
Stream<String> stream = list.stream();
stream.forEach(System.out::println);
stream.forEach(System.out::println); // THROWS IllegalStateException — stream already consumed
```

**2. Lazy evaluation — intermediate ops don't execute until terminal op**
```java
Stream<Integer> stream = Stream.of(1, 2, 3, 4, 5)
    .filter(n -> { System.out.println("filtering " + n); return n % 2 == 0; })
    .map(n -> { System.out.println("mapping " + n); return n * 10; });

// Nothing printed yet! Terminal operation triggers everything:
stream.findFirst(); // Only prints "filtering 1", "filtering 2", "mapping 2" — then stops
```

**3. Parallel streams — not always faster, sometimes wrong**
```java
// DANGEROUS — non-thread-safe collection with parallel stream
List<Integer> results = new ArrayList<>();
IntStream.range(0, 1000).parallel().forEach(results::add); // Race condition! Missing elements

// CORRECT — use thread-safe collection
List<Integer> results = IntStream.range(0, 1000).parallel()
    .boxed().collect(Collectors.toList()); // Collect handles thread safety

// Parallel streams hurt performance for:
// - Small datasets (overhead > benefit)
// - I/O bound operations
// - Operations with shared mutable state
// Good for: CPU-intensive, large datasets, independent operations
```

**4. `toMap` collision — throws if duplicate keys**
```java
// THROWS if two orders have same customerId
Map<String, Order> byCustomer = orders.stream()
    .collect(Collectors.toMap(Order::getCustomerId, o -> o));

// FIX — provide merge function
Map<String, Order> byCustomer = orders.stream()
    .collect(Collectors.toMap(
        Order::getCustomerId,
        o -> o,
        (existing, replacement) -> existing // keep first
    ));
```

---

## 4. Optional\<T\>

A container object that may or may not contain a value — designed to eliminate NullPointerExceptions by forcing explicit handling of absent values.

### Creating Optional

```java
Optional<String> present = Optional.of("value");       // Throws NPE if null
Optional<String> nullable = Optional.ofNullable(null); // Returns empty if null
Optional<String> empty    = Optional.empty();
```

### Real-World: Customer Lookup Service

```java
// BEFORE Java 8 — null-check pyramid
public String getCustomerCity(Long customerId) {
    Customer customer = customerRepo.findById(customerId);
    if (customer != null) {
        Address address = customer.getAddress();
        if (address != null) {
            City city = address.getCity();
            if (city != null) {
                return city.getName();
            }
        }
    }
    return "Unknown";
}

// WITH Optional — flat and readable
public String getCustomerCity(Long customerId) {
    return customerRepo.findById(customerId)    // Returns Optional<Customer>
        .map(Customer::getAddress)              // Optional<Address>
        .map(Address::getCity)                  // Optional<City>
        .map(City::getName)                     // Optional<String>
        .orElse("Unknown");
}

// Real-world service method
public CustomerDTO findCustomer(Long id) {
    return customerRepo.findById(id)
        .map(customerMapper::toDTO)
        .orElseThrow(() -> new CustomerNotFoundException("Customer not found: " + id));
}

// Conditional action
public void sendWelcomeEmail(Long customerId) {
    customerRepo.findById(customerId)
        .filter(Customer::hasEmailVerified)
        .map(Customer::getEmail)
        .ifPresent(emailService::sendWelcome);
}
```

### Corner Cases and Gotchas

**1. Never use `Optional.get()` without `isPresent()` — defeats the purpose**
```java
// WRONG — throws NoSuchElementException if empty (worse than NPE — less obvious)
String value = optional.get();

// RIGHT — always use orElse / orElseGet / orElseThrow / ifPresent
String value = optional.orElse("default");
String value = optional.orElseGet(() -> computeDefault()); // lazy
String value = optional.orElseThrow(() -> new NotFoundException("Not found"));
```

**2. `orElse` vs `orElseGet` — important performance difference**
```java
// orElse ALWAYS evaluates the argument — even if Optional is present
User user = optional.orElse(createExpensiveDefaultUser()); // Always called!

// orElseGet is lazy — only called if Optional is empty
User user = optional.orElseGet(() -> createExpensiveDefaultUser()); // Only if empty
```

**3. Never use Optional as a field, constructor parameter, or collection element**
```java
// WRONG — Optional is not Serializable; adds overhead; not designed for this
public class Customer {
    private Optional<String> middleName; // BAD — use nullable String field
}

// WRONG — don't put Optional in collections
List<Optional<String>> list; // BAD — use filtering instead

// RIGHT — Optional is for method return types only
public Optional<String> findMiddleName() { ... }
```

**4. `flatMap` for nested Optionals**
```java
// If getAddress() itself returns Optional<Address>
Optional<Customer> customer = Optional.of(new Customer());

// map gives Optional<Optional<Address>> — WRONG
Optional<Optional<Address>> wrong = customer.map(Customer::getOptionalAddress);

// flatMap unwraps — gives Optional<Address> — CORRECT
Optional<Address> correct = customer.flatMap(Customer::getOptionalAddress);
```

---

## 5. Date and Time API (java.time)

The old `java.util.Date` and `Calendar` are notoriously broken — mutable, thread-unsafe, confusing month indexing (0-based), no timezone handling. Java 8's `java.time` fixes all of this.

### Core Classes

| Class | Description | Example |
|-------|-------------|---------|
| `LocalDate` | Date without time or timezone | `2024-03-15` |
| `LocalTime` | Time without date or timezone | `14:30:45` |
| `LocalDateTime` | Date + time, no timezone | `2024-03-15T14:30:45` |
| `ZonedDateTime` | Date + time + timezone | `2024-03-15T14:30:45+01:00[Europe/London]` |
| `Instant` | Machine timestamp (epoch) | `2024-03-15T13:30:45Z` |
| `Duration` | Time-based amount | `PT2H30M` (2 hours 30 min) |
| `Period` | Date-based amount | `P1Y2M3D` (1 year 2 months 3 days) |
| `ZoneId` | Timezone identifier | `Europe/London`, `UTC` |

### Real-World: Subscription and Billing System

```java
public class SubscriptionService {

    // Calculating subscription expiry
    public LocalDate calculateExpiry(LocalDate startDate, SubscriptionPlan plan) {
        return switch (plan) {
            case MONTHLY  -> startDate.plusMonths(1);
            case QUARTERLY -> startDate.plusMonths(3);
            case ANNUAL   -> startDate.plusYears(1);
        };
    }

    // Check if subscription is expired
    public boolean isExpired(Subscription sub) {
        return LocalDate.now().isAfter(sub.getExpiryDate());
    }

    // Days until renewal
    public long daysUntilRenewal(Subscription sub) {
        return ChronoUnit.DAYS.between(LocalDate.now(), sub.getExpiryDate());
    }

    // Schedule a meeting across timezones
    public String formatMeetingTime(LocalDateTime utcTime, String customerTimezone) {
        ZonedDateTime utc = utcTime.atZone(ZoneId.of("UTC"));
        ZonedDateTime customerTime = utc.withZoneSameInstant(ZoneId.of(customerTimezone));
        return customerTime.format(DateTimeFormatter.ofPattern("dd MMM yyyy HH:mm z"));
        // "15 Mar 2024 09:30 IST" for customer in India
    }

    // Business days between two dates
    public long businessDaysBetween(LocalDate start, LocalDate end) {
        return start.datesUntil(end)
            .filter(date -> date.getDayOfWeek() != DayOfWeek.SATURDAY
                         && date.getDayOfWeek() != DayOfWeek.SUNDAY)
            .count();
    }

    // Parse various date formats from external APIs
    public LocalDate parseExternalDate(String dateStr) {
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd");
        try {
            return LocalDate.parse(dateStr, formatter);
        } catch (DateTimeParseException e) {
            // Try alternative format
            DateTimeFormatter altFormatter = DateTimeFormatter.ofPattern("MM/dd/yyyy");
            return LocalDate.parse(dateStr, altFormatter);
        }
    }
}
```

### Corner Cases and Gotchas

**1. Month is 1-based (unlike old Calendar's 0-based)**
```java
// Old java.util.Calendar — CONFUSING 0-based months
Calendar cal = Calendar.getInstance();
cal.set(2024, 2, 15); // Month 2 = MARCH (not February) — confusing!

// java.time — intuitive 1-based
LocalDate date = LocalDate.of(2024, 3, 15); // March 15, 2024 — clear
LocalDate date2 = LocalDate.of(2024, Month.MARCH, 15); // Even clearer
```

**2. Immutability — all operations return new instances**
```java
LocalDate today = LocalDate.now();
today.plusDays(7); // BUG — result is discarded! today is unchanged
LocalDate nextWeek = today.plusDays(7); // CORRECT — assign the result
```

**3. `Instant` vs `LocalDateTime` for storing timestamps**
```java
// Use Instant for machine timestamps — always UTC, unambiguous
Instant createdAt = Instant.now(); // Store in DB as UTC timestamp

// Use ZonedDateTime when you need to display to a user in their timezone
ZonedDateTime userTime = createdAt.atZone(ZoneId.of("Asia/Kolkata"));

// LocalDateTime is DANGEROUS for events — it's ambiguous (no timezone)
// "2024-03-15T02:30" could be any timezone — avoid for events
```

**4. DST (Daylight Saving Time) edge case**
```java
// In UK, clocks go forward on last Sunday of March at 01:00 → 02:00
// 01:30 UK time does NOT EXIST on that day!
ZoneId london = ZoneId.of("Europe/London");
LocalDateTime gapTime = LocalDateTime.of(2024, 3, 31, 1, 30);
ZonedDateTime zdt = ZonedDateTime.of(gapTime, london);
// Java resolves this automatically by adjusting forward — be aware of this!
System.out.println(zdt); // 2024-03-31T02:30+01:00[Europe/London]
```

---

## 6. Default and Static Interface Methods

### Default Methods

Allow adding new methods to interfaces without breaking existing implementations.

```java
// Real-world: Extending a Repository interface
public interface CustomerRepository {
    Optional<Customer> findById(Long id);
    List<Customer> findAll();

    // Default method — all implementations inherit this
    default Customer findByIdOrThrow(Long id) {
        return findById(id)
            .orElseThrow(() -> new CustomerNotFoundException(id));
    }

    default List<Customer> findActive() {
        return findAll().stream()
            .filter(Customer::isActive)
            .collect(Collectors.toList());
    }
}
```

### Corner Cases: Diamond Problem

```java
interface A {
    default void hello() { System.out.println("A"); }
}

interface B extends A {
    default void hello() { System.out.println("B"); }
}

// Class implementing both must resolve conflict explicitly
class C implements A, B {
    @Override
    public void hello() {
        B.super.hello(); // Explicitly choose B's implementation
    }
}
```

---

## 7. CompletableFuture — Async Programming

`CompletableFuture<T>` enables non-blocking, composable async operations — replacing callbacks and `Future.get()` blocking.

### Real-World: Product Details Aggregation

```java
public class ProductService {

    // Fetch product details, reviews, and inventory in parallel
    public ProductPage getProductPage(Long productId) throws ExecutionException, InterruptedException {

        CompletableFuture<Product> productFuture =
            CompletableFuture.supplyAsync(() -> productRepo.findById(productId));

        CompletableFuture<List<Review>> reviewsFuture =
            CompletableFuture.supplyAsync(() -> reviewService.findByProduct(productId));

        CompletableFuture<InventoryStatus> inventoryFuture =
            CompletableFuture.supplyAsync(() -> inventoryService.getStatus(productId));

        // Wait for all three in parallel — not sequential
        return CompletableFuture.allOf(productFuture, reviewsFuture, inventoryFuture)
            .thenApply(v -> new ProductPage(
                productFuture.join(),
                reviewsFuture.join(),
                inventoryFuture.join()
            ))
            .get();
    }

    // Chaining async operations
    public CompletableFuture<OrderConfirmation> placeOrderAsync(OrderRequest request) {
        return CompletableFuture
            .supplyAsync(() -> inventoryService.reserve(request))   // Step 1
            .thenApplyAsync(reservation -> paymentService.charge(request, reservation)) // Step 2
            .thenApplyAsync(payment -> orderService.create(request, payment))           // Step 3
            .thenApplyAsync(order -> notificationService.sendConfirmation(order))       // Step 4
            .exceptionally(ex -> {
                log.error("Order failed: {}", ex.getMessage());
                compensationService.rollback(request);
                throw new OrderFailedException(ex);
            });
    }
}
```

### Corner Cases

```java
// 1. exceptionally vs handle
future.exceptionally(ex -> defaultValue);      // Only handles exceptions
future.handle((result, ex) -> {                // Handles both success and failure
    if (ex != null) return defaultValue;
    return result;
});

// 2. Beware of ForkJoinPool.commonPool() — limited threads
// Long-running tasks can starve the common pool
// Provide a custom executor for I/O bound tasks
ExecutorService ioPool = Executors.newFixedThreadPool(50);
CompletableFuture.supplyAsync(() -> slowDbCall(), ioPool);

// 3. join() vs get()
// get() throws checked exceptions (InterruptedException, ExecutionException)
// join() throws unchecked CompletionException — easier in streams
List<CompletableFuture<Result>> futures = ...;
List<Result> results = futures.stream()
    .map(CompletableFuture::join) // join() works in lambda; get() doesn't (checked exception)
    .collect(Collectors.toList());
```

---

## 8. Collectors — Advanced Patterns

```java
// Joining
String emails = customers.stream()
    .map(Customer::getEmail)
    .collect(Collectors.joining(", ", "[", "]"));
// "[alice@example.com, bob@example.com]"

// Partitioning (split into two groups by predicate)
Map<Boolean, List<Order>> partitioned = orders.stream()
    .collect(Collectors.partitioningBy(Order::isPaid));
List<Order> paid = partitioned.get(true);
List<Order> unpaid = partitioned.get(false);

// Multi-level grouping
Map<String, Map<OrderStatus, List<Order>>> nested = orders.stream()
    .collect(Collectors.groupingBy(
        Order::getCustomerId,
        Collectors.groupingBy(Order::getStatus)
    ));

// Collecting statistics
IntSummaryStatistics stats = orders.stream()
    .mapToInt(Order::getItemCount)
    .summaryStatistics();
// stats.getMin(), getMax(), getAverage(), getSum(), getCount()

// Custom collector (advanced)
Collector<Order, ?, OrderSummary> orderSummaryCollector = Collector.of(
    OrderSummary::new,                          // Supplier
    (summary, order) -> summary.add(order),     // Accumulator
    (s1, s2) -> s1.merge(s2),                  // Combiner (parallel)
    OrderSummary::finish                        // Finisher
);
```

---

📘 [← Back to Index](./index.md) | ➡️ [Java 9–11 →](./java-9-10-11.md)
