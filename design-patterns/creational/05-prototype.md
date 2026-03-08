[← Builder](./04-builder.md) | [Index](../index.md)

---

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
