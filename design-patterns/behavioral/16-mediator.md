# Pattern 16 — Mediator
**Category:** Behavioral

---

## Intent

Define an object that **encapsulates how a set of objects interact**. Mediator promotes loose coupling by preventing objects from referring to each other explicitly.

---

## The Problem It Solves

In a chat application, without a mediator, every user object would need a reference to every other user. Adding a new user requires updating all existing users. Mediator centralises communication — users only know the mediator.

---

## Java Implementation

```java
// Mediator interface
public interface ChatMediator {
    void sendMessage(String message, User sender);
    void addUser(User user);
}

// Concrete Mediator
public class ChatRoom implements ChatMediator {
    private final List<User> users = new ArrayList<>();

    @Override
    public void addUser(User user) {
        users.add(user);
        System.out.println(user.getName() + " joined the chat");
    }

    @Override
    public void sendMessage(String message, User sender) {
        users.stream()
             .filter(u -> u != sender)
             .forEach(u -> u.receive(message, sender.getName()));
    }
}

// Colleague
public class User {
    private final String        name;
    private final ChatMediator  mediator;

    public User(String name, ChatMediator mediator) {
        this.name     = name;
        this.mediator = mediator;
    }

    public String getName() { return name; }

    public void send(String message) {
        System.out.println(name + " sends: " + message);
        mediator.sendMessage(message, this);
    }

    public void receive(String message, String from) {
        System.out.println(name + " receives from " + from + ": " + message);
    }
}

// Usage
ChatMediator room = new ChatRoom();
User alice = new User("Alice", room);
User bob   = new User("Bob",   room);
User carol = new User("Carol", room);

room.addUser(alice);
room.addUser(bob);
room.addUser(carol);

alice.send("Hello everyone!");
bob.send("Hi Alice!");
```

---

## Real-World Use Cases

- **Air traffic control** — planes communicate through the control tower (mediator)
- **Java's `java.util.concurrent.Executor`** — mediates between task submitters and worker threads
- **Spring's ApplicationEventPublisher** — mediates between event publishers and listeners
- **MVC pattern** — Controller is the mediator between Model and View

## When to Use

- When many objects communicate in complex ways, leading to tight coupling
- When you can't reuse a component because it communicates with too many others
- When you want a centralised point of control for object interactions

---

| | |
|---|---|
| [← Iterator](./15-iterator.md) | [Next → Memento](./17-memento.md) |

---
---

