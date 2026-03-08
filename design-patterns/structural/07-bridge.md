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

