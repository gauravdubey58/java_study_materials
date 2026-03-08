[← Proxy](../structural/12-proxy.md) | [Index](../index.md)

---

# Pattern 13 — Chain of Responsibility
**Category:** Behavioral

---

## Intent

Pass a request along a **chain of handlers**. Each handler decides either to process the request or to pass it to the next handler in the chain.

---

## The Problem It Solves

You need to process a request through multiple checks (authentication → authorisation → validation → business logic → logging). Hard-coding the chain in one class violates Single Responsibility and makes it hard to add/remove steps.

---

## Java Implementation

```java
// Handler interface
public abstract class SupportHandler {
    protected SupportHandler next;

    public SupportHandler setNext(SupportHandler next) {
        this.next = next;
        return next;  // enables fluent chaining
    }

    public abstract void handle(SupportTicket ticket);

    protected void passToNext(SupportTicket ticket) {
        if (next != null) {
            next.handle(ticket);
        } else {
            System.out.println("Ticket " + ticket.getId() + " escalated — no handler available");
        }
    }
}

// Concrete Handlers
public class Level1Support extends SupportHandler {
    @Override
    public void handle(SupportTicket ticket) {
        if (ticket.getPriority() == Priority.LOW) {
            System.out.println("L1 Support resolved: " + ticket.getIssue());
        } else {
            System.out.println("L1 escalating ticket " + ticket.getId());
            passToNext(ticket);
        }
    }
}

public class Level2Support extends SupportHandler {
    @Override
    public void handle(SupportTicket ticket) {
        if (ticket.getPriority() == Priority.MEDIUM) {
            System.out.println("L2 Support resolved: " + ticket.getIssue());
        } else {
            System.out.println("L2 escalating ticket " + ticket.getId());
            passToNext(ticket);
        }
    }
}

public class Level3Support extends SupportHandler {
    @Override
    public void handle(SupportTicket ticket) {
        System.out.println("L3 (Engineering) resolved: " + ticket.getIssue());
    }
}

// Build and use the chain
SupportHandler l1 = new Level1Support();
SupportHandler l2 = new Level2Support();
SupportHandler l3 = new Level3Support();

l1.setNext(l2).setNext(l3);   // fluent chain setup

l1.handle(new SupportTicket("T1", "Login not working", Priority.LOW));
l1.handle(new SupportTicket("T2", "Data corruption", Priority.HIGH));
```

### HTTP Filter Chain (real-world pattern)

```java
// Spring Security / Servlet filter chains are CoR
public class AuthenticationFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req,
                                    HttpServletResponse res,
                                    FilterChain chain) throws IOException, ServletException {
        // Handle authentication
        String token = req.getHeader("Authorization");
        if (token == null) {
            res.setStatus(401);
            return;
        }
        // Pass to next filter in chain
        chain.doFilter(req, res);
    }
}
```

---

## Real-World Use Cases

- **Java Servlet Filter chain** — each filter calls `chain.doFilter()` to pass the request along
- **Spring Security filter chain** — authentication, CSRF, session management filters
- **Java logging** — `Logger` passes records up the logger hierarchy
- **Exception handling chains** — try/catch blocks, global exception handlers

## When to Use

- When more than one object may handle a request, and the handler isn't known a priori
- When you want to decouple senders from receivers
- When the set of handlers and their order should be configurable

---

| | |
|---|---|
| [← Proxy](../structural/12-proxy.md) | [Next → Command](./14-command.md) |

---
---

# Pattern 14 — Command
**Category:** Behavioral

---

## Intent

**Encapsulate a request as an object**, allowing you to parameterise clients with queues of requests, support undo/redo operations, and log requests.

---

## The Problem It Solves

A text editor needs Undo. A UI needs to attach actions to buttons that may vary at runtime. A task scheduler needs to queue work. All these require treating operations as first-class objects.

---

## Java Implementation

