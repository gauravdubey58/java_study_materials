# 📑 Java Garbage Collection — Full Index

> Complete table of contents. Click any topic to jump directly to it.

---

[← Back to README](README.md)

---

## Sections Overview

| # | File | Topics Covered |
|---|---|---|
| 01 | [What is GC?](01-what-is-gc.md) | GC goals, object eligibility, GC roots, finalization |
| 02 | [JVM Memory Structure](02-jvm-memory-structure.md) | Heap regions, Eden, Survivor, Old Gen, Metaspace, Stack |
| 03 | [Generational GC](03-generational-gc.md) | Minor GC, Major GC, Full GC, promotion, tenuring, Metaspace GC |
| 04 | [GC Process](04-gc-process.md) | Marking, sweeping, copying, compaction, STW pauses, write barriers |
| 05 | [Garbage Collectors](05-garbage-collectors.md) | Serial, Parallel, CMS, G1, ZGC, Shenandoah, Epsilon |
| 06 | [GC Tuning](06-gc-tuning.md) | JVM flags, heap sizing formulas, G1/ZGC tuning |
| 07 | [GC Monitoring Tools](07-gc-monitoring-tools.md) | jstat, jcmd, VisualVM, GCEasy, JFR/JMC, log analysis |
| 08 | [Best Practices](08-best-practices.md) | Production tips, common pitfalls, allocation patterns |
| 09 | [Interview Questions](09-interview-questions.md) | 100 Q&A from moderate to complex |

---

## Detailed Index

