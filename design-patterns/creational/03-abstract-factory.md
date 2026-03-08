[← Factory Method](./02-factory-method.md) | [Index](../index.md)

---

# Pattern 3 — Abstract Factory
**Category:** Creational

---

## Intent

Provide an interface for creating **families of related objects** without specifying their concrete classes.

---

## The Problem It Solves

Imagine a UI framework that must support Windows and macOS. Each OS has its own style of Button, Checkbox, and TextField. You need to guarantee that the UI components created for one OS are never mixed with another. Factory Method would still require the client to choose which factory to call — Abstract Factory ensures the entire family is created consistently.

---

## The Solution

```
┌──────────────────────────┐
│   GUIFactory (interface) │
│ + createButton()         │
│ + createCheckbox()       │
└────────────┬─────────────┘
             │
    ┌────────┴──────────┐
    │                   │
┌───┴──────────┐  ┌─────┴────────────┐
│ WinFactory   │  │ MacFactory       │
│+createButton │  │+createButton()   │
│+createCheckbox│  │+createCheckbox() │
└──────────────┘  └──────────────────┘
```

---

## Java Implementation

```java
// Abstract products
public interface Button    { void render(); void onClick(); }
public interface Checkbox  { void render(); void toggle(); }

// Windows family
public class WindowsButton implements Button {
    @Override public void render()   { System.out.println("Rendering Windows-style button"); }
    @Override public void onClick()  { System.out.println("Windows button clicked"); }
}
public class WindowsCheckbox implements Checkbox {
    @Override public void render()   { System.out.println("Rendering Windows-style checkbox"); }
    @Override public void toggle()   { System.out.println("Windows checkbox toggled"); }
}

// macOS family
public class MacButton implements Button {
    @Override public void render()   { System.out.println("Rendering macOS-style button"); }
    @Override public void onClick()  { System.out.println("Mac button clicked"); }
}
public class MacCheckbox implements Checkbox {
    @Override public void render()   { System.out.println("Rendering macOS-style checkbox"); }
    @Override public void toggle()   { System.out.println("Mac checkbox toggled"); }
}

// Abstract Factory interface
public interface GUIFactory {
    Button   createButton();
    Checkbox createCheckbox();
}

// Concrete Factories
public class WindowsFactory implements GUIFactory {
    @Override public Button   createButton()   { return new WindowsButton(); }
    @Override public Checkbox createCheckbox() { return new WindowsCheckbox(); }
}
public class MacFactory implements GUIFactory {
    @Override public Button   createButton()   { return new MacButton(); }
    @Override public Checkbox createCheckbox() { return new MacCheckbox(); }
}

// Application that uses only the abstract factory
public class Application {
    private final Button   button;
    private final Checkbox checkbox;

    public Application(GUIFactory factory) {
        this.button   = factory.createButton();
        this.checkbox = factory.createCheckbox();
    }

    public void render() {
        button.render();
        checkbox.render();
    }
}

// Wiring
public class Main {
    public static void main(String[] args) {
        String os = System.getProperty("os.name").toLowerCase();
        GUIFactory factory = os.contains("win") ? new WindowsFactory() : new MacFactory();
        new Application(factory).render();
    }
}
```

---

## Real-World Use Cases

- **JDBC** — `Connection` is an abstract factory producing `Statement`, `PreparedStatement`, `CallableStatement`
- **Java AWT/Swing** — look-and-feel (L&F) system produces platform-consistent components
- **Spring Data** — `RepositoryFactory` creates matching repository implementations for JPA, MongoDB, Redis

## When to Use

- When your code must work with families of related products
- When you want to enforce that products from one family are used together

---

| | |
|---|---|
| [← Factory Method](./02-factory-method.md) | [Next → Builder](./04-builder.md) |

---
---

# Pattern 4 — Builder
**Category:** Creational

---

## Intent

Separate the **construction** of a complex object from its **representation**, allowing the same construction process to create different representations.

---

## The Problem It Solves

A class with many optional parameters leads to telescoping constructors (`new Pizza("large", true, false, true, false, "extra cheese", null, 3)`). This is unreadable, error-prone, and inflexible. Builder solves this by providing a fluent API to set only the parameters you need.

---

## Java Implementation