```java
// Command interface
public interface Command {
    void execute();
    void undo();
}

// Receiver — the object that does the actual work
public class TextEditor {
    private final StringBuilder content = new StringBuilder();

    public void insertText(String text, int position) {
        content.insert(position, text);
        System.out.println("Text after insert: " + content);
    }

    public void deleteText(int start, int length) {
        content.delete(start, start + length);
        System.out.println("Text after delete: " + content);
    }

    public String getContent() { return content.toString(); }
}

// Concrete Command — Insert
public class InsertTextCommand implements Command {
    private final TextEditor editor;
    private final String     text;
    private final int        position;

    public InsertTextCommand(TextEditor editor, String text, int position) {
        this.editor   = editor;
        this.text     = text;
        this.position = position;
    }

    @Override
    public void execute() { editor.insertText(text, position); }

    @Override
    public void undo()    { editor.deleteText(position, text.length()); }
}

// Invoker — manages command history
public class CommandHistory {
    private final Deque<Command> history = new ArrayDeque<>();
    private final Deque<Command> redoStack = new ArrayDeque<>();

    public void execute(Command command) {
        command.execute();
        history.push(command);
        redoStack.clear();
    }

    public void undo() {
        if (!history.isEmpty()) {
            Command cmd = history.pop();
            cmd.undo();
            redoStack.push(cmd);
        }
    }

    public void redo() {
        if (!redoStack.isEmpty()) {
            Command cmd = redoStack.pop();
            cmd.execute();
            history.push(cmd);
        }
    }
}

// Usage
TextEditor editor = new TextEditor();
CommandHistory history = new CommandHistory();

history.execute(new InsertTextCommand(editor, "Hello", 0));
history.execute(new InsertTextCommand(editor, " World", 5));
history.undo();   // removes " World"
history.redo();   // re-adds " World"
```

---

## Real-World Use Cases

- **Java's Runnable / Callable** — encapsulate tasks as command objects for ExecutorService
- **Spring Batch ItemProcessor** — encapsulates processing logic
- **javax.swing.Action** — command pattern for UI actions
- **Transaction management** — commit/rollback as commands
- **Message queues (Kafka, RabbitMQ)** — messages are commands consumed by workers

## When to Use

- When you need undo/redo functionality
- When you want to parameterise objects with operations
- When you want to queue, log, or schedule operations
- When you want to implement transactional behaviour

---

| | |
|---|---|
| [← Chain of Responsibility](./13-chain-of-responsibility.md) | [Next → Iterator](./15-iterator.md) |

---
---

# Pattern 15 — Iterator
**Category:** Behavioral

---

## Intent

Provide a way to **sequentially access elements** of a collection without exposing its underlying representation.

---

## The Problem It Solves

A collection might be backed by an array, a linked list, a tree, or a database query. Clients should be able to traverse any of these uniformly without knowing which data structure is underneath.

---

## Java Implementation

```java
// Iterator interface
public interface Iterator<T> {
    boolean hasNext();
    T next();
}

// Aggregate interface
public interface Collection<T> {
    Iterator<T> createIterator();
}

// Concrete Aggregate — a binary tree
public class BinaryTree<T extends Comparable<T>> implements Collection<T> {
    private Node<T> root;

    private static class Node<T> {
        T value;
        Node<T> left, right;
        Node(T value) { this.value = value; }
    }

    public void insert(T value) {
        root = insertRec(root, value);
    }

    private Node<T> insertRec(Node<T> node, T value) {
        if (node == null) return new Node<>(value);
        if (value.compareTo(node.value) < 0) node.left  = insertRec(node.left,  value);
        else                                  node.right = insertRec(node.right, value);
        return node;
    }

    // In-order iterator (produces sorted output for BST)
    @Override
    public Iterator<T> createIterator() {
        return new InOrderIterator<>(root);
    }

    private static class InOrderIterator<T> implements Iterator<T> {
        private final Deque<Node<T>> stack = new ArrayDeque<>();

        InOrderIterator(Node<T> root) {
            pushLeft(root);
        }

        private void pushLeft(Node<T> node) {
            while (node != null) {
                stack.push(node);
                node = node.left;
            }
        }

        @Override
        public boolean hasNext() { return !stack.isEmpty(); }

        @Override
        public T next() {
            Node<T> node = stack.pop();
            pushLeft(node.right);
            return node.value;
        }
    }
}

// Usage — client doesn't know it's traversing a tree
BinaryTree<Integer> tree = new BinaryTree<>();
tree.insert(5); tree.insert(3); tree.insert(7); tree.insert(1);

Iterator<Integer> it = tree.createIterator();
while (it.hasNext()) {
    System.out.print(it.next() + " ");  // 1 3 5 7 (sorted)
}
```

