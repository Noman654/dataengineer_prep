# Spark UI & `.explain()` — The Debugging Playbook

**Audience:** senior DE interview prep, and anyone who's ever been asked *"your job is slow — what do you look at first?"* in an interview.

**Read time:** 15 minutes (with screenshots practice, longer).

---

## TL;DR — the one-sentence mental model

> Reading the Spark UI and physical plans is the single biggest differentiator between people who *write* Spark and people who *debug* Spark. **The UI shows what actually happened; `.explain()` shows what Spark decided to do before it ran.** Senior DEs always check both.

If you remember nothing else: **`.explain()` + Spark UI are to Spark what `EXPLAIN ANALYZE` is to Postgres. Don't fly blind.**

---

## The Spark UI at a glance

When your job runs, Spark exposes a web UI at `http://<driver>:4040` (Databricks has its own wrapper). The tabs that matter:

| Tab | What it tells you | When to look |
|---|---|---|
| **Jobs** | High-level list of Spark jobs (each action = one job) | "Which action is slow?" |
| **Stages** | Stages within a job; each shuffle boundary = new stage | "Where in my job is the time going?" |
| **Storage** | What's cached and how much memory it's using | "Did my `.cache()` actually take effect?" |
| **SQL / DataFrame** | Per-query query plan + runtime metrics | "Was this join a broadcast or a shuffle?" |
| **Executors** | Memory / cores / GC time / logs per executor | "Am I getting killed by GC? Is an executor stuck?" |
| **Environment** | All Spark conf + system properties | "Is AQE actually enabled? What's `shuffle.partitions`?" |

---

## The Stages tab — your #1 debugging tool

Click a slow stage and scroll to **Summary Metrics**. This table shows the distribution of the stage's tasks across 5 percentiles:

| Metric | Min | 25% | Median | 75% | Max | What to check |
|---|---|---|---|---|---|---|
| Duration | | | | | | Max >> Median → **skew** |
| GC Time | | | | | | >10% of duration → **memory pressure** |
| Input Size | | | | | | Max >> Median → uneven partitioning at the source |
| Shuffle Read Size | | | | | | Max >> Median → **shuffle skew** |
| Shuffle Write Size | | | | | | Max >> Median → skewed upstream output |
| Shuffle Spill (memory) | | | | | | Non-zero → execution memory exhausted |
| Shuffle Spill (disk) | | | | | | Non-zero → same, compressed to disk |

**The core diagnostic question:** is Max ≈ Median (healthy) or Max >> Median (skewed)?

Below the summary is the **tasks table** — sort by duration descending. Outlier tasks jump out immediately.

### Patterns you'll see in the wild

| UI symptom | Likely cause |
|---|---|
| One task 50× slower than median, same input size | **Skew** — hot key dumping on that partition |
| All tasks similar duration but GC Time > 20% | **Memory pressure** — bump executor memory or reduce per-task data |
| Non-zero Shuffle Spill on most tasks | **Under-partitioned** — increase `spark.sql.shuffle.partitions` |
| First stage instant, second stage takes forever | Heavy transformation without enough parallelism; check the shuffle boundary |
| "199/200 succeeded" stuck at 95% for minutes | **Straggler** — either skew (likely) or a flaky executor |
| Tasks succeeded then the stage retries | Executor lost (OOM, network, preemption). Check **Executors tab → dead executors** |

---

## The SQL / DataFrame tab — see the actual physical plan

Every DataFrame action creates an entry here. Click it to see:

- **Query plan visualization** — boxes for each operator, edges showing data flow
- **Per-operator metrics** — rows in/out, data size, time spent
- **Shuffle boundaries** — where the plan breaks into stages

**What to look for:**

- `Scan parquet` should show `PushedFilters: [...]` if predicate pushdown worked. Empty `PushedFilters` = your filter didn't push down (e.g., you used a UDF).
- `ReadSchema` should list only the columns you actually use. Extra columns = column pruning failed.
- `Exchange hashpartitioning(...)` with a huge output = you're shuffling a lot of data. Ask: can one side be broadcast?
- `BroadcastHashJoin` = good. `SortMergeJoin` = fine for two big tables, bad if one side should have been broadcast.
- Per-operator row counts tell you where filters work (input >> output) vs. fail (input ≈ output).

---

## `.explain()` — before running the query

