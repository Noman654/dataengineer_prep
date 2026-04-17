# Memory Management — Quick Review

**Audience:** senior DE interview prep. Assumes you've written PySpark for a year or two and have hit at least one OOM in production.

**Read time:** 10–15 minutes.

---

## TL;DR — the one-sentence mental model

> An executor's JVM heap is split into **Reserved + User + Unified** regions. **Unified** is where Spark does its real work — it's shared between **Execution** (shuffle/join/sort buffers) and **Storage** (cached data), and they can borrow from each other. On top of the heap, there's **off-heap overhead** for Python workers, network buffers, and native code — which is where PySpark jobs usually die.

If you remember nothing else, remember: **the JVM heap is not all of Spark's memory, and in PySpark most OOMs happen outside the heap.**

---

## The executor memory model (Unified Memory, Spark 1.6+)

Total executor memory = `spark.executor.memory` (JVM heap) + `spark.executor.memoryOverhead` (off-heap).

Inside the JVM heap:

```
+-------------------------------------------------------------+
|  Reserved Memory  (hardcoded 300 MB — internal Spark objects)|
+-------------------------------------------------------------+
|  User Memory      (~25% of heap - 300MB)                     |
|                   RDDs you hold as variables, UDFs,          |
|                   data structures in your code               |
+-------------------------------------------------------------+
|  Unified Memory   (~75% of heap - 300MB = spark.memory.fraction)|
|                                                               |
|   +-------------------+-------------------+                   |
|   |  Execution Memory |  Storage Memory   |                   |
|   |  shuffle, join,   |  cached DFs/RDDs, |                   |
|   |  sort, aggregate  |  broadcast vars   |                   |
|   |  buffers          |                   |                   |
|   +-------------------+-------------------+                   |
|            ^ boundary is DYNAMIC — they borrow from each other|
+-------------------------------------------------------------+
```

Outside the heap (`spark.executor.memoryOverhead`):
- JVM native memory (metaspace, thread stacks)
- **Python worker processes** (PySpark) — each task spawns a Python worker
- Network buffers (Netty)
- Off-heap execution/storage if `spark.memory.offHeap.enabled=true`

**Default overhead:** `max(384 MB, 0.10 × executor memory)`. For PySpark, this is almost always too low.

---

## Execution vs Storage — and why "unified" matters

Before Spark 1.6, Execution and Storage had fixed boundaries. Cached data could sit idle while a shuffle starved and OOM'd. **Unified Memory fixed this:**

