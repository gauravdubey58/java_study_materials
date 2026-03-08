[← Prototype](../creational/05-prototype.md) | [Index](../index.md)

---

# Pattern 6 — Adapter
**Category:** Structural

---

## Intent

Convert the interface of a class into **another interface that clients expect**. Adapter lets classes work together that couldn't otherwise because of incompatible interfaces.

---

## The Problem It Solves

You have a third-party library or legacy class with a useful interface, but it doesn't match what your system expects. You can't modify the third-party code. Adapter wraps it to make it compatible — just like a power socket adapter lets a US plug work in a European outlet.

---

## The Solution

```
┌──────────┐       ┌───────────┐       ┌─────────────────┐
│  Client  │──────▶│  Target   │       │ Adaptee         │
│          │       │(interface)│       │ (existing class) │
└──────────┘       └───────────┘       │+ specificMethod()│
                         ▲             └────────┬────────┘
                         │                      │ wraps
                   ┌─────┴─────┐               │
                   │  Adapter  │───────────────┘
                   │+method()  │
                   └───────────┘
```

---

## Java Implementation

```java
// Target interface — what our system expects
public interface MediaPlayer {
    void play(String audioType, String fileName);
}

// Adaptee — existing third-party class with incompatible interface
public class AdvancedMediaPlayer {
    public void playMp4(String fileName) {
        System.out.println("Playing MP4: " + fileName);
    }
    public void playVlc(String fileName) {
        System.out.println("Playing VLC: " + fileName);
    }
    public void playAvi(String fileName) {
        System.out.println("Playing AVI: " + fileName);
    }
}

// Adapter — wraps Adaptee, implements Target
public class MediaAdapter implements MediaPlayer {

    private final AdvancedMediaPlayer advancedPlayer;

    public MediaAdapter() {
        this.advancedPlayer = new AdvancedMediaPlayer();
    }

    @Override
    public void play(String audioType, String fileName) {
        switch (audioType.toLowerCase()) {
            case "mp4" -> advancedPlayer.playMp4(fileName);
            case "vlc" -> advancedPlayer.playVlc(fileName);
            case "avi" -> advancedPlayer.playAvi(fileName);
            default    -> System.out.println("Unsupported format: " + audioType);
        }
    }
}

// Client uses only the Target interface
public class AudioPlayer implements MediaPlayer {
    private final MediaAdapter adapter = new MediaAdapter();

    @Override
    public void play(String audioType, String fileName) {
        if (audioType.equalsIgnoreCase("mp3")) {
            System.out.println("Playing MP3: " + fileName);
        } else {
            adapter.play(audioType, fileName);   // delegate to adapter
        }
    }
}

// Usage
AudioPlayer player = new AudioPlayer();
player.play("mp3", "song.mp3");
player.play("mp4", "video.mp4");
player.play("vlc", "movie.vlc");
```

### Real-World Java Example — Legacy Payment Gateway

```java
// Your system's expected interface
public interface PaymentProcessor {
    PaymentResult charge(String customerId, BigDecimal amount, String currency);
}

// Legacy bank API (cannot modify)
public class LegacyBankApi {
    public int processTransaction(String accountNo, double amountInCents, String curr) {
        // returns 0 for success, error code otherwise
        return 0;
    }
}

// Adapter bridges the two
public class LegacyBankAdapter implements PaymentProcessor {
    private final LegacyBankApi bankApi;

    public LegacyBankAdapter(LegacyBankApi bankApi) {
        this.bankApi = bankApi;
    }

    @Override
    public PaymentResult charge(String customerId, BigDecimal amount, String currency) {
        double cents = amount.multiply(BigDecimal.valueOf(100)).doubleValue();
        int result = bankApi.processTransaction(customerId, cents, currency);
        return result == 0
            ? PaymentResult.success()
            : PaymentResult.failure("Bank error code: " + result);
    }
}
```

---

## Real-World Use Cases

- **Java I/O** — `InputStreamReader` adapts `InputStream` (bytes) to `Reader` (characters)
- **SLF4J** — adapts various logging frameworks (Log4j, JUL) to a common interface
- **Spring MVC** — `HandlerAdapter` adapts different handler types to the dispatcher
- **Arrays.asList()** — adapts an array to the `List` interface

## When to Use