```python
df.explain()                    # just the physical plan (short)
df.explain(True)                # parsed + analyzed + optimized + physical
df.explain(mode="formatted")    # Spark 3+ — cleanest; good for long plans
df.explain(mode="cost")         # show Catalyst's size/row estimates
```

### The four plans (read from bottom up)

```
== Parsed Logical Plan ==      ← what you wrote, before any checking
== Analyzed Logical Plan ==    ← after column/type resolution
== Optimized Logical Plan ==   ← after Catalyst rule rewrites (filter pushdown, etc.)
== Physical Plan ==            ← what Spark will actually execute ← THIS is what matters
```

### Key physical operators (the ones interviewers ask about)

| Operator | What it means | Implications |
|---|---|---|
| `Scan parquet` | Read from Parquet files | Check `PushedFilters` + `ReadSchema` |
| `Project [col1, col2, ...]` | Column selection | Confirms column pruning |
| `Filter (...)` | Predicate applied | Can still appear even when the predicate pushed down successfully — Parquet's pushed filters aren't guaranteed exact, so Spark keeps a Filter for correctness. Don't use this as your pushdown signal; check `PushedFilters` instead |
| `Exchange hashpartitioning(k, 200)` | **Shuffle** — data redistributed by hash(k) into 200 partitions | New stage boundary; expensive |
| `Exchange SinglePartition` | Global collapse to one partition | Disaster sign — check for a **window** `orderBy` with no `partitionBy` (a global DataFrame `orderBy` uses `rangepartitioning`, not this), or `count().collect()` |
| `BroadcastExchange` | Small side being broadcast | Paired with BroadcastHashJoin |
| `BroadcastHashJoin [k]` | Broadcast join on key k | **What you want** |
| `SortMergeJoin [k]` | Both sides shuffled and sorted, then merged | Fine for big ↔ big joins; costly if one side could have been broadcast |
| `HashAggregate` | Hash-based aggregate | Preferred |
| `SortAggregate` | Sort-based aggregate | Fallback for complex keys |
| `Window [...]` | Window function over a partition | Preceded by Exchange on the partition key |
| `WholeStageCodegen (N)` | Multiple operators fused into one generated Java function | Performance good sign |
| `AdaptiveSparkPlan` | AQE wrapper | Confirms AQE is active; operators beneath may change at runtime |
| `InMemoryTableScan` | Read from cache | Confirms a `cache()` hit |

---

## The debugging playbook — "my job is slow, what do I do?"

A mental sequence you can recite in an interview:

1. **Spark UI → Jobs tab** — which action is slow? (If the whole app is slow, skip to 2.)
2. **Jobs → click slow job → Stages tab** — which stage dominates the time?
3. **Stages → click slow stage → Summary Metrics** —
   - **Max duration >> Median?** → **skew** → [data_skew.md](data_skew.md)
   - **GC Time > 10%?** → **memory pressure** → [memory_management.md](memory_management.md)
   - **Non-zero spill?** → **under-partitioned** — bump `spark.sql.shuffle.partitions` or executor memory
   - **Input Size huge?** → **no predicate pushdown** — check `.explain()` for `Scan parquet → PushedFilters`
4. **SQL tab → click query → inspect plan** — is the join strategy what you expected? (Broadcast vs SortMerge?)
5. **Executors tab** — any dead executors? Consistent memory usage? GC time spikes on specific executors?
6. **If all above look fine** — check `.explain(mode="cost")` for wildly wrong size estimates driving bad plan decisions.

---

## Reading the DAG visualization

The Stages tab shows a DAG for each job. Every **box** is an operator. Every **line between stages** is a shuffle.

**Quick heuristics:**
- More stages = more shuffles = more cost. Count them.
- A long narrow chain of stages = a pipeline with many shuffles; see if any can be combined.
- A single huge stage at the end with a ton of output bytes = your final write stage. If slow, you're either writing to slow storage or producing too many small files.

---

## Less-obvious UI features worth knowing

### Event Timeline
On the Stages page, click "Event Timeline" to see task-level timing visualized. Gaps between task groups = scheduling delay. Tasks bunched on one executor = uneven executor utilization.

### Task Deserialization Time vs. Task Time
If deserialization time is a huge chunk of task time, your closure is too big (lots of broadcast vars, huge Python imports in PySpark). Trim what's captured.

### Executor Lost events
Under Stages → failed tasks → error message. "ExecutorLostFailure" is the umbrella for OOM, preemption, network partition, etc. Always read the actual message, not just the label.

