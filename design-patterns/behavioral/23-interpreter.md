# Pattern 23 — Interpreter
**Category:** Behavioral

---

## Intent

Given a language, define a representation for its grammar and provide an **interpreter** that uses the representation to interpret sentences in the language.

---

## The Problem It Solves

You need to evaluate mathematical expressions, parse SQL-like query syntax, or process domain-specific rule expressions repeatedly. Rather than hardcoding a parser everywhere, Interpreter creates a grammar of expression objects.

---

## Java Implementation

```java
// Abstract Expression
public interface Expression {
    int interpret(Map<String, Integer> context);
}

// Terminal Expressions (leaves)
public class NumberExpression implements Expression {
    private final int number;
    public NumberExpression(int number) { this.number = number; }
    @Override
    public int interpret(Map<String, Integer> context) { return number; }
}

public class VariableExpression implements Expression {
    private final String name;
    public VariableExpression(String name) { this.name = name; }
    @Override
    public int interpret(Map<String, Integer> context) {
        return context.getOrDefault(name, 0);
    }
}

// Non-Terminal Expressions (composites)
public class AddExpression implements Expression {
    private final Expression left, right;
    public AddExpression(Expression left, Expression right) {
        this.left = left; this.right = right;
    }
    @Override
    public int interpret(Map<String, Integer> context) {
        return left.interpret(context) + right.interpret(context);
    }
}

public class MultiplyExpression implements Expression {
    private final Expression left, right;
    public MultiplyExpression(Expression left, Expression right) {
        this.left = left; this.right = right;
    }
    @Override
    public int interpret(Map<String, Integer> context) {
        return left.interpret(context) * right.interpret(context);
    }
}

// Usage — build and evaluate: (a + 5) * (b + 2)
Map<String, Integer> context = Map.of("a", 3, "b", 4);

Expression expr = new MultiplyExpression(
    new AddExpression(new VariableExpression("a"), new NumberExpression(5)),
    new AddExpression(new VariableExpression("b"), new NumberExpression(2))
);

System.out.println("(a+5)*(b+2) = " + expr.interpret(context));  // (3+5)*(4+2) = 48
```

---

## Real-World Use Cases

- **Java's `java.util.regex.Pattern`** — interprets regular expression grammar
- **Spring Expression Language (SpEL)** — `#{user.name}` expressions in config
- **Hibernate Query Language (HQL)** — interprets HQL grammar
- **SQL parsers** — interpret SQL query grammar
- **Business rule engines (Drools)** — interpret rule expressions

## When to Use

- When you need to interpret sentences in a simple language or grammar
- When the grammar is simple (complex grammars need proper parsers like ANTLR)
- When efficiency is not a critical concern

## When to Avoid

- When the grammar is complex — the class hierarchy becomes too large
- For performance-critical parsing — compiled or generated parsers are much faster

---

| | |
|---|---|
| [← Visitor](./22-visitor.md) | [Next → 50 Interview Questions](./24-interview-questions.md) |