- **Execution can evict Storage** (kick cached blocks out) when it needs room.
- **Storage can borrow unused Execution** space.
- `spark.memory.storageFraction = 0.5` — the fraction of Unified guaranteed to Storage (below this, Execution can't evict it).

**Why Execution wins when contested:** evicting a cached block is cheap (recompute later). Stopping an in-flight shuffle is not.

**Practical implication:** over-caching on a shuffle-heavy job is self-defeating. Your cache gets evicted the moment Execution needs room.

---

## The key configs (memorize these)

| Config | Default | What it controls |
|---|---|---|
| `spark.executor.memory` | 1g | JVM heap size per executor |
| `spark.executor.memoryOverhead` | max(384M, 10% of executor.memory) | Off-heap space — **Python workers live here** |
| `spark.executor.pyspark.memory` | unset | Dedicated Python worker memory (Spark 2.4+). Prevents Python from eating overhead. |
| `spark.memory.fraction` | 0.6 | Fraction of (heap − 300M) used for Unified Memory |
| `spark.memory.storageFraction` | 0.5 | Fraction of Unified that Storage is guaranteed (can't be evicted below this) |
| `spark.memory.offHeap.enabled` | false | Use off-heap for Execution + Storage (escapes GC) |
| `spark.memory.offHeap.size` | 0 | Size of off-heap pool (only matters if enabled) |
| `spark.driver.memory` | 1g | Driver heap — watch this for `.collect()`/`.toPandas()` |
| `spark.driver.maxResultSize` | 1g | Total bytes all tasks can return to driver. Safety net against `.collect()` OOM. |

---

## PySpark is different — where most PySpark OOMs actually happen

In Scala Spark, your data lives in the JVM heap and executes directly. In PySpark:

1. The driver sends your Python code to executors.
2. Each executor JVM launches **Python worker processes** (one per task slot).
3. Data is **serialized across a socket** between JVM and Python (or via Arrow if enabled).
4. Python runs your UDF / `mapInPandas` / RDD map logic.
5. Result is sent back to JVM over the socket.

**The Python workers live in `memoryOverhead`, not the JVM heap.** So the classic PySpark failure is:

```
Container killed by YARN for exceeding memory limits. 5.2 GB of 5 GB physical memory used.
Consider boosting spark.executor.memoryOverhead.
```

JVM is fine. Python ate the overhead. Fix:

```python
spark.conf.set("spark.executor.memoryOverhead", "2g")
# or dedicate a pool specifically to Python:
spark.conf.set("spark.executor.pyspark.memory", "1500m")
```

**Arrow helps** (`spark.sql.execution.arrow.pyspark.enabled=true`) — batches instead of row-at-a-time serialization, much lower Python memory pressure.

---

## Spill-to-disk — the pressure valve

When Execution memory runs out during a shuffle/sort/aggregate, Spark **spills** the in-memory buffer to local disk and keeps going. You don't OOM — you just get slow.

- **Shuffle spill (memory)** — bytes of spilled data measured in memory
- **Shuffle spill (disk)** — actual disk bytes written (smaller, because serialized + compressed)

**In the Spark UI:** both appear on the Stage page. Non-zero spill isn't fatal but means you're under-provisioned for the workload. Frequent spill = either bump `executor.memory` or reduce per-task data volume (more partitions).

**Caching does NOT spill to disk by default** — `MEMORY_ONLY` drops blocks when full. Use `MEMORY_AND_DISK` (the default for `DataFrame.cache()`) to spill cached data instead of dropping it.

---

## OOM patterns — how to diagnose from the error message

| Error | What it means | Fix |
|---|---|---|
| `Container killed by YARN, exceeded memory limits` | Off-heap overrun, usually Python workers or Netty | Bump `executor.memoryOverhead` (or set `pyspark.memory`) |
| `java.lang.OutOfMemoryError: Java heap space` | JVM heap exhausted. Big shuffle, big broadcast, or skew | Bump `executor.memory`; check for skew; reduce broadcast |
| `java.lang.OutOfMemoryError: GC overhead limit exceeded` | Spending >98% of CPU in GC, reclaiming <2%. Death spiral. | More memory, or switch to G1GC, or reduce object churn |
| `Total size of serialized results exceeds spark.driver.maxResultSize` | You called `.collect()` / `.toPandas()` on too much data | Don't collect — write to storage instead. Or bump `maxResultSize` if legit. |
| `Driver OutOfMemoryError` | Driver heap too small, usually `.collect()` / `.toPandas()` / too many broadcast vars | Bump `driver.memory`; stop collecting large data |
| `BufferHolder ... cannot grow beyond size 2147483632` | A single record exceeded 2 GB (Spark's int-indexed buffer) | One row is too big — nested data/arrays blowing up. Restructure schema. |

---

## Executor sizing — the "fat vs thin executor" question

You have N nodes, each with X cores and Y GB RAM. How do you size executors? This comes up in every senior interview.

**Rules of thumb:**

1. Leave **1 core and ~1 GB per node** for OS + node manager daemons.
2. **5 cores per executor** is the sweet spot. More than 5 causes HDFS/S3 throughput to drop (too many concurrent connections). Less than 5 underutilizes broadcast variables (they're per-executor, not per-core).
3. **Memory per executor** = `(usable_node_mem) / (executors_per_node) × 0.9` — the 0.9 leaves room for `memoryOverhead`.
4. Allocate `memoryOverhead` explicitly — don't let the 10% default bite you in PySpark.

**Worked example:** 10 nodes × 16 cores × 64 GB.
- Usable per node: 15 cores, 63 GB
- Executors per node: 15 / 5 = 3
- Memory per executor: 63 / 3 × 0.9 ≈ 18 GB heap + 2 GB overhead
- Total executors: 30

**Fat executor (1 per node, all cores):** easy broadcast sharing, but huge GC pauses and wastes a core on driver-side coordination.
**Thin executor (1 core each):** no concurrency benefits, broadcast vars duplicated per executor.
**Middle ground (5 cores) is almost always right.**

---

## Garbage Collection — the silent killer

Large heaps (>8 GB) with the default **ParallelGC** cause long stop-the-world pauses. Switch to **G1GC**:

```bash
--conf spark.executor.extraJavaOptions="-XX:+UseG1GC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps"
```

**In the Spark UI**, each task reports **GC Time**. If GC is >10% of task duration, you have memory pressure — either bump memory or reduce per-task data.

**Symptoms of GC trouble:** tasks hanging then resuming, executors lost and re-added, inconsistent task durations without skew.

---

## Caching — storage levels and when to use what

```python
df.cache()                                  # = MEMORY_AND_DISK (default for DataFrame)
df.persist(StorageLevel.MEMORY_ONLY)        # Fast, but drops blocks when full
df.persist(StorageLevel.MEMORY_AND_DISK)    # Spills to disk when full — safest default
df.persist(StorageLevel.MEMORY_ONLY_SER)    # Serialized — smaller but CPU cost to deserialize
df.persist(StorageLevel.DISK_ONLY)          # Disk only — slow, rarely the right answer
df.persist(StorageLevel.OFF_HEAP)           # Off-heap — requires offHeap.enabled=true
```

**`cache()` is lazy** — nothing is cached until the first action (e.g., `.count()`, `.show()`) runs.

**When to cache:** a DataFrame is used more than once AND the computation to produce it is expensive (joins, shuffles, UDFs). One-shot pipelines don't benefit.

**When NOT to cache:** write-once pipelines, DataFrames used once, or when the cache would get evicted by shuffles immediately anyway.

**`checkpoint()` vs `cache()`:** checkpoint truncates the lineage and writes to reliable storage (HDFS/S3). Cache keeps lineage; checkpoint replaces it. Use checkpoint for very long lineages that would otherwise be re-computed on failure.

---

## Interview one-liners (memorize these)

- **"How does Spark manage memory inside an executor?"** → Unified Memory model: heap is split into Reserved (300 MB), User (~25%), and Unified (~75%). Unified is shared between Execution (shuffle/join buffers) and Storage (cached data), and they can borrow from each other dynamically. Execution wins when contested.
- **"What's `memoryOverhead`?"** → Off-heap memory per executor. Hosts JVM native memory, Netty buffers, and — critically in PySpark — the Python worker processes. Default is `max(384 MB, 10% of executor memory)`, usually too low for PySpark.
- **"Why do PySpark jobs OOM differently from Scala?"** → Python workers run outside the JVM in `memoryOverhead`. A PySpark job can have a healthy JVM and still get killed by YARN for off-heap overrun.
- **"How do you size executors?"** → 5 cores per executor (HDFS/S3 sweet spot), memory = usable_node_mem / executors_per_node × 0.9, leave 1 core + 1 GB per node for OS, bump `memoryOverhead` for PySpark.
- **"What does it mean when Spark spills?"** → Execution memory ran out during shuffle/sort/aggregate, so Spark wrote the buffer to local disk and kept going. Not fatal, but signals under-provisioning.
- **"Cache vs persist vs checkpoint?"** → `cache()` = `persist(MEMORY_AND_DISK)`. `persist()` lets you pick a storage level. `checkpoint()` writes to reliable storage and truncates lineage — used for long iterative jobs.
- **"Driver OOM — what's your first suspect?"** → `.collect()`, `.toPandas()`, or too many broadcast variables. The driver holds the result of collects and the master copy of broadcasts.

---

## Common pitfalls (the stuff that bites in production)

1. **PySpark OOM with `memoryOverhead` at default.** Python workers eat 384 MB and YARN kills the container. Always set overhead explicitly for PySpark.
2. **`.toPandas()` on a billion-row DataFrame.** Pulls everything to driver. Use `.toPandas()` only after aggressive aggregation, or use `mapInPandas` to stay distributed.
3. **Over-caching in a shuffle-heavy job.** Execution evicts your cache anyway. Only cache DataFrames reused multiple times across actions.
4. **Broadcast variable > 1 GB.** The driver holds the master copy and sends it to every executor. Big broadcasts = driver OOM + network storm on job start.
5. **Fat executors (all node cores, 50+ GB heap).** Long GC pauses freeze the executor. Break into 5-core executors.
6. **Ignoring GC time in the UI.** >10% of task time in GC is memory pressure disguised as slowness.
7. **Assuming `cache()` is instant.** It's lazy — the first action triggers caching. If that action fails, nothing is cached.
8. **Not enabling Arrow for PySpark UDFs.** Row-at-a-time serialization is 10–100x slower than Arrow batches and burns Python memory.

---

## Further reading

**Official:**
- [Spark Memory Management (official tuning guide)](https://spark.apache.org/docs/latest/tuning.html#memory-management-overview)
- [Garbage Collection Tuning for Spark (official)](https://spark.apache.org/docs/latest/tuning.html#garbage-collection-tuning)
- [Unified Memory Management in Spark 1.6 (design doc)](https://issues.apache.org/jira/secure/attachment/12765646/unified-memory-management-spark-10000.pdf) — the original spec, still the clearest explanation.

**Community classics (the ones everyone shares on LinkedIn):**
- [**Deep Dive into Spark Memory Management** by Luminousmen](https://luminousmen.com/post/dive-into-spark-memory/) — probably the most-linked Spark memory post on LinkedIn. Clear diagrams of the unified memory model.
- [**Why Your Spark Applications Are Slow or Failing, Part 1: Memory Management** by Rishitesh Mishra (Unravel / DZone)](https://dzone.com/articles/common-reasons-your-spark-applications-are-slow-or) — classic troubleshooting framework. Also read [Part 2 on Skew & GC](https://dzone.com/articles/why-your-spark-apps-are-slow-or-failing-part-ii-da).
- [**Tuning Java Garbage Collection for Apache Spark Applications** — Databricks (2015)](https://www.databricks.com/blog/2015/05/28/tuning-java-garbage-collection-for-spark-applications.html) — the canonical G1GC post, still cited.
- [**Part 3: Cost-Efficient Executor Configuration for Apache Spark** by Brad Caffey (Expedia Group)](https://medium.com/expedia-group-tech/part-3-efficient-executor-configuration-for-apache-spark-b4602929262) — best writeup on the 5-cores rule and executor sizing math.
- [**Decoding Spark Executor Memory Management** by Shivank Chaturvedi (Airtel Digital)](https://medium.com/airteldigital/decoding-spark-executor-memory-management-tuning-spark-configs-17c481529ccd) — derives a heuristic formula for sizing configs end-to-end.

**Author's own:**
- [**The Ultimate Guide to Spark Memory Management**](https://www.linkedin.com/pulse/ultimate-guide-spark-memory-management-data-engineers-mohd-nauman-4mjec/) — by Mohd Nauman *(author of this repo)*.

Related in this repo: [`shuffle_and_partitioning.md`](shuffle_and_partitioning.md) — shuffle spill and skew interact heavily with memory.

---

*Last revised: 2026-04*