### [01 — What is GC?](01-what-is-gc.md)
- [1.1 Definition & Goals](01-what-is-gc.md#11-definition--goals)
- [1.2 Object Eligibility](01-what-is-gc.md#12-when-is-an-object-eligible-for-gc)
- [1.3 GC Roots](01-what-is-gc.md#13-gc-roots)
- [1.4 Reference Types](01-what-is-gc.md#14-reference-types)
- [1.5 Finalization & Cleaners](01-what-is-gc.md#15-finalization--cleaners)

### [02 — JVM Memory Structure](02-jvm-memory-structure.md)
- [2.1 Heap Overview](02-jvm-memory-structure.md#21-heap-overview)
- [2.2 Young Generation](02-jvm-memory-structure.md#22-young-generation)
- [2.3 Old Generation](02-jvm-memory-structure.md#23-old-generation-tenured-space)
- [2.4 Metaspace](02-jvm-memory-structure.md#24-metaspace)
- [2.5 Non-Heap Memory](02-jvm-memory-structure.md#25-non-heap-memory)
- [2.6 TLAB — Thread-Local Allocation Buffer](02-jvm-memory-structure.md#26-tlab--thread-local-allocation-buffer)

### [03 — Generational GC](03-generational-gc.md)
- [3.1 Weak Generational Hypothesis](03-generational-gc.md#31-weak-generational-hypothesis)
- [3.2 Minor GC (Young Generation)](03-generational-gc.md#32-minor-gc-young-generation)
- [3.3 Major GC / Full GC](03-generational-gc.md#33-major-gc--full-gc)
- [3.4 Metaspace GC](03-generational-gc.md#34-metaspace-gc)
- [3.5 Object Promotion & Tenuring](03-generational-gc.md#35-object-promotion--tenuring)

### [04 — GC Process](04-gc-process.md)
- [4.1 Phase 1: Marking](04-gc-process.md#41-phase-1-marking)
- [4.2 Phase 2: Sweeping / Copying](04-gc-process.md#42-phase-2-sweeping--copying)
- [4.3 Phase 3: Compaction](04-gc-process.md#43-phase-3-compaction)
- [4.4 Stop-The-World Pauses](04-gc-process.md#44-stop-the-world-stw-pauses)
- [4.5 Concurrent vs Parallel GC](04-gc-process.md#45-concurrent-vs-parallel-gc)
- [4.6 Write Barriers](04-gc-process.md#46-write-barriers)

### [05 — Garbage Collectors](05-garbage-collectors.md)
- [5.1 Serial GC](05-garbage-collectors.md#51-serial-gc)
- [5.2 Parallel GC](05-garbage-collectors.md#52-parallel-gc)
- [5.3 CMS GC (Deprecated)](05-garbage-collectors.md#53-cms--concurrent-mark-sweep)
- [5.4 G1 GC](05-garbage-collectors.md#54-g1-gc--garbage-first)
- [5.5 ZGC](05-garbage-collectors.md#55-zgc--z-garbage-collector)
- [5.6 Shenandoah GC](05-garbage-collectors.md#56-shenandoah-gc)
- [5.7 Epsilon GC](05-garbage-collectors.md#57-epsilon-gc)
- [5.8 Comparison Table](05-garbage-collectors.md#58-quick-comparison-table)

### [06 — GC Tuning](06-gc-tuning.md)
- [6.1 Selecting a Collector](06-gc-tuning.md#61-selecting-a-collector)
- [6.2 Heap Sizing Flags](06-gc-tuning.md#62-heap-sizing-flags)
- [6.3 Generation Sizing](06-gc-tuning.md#63-generation-sizing)
- [6.4 G1 Tuning Flags](06-gc-tuning.md#64-g1-tuning-flags)
- [6.5 ZGC Tuning Flags](06-gc-tuning.md#65-zgc-tuning-flags)
- [6.6 GC Logging](06-gc-tuning.md#66-gc-logging)
- [6.7 Heap Sizing Formulas](06-gc-tuning.md#67-heap-sizing-formulas)

### [07 — GC Monitoring Tools](07-gc-monitoring-tools.md)
- [7.1 jstat](07-gc-monitoring-tools.md#71-jstat)
- [7.2 jcmd](07-gc-monitoring-tools.md#72-jcmd)
- [7.3 jmap](07-gc-monitoring-tools.md#73-jmap)
- [7.4 VisualVM](07-gc-monitoring-tools.md#74-visualvm)
- [7.5 GCEasy & GCViewer](07-gc-monitoring-tools.md#75-gceasy--gcviewer)
- [7.6 Java Flight Recorder (JFR)](07-gc-monitoring-tools.md#76-java-flight-recorder-jfr--mission-control-jmc)
- [7.7 Reading GC Logs](07-gc-monitoring-tools.md#77-reading-gc-logs)

### [08 — Best Practices](08-best-practices.md)
- [8.1 Heap Configuration](08-best-practices.md#81-heap-configuration)
- [8.2 Collector Selection](08-best-practices.md#82-collector-selection)
- [8.3 Allocation Patterns](08-best-practices.md#83-allocation-patterns)
- [8.4 Common Pitfalls](08-best-practices.md#84-common-pitfalls)
- [8.5 GC-Friendly Code Patterns](08-best-practices.md#85-gc-friendly-code-patterns)

### [09 — 100 Interview Questions](09-interview-questions.md)
- [Q1–Q15: GC Fundamentals](09-interview-questions.md#gc-fundamentals-q1q15)
- [Q16–Q30: JVM Memory & Generations](09-interview-questions.md#jvm-memory--generations-q16q30)
- [Q31–Q50: Garbage Collectors](09-interview-questions.md#garbage-collectors-q31q50)
- [Q51–Q65: GC Tuning & Configuration](09-interview-questions.md#gc-tuning--configuration-q51q65)
- [Q66–Q80: GC Monitoring & Debugging](09-interview-questions.md#gc-monitoring--debugging-q66q80)
- [Q81–Q100: Advanced & Internals](09-interview-questions.md#advanced--internals-q81q100)

---

[← Back to README](README.md) | [Start Reading → What is GC?](01-what-is-gc.md)

---

*Last updated: 2026 | Java 21 LTS*