---

## Real-World Use Cases

- **Java's `java.util.Iterator`** — standard iterator used by all Collection classes
- **Java's enhanced for-loop** — uses `Iterable` and `Iterator` behind the scenes
- **Spring Data's `Page`** — iterates over query results page by page
- **ResultSet** — iterates over database rows

## When to Use

- When you want a standard way to traverse different types of collections
- When you want to hide the internal structure of a collection
- When you want multiple simultaneous traversals of a collection

---

| | |
|---|---|
| [← Command](./14-command.md) | [Next → Mediator](./16-mediator.md) |

---
---

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

# Pattern 17 — Memento
**Category:** Behavioral

---

## Intent

**Capture and externalise** an object's internal state without violating encapsulation, so the object can be restored to this state later.

---

## The Problem It Solves

You want to implement undo, or save/restore game state, or create checkpoints — but you don't want to expose the object's internal state to achieve this.

---

## Java Implementation

```java
// Memento — stores the state snapshot
public final class EditorMemento {
    private final String content;
    private final int    cursorPosition;
    private final String selectedText;
    private final Instant savedAt;

    EditorMemento(String content, int cursorPosition, String selectedText) {
        this.content        = content;
        this.cursorPosition = cursorPosition;
        this.selectedText   = selectedText;
        this.savedAt        = Instant.now();
    }

    // Only the Originator can access state
    String  getContent()        { return content; }
    int     getCursorPosition() { return cursorPosition; }
    String  getSelectedText()   { return selectedText; }
    Instant getSavedAt()        { return savedAt; }
}

// Originator — creates and restores mementos
public class TextEditor {
    private String content        = "";
    private int    cursorPosition = 0;
    private String selectedText   = "";

    public void type(String text) {
        content = content.substring(0, cursorPosition) + text + content.substring(cursorPosition);
        cursorPosition += text.length();
    }

    public EditorMemento save() {
        System.out.println("Saving state: '" + content + "'");
        return new EditorMemento(content, cursorPosition, selectedText);
    }

    public void restore(EditorMemento memento) {
        this.content        = memento.getContent();
        this.cursorPosition = memento.getCursorPosition();
        this.selectedText   = memento.getSelectedText();
        System.out.println("Restored state: '" + content + "'");
    }

    public String getContent() { return content; }
}

// Caretaker — manages the history of mementos
public class History {
    private final Deque<EditorMemento> stack = new ArrayDeque<>();

    public void push(EditorMemento memento) {
        stack.push(memento);
    }

    public EditorMemento pop() {
        return stack.isEmpty() ? null : stack.pop();
    }
}

// Usage
TextEditor editor = new TextEditor();
History history = new History();

editor.type("Hello");
history.push(editor.save());

editor.type(" World");
history.push(editor.save());

editor.type("!!!");
System.out.println("Current: " + editor.getContent());

editor.restore(history.pop());  // undo !!!
editor.restore(history.pop());  // undo World
```

---

## Real-World Use Cases

- **Ctrl+Z undo** in any editor
- **Database transaction rollback** — savepoint mechanism
- **Game saves** — checkpoint and reload
- **Version control** — each commit is a memento

## When to Use

- When you need to implement undo/redo
- When direct access to an object's state would expose implementation details
- When you need snapshots/checkpoints of an object's state

---

| | |
|---|---|
| [← Mediator](./16-mediator.md) | [Next → Observer](./18-observer.md) |

---
---

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

# Pattern 19 — State
**Category:** Behavioral

