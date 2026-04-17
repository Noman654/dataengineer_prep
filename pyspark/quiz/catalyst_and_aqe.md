# 🎯 Catalyst & AQE — Self-Check Quiz

**How to use this:** Read the question. Say your answer out loud like you're in an interview. Then click to expand.

**Paired with:** [`../theory/catalyst_and_aqe.md`](../theory/catalyst_and_aqe.md)

**Difficulty mix:** 🟢 basics → 🟡 intermediate → ⚡ senior

---

## 🟢 Basics

<details>
<summary><strong>Q1.</strong> What is Catalyst?</summary>

<br>

Spark's rule-based query optimizer. It takes your DataFrame or SQL code and compiles it through four stages:

1. **Parsed logical plan** — raw tree from your code.
2. **Analyzed logical plan** — column names resolved, types inferred.
3. **Optimized logical plan** — rewrite rules applied (predicate pushdown, column pruning, constant folding, join reorder).
4. **Physical plan** — concrete operators chosen (BroadcastHashJoin vs SortMergeJoin, HashAggregate, etc.).

All of this happens **before the job runs** — compile-time.

</details>

<details>
<summary><strong>Q2.</strong> What is predicate pushdown, and what file format makes it powerful?</summary>

<br>

**Predicate pushdown** moves filters as close to the data source as possible. If you filter on a column, the filter is pushed **into the file reader** so matching-free rows are never loaded into memory.

**Parquet** (and ORC, Delta) benefit most — they store row-group-level statistics (min/max per column), so entire row groups can be skipped based on the predicate.

**CSV / JSON don't support it** — those are row-oriented, unstructured; the reader has to parse every row.

</details>

<details>
<summary><strong>Q3.</strong> What is column pruning and why does it matter?</summary>

<br>

Spark reads **only the columns you reference**, not the whole file. If a Parquet file has 50 columns and your query does `select("a", "b")`, only 2 columns are read from disk.

**Why it matters:** columnar formats (Parquet/ORC) store each column separately on disk. Column pruning can reduce I/O by 10–50× on wide tables. It's the single biggest reason Parquet dominates CSV in data engineering.

</details>

---

## 🟡 Intermediate

<details>
<summary><strong>Q4.</strong> Why does AQE exist?</summary>

<br>

Catalyst picks a physical plan **before seeing any data**. It uses file sizes on disk, table stats (if available), and heuristics — **all of which lie in real life**:

- Compressed Parquet is 5–10× smaller than in-memory size → broadcast threshold often misjudged.
- Filter selectivity is unknowable until runtime.
- Skew only manifests with real data distributions.

**AQE re-optimizes at runtime** — at each shuffle boundary, it uses actual shuffle metrics to fix the mistakes Catalyst made.

</details>

<details>
<summary><strong>Q5.</strong> Name the three main things AQE does.</summary>

<br>

1. **Dynamic coalesce of shuffle partitions.** Post-shuffle, merges small partitions into larger ones targeting ~64 MB each. Lets you leave `spark.sql.shuffle.partitions` at the default without penalty.

2. **Dynamic switch to broadcast join.** If one side of a join ends up smaller than the broadcast threshold after filtering, AQE upgrades `SortMergeJoin` → `BroadcastHashJoin` at runtime.

3. **Skew join handling.** Detects partitions much larger than median (default 5× threshold) and splits them into sub-partitions, replicating the other side as needed.

</details>

<details>
<summary><strong>Q6.</strong> Your <code>.explain()</code> shows the filter <em>above</em> the <code>Scan parquet</code> line, not pushed inside it. What went wrong?</summary>

<br>

Predicate pushdown failed. Common causes:

- **UDF in the filter** — Catalyst can't push a UDF into a Parquet reader (black box).
- **Filter on a computed/non-source column** — e.g., `.filter(F.col("price") * 1.2 > 100)` may not push.
- **Data source doesn't support pushdown** — CSV, JSON, some JDBC drivers.
- **Filter applied after an operation that breaks pushdown** — e.g., after a shuffle.

**Fix:** rewrite the filter in pure Catalyst expressions (`F.col(...)` and built-in functions only), move it before any shuffle, and verify the plan shows `PushedFilters: [...]`.

</details>

<details>
<summary><strong>Q7.</strong> How do you enable AQE and what's the <em>minimum</em> set of configs?</summary>

<br>

```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
```

