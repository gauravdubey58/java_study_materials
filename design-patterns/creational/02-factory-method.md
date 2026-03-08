[← Singleton](./01-singleton.md) | [Index](../index.md)

---

# Pattern 2 — Factory Method
**Category:** Creational

---

## Intent

Define an interface for creating an object, but **let subclasses decide** which class to instantiate. Factory Method lets a class defer instantiation to subclasses.

---

## The Problem It Solves

Imagine a logistics application that initially only handles truck deliveries. The code is full of `new Truck()` calls. Now you need to add sea shipping. Changing all those `new Truck()` calls and adding `if/else` everywhere would be messy and violate the Open/Closed Principle.

Factory Method solves this by replacing direct constructor calls with a factory method that subclasses override to return different types of objects.

---

## The Solution

```
┌──────────────────────┐         ┌────────────────────┐
│   Creator (abstract) │         │   Product           │
├──────────────────────┤         │   (interface)       │
│ + createProduct()    │────────▶│ + deliver()         │
│ + someOperation()    │         └────────────────────┘
└──────────────────────┘                  ▲
          ▲                               │
          │                    ┌──────────┴──────────┐
┌─────────┴──────────┐    ┌────┴──────┐      ┌───────┴──────┐
│ TruckCreator       │    │  Truck    │      │  Ship        │
├────────────────────┤    └───────────┘      └──────────────┘
│ + createProduct()  │
└────────────────────┘
          │
┌─────────┴──────────┐
│ ShipCreator        │
├────────────────────┤
│ + createProduct()  │
└────────────────────┘
```

---

## Java Implementation

```java
// Product interface
public interface Transport {
    void deliver(String cargo, String destination);
    String getType();
}

// Concrete Products
public class Truck implements Transport {
    @Override
    public void deliver(String cargo, String destination) {
        System.out.printf("Truck delivering '%s' to %s by road%n", cargo, destination);
    }
    @Override
    public String getType() { return "Truck"; }
}

public class Ship implements Transport {
    @Override
    public void deliver(String cargo, String destination) {
        System.out.printf("Ship delivering '%s' to %s by sea%n", cargo, destination);
    }
    @Override
    public String getType() { return "Ship"; }
}

public class Drone implements Transport {
    @Override
    public void deliver(String cargo, String destination) {
        System.out.printf("Drone delivering '%s' to %s by air%n", cargo, destination);
    }
    @Override
    public String getType() { return "Drone"; }
}

// Creator (abstract class with factory method)
public abstract class Logistics {

    // THE FACTORY METHOD — subclasses must implement this
    protected abstract Transport createTransport();

    // Template that uses the factory method
    public void planDelivery(String cargo, String destination) {
        Transport transport = createTransport();
        System.out.println("Planning delivery using: " + transport.getType());
        transport.deliver(cargo, destination);
    }
}

// Concrete Creators
public class RoadLogistics extends Logistics {
    @Override
    protected Transport createTransport() {
        return new Truck();
    }
}

public class SeaLogistics extends Logistics {
    @Override
    protected Transport createTransport() {
        return new Ship();
    }
}

public class AirLogistics extends Logistics {
    @Override
    protected Transport createTransport() {
        return new Drone();
    }
}

// Client code
public class Main {
    public static void main(String[] args) {
        Logistics logistics;

        String mode = System.getenv("SHIPPING_MODE");
        switch (mode) {
            case "sea"  -> logistics = new SeaLogistics();
            case "air"  -> logistics = new AirLogistics();
            default     -> logistics = new RoadLogistics();
        }

        // Client code is identical regardless of transport type
        logistics.planDelivery("Electronics", "New York");
    }
}
```

### Simpler Static Factory Variant

A common variation is a static factory method that selects the product based on a parameter:

```java
public class TransportFactory {
    public static Transport create(String type) {
        return switch (type.toLowerCase()) {
            case "truck" -> new Truck();
            case "ship"  -> new Ship();
            case "drone" -> new Drone();
            default -> throw new IllegalArgumentException("Unknown transport: " + type);
        };
    }
}

// Usage
Transport t = TransportFactory.create("ship");
t.deliver("Cargo", "Rotterdam");
```

---

## Real-World Use Cases

**Java Standard Library**
- `Calendar.getInstance()` — returns a locale-appropriate Calendar subclass
- `NumberFormat.getInstance()` — returns locale-specific number formatter
- `Collection.iterator()` — each collection returns its own iterator implementation

**JDBC**
- `DriverManager.getConnection()` — returns a connection for the registered driver
- `Connection.createStatement()` — returns a database-specific Statement

**Spring Framework**
- `BeanFactory.getBean()` — returns different bean implementations based on configuration
- `SessionFactory.openSession()` — factory for Hibernate sessions

---

## When to Use

- When you don't know ahead of time what type of object you need to create
- When you want subclasses to specify the objects they create
- When creating an object requires complex logic that should live in one place
- When you want to provide users of your library with a way to extend its internal components

## When to Avoid

- When there are only a few product types and they rarely change (simple `if/else` or `switch` may be cleaner)
- When the factory itself becomes excessively complex

---

| | |
|---|---|
| [← Singleton](./01-singleton.md) | [Next → Abstract Factory](./03-abstract-factory.md) |
