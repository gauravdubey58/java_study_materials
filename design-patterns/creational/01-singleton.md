[← Index](../index.md)

---

# Pattern 1 — Singleton
**Category:** Creational

---

## Intent

Ensure a class has **only one instance** throughout the application's lifetime, and provide a single global point of access to it.

---

## The Problem It Solves

Some resources in a system must be shared and should exist as exactly one instance:
- A database connection pool — you don't want dozens of pools competing for connections
- A configuration manager — all parts of the app should read the same config
- A logging service — one logger, one log file

Without Singleton, different parts of the code might create multiple instances, leading to inconsistent state, wasted resources, or conflicting writes.

---

## The Solution

Make the constructor **private** so no external code can call `new`. Provide a **static method** that creates the instance on first call and returns the same instance on every subsequent call.

```
┌─────────────────────────────────┐
│           Singleton             │
├─────────────────────────────────┤
│ - instance: Singleton (static)  │
├─────────────────────────────────┤
│ - Singleton()                   │  ← private constructor
│ + getInstance(): Singleton      │  ← static factory method
│ + businessMethod()              │
└─────────────────────────────────┘
```

---

## Java Implementation

### Version 1: Eager Initialisation (simplest, thread-safe)

```java
public class AppConfig {

    // Instance created at class-load time — always thread-safe
    private static final AppConfig INSTANCE = new AppConfig();

    private final Properties props;

    private AppConfig() {
        props = new Properties();
        try (InputStream is = getClass().getResourceAsStream("/app.properties")) {
            props.load(is);
        } catch (Exception e) {
            throw new RuntimeException("Failed to load config", e);
        }
    }

    public static AppConfig getInstance() {
        return INSTANCE;
    }

    public String get(String key) {
        return props.getProperty(key);
    }
}
```

### Version 2: Lazy Initialisation with Double-Checked Locking (thread-safe, lazy)

```java
public class DatabaseConnectionPool {

    // volatile ensures visibility across threads
    private static volatile DatabaseConnectionPool instance;

    private final List<Connection> pool;
    private final int poolSize = 10;

    private DatabaseConnectionPool() {
        pool = new ArrayList<>();
        for (int i = 0; i < poolSize; i++) {
            pool.add(createConnection());
        }
    }

    public static DatabaseConnectionPool getInstance() {
        if (instance == null) {                          // first check (no lock)
            synchronized (DatabaseConnectionPool.class) {
                if (instance == null) {                  // second check (with lock)
                    instance = new DatabaseConnectionPool();
                }
            }
        }
        return instance;
    }

    public synchronized Connection getConnection() {
        return pool.remove(pool.size() - 1);
    }

    public synchronized void releaseConnection(Connection conn) {
        pool.add(conn);
    }

    private Connection createConnection() {
        // ... create JDBC connection
        return null;
    }
}
```

### Version 3: Enum Singleton (best practice in Java — serialisation-safe)

```java
public enum LoggerService {

    INSTANCE;

    private final List<String> logs = new ArrayList<>();

    public void log(String message) {
        String entry = LocalDateTime.now() + " — " + message;
        logs.add(entry);
        System.out.println(entry);
    }

    public List<String> getLogs() {
        return Collections.unmodifiableList(logs);
    }
}

// Usage:
LoggerService.INSTANCE.log("Application started");
LoggerService.INSTANCE.log("Processing order #42");
```

The enum approach automatically handles serialisation and reflection attacks.

---

## Real-World Use Cases

**Java Standard Library**
- `Runtime.getRuntime()` — single JVM runtime instance
- `System.out` — single standard output stream

**Spring Framework**
- All Spring beans are Singletons by default (`@Scope("singleton")`)
- The `ApplicationContext` itself is typically a singleton

**Logging frameworks**
- Log4j's `LogManager` — one manager for all loggers
- SLF4J's `LoggerFactory`

**Hardware access**
- Printer spooler
- File system manager
- GPU/graphics context

---

## When to Use

- When exactly one instance of a class must exist
- When you need controlled access to a shared resource
- When the single instance should be accessible globally

## When to Avoid

- When it makes unit testing hard (Singletons are difficult to mock — prefer dependency injection instead)
- When you need different configurations in different contexts (use a factory instead)
- When it introduces hidden global state that creates tight coupling

> **Modern alternative**: In Spring/CDI applications, use **dependency injection** rather than the Singleton pattern directly. Let the container manage object lifecycle — it achieves the same goal without the testability problems.

---

| | |
|---|---|
| [← Index](../index.md) | [Next → Factory Method](./02-factory-method.md) |