---

## Intent

Allow an object to **alter its behaviour when its internal state changes**. The object will appear to change its class.

---

## The Problem It Solves

A vending machine behaves differently depending on whether it has items, has coins inserted, or is dispensing. Without State pattern, this leads to massive if/else or switch blocks that grow whenever a new state is added.

---

## Java Implementation

```java
// State interface
public interface VendingMachineState {
    void insertCoin(VendingMachine machine);
    void pressButton(VendingMachine machine);
    void dispense(VendingMachine machine);
    String getStateName();
}

// Context
public class VendingMachine {
    private VendingMachineState state;
    private int itemCount;

    public VendingMachine(int itemCount) {
        this.itemCount = itemCount;
        this.state = itemCount > 0 ? new IdleState() : new EmptyState();
    }

    public void setState(VendingMachineState state)  { this.state = state; }
    public int  getItemCount()                        { return itemCount; }
    public void decrementItem()                       { itemCount--; }

    // Delegate all behaviour to current state
    public void insertCoin()  { state.insertCoin(this); }
    public void pressButton() { state.pressButton(this); }

    @Override
    public String toString() { return "VendingMachine[state=" + state.getStateName() + ", items=" + itemCount + "]"; }
}

// Concrete States
public class IdleState implements VendingMachineState {
    @Override
    public void insertCoin(VendingMachine m) {
        System.out.println("Coin inserted. Press button to dispense.");
        m.setState(new CoinInsertedState());
    }
    @Override
    public void pressButton(VendingMachine m) { System.out.println("Please insert a coin first."); }
    @Override
    public void dispense(VendingMachine m)    { System.out.println("Please insert a coin first."); }
    @Override
    public String getStateName() { return "IDLE"; }
}

public class CoinInsertedState implements VendingMachineState {
    @Override
    public void insertCoin(VendingMachine m) { System.out.println("Coin already inserted."); }
    @Override
    public void pressButton(VendingMachine m) {
        System.out.println("Dispensing item...");
        m.decrementItem();
        m.setState(m.getItemCount() > 0 ? new IdleState() : new EmptyState());
    }
    @Override
    public void dispense(VendingMachine m) { System.out.println("Press button first."); }
    @Override
    public String getStateName() { return "COIN_INSERTED"; }
}

public class EmptyState implements VendingMachineState {
    @Override
    public void insertCoin(VendingMachine m) { System.out.println("Machine is empty. Returning coin."); }
    @Override
    public void pressButton(VendingMachine m) { System.out.println("Machine is empty."); }
    @Override
    public void dispense(VendingMachine m)    { System.out.println("Machine is empty."); }
    @Override
    public String getStateName() { return "EMPTY"; }
}

// Usage
VendingMachine machine = new VendingMachine(2);
machine.pressButton();    // Please insert a coin first
machine.insertCoin();     // Coin inserted
machine.pressButton();    // Dispensing item
machine.pressButton();    // Please insert a coin first (back to idle)
machine.insertCoin();
machine.pressButton();    // Dispensing last item — transitions to EMPTY
machine.insertCoin();     // Machine is empty. Returning coin.
```

---

## Real-World Use Cases

- **Spring Batch** — job execution states (STARTED, STOPPED, FAILED, COMPLETED)
- **TCP connection states** — CLOSED, LISTEN, SYN_SENT, ESTABLISHED, etc.
- **Order management** — PLACED, CONFIRMED, SHIPPED, DELIVERED, CANCELLED
- **Thread lifecycle** — NEW, RUNNABLE, BLOCKED, WAITING, TERMINATED

## When to Use

- When an object's behaviour depends on its state and must change at runtime
- When operations have large, multi-part conditional statements based on the object's state
- When state transitions have their own logic and side effects

---

| | |
|---|---|
| [← Observer](./18-observer.md) | [Next → Strategy](./20-strategy.md) |

---
---

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

# Pattern 21 — Template Method
**Category:** Behavioral

---

## Intent