- When you want to use an existing class but its interface doesn't match what you need
- When you're integrating third-party or legacy libraries
- When you want to create a reusable class that works with classes that don't necessarily have compatible interfaces

---

| | |
|---|---|
| [← Prototype](../creational/05-prototype.md) | [Next → Bridge](./07-bridge.md) |

---
---

# Pattern 7 — Bridge
**Category:** Structural

---

## Intent

**Decouple an abstraction from its implementation** so that the two can vary independently.

---

## The Problem It Solves

Without Bridge, adding a new shape AND a new rendering method would require adding multiple new classes (a multiplicative explosion). Bridge separates the two hierarchies so they grow independently.

Without Bridge: `RedCircle`, `BlueCircle`, `RedSquare`, `BlueSquare` — 4 classes for 2 shapes × 2 colours.  
With Bridge: `Circle(RedPaint)`, `Circle(BluePaint)`, `Square(RedPaint)` — 2 shapes + 2 colours = 4 combinations from just 4 classes.

---

## Java Implementation

```java
// Implementation interface (the "bridge")
public interface Renderer {
    void renderCircle(double radius);
    void renderRectangle(double width, double height);
}

// Concrete Implementations
public class VectorRenderer implements Renderer {
    @Override
    public void renderCircle(double radius) {
        System.out.printf("Drawing circle (radius=%.1f) as vector%n", radius);
    }
    @Override
    public void renderRectangle(double w, double h) {
        System.out.printf("Drawing rectangle (%.1fx%.1f) as vector%n", w, h);
    }
}

public class RasterRenderer implements Renderer {
    @Override
    public void renderCircle(double radius) {
        System.out.printf("Drawing circle (radius=%.1f) as pixels%n", radius);
    }
    @Override
    public void renderRectangle(double w, double h) {
        System.out.printf("Drawing rectangle (%.1fx%.1f) as pixels%n", w, h);
    }
}

// Abstraction — has a reference to the implementation
public abstract class Shape {
    protected final Renderer renderer;   // THE BRIDGE

    public Shape(Renderer renderer) {
        this.renderer = renderer;
    }

    public abstract void draw();
    public abstract void resize(double factor);
}

// Refined Abstractions
public class Circle extends Shape {
    private double radius;

    public Circle(Renderer renderer, double radius) {
        super(renderer);
        this.radius = radius;
    }

    @Override
    public void draw() {
        renderer.renderCircle(radius);
    }

    @Override
    public void resize(double factor) {
        radius *= factor;
    }
}

public class Rectangle extends Shape {
    private double width, height;

    public Rectangle(Renderer renderer, double width, double height) {
        super(renderer);
        this.width = width;
        this.height = height;
    }

    @Override
    public void draw() {
        renderer.renderRectangle(width, height);
    }

    @Override
    public void resize(double factor) {
        width *= factor;
        height *= factor;
    }
}

// Usage — mix and match freely
Shape vectorCircle = new Circle(new VectorRenderer(), 5.0);
Shape rasterCircle = new Circle(new RasterRenderer(), 5.0);
Shape vectorRect   = new Rectangle(new VectorRenderer(), 10, 6);

vectorCircle.draw();
rasterCircle.draw();
vectorRect.draw();
```

## Real-World Use Cases

- **JDBC** — `DriverManager` is the abstraction; individual JDBC drivers are implementations
- **Java logging** — `Logger` abstraction bridges to `Handler` implementations (FileHandler, ConsoleHandler)
- **Spring's PlatformTransactionManager** — abstraction over different transaction implementations (JPA, JDBC, JTA)

## When to Use

- When you want to avoid a permanent binding between abstraction and implementation
- When both abstractions and implementations should be extensible via subclassing
- When changes in implementation should not affect clients

---

| | |
|---|---|
| [← Adapter](./06-adapter.md) | [Next → Composite](./08-composite.md) |

---
---

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

# Pattern 10 — Facade
**Category:** Structural

---

## Intent

Provide a **simplified interface to a complex subsystem**. A Facade hides the complexity of the subsystem and provides a simple, high-level interface.

---

## The Problem It Solves

An e-commerce order placement involves: inventory checking, payment processing, fraud detection, shipping calculation, notification sending, and order persistence. Clients shouldn't need to know how all these systems interact. Facade provides a single `placeOrder()` method.

---

## Java Implementation

