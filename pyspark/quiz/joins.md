# 🔗 Joins — Self-Check Quiz

**How to use this:** Read the question. Say your answer out loud like you're in an interview. Then click to expand.

**Paired with:** [`../joins.ipynb`](../joins.ipynb)

**Difficulty mix:** 🟢 basics → 🟡 intermediate → ⚡ senior

---

## 🟢 Basics

<details>
<summary><strong>Q1.</strong> Name the 6 PySpark join types and one use case each.</summary>

<br>

| Type | Returns | Use case |
|---|---|---|
| `inner` | Rows matching on both sides | Default reconciliation |
| `left` | All left + matching right (nulls where unmatched) | Keep all orders, attach customer info where available |
| `right` | All right + matching left | Rarely used — just flip and use `left` |
| `full_outer` | All rows from both sides | Full reconciliation audit |
| `left_semi` | Left rows with a match, only left columns | "Customers who placed an order" (cheaper than `inner + distinct`) |
| `left_anti` | Left rows with **no** match | "Customers who never ordered" — the SQL `NOT EXISTS` pattern |

</details>

<details>
<summary><strong>Q2.</strong> You inner-join POS (<code>store_id: int</code>) with loyalty (<code>store_id: string "store_042"</code>) on <code>store_id</code>. Count is 0 but both tables have data. What happened?</summary>

<br>

**Type mismatch.** Spark compares `42` (int) with `"store_042"` (string) — they never hash-equal. Depending on version, you get either an empty result or an `AnalysisException`.

**Fix:** normalize one side. `F.regexp_replace("store_id", "store_0*", "").cast("int")` strips the prefix and casts to int.

**Prevention:** always run `.printSchema()` on both sides before joining.

</details>

<details>
<summary><strong>Q3.</strong> What's the difference between <code>left_semi</code> and <code>inner</code>?</summary>

<br>

- `inner` returns columns from **both** sides.
- `left_semi` returns columns from the **left side only** — it's "does a match exist?" without actually pulling the right-side data.

`left_semi` is faster and avoids the column-collision headache. Use it when you only want the existence check, e.g., "customers who placed an order" — you don't need the order details, just the customer.

</details>

---

## 🟡 Intermediate

<details>
<summary><strong>Q4.</strong> What's a broadcast join, when should you use it, and what's the default size threshold?</summary>

<br>

**What:** Spark sends a full copy of the **small** side to every executor. Each executor then joins its local slice of the big side against the in-memory copy — **no shuffle**.

**When:** one side is small enough to fit in executor memory. Typically dimension tables — product catalog, store lookup, country codes.

**Default threshold:** `spark.sql.autoBroadcastJoinThreshold = 10 MB`. Spark *sometimes* auto-broadcasts based on size estimates, but estimates are often wrong. **Be explicit:**

```python
from pyspark.sql.functions import broadcast
big.join(broadcast(small), "key")
```

**Don't broadcast:** anything > ~100 MB. The driver has to collect the entire small side first, then send it to every executor — memory pressure + network storm.

</details>

<details>
<summary><strong>Q5.</strong> How do you detect skew in a join, both from code and from the Spark UI?</summary>

<br>

**From code:**
```python
df.groupBy("join_key").count().orderBy(F.col("count").desc()).show(20)
```
If the top key has 10x+ the median count, you have skew.

**From Spark UI:**
- Go to **Stages → slow stage → Summary Metrics**
- Compare **Min / Median / Max task duration** and **Shuffle Read Size**
- Max > 10× median = skew
- In the tasks table below, one or two tasks with much larger shuffle read sizes confirm it

**The tell:** most tasks finish quickly, one task takes forever. Your total job time = that one task's time.

</details>

<details>
<summary><strong>Q6.</strong> Three levels of fix for a skewed join, in order of preference.</summary>

<br>

1. **Enable AQE skew join** (Spark 3+):
   ```python
   spark.conf.set("spark.sql.adaptive.enabled", "true")
   spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
   ```
   AQE detects skewed partitions at runtime and splits them automatically. **Zero code change** — try this first.

2. **Broadcast the smaller side** — if it fits in memory, this eliminates the shuffle entirely, which eliminates skew by definition.

3. **Manual salting** — nuclear option. Add a random `salt` column to the big side (values 0 to N-1), explode the small side N times with every salt value, join on `(key, salt)`. See Q8.

</details>

<details>
<summary><strong>Q7.</strong> You want "POS transactions at stores that have <em>no</em> loyalty records" — how do you write that efficiently?</summary>

