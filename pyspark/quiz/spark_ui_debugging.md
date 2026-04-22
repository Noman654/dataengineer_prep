# 🔍 Spark UI & `.explain()` — Self-Check Quiz

**How to use this:** Read the question. Say your answer out loud like you're in an interview. Then click to expand.

**Paired with:** [`../theory/spark_ui_debugging.md`](../theory/spark_ui_debugging.md)

**Difficulty mix:** 🟢 basics → 🟡 intermediate → ⚡ senior

---

## 🟢 Basics

<details>
<summary><strong>Q1.</strong> What's the default URL for the Spark UI during a job?</summary>

<br>

`http://<driver-host>:4040` while the application is running.

After the app finishes, the UI disappears unless you have a **history server** running (`http://<host>:18080`). In Databricks, the UI is embedded in the job run page and persists automatically.

</details>

<details>
<summary><strong>Q2.</strong> Name the six Spark UI tabs and one thing each tells you.</summary>

<br>

| Tab | Use it for |
|---|---|
| **Jobs** | Which action is slow (each action = one job) |
| **Stages** | Where time is spent inside a job; skew / GC / spill |
| **Storage** | What's cached and its memory footprint |
| **SQL / DataFrame** | Per-query physical plan + runtime metrics |
| **Executors** | Per-executor memory, cores, GC time, logs |
| **Environment** | All Spark conf — verify `adaptive.enabled`, `shuffle.partitions`, etc. |

</details>

<details>
<summary><strong>Q3.</strong> What do the four plans in <code>df.explain(True)</code> represent?</summary>

<br>

1. **Parsed Logical Plan** — raw tree from your code, no checking yet.
2. **Analyzed Logical Plan** — column names resolved, types inferred.
3. **Optimized Logical Plan** — Catalyst rewrite rules applied.
4. **Physical Plan** — concrete operators that will execute. **This is the one that matters.**

Read physical plans **bottom-up** — execution starts at the leaves (Scans) and flows toward the root.

</details>

---

## 🟡 Intermediate

<details>
<summary><strong>Q4.</strong> In the Stages tab, what does it mean when Max task duration is 50× Median?</summary>

<br>

**Skew.** One or two tasks are doing far more work than the rest. Your total stage runtime = slowest task's runtime, so 199 tasks sit idle while one grinds.

**Next steps:**
1. Scroll to the tasks table below, sort by duration.
2. Click the slow task → check its **Shuffle Read Size** (confirms data-volume skew).
3. Check which executor ran it and whether that executor had other issues.
4. Address via AQE skew join / broadcast / salting.

</details>

<details>
<summary><strong>Q5.</strong> What does a non-zero "Shuffle Spill (Memory)" mean?</summary>

<br>

Execution memory ran out during a shuffle/sort/aggregate, so Spark wrote the in-memory buffer to local disk and kept going.

**Not fatal**, but signals under-provisioning:
- Each task is handling too much data → increase `spark.sql.shuffle.partitions`
- Or the executor is under-memoried → bump `spark.executor.memory`
- Or enable off-heap memory to give execution more room

"Shuffle Spill (disk)" is the compressed on-disk footprint — usually ~5× smaller than the memory spill. Both numbers matter.

</details>

<details>
<summary><strong>Q6.</strong> In a physical plan you see <code>Exchange hashpartitioning(store_id, 200)</code>. What is it?</summary>

<br>

**A shuffle.** The rows are being redistributed across 200 partitions by hash of `store_id`.

- Every `Exchange` = one shuffle = one new stage boundary.
- Count the Exchanges to count your job's shuffles.
- `hashpartitioning` = the standard shuffle (for joins, groupBy).
- `SinglePartition` = collapse everything to one partition (disaster sign — usually from `orderBy` without `partitionBy` or `collect` on large data).
- `rangepartitioning` = used by global `orderBy`.

</details>

<details>
<summary><strong>Q7.</strong> How do you verify predicate pushdown worked from the physical plan?</summary>

<br>

Look at the `Scan parquet` line. If the filter pushed down, you'll see:

```
Scan parquet [col1, col2, col3] PushedFilters: [EqualTo(year, 2024), IsNotNull(store_id)]
```

If `PushedFilters: []` is empty or missing, the filter didn't push down — the reader is loading every row and filtering in memory.

**Common reasons it fails:**
- You used a UDF in the filter.
- The file format doesn't support pushdown (CSV, JSON).
- You applied the filter after a shuffle that invalidated the source metadata.

</details>

---