**Default:** on in Spark 3.2+, off in 3.0/3.1. Always verify.

**Confirm it's active:** `.explain()` shows `AdaptiveSparkPlan` at the top of the physical plan.

</details>

---

## ⚡ Senior / interview judgment

<details>
<summary><strong>Q8.</strong> When does AQE <em>not</em> help?</summary>

<br>

- **Single-stage jobs.** AQE works at shuffle boundaries; `read → filter → write` has none.
- **Very small datasets.** Re-planning overhead > the benefit.
- **Extreme skew** where one key is >50% of the data. AQE can split a skewed partition, but splitting the one huge key doesn't parallelize well. Manual salting beats AQE here.
- **Pure groupBy skew.** AQE's skew handling is join-focused. For aggregation skew, use two-phase aggregation with a random prefix.
- **Broadcasts that should have happened from the start.** AQE can upgrade to broadcast mid-plan, but the first stage has already been paid. Use explicit `broadcast()` when you know.

</details>

<details>
<summary><strong>Q9.</strong> What's the difference between CBO (Cost-Based Optimizer) and AQE?</summary>

<br>

**CBO** = Cost-Based Optimizer. A **compile-time** feature that reorders multi-way joins and picks strategies based on table statistics you've pre-computed with `ANALYZE TABLE ... COMPUTE STATISTICS`. Off by default because it needs stats.

**AQE** = Adaptive Query Execution. A **runtime** feature that re-plans based on actual shuffle metrics as the job runs. No pre-computation needed.

**Key distinction:** CBO relies on pre-computed stats (which get stale). AQE sees reality at runtime. AQE is almost always the right tool; CBO helps only when you have fresh stats and want good plans for the *first* stage too.

</details>

<details>
<summary><strong>Q10.</strong> Walk through reading this physical plan snippet and diagnose what it does:

```
== Physical Plan ==
AdaptiveSparkPlan isFinalPlan=true
+- SortMergeJoin [store_id], Inner
   :- Sort [store_id]
   :  +- Exchange hashpartitioning(store_id, 200)
   :     +- Project [store_id, tx_id, amount]
   :        +- Filter (isnotnull(store_id))
   :           +- Scan parquet [store_id, tx_id, amount] PushedFilters: [IsNotNull(store_id)]
   +- Sort [store_id]
      +- Exchange hashpartitioning(store_id, 200)
         +- Scan parquet [store_id, points]
```
</summary>

<br>

**Reading bottom-up (as plans execute):**

1. Two Parquet scans — one reading `(store_id, tx_id, amount)`, one reading `(store_id, points)`. Column pruning worked (only requested columns shown).
2. The first scan has `PushedFilters: [IsNotNull(store_id)]` — the null filter pushed down. Good.
3. Both sides `Exchange hashpartitioning(store_id, 200)` — both get **shuffled** into 200 partitions by `store_id`. **Two shuffles = two stage boundaries.**
4. Both sides `Sort [store_id]` — sorted locally after the shuffle.
5. `SortMergeJoin [store_id]` — final join via merge.
6. `AdaptiveSparkPlan isFinalPlan=true` — AQE was active; this is the plan after all runtime adjustments.

**What's (mildly) wrong / could improve:**
- Two full shuffles of both sides. If one side is small, this should be a `BroadcastHashJoin` (one shuffle avoided). Check sizes and add a `broadcast()` hint.
- 200 partitions is the default — may be too few for a huge join or too many for a small one.

**Interview answer:** "This is a SortMergeJoin with both sides shuffled on `store_id`. Column pruning and null-filter pushdown worked. If either side is under ~100 MB, I'd replace this with a broadcast hint to eliminate one shuffle."

</details>

---

## 🎯 How did you do?

- **9–10 correct** → you can read plans and reason about optimization. Drill [`spark_ui_debugging.md`](../theory/spark_ui_debugging.md) next.
- **6–8 correct** → solid conceptual grasp. Practice reading more `.explain()` outputs.
- **< 6 correct** → re-read [catalyst_and_aqe.md](../theory/catalyst_and_aqe.md) and the [Databricks Catalyst deep dive](https://www.databricks.com/blog/2015/04/13/deep-dive-into-spark-sqls-catalyst-optimizer.html).

**Related:** [quiz/data_skew.md](data_skew.md), [quiz/spark_ui_debugging.md](spark_ui_debugging.md)
