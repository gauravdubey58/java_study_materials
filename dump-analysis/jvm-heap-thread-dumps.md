# JVM Heap Dumps & Thread Dumps — Complete Notes

---

## Table of Contents
1. [Overview](#1-overview)
2. [Heap Dumps](#2-heap-dumps)
   - [What is a Heap Dump?](#21-what-is-a-heap-dump)
   - [When to Take a Heap Dump](#22-when-to-take-a-heap-dump)
   - [How to Generate a Heap Dump](#23-how-to-generate-a-heap-dump)
   - [Heap Dump File Format](#24-heap-dump-file-format)
   - [Heap Dump Analysis Techniques](#25-heap-dump-analysis-techniques)
   - [Heap Dump Analysis Tools](#26-heap-dump-analysis-tools)
3. [Thread Dumps](#3-thread-dumps)
   - [What is a Thread Dump?](#31-what-is-a-thread-dump)
   - [When to Take a Thread Dump](#32-when-to-take-a-thread-dump)
   - [How to Generate a Thread Dump](#33-how-to-generate-a-thread-dump)
   - [Anatomy of a Thread Dump](#34-anatomy-of-a-thread-dump)
   - [Thread States](#35-thread-states)
   - [Thread Dump Analysis Techniques](#36-thread-dump-analysis-techniques)
   - [Thread Dump Analysis Tools](#37-thread-dump-analysis-tools)
4. [Common Problems & Patterns](#4-common-problems--patterns)
5. [Heap vs Thread Dump — Quick Reference](#5-heap-vs-thread-dump--quick-reference)
6. [Best Practices](#6-best-practices)
7. [References](#7-references)

---

## 1. Overview

| Dump Type | What it Captures | Primary Use |
|---|---|---|
| **Heap Dump** | Snapshot of all objects in JVM heap memory at a point in time | Diagnose memory leaks, high memory usage, OutOfMemoryError |
| **Thread Dump** | Snapshot of all active threads and their stack traces at a point in time | Diagnose deadlocks, CPU spikes, thread hangs, contention |

> Both are **point-in-time snapshots** — take **multiple dumps** (especially thread dumps) for accurate diagnosis.

---

## 2. Heap Dumps

### 2.1 What is a Heap Dump?

A **heap dump** is a binary snapshot of the JVM's heap memory containing:
- All **live objects** and their sizes.
- **References** between objects (object graph).
- **Class metadata** and static fields.
- **GC roots** — the entry points from which object reachability is determined.

```
Heap Dump = Object Graph Snapshot
  GC Roots
    └── Class (static fields)
    └── Thread Stack (local variables)
    └── JNI References
          └── Object A → Object B → Object C
                             └── Object D
```

### 2.2 When to Take a Heap Dump

| Symptom | Action |
|---|---|
| `java.lang.OutOfMemoryError: Java heap space` | Take heap dump immediately (or auto-capture). |
| Steadily increasing heap usage over time | Suspect a memory leak — take heap dumps at intervals. |
| High GC frequency / long GC pauses | Inspect what is filling the heap. |
| Application slowdown with no CPU spike | Could be excessive GC — heap dump + GC logs. |
| Unexpectedly high memory footprint | Profile what dominates the heap. |

---

### 2.3 How to Generate a Heap Dump

#### Option 1: JVM Flag — Auto-capture on OOM (Recommended for Production)
```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/logs/heapdump.hprof
```
> The JVM automatically writes a heap dump when `OutOfMemoryError` is thrown. Always enable this in production.

#### Option 2: `jmap` (JDK Built-in)
```bash
# Live objects only (faster, smaller file)
jmap -dump:live,format=b,file=heapdump.hprof <pid>

# All objects including unreachable
jmap -dump:format=b,file=heapdump.hprof <pid>

# Get PID with: jps -l
```
> ⚠️ `jmap` triggers a **stop-the-world pause** — use with caution in production.

#### Option 3: `jcmd` (Preferred over jmap for Java 8+)
```bash
jcmd <pid> GC.heap_dump /path/to/heapdump.hprof

# Example
jcmd 12345 GC.heap_dump /var/logs/heapdump.hprof
```

#### Option 4: JConsole / VisualVM (GUI)
- Connect to the running JVM via JMX.
- Navigate to **Memory** tab → **Heap Dump** button.

#### Option 5: Programmatically via MBean
```java
import com.sun.management.HotSpotDiagnosticMXBean;
import java.lang.management.ManagementFactory;

public class HeapDumpUtil {
    private static final String HOTSPOT_BEAN = "com.sun.management:type=HotSpotDiagnostic";

    public static void dumpHeap(String filePath, boolean liveOnly) throws Exception {
        MBeanServer server = ManagementFactory.getPlatformMBeanServer();
        HotSpotDiagnosticMXBean mxBean = ManagementFactory.newPlatformMXBeanProxy(
            server, HOTSPOT_BEAN, HotSpotDiagnosticMXBean.class);
        mxBean.dumpHeap(filePath, liveOnly);
    }
}
```

---

### 2.4 Heap Dump File Format

Heap dumps are saved in **HPROF format** (`.hprof`) — a binary format originally from Sun Microsystems.

```
heapdump.hprof
├── Header
│   ├── Magic number
│   ├── Identifier size (4 or 8 bytes)
│   └── Timestamp
├── Records
│   ├── STRING  (interned strings)
│   ├── LOAD CLASS (class serial numbers)
│   ├── HEAP DUMP SEGMENT
│   │   ├── ROOT entries (GC roots)
│   │   ├── CLASS DUMP (class metadata, static fields)
│   │   └── INSTANCE DUMP (object data)
│   └── HEAP DUMP END
```

> File sizes can range from **hundreds of MB to several GB** depending on heap size. Use `-dump:live` to reduce file size.

---

### 2.5 Heap Dump Analysis Techniques

#### Technique 1: Dominator Tree Analysis
- The **dominator tree** shows which objects are responsible for retaining the most memory.
- An object **A dominates** object **B** if every path from GC root to B passes through A.
- **Retained Heap** = total memory freed if the object (and all objects it dominates) are collected.

```
Dominator Tree Example:
Root
 └── ServiceCache  [Retained: 450 MB]  ← prime suspect
       └── HashMap
             └── ArrayList (10,000 entries)
                   └── CustomerData objects
```
> Sort by **Retained Heap** to find memory leak culprits quickly.

#### Technique 2: Shallow vs. Retained Heap
| Metric | Definition | Use |
|---|---|---|
| **Shallow Heap** | Memory consumed by the object itself (not its references). | Identify large individual objects. |
| **Retained Heap** | Memory freed if this object is GC'd (includes all dominated objects). | Identify objects holding large object graphs. |

#### Technique 3: Histogram Analysis
- Lists all classes and the count/size of their instances.
- Look for:
  - **Unexpectedly high instance counts** (e.g., millions of `String`, `byte[]`, or custom objects).
  - **Classes growing over time** (compare multiple dumps).

```
Class Histogram (sample):
 num  #instances  #bytes  class name
   1:    2500000  200MB   [B  (byte arrays)
   2:    1800000  144MB   java.lang.String
   3:      50000   40MB   com.example.SessionData
   4:     300000   24MB   java.util.HashMap$Entry
```

#### Technique 4: GC Root Path (Path to GC Roots)
- For a suspected leaked object, find the **reference chain from GC Root to the object**.
- This reveals **why the object is not being collected**.

```
GC Root Path Example:
Thread "main" (GC Root)
  └── ApplicationContext (static field)
        └── BeanFactory
              └── SessionManager
                    └── ConcurrentHashMap
                          └── SessionData (leaked object) ← HERE
```

#### Technique 5: Duplicate Objects Detection
- Look for many instances of identical objects (e.g., duplicate `String` literals, cached configs).
- Use `String` deduplication (`-XX:+UseStringDeduplication` with G1 GC).

#### Technique 6: Compare Two Heap Dumps (Leak Detection)
- Take a heap dump at **Time T1** and another at **Time T2**.
- Compare histograms — classes with growing instance counts are leak candidates.

---

### 2.6 Heap Dump Analysis Tools

---

#### Eclipse MAT (Memory Analyzer Tool) — Most Popular
- **Type:** Standalone GUI / Eclipse Plugin
- **Download:** https://eclipse.dev/mat/
- **Key Features:**
  - Automatic **Leak Suspects Report** (highlights likely leaks with explanations).
  - **Dominator Tree** view.
  - **Histogram** with shallow/retained heap.
  - **OQL (Object Query Language)** — SQL-like queries on heap objects.
  - **Path to GC Roots** for any object.
  - Handles very large heap dumps efficiently.

```sql
-- OQL Example: Find all SessionData objects older than 30 mins
SELECT s FROM com.example.SessionData s
WHERE s.createdAt < (System.currentTimeMillis() - 1800000)
```

**How to use:**
```
1. File → Open Heap Dump → select .hprof file
2. Run "Leak Suspects Report" for quick analysis
3. Use "Dominator Tree" to find biggest retained objects
4. Use "Histogram" to find most frequent object types
5. Right-click object → "Path to GC Roots" to trace references
```

---

#### VisualVM
- **Type:** Standalone GUI (bundled with JDK up to Java 8, standalone after)
- **Download:** https://visualvm.github.io/
- **Key Features:**
  - Live heap monitoring + heap dump capture.
  - Heap dump analysis (histogram, object details).
  - Thread monitoring and thread dump capture.
  - CPU & memory profiling.

---

#### IntelliJ IDEA Profiler / JProfiler
- **Type:** Commercial IDE-integrated profiler
- **Key Features:**
  - Deep heap analysis integrated into IDE.
  - Live allocation tracking.
  - Call tree + memory views.
  - Best for development-time profiling.

---

#### JVM Heap Analyzer (IBM / Red Hat)
- IBM HeapAnalyzer: for IBM J9 JVM heap dumps.
- Red Hat JVM Inspector: for OpenJDK / JBoss environments.

---

#### GCEasy / HeapHero (Web-based)
- **URL:** https://heaphero.io / https://gceasy.io
- Upload `.hprof` file — get automated analysis report in browser.
- Good for quick, no-setup analysis.

> ⚠️ Avoid uploading production heap dumps containing PII to public web tools.

---

## 3. Thread Dumps

### 3.1 What is a Thread Dump?

A **thread dump** is a snapshot of all threads running inside the JVM at a specific moment, including:
- Thread **name, ID, and priority**.
- Thread **state** (RUNNABLE, BLOCKED, WAITING, etc.).
- **Stack trace** — the exact method call chain for each thread.
- **Lock information** — which locks a thread holds or is waiting for.

```
Thread Dump = All Threads Snapshot
  Thread "http-nio-8080-exec-1" [RUNNABLE]
    at com.example.OrderService.processOrder(OrderService.java:45)
    at com.example.OrderController.placeOrder(OrderController.java:32)
    ...

  Thread "http-nio-8080-exec-2" [BLOCKED]
    waiting to lock <0x000000076b3f2a10> (com.example.OrderService)
    held by Thread "http-nio-8080-exec-1"
    ...
```

---

### 3.2 When to Take a Thread Dump

| Symptom | Action |
|---|---|
| Application appears **hung / unresponsive** | Take 3 thread dumps, 10 seconds apart. |
| **CPU usage 100%** with no progress | Look for runnable threads in tight loops. |
| **Deadlock detected** by JVM | Thread dump will show deadlock details explicitly. |
| **Slow response times** under load | Check for thread contention / lock waiting. |
| **Thread pool exhaustion** | Check what threads are doing (likely all blocked). |
| **Intermittent timeouts** | Capture thread dump during the timeout window. |

> 💡 **Best Practice:** Take **3–5 consecutive thread dumps** with a 5–10 second gap. A single dump shows a moment in time; multiple dumps reveal stuck vs. progressing threads.

---

### 3.3 How to Generate a Thread Dump

#### Option 1: `kill -3` Signal (Linux/macOS)
```bash
kill -3 <pid>
# Output goes to stdout/console of the JVM process
# Redirect output: nohup java -jar app.jar > app.log 2>&1 &
```

#### Option 2: `jstack` (JDK Built-in) — Most Common
```bash
# Basic thread dump
jstack <pid>

# Include mixed-mode stack traces (Java + native)
jstack -m <pid>

# Force dump even if JVM is unresponsive
jstack -F <pid>

# Save to file
jstack <pid> > threaddump.txt
```

#### Option 3: `jcmd` (Preferred for Java 8+)
```bash
jcmd <pid> Thread.print

# Save to file
jcmd <pid> Thread.print > threaddump.txt
```

#### Option 4: JConsole / VisualVM (GUI)
- **JConsole:** Threads tab → "Detect Deadlock" button + thread list.
- **VisualVM:** Threads tab → "Thread Dump" button.

#### Option 5: Programmatically via ThreadMXBean
```java
import java.lang.management.ManagementFactory;
import java.lang.management.ThreadInfo;
import java.lang.management.ThreadMXBean;

public class ThreadDumpUtil {
    public static void printThreadDump() {
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        ThreadInfo[] threadInfos = threadMXBean.dumpAllThreads(true, true);
        for (ThreadInfo info : threadInfos) {
            System.out.println(info);
        }
    }

    public static void detectDeadlocks() {
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        long[] deadlockedThreads = threadMXBean.findDeadlockedThreads();
        if (deadlockedThreads != null) {
            ThreadInfo[] infos = threadMXBean.getThreadInfo(deadlockedThreads, true, true);
            System.out.println("DEADLOCK DETECTED:");
            for (ThreadInfo info : infos) {
                System.out.println(info);
            }
        }
    }
}
```

#### Option 6: Windows — `Ctrl+Break`
```
In the console window running the JVM:
Press Ctrl + Break
Thread dump is printed to stdout.
```

---

### 3.4 Anatomy of a Thread Dump

```
=============== THREAD DUMP HEADER ===============
Full thread dump OpenJDK 64-Bit Server VM (21.0.1+12 mixed mode):

=============== INDIVIDUAL THREAD ENTRY ===============

"http-nio-8080-exec-3" #45 daemon prio=5 os_prio=0 cpu=1234ms elapsed=300s tid=0x00007f1c2c003800 nid=0x3f5a waiting on condition [0x00007f1c24aff000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x000000076b3fa200> (java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
        at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
        at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:107)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1074)

"VM Thread" os_prio=0 cpu=45ms elapsed=305s tid=0x00007f1c2c200000 nid=0x3e10 runnable
```

| Field | Description |
|---|---|
| `"http-nio-8080-exec-3"` | Thread name |
| `#45` | Thread ID (JVM internal) |
| `daemon` | Whether it's a daemon thread |
| `prio=5` | Java thread priority (1–10) |
| `os_prio=0` | OS-level thread priority |
| `cpu=1234ms` | CPU time consumed (Java 9+) |
| `elapsed=300s` | Time since thread started |
| `tid=0x00007f...` | JVM thread pointer |
| `nid=0x3f5a` | Native OS thread ID (hex) |
| `Thread.State` | Current Java thread state |
| Stack trace lines | Method call chain |
| `- locked <0x...>` | Lock currently held |
| `- waiting to lock <0x...>` | Lock being waited for |

---

### 3.5 Thread States

```
                          ┌─────────────┐
                          │     NEW     │
                          └──────┬──────┘
                                 │ start()
                          ┌──────▼──────┐
                   ┌──────│  RUNNABLE   │◄────┐
                   │      └──────┬──────┘     │
              lock │             │ wait/sleep  │ notify/timeout
            held   │      ┌──────▼──────┐     │
            by     │      │   WAITING   │─────┘
            other  │      │  (TIMED)    │
                   │      └─────────────┘
                   │
            ┌──────▼──────┐
            │   BLOCKED   │  (waiting for monitor lock)
            └─────────────┘
                   │
            ┌──────▼──────┐
            │  TERMINATED │
            └─────────────┘
```

| State | Description | Common Cause |
|---|---|---|
| `RUNNABLE` | Thread is executing or ready to execute. | Normal operation or CPU-bound loop. |
| `BLOCKED` | Waiting to acquire a **monitor lock** (synchronized block). | Lock contention. |
| `WAITING` | Waiting indefinitely for a notification (`Object.wait()`, `LockSupport.park()`). | Thread pool idle threads, `join()`. |
| `TIMED_WAITING` | Waiting for a specified time (`Thread.sleep()`, `wait(timeout)`, `parkNanos()`). | Scheduled tasks, retries. |
| `TERMINATED` | Thread has completed execution. | — |
| `NEW` | Thread created but not yet started. | — |

---

### 3.6 Thread Dump Analysis Techniques

#### Technique 1: Identify Deadlocks
- Look for `"Found one Java-level deadlock:"` section at the bottom of the dump (JVM auto-detects).
- Manually look for a **circular lock dependency** across threads.

```
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x000000076b3f1a10 (object of type com.example.Account)
  which is held by "Thread-2"

"Thread-2":
  waiting to lock monitor 0x000000076b3f2b20 (object of type com.example.Account)
  which is held by "Thread-1"
```

**Fix pattern:** Always acquire locks in a **consistent order** across all threads.

---

#### Technique 2: Identify CPU Spikes (Hot Threads)
1. Find the PID of the process: `jps -l`
2. Find the OS thread consuming CPU:
```bash
# On Linux — top by thread
top -H -p <pid>

# Note the PID of the high-CPU thread (decimal)
# Convert to hex: printf '%x\n' <thread_pid>
```
3. Match hex thread ID to `nid=0x...` in the thread dump.
4. Inspect the stack trace of that thread.

```bash
# Example
top -H -p 12345
# Thread PID: 12390 → hex: 0x3066

# In thread dump, find:
"GC Task Thread#3" nid=0x3066 runnable
```

---

#### Technique 3: Identify Thread Contention / Lock Waiting
- Look for many threads in **BLOCKED** state.
- They will show: `waiting to lock <0x...> held by "Thread-X"`.
- The **thread holding the lock** is the bottleneck — inspect its stack trace.

```
"exec-5" BLOCKED
  waiting to lock <0x000000076b3f2a10> (com.example.OrderService)
  held by "exec-1"

"exec-1" RUNNABLE
  at com.example.OrderService.processOrder(OrderService.java:45)  ← bottleneck
  - locked <0x000000076b3f2a10>
```

**Fix pattern:** Reduce synchronized block scope, use `ConcurrentHashMap`, `ReadWriteLock`, or atomic classes.

---

#### Technique 4: Identify Thread Pool Starvation
- All worker threads are **BLOCKED** or **WAITING**.
- No threads are processing new requests → request queue grows.

```
"http-exec-1" WAITING  (idle, waiting for task)
"http-exec-2" BLOCKED  (waiting for DB connection pool)
"http-exec-3" BLOCKED  (waiting for DB connection pool)
"http-exec-4" BLOCKED  (waiting for DB connection pool)
                       ↑ DB connection pool exhausted!
```

**Fix pattern:** Increase connection pool size, add timeouts, use async I/O.

---

#### Technique 5: Identify Hung Threads
- Compare multiple dumps (3 dumps, 10s apart).
- A thread at the **same stack frame across all dumps** = stuck thread.

```
Dump 1 (T=0):   "exec-2" at com.example.ExternalService.call(line 78)
Dump 2 (T=10):  "exec-2" at com.example.ExternalService.call(line 78)
Dump 3 (T=20):  "exec-2" at com.example.ExternalService.call(line 78)
                                                              ↑ Stuck! — likely network timeout
```

**Fix pattern:** Add timeouts to all external calls (HTTP, DB, RPC).

---

#### Technique 6: Group Threads by State
Quickly tally thread states across the dump:
```bash
# Count thread states from a thread dump file
grep "java.lang.Thread.State" threaddump.txt | sort | uniq -c | sort -rn
```
```
  120 WAITING (parking)       ← healthy idle pool threads
   45 TIMED_WAITING (sleeping)
   30 BLOCKED                 ← ⚠️ contention
    5 RUNNABLE                ← actively processing
```

---

### 3.7 Thread Dump Analysis Tools

---

#### FastThread (Web-based) — Most Popular
- **URL:** https://fastthread.io
- Upload thread dump → instant visual analysis report.
- Groups threads by state, detects deadlocks, highlights hot threads.
- Compares multiple dumps side-by-side.
- **Free tier available.**

---

#### VisualVM
- **Type:** Standalone GUI
- **URL:** https://visualvm.github.io
- Capture live thread dumps, view thread states visually on timeline.
- Color-coded thread state graph over time.

---

#### JStack Review (IntelliJ IDEA)
- IntelliJ IDEA has a built-in **Thread Dump Analyzer**.
- Open any `jstack` output file inside IntelliJ for a formatted, grouped view.
- **Menu:** Analyze → Stack Trace or Thread Dump.

---

#### TDA — Thread Dump Analyzer (Open Source)
- **URL:** https://github.com/irockel/tda
- Standalone GUI tool for parsing and comparing thread dumps.
- Monitors thread counts and highlights deadlocks.

---

#### Samurai (Open Source)
- Visualizes **multiple consecutive thread dumps** on a timeline.
- Shows which threads are consistently stuck across dumps.
- Good for identifying intermittent hang patterns.

---

#### GCEasy / Tier1app
- **URL:** https://tier1app.com / https://gceasy.io
- Web-based — upload thread dump for automated report.

---

## 4. Common Problems & Patterns

### 4.1 Memory Leak Pattern (Heap Dump)
```
Symptom:   Heap grows continuously, OOM after hours/days
Heap Dump: Dominator tree shows a Cache/Map holding millions of objects
Path:      StaticField → ApplicationCache → HashMap → [millions of MyObject]
Fix:       Use weak references (WeakHashMap), add eviction policy (Caffeine/Guava cache)
```

### 4.2 Deadlock Pattern (Thread Dump)
```
Symptom:   Application hangs completely, no progress
Dump:      "Found one Java-level deadlock" section present
Pattern:   Thread A holds Lock1, waits for Lock2
           Thread B holds Lock2, waits for Lock1
Fix:       Enforce consistent lock ordering; use tryLock() with timeout
```

### 4.3 Thread Pool Exhaustion (Thread Dump)
```
Symptom:   Requests queuing up, response times increasing
Dump:      All worker threads BLOCKED on external resource
Pattern:   DB/HTTP connection pool exhausted
Fix:       Increase pool size, add connection timeouts, use circuit breaker
```

### 4.4 High CPU Infinite Loop (Thread Dump)
```
Symptom:   CPU at 100%, app unresponsive
Dump:      One RUNNABLE thread always at same tight loop stack frame
Pattern:   "exec-1" RUNNABLE at com.example.Parser.parse (line 120) — same across 3 dumps
Fix:       Add loop termination condition, fix recursive call without base case
```

### 4.5 String Memory Bloat (Heap Dump)
```
Symptom:   Heap dominated by char[] or byte[] arrays
Histogram: Millions of String / byte[] instances
Fix:       Intern strings, use StringBuilder, enable String deduplication (-XX:+UseStringDeduplication)
```

### 4.6 ClassLoader Leak (Heap Dump / Metaspace OOM)
```
Symptom:   OOM: Metaspace — after repeated redeploys in app servers
Heap Dump: Many ClassLoader instances referencing loaded classes
Pattern:   Old ClassLoaders not GC'd due to static field holding reference
Fix:       Audit static fields in libraries; use OSGi or proper isolation
```

---

## 5. Heap vs Thread Dump — Quick Reference

| Aspect | Heap Dump | Thread Dump |
|---|---|---|
| **Captures** | All objects in heap memory | All thread states + stack traces |
| **File size** | Large (100 MB – several GB) | Small (KB – few MB) |
| **Generation time** | Seconds to minutes (STW pause) | Milliseconds (very fast) |
| **JVM impact** | High (triggers full STW) | Very low |
| **Primary use** | Memory leaks, OOM errors | Deadlocks, hangs, CPU spikes |
| **Best generated with** | `jcmd GC.heap_dump` / `-XX:+HeapDumpOnOutOfMemoryError` | `jstack` / `jcmd Thread.print` |
| **Best analyzed with** | Eclipse MAT, VisualVM | FastThread, IntelliJ, VisualVM |
| **Frequency in production** | On OOM or periodically | Multiple snapshots during incident |
| **Contains lock info** | No | Yes |
| **Contains object data** | Yes | No |

---

## 6. Best Practices

### Heap Dumps
- ✅ Always enable `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=<dir>` in production.
- ✅ Use `jcmd GC.heap_dump` over `jmap` in production (less disruptive).
- ✅ Use `-dump:live` to reduce file size (excludes unreachable objects).
- ✅ Analyze with Eclipse MAT — always start with the **Leak Suspects Report**.
- ✅ Compare heap histograms over time to detect gradual leaks.
- ✅ Set a heap dump directory with enough disk space (can be several GB).
- ⚠️ Do **not** upload production heap dumps containing PII to public web tools.
- ⚠️ Heap dumps cause a **stop-the-world pause** — time them appropriately.

### Thread Dumps
- ✅ Always take **3 consecutive thread dumps** with 5–10 second intervals.
- ✅ Include timestamps in filenames: `jstack <pid> > thread_$(date +%H%M%S).txt`.
- ✅ Use `jcmd Thread.print` as the preferred method (Java 8+).
- ✅ Correlate thread `nid` with OS thread ID to find CPU-consuming threads.
- ✅ Use FastThread.io for quick visual analysis.
- ✅ Check for the **deadlock section** at the bottom of every dump first.
- ⚠️ A single thread dump is rarely enough — always take multiple.

---

## 7. References

- [Oracle — Troubleshooting Memory Leaks](https://docs.oracle.com/en/java/javase/21/troubleshoot/troubleshooting-memory-leaks.html)
- [Oracle — Thread Dump Analysis](https://docs.oracle.com/en/java/javase/21/troubleshoot/diagnostic-tools.html)
- [Eclipse MAT Documentation](https://help.eclipse.org/latest/index.jsp?topic=/org.eclipse.mat.ui.help/welcome.html)
- [VisualVM Documentation](https://visualvm.github.io/documentation.html)
- [FastThread — Thread Dump Analyzer](https://fastthread.io)
- [HeapHero — Heap Dump Analyzer](https://heaphero.io)
- [jcmd Reference — Oracle Docs](https://docs.oracle.com/en/java/javase/21/docs/specs/man/jcmd.html)

---

*Last updated: 2026 | Java 21 LTS*
