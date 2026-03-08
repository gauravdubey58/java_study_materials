# Index — Design Patterns Notes

> Complete table of contents. Every pattern follows the same structure: Intent → Problem → Solution → Java Code → Real-World Use Cases → When to Use / Avoid.

---

## 🏗️ Creational Patterns
*Concerned with the process of object creation.*

| # | Pattern | One-Line Summary |
|---|---|---|
| 1 | [Singleton](./creational/01-singleton.md) | Ensure a class has only one instance, and provide a global access point to it |
| 2 | [Factory Method](./creational/02-factory-method.md) | Define an interface for creating an object, but let subclasses decide which class to instantiate |
| 3 | [Abstract Factory](./creational/03-abstract-factory.md) | Provide an interface for creating families of related objects without specifying concrete classes |
| 4 | [Builder](./creational/04-builder.md) | Separate the construction of a complex object from its representation |
| 5 | [Prototype](./creational/05-prototype.md) | Create new objects by copying an existing object |

---

## 🔩 Structural Patterns
*Concerned with how classes and objects are composed to form larger structures.*

| # | Pattern | One-Line Summary |
|---|---|---|
| 6 | [Adapter](./structural/06-adapter.md) | Convert the interface of a class into another interface that clients expect |
| 7 | [Bridge](./structural/07-bridge.md) | Decouple an abstraction from its implementation so the two can vary independently |
| 8 | [Composite](./structural/08-composite.md) | Compose objects into tree structures to represent part-whole hierarchies |
| 9 | [Decorator](./structural/09-decorator.md) | Attach additional responsibilities to an object dynamically |
| 10 | [Facade](./structural/10-facade.md) | Provide a simplified interface to a complex subsystem |
| 11 | [Flyweight](./structural/11-flyweight.md) | Use sharing to efficiently support a large number of fine-grained objects |
| 12 | [Proxy](./structural/12-proxy.md) | Provide a surrogate or placeholder for another object to control access to it |

---

## 🔄 Behavioral Patterns
*Concerned with communication and responsibility between objects.*

| # | Pattern | One-Line Summary |
|---|---|---|
| 13 | [Chain of Responsibility](./behavioral/13-chain-of-responsibility.md) | Pass a request along a chain of handlers until one handles it |
| 14 | [Command](./behavioral/14-command.md) | Encapsulate a request as an object, allowing undo, queuing, and logging |
| 15 | [Iterator](./behavioral/15-iterator.md) | Provide a way to sequentially access elements without exposing the underlying structure |
| 16 | [Mediator](./behavioral/16-mediator.md) | Define an object that encapsulates how a set of objects interact |
| 17 | [Memento](./behavioral/17-memento.md) | Capture and externalise an object's internal state for later restoration |
| 18 | [Observer](./behavioral/18-observer.md) | Define a one-to-many dependency so that when one object changes state, all dependents are notified |
| 19 | [State](./behavioral/19-state.md) | Allow an object to alter its behaviour when its internal state changes |
| 20 | [Strategy](./behavioral/20-strategy.md) | Define a family of algorithms, encapsulate each one, and make them interchangeable |
| 21 | [Template Method](./behavioral/21-template-method.md) | Define the skeleton of an algorithm, deferring some steps to subclasses |
| 22 | [Visitor](./behavioral/22-visitor.md) | Represent an operation to be performed on elements of an object structure |
| 23 | [Interpreter](./behavioral/23-interpreter.md) | Define a representation for a grammar and an interpreter that uses it |

---

## 🎯 Interview Preparation

| Resource | Description |
|---|---|
| [50 Interview Questions & Answers](./behavioral/24-interview-questions.md) | Covers pattern identification, trade-offs, real-world scenarios, and Java-specific questions |

---

[← Back to README](./README.md)
