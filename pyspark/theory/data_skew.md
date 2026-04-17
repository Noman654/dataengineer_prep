# Data Skew — The Playbook

**Audience:** senior DE interview prep. Skew is **the** senior topic — more than any other, it separates people who've *run* Spark in production from people who've only *written* Spark.

**Read time:** 10–15 minutes.

**See also:** [`shuffle_and_partitioning.md`](shuffle_and_partitioning.md) covers skew from the shuffle-internals angle. This doc is the focused **fix playbook**.

---

## TL;DR — the one-sentence mental model

> Skew happens when one key has dramatically more rows than others. Because shuffles distribute by hash of the key, **a skewed key floods one reducer while the others sit idle**. Your job's total time = the slowest task's time. One hot key can turn a 5-minute job into an hour.

**The fix hierarchy:** AQE skew join → broadcast → salting → two-phase aggregation → redesign the key.

---

## What skew looks like in practice

Normal healthy job:
```
Task durations (ms):  120, 135, 128, 140, 125, 130, 122, 138  ← min~max, all parallel
```

Skewed job:
```
Task durations (ms):  120, 135, 128,  140, 125, 130, 122, 43000  ← one task takes 300×
                                                          ^^^^^
                                                    that one task = your total runtime
```

Your cluster has 200 cores but one task is doing most of the work. The other 199 cores sit idle waiting.

---

## Where skew shows up

### 1. Join skew
`big.join(medium, "customer_id")`. If one customer has 40% of the rows, one reducer gets 40% of the shuffle.

### 2. GroupBy skew
`df.groupBy("country").count()`. If 60% of your users are in one country, one task handles 60% of the data.

### 3. Window skew
`Window.partitionBy("user_id")` is fine (high cardinality). `Window.partitionBy("status")` where 90% of rows are `"active"` is skewed.

### 4. Writes partitioned by a skewed column
`df.write.partitionBy("category").parquet(...)` — one category directory gets GB, others get KB. Small-file explosion on the light keys, huge file on the heavy one.

---

## Detection — before and after

### Before running — check the key distribution
```python
df.groupBy("join_key").count().orderBy(F.col("count").desc()).show(20)
```
If the top key has **10× more rows than the median**, you have skew. If it has 100×, you have severe skew.

Compute the imbalance ratio:
```python
from pyspark.sql import functions as F
counts = df.groupBy("join_key").count()
stats = counts.agg(
    F.max("count").alias("max"),
    F.expr("percentile_approx(count, 0.5)").alias("median"),
).first()
print(f"Max/Median ratio: {stats.max / stats.median:.1f}x")
```

### During/after running — check the Spark UI
1. Go to **Stages** → click the slow stage.
2. Look at **Summary Metrics** at the top.
3. Compare **Min / 25th / Median / 75th / Max** for:
   - **Duration** — max >> median means skewed runtimes.
   - **Shuffle Read Size** — max >> median means skewed data volumes.
4. In the **tasks table** below, sort by duration. One or two outlier tasks that took 50× the median = skew confirmed.

**The tell in the UI:** your stage sits at "199/200 tasks succeeded" for 40 minutes while that one task grinds.

---

## The fix hierarchy

### Level 1: AQE skew join (Spark 3+)

**Always try this first.** Zero code change.

```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionFactor", "5")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "256MB")
```

**What it does:** detects partitions > 5× median AND > 256 MB, splits them into smaller sub-partitions, replicates the matching rows from the other side as needed.

**When it works:** most skew you'll see in production. AQE solves it without you writing a line of code.

**When it doesn't:** extreme skew (one key > 50% of data), or pure `groupBy` skew (AQE's skew handling is join-focused; less effective on bare aggregations).

### Level 2: Broadcast the small side

If one side is small (< a few hundred MB), broadcast it. **No shuffle = no skew.**

```python
from pyspark.sql.functions import broadcast
big_skewed.join(broadcast(small_lookup), "key")
```

**When it works:** dimension tables, lookup tables. Works regardless of how skewed the big side is, because the big side never shuffles.

