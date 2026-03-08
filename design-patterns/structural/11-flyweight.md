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