### SQL metrics — "number of output rows"
For each operator. If `Filter` shows input 1M, output 999K, your filter doesn't filter much — reconsider whether it belongs. If `Scan parquet` shows 10M input but the final output is 1K, you're reading way more than you need — check if the filter pushed down.

---

## Interview one-liners (memorize these)

- **"Your job is slow. What do you look at first?"** → Spark UI → Stages tab → find the dominant stage → Summary Metrics. Check Max vs Median duration (skew), GC time (memory), spill (partitioning), input size (pushdown).
- **"How do you know if predicate pushdown worked?"** → In the physical plan, the `Scan parquet` line shows `PushedFilters: [...]` with your filter listed. If empty, it didn't push — usually because a UDF or a non-supported data source is in the way.
- **"How do you detect skew from the UI?"** → Stages → Summary Metrics. Max task duration (or shuffle read size) > 10× Median = skew. One or two outlier tasks in the tasks table confirm it.
- **"What does an `Exchange` operator mean in the plan?"** → A shuffle. Each Exchange = one shuffle boundary = one new stage. Count them to count your job's shuffles.
- **"What's the difference between `BroadcastHashJoin` and `SortMergeJoin` in the plan?"** → Broadcast sends the small side to every executor, no big-side shuffle — fast. SortMerge shuffles + sorts both sides — the fallback.
- **"What does `WholeStageCodegen` do?"** → Fuses a chain of operators into a single generated Java function. Skips the overhead of iterator-per-operator. You want to see this in plans for performance.
- **"How do you verify AQE is active?"** → `.explain()` shows `AdaptiveSparkPlan` at the top. Also check `spark.conf.get("spark.sql.adaptive.enabled")` returns `true`.

---

## Common pitfalls

1. **Flying blind.** Running a job and waiting for it to finish without ever opening the UI. Debug iteratively, not after.
2. **Trusting `.explain()` without the UI.** The plan shows what Spark *intended*. The UI shows what actually happened — including runtime skew, stragglers, and AQE re-planning.
3. **Ignoring GC Time.** Consistent >10% GC time is always a real issue, even when the job "completes."
4. **Reading plans top-down.** Physical plans execute bottom-up. Read them from the leaf nodes (Scans) up to the root.
5. **Not checking `PushedFilters`.** A silent missing pushdown can make a job read 10× more data than needed.
6. **Confusing job-level and stage-level time.** A job can look fast while a stage inside is slow if there are few stages. Always drill into stages.
7. **Assuming the first stage is the important one.** Usually the longest stage is somewhere in the middle — the one where the big shuffle or join happens.
8. **Closing the UI when the job finishes.** In a live cluster, the UI stays available until the app shuts down. In a batch run, the **history server** (`http://<host>:18080`) has the UI for completed apps. Always check the history server for post-mortems.

---

## Practice exercise

Open [`../joins.ipynb`](../joins.ipynb). Before running each cell:

1. Predict what the physical plan will look like (number of shuffles, join strategy).
2. Run `.explain()` and check your prediction.
3. Run the cell.
4. Open `localhost:4040` → SQL tab → click the latest query → compare the runtime plan against your prediction.

Do this three times with three different joins (inner, broadcast, salted) and your muscle memory will build fast.

---

## Further reading

**Official:**
- [Monitoring and Instrumentation (official Spark docs)](https://spark.apache.org/docs/latest/monitoring.html) — every UI tab, explained.
- [Web UI documentation (official)](https://spark.apache.org/docs/latest/web-ui.html) — detailed breakdown of each page.

**Community:**
- [**A Deep Dive into Spark UI for Job Optimization** — Microsoft Tech Community](https://techcommunity.microsoft.com/blog/microsoftmissioncriticalblog/a-deep-dive-into-spark-ui-for-job-optimization/4442229) — excellent walkthrough with screenshots.
- [**Understanding your Apache Spark Application Through Visualization** — Databricks (2015)](https://www.databricks.com/blog/2015/06/22/understanding-your-apache-spark-application-through-visualization.html) — the original post that introduced the DAG viz.

Related in this repo: [`shuffle_and_partitioning.md`](shuffle_and_partitioning.md), [`memory_management.md`](memory_management.md), [`data_skew.md`](data_skew.md), [`catalyst_and_aqe.md`](catalyst_and_aqe.md).

---

*Last revised: 2026-04*