**When it doesn't:** both sides are large, or the "small" side is actually a few GB.

### Level 3: Manual salting

The nuclear option. Use only when AQE + broadcast aren't enough.

**Pattern for a single hot key:**
```python
N = 20  # salt buckets — tune based on skew severity

# 1. Salt the big side — every row gets a random salt 0..N-1
big_salted = big.withColumn("salt", (F.rand() * N).cast("int"))

# 2. Explode the small side — each row replicated N times with every salt value
small_exploded = small.withColumn(
    "salt",
    F.explode(F.array([F.lit(i) for i in range(N)]))
)

# 3. Join on the composite (key, salt)
result = big_salted.join(small_exploded, ["key", "salt"]).drop("salt")
```

**What happened:** the hot key's rows distribute across 20 partitions instead of 1. Small side is 20× bigger (still cheap if it started small). The shuffle balances.

**Tuning `N`:** start with `N = ceil(hot_key_count / median_key_count)`. Too small → skew remains. Too large → small-side replication overhead dominates.

**Asymmetric salting (more surgical):** only salt the known hot keys, leave the rest unchanged. Splits each hot key but doesn't inflate the small side by N×.

```python
# Tag only the hot key; other keys get salt=0 (no replication on small side)
hot_keys = ["42"]  # known hot keys
big_tagged = big.withColumn(
    "salt",
    F.when(F.col("key").isin(hot_keys), (F.rand() * N).cast("int")).otherwise(F.lit(0))
)
# Small side: replicate only rows matching hot keys; others stay salt=0
small_tagged = small.withColumn(
    "salt_array",
    F.when(F.col("key").isin(hot_keys), F.array([F.lit(i) for i in range(N)]))
     .otherwise(F.array(F.lit(0)))
).withColumn("salt", F.explode("salt_array")).drop("salt_array")
```

### Level 4: Two-phase aggregation (for groupBy skew)

AQE doesn't solve groupBy skew well. For `df.groupBy("country").agg(...)` where one country dominates, use a two-phase pattern:

```python
# Phase 1: pre-aggregate with a random prefix to distribute load
df_pre = (
    df
    .withColumn("salt", (F.rand() * 100).cast("int"))
    .groupBy("country", "salt")
    .agg(F.sum("amount").alias("partial_sum"))
)

# Phase 2: final aggregate — the inputs are now balanced
result = (
    df_pre
    .groupBy("country")
    .agg(F.sum("partial_sum").alias("total"))
)
```

**Why this works:** phase 1 uses `country + salt` as the key, spreading the hot country across 100 partitions. Phase 2 rolls up the small partial sums into per-country totals. The expensive shuffle happens on balanced data.

### Level 5: Redesign the key

The architectural fix. Sometimes the key is the problem.

- **Composite keys:** if `store_id` is skewed but `(store_id, hour)` is even, aggregate/join on the composite.
- **Bucketing:** pre-partition the data by hash of the key so recurring joins avoid the shuffle entirely: `df.write.bucketBy(100, "user_id").saveAsTable(...)`.
- **Range-partition unknown data:** if you can't pick a good key, range-partition on a natural dimension (date).

---

## Skew-aware design patterns

### Write carefully when partitioning by a skewed column
```python
# BAD — skewed column → some directories tiny, one directory huge
df.write.partitionBy("country").parquet(path)

# BETTER — add a bucketed sub-partition to balance
df.withColumn("bucket", F.hash("user_id") % 10)\
  .write.partitionBy("country", "bucket").parquet(path)
```

### Detect skew proactively in prod pipelines
Add a skew check as a data quality gate:
```python
max_ratio = (
    df.groupBy(key).count()
    .agg((F.max("count") / F.avg("count")).alias("ratio"))
    .first().ratio
)
if max_ratio > 20:
    raise ValueError(f"Skew detected: {max_ratio:.1f}x imbalance on {key}")
```

