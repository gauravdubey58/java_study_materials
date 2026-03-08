# Pattern 9 — Decorator
**Category:** Structural

---

## Intent

**Attach additional responsibilities to an object dynamically**. Decorators provide a flexible alternative to subclassing for extending functionality.

---

## The Problem It Solves

You want to add logging, caching, compression, or encryption to a class, but:
- You can't (or don't want to) modify the original class
- Subclassing would create a class explosion (LoggingCachingCompressingRepository, etc.)

Decorator wraps the original object and adds behaviour before/after delegating to it.

---

## Java Implementation

```java
// Component interface
public interface DataSource {
    void writeData(String data);
    String readData();
}

// Concrete Component (the real implementation)
public class FileDataSource implements DataSource {
    private final String filename;

    public FileDataSource(String filename) {
        this.filename = filename;
    }

    @Override
    public void writeData(String data) {
        System.out.println("Writing to file: " + filename);
        // ... actual file I/O
    }

    @Override
    public String readData() {
        System.out.println("Reading from file: " + filename);
        return "raw data";  // ... actual file read
    }
}

// Base Decorator — delegates everything to the wrapped component
public abstract class DataSourceDecorator implements DataSource {
    protected final DataSource wrapped;

    public DataSourceDecorator(DataSource wrapped) {
        this.wrapped = wrapped;
    }

    @Override
    public void writeData(String data) { wrapped.writeData(data); }

    @Override
    public String readData() { return wrapped.readData(); }
}

// Concrete Decorators
public class EncryptionDecorator extends DataSourceDecorator {
    public EncryptionDecorator(DataSource wrapped) { super(wrapped); }

    @Override
    public void writeData(String data) {
        System.out.println("[Encryption] Encrypting data...");
        wrapped.writeData(encrypt(data));
    }

    @Override
    public String readData() {
        return decrypt(wrapped.readData());
    }

    private String encrypt(String data) { return "ENCRYPTED[" + data + "]"; }
    private String decrypt(String data) { return data.replace("ENCRYPTED[", "").replace("]", ""); }
}

public class CompressionDecorator extends DataSourceDecorator {
    public CompressionDecorator(DataSource wrapped) { super(wrapped); }

    @Override
    public void writeData(String data) {
        System.out.println("[Compression] Compressing data...");
        wrapped.writeData(compress(data));
    }

    @Override
    public String readData() {
        return decompress(wrapped.readData());
    }

    private String compress(String data)   { return "COMPRESSED[" + data + "]"; }
    private String decompress(String data) { return data.replace("COMPRESSED[", "").replace("]", ""); }
}

// Usage — stack decorators like layers
DataSource source = new FileDataSource("data.txt");
DataSource encrypted = new EncryptionDecorator(source);
DataSource encryptedAndCompressed = new CompressionDecorator(encrypted);

encryptedAndCompressed.writeData("Hello World");
// Output:
// [Compression] Compressing data...
// [Encryption] Encrypting data...
// Writing to file: data.txt
```

## Real-World Use Cases

- **Java I/O streams** — `BufferedInputStream(new FileInputStream(...))`, `GZIPOutputStream(new FileOutputStream(...))`
- **Spring Security filter chain** — each filter decorates the next
- **Java Collections** — `Collections.synchronizedList()`, `Collections.unmodifiableList()`
- **Servlet filters** — each filter wraps the request/response

## When to Use

- When you need to add behaviour to individual objects without affecting others
- When extension by subclassing is impractical
- When you want to combine behaviours by stacking wrappers

---

| | |
|---|---|
| [← Composite](./08-composite.md) | [Next → Facade](./10-facade.md) |

---
---

