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