### Watch for skew drift
Joins that were fine last quarter may become skewed as data grows. A key that was 5% of rows can become 40% as a feature ships. Monitor the key distribution over time.

---

## Interview one-liners (memorize these)

- **"How do you detect skew?"** → Spark UI → Stages → Summary Metrics. If max task duration > 10× median, you have skew. Or programmatically: `df.groupBy(key).count().orderBy(desc).show()`.
- **"How do you handle a skewed join?"** → (1) Enable AQE skew join first, zero code. (2) If that's not enough, broadcast the small side. (3) If both fail, salt the hot key manually.
- **"Walk me through salting."** → Add a random salt column (0 to N-1) to the big/skewed side. Replicate the other side N times, once per salt value. Join on the composite key (original_key, salt). The hot key now distributes across N partitions.
- **"Why does AQE help with skew?"** → It re-plans at runtime once it sees actual shuffle partition sizes. Detects partitions much larger than median and splits them, replicating the other side as needed — all without code changes.
- **"When does AQE not help?"** → Extreme skew (one key > half the data), pure groupBy skew (AQE's skew handling is join-focused), and very small datasets where the re-planning overhead dominates.
- **"What's a two-phase aggregation?"** → For groupBy skew: add a random salt, aggregate by `(key + salt)`, then roll up by `key`. The expensive shuffle happens on balanced data.

---

## Common pitfalls

1. **Jumping to salting before trying AQE.** AQE solves 80% of skew cases with zero code. Always try it first.
2. **Picking N too small in salting.** If you salt with N=4 but the hot key is 40% of the data, you've just created 4 medium-hot partitions instead of 1 hot one. Size N to the skew ratio.
3. **Salting every key instead of just the hot ones.** Full salting inflates the small side by N× unnecessarily. Asymmetric salting is usually better in prod.
4. **Assuming AQE is on.** In Spark 3.0/3.1 it was off by default. Always verify: `spark.conf.get("spark.sql.adaptive.enabled")`.
5. **Skew hidden by caching.** Cache hides skew symptoms (the shuffle happens once, then reads are fast). Inspect the first computation, not the cached reads.
6. **Using `count()` alone to detect skew.** Row count ≈ data size for uniform rows, but a skewed key might also have much bigger rows (nested structs, long strings). Check both row count AND shuffle read size.
7. **Partitioning writes by a skewed column.** Creates one massive file per skewed partition and millions of tiny files per light one. Add a bucket sub-partition.

---

## Further reading

**Official:**
- [AQE skew join docs (Spark)](https://spark.apache.org/docs/latest/sql-performance-tuning.html#optimizing-skew-join)

**Community classics:**
- [**Databricks glossary: Data skew**](https://www.databricks.com/glossary/data-skew) — concise intro.
- [**Why Your Spark Apps Are Slow or Failing, Part 2: Data Skew and GC** — Rishitesh Mishra](https://dzone.com/articles/why-your-spark-apps-are-slow-or-failing-part-ii-da) — the troubleshooting framework everyone cites.
- [**Adaptive Query Execution: Speeding Up Spark SQL at Runtime** — Databricks](https://www.databricks.com/blog/2020/05/29/adaptive-query-execution-speeding-up-spark-sql-at-runtime.html) — AQE launch with skew benchmarks.
- [**Apache Spark Salting Technique** by Syeed Hasan](https://medium.com/@syeedhasan/salting-in-spark-b7c2236f4ae6) — code walkthrough of asymmetric salting.
- [**Spark Skew: How to Handle Data Skew in Spark** — Daniel Tomes (Databricks)](https://www.youtube.com/watch?v=6zg7NTw-kTQ) — the canonical talk. Watch it once.

**Hands-on in this repo:** [`../joins.ipynb`](../joins.ipynb) Boss Level — manual salting walk-through on Zephyr's skewed store 42.

Related: [`shuffle_and_partitioning.md`](shuffle_and_partitioning.md), [`catalyst_and_aqe.md`](catalyst_and_aqe.md), [`memory_management.md`](memory_management.md).

---

*Last revised: 2026-04*
