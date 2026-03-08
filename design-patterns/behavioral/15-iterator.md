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

