# 09 — 100 Interview Questions: Java Garbage Collection

> 100 GC interview questions with complete answers. Difficulty: 🟡 Moderate | 🔴 Complex.

---

**Navigation**

[← Best Practices](08-best-practices.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md)

---

## Table of Contents

- [GC Fundamentals (Q1–Q15)](#gc-fundamentals-q1q15)
- [JVM Memory & Generations (Q16–Q30)](#jvm-memory--generations-q16q30)
- [Garbage Collectors (Q31–Q50)](#garbage-collectors-q31q50)
- [GC Tuning & Configuration (Q51–Q65)](#gc-tuning--configuration-q51q65)
- [GC Monitoring & Debugging (Q66–Q80)](#gc-monitoring--debugging-q66q80)
- [Advanced & Internals (Q81–Q100)](#advanced--internals-q81q100)

---

## GC Fundamentals (Q1–Q15)

---

**Q1 🟡** What is Garbage Collection? Why does Java need it?

> **Answer:** Garbage Collection is the automatic reclamation of heap memory occupied by objects that are no longer reachable by the application. Java needs it because manual memory management (as in C/C++) is error-prone — developers can forget to free memory (leaks), free memory too early (dangling pointers), or free it twice (double-free crashes). GC eliminates these classes of bugs by automatically tracking object reachability and reclaiming unreachable objects.

---

**Q2 🟡** When does an object become eligible for Garbage Collection?

> **Answer:** An object becomes eligible for GC when it is no longer **reachable** from any GC root through any chain of strong references. GC roots include: active thread stack frames, static fields, JNI global references, and active monitor objects. Notably, Java uses **tracing GC** (not reference counting), so cyclic references between two unreachable objects are handled correctly — both are collected.

---

**Q3 🟡** What are GC Roots? List at least 5 types.

> **Answer:** GC Roots are the starting points of the reachability graph — any object reachable from a root is considered live. Types include: (1) **Local variables** on active thread call stacks, (2) **Static fields** of loaded classes, (3) **Active Java threads** themselves, (4) **JNI global references** held by native code, (5) **Objects held by class loaders**, (6) **Monitor objects** held in synchronization, (7) **JVM internal references** (interned strings, exception objects).

---

**Q4 🟡** What is the difference between Minor GC, Major GC, and Full GC?

> **Answer:** **Minor GC** collects only the Young Generation (Eden + Survivor spaces). It is frequent and fast (1–50ms). **Major GC** collects the Old Generation — usually triggered when Old Gen fills up. **Full GC** collects all regions: Young + Old + Metaspace. Full GC is the most expensive and includes compaction. In G1, "Major GC" is typically done concurrently; "Full GC" is the single-threaded STW fallback.

---

**Q5 🟡** What is a Stop-The-World (STW) pause? Why is it necessary?

> **Answer:** A STW pause halts all application threads while GC work is performed. It is necessary because most GC operations (marking, copying, compaction) require a consistent snapshot of the heap — if application threads keep modifying references while the GC traverses the object graph, the GC could miss live objects (collecting live data) or see dead objects as live (memory leak). Modern collectors like ZGC minimize STW to < 1ms by doing most work concurrently but still require brief STW pauses for safepoint operations.

---

**Q6 🟡** What is the Weak Generational Hypothesis and how does it influence GC design?

> **Answer:** The Weak Generational Hypothesis observes that **most objects die young** — empirically, 80–98% of objects become unreachable within their first few GC cycles. This influences GC design by dividing the heap into generations: Young Generation (collected frequently and cheaply using copy collection) and Old Generation (collected rarely). This ensures the GC spends most of its time collecting a small, mostly-dead Young Generation rather than scanning the entire heap.

---

**Q7 🟡** What are the four Java reference types and how do they interact with GC?

> **Answer:** (1) **Strong** — never collected while held; standard variable assignment. (2) **Soft** (`SoftReference`) — collected under memory pressure just before OOM; used for memory-sensitive caches. (3) **Weak** (`WeakReference`) — collected at the next GC cycle when no strong references exist; used for `WeakHashMap`, canonicalization caches. (4) **Phantom** (`PhantomReference`) — the referent is always inaccessible (`get()` returns null); the Reference is enqueued after finalization for post-GC cleanup hooks.

---

**Q8 🟡** Can you force Garbage Collection in Java? Should you?

> **Answer:** You can **suggest** GC with `System.gc()`, but the JVM may ignore it (it's just a hint). You should **not** call it in application code because: (1) it may trigger a Full GC causing multi-second STW pauses, (2) modern GCs are better at deciding when to collect, (3) it can interfere with GC heuristics and reduce throughput. Use `-XX:+DisableExplicitGC` to ignore all `System.gc()` calls, or `-XX:+ExplicitGCInvokesConcurrent` to make them trigger concurrent GC instead of Full GC.

---

**Q9 🟡** What is `finalize()`? Why is it deprecated?

> **Answer:** `Object.finalize()` was a method called by the GC before collecting an object, intended for native resource cleanup. It was deprecated in Java 9 and removed in Java 18 because: (1) **unpredictable timing** — no guarantee when (or if) it runs, (2) **performance impact** — finalizable objects survive at least one extra GC cycle, creating a queue that can back up, (3) **can resurrect objects** — calling `finalize()` can re-assign `this`, preventing collection, (4) **dangerous** — `finalize()` can run in a concurrent thread while the application is still running. Use `Cleaner` (Java 9+) or try-with-resources instead.

---

**Q10 🟡** What is Metaspace? How does it differ from PermGen?

> **Answer:** Metaspace (Java 8+) stores class metadata. Key differences from PermGen: (1) **Location** — PermGen was on the JVM heap; Metaspace is in native OS memory. (2) **Size** — PermGen had a fixed default cap (often 64–256 MB), causing frequent `OutOfMemoryError: PermGen space`; Metaspace grows dynamically. (3) **GC** — PermGen was collected with Full GC; Metaspace is also collected with Full GC but is generally more efficient. (4) **Overflow** — Metaspace can exhaust native memory if not capped with `-XX:MaxMetaspaceSize`.

---

**Q11 🟡** What is a memory leak in Java? Give a common example.

> **Answer:** A Java memory leak occurs when objects are no longer needed by the application but are still reachable from GC roots — the GC correctly keeps them alive because the code holds references. Common example: a static `Map` used as a cache with no eviction policy — entries are added but never removed, growing indefinitely. Other examples: event listeners not deregistered, `ThreadLocal` values not removed in thread pools, and inner class instances holding a reference to the outer class.

---

**Q12 🟡** What is object promotion (tenuring)?

> **Answer:** Object promotion (tenuring) is the process of moving a long-lived object from Young Generation to Old Generation. Each object has an age counter (in its object header) that increments with each Minor GC it survives. When age reaches `MaxTenuringThreshold` (default 15), the object is promoted to Old Gen. The JVM can also promote early via **dynamic tenuring** (when Survivor space is > 50% full) or **space overflow** (when Survivor is completely full).

---

**Q13 🟡** What happens if there is no space in the Survivor space during Minor GC?

> **Answer:** If live objects from Eden + the active Survivor space don't fit in the empty Survivor space, a **premature promotion** occurs — all surviving objects are promoted directly to Old Generation, regardless of their age. This leads to Old Gen filling up faster than expected, triggering more frequent Major GC or Full GC cycles. Fix: increase Survivor space size (`-XX:SurvivorRatio` or `-Xmn`), or tune `MaxTenuringThreshold`.

---

**Q14 🟡** What is the difference between `WeakHashMap` and `HashMap`?

> **Answer:** In a `WeakHashMap`, **keys are stored as weak references**. When a key has no strong references outside the map (i.e., the GC can collect the key object), the corresponding entry is automatically removed from the map at the next GC. `HashMap` uses strong references for keys — entries are never automatically removed. `WeakHashMap` is useful for caches keyed on external objects (e.g., widget → metadata) where the cache entry should disappear when the key object is no longer used elsewhere.

---

**Q15 🟡** What is the card table and why does GC need it?

> **Answer:** The card table is a JVM data structure that divides Old Generation memory into 512-byte cards. Whenever a reference field in an Old Gen object is updated to point to a Young Gen object, that card is marked "dirty." During Minor GC, instead of scanning all of Old Gen to find references into Young Gen (which would be very slow), the GC only scans dirty cards. A write barrier (JIT-injected code on every reference write) maintains the card table automatically.

---

## JVM Memory & Generations (Q16–Q30)

---

**Q16 🟡** Describe the JVM heap structure and its regions.

> **Answer:** The JVM heap is divided into: (1) **Young Generation** — where all new objects are allocated; subdivided into Eden (~80%) and two Survivor spaces S0/S1 (~10% each). (2) **Old Generation (Tenured)** — long-lived objects promoted from Young Gen; larger and collected less frequently. (3) **Metaspace** (Java 8+) — off-heap; stores class metadata, method bytecodes, constant pool. Additionally: thread stacks, JIT Code Cache, and direct buffers live outside the GC-managed heap.

---

**Q17 🟡** Why are there two Survivor spaces (S0 and S1)?

> **Answer:** Copy collection (used in Young Gen) requires a source and a destination. After Minor GC, live objects are copied from Eden + active Survivor (source) to the empty Survivor (destination). This ping-pong guarantees that the destination space is always empty and contiguous — enabling pointer-bump allocation with no fragmentation. Only one Survivor space is "active" at any time; the other is always kept empty as the copy destination.

---

**Q18 🟡** What is TLAB (Thread-Local Allocation Buffer)? What problem does it solve?

> **Answer:** A TLAB is a private chunk of Eden space assigned to each thread. Object allocation within a TLAB is lock-free — the thread simply bumps a pointer. Without TLABs, all threads would contend on a shared pointer to Eden, requiring synchronization on every object creation. TLABs make allocation essentially free (~1 ns per object) and improve cache locality (objects allocated by the same thread are in the same memory area).

---

**Q19 🟡** What is `-XX:NewRatio` and how does it affect GC behaviour?

> **Answer:** `-XX:NewRatio=N` sets the ratio of Old Gen to Young Gen size. `NewRatio=2` (default) means Old:Young = 2:1 → Young Gen = 33% of total heap. A larger Young Gen (lower ratio) means: fewer Minor GCs (Eden fills less frequently) but each Minor GC takes longer (more objects to scan/copy). A smaller Young Gen: more frequent Minor GCs with shorter pauses. Tune based on allocation rate and object lifetime profile.

---

**Q20 🟡** What is `SurvivorRatio` and when would you change it?

> **Answer:** `-XX:SurvivorRatio=N` sets the ratio of Eden to each Survivor space. `SurvivorRatio=8` (default) means Eden:S0:S1 = 8:1:1 → Eden = ~80% of Young Gen. Increase Survivor space (lower ratio, e.g., `SurvivorRatio=4`) when you see premature promotion — objects being promoted too early because Survivor fills up. You'd notice this via `jstat` showing Old Gen growing steadily and `PrintTenuringDistribution` showing objects at age 1–2 being promoted.

---

**Q21 🟡** What is the difference between `-Xms` and `-Xmx`?

> **Answer:** `-Xms` sets the **initial (minimum) heap size** allocated at JVM startup. `-Xmx` sets the **maximum heap size** the JVM will not exceed. If `-Xms < -Xmx`, the heap can grow as needed (with occasional resize pauses). In production, setting `-Xms == -Xmx` commits all memory at startup, eliminating heap resize pauses and ensuring the JVM won't fail to grow under load.

---

**Q22 🟡** What causes `OutOfMemoryError: Java heap space`?

> **Answer:** This OOM means the JVM cannot allocate an object because there is no more room in the heap (after GC attempts). Common causes: (1) **Memory leak** — objects accumulate and are never GC'd (static collections, listener leaks), (2) **Heap too small** for the workload (`-Xmx` too low), (3) **Memory spike** — sudden burst allocation beyond configured heap size, (4) **Large array allocations** (e.g., loading an entire large file into a `byte[]`), (5) **Excessive caching** without eviction limits.

---

**Q23 🟡** What causes `OutOfMemoryError: Metaspace`?

> **Answer:** Metaspace OOM means the JVM ran out of native memory for class metadata. Causes: (1) **Class loading leak** — frameworks generate new classes without releasing them (CGLIB proxies not cached, Groovy scripts loaded in a loop, reflection-based serialization creating classes for each type), (2) **No `MaxMetaspaceSize` cap** — Metaspace grows until native memory exhausted, (3) **Too many deployed applications** sharing a JVM (application servers) without enough Metaspace.

---

**Q24 🟡** What is a Humongous object in G1 GC?

> **Answer:** In G1 GC, a Humongous object is any object larger than **50% of the G1 region size**. For example, with 4 MB regions, objects > 2 MB are Humongous. These objects are allocated directly in contiguous Old Gen regions (bypassing Young Gen), which: (1) counts against Old Gen occupancy immediately — triggering concurrent marking early, (2) can cause fragmentation if Humongous regions don't align, (3) cannot be collected in Young GC — only in Mixed GC or Full GC. Fix: increase region size (`-XX:G1HeapRegionSize=32m`) or avoid large allocations.

---

**Q25 🟡** What is dynamic tenuring?

> **Answer:** Dynamic tenuring is a JVM optimization where `MaxTenuringThreshold` is automatically lowered when Survivor space becomes too full. If the total size of objects at a given age exceeds 50% of Survivor space capacity, the JVM promotes all objects at that age and above, even if they haven't reached the configured max threshold. This prevents Survivor overflow and is printed in GC logs as "new threshold X (max Y)" where X < Y.

---

**Q26 🔴** Why does object allocation in Java typically happen in Eden and not directly in Old Gen?

> **Answer:** Eden allocation is **extremely fast** because of TLAB bump-pointer allocation (~1 ns). More importantly, the **Weak Generational Hypothesis** tells us most objects die young — allocating in Eden means they are collected in cheap Minor GC (copy collection, no fragmentation). If all objects went to Old Gen, the Old Gen would fill rapidly, requiring expensive mark-compact cycles. Eden acts as a "nursery" — frequent, cheap collection of mostly-dead objects keeps Old Gen clean and full GC rare.

---

**Q27 🔴** What is a promotion failure and how can you prevent it?

> **Answer:** Promotion failure (in G1 called "evacuation failure") occurs when the GC cannot find enough space in Old Gen to promote objects from Young Gen. This forces a Full GC (STW) to compact Old Gen and make room. Prevention: (1) **Size Old Gen adequately** — increase `-Xmx`, (2) **Lower IHOP** (`-XX:InitiatingHeapOccupancyPercent`) so G1 starts concurrent marking earlier, (3) **Reduce allocation rate** — identify and fix allocation hotspots, (4) **Increase Young Gen size** — fewer objects survive to promotion per cycle.

---

**Q28 🔴** Explain what happens internally during a Minor GC, step by step.

> **Answer:** (1) **STW pause** — all application threads halted at safepoints. (2) **Root scanning** — scan GC root references pointing into Young Gen. (3) **Card table scan** — find Old Gen references into Young Gen by checking dirty cards. (4) **Mark live objects** — trace from roots through Young Gen object graph; mark survivors. (5) **Copy** — live objects from Eden + active Survivor are copied to the empty Survivor space with age incremented. Objects reaching age threshold are promoted to Old Gen. (6) **Clear** — Eden and old Survivor cleared (just pointer reset — very fast). (7) **Resume** — application threads resume.

---

**Q29 🔴** Why does Java need a card table even though it has a generational hypothesis?

> **Answer:** The generational hypothesis says most objects die young, so Minor GC focuses on Young Gen. But Old Gen objects CAN reference Young Gen objects (e.g., a cached entity in Old Gen references a newly created DTO). During Minor GC, these cross-generational references must be treated as GC roots for Young Gen objects — otherwise live Young Gen objects referenced only from Old Gen would be incorrectly collected. The card table efficiently identifies which Old Gen regions have been recently modified to contain Young Gen references, avoiding a full Old Gen scan.

---

**Q30 🔴** What is the "remembered set" in G1 GC?

> **Answer:** In G1, every region has a **Remembered Set (RSet)** — a hash table recording references INTO that region from other regions. When G1 collects a region, it uses the RSet to find all incoming references without scanning the entire heap. Write barriers maintain the RSets: every reference write is intercepted, and if the write creates a cross-region reference, the source address is added to the target region's RSet. RSets enable G1's key capability: selective collection of individual regions, not entire generations.

---

## Garbage Collectors (Q31–Q50)

---

**Q31 🟡** Name all 7 JVM garbage collectors. For each, state one key characteristic.

> **Answer:** (1) **Serial GC** — single-threaded, full STW, best for small heaps. (2) **Parallel GC** — multi-threaded STW, highest throughput, default Java 8. (3) **CMS** — concurrent mark-sweep, deprecated/removed, no compaction. (4) **G1 GC** — region-based, concurrent marking, default Java 9+. (5) **ZGC** — sub-millisecond pauses, concurrent compaction via colored pointers, scales to 16 TB. (6) **Shenandoah** — concurrent compaction via forwarding pointers, OpenJDK only. (7) **Epsilon** — no-op collector, never collects, for testing.

---

**Q32 🟡** Which GC is the default in Java 8? And in Java 9+?

> **Answer:** Java 8: **Parallel GC** (also called the Throughput Collector) is the default on server-class machines. Java 9+: **G1 GC** is the default on server-class machines (more than 1 CPU and more than 1792 MB RAM). For client-class machines (single CPU, limited RAM), Serial GC remains the default in both.

---

**Q33 🟡** What are the main phases of G1 GC?

> **Answer:** G1 phases: (1) **Young GC (STW)** — evacuates Eden + Survivor regions, promotes to Old. (2) **Concurrent Marking (concurrent)** — marks live objects in Old regions; triggered when Old Gen > IHOP%. (3) **Remark (short STW)** — finalizes marking; processes SATB buffers. (4) **Cleanup (short STW)** — identifies empty regions for immediate reclaim; updates accounting. (5) **Mixed GC (STW)** — collects Young + selected Old regions with most garbage. (6) **Full GC (STW fallback)** — single-threaded compaction when concurrent can't keep up.

---

**Q34 🟡** Why was CMS GC deprecated and removed?

> **Answer:** CMS was deprecated in Java 9 and removed in Java 14 for several reasons: (1) **No compaction** — CMS uses mark-sweep, creating fragmentation. When fragmented Old Gen can't fit promoted objects, a **Concurrent Mode Failure** triggers a single-threaded Full GC with worse pause than just using Parallel GC. (2) **G1 GC** achieves better or equal latency **with** compaction, making CMS obsolete. (3) **Complex tuning** — CMS had many configuration knobs and failure modes. G1 is simpler and more predictable.

---

**Q35 🟡** What is the difference between `-XX:+UseParallelGC` and `-XX:+UseG1GC`?

> **Answer:** **Parallel GC** uses multiple GC threads but is fully **stop-the-world** for all phases. It achieves maximum throughput but can have long pauses (seconds for large heaps). It collects the entire heap at once. **G1 GC** uses **concurrent marking** alongside the application, then performs targeted STW collection of regions with the most garbage (Mixed GC). G1 targets a configurable pause time (`MaxGCPauseMillis`) and selects how much Old Gen to collect per cycle to stay within that target. G1 trades slightly lower peak throughput for much lower pause times.

---

**Q36 🟡** When would you choose ZGC over G1?

> **Answer:** Choose ZGC when: (1) **Pause time SLA < 10ms** — ZGC achieves sub-millisecond pauses; G1 typically 50–200ms. (2) **Very large heaps** (> 100 GB) — G1's concurrent marking scales poorly; ZGC designed for up to 16 TB. (3) **Real-time or latency-sensitive applications** — trading platforms, gaming, ad-serving systems where latency spikes are unacceptable. Trade-off: ZGC has slightly higher throughput overhead (~5–15%) vs G1 due to load barriers.

---

**Q37 🟡** What is Epsilon GC? When would you use it?

> **Answer:** Epsilon is a no-op GC that allocates memory but never collects. The JVM exits with OOM when heap is exhausted. Uses: (1) **Performance benchmarking** — measure pure allocation overhead without GC interference. (2) **Short-lived tools** with known bounded allocation (e.g., a batch importer that terminates in 30 seconds). (3) **Testing GC overhead** — compare app performance with vs without GC to quantify GC cost. Never use in long-running production services.

---

**Q38 🔴** How does ZGC achieve sub-millisecond pause times?

> **Answer:** ZGC achieves < 1ms pauses through: (1) **Colored pointers** — GC state (marked, remapped, finalizable) encoded in the 64-bit pointer itself, not in object headers — allows concurrent state checking. (2) **Load barriers** — JIT-injected code on every reference READ that detects stale pointers and heals them inline. This allows concurrent relocation — objects can be moved while the app runs; the next read fixes up the reference. (3) **Only 3 STW phases** — all < 1ms: initial mark (scan roots), remark (finalize marking), relocate start (scan roots for relocation). Everything else is concurrent.

---

**Q39 🔴** What is the difference between concurrent marking and parallel marking?

> **Answer:** **Parallel marking** runs multiple GC threads simultaneously — but all application threads are stopped (STW). It is faster than single-threaded marking but still pauses the app. **Concurrent marking** runs GC threads at the same time as application threads — the app keeps running. This eliminates the marking pause but creates complexity: the app can modify the object graph during marking (handled by write barriers like SATB or incremental update). ZGC and G1 use concurrent marking for most of the work.

---

**Q40 🔴** What is SATB (Snapshot-At-The-Beginning) and why does G1 need it?

> **Answer:** SATB is G1's write barrier for concurrent marking. At the start of concurrent marking, a logical "snapshot" of the heap is taken. If an application thread modifies a reference during concurrent marking (e.g., nulling a reference to object X), SATB records the **old value** (X) in a SATB buffer. During the Remark STW phase, SATB buffers are drained — old references are re-marked as live. This ensures: even if the app changes the graph to hide X from the GC's view, SATB guarantees X was alive at marking start and won't be falsely collected (conservative correctness).

---

**Q41 🔴** Explain Shenandoah's Brooks forwarding pointer mechanism.

> **Answer:** Every object in Shenandoah has an extra **forwarding pointer** word prepended to its header. In normal state, this pointer points back to the object itself (identity). During concurrent evacuation: (1) A copy is made at the new location. (2) The forwarding pointer in the old object is atomically updated to point to the new location. (3) All subsequent accesses to the old reference transparently follow the forwarding pointer to the new location. (4) During concurrent reference update phase, all references are updated to point directly to the new location, and the old objects become garbage.

---

**Q42 🔴** What is Generational ZGC (Java 21)? How does it improve on single-generation ZGC?

> **Answer:** Generational ZGC adds separate Young and Old generations to ZGC. This improves on single-generation ZGC by: (1) **Higher efficiency** — young objects are collected more frequently and cheaply (most die young); single-gen ZGC scanned the entire heap equally. (2) **Lower overhead** — fewer objects in the concurrent marking/relocation phase per cycle. (3) **Reduced allocation stalls** — more proactive collection frees space before allocation stalls occur. Enabled by default in Java 21 (`-XX:+UseZGC` automatically uses generational mode).

---

**Q43 🔴** What happens when G1 GC falls back to a Full GC?

> **Answer:** G1 Full GC occurs when concurrent GC cannot keep up with promotion demand — Old Gen fills faster than G1's concurrent cycle can reclaim it. The Full GC: (1) Is **single-threaded** (in Java 8/G1 original), or **parallel** (Java 10+, `-XX:+UseG1GC` uses parallel Full GC threads). (2) **Stop-The-World** for the entire duration. (3) Collects all generations: Young + Old + Metaspace. (4) Performs full compaction. For a large heap, this can take seconds to minutes. Always investigate root cause: is IHOP too high? Allocation rate too high? Humongous allocations?

---

**Q44 🔴** What is a "concurrent mode failure" in CMS?

> **Answer:** Concurrent Mode Failure in CMS occurs when Old Gen fills up **before** the concurrent sweep phase completes. CMS starts sweeping concurrently while the app continues allocating and promoting. If Old Gen is exhausted before the sweep frees enough space: (1) CMS aborts the concurrent cycle. (2) Falls back to a **single-threaded Serial Full GC** with compaction. (3) This can produce pauses **worse than just using Parallel GC** for large heaps. Prevention: start CMS earlier (lower `-XX:CMSInitiatingOccupancyFraction`) or switch to G1.

---

**Q45 🔴** Compare G1 Mixed GC with CMS Concurrent Sweep. Which is better and why?

> **Answer:** **G1 Mixed GC** (STW, parallel) collects Young + selected Old regions in a targeted way; it also **compacts** the collected regions — no fragmentation. **CMS Concurrent Sweep** runs concurrently (lower pause) but only sweeps — no compaction. Over time, CMS Old Gen becomes fragmented, eventually causing Concurrent Mode Failure. G1 Mixed GC's targeted collection with compaction produces **predictable, low pauses** without fragmentation risk, making it superior. This is the primary reason CMS was removed.

---

**Q46 🔴** What is the G1 region size and how should you configure it?

> **Answer:** G1 divides the heap into equal-sized regions of 1, 2, 4, 8, 16, or 32 MB (always a power of 2). Default: `heap_size / 2048`, rounded to the nearest power of 2. Humongous threshold = 50% of region size. Configure larger regions (`-XX:G1HeapRegionSize=32m`) when: you have many large object allocations to raise the Humongous threshold; or when the default produces too many regions (> 2048), increasing GC overhead. Smaller regions: finer-grained collection selection for Mixed GC.

---

**Q47 🟡** What does `-XX:MaxGCPauseMillis` actually control in G1?

> **Answer:** `-XX:MaxGCPauseMillis` is a **soft target**, not a hard guarantee. G1 uses this value to decide how many Old Generation regions to collect in each Mixed GC cycle. To stay within the pause target, G1 may collect fewer Old regions per cycle (spreading the work over more cycles). It does NOT prevent Young GC pauses from exceeding the target if the Young Gen is too large. G1 will violate the target if: all regions are equally garbage-dense, or a Full GC is triggered.

---

**Q48 🟡** What is IHOP (`InitiatingHeapOccupancyPercent`) in G1?

> **Answer:** IHOP (`-XX:InitiatingHeapOccupancyPercent=45`) controls when G1 starts a concurrent marking cycle. When Old Gen occupancy exceeds IHOP% of the total heap, G1 triggers concurrent marking. Default is 45%. **Too high:** G1 starts marking too late → Old Gen fills before marking completes → concurrent mode failure → Full GC. **Too low:** G1 marks too frequently → unnecessary CPU overhead. For applications with bursty promotions, lowering IHOP (e.g., to 35%) gives G1 more lead time to handle sudden growth.

---

**Q49 🔴** What is the write barrier overhead of different GC algorithms?

> **Answer:** Write barriers are injected by the JIT on every reference write. Overhead varies: **Parallel/Serial GC** — simple card table dirty mark (~1 instruction). **G1** — card table + SATB barrier (check if concurrent marking is active, enqueue old value if so) — ~3–5 instructions normally, more during concurrent marking. **ZGC** — load barrier (on every READ, not write) — colored pointer check + possible slow path — ~3 instructions normal case. **Shenandoah** — both read and write barriers for forwarding pointers. Modern CPUs execute these in the instruction pipeline with minimal real-world throughput impact (~1–5%).

---

**Q50 🔴** What is the maximum heap size supported by each major GC?

> **Answer:** **Serial GC** — practical limit ~1 GB (single-thread GC pause scales poorly). **Parallel GC** — up to ~100 GB (parallel STW still causes seconds-long pauses). **G1 GC** — up to ~100 GB recommended; JVM tested up to several TB but pauses get longer. **ZGC** — officially supports up to **16 TB** with sub-millisecond pauses even at 1 TB+. **Shenandoah** — no hard limit; sub-10ms pauses regardless of heap size. **Epsilon** — bounded by available memory (no collection).

---

## GC Tuning & Configuration (Q51–Q65)

---

**Q51 🟡** What JVM flags would you set for a production web service?

> **Answer:**
> ```bash
> -XX:+UseG1GC -Xms8g -Xmx8g
> -XX:MaxGCPauseMillis=200
> -XX:MaxMetaspaceSize=512m
> -XX:+HeapDumpOnOutOfMemoryError
> -XX:HeapDumpPath=/dumps/
> -Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags:filecount=10,filesize=20m
> ```
> Key points: fixed heap (Xms==Xmx), capped Metaspace, GC logging, and OOM heap dump capture.

---

**Q52 🟡** Why should `-Xms` equal `-Xmx` in production?

> **Answer:** When `-Xms < -Xmx`, the JVM grows the heap dynamically as demand increases. Heap growth requires requesting pages from the OS and committing them — this creates **unpredictable pause spikes** when the heap expands under load. Additionally, when load drops, the JVM may shrink the heap and return memory to the OS, only to re-request it during the next load spike. Setting `-Xms == -Xmx` commits all memory at startup — no resize pauses, predictable GC behaviour, and no risk of failing to grow during traffic spikes.

---

**Q53 🟡** What does `-XX:+DisableExplicitGC` do and when would you use it?

> **Answer:** `-XX:+DisableExplicitGC` causes the JVM to silently ignore all `System.gc()` calls. Use when: (1) A third-party library calls `System.gc()` — this can trigger unexpected Full GC pauses in production. (2) RMI's distributed GC calls `System.gc()` every minute by default. Alternative: `-XX:+ExplicitGCInvokesConcurrent` makes `System.gc()` trigger a concurrent GC (G1/ZGC) instead of a Full GC — safer than ignoring it completely.

---

**Q54 🟡** What is `-XX:+AlwaysPreTouch` and when is it important?

> **Answer:** `-XX:+AlwaysPreTouch` instructs the JVM to access (fault) every heap page at startup. Without it, the OS uses lazy allocation — heap pages are only physically committed on first access. During warm-up, the first accesses to new heap pages cause page faults — OS interrupts that add latency. With `AlwaysPreTouch`, all page faults happen at startup (increasing start time), making GC pauses more consistent in production. Important for latency-sensitive services and when using ZGC (which aims for < 1ms pauses — page faults would violate this).

---

**Q55 🟡** What is `GCTimeRatio` and how does it affect Parallel GC?

> **Answer:** `-XX:GCTimeRatio=N` sets the desired ratio of application time to GC time. `GCTimeRatio=99` means 99:1 ratio → max 1% of CPU in GC (default). `GCTimeRatio=19` means 95:5 ratio → allows 5% of CPU in GC. Parallel GC uses this to auto-size the Young Gen via `UseAdaptiveSizePolicy`: if GC overhead exceeds the ratio, the JVM grows the heap or adjusts generation sizes to reduce GC frequency. For batch jobs where throughput matters: lower ratio (allow more GC %).

---

**Q56 🟡** How do you configure G1 GC for a large heap (64 GB)?

> **Answer:**
> ```bash
> -XX:+UseG1GC -Xms64g -Xmx64g
> -XX:G1HeapRegionSize=32m          # Larger regions for large heap (fewer regions)
> -XX:MaxGCPauseMillis=200          # Keep target
> -XX:InitiatingHeapOccupancyPercent=35  # Start marking earlier
> -XX:G1NewSizePercent=10           # Don't over-provision Young Gen
> -XX:G1MaxNewSizePercent=30
> -XX:ConcGCThreads=8               # More concurrent marking threads
> -XX:ParallelGCThreads=16          # More STW GC threads
> ```

---

**Q57 🔴** What is `SoftMaxHeapSize` in ZGC?

> **Answer:** `-XX:SoftMaxHeapSize=N` sets a **soft heap limit** for ZGC. ZGC proactively tries to keep the heap below this size, doing more frequent GC to stay within it. If the app needs more memory (sudden spike), ZGC can still grow up to `-Xmx`. This is useful for: (1) Running alongside other memory consumers in a container — soft limit leaves headroom. (2) Triggering proactive GC to return memory to the OS faster. Typically set to ~85% of `-Xmx`.

---

**Q58 🔴** What is `InitiatingHeapOccupancyPercent` and how do you tune it?

> **Answer:** IHOP (`-XX:InitiatingHeapOccupancyPercent=45`) is the % of the total heap at which G1 triggers concurrent marking. Tuning: (1) **Monitor concurrent mode failures** in GC log — if you see Full GC due to promotion failure, lower IHOP (start marking earlier). (2) **Monitor GC overhead** — if marking runs too frequently (many consecutive concurrent cycles), raise IHOP. Target: G1 should complete concurrent marking with enough buffer left to absorb promotions until Mixed GC cleans up. Start with 35% for high-allocation apps, 45% for stable apps.

---

**Q59 🔴** How do you configure JVM flags for a containerized Java service (Docker/Kubernetes)?

> **Answer:**
> ```bash
> # Java 11+ automatically respects container memory limits
> -XX:+UseContainerSupport        # Enabled by default in Java 11+
> -XX:MaxRAMPercentage=75.0       # Use 75% of container memory as max heap
> -XX:InitialRAMPercentage=50.0   # Start heap at 50%
> -XX:MaxMetaspaceSize=256m       # Prevent Metaspace from consuming remaining memory
> -XX:+UseG1GC                    # Or ZGC for latency
> -XX:+HeapDumpOnOutOfMemoryError
> -XX:HeapDumpPath=/tmp/
> ```
> Important: set container memory limits (e.g., `resources.limits.memory: 4Gi` in k8s). Without limits, `MaxRAMPercentage` falls back to host memory — the heap will be very large.

---

**Q60 🔴** What is `G1ReservePercent` and when would you increase it?

> **Answer:** `-XX:G1ReservePercent=10` reserves 10% of the G1 heap as a "to-space reserve" buffer for promotions during GC. If during a Mixed GC, more objects need to be promoted than can fit in the available free regions, an evacuation failure occurs. Increasing `G1ReservePercent` (e.g., to 20%) gives more buffer — at the cost of less usable heap. Increase when you see "to-space exhausted" or evacuation failure messages in GC logs.

---

**Q61 🟡** What does `-XX:+HeapDumpOnOutOfMemoryError` do? Why is it essential in production?

> **Answer:** This flag instructs the JVM to **automatically capture a heap dump** (`.hprof` file) when an `OutOfMemoryError` occurs. Essential in production because: (1) OOM crashes are often transient and hard to reproduce. (2) The heap dump contains a snapshot of all live objects at the moment of OOM — invaluable for identifying the leak (what objects are consuming memory, what's holding references to them). Without this flag, you'd have no evidence to analyse after the JVM crash. Always pair with `-XX:HeapDumpPath=/dumps/`.

---

**Q62 🟡** What is the difference between `-Xmn` and `-XX:NewRatio`?

> **Answer:** `-Xmn` sets an **absolute fixed size** for Young Generation (e.g., `-Xmn2g` = 2 GB Young Gen always). `-XX:NewRatio=N` sets a **ratio** of Old:Young. If both are specified, `-Xmn` takes precedence. Use `-Xmn` when you know exactly how large Young Gen should be. Use `-XX:NewRatio` when you want proportional sizing that scales with total heap changes. Note: with Adaptive Size Policy enabled (default with Parallel GC), both may be overridden by the JVM's auto-tuning.

---

**Q63 🔴** How do you tune GC for a microservice with very high allocation rate (e.g., 1 GB/s)?

> **Answer:** (1) **Increase Young Gen** — a high allocation rate fills Eden quickly → frequent Minor GC. Larger Young Gen means longer intervals: `-XX:G1NewSizePercent=40 -XX:G1MaxNewSizePercent=60`. (2) **Use G1 or ZGC** — parallel STW collectors can't keep up at extreme allocation rates. (3) **Profile allocations with JFR** — identify top allocating classes and reduce unnecessary object creation (escape analysis, pooling, value types). (4) **Evaluate off-heap** for specific hot data structures. (5) Consider Java 21 virtual threads + ZGC — reduces thread-local allocation buffer waste.

---

**Q64 🔴** What `-Xss` controls and what is the risk of setting it too low?

> **Answer:** `-Xss` sets the **stack size per thread**. Default is typically 512 KB–1 MB. Reducing it (e.g., `-Xss256k`) allows more threads to fit in available memory. Risk of too-low stack size: `StackOverflowError` for deep call stacks (recursive methods, deep framework call chains, complex serialization/deserialization). In thread-heavy systems (thousands of threads), reducing `-Xss` from 1 MB to 256 KB can save gigabytes of stack memory — but test thoroughly with your application's deepest call stacks.

---

**Q65 🔴** What is Adaptive Size Policy in Parallel GC and when should you disable it?

> **Answer:** Adaptive Size Policy (`-XX:+UseAdaptiveSizePolicy`, default on with Parallel GC) automatically tunes Eden/Survivor/Old Gen sizes and `MaxTenuringThreshold` based on observed GC behaviour — trying to meet the `MaxGCPauseMillis` and `GCTimeRatio` goals. Disable it (`-XX:-UseAdaptiveSizePolicy`) when: (1) You have determined optimal sizes from profiling and want fixed sizes. (2) You see erratic heap sizing causing unpredictable GC behaviour. (3) You're running benchmarks and need deterministic sizing. Always set `-Xmn` or `-XX:NewRatio` explicitly when disabling.

---

## GC Monitoring & Debugging (Q66–Q80)

---

**Q66 🟡** How do you monitor GC activity on a live JVM?

> **Answer:** Multiple approaches: (1) **`jstat -gcutil <pid> 1000`** — live GC stats every second: Eden%, Old%, GC counts and times. (2) **GC logs** (`-Xlog:gc*:file=gc.log`) — comprehensive, low overhead, essential for post-incident analysis. (3) **VisualVM** with Visual GC plugin — real-time animated heap visualization. (4) **JMX metrics** — `java.lang:type=GarbageCollector` MBeans exposed via JMX; scraped by Prometheus JMX exporter. (5) **jcmd <pid> GC.heap_info** — on-demand heap snapshot.

---

**Q67 🟡** What does this `jstat` output tell you?
```
S0    S1    E     O     M    YGC  YGCT  FGC  FGCT  GCT
0.00 45.32 72.14 88.21 96.12 142  2.54  5    8.234  10.78
```

> **Answer:** This shows a **concerning** state: (1) **O = 88.21%** — Old Gen is nearly full and growing; risk of Full GC imminent. (2) **FGC = 5** — 5 Full GCs have occurred; significant pauses (8.2 seconds total). (3) **GCT = 10.78s** — 10.78 seconds of total GC time; if uptime is 300s, that's ~3.6% GC overhead. (4) **E = 72.14%** — Eden is 72% full; Minor GC will happen soon. Action: investigate Old Gen leak, increase heap, or lower IHOP.

---

**Q68 🟡** What is the difference between `jmap -histo` and `jmap -histo:live`?

> **Answer:** `jmap -histo <pid>` shows a histogram of ALL objects in the heap — including dead objects not yet collected. Fast but includes garbage. `jmap -histo:live <pid>` first triggers a **Full GC** to collect all dead objects, then shows only live objects. More accurate for leak analysis (shows exactly what's consuming heap) but causes a **Full GC pause** in production. For production: use `jmap -histo` first; switch to `:live` only during maintenance windows or non-peak hours.

---

**Q69 🟡** How do you detect a memory leak using GC tools?

> **Answer:** Step-by-step: (1) **Enable GC logging** — `-Xlog:gc*:file=gc.log`. (2) **Monitor with jstat** — watch if `O%` (Old Gen %) grows steadily over time without plateauing. (3) **Capture heap histograms** over time — `jmap -histo <pid>` at T+0, T+30min, T+60min; compare which classes are growing. (4) **Take heap dumps** — `jcmd <pid> GC.heap_dump /tmp/heap.hprof`. (5) **Analyze in Eclipse MAT** — run Leak Suspects report; inspect Dominator Tree; find what's holding large retained heaps. (6) Identify the GC root chain keeping objects alive.

---

**Q70 🟡** What is GCEasy and what does it analyze?

> **Answer:** GCEasy (gceasy.io) is a web-based GC log analyzer. Upload your GC log and it provides: (1) **Throughput %** — time spent in application vs GC. (2) **Pause time distribution** — p50/p95/p99/max pause times. (3) **Heap usage charts** — before/after GC over time. (4) **GC cause breakdown** — what triggered each GC cycle. (5) **Problem detection** — alerts on excessive Full GCs, long pauses, Metaspace growth, promotion failures. (6) **JVM flag recommendations** — tuning suggestions based on observed behavior.

---

**Q71 🟡** How do you use JFR (Java Flight Recorder) to analyse GC issues?

> **Answer:** (1) **Start recording**: `jcmd <pid> JFR.start name=gc duration=120s filename=/tmp/gc.jfr`. (2) **Open in JMC** (Mission Control): File → Open → gc.jfr. (3) **Key views**: (a) Memory → Heap tab: heap usage over time, GC events. (b) Memory → Allocation Pressure: top allocating classes and call stacks. (c) GC tab: pause time breakdown per phase. (d) Code → Hot Methods: find methods with high allocation rates. (4) For leaks: Memory → Old Object Sample shows long-lived objects and their allocation stack.

---

**Q72 🔴** You see frequent Full GCs in production. What is your systematic debugging approach?

> **Answer:** (1) **Check GC log cause** — what's triggering Full GC? "G1 Compaction Pause", "Metadata GC Threshold", "System.gc()", "Allocation Failure". (2) **Check Old Gen trend** — is O% steadily growing (leak) or just peaking under load? (3) **If Metaspace** — check with `jcmd <pid> VM.metaspace`; look for ClassLoader leaks. (4) **If promotion failure** — lower IHOP, increase heap. (5) **If allocation flood** — `jmap -histo:live` to find top live classes; JFR allocation profiling. (6) **Take heap dump** during/after Full GC — analyze in Eclipse MAT for leak suspects. (7) **Check for System.gc() calls** — add `-Xlog:gc+cause` to see all causes.

---

**Q73 🔴** How do you interpret a G1 GC log entry showing evacuation failure?

```
[45.123s][gc] GC(100) Pause Young (Normal) (G1 Evacuation Pause)
[45.123s][gc] GC(100)  To-space exhausted
[45.678s][gc] GC(100) Pause Young (Normal) (G1 Evacuation Pause) 6144M->6100M(8192M) 555ms
```

> **Answer:** "To-space exhausted" means G1 ran out of free regions to copy live objects into during the evacuation pause. The GC couldn't move all live objects from Eden → Survivor/Old. Causes: (1) Old Gen is nearly full — not enough free regions. (2) Promotion rate spike — sudden surge of long-lived objects. Fixes: (1) Increase `-Xmx` or `-XX:G1ReservePercent`. (2) Lower `-XX:InitiatingHeapOccupancyPercent` so concurrent marking starts earlier. (3) Investigate why Old Gen is filling — memory leak or insufficient heap. Note: the 555ms pause is unusually long — this is a bad sign.

---

**Q74 🔴** What is safepoint latency and how does it affect GC pauses?

> **Answer:** Before any STW pause, the JVM must wait for all application threads to reach a **safepoint** — a point in code where thread state is well-defined. "Time to safepoint" is the delay from GC requesting a pause to all threads actually stopping. This adds to the reported GC pause time. Causes of long safepoint latency: tight loops with no method calls or backward branches (JIT optimizes them to have no safepoints), code doing very long native operations. Diagnose with `-Xlog:safepoint*` — look for large "vmop begin" to "vmop end" gaps.

---

**Q75 🔴** How do you calculate GC overhead percentage from GC logs?

> **Answer:**
> ```
> GC overhead % = (Total GC time / JVM uptime) × 100
>
> Example from jstat:
>   GCT = 10.78s   (total GC time)
>   Uptime = 300s
>   GC overhead = 10.78 / 300 × 100 = 3.6%
>
> From GC log:
>   Sum all "Pause" durations
>   Divide by elapsed wall-clock time
>
> Thresholds:
>   < 5%   = Good
>   5–10%  = Acceptable, monitor closely
>   > 10%  = Problem — performance degraded significantly
>   > 25%  = Critical — consider OOM imminent
> ```

---

**Q76 🟡** What does `jcmd VM.metaspace` show and why is it useful?

> **Answer:** `jcmd <pid> VM.metaspace` (Java 16+) shows a detailed breakdown of Metaspace usage by ClassLoader — including: total committed and reserved Metaspace, per-ClassLoader breakdown showing which ClassLoader owns how much class metadata. This is critical for diagnosing Metaspace leaks: if a specific ClassLoader is accumulating large amounts (and growing) without being released, it indicates classes being loaded repeatedly without the ClassLoader being GC'd (e.g., CGLIB proxy generation without caching).

---

**Q77 🔴** Explain how to diagnose and fix a Metaspace OOM.

> **Answer:** (1) **Confirm**: GC log shows "Metadata GC Threshold" as Full GC cause; OOM message says "Metaspace". (2) **Measure**: `jcmd <pid> VM.metaspace` — identify which ClassLoaders are largest. (3) **Heap dump**: look for ClassLoader instances and what's keeping them alive — `org.springframework.cglib.proxy.*` or `groovy.lang.GroovyClassLoader` etc. (4) **Root cause**: CGLIB proxy creation not cached (Spring AOP, Hibernate), Groovy scripts loaded in loop, OSGi bundle reloading without cleanup. (5) **Fix**: Cache generated proxies, use a shared ClassLoader, configure Groovy to reuse class definitions. (6) **Temporary**: add `-XX:MaxMetaspaceSize=512m` to fail fast with OOM rather than crashing the OS.

---

**Q78 🔴** What is heap fragmentation and which collectors are vulnerable to it?

> **Answer:** Heap fragmentation occurs when free memory is scattered in small non-contiguous chunks — total free space is sufficient but no single chunk is large enough for a requested allocation. This causes `OutOfMemoryError` despite apparent free memory. **CMS is most vulnerable** — it uses mark-sweep (no compaction), and after many cycles, Old Gen becomes heavily fragmented → allocation failures → Concurrent Mode Failure → emergency Full GC. **Serial, Parallel, G1** compact Old Gen — no fragmentation. **ZGC, Shenandoah** compact concurrently — no fragmentation. This is one of the main reasons CMS was removed.

---

**Q79 🟡** What GC-related JVM metrics should you export to Prometheus/Grafana?

> **Answer:** Key metrics via JMX/Micrometer: (1) **Heap used by pool** — `jvm.memory.used{area=heap, id=G1 Old Gen}` (2) **GC pause count and duration** — `jvm.gc.pause{action=end of major GC}` (3) **GC pause max** — p99 pause time (4) **Allocation rate** — rate of `jvm.gc.memory.allocated` (5) **Promotion rate** — rate of `jvm.gc.memory.promoted` (6) **Old Gen % used** — alert when > 80% (7) **Full GC count** — alert on any increase (8) **Metaspace used** — alert when > 80% of max.

---

**Q80 🔴** You notice GC logs show very long "Time from last GC" intervals followed by large Full GC pauses. What's happening and how do you fix it?

> **Answer:** This pattern typically indicates an application that doesn't allocate much but accumulates objects in Old Gen slowly — then hits a threshold that triggers Full GC. Possible causes: (1) **Gradual memory leak** — objects accumulate in static caches without eviction. (2) **Long-lived session objects** — user sessions accumulating in memory. (3) **Mistuned IHOP** — G1 not starting concurrent marking until Old Gen is nearly full. Investigation: (1) Take heap histograms at multiple intervals — find growing class. (2) Heap dump + MAT — find what's holding references. Fix: eviction policies on caches, session timeout, lower IHOP.

---

## Advanced & Internals (Q81–Q100)

---

**Q81 🔴** What is escape analysis and how does the JVM use it to reduce GC pressure?

> **Answer:** Escape analysis is a JIT compiler optimization that determines whether an object's reference can "escape" the method or thread where it's created. If an object does NOT escape: (1) **Stack allocation** — the JVM may allocate it on the stack (Java currently does scalar replacement instead). (2) **Scalar replacement** — the object is decomposed into its primitive fields stored in registers/stack — never allocated on the heap at all. (3) **Synchronization elision** — locks on non-escaping objects are removed. This reduces GC pressure by eliminating heap allocations for short-lived objects. Enable with `-XX:+DoEscapeAnalysis` (default: on).

---

**Q82 🔴** What is the difference between shallow heap and retained heap?

> **Answer:** **Shallow heap** = the amount of memory consumed by the object itself (its fields) — does NOT include referenced objects. For a `String`, shallow heap = String object header + char[] reference. **Retained heap** = the total memory that would be freed if this object were GC'd — the object itself + all objects that can ONLY be reached through this object. In Eclipse MAT: an `ArrayList` with 10,000 entries has small shallow heap (~40 bytes) but potentially large retained heap (40 bytes + all entry objects). Retained heap is the key metric for identifying memory leak culprits.

---

**Q83 🔴** Explain the tri-color marking invariant and why it matters for concurrent GC correctness.

> **Answer:** The tri-color invariant: **a black object must never reference a white object directly**. If a black object (already fully processed) gains a reference to a white object (not yet visited), the GC won't re-scan the black object — the white object would be missed and incorrectly collected. This is the core correctness challenge of concurrent marking. Solutions: (1) **SATB** (G1) — if a reference is removed from a grey/white object, save the old value; ensures snapshot completeness. (2) **Incremental Update** (CMS) — when a black object gets a new white reference, make it grey again for re-scanning. (3) **Load Barrier** (ZGC) — ensure the GC's view of the object graph is always consistent.

---

**Q84 🔴** What is a "floating garbage" in concurrent GC and why is it acceptable?

> **Answer:** Floating garbage refers to objects that **become unreachable AFTER** the concurrent marking snapshot was taken but BEFORE collection completes. Since the GC marks based on a snapshot (or SATB), objects that die during the concurrent phase are not identified as garbage in this cycle — they survive until the NEXT GC cycle. This is acceptable because: (1) It's a bounded delay — one extra GC cycle. (2) It doesn't cause memory leaks — the objects ARE eventually collected. (3) The alternative (re-scanning everything) would eliminate concurrency. G1, ZGC, and Shenandoah all accept floating garbage as a design trade-off.

---

**Q85 🔴** What is the G1 mixed GC live threshold and how does it affect performance?

> **Answer:** `-XX:G1MixedGCLiveThresholdPercent=85` (default) means G1 will only select Old Gen regions for Mixed GC if they are **less than 85% live** — i.e., at least 15% garbage. Regions that are > 85% live are skipped (too expensive to evacuate for little gain). Lowering this threshold (e.g., 65%) means G1 skips more regions (only collects highly-garbage regions) — less work per Mixed GC but Old Gen reclaimed more slowly. Raising it means G1 collects even mostly-live regions — more work per cycle but faster Old Gen reclamation.

---

**Q86 🔴** How does ZGC use "colored pointers" (load barriers) differently from write barriers?

> **Answer:** Traditional collectors (G1, CMS, Parallel) use **write barriers** — code injected on reference WRITES to maintain the card table and SATB buffers. ZGC uses **load barriers** — code injected on reference READS. When a ZGC load barrier executes, it checks the color bits in the 64-bit pointer: if the color is "bad" (e.g., pointing to an object that has been relocated), the slow path runs to find the new location and self-heal the reference. Write barriers track what changed; load barriers correct stale references on access. Load barriers enable ZGC's ability to relocate objects concurrently.

---

**Q87 🔴** What is the JVM's "object header" and what GC-relevant information does it contain?

> **Answer:** Every Java object has a 2-word header (8–16 bytes): (1) **Mark Word** (8 bytes) — contains: GC age (4 bits, max 15), bias lock thread ID, lock state (unlocked/biased/thin/fat), identity hashcode. In ZGC, the entire object address is in the pointer (colored), not the header. (2) **Class Pointer** (4–8 bytes, compressed with `-XX:+UseCompressedOops`) — pointer to the class metadata in Metaspace. The mark word's **age bits** are what generational GC uses to decide when to promote an object to Old Gen.

---

**Q88 🔴** What is compressed ordinary object pointers (CompressedOops)?

> **Answer:** `-XX:+UseCompressedOops` (default on 64-bit JVMs with heap ≤ 32 GB) encodes 64-bit object references using 32 bits. The JVM shifts the pointer right by 3 bits (exploiting 8-byte object alignment) — this allows 32 bits to address 2³² × 8 = 32 GB. Benefits: reduces memory usage (~30% heap reduction) and improves cache efficiency (smaller references = more fit in cache). Disabled automatically when `-Xmx > ~32 GB`. For heaps just above 32 GB, consider the "compressed class pointers" zone — the actual threshold varies slightly by platform.

---

**Q89 🔴** Explain the concept of region evacuation in G1 GC.

> **Answer:** Evacuation in G1 is the process of copying all live objects out of a selected set of regions into new regions, then reclaiming the evacuated regions entirely. During Young GC: all Eden + Survivor regions are evacuated. During Mixed GC: Eden + Survivor + a selection of Old regions with the most garbage. Evacuation requires: (1) Identifying all live objects in the region (via marking + RSets). (2) Copying them to free regions (promotes to Old, or keeps in Survivor). (3) Updating all references to the moved objects. (4) Completely freeing the source regions. This is G1's primary reclamation mechanism — efficient for high-garbage regions.

---

**Q90 🔴** What is "concurrent refinement" in G1 GC?

> **Answer:** G1's concurrent refinement is a set of background threads that continuously process **Dirty Card Queue (DCQ)** entries — the write barrier enqueues dirty card locations here. Refinement threads drain this queue and update the RSets of the target regions. This spreads the RSet maintenance work across many small background operations rather than doing it all during STW pause. Without refinement, the dirty card queue would grow large and require all processing during the next GC pause. Configure: `-XX:G1ConcRefinementThreads=4`.

---

**Q91 🔴** What is the difference between `jvm.gc.pause` and `jvm.gc.overhead` metrics?

> **Answer:** `jvm.gc.pause` measures the **duration of each individual STW pause** event (in seconds or milliseconds) — useful for latency percentile analysis (p50/p95/p99 pause times). `jvm.gc.overhead` (or calculated from GC time / elapsed time) measures the **fraction of total CPU time spent in GC** — directly represents the throughput cost of GC. A service can have frequent but very short pauses (low overhead, high pause count) or infrequent but long pauses (high overhead, low pause count). Monitor both: overhead for throughput SLAs, pause duration for latency SLAs.

---

**Q92 🔴** How does Java handle GC for objects with native resources (e.g., file handles)?

> **Answer:** GC only frees Java heap memory — it does **not** close file handles, release native memory, or call native destructors. Options for native resource cleanup: (1) **try-with-resources** — deterministic cleanup at end of scope (preferred). (2) **Explicit `close()`** — caller must remember to call. (3) **`Cleaner` API** (Java 9+) — registers a cleanup action that runs AFTER the object is GC'd (no guarantee of timing, but better than `finalize()`). (4) **`PhantomReference` + `ReferenceQueue`** — detect post-GC for custom cleanup (used internally by NIO's `DirectByteBuffer`).

---

**Q93 🔴** What is GC ergonomics?

> **Answer:** GC ergonomics refers to the JVM's automatic tuning of GC parameters at runtime — primarily used by Parallel GC with `UseAdaptiveSizePolicy`. The JVM observes GC behavior (pause times, throughput) and dynamically adjusts: Eden size, Survivor sizes, `MaxTenuringThreshold`, and Old Gen size to approach the configured `MaxGCPauseMillis` and `GCTimeRatio` targets. GC ergonomics can be logged with `-Xlog:gc+ergo*` to understand what adjustments the JVM is making and why.

---

**Q94 🔴** What is "pause prediction model" in G1 GC and how does it work?

> **Answer:** G1 maintains a **pause prediction model** based on historical GC data. Before each GC, G1 estimates: (1) How long it will take to scan roots and process the work queue. (2) How many Old Gen regions it can evacuate within the `MaxGCPauseMillis` target. (3) Whether to include Old Gen regions at all (Mixed GC) or just Young Gen. The model uses exponential weighted moving averages of previous GC phase durations. This allows G1 to dynamically adjust the collection set (which regions to collect) to stay within the pause target, getting better with time.

---

**Q95 🔴** What happens to ZGC's load barrier performance in CPU-intensive code?

> **Answer:** ZGC's load barriers execute on every reference read. In CPU-intensive code with many object field accesses, this adds overhead. JIT-compiled load barriers typically cost ~1–3 CPU instructions in the common (fast path) case — similar to a branch-prediction-friendly conditional. The slow path (actual relocation) is rare. In practice, ZGC's throughput overhead is ~5–15% vs Parallel GC — mostly from load barriers. For truly CPU-bound, allocation-light code, Parallel GC may still win on throughput. For mixed workloads, ZGC's reduced GC pause overhead compensates.

---

**Q96 🔴** What is `ZGC Garbage Collection (Proactive)` vs `ZGC Garbage Collection (Allocation Rate)?`

> **Answer:** ZGC triggers GC for different reasons: (1) **Proactive** — ZGC proactively collects before the heap reaches critical occupancy. Triggered when occupancy exceeds a heuristic threshold. Keeps the heap clean to avoid allocation stalls. (2) **Allocation Rate** — GC triggered because the allocation rate is too high and the heap may be exhausted. More urgent. (3) **Allocation Stall** — the most serious: application thread is blocked waiting for GC to free memory before it can allocate. Indicates ZGC can't keep up — increase heap or reduce allocation rate.

---

**Q97 🔴** What is Shenandoah's "heuristics" system?

> **Answer:** Shenandoah uses different **GC heuristics modes** that control when and how aggressively to trigger GC: (1) **adaptive** (default) — monitors allocation rate and heap occupancy to decide when to start GC cycles; adjusts timing based on observed behavior. (2) **compact** — very aggressive; triggers GC cycles as frequently as possible; maximum compactness but highest CPU overhead. (3) **static** — triggers based on fixed occupancy thresholds. (4) **aggressive** — for testing; runs GC as aggressively as possible. Configure: `-XX:ShenandoahGCHeuristics=adaptive`.

---

**Q98 🔴** Why does G1 Full GC use a single thread in older Java versions?

> **Answer:** G1's Full GC was single-threaded until Java 10 (JEP 307: Parallel Full GC for G1). The original G1 design assumed Full GC would be rare (a fallback), so investing in parallel full GC implementation was deprioritized. In practice, Full GC was triggered more often than expected in production workloads. Java 10+ parallelized G1 Full GC using `-XX:ParallelGCThreads` threads, significantly reducing worst-case pause times. Still, Full GC should be avoided through proper tuning — parallel just makes the fallback less catastrophic.

---

**Q99 🔴** What are value types (Project Valhalla) and how will they affect GC?

> **Answer:** Project Valhalla introduces **value types** (classes declared with `value`) — objects whose identity is defined by their values, not their reference. Value types can be **flattened** into arrays and object fields without boxing — a `Point[]` of value types stores the x,y coordinates inline rather than as pointers to heap-allocated Point objects. GC impact: (1) Dramatically fewer heap allocations — value type arrays eliminate millions of small object allocations. (2) Less indirection — better cache locality, fewer GC roots. (3) Smaller heap → less GC work. Java 22+ has preview value classes. Full impact on GC will be significant for numeric/data-heavy workloads.

---

**Q100 🔴** If you could redesign Java's GC strategy from scratch for modern cloud workloads (serverless, containers, microservices), what would you recommend and why?

> **Answer:** For modern cloud workloads: (1) **Generational ZGC (Java 21)** as the universal default — sub-ms pauses are non-negotiable in containerized microservices where JVM startup and tail latency directly affect user experience. (2) **Aggressive tiered startup** — pre-warmed class loading and AOT compilation (GraalVM Native Image approach) to eliminate the JVM warm-up period. (3) **Container-aware memory management** — JVM should communicate memory pressure bidirectionally with the container runtime (`MaxRAMPercentage` is a start). (4) **Virtual threads (Java 21)** — each virtual thread allocates a stack on the heap (not OS stack) — requires GC to handle millions of tiny stacks efficiently, driving generational ZGC. (5) **Value types (Valhalla)** — eliminate the allocation tax for small data objects. The trend is clear: minimize heap allocation pressure + minimize GC pause impact = sub-ms pauses with competitive throughput.

---

**Navigation**

[← Best Practices](08-best-practices.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md)

---

*Last updated: 2026 | Java 21 LTS*
