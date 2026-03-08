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
