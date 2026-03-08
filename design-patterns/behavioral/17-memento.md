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

