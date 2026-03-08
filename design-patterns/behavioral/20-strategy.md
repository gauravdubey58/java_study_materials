# Pattern 20 — Strategy
**Category:** Behavioral

---

## Intent

Define a **family of algorithms**, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from the clients that use it.

---

## The Problem It Solves

An e-commerce site needs different sorting algorithms for products, multiple payment methods, and interchangeable discount strategies. Hardcoding all options with if/else violates Open/Closed Principle — adding a new strategy requires modifying existing code.

---

## Java Implementation

```java
// Strategy interface
public interface SortStrategy<T> {
    void sort(List<T> list, Comparator<T> comparator);
    String getName();
}

// Concrete Strategies
public class BubbleSortStrategy<T> implements SortStrategy<T> {
    @Override
    public void sort(List<T> list, Comparator<T> comp) {
        // ... bubble sort implementation
        System.out.println("Bubble sorting " + list.size() + " elements");
    }
    @Override public String getName() { return "BubbleSort"; }
}

public class QuickSortStrategy<T> implements SortStrategy<T> {
    @Override
    public void sort(List<T> list, Comparator<T> comp) {
        list.sort(comp);   // Java's TimSort (hybrid QuickSort)
        System.out.println("Quick sorting " + list.size() + " elements");
    }
    @Override public String getName() { return "QuickSort"; }
}

// Context — uses a strategy
public class ProductCatalog {
    private SortStrategy<Product> sortStrategy;

    public ProductCatalog(SortStrategy<Product> strategy) {
        this.sortStrategy = strategy;
    }

    // Strategy can be changed at runtime
    public void setSortStrategy(SortStrategy<Product> strategy) {
        this.sortStrategy = strategy;
    }

    public List<Product> getSortedProducts(List<Product> products, Comparator<Product> comp) {
        List<Product> sorted = new ArrayList<>(products);
        System.out.println("Sorting using: " + sortStrategy.getName());
        sortStrategy.sort(sorted, comp);
        return sorted;
    }
}

// --- Payment Strategy Example (more real-world) ---

public interface PaymentStrategy {
    PaymentResult pay(BigDecimal amount);
    String getMethodName();
}

public class CreditCardPayment implements PaymentStrategy {
    private final String cardNumber;
    public CreditCardPayment(String cardNumber) { this.cardNumber = cardNumber; }
    @Override
    public PaymentResult pay(BigDecimal amount) {
        System.out.printf("Charging $%.2f to card ending in %s%n",
                amount, cardNumber.substring(cardNumber.length() - 4));
        return PaymentResult.success();
    }
    @Override public String getMethodName() { return "CreditCard"; }
}

public class PayPalPayment implements PaymentStrategy {
    private final String email;
    public PayPalPayment(String email) { this.email = email; }
    @Override
    public PaymentResult pay(BigDecimal amount) {
        System.out.printf("Processing $%.2f via PayPal account %s%n", amount, email);
        return PaymentResult.success();
    }
    @Override public String getMethodName() { return "PayPal"; }
}

// Usage — switch strategies at runtime
ShoppingCart cart = new ShoppingCart();
cart.setPaymentStrategy(new CreditCardPayment("4111111111111234"));
cart.checkout();  // uses credit card

cart.setPaymentStrategy(new PayPalPayment("user@example.com"));
cart.checkout();  // now uses PayPal
```

---

## Real-World Use Cases

- **`java.util.Comparator`** — a Strategy for comparison
- **Spring's `ResourceLoader`** — different strategies for loading resources
- **Hibernate's dialect** — different SQL generation strategies per database
- **Spring Security authentication** — pluggable authentication strategies
- **Java's `ThreadPoolExecutor`** — pluggable rejection policies (`AbortPolicy`, `CallerRunsPolicy`)

## When to Use

- When you want to define a class that will have one behaviour that's similar to other behaviours in a list
- When you need different variants of an algorithm
- When you want to switch algorithms used at runtime
- When there are many classes that differ only in their behaviour

---

| | |
|---|---|
| [← State](./19-state.md) | [Next → Template Method](./21-template-method.md) |

---
---