Define the **skeleton of an algorithm** in a base class, deferring some steps to subclasses. Template Method lets subclasses redefine certain steps of an algorithm without changing the algorithm's structure.

---

## The Problem It Solves

Two data mining programs extract data from different sources (CSV and XML), but the overall process is the same: open → parse → extract → analyse → report. The structure shouldn't be duplicated; only the specific steps differ.

---

## Java Implementation

```java
// Abstract class with template method
public abstract class DataProcessor {

    // TEMPLATE METHOD — defines the algorithm skeleton, final so it can't be overridden
    public final ProcessingResult process(String source) {
        System.out.println("=== Starting data processing ===");
        RawData    raw       = openDataSource(source);      // abstract
        ParsedData parsed    = parseData(raw);               // abstract
        Data       extracted = extractData(parsed);          // abstract
        Data       filtered  = filterData(extracted);        // hook (optional override)
        Report     report    = generateReport(filtered);     // concrete (shared)
        closeDataSource(raw);                                // concrete (shared)
        return new ProcessingResult(report);
    }

    // Abstract steps — subclasses MUST implement
    protected abstract RawData    openDataSource(String source);
    protected abstract ParsedData parseData(RawData raw);
    protected abstract Data       extractData(ParsedData parsed);

    // Hook methods — subclasses MAY override (have default implementation)
    protected Data filterData(Data data) {
        System.out.println("Applying default filter (no-op)");
        return data;
    }

    // Concrete steps — shared by all subclasses
    private Report generateReport(Data data) {
        System.out.println("Generating standardised report");
        return new Report(data);
    }

    private void closeDataSource(RawData raw) {
        System.out.println("Closing data source");
    }
}

// Concrete implementations
public class CsvDataProcessor extends DataProcessor {
    @Override
    protected RawData openDataSource(String path) {
        System.out.println("Opening CSV file: " + path);
        return new RawData(path);
    }
    @Override
    protected ParsedData parseData(RawData raw) {
        System.out.println("Parsing CSV with comma delimiter");
        return new ParsedData();
    }
    @Override
    protected Data extractData(ParsedData parsed) {
        System.out.println("Extracting columns from CSV rows");
        return new Data();
    }
}

public class XmlDataProcessor extends DataProcessor {
    @Override
    protected RawData openDataSource(String path) {
        System.out.println("Opening XML file: " + path);
        return new RawData(path);
    }
    @Override
    protected ParsedData parseData(RawData raw) {
        System.out.println("Parsing XML DOM tree");
        return new ParsedData();
    }
    @Override
    protected Data extractData(ParsedData parsed) {
        System.out.println("Extracting elements using XPath");
        return new Data();
    }
    @Override
    protected Data filterData(Data data) {
        System.out.println("XML: Removing namespace prefixes");
        return data;
    }
}

// Usage
DataProcessor csvProcessor = new CsvDataProcessor();
csvProcessor.process("report.csv");

DataProcessor xmlProcessor = new XmlDataProcessor();
xmlProcessor.process("data.xml");
```

---

## Real-World Use Cases

- **Spring Batch** — `AbstractItemCountingItemStreamItemReader` defines the template; subclasses implement `doRead()`
- **Spring's JdbcTemplate** — template method controls connection lifecycle; callback provides SQL
- **Java's `AbstractList`** — `get()` and `size()` are abstract; everything else (iterator, subList, etc.) is concrete
- **HttpServlet** — `service()` is the template; `doGet()`, `doPost()` are the steps

## When to Use

- When you want to let subclasses implement varying behaviour while keeping shared structure in one place
- When common behaviour across multiple subclasses should be factored into a shared class to avoid duplication
- When you want to control which steps subclasses may extend (hooks vs abstract methods)

---

| | |
|---|---|
| [← Strategy](./20-strategy.md) | [Next → Visitor](./22-visitor.md) |

---
---

# Pattern 22 — Visitor
**Category:** Behavioral

---

## Intent

Represent an operation to be performed on elements of an object structure. Visitor lets you define a new operation **without changing the classes** of the elements on which it operates.

---

## The Problem It Solves

