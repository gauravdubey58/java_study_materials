# 01 — What is Garbage Collection?

> Covers: GC definition, goals, object eligibility, GC roots, reference types (Strong/Soft/Weak/Phantom), and finalization.

---

**Navigation**

[🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → JVM Memory Structure](02-jvm-memory-structure.md)

---

## Table of Contents

- [1.1 Definition & Goals](#11-definition--goals)
- [1.2 When is an Object Eligible for GC?](#12-when-is-an-object-eligible-for-gc)
- [1.3 GC Roots](#13-gc-roots)
- [1.4 Reference Types](#14-reference-types)
- [1.5 Finalization & Cleaners](#15-finalization--cleaners)
- [Summary](#summary)

---

## 1.1 Definition & Goals

**Garbage Collection (GC)** is the automatic memory management process in Java where the JVM reclaims heap memory occupied by objects that are no longer reachable by the application.

Unlike C/C++ where developers manually call `malloc()`/`free()`, Java developers never explicitly deallocate memory — the GC handles it automatically.

### Primary Goals

| Goal | Description |
|---|---|
| **Reclaim unused memory** | Free heap space held by unreachable objects |
| **Avoid memory leaks** | Prevent unbounded heap growth over time |
| **Minimize pause times** | Reduce or eliminate stop-the-world (STW) pauses |
| **Maximize throughput** | Dedicate maximum CPU time to the application |
| **Avoid fragmentation** | Keep heap memory contiguous for fast allocation |

### The GC Trade-off Triangle

```
        Throughput
           /\
          /  \
         /    \
        /      \
       /________\
  Latency    Footprint
  (pauses)  (memory used)

You can optimize for at most TWO of these three at a time.
- Parallel GC:    High Throughput, small Footprint, higher Latency
- G1 GC:          Balanced Throughput, moderate Footprint, low Latency
- ZGC:            High Throughput, larger Footprint, ultra-low Latency
```

### What GC Does NOT Do

- GC does **not** guarantee immediate collection — it runs on its own schedule.
- `System.gc()` is only a **hint** to the JVM — it may be ignored.
- GC does **not** prevent all memory leaks — logical leaks (e.g., forgotten references in a static `List`) are not detected.
- GC does **not** manage native memory (allocated via `ByteBuffer.allocateDirect()` or JNI).

---

## 1.2 When is an Object Eligible for GC?

An object becomes **eligible** for garbage collection when it is **no longer reachable** from any GC root through any chain of references.

```java
// Object becomes eligible immediately after this block
{
    Object obj = new Object();   // obj created on heap
    // ... use obj ...
}   // obj goes out of scope → no more references → eligible for GC

// Explicit nulling
String s = "hello";
s = null;   // Original string may now be eligible (if no other references)

// Cyclic references — GC handles these correctly
class Node {
    Node next;
}
Node a = new Node();
Node b = new Node();
a.next = b;
b.next = a;
a = null;
b = null;
// Both a and b are now eligible — GC traces from roots, not reference counts
```

### Why Java GC Handles Cyclic References

Java uses **tracing GC** (reachability analysis from roots), **not reference counting** like Python. An object is dead if it cannot be reached from any root, regardless of how many other dead objects point to it.

```
GC Root ──► A ──► B ──► C       A, B, C = LIVE (reachable from root)

GC Root                         X ──► Y
                                ↑         ↓
                                Z ◄───────┘

X, Y, Z = DEAD (not reachable from any root, even though they reference each other)
```

---

## 1.3 GC Roots

GC Roots are the **starting points** of the reachability graph. Any object reachable from a GC root is considered live and will NOT be collected.

| GC Root Type | Example |
|---|---|
| **Local variables** | Variables on the call stack of any active thread |
| **Active thread objects** | `Thread` instances that are alive |
| **Static fields** | `static` fields of loaded classes |
| **JNI references** | References held by native code via JNI |
| **Monitor objects** | Objects used as synchronization locks |
| **Class loaders** | Active `ClassLoader` instances |
| **JVM internals** | String interning pool, exception objects, etc. |

```java
public class GcRootExample {
    static Object STATIC_ROOT = new Object();  // GC Root: static field

    public void method() {
        Object localRoot = new Object();        // GC Root: local variable (while on stack)

        Thread t = new Thread(() -> {
            Object threadRoot = new Object();   // GC Root: local var of active thread
        });
        t.start(); // threadRoot is a root while the thread is alive
    }
}
```

> **Key insight:** The GC never collects objects reachable from roots. Memory leaks in Java happen when roots (usually static collections or listener registries) hold references to objects the application no longer needs.

---

## 1.4 Reference Types

Java provides four reference strength levels, giving developers control over how objects interact with GC.

### Strong Reference (default)

```java
Object obj = new Object(); // Strong reference
// obj will NEVER be collected while this reference exists
```

### Soft Reference — Collected under Memory Pressure

```java
import java.lang.ref.SoftReference;

SoftReference<byte[]> softRef = new SoftReference<>(new byte[1024 * 1024]);

// JVM guarantees to clear soft references BEFORE throwing OutOfMemoryError
byte[] data = softRef.get(); // Returns null if GC has collected the object
if (data == null) {
    data = reloadData(); // Cache miss — reload
    softRef = new SoftReference<>(data);
}
```

**Use case:** In-memory caches. JVM evicts cache entries only when memory is truly needed.

### Weak Reference — Collected at Next GC

```java
import java.lang.ref.WeakReference;

WeakReference<Object> weakRef = new WeakReference<>(new Object());

// Object is eligible for collection at the very NEXT GC cycle
// (if no strong references exist)
Object obj = weakRef.get(); // Returns null if already collected
if (obj != null) {
    use(obj);
}
```

**Use case:** `WeakHashMap` — map entries auto-removed when keys have no strong references. Useful for caches keyed on external objects.

```java
WeakHashMap<Widget, WidgetMetadata> cache = new WeakHashMap<>();
// When 'widget' object is GC'd, its entry is automatically removed from cache
```

### Phantom Reference — Post-Collection Cleanup Hook

```java
import java.lang.ref.*;

ReferenceQueue<Object> queue = new ReferenceQueue<>();
PhantomReference<Object> phantomRef = new PhantomReference<>(new Object(), queue);

// phantomRef.get() ALWAYS returns null — you can never access the referent
// But the Reference is enqueued to 'queue' AFTER the object is finalized
// Use to perform cleanup AFTER GC without the risks of finalize()
```

**Use case:** Implementing off-heap resource cleanup (file handles, native memory) more safely than `finalize()`.

### Reference Strength Summary

| Type | Collected When | `get()` returns null | Use Case |
|---|---|---|---|
| **Strong** | Never (while held) | Never | Normal usage |
| **Soft** | Memory pressure (before OOM) | Yes, after collection | Memory-sensitive caches |
| **Weak** | Next GC (if no strong ref) | Yes, after collection | `WeakHashMap`, canonicalization |
| **Phantom** | After finalization | Always | Post-GC cleanup hooks |

---

## 1.5 Finalization & Cleaners

### `finalize()` — Deprecated in Java 9, Removed in Java 18

`Object.finalize()` was called by the GC before collecting an object, allowing cleanup of native resources. It was fundamentally broken:

- **Unpredictable timing** — no guarantee when or if it runs
- **Performance impact** — finalizable objects survive at least one extra GC cycle
- **Can resurrect objects** — re-assigning `this` in `finalize()` prevents collection
- **Thread-unsafe** — runs on a dedicated finalizer thread with no ordering guarantees

```java
// DON'T DO THIS (deprecated pattern)
@Override
protected void finalize() throws Throwable {
    try { closeResource(); }
    finally { super.finalize(); }
}
```

### `Cleaner` — The Modern Alternative (Java 9+)

```java
import java.lang.ref.Cleaner;

public class NativeBuffer implements AutoCloseable {
    private static final Cleaner CLEANER = Cleaner.create();

    private final long nativePtr;
    private final Cleaner.Cleanable cleanable;

    public NativeBuffer(int size) {
        this.nativePtr = allocateNative(size);  // Allocate off-heap memory
        // Register cleanup action — runs AFTER object is GC'd if close() wasn't called
        this.cleanable = CLEANER.register(this, new CleanAction(nativePtr));
    }

    @Override
    public void close() {
        cleanable.clean(); // Explicit cleanup — preferred path
    }

    // Static class — must NOT reference the outer NativeBuffer instance
    // (would prevent GC of the outer object)
    private static class CleanAction implements Runnable {
        private final long ptr;
        CleanAction(long ptr) { this.ptr = ptr; }

        @Override public void run() {
            freeNative(ptr); // Native cleanup
        }
    }
}

// Usage with try-with-resources (preferred)
try (NativeBuffer buf = new NativeBuffer(1024)) {
    buf.write(data);
} // close() called → cleanable.clean() → freeNative()
```

### Try-With-Resources — Best Practice for Resource Cleanup

```java
// Best approach: explicit resource management via AutoCloseable
try (FileInputStream fis = new FileInputStream("file.txt");
     BufferedReader br = new BufferedReader(new InputStreamReader(fis))) {
    String line;
    while ((line = br.readLine()) != null) process(line);
} // Resources closed in reverse order automatically
```

---

## Summary

| Concept | Key Takeaway |
|---|---|
| GC purpose | Automatic heap memory reclamation — developers never call `free()` |
| Eligibility | Object is dead when no GC root can reach it |
| GC roots | Active thread stacks, static fields, JNI refs — roots are never collected |
| Cyclic refs | Handled correctly — Java uses tracing, not reference counting |
| Soft refs | Cleared before OOM — good for caches |
| Weak refs | Cleared at next GC — good for `WeakHashMap` |
| Phantom refs | For post-GC cleanup hooks |
| `finalize()` | Deprecated/removed — use `Cleaner` or try-with-resources |

---

**Navigation**

[🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → JVM Memory Structure](02-jvm-memory-structure.md)

---

*Last updated: 2026 | Java 21 LTS*
