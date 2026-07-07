# Catalyst Optimizer & Adaptive Query Execution (AQE) — Quick Review

**Audience:** senior DE interview prep. Assumes you can read a physical plan and have at least heard of predicate pushdown.

**Read time:** 10–15 minutes.

---

## TL;DR — the one-sentence mental model

> **Catalyst** is Spark's rule-based optimizer: it takes your DataFrame code, builds a logical plan, applies rewrite rules (predicate pushdown, column pruning, constant folding, join reorder), and produces a physical plan — all **before the job runs**. **AQE** then fixes the mistakes Catalyst made by re-optimizing **at runtime** once it sees real data statistics.

If you remember nothing else: **Catalyst is compile-time, AQE is run-time. AQE exists because compile-time estimates lie.**

---

## The four stages of a Spark query

Every DataFrame operation goes through this pipeline:

```
  Your code
     │
     ▼
┌─────────────────┐
│ 1. Unresolved   │  — raw parsed tree, column names not yet checked against schema
│    Logical Plan │
└─────────────────┘
     │
     ▼
┌─────────────────┐
│ 2. Analyzed     │  — column names resolved against catalog, types inferred
│    Logical Plan │
└─────────────────┘
     │
     ▼
┌─────────────────┐
│ 3. Optimized    │  — Catalyst rules applied: predicate pushdown, column pruning,
│    Logical Plan │    constant folding, filter reorder, join reorder (CBO), etc.
└─────────────────┘
     │
     ▼
┌─────────────────┐
│ 4. Physical Plan│  — concrete operators chosen: BroadcastHashJoin vs SortMergeJoin,
│                 │    HashAggregate vs SortAggregate, Exchange (shuffle) boundaries
└─────────────────┘
     │
     ▼
  RDD + Whole-Stage Codegen → executed
```

See all four with `df.explain(mode="extended")`. The physical plan is the one you actually care about in interviews.

---

## What Catalyst actually does — the rules that matter

Catalyst is a collection of **rewrite rules** applied to the tree. The ones to know:

### Predicate pushdown
Moves filters as close to the data source as possible. If you read Parquet and filter on a column, the filter gets pushed to the Parquet reader so it never loads matching-free rows.

```python
spark.read.parquet("/data").filter("year = 2024").select("tx_id")
# Catalyst pushes year=2024 into the Parquet scan
```

Works for: Parquet, ORC, JDBC (limited), Delta — and CSV/JSON too since Spark 3.0 (`spark.sql.csv.filterPushdown.enabled` / `spark.sql.json.filterPushdown.enabled`, both default `true`): the reader skips non-matching rows without materializing them, and the filter shows up in `PushedFilters`. What CSV/JSON can't do is min/max **row-group skipping** — they're row-oriented with no per-column statistics, so the reader still has to open every row group. That row-group skip is Parquet/ORC/Delta-only.

### Column pruning (projection pushdown)
Reads only the columns you reference. A Parquet file with 50 columns + your `select("a", "b")` = only 2 columns read from disk. This is why Parquet dominates in DE — columnar storage + column pruning = massive I/O savings.

### Constant folding
`filter("year = 2020 + 4")` → Catalyst simplifies to `filter("year = 2024")` before execution.

### Filter reorder
`filter(expensive_udf()).filter(cheap_condition)` → Catalyst moves cheap filters first to reduce the input to the expensive one.

### Join reorder (CBO — Cost-Based Optimizer)
Enabled with `spark.sql.cbo.enabled=true` + up-to-date table stats (`ANALYZE TABLE ... COMPUTE STATISTICS`). Reorders multi-way joins to minimize intermediate result size (e.g., filter-heavy joins first). **Off by default** because it needs stats.

### Join strategy selection
Picks `BroadcastHashJoin` if one side's estimated size < `autoBroadcastJoinThreshold` (default 10 MB). Falls back to `SortMergeJoin` otherwise. **This is where Catalyst most often gets it wrong** — estimates are based on stats that may not exist or be stale.

---

## Reading a physical plan

```python
df.explain()  # short form
df.explain(True)  # full (parsed + analyzed + optimized + physical)
df.explain(mode="formatted")  # Spark 3+ — cleaner for long plans
```

