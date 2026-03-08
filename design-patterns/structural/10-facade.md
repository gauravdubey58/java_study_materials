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