```java
// The complex object
public class HttpRequest {
    private final String  method;
    private final String  url;
    private final Map<String, String> headers;
    private final String  body;
    private final int     timeoutMs;
    private final boolean followRedirects;

    // Private constructor — only the Builder can call it
    private HttpRequest(Builder builder) {
        this.method          = builder.method;
        this.url             = builder.url;
        this.headers         = Collections.unmodifiableMap(builder.headers);
        this.body            = builder.body;
        this.timeoutMs       = builder.timeoutMs;
        this.followRedirects = builder.followRedirects;
    }

    // Getters...
    public String getMethod()  { return method; }
    public String getUrl()     { return url; }
    public String getBody()    { return body; }

    @Override
    public String toString() {
        return String.format("%s %s (timeout=%dms, redirects=%b)%nHeaders: %s%nBody: %s",
                method, url, timeoutMs, followRedirects, headers, body);
    }

    // Static inner Builder class
    public static class Builder {
        // Required parameters
        private final String method;
        private final String url;

        // Optional parameters with defaults
        private Map<String, String> headers = new HashMap<>();
        private String  body            = null;
        private int     timeoutMs       = 5000;
        private boolean followRedirects = true;

        public Builder(String method, String url) {
            this.method = method;
            this.url    = url;
        }

        public Builder header(String key, String value) {
            this.headers.put(key, value);
            return this;    // fluent chaining
        }

        public Builder body(String body) {
            this.body = body;
            return this;
        }

        public Builder timeout(int timeoutMs) {
            this.timeoutMs = timeoutMs;
            return this;
        }

        public Builder followRedirects(boolean follow) {
            this.followRedirects = follow;
            return this;
        }

        public HttpRequest build() {
            // Validation
            if (method == null || method.isBlank())
                throw new IllegalStateException("HTTP method is required");
            if (url == null || url.isBlank())
                throw new IllegalStateException("URL is required");
            return new HttpRequest(this);
        }
    }
}

// Usage — clean, readable, and self-documenting
HttpRequest request = new HttpRequest.Builder("POST", "https://api.example.com/orders")
        .header("Authorization", "Bearer token123")
        .header("Content-Type", "application/json")
        .body("{\"item\": \"book\", \"qty\": 2}")
        .timeout(3000)
        .followRedirects(false)
        .build();
```

---

## Real-World Use Cases

- **Lombok's `@Builder`** — generates builder code automatically
- **StringBuilder / StringBuffer** — simple builder for Strings
- **OkHttp's Request.Builder** — HTTP client request construction
- **JPA Criteria API** — `CriteriaBuilder` for type-safe queries
- **Java Stream's Collectors** — complex collector composition

## When to Use

- When constructing an object requires many parameters, especially optional ones
- When you want immutable objects with a readable construction API
- When the same construction process should produce different representations

---

| | |
|---|---|
| [← Abstract Factory](./03-abstract-factory.md) | [Next → Prototype](./05-prototype.md) |

---
---

# Pattern 5 — Prototype
**Category:** Creational

---

## Intent

Create new objects by **copying (cloning) an existing object**, rather than creating from scratch.

---

## The Problem It Solves

Creating an object from scratch can be expensive — it may involve complex initialisation, database calls, or lengthy configuration. If you need many similar objects, cloning an already-configured "prototype" is far cheaper.

---

## Java Implementation

```java
// Prototype interface
public interface Prototype {
    Prototype clone();
}

// Concrete prototype — a configurable report template
public class ReportTemplate implements Prototype {
    private String  title;
    private String  headerColor;
    private List<String> sections;
    private Map<String, String> metadata;

    public ReportTemplate(String title, String headerColor) {
        this.title       = title;
        this.headerColor = headerColor;
        this.sections    = new ArrayList<>();
        this.metadata    = new HashMap<>();
    }

    // Deep copy constructor
    private ReportTemplate(ReportTemplate source) {
        this.title       = source.title;
        this.headerColor = source.headerColor;
        this.sections    = new ArrayList<>(source.sections);    // deep copy list
        this.metadata    = new HashMap<>(source.metadata);      // deep copy map
    }

    @Override
    public ReportTemplate clone() {
        return new ReportTemplate(this);
    }

    public ReportTemplate withTitle(String title) {
        ReportTemplate copy = this.clone();
        copy.title = title;
        return copy;
    }

    public ReportTemplate addSection(String section) {
        this.sections.add(section);
        return this;
    }

    @Override
    public String toString() {
        return String.format("Report[title=%s, color=%s, sections=%s]",
                title, headerColor, sections);
    }
}

// Prototype Registry
public class TemplateRegistry {
    private final Map<String, ReportTemplate> cache = new HashMap<>();

    public void register(String key, ReportTemplate template) {
        cache.put(key, template);
    }

    public ReportTemplate get(String key) {
        ReportTemplate proto = cache.get(key);
        if (proto == null) throw new IllegalArgumentException("Unknown template: " + key);
        return proto.clone();   // always return a clone, never the original
    }
}

// Usage
TemplateRegistry registry = new TemplateRegistry();

// Build and register prototypes ONCE (expensive setup)
ReportTemplate quarterly = new ReportTemplate("Q? Report", "#003366")
        .addSection("Executive Summary")
        .addSection("Financial Results")
        .addSection("Outlook");
registry.register("quarterly", quarterly);

// Clone cheaply whenever needed
ReportTemplate q1 = registry.get("quarterly").withTitle("Q1 2024 Report");
ReportTemplate q2 = registry.get("quarterly").withTitle("Q2 2024 Report");
```

---

## Real-World Use Cases

- **Java's `Object.clone()`** — Cloneable interface (though often considered flawed)
- **Spring's prototype scope** — `@Scope("prototype")` gives a fresh bean copy per request
- **Game development** — Clone enemy or terrain tile objects from prototypes
- **Document templates** — Clone a base document and customise for each recipient

## When to Use

- When object creation is costly and you need many similar instances
- When you need independent copies of objects that won't affect each other
- When you want to avoid rebuilding complex object graphs from scratch

## When to Avoid

- When the objects being cloned have circular references (deep copy becomes complex)
- When objects are immutable (just reuse them directly — no need to clone)

---

| | |
|---|---|
| [← Abstract Factory](./03-abstract-factory.md) | [Next → Adapter](../structural/06-adapter.md) |