Key operators to recognize:

| Operator | Meaning |
|---|---|
| `Scan parquet` | Read from Parquet (look for `PushedFilters` and `ReadSchema` to see predicate/projection pushdown) |
| `Exchange hashpartitioning(...)` | **A shuffle**. Each Exchange = one shuffle boundary = one stage break |
| `BroadcastHashJoin` | Small side broadcast; no shuffle on big side. What you want. |
| `SortMergeJoin` | Both sides shuffled + sorted. Fallback when broadcast isn't possible. |
| `ShuffledHashJoin` | Both sides shuffled, hash-joined. Rare. |
| `BroadcastExchange` | The step that broadcasts the small side (only shows with BroadcastHashJoin) |
| `HashAggregate` | Hash-based aggregation (preferred). |
| `SortAggregate` | Sort-based aggregation (fallback if the key has non-hashable/complex types). |
| `WholeStageCodegen (N)` | Multiple operators fused into a single generated Java function for speed. |

**Interview tip:** reading plans is senior-level muscle memory. Practice it on your own jobs until you can eyeball a plan and spot the shuffles in 5 seconds.

---

## Why AQE exists — the problem with compile-time plans

Catalyst picks a physical plan **before seeing any data**. It uses:
- File sizes on disk
- Stats from `ANALYZE TABLE` (if you ran it)
- Heuristics

**All three lie in real life:**
- Compressed Parquet files are ~5–10× smaller than in-memory size → broadcast threshold estimates off by an order of magnitude.
- Filtering produces unknown amounts of output — Catalyst guesses.
- Skew only manifests at run-time with real data distributions.

**Result without AQE:** jobs pick wrong join strategies, end up with wildly imbalanced partitions, and hit slow tasks that Catalyst couldn't predict.

---

## AQE — the runtime fixer (Spark 3+)

AQE re-optimizes the plan **at each shuffle boundary**, using actual shuffle metrics. Three main features:

### 1. Dynamic coalesce of shuffle partitions
Reduces `spark.sql.shuffle.partitions` at runtime based on actual data size.

- Post-shuffle, AQE sees each partition's real byte size
- Merges small partitions into larger ones targeting `advisoryPartitionSizeInBytes` (default 64 MB)
- **Effect:** you can leave `spark.sql.shuffle.partitions = 200` as default; AQE coalesces down to whatever's appropriate

Config:
```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.advisoryPartitionSizeInBytes", "64MB")
```

### 2. Dynamic switch to broadcast join
If one side of a join ends up smaller than expected after a filter, AQE upgrades a `SortMergeJoin` to a `BroadcastHashJoin` at runtime.

Example: `big.join(huge.filter("date = today"))` — Catalyst thinks `huge.filter(...)` is big, picks SortMergeJoin. AQE sees the filter produced only 5 MB, switches to broadcast.

### 3. Skew join handling
Detects skewed partitions at runtime and splits them into smaller sub-partitions (replicating the other side as needed). This is the single best "skew handling with zero code change" feature in Spark 3.

Config:
```python
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
# How much bigger than the median a partition has to be to count as skewed:
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionFactor", "5")
# Minimum absolute size to be considered skewed:
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "256MB")
```

**Default enabled in Spark 3.2+**, but always verify — older deployments still ship with it off.

---

## When AQE doesn't help

- **Single-stage jobs.** AQE works at shuffle boundaries; a simple `read → filter → write` has no shuffles to re-optimize.
- **Very small datasets.** The overhead of re-planning > the benefit.
- **Pathological skew that requires salting.** AQE splits a skewed partition *per join*, but if the salt-level skew is extreme (one key > half the data), you still need manual salting.
- **Broadcasts that should have happened from the start.** AQE can upgrade SortMergeJoin → broadcast, but the first stage has already been paid. Use an explicit `broadcast()` hint when you know.

---

## Interview one-liners (memorize these)

