# 🪟 Window Functions — Self-Check Quiz

**How to use this:** Read the question. Think about your answer (say it out loud like you're in an interview). Then click to expand the answer.

**Paired with:** [`../window_functions.ipynb`](../window_functions.ipynb)

**Difficulty mix:** 🟢 basics → 🟡 intermediate → ⚡ senior

---

## 🟢 Basics

<details>
<summary><strong>Q1.</strong> What's the difference between <code>groupBy</code> and a window function?</summary>

<br>

`groupBy` **collapses** rows — you get one row per group. Window functions **keep every row** and attach a computed value (rank, running total, previous value, etc.) to each one.

Use `groupBy` when you want aggregated output. Use windows when you want to compare each row to its neighbors *without losing the original rows*.

</details>

<details>
<summary><strong>Q2.</strong> What does this window definition mean? <code>Window.partitionBy("customer_id").orderBy("month")</code></summary>

<br>

"Group rows by `customer_id`, and within each group, sort by `month` ascending."

Any window function applied with `.over(w)` then operates **inside that sorted group**. `lag("month", 1).over(w)` = "for each row, the month value from one row earlier, within this customer's sorted timeline."

Without `partitionBy`, the window spans the whole DataFrame (usually not what you want and can crash on large data).

</details>

<details>
<summary><strong>Q3.</strong> What does <code>lag("spend", 2)</code> return for the first two rows of each partition?</summary>

<br>

`null`. There's no "2 rows earlier" for them. This is normal — you filter or handle nulls downstream. It's also why you can't just do `spend < prev_spend` without thinking about nulls (they'll silently drop from comparisons).

</details>

---

## 🟡 Intermediate

<details>
<summary><strong>Q4.</strong> Explain the difference between <code>row_number()</code>, <code>rank()</code>, and <code>dense_rank()</code> using ties.</summary>

<br>

Suppose 4 customers with spends: 100, 90, 90, 80.

| Function | Output |
|---|---|
| `row_number()` | 1, 2, 3, 4 — unique sequential, ties broken arbitrarily |
| `rank()` | 1, 2, 2, 4 — ties share a rank, then **skip** |
| `dense_rank()` | 1, 2, 2, 3 — ties share a rank, **no skip** |

**When to pick which:**
- `row_number()` → you need exactly N rows regardless of ties (top 2 for a slide deck).
- `rank()` → ties should both appear; gaps are meaningful ("rank 3 means 2 were ahead").
- `dense_rank()` → ties should both appear; gaps don't matter ("top 3 tiers" regardless of count).

</details>

<details>
<summary><strong>Q5.</strong> You want to find customers who bought in <strong>3 consecutive months</strong>. Why can't you solve this with <code>groupBy(customer_id).count()</code>?</summary>

<br>

`count()` tells you *how many* months they bought in — not whether those months were *adjacent*. Bob buying in Jan, Feb, and April has a count of 3 but is NOT 3 consecutive.

You need each row to "know" what came before it, which means a window function with `lag()`.

</details>

<details>
<summary><strong>Q6.</strong> What's the "lag-the-flag" trick for finding a streak of 3?</summary>

<br>

1. Create `is_consecutive = 1` when `current_month - prev_month == 1`, else 0.
2. That flag only tells you streaks of **2**. To find streaks of **3**, lag the flag itself: `prev_consecutive = lag(is_consecutive, 1)`.
3. Filter rows where both `is_consecutive = 1` AND `prev_consecutive = 1` — that means **this row touches the previous**, and **the previous touches the one before**. Three in a row.

Generalizes: for a streak of N, you lag the flag N-2 times.

</details>

<details>
<summary><strong>Q7.</strong> You need the top 2 customers <em>per month</em>. Your window is <code>partitionBy("customer_id").orderBy("spend")</code>. What's wrong?</summary>

<br>

The partition key is wrong. "Top 2 per month" means you want to rank **within each month**, so you partition by `month` and order by `spend desc`:

```python
w = Window.partitionBy("month").orderBy(F.col("spend").desc())
df.withColumn("rn", F.row_number().over(w)).filter("rn <= 2")
```

**The mental move:** the partition key is "the group I want rankings *inside*". Change the question → change the partition.

</details>

---

## ⚡ Senior / interview judgment

<details>
<summary><strong>Q8.</strong> A job using <code>Window.orderBy("ts")</code> (no <code>partitionBy</code>) is extremely slow on a 500M-row table. Why, and how do you fix it?</summary>

<br>

**Why slow:** without `partitionBy`, Spark has to pull the entire dataset into a **single partition** to compute the ordering globally. One task does all the work — no parallelism, likely OOM.

**Fix:**
- If the business logic allows, add a `partitionBy` on a meaningful key (customer, store, day) so the window parallelizes.
- If you genuinely need a global order, ask whether you actually need it — often "top 100 by recency" can be approximated per-partition then unioned.
- As a last resort, pre-sort and write out sorted by the key, then use a local windowed scan.

**Interview signal:** senior candidates spot the missing `partitionBy` instantly. Juniors throw more memory at it.

</details>

<details>
<summary><strong>Q9.</strong> Two customers tie for spend in January. Marcus asks for "top 2 per month" for a board slide. Which ranking function do you use, and what should you do <em>before</em> writing the code?</summary>

<br>

**What you do first:** ask Marcus whether he wants exactly 2 rows (breaking the tie arbitrarily) or both tied customers shown (returning 3 rows that month). **There is no universal right answer** — the requirement is ambiguous and your job is to surface the ambiguity, not guess.

**After asking:**
- "Exactly 2, I don't care which" → `row_number()`
- "Show all tied winners" → `rank()` with `<= 2` (but be explicit that row count per month can vary)
- "Show top 2 tiers" → `dense_rank()`

The senior-vs-junior distinction here isn't the SQL — it's noticing the ambiguity and asking.

</details>

<details>
<summary><strong>Q10.</strong> Window functions cause shuffles. How can you minimize shuffle cost for a recurring window query on the same key?</summary>

<br>

- **Bucket your storage by the window's partition key.** E.g., if you always `partitionBy("customer_id")`, write the table with `bucketBy(200, "customer_id")` — a saved table, not just `df.write.partitionBy(...)`. File-source directory partitioning doesn't report output partitioning to the planner, so a plain partitioned-Parquet layout still triggers an Exchange; bucketing is what actually lets subsequent window queries skip the shuffle, because Spark's catalog knows the data is already co-located.
- **Enable AQE** (`spark.sql.adaptive.enabled=true`) — coalesces small post-shuffle partitions automatically.
- **Pre-aggregate before the window** when possible. If you only need monthly rollups, `groupBy(customer_id, month).agg(...)` first — that shuffle is cheaper than a shuffle over raw transactions, and the window runs on a much smaller DataFrame.
- **Avoid re-keying unnecessarily.** If a DataFrame is already partitioned by `customer_id` from an upstream step, a window `partitionBy("customer_id")` can skip the shuffle.

See also: [`../theory/shuffle_and_partitioning.md`](../theory/shuffle_and_partitioning.md).

</details>

---

## 🎯 How did you do?

- **9–10 correct** → you're interview-ready on windows. Drill the theory docs next.
- **6–8 correct** → solid grasp. Re-walk the notebook sections where you missed.
- **< 6 correct** → go back to [`../window_functions.ipynb`](../window_functions.ipynb) and do the Boss Level from scratch before returning here.

**Related:** [quiz/joins.md](joins.md), [quiz/memory_management.md](memory_management.md)
