# ☕ Java 8 → 21: Modern Java Features Reference

> Comprehensive notes covering every major Java version feature with real-world examples, use cases, and corner cases.  
> Focus: Java developers upgrading their knowledge and preparing for modern Java development.

---

## 📁 Files in This Series

| File | Versions | Key Features |
|------|---------|-------------|
| [java-8.md](./java-8.md) | Java 8 (2014) | Lambdas, Streams, Optional, Date/Time API, Default Methods, CompletableFuture |
| [java-9-10-11.md](./java-9-10-11.md) | Java 9–11 (2017–2018) | Modules, `var`, HTTP Client, String/Collection improvements |
| [java-12-14.md](./java-12-14.md) | Java 12–14 (2019–2020) | Switch Expressions, Text Blocks, Records (preview), Helpful NPE |
| [java-15-17.md](./java-15-17.md) | Java 15–17 LTS (2020–2021) | Sealed Classes, Records (final), Pattern Matching instanceof, Text Blocks (final) |
| [java-18-21.md](./java-18-21.md) | Java 18–21 LTS (2022–2023) | Virtual Threads, Sequenced Collections, Pattern Matching Switch, Record Patterns, Structured Concurrency |

---

## 🗺️ Feature Timeline at a Glance

```
Java 8  (LTS) ── Lambdas · Streams · Optional · Date/Time · Default Methods · CompletableFuture
Java 9        ── Module System · Collection.of() · Stream improvements · HTTP Client (incubator)
Java 10       ── var (local type inference)
Java 11 (LTS) ── HTTP Client (final) · String methods · Files.readString() · Running single files
Java 12       ── Switch Expression (preview)
Java 13       ── Text Blocks (preview) · Switch Expression (preview 2)
Java 14       ── Switch Expression (final) · Records (preview) · Helpful NPE messages
Java 15       ── Text Blocks (final) · Sealed Classes (preview) · Records (preview 2)
Java 16       ── Records (final) · Pattern Matching instanceof (final) · Stream.toList()
Java 17 (LTS) ── Sealed Classes (final) · Strong encapsulation of JDK internals
Java 18       ── UTF-8 by default · Simple Web Server · @snippet Javadoc
Java 19       ── Virtual Threads (preview) · Structured Concurrency (incubator)
Java 20       ── Scoped Values (incubator) · Record Patterns (preview 2)
Java 21 (LTS) ── Virtual Threads (final) · Sequenced Collections · Pattern Matching Switch (final)
               ── Record Patterns (final) · Structured Concurrency (preview)
```

---

## 🏆 LTS Versions to Know

| Version | LTS | Support Until | Why It Matters |
|---------|-----|---------------|---------------|
| Java 8 | ✅ | 2030 (Azul/Amazon) | Still widely deployed in enterprise |
| Java 11 | ✅ | 2026+ | First major post-8 LTS |
| Java 17 | ✅ | 2029 | Current enterprise standard |
| Java 21 | ✅ | 2031 | Modern Java — Virtual Threads, Pattern Matching |

---

## ⚡ Top Features by Impact (for Senior Java Developers)

| Feature | Version | Impact Level |
|---------|---------|-------------|
| Virtual Threads (Project Loom) | Java 21 | 🔴 Game-changing for backend services |
| Records | Java 16 | 🔴 Replaces most DTO boilerplate |
| Pattern Matching Switch | Java 21 | 🟠 Transforms conditional logic |
| Sealed Classes | Java 17 | 🟠 Better domain modelling |
| Streams + Lambdas | Java 8 | 🔴 Already essential — master deeply |
| CompletableFuture | Java 8 | 🟠 Async programming foundation |
| Text Blocks | Java 15 | 🟡 Practical daily-use improvement |
| `var` | Java 10 | 🟡 Reduces verbosity |
| HTTP Client | Java 11 | 🟡 Replaces Apache HttpClient in many cases |
| Sequenced Collections | Java 21 | 🟡 Fixes long-standing API gap |

---

## 🔗 Official Resources

| Resource | Link |
|----------|------|
| OpenJDK JEP Index | [openjdk.org/jeps](https://openjdk.org/jeps/0) |
| Java SE Release Notes | [docs.oracle.com/en/java/javase](https://docs.oracle.com/en/java/javase/) |
| Baeldung Java Guides | [baeldung.com](https://www.baeldung.com) |
| Inside Java | [inside.java](https://inside.java) |