```java
// Complex subsystem classes
public class InventoryService {
    public boolean isAvailable(String productId, int qty) {
        System.out.println("Checking inventory for " + productId);
        return true;
    }
    public void reserve(String productId, int qty) {
        System.out.println("Reserving " + qty + " units of " + productId);
    }
}

public class PaymentService {
    public boolean processPayment(String customerId, BigDecimal amount) {
        System.out.println("Processing payment of $" + amount + " for " + customerId);
        return true;
    }
}

public class ShippingService {
    public String createShipment(String orderId, String address) {
        System.out.println("Creating shipment for order " + orderId);
        return "TRACK-" + orderId;
    }
}

public class NotificationService {
    public void sendOrderConfirmation(String email, String orderId) {
        System.out.println("Sending confirmation email to " + email);
    }
}

// FACADE — simple interface over the complex subsystem
public class OrderFacade {
    private final InventoryService   inventory;
    private final PaymentService     payment;
    private final ShippingService    shipping;
    private final NotificationService notification;

    public OrderFacade() {
        this.inventory    = new InventoryService();
        this.payment      = new PaymentService();
        this.shipping     = new ShippingService();
        this.notification = new NotificationService();
    }

    // One simple method hides all the complexity
    public OrderResult placeOrder(String customerId, String productId,
                                   int qty, BigDecimal price, String email, String address) {
        // 1. Check inventory
        if (!inventory.isAvailable(productId, qty)) {
            return OrderResult.failure("Product not available");
        }

        // 2. Process payment
        if (!payment.processPayment(customerId, price)) {
            return OrderResult.failure("Payment failed");
        }

        // 3. Reserve stock
        inventory.reserve(productId, qty);

        // 4. Create shipment
        String orderId    = UUID.randomUUID().toString().substring(0, 8);
        String trackingNo = shipping.createShipment(orderId, address);

        // 5. Notify customer
        notification.sendOrderConfirmation(email, orderId);

        return OrderResult.success(orderId, trackingNo);
    }
}

// Client code — beautifully simple
OrderFacade orderService = new OrderFacade();
OrderResult result = orderService.placeOrder(
        "CUST-001", "PROD-JAVA-BOOK", 1,
        BigDecimal.valueOf(49.99), "user@example.com", "123 Main St");
```

## Real-World Use Cases

- **SLF4J** — simple logging facade over multiple frameworks
- **Spring's JdbcTemplate** — facade over raw JDBC complexity
- **Hibernate's Session** — facade over SQL, caching, lazy loading
- **javax.faces.context.FacesContext** — facade over JSF infrastructure

## When to Use

- When you want to provide a simple interface to a complex subsystem
- When there are many dependencies between clients and implementation classes
- When you want to layer your subsystems

---

| | |
|---|---|
| [← Decorator](./09-decorator.md) | [Next → Flyweight](./11-flyweight.md) |

---
---

# Pattern 11 — Flyweight
**Category:** Structural

---

## Intent

Use **sharing to efficiently support a large number of fine-grained objects**. Flyweight is an optimisation pattern — it reduces memory usage by sharing common state.

---

## The Problem It Solves

A text editor displays millions of characters. Each Character object with its own font, size, colour, and glyph data would consume enormous memory. Flyweight separates shared (intrinsic) state from unique (extrinsic) state and shares the intrinsic part.

---

## Intrinsic vs Extrinsic State

**Intrinsic state** — shared, context-independent (stored in the Flyweight)  
**Extrinsic state** — unique, context-dependent (passed in by the client)

---

## Java Implementation

```java
// Flyweight — contains intrinsic (shared) state only
public class CharacterStyle {
    private final String fontName;
    private final int    fontSize;
    private final String color;
    private final char   glyph;

    public CharacterStyle(char glyph, String fontName, int fontSize, String color) {
        this.glyph    = glyph;
        this.fontName = fontName;
        this.fontSize = fontSize;
        this.color    = color;
        System.out.println("Creating flyweight for: " + glyph + " (" + fontName + ")");
    }

    // Extrinsic state (position) is passed in at render time
    public void render(int x, int y) {
        System.out.printf("Rendering '%c' at (%d,%d) [%s, %dpx, %s]%n",
                glyph, x, y, fontName, fontSize, color);
    }
}

// Flyweight Factory — cache and reuse flyweights
public class CharacterStyleFactory {
    private static final Map<String, CharacterStyle> cache = new HashMap<>();

    public static CharacterStyle getStyle(char glyph, String font, int size, String color) {
        String key = glyph + "_" + font + "_" + size + "_" + color;
        return cache.computeIfAbsent(key,
                k -> new CharacterStyle(glyph, font, size, color));
    }

    public static int getCacheSize() { return cache.size(); }
}

// Usage — millions of characters, but only a handful of flyweight objects
List<int[]> positions = new ArrayList<>();  // extrinsic state

// Simulate placing 1,000,000 characters
for (int i = 0; i < 1_000_000; i++) {
    char c = (char) ('a' + i % 26);
    CharacterStyle style = CharacterStyleFactory.getStyle(c, "Arial", 12, "black");
    // Store only position + reference to shared flyweight
}

System.out.println("Flyweight objects created: " + CharacterStyleFactory.getCacheSize());
// Only 26 objects for 1,000,000 characters!
```

