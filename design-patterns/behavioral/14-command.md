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

