# Pattern 8 — Composite
**Category:** Structural

---

## Intent

Compose objects into **tree structures** to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions of objects uniformly.

---

## The Problem It Solves

You have a file system with files and folders. Folders can contain files or more folders. You want to call `getSize()` on any node and have it work correctly whether it's a file (returns its own size) or a folder (returns the sum of all children). Composite makes this work transparently.

---

## Java Implementation

```java
// Component — common interface for both leaf and composite
public interface FileSystemComponent {
    String getName();
    long getSize();
    void print(String indent);
}

// Leaf — no children
public class File implements FileSystemComponent {
    private final String name;
    private final long   size;

    public File(String name, long size) {
        this.name = name;
        this.size = size;
    }

    @Override public String getName() { return name; }
    @Override public long   getSize() { return size; }

    @Override
    public void print(String indent) {
        System.out.printf("%s📄 %s (%d bytes)%n", indent, name, size);
    }
}

// Composite — can hold children (files or other folders)
public class Directory implements FileSystemComponent {
    private final String name;
    private final List<FileSystemComponent> children = new ArrayList<>();

    public Directory(String name) { this.name = name; }

    public void add(FileSystemComponent component)    { children.add(component); }
    public void remove(FileSystemComponent component) { children.remove(component); }

    @Override public String getName() { return name; }

    @Override
    public long getSize() {
        // Recursively sum all children — works for any depth
        return children.stream().mapToLong(FileSystemComponent::getSize).sum();
    }

    @Override
    public void print(String indent) {
        System.out.printf("%s📁 %s (%d bytes total)%n", indent, name, getSize());
        children.forEach(c -> c.print(indent + "  "));
    }
}

// Usage
Directory root = new Directory("root");
root.add(new File("readme.txt", 1024));

Directory src = new Directory("src");
src.add(new File("Main.java", 4096));
src.add(new File("App.java", 2048));
root.add(src);

Directory lib = new Directory("lib");
lib.add(new File("spring-core.jar", 2_097_152));
root.add(lib);

root.print("");
System.out.println("Total: " + root.getSize() + " bytes");
```

## Real-World Use Cases

- **Java AWT/Swing** — `Container` (composite) holds `Component` (leaf or composite)
- **HTML/XML DOM** — nodes can be elements (composite) or text nodes (leaf)
- **Spring Security** — `CompositeFilter` chains filters
- **JUnit** — `TestSuite` (composite) contains `TestCase` objects (leaves)

## When to Use

- When you want clients to treat individual objects and compositions uniformly
- When you need to represent part-whole hierarchies (trees)

---

| | |
|---|---|
| [← Bridge](./07-bridge.md) | [Next → Decorator](./09-decorator.md) |

---
---

