[← Interpreter](./23-interpreter.md) | [Index](../index.md)

---

# 50 Design Patterns Interview Questions & Answers

---

## Section 1 — Fundamentals (Q1–Q10)

---

**Q1. What is a design pattern? Why are they important?**

A design pattern is a reusable, named solution to a commonly occurring problem in software design. They are important because they:
- Provide a shared vocabulary for developers ("use a Singleton here" communicates more than explaining the code)
- Encode proven solutions that avoid known pitfalls
- Speed up design by providing tested templates
- Make code easier to understand and maintain

---

**Q2. What are the three categories of GoF design patterns?**

**Creational** — concerned with how objects are created (Singleton, Factory Method, Abstract Factory, Builder, Prototype).

**Structural** — concerned with how classes and objects are composed to form larger structures (Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy).

**Behavioral** — concerned with algorithms and how objects interact and distribute responsibility (Chain of Responsibility, Command, Iterator, Mediator, Memento, Observer, State, Strategy, Template Method, Visitor, Interpreter).

---

**Q3. What is the difference between a design pattern and an algorithm?**

An **algorithm** is a precise, step-by-step procedure for solving a specific computational problem. It defines what happens — there is a definite expected output.

A **design pattern** is a high-level, reusable template for solving a recurring design problem. It defines a structural approach — how to organise classes and objects — not an exact sequence of steps. The same pattern can be implemented in many different ways.

---

**Q4. What are the SOLID principles and how do they relate to design patterns?**

**S — Single Responsibility**: A class should have one reason to change. Patterns like Facade and Command help by separating concerns.

**O — Open/Closed**: Open for extension, closed for modification. Strategy, Decorator, and Template Method embody this.

**L — Liskov Substitution**: Subtypes must be fully substitutable for their base types. Violated by careless use of Singleton or inheritance hierarchies.

**I — Interface Segregation**: Clients should not depend on interfaces they don't use. Facade and Adapter help by providing tailored interfaces.

**D — Dependency Inversion**: Depend on abstractions, not concretions. Factory Method, Abstract Factory, and Proxy all depend on abstractions.

---

**Q5. When should you NOT use a design pattern?**

