# 07 — GC Monitoring Tools

> Covers: jstat, jcmd, jmap, VisualVM, GCEasy, GCViewer, Java Flight Recorder (JFR), and reading GC logs.

---

**Navigation**

[← GC Tuning](06-gc-tuning.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Best Practices](08-best-practices.md)

---

## Table of Contents

- [7.1 jstat](#71-jstat)
- [7.2 jcmd](#72-jcmd)
- [7.3 jmap](#73-jmap)
- [7.4 VisualVM](#74-visualvm)
- [7.5 GCEasy & GCViewer](#75-gceasy--gcviewer)
- [7.6 Java Flight Recorder (JFR) & Mission Control (JMC)](#76-java-flight-recorder-jfr--mission-control-jmc)
- [7.7 Reading GC Logs](#77-reading-gc-logs)
- [Summary](#summary)

---

## 7.1 jstat

`jstat` provides **live GC statistics** from a running JVM. Essential for quick diagnosis.

### Finding the PID

```bash
jps -l                          # List all Java processes with their main class
ps aux | grep java              # Alternative
```

### Core jstat Commands

```bash
# GC summary — most useful for day-to-day monitoring
jstat -gcutil <pid> <interval_ms> <count>
jstat -gcutil 12345 1000 60     # Every 1s, 60 times = 1 minute of data

# Output columns:
#  S0    S1    E     O     M     CCS   YGC  YGCT  FGC  FGCT  CGC  CGCT   GCT
#  0.00 45.32 72.14 33.21 96.12 93.45  142  2.543   2  0.234   8  0.123  2.900
#
#  S0/S1  = Survivor 0/1 utilization %
#  E      = Eden utilization %
#  O      = Old Gen utilization %
#  M      = Metaspace utilization %
#  CCS    = Compressed class space utilization %
#  YGC    = Young GC count
#  YGCT   = Young GC total time (seconds)
#  FGC    = Full GC count
#  FGCT   = Full GC total time (seconds)
#  CGC    = Concurrent GC count (G1/ZGC)
#  GCT    = Total GC time (seconds)

# Detailed GC stats (raw bytes)
jstat -gc <pid> 1000 30

# Capacity stats
jstat -gccapacity <pid>

# New generation stats only
jstat -gcnew <pid> 500

# Old generation stats only
jstat -gcold <pid> 500

# GC cause (what triggered each GC)
jstat -gccause <pid>
```

### Reading jstat Output — What to Watch For

```
HEALTHY pattern (steady state):
  S0    S1    E     O     M     YGC  YGCT  FGC
  0.00 45.00 72.00 33.00 96.00  142  2.54   0   ← FGC = 0, O% stable

PROBLEM: Old Gen filling (premature promotion or leak)
  0.00 45.00 72.00 45.00 96.00  143  2.55   0
  0.00 45.00 72.00 58.00 96.00  145  2.60   0   ← O% growing!
  0.00 45.00 72.00 71.00 96.00  148  2.68   0
  0.00 45.00 72.00 89.00 96.00  151  2.77   1   ← FGC triggered!

PROBLEM: Eden draining too fast (high allocation rate)
  0.00 45.00 95.00 33.00 96.00  142  2.54   0
  0.00 45.00 12.00 34.00 96.00  143  2.56   0   ← YGC every sample
  0.00 45.00 88.00 35.00 96.00  144  2.58   0
```

---

## 7.2 jcmd

`jcmd` is the Swiss Army knife for JVM diagnostics — more powerful than jstat.

```bash
# List all available commands for a process
jcmd <pid> help

# GC commands
jcmd <pid> GC.run                      # Suggest a GC run (like System.gc())
jcmd <pid> GC.heap_info                # Current heap usage summary
jcmd <pid> GC.heap_dump /path/heap.hprof  # Take a heap dump

# VM info
jcmd <pid> VM.flags                    # All JVM flags (including defaults)
jcmd <pid> VM.system_properties        # System properties
jcmd <pid> VM.version                  # JVM version info
jcmd <pid> VM.uptime                   # JVM uptime

# Thread info
jcmd <pid> Thread.print                # Thread dump (like jstack)
jcmd <pid> Thread.print -l             # Include lock info

# Compiler info
jcmd <pid> Compiler.queue             # JIT compilation queue
jcmd <pid> Compiler.codecache         # Code cache usage

# Metaspace
jcmd <pid> VM.metaspace               # Detailed Metaspace breakdown
jcmd <pid> VM.metaspace -verbose      # Per-ClassLoader breakdown (Java 16+)

# JFR (Java Flight Recorder)
jcmd <pid> JFR.start name=myrecording duration=60s filename=/tmp/recording.jfr
jcmd <pid> JFR.dump name=myrecording filename=/tmp/recording.jfr
jcmd <pid> JFR.stop name=myrecording
```

### jcmd GC.heap_info Output

```
$ jcmd 12345 GC.heap_info

 garbage-first heap   total 8388608K, used 2097152K [0x00000006c0000000, 0x00000006c0400000)
  region size 4096K, 256 young (1048576K), 48 survivors (196608K)
 Metaspace       used 87456K, committed 88576K, reserved 1114112K
  class space    used 10864K, committed 11264K, reserved 1048576K
```

---

## 7.3 jmap

`jmap` generates heap dumps and histogram reports.

```bash
# Heap histogram — shows object counts and sizes by class
jmap -histo <pid>             # All objects (including dead — before GC)
jmap -histo:live <pid>        # Live objects only (triggers a Full GC first!)

# Histogram output format:
#  num     #instances    #bytes  class name
#    1:       1523456  36562944  [B                    (byte arrays)
#    2:        892341  21376184  java.lang.String
#    3:        245123  11765904  java.util.HashMap$Node
#    4:         89234   7138720  [Ljava.lang.Object;

# Take a heap dump (live objects only — triggers Full GC)
jmap -dump:live,format=b,file=/dumps/heap.hprof <pid>

# Take a heap dump (all objects including dead)
jmap -dump:format=b,file=/dumps/heap-all.hprof <pid>

# Heap summary
jmap -heap <pid>              # Overall heap structure and usage
```

> ⚠️ `jmap -histo:live` and `jmap -dump:live` trigger a **Full GC** before collecting data. Use with caution in production — the Full GC pause can impact users. Prefer `jcmd GC.heap_dump` which is more controlled.

---

## 7.4 VisualVM

VisualVM is a free GUI tool for local and remote JVM monitoring.

### Setup

```bash
# Download: https://visualvm.github.io/
# Or: install via SDKMAN
sdk install visualvm

# Launch
visualvm &

# Connect to remote JVM — add JMX flags to target JVM:
-Dcom.sun.management.jmxremote
-Dcom.sun.management.jmxremote.port=9010
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false
```

### Key VisualVM Features for GC

| Tab | What to Look For |
|---|---|
| **Monitor** | Heap used vs committed over time; GC activity frequency |
| **Visual GC** (plugin) | Real-time animated view of Eden/Survivor/Old Gen filling and GC |
| **Profiler → Memory** | Allocation profiling — which classes allocate most |
| **Sampler → Memory** | Live heap object counts by class |
| **Heap Dump** | Load .hprof files; OQL queries; reference chains |

### Visual GC Plugin

The Visual GC plugin provides a live, colour-coded view of all heap regions:

```
Visual GC:
┌──────────────────────────────────────────────────────┐
│ Spaces         │ Graphs                              │
│ Eden: ████░░░  │ ▁▂▃▄▅▆▇█▁▂▃ Eden fill over time    │
│ S0:   ░░░░░░   │ ▁▁▁▁▁▁█▁▁▁▁ S0 occupation          │
│ S1:   █████░   │                                     │
│ Old:  ███░░░   │ ▁▁▁▁▁▁▁▂▂▃▃ Old Gen growth          │
│ Meta: ████░░   │                                     │
└──────────────────────────────────────────────────────┘
```

---

## 7.5 GCEasy & GCViewer

### GCEasy (web-based — gceasy.io)

Upload your GC log file for automated analysis:

```
What GCEasy provides:
  ✅ GC pause time distribution (percentiles: p50, p95, p99, max)
  ✅ Throughput % (time spent in app vs GC)
  ✅ Heap usage over time chart
  ✅ GC cause breakdown (Normal, Humongous, Concurrent Mode Failure, etc.)
  ✅ Problem detection (excessive Full GC, long pauses, Metaspace issues)
  ✅ JVM flag recommendations

How to use:
  1. Enable GC logging: -Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags
  2. Upload gc.log to https://gceasy.io
  3. Review the automated analysis report
```

### GCViewer (desktop)

Open-source Java desktop application for visualising GC log files:

```bash
# Download: https://github.com/chewiebug/GCViewer
java -jar gcviewer.jar /path/to/gc.log

# Key charts:
#   - Heap before/after each GC
#   - GC pause duration over time
#   - Full GC events highlighted in red
#   - Throughput % over time
```

---

## 7.6 Java Flight Recorder (JFR) & Mission Control (JMC)

JFR is the most comprehensive profiling tool — low overhead (~1%) and captures GC, CPU, threading, I/O, allocations, and more.

### Starting JFR

```bash
# Option 1: At JVM startup
java -XX:+FlightRecorder \
     -XX:StartFlightRecording=duration=120s,filename=/tmp/app.jfr \
     -jar app.jar

# Option 2: Attach to running JVM via jcmd
jcmd <pid> JFR.start name=myrecording maxsize=100m maxage=5m
jcmd <pid> JFR.dump name=myrecording filename=/tmp/snapshot.jfr
jcmd <pid> JFR.stop name=myrecording

# Option 3: Continuous recording with ring buffer (no disk until dumped)
jcmd <pid> JFR.start name=continuous settings=profile maxsize=250m
# Later, when problem occurs:
jcmd <pid> JFR.dump name=continuous filename=/tmp/incident.jfr
```

### Analysing with Mission Control (JMC)

```bash
# Download: https://adoptium.net/jmc/
jmc &
# File → Open → select .jfr file
```

### Key JFR GC Events to Analyse

| Event | What it shows |
|---|---|
| `jdk.GCHeapSummary` | Heap usage before/after each GC |
| `jdk.GarbageCollection` | Duration, cause, and type of each GC |
| `jdk.G1GarbageCollection` | G1-specific phase breakdown |
| `jdk.ObjectAllocationInNewTLAB` | Large allocation spikes — top allocating classes |
| `jdk.OldObjectSample` | Old objects in heap — memory leak candidates |
| `jdk.Safepoint` | Safepoint entry/exit and operations |
| `jdk.ExecutionSample` | CPU flame graph — allocation hot spots |

### JFR GC Allocation Profiling

```bash
# Find top 20 allocation hot spots (classes + stack traces)
# In JMC: Memory → Allocation Pressure tab
# Or: jfr print --events jdk.ObjectAllocationInNewTLAB app.jfr | head -100
```

---

## 7.7 Reading GC Logs

### G1 GC Log Examples (Java 9+ unified format)

```
# Minor (Young) GC — normal
[0.500s][gc] GC(1) Pause Young (Normal) (G1 Evacuation Pause)
[0.500s][gc] GC(1) Pause Young (Normal) (G1 Evacuation Pause) 512M->256M(2048M) 23.456ms
#                                                              ↑Before ↑After   ↑Pause

# Young GC — concurrent marking triggered
[5.000s][gc] GC(20) Pause Young (Concurrent Start) (G1 Humongous Allocation) 1024M->512M(2048M) 35.123ms
#                            ↑ concurrent marking will start after this

# Concurrent marking
[5.035s][gc] GC(20) Concurrent Cycle
[5.035s][gc] GC(20) Concurrent Mark From Roots
[6.123s][gc] GC(20) Concurrent Mark From Roots 1088.234ms
[6.123s][gc] GC(20) Pause Remark 1200M->1200M(2048M) 12.345ms
[6.135s][gc] GC(20) Pause Cleanup 1200M->1150M(2048M) 3.456ms
[6.139s][gc] GC(20) Concurrent Cycle 1104.123ms

# Mixed GC — collecting Young + selected Old regions
[6.200s][gc] GC(21) Pause Young (Mixed) (G1 Evacuation Pause) 1150M->900M(2048M) 45.678ms
#                           ↑ "Mixed" = Young + Old regions

# ALERT: Full GC (should be rare!)
[120.000s][gc] GC(100) Pause Full (G1 Compaction Pause) 7000M->4500M(8192M) 8234.567ms
#                       ↑ FULL GC — 8.2 second pause!!! Investigate root cause
```

### ZGC Log Examples

```
# ZGC phases (all STW < 1ms)
[1.000s][gc] GC(1) Garbage Collection (Proactive)
[1.001s][gc] GC(1) Pause Mark Start 0.345ms    ← < 1ms STW!
[1.500s][gc] GC(1) Pause Mark End 0.234ms      ← < 1ms STW!
[1.501s][gc] GC(1) Pause Relocate Start 0.123ms ← < 1ms STW!
[2.100s][gc] GC(1) Garbage Collection (Proactive) 1234M->456M(8192M) 1100ms
#                                                 ↑Before ↑After    ↑Concurrent time (not pause)
```

### Common GC Log Warning Patterns

```
# 1. Concurrent Mode Failure (G1) — Old Gen filling up, fallback to Full GC
Pause Full (G1 Compaction Pause)

# 2. Humongous Allocation forcing early marking
Pause Young (Concurrent Start) (G1 Humongous Allocation)

# 3. Evacuation Failure — couldn't copy all objects (Survivor/Old too full)
Pause Young (Normal) (G1 Evacuation Pause) ... Evacuation Failure

# 4. To-space exhausted (G1) — promotion failed
To-space exhausted
```

### GC Pause Percentile Analysis

```bash
# Extract all pause times from G1 GC log
grep "Pause Young\|Pause Mixed\|Pause Full\|Pause Remark\|Pause Cleanup" gc.log \
  | grep -oP '\d+\.\d+ms' \
  | sort -n \
  | awk 'BEGIN{c=0} {a[c++]=$1} END{
      print "Count: " c
      print "p50: " a[int(c*0.50)]
      print "p95: " a[int(c*0.95)]
      print "p99: " a[int(c*0.99)]
      print "Max: " a[c-1]
    }'
```

---

## Summary

| Tool | Type | Best For | Requires |
|---|---|---|---|
| `jstat` | CLI | Quick live GC stats | JVM PID |
| `jcmd` | CLI | Heap dumps, flags, thread dumps | JVM PID |
| `jmap` | CLI | Heap histograms, heap dumps | JVM PID |
| VisualVM | GUI | Visual real-time monitoring, heap dump analysis | Local/JMX |
| GCEasy | Web | Automated GC log analysis + recommendations | GC log file |
| GCViewer | Desktop | GC log visualization | GC log file |
| JFR + JMC | GUI/CLI | Deep profiling (allocations, CPU, GC, I/O) | Java 11+ |

---

**Navigation**

[← GC Tuning](06-gc-tuning.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → Best Practices](08-best-practices.md)

---

*Last updated: 2026 | Java 21 LTS*
