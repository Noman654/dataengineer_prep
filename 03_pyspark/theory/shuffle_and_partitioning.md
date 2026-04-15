# Shuffle & Partitioning — Quick Review

**Audience:** senior DE interview prep. Assumes you've written PySpark for a year or two and want a tight mental model before the whiteboard round.

**Read time:** 10–15 minutes.

---

## TL;DR — the one-sentence mental model

> A **shuffle** writes every mapper's output to local disk, grouped by target reducer, then every reducer fetches its portion over the network. It's the most expensive thing Spark does. **Everything you tune — partitioning, broadcast joins, bucketing, AQE, salting — is ultimately about avoiding shuffles, shrinking them, or balancing them.**

If you remember nothing else, remember that.

---

## What is a shuffle, precisely

A shuffle is the **physical redistribution of data across executors** to satisfy a transformation that requires rows with the same key to land on the same partition. It happens when Spark can't execute a transformation within a single partition and has to move data across the network.

Each shuffle has two sides:

| Side | What it does |
|---|---|
| **Map (shuffle write)** | Every task of the upstream stage writes its output to local disk, bucketed by the hash of the shuffle key. Each bucket = one target partition in the downstream stage. |
| **Reduce (shuffle read)** | Every task of the downstream stage fetches its assigned bucket from every upstream node over the network, then processes. |

The Spark UI shows both sides separately: **Shuffle Write** (bytes leaving this stage) and **Shuffle Read** (bytes entering the next stage). For a healthy job, these should be roughly equal across stages.

---

## When shuffles happen (the operations to watch for)

These are **wide transformations** — they span partitions, so Spark inserts a shuffle boundary:

- `groupBy` / `groupByKey` / `reduceByKey`
- `join` (unless one side is broadcast or the data is bucketed on the join key)
- `distinct`
- `repartition(n)` / `repartition(col)`
- `orderBy` / `sort`
- `window` functions when the `partitionBy` key differs from the current partitioning
- Set operations: `union` (only if partitioning differs), `intersect`, `exceptAll`

**Narrow transformations** (no shuffle): `map`, `filter`, `select`, `withColumn`, `union` (when compatible). These pipeline inside a single stage — cheap.

**Cheat to identify stage boundaries:** each shuffle = one new stage. Count the stages in the Spark UI to count your shuffles.

---

## The shuffle lifecycle — what actually happens on disk and network

1. Upstream stage's tasks finish computing their output.
2. Each task **partitions its output by hash of the shuffle key** modulo `spark.sql.shuffle.partitions` (default `200`).
3. Each task writes its output into local disk files, one file per target partition. These are the **shuffle files**.
4. The downstream stage starts. Each of its tasks is assigned one target partition (a "reducer").
5. Each reducer task **fetches its assigned files from every upstream node over the network** (this is the expensive part).
6. The reducer combines its fetched data and processes it.

**Why it's slow:**
- Disk I/O on both sides (write on mappers, read on reducers)
- Network I/O during the fetch phase
- Serialization / deserialization
- Potential OOM during the fetch if a single reducer has to pull too much data (hello, skew)

---

## `spark.sql.shuffle.partitions` — the most tuned config in Spark

```python
spark.conf.set("spark.sql.shuffle.partitions", 200)  # default
```

This is the **number of partitions Spark produces after a shuffle** — for every `groupBy`, `join`, `window`, etc. The default of 200 is almost always wrong for your data. Tune based on actual data volume:

| Total shuffled data | Good target |
|---|---|
| < 10 GB | 50–100 partitions |
| 10 GB – 1 TB | ~128 MB per partition (so 80–8000 partitions) |
| > 1 TB | 256 MB per partition |

**Rule of thumb:** aim for **128–256 MB per task** post-shuffle. Smaller → scheduler overhead dominates. Larger → risk of OOM and stragglers.

**Adaptive Query Execution (AQE)** in Spark 3+ adjusts this automatically at runtime — if you enable it (`spark.sql.adaptive.enabled=true`, default in Spark 3.2+), you can often leave the 200 default and AQE will coalesce partitions down after looking at actual shuffle data sizes.

