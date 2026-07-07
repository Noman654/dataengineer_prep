# 🧠 Memory Management — Self-Check Quiz

**How to use this:** Read the question. Say your answer out loud like you're in an interview. Then click to expand.

**Paired with:** [`../theory/memory_management.md`](../theory/memory_management.md)

**Difficulty mix:** 🟢 basics → 🟡 intermediate → ⚡ senior

---

## 🟢 Basics

<details>
<summary><strong>Q1.</strong> Name the three regions inside Spark's JVM executor heap.</summary>

<br>

1. **Reserved Memory** — 300 MB hardcoded, for internal Spark objects.
2. **User Memory** — ~40% of (heap − 300 MB). For your own data structures, UDFs, variables.
3. **Unified Memory** — ~60% of (heap − 300 MB), per `spark.memory.fraction`. Where Spark does its real work; shared between Execution and Storage.

</details>

<details>
<summary><strong>Q2.</strong> Execution vs Storage memory — what's the difference, and who wins when they fight?</summary>

<br>

- **Execution** = shuffle, join, sort, aggregation buffers. Can't be evicted mid-operation.
- **Storage** = cached DataFrames/RDDs, broadcast variables. Can be evicted (just re-compute).

**Execution wins when contested** — it can kick cached blocks out to reclaim space. That's why over-caching on a shuffle-heavy job is self-defeating: your cache gets evicted immediately.

`spark.memory.storageFraction = 0.5` guarantees Storage half of Unified — below that threshold, Execution can't evict.

</details>

<details>
<summary><strong>Q3.</strong> What is <code>spark.executor.memoryOverhead</code> and what's the default?</summary>

<br>

**Off-heap memory per executor** — lives outside the JVM heap. Hosts:
- JVM native memory (metaspace, thread stacks)
- Netty network buffers
- **In PySpark: the Python worker processes**
- Off-heap execution/storage (if enabled)

**Default:** `max(384 MB, 10% of executor memory)`.

For PySpark, almost always too low — Python workers easily eat more than 384 MB. Bump it explicitly.

</details>

---

## 🟡 Intermediate

<details>
<summary><strong>Q4.</strong> Why do PySpark jobs OOM differently from Scala Spark jobs?</summary>

<br>

In Scala, data lives in the JVM heap. In PySpark:

1. Driver sends Python code to executors.
2. Each executor JVM **spawns Python worker processes** (one per task slot).
3. Data is serialized across a socket between JVM and Python (or Arrow batches if enabled).
4. Python workers run your UDF / `mapInPandas` / RDD map logic.

**Python workers live in `memoryOverhead`, NOT the JVM heap.** So the JVM can look healthy while YARN kills the container because Python overran overhead. The classic error:

```
Container killed by YARN for exceeding memory limits.
Consider boosting spark.executor.memoryOverhead.
```

**Fix:** bump `memoryOverhead` or dedicate `spark.executor.pyspark.memory`. Enable Arrow (`spark.sql.execution.arrow.pyspark.enabled=true`) to reduce Python memory pressure.

</details>

<details>
<summary><strong>Q5.</strong> What does it mean when Spark reports non-zero "Shuffle Spill"?</summary>

<br>

Execution memory ran out during a shuffle/sort/aggregate, so Spark **wrote the in-memory buffer to local disk** and kept going.

**Not fatal** — you didn't OOM. **But it signals under-provisioning.**

- "Shuffle spill (memory)" = bytes in memory before spill
- "Shuffle spill (disk)" = actual disk bytes written (smaller, compressed)

**Fix:** bump `executor.memory`, reduce per-task data (more partitions), or enable off-heap memory.

**Caching is different:** `MEMORY_ONLY` drops blocks when full — no spill. Use `MEMORY_AND_DISK` (default for `cache()`) to spill cached data.

</details>

<details>
<summary><strong>Q6.</strong> <code>cache()</code> vs <code>persist()</code> vs <code>checkpoint()</code>. When do you use each?</summary>

<br>

| Method | Storage | Lineage | Use case |
|---|---|---|---|
| `cache()` | = `persist(MEMORY_AND_DISK)` | Kept | Reuse a DF across actions, don't care about storage level |
| `persist(level)` | Your choice | Kept | When you need a specific storage level (MEMORY_ONLY_SER, DISK_ONLY, etc.) |
| `checkpoint()` | Reliable storage (HDFS/S3) | **Truncated** | Very long lineages (iterative ML, 50+ transformation chains) where re-computation on failure would be expensive |

**Key distinction:** cache/persist keep the DAG intact (fast, but re-compute if executor dies). Checkpoint replaces the lineage with a read from storage (slower write, but survives failures).

**Gotcha:** `cache()` is **lazy** — nothing is cached until the first action runs.

</details>

