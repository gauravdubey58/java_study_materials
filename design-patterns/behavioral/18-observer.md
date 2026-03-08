# Pattern 18 — Observer
**Category:** Behavioral

---

## Intent

Define a **one-to-many dependency** between objects so that when one object changes state, all its dependents are notified and updated automatically.

---

## The Problem It Solves

A stock price tracker must notify multiple displays (mobile app, website, email alerts) when a stock price changes. The price source shouldn't know about or depend on the displays — they might be added or removed at runtime.

---

## Java Implementation

```java
// Observer interface
public interface StockObserver {
    void onPriceChanged(String symbol, BigDecimal oldPrice, BigDecimal newPrice);
}

// Subject (Observable)
public class StockMarket {
    private final Map<String, BigDecimal>       prices    = new HashMap<>();
    private final List<StockObserver>           observers = new ArrayList<>();

    public void addObserver(StockObserver observer)    { observers.add(observer); }
    public void removeObserver(StockObserver observer) { observers.remove(observer); }

    public void updatePrice(String symbol, BigDecimal newPrice) {
        BigDecimal oldPrice = prices.getOrDefault(symbol, BigDecimal.ZERO);
        prices.put(symbol, newPrice);

        if (!oldPrice.equals(newPrice)) {
            notifyObservers(symbol, oldPrice, newPrice);
        }
    }

    private void notifyObservers(String symbol, BigDecimal oldPrice, BigDecimal newPrice) {
        observers.forEach(o -> o.onPriceChanged(symbol, oldPrice, newPrice));
    }
}

// Concrete Observers
public class MobileApp implements StockObserver {
    @Override
    public void onPriceChanged(String symbol, BigDecimal old, BigDecimal now) {
        System.out.printf("[Mobile] %s: $%.2f → $%.2f%n", symbol, old, now);
    }
}

public class EmailAlertService implements StockObserver {
    private final BigDecimal threshold;
    private final String recipientEmail;

    public EmailAlertService(BigDecimal threshold, String email) {
        this.threshold = threshold;
        this.recipientEmail = email;
    }

    @Override
    public void onPriceChanged(String symbol, BigDecimal old, BigDecimal now) {
        BigDecimal change = now.subtract(old).abs().divide(old, 4, RoundingMode.HALF_UP);
        if (change.compareTo(threshold) > 0) {
            System.out.printf("[Email to %s] ALERT: %s moved %.2f%%%n",
                    recipientEmail, symbol, change.multiply(BigDecimal.valueOf(100)));
        }
    }
}

// Usage
StockMarket market = new StockMarket();
market.addObserver(new MobileApp());
market.addObserver(new EmailAlertService(new BigDecimal("0.05"), "trader@example.com"));

market.updatePrice("AAPL", new BigDecimal("150.00"));
market.updatePrice("AAPL", new BigDecimal("165.00"));  // 10% change — triggers email alert
```

---

## Real-World Use Cases

- **Java's `java.util.Observable`** (deprecated, replaced by modern patterns)
- **JavaFX/Swing event listeners** — ActionListener, ChangeListener
- **Spring's ApplicationEvent / ApplicationEventPublisher**
- **RxJava / Project Reactor** — reactive streams are Observer pattern at scale
- **MVC pattern** — View observes the Model

## When to Use

- When a change in one object requires updating others, and you don't know how many
- When objects should be able to notify other objects without assumptions about them
- When you want to implement distributed event handling

---

| | |
|---|---|
| [← Memento](./17-memento.md) | [Next → State](./19-state.md) |

---
---

