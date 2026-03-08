# Design Patterns — Complete Reference Notes

> A structured, book-style guide to the 23 classic Gang of Four design patterns, with real-world use cases, Java code examples, diagrams, and 50 interview questions with answers.

---

## What Are Design Patterns?

Design patterns are **reusable solutions to commonly occurring problems** in software design. They are not finished code you can copy-paste — they are templates and blueprints that describe how to solve a problem in a way that has been proven to work.

The term was popularised by the **Gang of Four (GoF)** — Erich Gamma, Richard Helm, Ralph Johnson, and John Vlissides — in their landmark 1994 book *Design Patterns: Elements of Reusable Object-Oriented Software*.

---

## The Three Categories

| Category | Purpose | Patterns |
|---|---|---|
| **Creational** | How objects are created | Singleton, Factory Method, Abstract Factory, Builder, Prototype |
| **Structural** | How objects are composed | Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy |
| **Behavioral** | How objects communicate | Chain of Responsibility, Command, Iterator, Mediator, Memento, Observer, State, Strategy, Template Method, Visitor, Interpreter |

---

## Repository Structure

```
design-patterns-notes/
├── README.md                        ← You are here
├── index.md                         ← Full table of contents
│
├── creational/
│   ├── 01-singleton.md
│   ├── 02-factory-method.md
│   ├── 03-abstract-factory.md
│   ├── 04-builder.md
│   └── 05-prototype.md
│
├── structural/
│   ├── 06-adapter.md
│   ├── 07-bridge.md
│   ├── 08-composite.md
│   ├── 09-decorator.md
│   ├── 10-facade.md
│   ├── 11-flyweight.md
│   └── 12-proxy.md
│
└── behavioral/
    ├── 13-chain-of-responsibility.md
    ├── 14-command.md
    ├── 15-iterator.md
    ├── 16-mediator.md
    ├── 17-memento.md
    ├── 18-observer.md
    ├── 19-state.md
    ├── 20-strategy.md
    ├── 21-template-method.md
    ├── 22-visitor.md
    ├── 23-interpreter.md
    └── 24-interview-questions.md
```

---

## How to Use These Notes

- Start at **[index.md](./index.md)** for the full table of contents
- Each pattern page follows the same structure: Intent → Problem → Solution → Java Example → Real-World Use Cases → When to Use / Avoid
- End with **[50 Interview Questions](./behavioral/24-interview-questions.md)** to test your knowledge

---

## Key Principles Behind Patterns

Before diving into patterns, these **SOLID principles** underpin most of them:

**S — Single Responsibility**: A class should have only one reason to change.  
**O — Open/Closed**: Open for extension, closed for modification.  
**L — Liskov Substitution**: Subtypes must be substitutable for their base types.  
**I — Interface Segregation**: Clients should not depend on interfaces they don't use.  
**D — Dependency Inversion**: Depend on abstractions, not concretions.

---

*Gang of Four book: Design Patterns — Elements of Reusable Object-Oriented Software (1994)*  
*Authors: Erich Gamma, Richard Helm, Ralph Johnson, John Vlissides*