You have a document model (Paragraph, Image, Table, Heading). You want to add operations like export to HTML, export to PDF, spell-check, and word-count — without modifying the document element classes each time.

---

## Java Implementation

```java
// Visitor interface — one visit method per element type
public interface DocumentVisitor {
    void visitParagraph(Paragraph paragraph);
    void visitImage(Image image);
    void visitTable(Table table);
    void visitHeading(Heading heading);
}

// Element interface
public interface DocumentElement {
    void accept(DocumentVisitor visitor);  // "double dispatch"
}

// Concrete Elements
public class Paragraph implements DocumentElement {
    private final String text;
    public Paragraph(String text) { this.text = text; }
    public String getText() { return text; }
    @Override
    public void accept(DocumentVisitor visitor) { visitor.visitParagraph(this); }
}

public class Image implements DocumentElement {
    private final String url;
    private final int width, height;
    public Image(String url, int w, int h) { this.url = url; width = w; height = h; }
    public String getUrl() { return url; }
    public int getWidth()  { return width; }
    public int getHeight() { return height; }
    @Override
    public void accept(DocumentVisitor visitor) { visitor.visitImage(this); }
}

public class Heading implements DocumentElement {
    private final String text;
    private final int level;
    public Heading(String text, int level) { this.text = text; this.level = level; }
    public String getText()  { return text; }
    public int    getLevel() { return level; }
    @Override
    public void accept(DocumentVisitor visitor) { visitor.visitHeading(this); }
}

// Concrete Visitors — new operations without touching elements
public class HtmlExportVisitor implements DocumentVisitor {
    private final StringBuilder html = new StringBuilder();

    @Override
    public void visitParagraph(Paragraph p) {
        html.append("<p>").append(p.getText()).append("</p>\n");
    }
    @Override
    public void visitImage(Image img) {
        html.append(String.format("<img src=\"%s\" width=\"%d\" height=\"%d\"/>%n",
                img.getUrl(), img.getWidth(), img.getHeight()));
    }
    @Override
    public void visitTable(Table t) { html.append("<table>...</table>\n"); }
    @Override
    public void visitHeading(Heading h) {
        html.append("<h").append(h.getLevel()).append(">")
            .append(h.getText())
            .append("</h").append(h.getLevel()).append(">\n");
    }
    public String getHtml() { return html.toString(); }
}

public class WordCountVisitor implements DocumentVisitor {
    private int count = 0;

    @Override
    public void visitParagraph(Paragraph p) { count += p.getText().split("\\s+").length; }
    @Override
    public void visitHeading(Heading h)     { count += h.getText().split("\\s+").length; }
    @Override
    public void visitImage(Image img)       { /* images have no words */ }
    @Override
    public void visitTable(Table t)         { /* simplification */ }
    public int getWordCount() { return count; }
}

// Usage
List<DocumentElement> doc = List.of(
    new Heading("Introduction", 1),
    new Paragraph("This is the first paragraph."),
    new Image("photo.jpg", 800, 600)
);

HtmlExportVisitor htmlVisitor = new HtmlExportVisitor();
WordCountVisitor  wordVisitor = new WordCountVisitor();

doc.forEach(el -> el.accept(htmlVisitor));
doc.forEach(el -> el.accept(wordVisitor));

System.out.println(htmlVisitor.getHtml());
System.out.println("Words: " + wordVisitor.getWordCount());
```

---

## Real-World Use Cases

- **Java compiler AST visitors** — `javax.lang.model.element.ElementVisitor`
- **Spring's BeanDefinitionVisitor** — traverses bean definition structures
- **ANTLR parse tree visitors** — visit AST nodes for language processing
- **Hibernate's SQL AST** — visitor-based SQL generation

## When to Use

- When you need to perform many distinct and unrelated operations on an object structure
- When the object structure classes rarely change, but you often add new operations
- When you don't want to pollute element classes with unrelated operations

---

| | |
|---|---|
| [← Template Method](./21-template-method.md) | [Next → Interpreter](./23-interpreter.md) |

---
---

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