## Real-World Use Cases

- **Java's String Pool** — string literals are interned and shared
- **Integer cache** — `Integer.valueOf(-128 to 127)` returns cached instances
- **Java's Boolean.TRUE / Boolean.FALSE** — shared constants
- **Game development** — reuse tree/grass/rock objects, vary only position

## When to Use

- When you need to create a very large number of similar objects
- When memory usage is a concern
- When most object state can be made extrinsic

---

| | |
|---|---|
| [← Facade](./10-facade.md) | [Next → Proxy](./12-proxy.md) |

---
---

# Pattern 12 — Proxy
**Category:** Structural

---

## Intent

Provide a **surrogate or placeholder** for another object to control access to it.

---

## Types of Proxy

| Type | Purpose |
|---|---|
| **Virtual Proxy** | Lazy-load expensive objects only when needed |
| **Protection Proxy** | Control access based on permissions |
| **Remote Proxy** | Represent an object in a different address space |
| **Caching Proxy** | Cache results of expensive operations |
| **Logging Proxy** | Log all calls to the real subject |

---

## Java Implementation

```java
// Subject interface
public interface ImageService {
    Image loadImage(String url);
    void  displayImage(String url);
}

// Real Subject — expensive operation
public class RealImageService implements ImageService {
    @Override
    public Image loadImage(String url) {
        System.out.println("Loading image from network: " + url);
        // ... expensive network call ...
        return new Image(url, new byte[1024]);
    }

    @Override
    public void displayImage(String url) {
        Image img = loadImage(url);
        System.out.println("Displaying: " + url);
    }
}

// Virtual + Caching Proxy
public class ImageServiceProxy implements ImageService {
    private final RealImageService realService = new RealImageService();
    private final Map<String, Image> cache = new HashMap<>();

    @Override
    public Image loadImage(String url) {
        if (cache.containsKey(url)) {
            System.out.println("Cache hit for: " + url);
            return cache.get(url);
        }
        Image img = realService.loadImage(url);
        cache.put(url, img);
        return img;
    }

    @Override
    public void displayImage(String url) {
        Image img = loadImage(url);  // uses cache
        System.out.println("Displaying: " + url);
    }
}

// Protection Proxy
public class SecureImageServiceProxy implements ImageService {
    private final RealImageService realService = new RealImageService();
    private final User currentUser;

    public SecureImageServiceProxy(User user) {
        this.currentUser = user;
    }

    @Override
    public Image loadImage(String url) {
        if (!currentUser.hasPermission("IMAGE_READ")) {
            throw new SecurityException("Access denied for user: " + currentUser.getName());
        }
        return realService.loadImage(url);
    }

    @Override
    public void displayImage(String url) {
        loadImage(url);
    }
}
```

## Real-World Use Cases

- **Spring AOP** — Spring creates JDK dynamic proxies or CGLIB proxies for `@Transactional`, `@Cacheable`, `@Async`
- **Hibernate Lazy Loading** — `getOrders()` on a User returns a proxy; SQL runs only when you iterate
- **Java RMI** — remote stubs are proxies for remote objects
- **Java's Proxy class** — `java.lang.reflect.Proxy` for dynamic proxy creation

## When to Use

- When you need lazy initialisation of a heavyweight object
- When you need access control to an object
- When you need to log calls, cache results, or add cross-cutting concerns

---

| | |
|---|---|
| [← Flyweight](./11-flyweight.md) | [Next → Chain of Responsibility](../behavioral/13-chain-of-responsibility.md) |
