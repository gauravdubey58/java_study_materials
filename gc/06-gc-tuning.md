# 06 — GC Tuning

> Covers: Collector selection flags, heap sizing, generation ratios, G1/ZGC/Parallel tuning, GC logging flags, and heap sizing formulas.

---

**Navigation**

[← Garbage Collectors](05-garbage-collectors.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → GC Monitoring Tools](07-gc-monitoring-tools.md)

---

## Table of Contents

- [6.1 Selecting a Collector](#61-selecting-a-collector)
- [6.2 Heap Sizing Flags](#62-heap-sizing-flags)
- [6.3 Generation Sizing](#63-generation-sizing)
- [6.4 G1 Tuning Flags](#64-g1-tuning-flags)
- [6.5 ZGC Tuning Flags](#65-zgc-tuning-flags)
- [6.6 Parallel GC Tuning Flags](#66-parallel-gc-tuning-flags)
- [6.7 GC Logging](#67-gc-logging)
- [6.8 Heap Sizing Formulas](#68-heap-sizing-formulas)
- [6.9 Complete Production JVM Flag Templates](#69-complete-production-jvm-flag-templates)

---

## 6.1 Selecting a Collector

```bash
# Serial — single-threaded, small heaps
-XX:+UseSerialGC

# Parallel — multi-threaded STW, highest throughput
-XX:+UseParallelGC

# G1 — balanced, default Java 9+
-XX:+UseG1GC

# ZGC — sub-ms pauses, large heaps, Java 15+
-XX:+UseZGC

# Shenandoah — low latency, OpenJDK only
-XX:+UseShenandoahGC

# Epsilon — no-op (testing only)
-XX:+UnlockExperimentalVMOptions -XX:+UseEpsilonGC
```

---

## 6.2 Heap Sizing Flags

```bash
# Initial and maximum heap size
-Xms512m              # Minimum (initial) heap — JVM starts with this
-Xmx4g                # Maximum heap — JVM never exceeds this

# Production best practice: set Xms == Xmx to avoid heap resize pauses
-Xms8g -Xmx8g

# Young generation size (overrides NewRatio)
-Xmn2g                # Fixed Young Gen size (2 GB)

# Stack size per thread (smaller = more threads per JVM)
-Xss256k              # Default ~512k; reduce for thread-heavy apps
```

### Xms vs Xmx

```
-Xms < -Xmx (variable heap):
  JVM starts small → heap grows as needed → potential pauses during resize
  Good for: development, uncertain workloads
  
-Xms == -Xmx (fixed heap):
  JVM commits all memory at startup → no resize pauses
  Good for: production services with known memory requirements
  Trade-off: reserves memory even if unused
```

---

## 6.3 Generation Sizing

```bash
# Young:Old ratio
-XX:NewRatio=2         # Old:Young = 2:1 → Young = 33% of heap (default)
-XX:NewRatio=3         # Old:Young = 3:1 → Young = 25% of heap

# Eden:Survivor ratio (within Young Gen)
-XX:SurvivorRatio=8    # Eden:S0:S1 = 8:1:1 → Eden = 80% of Young (default)
-XX:SurvivorRatio=6    # Eden:S0:S1 = 6:1:1 → larger Survivors

# Object age before promotion
-XX:MaxTenuringThreshold=15    # Default 15 (range 1–15)
                               # Lower → promotes earlier → objects spend less time in Young Gen
                               # Higher → keeps objects in Young Gen longer

# Print age distribution (helpful for tuning)
-Xlog:gc+age=trace     # Java 9+ (replaces -XX:+PrintTenuringDistribution)
```

### Diagnosing Survivor Space Problems

```bash
# Too-small Survivors → premature promotion → Old Gen fills fast
# Signs:
#   - jstat shows O% (Old Gen) growing steadily
#   - Frequent Full GCs despite most objects being short-lived
# Fix:
-XX:SurvivorRatio=6    # Larger Survivors
-Xmn3g                 # Bigger Young Gen overall
```

---

## 6.4 G1 Tuning Flags

```bash
# Pause time target (soft goal)
-XX:MaxGCPauseMillis=200       # Default 200ms; lower for more responsive apps
                               # G1 will collect fewer Old regions per Mixed GC to meet this

# Region size (must be power of 2, 1–32 MB)
-XX:G1HeapRegionSize=4m        # Default: heap / 2048, rounded to power of 2
                               # Larger regions = fewer regions = less overhead
                               # Smaller regions = finer granularity

# Humongous object threshold = 50% of region size
# -XX:G1HeapRegionSize=32m → Humongous threshold = 16 MB

# Young generation size bounds
-XX:G1NewSizePercent=20        # Min Young Gen % of heap (default 5)
-XX:G1MaxNewSizePercent=40     # Max Young Gen % of heap (default 60)

# Concurrent marking trigger
-XX:InitiatingHeapOccupancyPercent=45   # Start concurrent marking when Old Gen > 45% (default)
                                        # Lower → earlier marking → fewer Full GC fallbacks
                                        # Too low → too much concurrent GC overhead

# Mixed GC tuning
-XX:G1MixedGCLiveThresholdPercent=85   # Collect Old regions with < 85% live objects (default)
-XX:G1HeapWastePercent=5               # Stop Mixed GC when < 5% of Old regions are reclaimable
-XX:G1MixedGCCountTarget=8             # Spread Old region collection over 8 Mixed GCs (default)

# Reserve for promotion during Mixed GC
-XX:G1ReservePercent=10                # Keep 10% of heap as buffer (default)

# GC thread count
-XX:ParallelGCThreads=8        # STW GC threads (default: CPU core count)
-XX:ConcGCThreads=2            # Concurrent marking threads (default: ParallelGCThreads / 4)
```

### G1 Tuning Flow

```
Start with defaults:
  -XX:+UseG1GC -Xms8g -Xmx8g -XX:MaxGCPauseMillis=200

Monitor with GC logs. If you see:
  ┌─────────────────────────────────────────────────────────┐
  │ Problem                │ Likely Cause   │ Fix           │
  ├─────────────────────────────────────────────────────────┤
  │ Frequent Full GCs      │ IHOP too high  │ Lower IHOP    │
  │ Pauses exceed target   │ Too few regions│ Larger heap   │
  │ Humongous alloc issues │ Small regions  │ Larger regions│
  │ High Old Gen growth    │ Small Young Gen│ Increase Xmn  │
  └─────────────────────────────────────────────────────────┘
```

---

## 6.5 ZGC Tuning Flags

```bash
# Basic ZGC setup
-XX:+UseZGC
-Xms16g -Xmx32g

# Soft heap limit (ZGC tries to keep heap below this)
-XX:SoftMaxHeapSize=28g        # ~85% of Xmx; allows ZGC to do more proactive GC

# Return unused memory to OS
-XX:ZUncommitDelay=300         # Wait 300s before returning idle memory (default 300)
-XX:+ZUncommit                 # Enable uncommit (default: enabled)

# GC threads
-XX:ConcGCThreads=4            # Concurrent GC threads (default: CPU / 4)

# Enable generational ZGC (Java 21, default)
-XX:+ZGenerational

# Logging
-Xlog:gc*:file=zgc.log:time,uptime,level,tags
```

### ZGC Performance Tips

```bash
# 1. Use transparent huge pages (Linux) for better TLB performance
-XX:+UseTransparentHugePages

# 2. Pin GC threads to NUMA nodes for NUMA systems
-XX:+UseNUMA

# 3. For latency-critical apps: eliminate all other STW sources
-XX:-OmitStackTraceInFastThrow    # Consistent stack traces
-XX:+AlwaysPreTouch               # Pre-fault heap pages at startup (eliminate page faults)
```

---

## 6.6 Parallel GC Tuning Flags

```bash
-XX:+UseParallelGC
-XX:ParallelGCThreads=N           # Default: CPU core count
-XX:MaxGCPauseMillis=200          # Pause target (used to size Young Gen)
-XX:GCTimeRatio=99                # App:GC CPU ratio. 99 = 1% GC overhead max (default)
                                  # Lower value = allow more GC time for faster Old Gen collection
-XX:+UseAdaptiveSizePolicy        # JVM auto-tunes Young Gen size (default: on)
-XX:-UseAdaptiveSizePolicy        # Disable auto-tuning for fixed sizing
```

---

## 6.7 GC Logging

GC logging is essential for diagnosing memory issues. Always enable in production.

### Java 9+ Unified Logging (`-Xlog`)

```bash
# Standard GC log (recommended for production)
-Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags:filecount=10,filesize=20m

# Breakdown:
#   gc*          = all GC-related log messages
#   file=...     = log file path
#   time         = wall clock timestamp
#   uptime       = JVM uptime in seconds
#   level        = log level (debug/info/warning)
#   tags         = GC tag categories
#   filecount=10 = rotate after 10 files (rolling)
#   filesize=20m = max 20 MB per file

# Verbose (for diagnosing issues)
-Xlog:gc*,gc+heap=debug,gc+ref=debug:file=/logs/gc-verbose.log:time,uptime,tags

# Age distribution (tenuring analysis)
-Xlog:gc+age=trace:file=/logs/gc-age.log:time,uptime

# Safepoint analysis
-Xlog:safepoint*:file=/logs/safepoint.log:time,uptime
```

### Heap Dump on OOM (Production Essential)

```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/dumps/heap-$(date +%Y%m%d-%H%M%S).hprof
```

### Key Metrics to Watch in GC Logs

```
# G1 Young GC entry:
[2.500s][gc] GC(5) Pause Young (Normal) (G1 Evacuation Pause) 512M->256M(2048M) 45.231ms
                                                               ↑before  ↑after   ↑pause

# G1 Concurrent Marking:
[5.123s][gc] GC(8) Concurrent Cycle
[7.456s][gc] GC(8) Concurrent Cycle 2333ms

# G1 Full GC (BAD — investigate if frequent):
[60.123s][gc] GC(42) Pause Full (G1 Compaction Pause) 7168M->5120M(8192M) 8456ms
                                                                             ↑ 8.4 seconds!
```

---

## 6.8 Heap Sizing Formulas

### General Rule

```
Heap Size = Live Data Set × Safety Multiplier

Live Data Set = heap usage after a Full GC (when only live objects remain)
Safety Multiplier = 3x for G1/ZGC; 4x for Parallel (room for allocation between GCs)

Example:
  Live data = 2 GB after Full GC
  With G1: Xmx = 2 GB × 3 = 6 GB
```

### Young Generation Sizing

```
Young Gen too small:
  → Frequent Minor GCs → high GC overhead
  → Premature promotion → Old Gen grows fast → frequent Full GCs

Young Gen too large:
  → Infrequent but long Minor GC pauses
  → Less room for Old Gen

Guideline:
  Young Gen = 25–33% of total heap
  (Default NewRatio=2 → Young = 33%)

For high-allocation apps (web services):
  Young Gen = 40–50% of heap
  -XX:G1NewSizePercent=30 -XX:G1MaxNewSizePercent=50
```

### Container / Kubernetes Sizing

```bash
# Java 10+: JVM respects container memory limits
-XX:+UseContainerSupport               # Default: enabled in Java 10+
-XX:MaxRAMPercentage=75.0              # Use 75% of container RAM as heap
-XX:InitialRAMPercentage=50.0          # Start heap at 50% of container RAM
-XX:MinRAMPercentage=25.0              # Minimum heap for small containers

# Example: 4 GB container
# Heap: 4 GB × 75% = 3 GB → -Xmx3g equivalent
# Non-heap (Metaspace, stacks, direct buffers) needs the remaining 1 GB
```

---

## 6.9 Complete Production JVM Flag Templates

### G1 GC — General Purpose Web Service

```bash
java \
  -server \
  -XX:+UseG1GC \
  -Xms8g -Xmx8g \
  -XX:MaxGCPauseMillis=200 \
  -XX:G1HeapRegionSize=16m \
  -XX:InitiatingHeapOccupancyPercent=35 \
  -XX:G1NewSizePercent=20 \
  -XX:G1MaxNewSizePercent=40 \
  -XX:MaxMetaspaceSize=512m \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/dumps/ \
  -Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags:filecount=10,filesize=20m \
  -jar app.jar
```

### ZGC — Low Latency Service

```bash
java \
  -server \
  -XX:+UseZGC \
  -Xms16g -Xmx32g \
  -XX:SoftMaxHeapSize=28g \
  -XX:ZUncommitDelay=300 \
  -XX:MaxMetaspaceSize=512m \
  -XX:+AlwaysPreTouch \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/dumps/ \
  -Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags:filecount=10,filesize=20m \
  -jar app.jar
```

### Parallel GC — Batch Processing

```bash
java \
  -server \
  -XX:+UseParallelGC \
  -Xms4g -Xmx4g \
  -XX:ParallelGCThreads=8 \
  -XX:MaxGCPauseMillis=500 \
  -XX:GCTimeRatio=19 \
  -XX:MaxMetaspaceSize=256m \
  -XX:+HeapDumpOnOutOfMemoryError \
  -Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags:filecount=5,filesize=10m \
  -jar batch.jar
```

### Container / Kubernetes (Java 11+)

```bash
java \
  -XX:+UseG1GC \
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:MaxMetaspaceSize=256m \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/tmp/ \
  -Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags:filecount=5,filesize=10m \
  -jar app.jar
```

---

**Navigation**

[← Garbage Collectors](05-garbage-collectors.md) | [🏠 Home](README.md) | [📑 Index](INDEX.md) | [Next → GC Monitoring Tools](07-gc-monitoring-tools.md)

---

*Last updated: 2026 | Java 21 LTS*