- When the problem doesn't require it — patterns add complexity, and over-engineering is a real risk (known as "pattern fever")
- When the pattern solves a problem you don't actually have
- When a simpler solution (a plain class, a lambda, a utility method) works just as well
- When using a framework that already provides the solution natively (e.g., Spring's DI removes the need for manual Singletons)

> "A pattern is a solution to a problem in a context." — Christopher Alexander

---

**Q6. What is the difference between class patterns and object patterns?**

**Class patterns** use inheritance to compose interfaces and implementations. Relationships are established at compile time. Examples: Template Method (inheritance-based), Factory Method.

**Object patterns** use object composition. Relationships are established at runtime and are more flexible. Examples: Strategy (composition), Decorator (wrapping), Observer (list of observers).

Modern Java favours composition over inheritance: "Favour object composition over class inheritance."

---

**Q7. Explain the principle "Program to an interface, not an implementation."**

Instead of:
```java
ArrayList<String> list = new ArrayList<>();  // bound to ArrayList
```

Write:
```java
List<String> list = new ArrayList<>();  // can swap to LinkedList, CopyOnWriteArrayList, etc.
```

This principle underpins nearly every design pattern. Strategy declares a strategy interface; Decorator wraps an interface; Observer defines an observer interface. Depending on concrete classes makes code brittle and hard to extend.

---

**Q8. What is the difference between Association, Aggregation, and Composition?**

**Association** — A general relationship. Class A uses Class B. No ownership implied. Example: A `Teacher` teaches `Student`s.

**Aggregation** — A "has-a" relationship with independent lifecycles. Class A contains Class B, but B can exist without A. Example: A `Department` has `Employee`s — employees can exist without the department.

**Composition** — A strong "owns-a" relationship. Class B cannot exist without Class A. If A is destroyed, B is too. Example: A `House` has `Room`s — rooms don't exist without the house.

Design patterns frequently use Aggregation (Strategy, Decorator) and Composition (Composite).

---

**Q9. What is coupling and cohesion? How do patterns improve them?**

**Coupling** — the degree to which one class depends on another. Low coupling is desirable; changes in one class don't break others. Patterns like Mediator, Observer, and Facade reduce coupling.

**Cohesion** — the degree to which the responsibilities of a class belong together. High cohesion is desirable; a class does one thing well. Patterns like Strategy (extract algorithms) and Command (extract requests) increase cohesion.

---

**Q10. What is the difference between a pattern and an anti-pattern?**

A **pattern** is a proven, recommended solution to a recurring problem.

An **anti-pattern** is a commonly used but ineffective or counterproductive solution. Examples:
- **God Object** — one class knows and does too much (violates SRP)
- **Spaghetti Code** — tangled, unstructured code with no clear design
- **Golden Hammer** — applying one favourite pattern to every problem ("if all you have is a hammer...")
- **Singleton overuse** — using Singleton as a glorified global variable, making testing impossible

---

## Section 2 — Creational Patterns (Q11–Q18)

---

**Q11. What are the different ways to implement Singleton in Java, and which is best?**

| Implementation | Thread-safe | Lazy | Notes |
|---|---|---|---|
| Eager initialisation | ✅ | ❌ | Simple; instance created at class load |
| Synchronized method | ✅ | ✅ | Safe but slow — synchronises every call |
| Double-checked locking | ✅ | ✅ | Requires `volatile`; fast after init |
| Static inner class | ✅ | ✅ | Elegant; relies on JVM class loading |
| Enum | ✅ | ❌ | Best — handles serialisation and reflection |

**Best practice**: Use the **Enum** singleton for simplicity and safety, or **Static inner class** if lazy initialisation is needed without enum.

```java
public enum AppConfig { INSTANCE;  /* methods here */ }
```

---

**Q12. How does Singleton relate to Spring beans?**

By default, all Spring beans are singletons — but they are container-managed singletons, not class-level singletons. The difference:

- **Class-level Singleton**: controlled by `private constructor` + `static getInstance()`; one instance per ClassLoader
- **Spring Singleton**: controlled by the container; one instance per ApplicationContext

Spring singleton beans are better for testing because you can swap them with mocks by providing a different ApplicationContext configuration. You can also have multiple ApplicationContexts in the same JVM with different "singleton" instances.

---

**Q13. What is the difference between Factory Method and Abstract Factory?**

| | Factory Method | Abstract Factory |
|---|---|---|
| Creates | One product | A family of related products |
| Structure | Subclass overrides a method | Separate factory object |
| Focus | Defer instantiation to subclasses | Ensure product families are consistent |
| Example | `Calendar.getInstance()` | JDBC `Connection` creating `Statement`, `PreparedStatement` |

A good rule of thumb: if you need one object, consider Factory Method. If you need a group of related objects that must be used together, use Abstract Factory.

---

**Q14. When would you use Builder over a constructor with many parameters?**

Use Builder when:
- A class has **4 or more parameters**, especially optional ones
- You want **immutable objects** with a readable construction API
- The construction steps can be reordered or the object may have multiple valid representations

The classic "telescoping constructor" anti-pattern it replaces:
```java
new Pizza(12, true, false, true, false, null, 3);  // what do these mean?

// vs Builder:
new Pizza.Builder(12).cheese(true).mushrooms(true).extraSauce(3).build();
```

---

**Q15. What is the difference between Prototype and copy constructors / clone()?**

| | Prototype Pattern | Java clone() | Copy Constructor |
|---|---|---|---|
| Registry | Often includes a registry | No | No |
| Deep copy | Explicitly defined | Tricky | Explicitly defined |
| Type safety | Yes | Requires cast | Yes |
| Inheritance | Works naturally | Requires Cloneable | Must be reimplemented |

`Object.clone()` is considered problematic in Java (must implement `Cloneable`, default is shallow copy, checked exception). Prefer **copy constructors** or **explicit `clone()` methods** that do deep copying. The Prototype pattern is more about the concept (clone from a registry) than Java's Cloneable mechanism.

---

**Q16. What is a static factory method? How is it different from Factory Method pattern?**

A **static factory method** is simply a static method that returns an instance of a class (Joshua Bloch, Effective Java). It's a programming idiom, not a GoF pattern:

```java
// Static factory methods
Boolean.valueOf(true);
Optional.of("value");
List.of(1, 2, 3);
LocalDate.of(2024, 1, 1);
```

The **Factory Method pattern** is a GoF design pattern where a class defines an interface for object creation and subclasses decide what to create. They share the word "factory" but are different things.

---

**Q17. In what real Java API scenarios do you see the Builder pattern?**

- **`StringBuilder` / `StringBuffer`** — `append().append().toString()`
- **`java.util.stream.Stream.Builder`** — stream construction
- **Lombok's `@Builder`** — code-generated builder for any class
- **`HttpRequest.Builder`** in Java 11+ HTTP client
- **`ProcessBuilder`** — building OS process parameters
- **JPA Criteria API's `CriteriaBuilder`** — type-safe query construction

---

**Q18. What is an Object Pool pattern? How does it relate to creational patterns?**

Object Pool is a creational pattern (not in the original GoF 23) where a set of initialised objects is kept ready for use, avoiding the expensive cost of creating and destroying them repeatedly.

Examples: **database connection pools** (HikariCP, C3P0), **thread pools** (`ExecutorService`), **HTTP connection pools** (Apache HttpClient).

It is most useful when:
- Object creation is expensive (database connections, TCP connections)
- You need many short-lived objects (thread tasks)
- The number of instances needs to be bounded

---

## Section 3 — Structural Patterns (Q19–Q28)

---

**Q19. What is the difference between Adapter and Decorator?**

| | Adapter | Decorator |
|---|---|---|
| **Purpose** | Change the interface | Add behaviour, keep the same interface |
| **Interface** | Client sees a different interface | Client sees the same interface |
| **Wraps** | An incompatible class | The same type as itself |
| **When to use** | Integration with legacy/third-party code | Adding features without subclassing |

Adapter is about **compatibility**. Decorator is about **enhancement**.

---

**Q20. What is the difference between Decorator and Proxy?**

| | Decorator | Proxy |
|---|---|---|
| **Purpose** | Add new behaviour | Control access to existing behaviour |
| **Interface** | Same as decorated object | Same as real subject |
| **Composition** | Decorators can be stacked | Usually one proxy per subject |
| **Knowledge of subject** | Client creates and stacks decorators | Proxy may hide/control the subject |
| **Examples** | Java I/O streams | Spring AOP, Hibernate lazy loading |

Both wrap an object and implement the same interface, but with different intent.

---

**Q21. How does Spring AOP use the Proxy pattern?**

When you annotate a Spring bean with `@Transactional`, `@Cacheable`, or `@Async`, Spring creates a **proxy** around that bean. The proxy intercepts method calls, adds cross-cutting behaviour (begin transaction, check cache, run async), then delegates to the real object.

Spring uses two proxy mechanisms:
- **JDK Dynamic Proxy** — when the bean implements an interface (uses `java.lang.reflect.Proxy`)
- **CGLIB proxy** — when the bean is a concrete class (creates a subclass at runtime)

This is why calling `@Transactional` methods from within the same class doesn't work — the call bypasses the proxy.

---

**Q22. What is the difference between Facade and Mediator?**

| | Facade | Mediator |
|---|---|---|
| **Direction** | One-way — simplifies access to subsystem | Two-way — coordinates communication between peers |
| **Subsystem knowledge** | Subsystem doesn't know about Facade | Colleagues know about the Mediator |
| **Purpose** | Simplify a complex subsystem for clients | Reduce coupling between objects that interact |
| **Example** | `JdbcTemplate` over JDBC | Chat room, event bus |

Facade is about **simplification**. Mediator is about **decoupling peer interactions**.

---

**Q23. Explain the Flyweight pattern with a concrete Java example.**

The Flyweight pattern shares common (intrinsic) state to reduce memory usage.

**Java String pool** is the most famous example: string literals are interned and shared, so `"hello" == "hello"` is `true` — they are the same object in memory.

**Integer cache**: `Integer.valueOf(127) == Integer.valueOf(127)` is `true` (same cached instance), but `Integer.valueOf(128) == Integer.valueOf(128)` is `false` (above the cache range, new objects).

In game development: instead of creating 10,000 `Tree` objects each holding texture data, create one `TreeType` flyweight with the texture, and 10,000 lightweight `Tree` objects that store only position + reference to the shared `TreeType`.

---

**Q24. What is the difference between Composite and Decorator?**

| | Composite | Decorator |
|---|---|---|
| **Structure** | Tree (parent-child hierarchy) | Linear wrapping chain |
| **Purpose** | Treat individual and groups uniformly | Add behaviour to an individual object |
| **Children** | Can have many children | Wraps exactly one component |
| **Focus** | Part-whole hierarchies | Responsibility enhancement |

Composite is for **hierarchies** (file system, UI component trees). Decorator is for **adding features** (encryption + compression + logging on top of a base class).

---

**Q25. When would you use Bridge instead of Adapter?**

**Use Adapter** when you have an existing class with an incompatible interface and you need to make it work with your code. Adapter is applied retroactively.

**Use Bridge** when you are designing a new system and want to avoid permanent binding between abstraction and implementation. Bridge is applied proactively.

Analogy: Adapter is like a travel adapter plug (retrofitted), while Bridge is like designing a car where the engine (implementation) is intentionally separable from the car body (abstraction).

---

**Q26. Where is the Composite pattern used in the Java/Spring ecosystem?**

- **Java AWT/Swing** — `Container extends Component`; containers hold components, which can themselves be containers
- **Spring Security** — `CompositeFilter` chains security filters
- **Spring's `CompositeTransactionAttributeSource`** — combines multiple attribute sources
- **JUnit 5** — `TestSuite` contains test cases or other suites
- **JSON/XML parsing** — object nodes vs value nodes in Jackson's `JsonNode`

---

**Q27. What are Virtual Proxy, Protection Proxy, and Remote Proxy?**

**Virtual Proxy** — delays creation of an expensive object until first use. Example: Hibernate returns a proxy for associated entities; SQL is only executed when you access a field.

**Protection Proxy** — controls access based on permission. Example: Spring Security's method security creates proxies that check `@PreAuthorize` before delegating.

**Remote Proxy** — represents an object in a different address space (different JVM or machine). Example: Java RMI stubs; EJB remote proxies; gRPC client stubs.

**Caching Proxy** — caches results to avoid repeated expensive operations. Example: Spring's `@Cacheable` creates a proxy that caches method results.

---

**Q28. How does the Adapter pattern appear in Java I/O?**

Java I/O has two hierarchies: `InputStream`/`OutputStream` (byte-based) and `Reader`/`Writer` (character-based). `InputStreamReader` and `OutputStreamWriter` are Adapters that bridge between them:

```java
// InputStreamReader adapts InputStream (bytes) to Reader (characters)
Reader reader = new InputStreamReader(System.in, StandardCharsets.UTF_8);

// OutputStreamWriter adapts OutputStream to Writer
Writer writer = new OutputStreamWriter(new FileOutputStream("out.txt"), "UTF-8");
```

`Arrays.asList()` adapts an array to the `List` interface. `Collections.enumeration()` adapts a `Collection` to the older `Enumeration` interface.

---

## Section 4 — Behavioral Patterns (Q29–Q42)

---

**Q29. What is the difference between Strategy and State?**

| | Strategy | State |
|---|---|---|
| **Purpose** | Interchangeable algorithms | Object changes behaviour with internal state |
| **Who changes** | Client changes strategy explicitly | Object transitions states internally |
| **Awareness** | Strategies are independent of each other | States may know about each other (for transitions) |
| **Example** | Payment methods, sorting algorithms | Vending machine, TCP connection, order status |

Key question: **Who** decides to change the behaviour? If the client decides → Strategy. If the object itself decides based on internal state → State.

---

**Q30. What is the difference between Observer and Mediator?**

Both reduce coupling, but differently:

**Observer** — One-to-many. A subject broadcasts to all its observers, who registered their interest. Observers don't communicate with each other; they only receive notifications from the subject.

**Mediator** — Many-to-many. Multiple peers communicate with each other, but only through the mediator. The mediator centralises all the interaction logic.

If you have an event bus where publishers don't know subscribers — that's Observer. If you have a chat room where any participant can message any other — that's Mediator.

---

**Q31. How does Java's ExecutorService relate to the Command pattern?**

`Runnable` and `Callable` are Command objects. `ExecutorService` is the Invoker that queues, schedules, and executes commands. The actual work is in the `run()` or `call()` method (the execute() of the command).

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

// Runnable = Command
Runnable command = () -> System.out.println("Processing order #42");
executor.submit(command);   // Executor queues and runs the command

// Callable = Command with result
Callable<String> callableCommand = () -> "Order processed";
Future<String> result = executor.submit(callableCommand);
```

---

**Q32. Explain how Template Method is used in Spring Batch.**

Spring Batch's `AbstractItemCountingItemStreamItemReader` defines the template method `read()`:

```java
// Template method (simplified)
public final T read() throws Exception {
    T item = doRead();           // abstract — subclass implements
    if (item != null) count++;
    return item;
}
```

`FlatFileItemReader`, `JdbcCursorItemReader`, and `JpaPagingItemReader` all implement `doRead()` differently (CSV parsing, JDBC ResultSet, JPA query), but they all share the same read lifecycle management.

`JdbcTemplate` is also Template Method: `execute()` manages connection/statement lifecycle; you provide the SQL and parameter callback.

---

**Q33. What is the difference between Iterator and for-each loop in Java?**

The **for-each loop** (`for (T item : collection)`) is syntactic sugar that the compiler translates into:
1. A call to `collection.iterator()` to get an `Iterator`
2. A `while (iterator.hasNext())` loop calling `iterator.next()`

To use for-each, a class must implement `Iterable<T>`, which requires implementing `iterator()`. This is the Iterator pattern built into the Java language.

The explicit Iterator is useful when you need to:
- Remove elements while iterating (`iterator.remove()`)
- Iterate two collections simultaneously
- Use a cursor-based approach (databases)

---

**Q34. How does Chain of Responsibility differ from Decorator?**

| | Chain of Responsibility | Decorator |
|---|---|---|
| **Request handling** | Only one handler processes the request | All decorators process every call |
| **Chain breaking** | A handler may stop the chain | Chain is always fully traversed |
| **Purpose** | Find the right handler | Add behaviour around the same operation |
| **Example** | Exception handler chains, support escalation | Java I/O streams, Spring filters |

Spring Security filters are an interesting case — they behave like CoR (a filter can short-circuit), but also like Decorator (they wrap request/response). In practice, both patterns appear together.

---

**Q35. What is the Memento pattern and where is it used in Java?**

Memento captures an object's internal state without exposing it, so the state can be restored later (undo functionality).

Real-world Java uses:
- **`Serializable`** — serialising an object to bytes is capturing its memento; deserialising is restoration
- **Database transactions** — `SAVEPOINT` and `ROLLBACK TO SAVEPOINT` are mementos
- **Java undo frameworks** — `javax.swing.undo.UndoManager` and `UndoableEdit`
- **Git commits** — each commit is a memento of the repository state

---

**Q36. What is the difference between Command and Strategy?**

Both encapsulate behaviour in an object, but with different intents:

**Command** — encapsulates a **request** to be executed. Focus is on what happened (can be logged, queued, undone). A Command is typically executed once.

**Strategy** — encapsulates an **algorithm**. Focus is on how something is done. A Strategy is configured once and used repeatedly.

Think of it as: Command = noun ("the action to take"), Strategy = adjective ("the way to do it").

---

**Q37. Where do you see Observer in modern Java frameworks?**

- **Spring Events** — `@EventListener`, `ApplicationEventPublisher`
- **JavaFX** — `ObservableList`, `ChangeListener`, `InvalidationListener`
- **RxJava** — `Observable.subscribe(observer)` is Observer at scale
- **Project Reactor** — `Flux` and `Mono` publishers with subscribers
- **Spring WebSocket** — server pushes events to connected clients
- **JMX** — `NotificationListener` for monitoring events

---

**Q38. Explain how the Visitor pattern uses "double dispatch."**

Java uses **single dispatch** — method selection is based on the runtime type of the receiver only.

Visitor achieves **double dispatch** — method selection based on BOTH the visitor type AND the element type — using two polymorphic calls:

```java
// First dispatch: based on element type (Circle, Rectangle, etc.)
element.accept(visitor);

// Inside accept() — second dispatch: based on visitor type (HtmlVisitor, PdfVisitor, etc.)
public void accept(DocumentVisitor visitor) {
    visitor.visitCircle(this);   // visitor's type determines which visitXxx is called
}
```

The result is that the right operation for the right element type is always invoked, even in languages without multi-methods.

---

**Q39. How does the State pattern eliminate complex if/else chains?**

Without State:
```java
public void insertCoin() {
    if (state == IDLE)          { state = COIN_INSERTED; System.out.println("OK"); }
    else if (state == COIN_INSERTED) { System.out.println("Already inserted"); }
    else if (state == EMPTY)    { System.out.println("Machine empty"); }
    // ... every method repeats this structure for every state
}
```

With State:
```java
// Context delegates to current state object
public void insertCoin() { state.insertCoin(this); }

// Each State class handles only its own behaviour
class IdleState { void insertCoin(Machine m) { m.setState(new CoinInsertedState()); } }
```

Adding a new state means adding a new class, not modifying existing ones (Open/Closed Principle).

---

**Q40. What is the Null Object pattern and how does it relate to behavioral patterns?**

The **Null Object** pattern (not in the original GoF 23) provides a do-nothing object that implements an interface, eliminating the need for null checks.

```java
// Without Null Object
Logger logger = getLogger();  // might return null
if (logger != null) {
    logger.log("message");
}

// With Null Object
public class NullLogger implements Logger {
    @Override public void log(String msg) { /* do nothing */ }
}

Logger logger = getLogger();  // always returns real or NullLogger — never null
logger.log("message");        // always safe
```

It relates to Strategy (interchangeable behaviour), and to Java 8's `Optional` (which is a monadic null object).

---

**Q41. What is the difference between Mediator and Event Bus?**

An **Event Bus** (like Spring's ApplicationEventPublisher or Guava EventBus) is a specific implementation of the Mediator pattern where:
- Publishers fire events without knowing who receives them
- Subscribers register interest in specific event types
- The bus (mediator) routes events to appropriate subscribers

The difference: a traditional Mediator may contain complex coordination logic and knows all its colleagues explicitly. An Event Bus is a more generic, lightweight mediator with no knowledge of business logic — it just routes messages by type.

---

**Q42. How does the Interpreter pattern differ from using a parser library?**

| | Interpreter Pattern | Parser Library (ANTLR, Parboiled) |
|---|---|---|
| **Grammar size** | Simple (< 10 rules) | Any complexity |
| **Implementation** | Hand-coded class per grammar rule | Generated from grammar specification |
| **Performance** | Slow (object tree traversal) | Fast (compiled) |
| **Flexibility** | Hard to change grammar | Easy to modify grammar file |
| **Example** | SpEL, simple rule engines | SQL parsers, full language compilers |

Use Interpreter for simple domain-specific languages. Use a parser generator for anything complex.

---

## Section 5 — Advanced & Real-World (Q43–Q50)

---

**Q43. How do you identify which pattern to use when solving a problem?**

Ask these questions:

1. **What kind of problem is it?**
   - Creating objects → Creational
   - Composing objects → Structural
   - Objects communicating → Behavioral

2. **What is the core challenge?**
   - "I need exactly one instance" → Singleton
   - "I need to add behaviour without changing the class" → Decorator
   - "I need to handle multiple variants of an algorithm" → Strategy
   - "I need to notify many objects about a change" → Observer
   - "I want to undo operations" → Command + Memento

3. **What relationship is involved?**
   - Is-a → consider inheritance-based patterns
   - Has-a → consider composition-based patterns

---

**Q44. Can multiple patterns be used together? Give an example.**

Yes, absolutely. Real applications routinely combine patterns.

**Example: Spring MVC request handling**

- **Front Controller** (Facade) — `DispatcherServlet` is the single entry point
- **Command** — each `@RequestMapping` handler encapsulates a request
- **Strategy** — `HandlerMapping` strategies select the right handler
- **Chain of Responsibility** — interceptors process the request in a chain
- **Template Method** — `AbstractController.handleRequest()` defines the template
- **Observer** — `ApplicationEventPublisher` notifies listeners after request processing

---

**Q45. What design patterns are commonly used in microservices architecture?**

| Pattern | Microservice Use Case |
|---|---|
| **Facade** | API Gateway — single entry point for all services |
| **Proxy** | Service Mesh (Envoy/Istio sidecar proxies) |
| **Observer** | Event-driven architecture — services publish/subscribe to events |
| **Command** | CQRS — commands modify state, queries read state |
| **Chain of Responsibility** | API Gateway middleware, request filters |
| **Strategy** | Circuit breaker fallback strategies |
| **Template Method** | Base service implementations in shared libraries |
| **Decorator** | Adding retry logic, rate limiting, logging to service clients |

---

**Q46. What is the difference between Dependency Injection and Factory patterns?**

Both solve the "create the right object" problem, but differently:

**Factory** — the client asks the factory for an object. There is still a dependency on the factory itself.

**Dependency Injection (DI)** — the container creates the object and injects it into the client. The client never asks for anything — it declares what it needs (via constructor or `@Autowired`), and the container provides it. The client has no dependency on any factory.

DI (IoC containers like Spring) makes Factory patterns largely unnecessary for application code — but factories are still used to configure the container itself.

---

**Q47. How would you refactor a large switch/if-else block using design patterns?**

```java
// Before: large switch statement
public double calculate(String type, double a, double b) {
    return switch (type) {
        case "add"      -> a + b;
        case "subtract" -> a - b;
        case "multiply" -> a * b;
        case "divide"   -> a / b;
        default         -> throw new IllegalArgumentException(type);
    };
}
```

Refactor using **Strategy** + **Factory Method**:

```java
public interface MathOperation { double apply(double a, double b); }

Map<String, MathOperation> operations = Map.of(
    "add",      (a, b) -> a + b,
    "subtract", (a, b) -> a - b,
    "multiply", (a, b) -> a * b,
    "divide",   (a, b) -> a / b
);

public double calculate(String type, double a, double b) {
    MathOperation op = operations.get(type);
    if (op == null) throw new IllegalArgumentException(type);
    return op.apply(a, b);
}
```

Now adding a new operation means adding an entry to the map — no modification of existing code.

---

**Q48. What patterns are most commonly asked about in interviews?**

In order of frequency:

1. **Singleton** — always asked. Know thread-safety and the enum approach.
2. **Observer** — know how Spring Events and RxJava use it.
3. **Strategy** — know how `Comparator` is Strategy.
4. **Decorator** — know Java I/O streams.
5. **Factory Method / Abstract Factory** — understand the difference.
6. **Builder** — know when to use it vs constructors.
7. **Proxy** — know how Spring AOP uses it.
8. **Template Method** — know how JdbcTemplate and Spring Batch use it.
9. **Command** — know Runnable/Callable as commands.
10. **Composite** — know the file system and UI component examples.

---

**Q49. How do design patterns relate to functional programming and lambdas in Java?**

Java 8+ lambdas replace some patterns with simpler code:

| Pattern | Lambda equivalent |
|---|---|
| **Command** | `Runnable r = () -> doWork();` |
| **Strategy** | `Comparator<T> comp = (a, b) -> a.compareTo(b);` |
| **Observer** | `button.addActionListener(e -> handleClick());` |
| **Template Method** | Higher-order functions: `process(data, this::transform)` |
| **Factory Method** | `Supplier<Connection> factory = DriverManager::getConnection;` |

Patterns like Singleton, Decorator, Composite, Proxy, and Builder still benefit from class-based implementations. Lambdas simplify patterns involving "encapsulate a single behaviour" but don't replace structural patterns.

---

**Q50. How would you explain design patterns to a junior developer?**

Use an analogy: design patterns are like **recipes in a cookbook**.

A recipe isn't a specific meal — it's a proven method for preparing a category of dish. "Beef stew recipe" tells you the general approach: brown the meat, add vegetables, slow cook in liquid. You adapt the specifics (which vegetables, what spices) to your situation.

Similarly, the "Observer pattern" isn't specific code — it's a proven approach for "notify multiple objects when one changes." You adapt the specific classes, interfaces, and notification mechanism to your project.

The value isn't that patterns save you thinking — it's that they give you a vocabulary to **communicate design decisions** quickly and unambiguously with your team, and they encode lessons learned by thousands of developers before you.

---

*For further study: "Design Patterns" by GoF (1994), "Head First Design Patterns" (Freeman & Freeman), "Effective Java" by Joshua Bloch*

---

| | |
|---|---|
| [← Interpreter](./23-interpreter.md) | [↑ Back to Index](../index.md) |