## ⚡ Senior / interview judgment

<details>
<summary><strong>Q8.</strong> "Your job is slow, what do you look at first?" — walk through the diagnostic sequence.</summary>

<br>

1. **Spark UI → Jobs tab.** Which action is slow? (If the whole app is slow, proceed anyway.)
2. **Jobs → click slow job → Stages.** Which stage dominates the runtime?
3. **Stages → click slow stage → Summary Metrics.**
   - Max duration >> Median → **skew** → see [data_skew.md](../theory/data_skew.md)
   - GC Time > 10% → **memory pressure** → see [memory_management.md](../theory/memory_management.md)
   - Non-zero shuffle spill → **under-partitioned** — bump `shuffle.partitions`
   - Max input size much bigger than necessary → **no predicate pushdown** — check `.explain()`
4. **SQL tab → click query → inspect plan.** Is the join strategy what you expected? `BroadcastHashJoin` vs `SortMergeJoin`?
5. **Executors tab.** Any dead executors? Consistent memory? GC spikes?
6. **Environment tab.** Is AQE actually enabled? What's `shuffle.partitions`? What's `autoBroadcastJoinThreshold`?

Reciting this sequence is a senior-level interview answer by itself.

</details>

<details>
<summary><strong>Q9.</strong> You see <code>BroadcastHashJoin</code> in the plan but the job is still slow. What else could be wrong?</summary>

<br>

The join strategy is right — time to look elsewhere:

1. **The "small" side is actually too big to broadcast well.** Check `BroadcastExchange` row count. If it's 100 MB+ being broadcast to 200 executors = 20 GB of network + driver memory pressure. Reduce broadcast size or use SortMergeJoin intentionally.
2. **Downstream operations are the bottleneck.** The join is fast; the `groupBy` after it is slow. Inspect the *next* stage.
3. **The big side is huge AND unfiltered.** Broadcast doesn't help if you're still scanning 5 TB. Add filters before the join and verify they pushed down.
4. **Skew in the big side.** Broadcast eliminates join-shuffle skew, but if the big side has skewed upstream partitioning (e.g., read from bucketed storage with uneven bucket sizes), you'll still see skew in the join stage.
5. **Driver OOM from the broadcast.** Symptoms: job hangs at "Broadcasting..." then driver dies. Reduce broadcast size or bump `spark.driver.memory` + `spark.driver.maxResultSize`.

</details>

<details>
<summary><strong>Q10.</strong> A job looks fine in <code>.explain()</code> — sensible shuffles, BroadcastHashJoin, filters pushed. But in the UI, one stage takes 30 minutes while the rest finish in seconds. What's the mismatch and how do you investigate?</summary>

<br>

**The mismatch:** `.explain()` shows what Spark *planned*. The UI shows what *actually happened*. The plan can be perfect while runtime goes wrong due to **data characteristics the plan didn't anticipate**.

**Likely causes:**
1. **Runtime skew.** Plan looks balanced because Catalyst assumed uniform key distribution; reality isn't. → check Summary Metrics Max/Median.
2. **Straggler executor.** One executor is slow due to hardware issue, GC pressure, or being colocated with a noisy neighbor. → Executors tab, look for outlier GC time or task time per executor.
3. **AQE re-plan.** If AQE is on, the actual executed plan may differ from what `.explain()` showed before runtime. Check the SQL tab's final plan (post-AQE) rather than the `.explain()` output.
4. **Executor lost + task retry.** A silent task failure triggered a retry on another executor. Look at the tasks table for failed/retried tasks.
5. **Shuffle fetch failures.** Slow network between executors. Look for "FetchFailedException" in executor logs.

**Investigation workflow:**
- Stage Summary Metrics (duration, GC, shuffle).
- Executor tab for outlier executors.
- Event Timeline on the stage for task scheduling gaps.
- SQL tab's final plan to see what AQE actually decided.

</details>

---

## 🎯 How did you do?

- **9–10 correct** → you can debug real Spark jobs. This is senior-level territory.
- **6–8 correct** → solid grasp. Practice on your own jobs — open the UI for every run.
- **< 6 correct** → re-read [spark_ui_debugging.md](../theory/spark_ui_debugging.md), then walk through the [official Spark Web UI docs](https://spark.apache.org/docs/latest/web-ui.html) tab by tab.

**Related:** [quiz/catalyst_and_aqe.md](catalyst_and_aqe.md), [quiz/data_skew.md](data_skew.md), [quiz/memory_management.md](memory_management.md)
