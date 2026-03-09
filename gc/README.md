# ♻️ Java Garbage Collection — Complete Study Notes

> Comprehensive notes covering everything about Java GC — from heap memory structure to all 7 collectors, tuning flags, monitoring tools, and 100 interview questions.

---

## 📖 About These Notes

A progressive guide through Java's Garbage Collection system. Each section builds on the previous — from understanding what GC is, through JVM memory layout, generational collection, all collector types, tuning, monitoring, and production best practices.

**Java Version:** Java 21 LTS
**Covers:** GC fundamentals, JVM memory model, all 7 collectors (Serial → ZGC), tuning flags, monitoring tools, best practices, and 100 interview Q&A.

---

## 📂 File Structure

```
java-gc/
│
├── README.md                        ← You are here
├── INDEX.md                         ← Full table of contents
│
├── 01-what-is-gc.md                 ← GC fundamentals, goals, object eligibility
├── 02-jvm-memory-structure.md       ← Heap regions, Eden, Survivor, Old Gen, Metaspace
├── 03-generational-gc.md            ← Minor GC, Major GC, promotion, tenuring
├── 04-gc-process.md                 ← Marking, sweeping, copying, compaction phases
├── 05-garbage-collectors.md         ← All 7 collectors: Serial, Parallel, CMS, G1, ZGC, Shenandoah, Epsilon
├── 06-gc-tuning.md                  ← JVM flags, heap sizing, G1/ZGC tuning
├── 07-gc-monitoring-tools.md        ← jstat, jcmd, VisualVM, GCEasy, JFR/JMC
├── 08-best-practices.md             ← Production tips and common pitfalls
└── 09-interview-questions.md        ← 100 Q&A (moderate → complex)
```

---

## 🚀 Quick Start

| Goal | Start Here |
|---|---|
| New to GC | [01 — What is GC?](01-what-is-gc.md) |
| Understand heap layout | [02 — JVM Memory Structure](02-jvm-memory-structure.md) |
| Understand Minor/Major GC | [03 — Generational GC](03-generational-gc.md) |
| Compare all collectors | [05 — Garbage Collectors](05-garbage-collectors.md) |
| Tune GC in production | [06 — GC Tuning](06-gc-tuning.md) |
| Debug GC issues | [07 — Monitoring Tools](07-gc-monitoring-tools.md) |
| Prepare for interviews | [09 — 100 Interview Questions](09-interview-questions.md) |

---

## 🗺️ Learning Path

```
Beginner
  ├── 01 What is GC ──────────► Goals, eligibility, System.gc()
  ├── 02 JVM Memory ──────────► Heap regions, Metaspace
  ├── 03 Generational GC ─────► Minor GC, Major GC, promotion
  │
Intermediate
  ├── 04 GC Process ──────────► Mark → Sweep/Copy → Compact
  ├── 05 Collectors ──────────► Serial, Parallel, CMS, G1, ZGC, Shenandoah, Epsilon
  │
Advanced
  ├── 06 GC Tuning ───────────► JVM flags, sizing formulas
  ├── 07 Monitoring ──────────► jstat, GCEasy, JFR, log analysis
  ├── 08 Best Practices ──────► Production-ready patterns
  └── 09 Interview Questions ─► 100 Q&A, moderate to complex
```

---

## ⚡ Quick Cheat Sheet

```
Default collector (Java 9+): G1GC
Best for throughput:          Parallel GC  (-XX:+UseParallelGC)
Best for low latency:         ZGC          (-XX:+UseZGC)
Best general purpose:         G1 GC        (-XX:+UseG1GC)
No-op (testing only):         Epsilon GC   (-XX:+UseEpsilonGC)

Key heap flags:
  -Xms / -Xmx           Initial / max heap size
  -XX:NewRatio=3         Old:Young ratio
  -XX:MaxGCPauseMillis=200  G1 pause target

Key monitoring:
  jstat -gcutil <pid> 1000   Live GC stats
  jcmd <pid> GC.heap_info    Heap summary
  -Xlog:gc*:file=gc.log      Enable GC logging
```

---

## 📚 Related Notes

- [Java Multithreading Notes](../java-multithreading/README.md)
- [JVM Heap & Thread Dumps Analysis](../jvm-heap-thread-dumps.md)

---

## 🔗 References

- [Oracle Java GC Tuning Guide](https://docs.oracle.com/en/java/javase/21/gctuning/)
- [OpenJDK ZGC Wiki](https://wiki.openjdk.org/display/zgc)
- [Shenandoah GC — Red Hat](https://developers.redhat.com/articles/2021/09/16/shenandoah-garbage-collector)

---

*Last updated: 2026 | Java 21 LTS*