---

## Partitioning strategies

### Hash partitioning (the default)
`partition_id = hash(key) % num_partitions`. Used by `groupBy`, `join`, `repartition(col)`. Works fine if the key is uniformly distributed. Fails hard if it isn't (→ skew, see below).

### Range partitioning
Used by `orderBy` / `sort`. Spark samples the data to decide range boundaries, then assigns each row to a partition based on which range its key falls in. More expensive than hash but produces sorted partitions.

### Custom partitioning
Rare in DataFrame API. In the RDD API you can pass a `Partitioner`. Almost always not worth it — stick with hash unless you have a very specific reason.

---

## Partition pruning & predicate pushdown

If data on disk is **physically partitioned by a column** (e.g., `/data/transactions/year=2024/month=03/...`), then queries filtering on that column can **skip entire directories without reading them**. This is *partition pruning* — it's the single biggest speedup you can get from good partitioning.

```python
# Writing with a partition column
df.write.partitionBy("year", "month").parquet("/data/transactions/")

# Reading: Spark only reads March 2024's files
spark.read.parquet("/data/transactions/").filter("year = 2024 AND month = 3")
```

**Gotchas:**
- Partitioning on **high-cardinality columns** (like `customer_id`) creates too many small files → degraded performance. Rule: partition on columns with < ~10,000 distinct values.
- Partitioning on **multiple columns** multiplies: year × month × region = huge directory explosion. Use sparingly.
- **Predicate pushdown** is a different but related optimization — filters that can be evaluated by the file format (Parquet/ORC) are pushed down so Spark reads less data per file. Works even without directory partitioning.

---

## `repartition()` vs `coalesce()`

```python
df.repartition(100)            # shuffle, exact 100 partitions
df.repartition(100, "region")  # shuffle by hash of region, 100 partitions
df.coalesce(10)                # NO shuffle, merges existing partitions down
```

- **`repartition(n)`** — always triggers a full shuffle. Use when you need to *increase* partitions, rebalance, or re-key on a specific column.
- **`coalesce(n)`** — avoids a shuffle by merging partitions on existing executors. Use to *decrease* partitions before writing out (e.g., avoiding the small-file problem).

**Pitfall:** `coalesce(1)` to "write one file" sounds harmless but **it collapses the entire stage upstream into a single task**. If you have an expensive transformation before the coalesce, you just destroyed your parallelism. Use `repartition(1)` instead, or better, write with `.partitionBy()` and accept many files.

---

## Data skew — the senior interview topic

### What it is
Skew happens when **one key has dramatically more rows than others**. Because shuffles distribute by hash of the key, skewed keys mean one reducer task gets flooded while others sit idle. Your job's total runtime = runtime of the slowest task. One hot key can make a 5-minute job take an hour.

### How to detect it

**In the Spark UI:**
- Go to **Stages** → click the slow stage → look at **Summary Metrics** at the top
- Compare **Min / Median / Max task duration** and **Shuffle Read Size**
- If max is 10x+ median → you have skew
- Look at the **tasks table** below — one or two tasks much slower than the rest = skewed

**Programmatically:**
```python
# Check distribution of your join/groupBy key
df.groupBy("store_id").count().orderBy(F.col("count").desc()).show(20)
# If one key has 100x more rows than the median, you have skew
```

### How to fix it

#### 1. Enable AQE skew join (Spark 3+)
```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
```
AQE detects skewed partitions at runtime and splits them into smaller ones automatically. **This is the first thing to try** — often solves the problem with zero code changes.

#### 2. Broadcast join (if one side is small enough)
```python
from pyspark.sql.functions import broadcast
result = large_df.join(broadcast(small_df), "key")
```
Avoids the shuffle entirely — the small side is sent to every executor. Default threshold: 10 MB (`spark.sql.autoBroadcastJoinThreshold`). Good for dimension tables (< a few hundred MB).

#### 3. Salting — the manual fix when AQE + broadcast don't work