<br>

**`left_anti` join.** It's the SQL `WHERE NOT EXISTS` implemented as a set operation:

```python
orphan_pos = pos_transactions.join(
    loyalty_normalized,
    on="store_id",
    how="left_anti",
)
```

Returns every POS row whose `store_id` has no match in loyalty. Fast and clean — no subquery, no correlation.

**Common mistake:** using `left` join + `.filter(col("loyalty_id").isNull())`. Works but slower and more verbose.

</details>

---

## ⚡ Senior / interview judgment

<details>
<summary><strong>Q8.</strong> Walk through manual salting for a skewed join. Big side has 1B rows with one hot <code>store_id=42</code> (20% of data). Small side has 10M rows.</summary>

<br>

**Goal:** distribute store 42's rows across N partitions instead of 1.

```python
N = 20  # salt bucket count — more for heavier skew

# 1. Salt the big side: every row gets a random salt 0..N-1
big_salted = big.withColumn("salt", (F.rand() * N).cast("int"))

# 2. Explode the small side: each row replicated N times, once per salt
small_exploded = small.withColumn(
    "salt",
    F.explode(F.array([F.lit(i) for i in range(N)]))
)

# 3. Join on the composite key (original_key, salt)
result = big_salted.join(small_exploded, ["store_id", "salt"])
```

**What happened:** store 42's rows now distribute across 20 partitions. Small side grew 20× (still small). The hot partition is gone.

**Cost:** 20× more rows on the small side in memory + some shuffle overhead on the replication. **Benefit:** the 20% skew is eliminated.

**When to use:** AQE didn't help AND the small side is too big to broadcast. It's the "I know what I'm doing" override.

</details>

<details>
<summary><strong>Q9.</strong> You run <code>.explain()</code> on a join and see <code>SortMergeJoin</code>. You expected <code>BroadcastHashJoin</code>. Walk through the diagnosis.</summary>

<br>

**Possible causes, in order of likelihood:**

1. **Small side is over the broadcast threshold.** Default is 10 MB. Check:
   ```python
   spark.conf.get("spark.sql.autoBroadcastJoinThreshold")
   small.rdd.map(lambda r: len(str(r))).sum()  # rough size estimate
   ```
   Fix: bump `autoBroadcastJoinThreshold` or use an explicit `broadcast()` hint.

2. **Size estimate is wrong.** Spark's stats can lie, especially for Parquet files that weren't collected or for DataFrames built from expensive transformations. Use the explicit `broadcast()` hint — it forces the choice.

3. **Broadcast hint was applied to the wrong side.** `big.join(broadcast(small))` ≠ `broadcast(small).join(big)` — check which side actually has the hint.

4. **Size exceeds `spark.driver.maxResultSize`.** The driver has to collect the small side before broadcasting. If it blows the driver, Spark falls back to SortMergeJoin.

5. **Type mismatch on the join key** — Spark can't use a hash join strategy when the types don't match exactly.

**How to verify the fix:** re-run `.explain()` and grep for `BroadcastHashJoin` in the physical plan.

</details>

<details>
<summary><strong>Q10.</strong> What's the real-world difference between an inner join and a semi join at scale (500M rows)? When does the distinction actually matter?</summary>

<br>

**Technically:** `inner` returns columns from both sides; `left_semi` returns only left columns.

**At scale, it matters because:**
- `left_semi` doesn't pull the right-side data across the shuffle — less data transferred, faster
- Inner can produce **multiple output rows** per left row if the right side has duplicates on the join key. Semi produces exactly one.
- Inner requires handling column collisions; semi doesn't.

**When it matters:**
- "Users who made a purchase in the last 30 days" — use `left_semi`. You want user rows, not user-purchase combinations. `inner` would explode to one row per purchase.
- "Enrich orders with user details" — use `inner` (you want the user columns).

**Interview signal:** people who reach for `inner + distinct` when they actually want `left_semi` are missing a tool. Senior DEs pick the right join by intent.

</details>

---

## 🎯 How did you do?

- **9–10 correct** → interview-ready on joins. Drill the [shuffle theory doc](../theory/shuffle_and_partitioning.md) for the internals.
- **6–8 correct** → solid grasp. Re-walk the sections you missed in the notebook.
- **< 6 correct** → go back to [`../joins.ipynb`](../joins.ipynb), do the Boss Level, then return.

**Related:** [quiz/window_functions.md](window_functions.md), [quiz/memory_management.md](memory_management.md)