<details>
<summary><strong>Q7.</strong> Why is <code>.collect()</code> on a 100M-row DataFrame a bad idea?</summary>

<br>

It sends every row to the **driver**, which holds them in memory as a single Python/Scala list.

**What breaks:**
- Driver OOM (heap wasn't sized for the whole dataset)
- `spark.driver.maxResultSize` (default 1 GB) aborts the job with "Total size of serialized results exceeds..."
- Network bottleneck as all executors stream data to one node
- You lose distribution — your "Spark job" is now a single-machine Python program

**Better alternatives:**
- `.write.parquet(path)` — keep it distributed
- `.toPandas()` on an *aggregated* result (100 rows, not 100M)
- `.take(n)` if you just want a sample
- `mapInPandas` to stay distributed while using pandas code

</details>

---

## ⚡ Senior / interview judgment

<details>
<summary><strong>Q8.</strong> Walk through executor sizing for a cluster of 10 nodes × 16 cores × 64 GB RAM.</summary>

<br>

**Step 1 — Leave room for OS + node manager:**
- Usable per node: **15 cores, 63 GB** (subtract 1 core, 1 GB)

**Step 2 — Pick executor cores:**
- **5 cores per executor** is the sweet spot. More than 5 hurts HDFS/S3 throughput (too many concurrent connections). Fewer than 5 wastes broadcast efficiency (broadcasts are per-executor).

**Step 3 — Executors per node:**
- `15 cores / 5 = 3 executors per node`
- Total executors: `3 × 10 = 30`

**Step 4 — Memory per executor:**
- `(63 GB / 3 executors) × 0.9 = ~18.9 GB heap`
- The × 0.9 leaves room for `memoryOverhead`
- Overhead: `2 GB` explicit (bump higher for PySpark)

**Final config:**
```
--num-executors 30
--executor-cores 5
--executor-memory 18g
--conf spark.executor.memoryOverhead=2g
```

**Anti-patterns to call out:**
- Fat executor (1 per node, all cores) → long GC pauses.
- Thin executor (1 core each) → broadcasts duplicated everywhere, no parallelism benefit inside the executor.

</details>

<details>
<summary><strong>Q9.</strong> Your job fails with <code>java.lang.OutOfMemoryError: GC overhead limit exceeded</code>. What does this mean and how do you diagnose?</summary>

<br>

**What it means:** the JVM is spending >98% of its time doing GC and reclaiming <2% of heap. It's stuck in a death spiral — doing work, producing garbage, GC'ing, producing more garbage.

**Diagnose:**
1. **Check Spark UI → Executors tab → GC Time column.** If >10% of task time is GC, you have memory pressure.
2. **Look at what's allocated:** are you holding large collections? Calling `collect()`? Creating millions of objects in a UDF?
3. **Check the stage.** If it's a shuffle stage, you're probably under-partitioned (each task holds too much data).

**Fixes, in order:**
1. Bump executor memory.
2. Reduce per-task data: increase `spark.sql.shuffle.partitions`.
3. Switch to G1GC: `-XX:+UseG1GC` (better for large heaps).
4. Rewrite UDFs to use built-ins (Catalyst can't optimize UDF memory usage).
5. Enable off-heap memory (`spark.memory.offHeap.enabled=true`) — escapes GC entirely.

</details>

<details>
<summary><strong>Q10.</strong> When would you use <code>StorageLevel.OFF_HEAP</code>?</summary>

<br>

**When:** caching large DataFrames on a machine with lots of RAM where GC pauses are hurting performance.

**How:** off-heap storage lives in native memory, managed by Spark directly — not the JVM garbage collector. No GC pauses, no competing with execution for heap.

**Required configs:**
```
spark.memory.offHeap.enabled=true
spark.memory.offHeap.size=10g   # explicit size; doesn't come from executor.memory
```

Then:
```python
df.persist(StorageLevel.OFF_HEAP)
```

**Tradeoffs:**
- **Pro:** no GC pressure, more predictable latency.
- **Con:** serialized-only (can't hold Java objects), more CPU to deserialize on read.
- **Con:** harder to tune — you're managing memory outside the JVM's view.

**Real-world use:** iterative ML jobs caching intermediate datasets, or long-running Spark Streaming jobs where GC pauses were causing missed microbatch deadlines. Rarely needed for batch jobs.

</details>

---

## 🎯 How did you do?

- **9–10 correct** → interview-ready on memory. Drill [shuffle theory](../theory/shuffle_and_partitioning.md) next.
- **6–8 correct** → solid grasp. Re-read [memory_management.md](../theory/memory_management.md) sections you missed.
- **< 6 correct** → start with the [Luminousmen deep dive](https://luminousmen.com/post/dive-into-spark-memory/), then re-read the theory doc.

**Related:** [quiz/joins.md](joins.md), [quiz/window_functions.md](window_functions.md)