The idea: **break the hot key into N virtual sub-keys** so its rows distribute across multiple partitions. Then join against a replicated version of the other side.

```python
from pyspark.sql import functions as F

# Salt the hot side: each row gets a random salt bucket 0-9
N = 10
large_salted = large_df.withColumn("salt", (F.rand() * N).cast("int"))

# Replicate the small side 10 times, once per salt value
small_exploded = (
    small_df
    .withColumn("salt", F.explode(F.array([F.lit(i) for i in range(N)])))
)

# Join on the composite key (key, salt)
result = large_salted.join(small_exploded, ["key", "salt"])
```

What happened: the hot key's rows now distribute across 10 partitions instead of 1. The small side is duplicated 10x (cheap, since it's small). The shuffle balances out. Cost: 10x more rows on the small side — worth it.

Use salting only after AQE has failed you. It's the manual override, not the default.

#### 4. Bucketing (for recurring joins on the same key)
Pre-partition your storage by the join key:
```python
df.write.bucketBy(100, "user_id").saveAsTable("users_bucketed")
```
Subsequent joins on `user_id` avoid the shuffle entirely — the buckets line up. Worth it only if you join on the same key repeatedly.

---

## Interview one-liners (memorize these)

- **"How does a shuffle work?"** → Map tasks write output to local disk partitioned by hash of the key. Reduce tasks fetch their partition from every map node over the network. Disk + network + serialization = expensive.
- **"When does a shuffle happen?"** → Wide transformations: `groupBy`, `join`, `distinct`, `repartition`, `orderBy`, window functions with a different partition key. Each one creates a new stage.
- **"How do you avoid shuffles?"** → Broadcast joins for small dim tables, bucketing for recurring joins, partition your storage by the key you filter on, use `coalesce` instead of `repartition` when shrinking.
- **"How do you handle skew?"** → Enable AQE skew join first. If that's not enough, broadcast the small side. If both fail, salt the hot key manually.
- **"Why is `spark.sql.shuffle.partitions` important?"** → It's the number of partitions after any shuffle. Default 200 is almost always wrong. Tune so each post-shuffle task handles ~128–256 MB.
- **"`repartition` vs `coalesce`?"** → `repartition` triggers a shuffle to reach an exact count and can rebalance. `coalesce` merges existing partitions without a shuffle but destroys upstream parallelism if used too aggressively. `repartition` to increase, `coalesce` to decrease before write.

---

## Common pitfalls (the stuff that bites in production)

1. **Leaving `spark.sql.shuffle.partitions=200` with a 500 GB dataset.** Tasks are 2.5 GB each → OOM.
2. **`coalesce(1)` before a wide transformation.** You just serialized your whole pipeline into one task.
3. **Partitioning by `customer_id` on disk.** Creates millions of tiny files, kills read performance worse than not partitioning at all.
4. **UDFs after a shuffle.** UDFs are opaque to Catalyst, so the optimizer can't push them up or fuse them. Use built-in functions when you can.
5. **Not enabling AQE.** Spark 3+ ships with AQE *disabled by default* in some older distributions. Always check and enable it — it's the single easiest optimization you'll ever apply.
6. **Trusting `explain()` without also looking at Spark UI.** The plan shows what *should* happen. The UI shows what *did* happen — including skew, stragglers, and GC pressure.

---

## Further reading

- [Spark Performance Tuning (official)](https://spark.apache.org/docs/latest/sql-performance-tuning.html)
- [Mastering Spark Internals: Shuffle](https://books.japila.pl/apache-spark-internals/shuffle/) — Jacek Laskowski, free gitbook
- [Databricks: Adaptive Query Execution](https://www.databricks.com/blog/2020/05/29/adaptive-query-execution-speeding-up-spark-sql-at-runtime.html) — canonical AQE post
- [Databricks glossary: Data skew](https://www.databricks.com/glossary/data-skew)
- The salting pattern is practiced hands-on in [`../03_joins.ipynb`](../03_joins.ipynb) Boss Level.

---

*Last revised: 2026-04*