- **"What is Catalyst?"** → Spark's rule-based query optimizer. Takes your DataFrame/SQL, builds a logical plan, applies rewrite rules (predicate pushdown, column pruning, constant folding, join reorder, join strategy selection), and produces a physical plan. All compile-time.
- **"What does predicate pushdown do?"** → Moves filters as close to the data source as possible. Parquet/ORC/Delta readers can also skip entire row groups via min/max stats; CSV/JSON get filter pushdown too (since Spark 3.0) but can't skip row groups since they're not columnar.
- **"What's column pruning?"** → Only reads the columns referenced in your query. Key reason Parquet beats CSV at scale.
- **"Why does AQE exist?"** → Catalyst picks a plan before seeing data, using stats that are often missing or wrong. AQE re-optimizes at runtime once real shuffle metrics are known.
- **"Three things AQE does?"** → (1) Coalesces small post-shuffle partitions. (2) Upgrades SortMergeJoin to BroadcastHashJoin when runtime size allows. (3) Detects and splits skewed join partitions.
- **"How do you enable AQE?"** → `spark.sql.adaptive.enabled=true` (plus `coalescePartitions.enabled` and `skewJoin.enabled`). Default on in 3.2+.
- **"How do you read a physical plan?"** → Bottom-up. Each `Exchange` = one shuffle = one stage boundary. `BroadcastHashJoin` is the good join, `SortMergeJoin` is the fallback. `Scan parquet` with `PushedFilters` confirms predicate pushdown worked.

---

## Common pitfalls (the stuff that bites in production)

1. **Leaving AQE disabled on Spark 3.0 / 3.1.** Default was off; 3.2+ flipped it. Always check `spark.conf.get("spark.sql.adaptive.enabled")`.
2. **Broadcast hints on joins where the "small" side isn't small.** Blows the driver when Spark tries to collect it. Measure first.
3. **Relying on CBO without running `ANALYZE TABLE`.** CBO needs stats. Without them, it's no better than the rule-based optimizer.
4. **Assuming Catalyst can optimize UDFs.** It can't — UDFs are opaque black boxes. Rewrite as built-in functions when possible.
5. **Writing a `filter` after a `cache`.** Catalyst can't push the filter through the cache. Cache the filtered result instead.
6. **Not looking at `PushedFilters` in the plan.** If your filter didn't push down, your job reads 10× more than it needs to. Verify.
7. **`spark.sql.shuffle.partitions = 200` with AQE enabled and a 500 GB shuffle.** Even with coalesce, you want a sensible starting point. Set it to something like `max(200, total_shuffle_bytes / 128MB)`.

---

## Further reading

**Official:**
- [Spark SQL Performance Tuning (including AQE)](https://spark.apache.org/docs/latest/sql-performance-tuning.html)
- [Catalyst Optimizer (Databricks docs)](https://www.databricks.com/glossary/catalyst-optimizer)

**Community classics (shared widely on LinkedIn):**
- [**Deep Dive into Spark SQL's Catalyst Optimizer** — Databricks (2015)](https://www.databricks.com/blog/2015/04/13/deep-dive-into-spark-sqls-catalyst-optimizer.html) — the original and canonical post by Michael Armbrust & Yin Huai. Still the best explanation.
- [**Adaptive Query Execution: Speeding Up Spark SQL at Runtime** — Databricks (2020)](https://www.databricks.com/blog/2020/05/29/adaptive-query-execution-speeding-up-spark-sql-at-runtime.html) — the AQE launch post, with benchmark numbers.
- [**Catalyst Optimizer: The Power of Spark SQL** by Shikha Bhatia (Medium)](https://medium.com/@Shkha_24/catalyst-optimizer-the-power-of-spark-sql-cad8af46097f) — concise walkthrough with examples.
- [**Mastering Spark SQL** by Jacek Laskowski (gitbook)](https://jaceklaskowski.gitbooks.io/mastering-spark-sql/) — for when you want to go deeper into specific rules.

**Author's own:**
- [**Spark's Secret Ingredient: The DAG**](https://www.linkedin.com/pulse/sparks-secret-ingredient-dag-directed-acyclic-graph-revealed-nauman-wzyec/) — by Mohd Nauman *(author of this repo)*.

Related in this repo: [`shuffle_and_partitioning.md`](shuffle_and_partitioning.md), [`data_skew.md`](data_skew.md), [`spark_ui_debugging.md`](spark_ui_debugging.md).

---

*Last revised: 2026-04*
