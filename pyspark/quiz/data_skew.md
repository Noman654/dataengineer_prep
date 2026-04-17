# 🔥 Data Skew — Self-Check Quiz

**How to use this:** Read the question. Say your answer out loud like you're in an interview. Then click to expand.

**Paired with:** [`../theory/data_skew.md`](../theory/data_skew.md)

**Difficulty mix:** 🟢 basics → 🟡 intermediate → ⚡ senior

---

## 🟢 Basics

<details>
<summary><strong>Q1.</strong> What is data skew in Spark?</summary>

<br>

One key has dramatically more rows than others. Because shuffles distribute by hash of the key, the skewed key ends up on **one reducer task while others sit idle**. Total job time = slowest task's time, so one hot key can make a 5-minute job take an hour.

</details>

<details>
<summary><strong>Q2.</strong> Name three places skew shows up in Spark workloads.</summary>

<br>

1. **Join skew** — joining on a key with uneven distribution (`customer_id`, `store_id`).
2. **GroupBy skew** — aggregating on a key where one value dominates (`country` in a US-centric dataset).
3. **Window skew** — `Window.partitionBy("status")` when 90% of rows are `"active"`.

Bonus: **write skew** — `df.write.partitionBy("category")` with skewed categories creates one huge file and thousands of tiny ones.

</details>

<details>
<summary><strong>Q3.</strong> You check <code>df.groupBy("key").count().orderBy(desc).show()</code> and see the top key has 1M rows while the median is 1K. What does that tell you?</summary>

<br>

**1000× skew.** Severe. Any `groupBy` or `join` on this key will have one task handling 1000× the work of the median task.

You need to do something before running downstream joins or aggregations on this key:
- Enable AQE skew join (first line of defense).
- Broadcast the small side if joining against a dimension table.
- If both fail: manual salting.

</details>

---

## 🟡 Intermediate

<details>
<summary><strong>Q4.</strong> How do you detect skew from the Spark UI?</summary>

<br>

1. **Stages tab** → click the slow stage.
2. **Summary Metrics** at the top of the stage page.
3. Compare **Min / Median / Max** for:
   - **Duration** — Max >> Median = skewed runtimes.
   - **Shuffle Read Size** — Max >> Median = skewed data volumes.
4. **Tasks table** below — sort by duration descending. One or two outlier tasks 50× slower than the rest = confirmed skew.

**Telltale sign:** the stage sits at "199/200 succeeded" for 40 minutes waiting on that one task.

</details>

<details>
<summary><strong>Q5.</strong> Walk through the fix hierarchy for a skewed join (4 levels).</summary>

<br>

1. **AQE skew join** (Spark 3+). Zero code change. Detects skewed partitions at runtime and splits them. Solves 80% of cases.

2. **Broadcast the small side.** If one side fits in memory (< ~100 MB), no shuffle = no skew.

3. **Manual salting.** Add random salt 0..N-1 to the big side, replicate the small side N times with all salt values, join on `(key, salt)`.

4. **Two-phase aggregation** (for groupBy skew only). Aggregate by `(key + random_salt)` first, then roll up by `key`. The expensive shuffle happens on balanced data.

Try them in that order. Don't skip to salting — it adds complexity you may not need.

</details>

<details>
<summary><strong>Q6.</strong> Explain the salting technique in one paragraph — what it does and why it works.</summary>

<br>

Salting breaks one hot key into N virtual sub-keys. You add a random `salt` column (0 to N-1) to the big/skewed side so each row gets one of N possible salt values. On the small side, you explode each row N times — once per salt value — so every (key, salt) combination has a match. Then you join on the composite key `(original_key, salt)`. Because the hash is now on `(key, salt)`, the hot key's rows distribute across N partitions instead of landing on one. You pay 20× more rows on the small side (still cheap if it started small) but eliminate the one slow task. It works because you've artificially increased key cardinality to break the hash collision that was causing skew.

</details>

<details>
<summary><strong>Q7.</strong> What's "asymmetric salting" and when would you use it?</summary>

<br>

**Full salting** inflates the small side N× for every key, even the ones that aren't hot. That's wasteful.

**Asymmetric salting** only salts the known hot keys:
- Big side: hot keys get salt 0..N-1, other keys get salt=0.
- Small side: hot-key rows get exploded N times, non-hot-key rows stay salt=0.

Non-hot keys join normally; hot keys distribute across N partitions.

**When to use:** production pipelines where you know which keys are hot (e.g., `store_id = 42` is always the downtown flagship) and you don't want to inflate the whole small side.

**Tradeoff:** more code complexity, but much cheaper on the small side.

</details>

---

## ⚡ Senior / interview judgment

<details>
<summary><strong>Q8.</strong> AQE is enabled but your skewed join is still slow. Why might that happen, and what do you do next?</summary>

<br>

**Possible reasons:**

1. **Extreme skew.** One key is > 50% of the data. AQE splits the skewed partition, but splitting one huge key doesn't help much if the replicated other-side rows are also large.

2. **Threshold not met.** AQE skew detection triggers when a partition is **both** > `skewedPartitionFactor × median` (default 5×) **and** > `skewedPartitionThresholdInBytes` (default 256 MB). If your partition is skewed but under 256 MB, AQE doesn't kick in. Lower the threshold: `spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "64MB")`.

3. **It's a groupBy, not a join.** AQE's skew handling is join-focused. For aggregation skew, use two-phase aggregation with a salt prefix.

4. **AQE isn't actually enabled.** Check `spark.conf.get("spark.sql.adaptive.enabled")`. In 3.0/3.1 it was off by default.

**Next step:** fall through to manual salting, asymmetric if you know the hot keys.

</details>

<details>
<summary><strong>Q9.</strong> You have <code>df.groupBy("country").agg(F.sum("amount"))</code> on a dataset that's 70% US rows. AQE didn't help. What do you do?</summary>

<br>

**Two-phase aggregation with a salt prefix:**

```python
# Phase 1: pre-aggregate with random salt to distribute load
df_pre = (
    df
    .withColumn("salt", (F.rand() * 100).cast("int"))
    .groupBy("country", "salt")
    .agg(F.sum("amount").alias("partial"))
)

# Phase 2: final aggregate — inputs are now balanced
result = (
    df_pre
    .groupBy("country")
    .agg(F.sum("partial").alias("total"))
)
```

**Why this works:** phase 1 uses `(country, salt)` as the key, spreading US across 100 partitions instead of 1. Phase 2 rolls up 100 small partial sums per country into the final totals.

**Cost:** two shuffles instead of one, but each is balanced. For severe skew, two balanced shuffles beat one skewed shuffle by orders of magnitude.

</details>

<details>
<summary><strong>Q10.</strong> Your pipeline has worked for 18 months. Last week it started taking 3 hours instead of 20 minutes. Nothing in the code changed. How do you diagnose?</summary>

<br>

**Working hypothesis: skew drift.** A key that was uniform has become skewed as the data grew.

**Diagnosis:**
1. **Compare key distributions over time.** Run `df.groupBy(key).count().orderBy(desc).show(20)` on yesterday's data and on data from 3 months ago. If one key is suddenly 30× more common, there's your answer.
2. **Check the Spark UI from the slow run vs a past successful run.** Compare stage-level Max/Median task durations. A healthy stage 3 months ago with Max 2× Median and a skewed one today with Max 50× Median pinpoints it.
3. **Ask what changed in the *data*, not the *code*.** A new customer onboarded? A feature launched that concentrated activity? A partner merger combined two previously separate populations under one key?

**Fix (in order):**
1. Enable or verify AQE skew join is active.
2. If a specific hot key is identified, add asymmetric salting for it.
3. Consider re-keying: if `customer_id` is now skewed, maybe `(customer_id, partition_date)` isn't.

**Interview signal:** senior DEs think about *data drift* causing performance changes, not just code bugs. Juniors look at the code first.

</details>

---

## 🎯 How did you do?

- **9–10 correct** → skew is your strong suit. Drill [`catalyst_and_aqe.md`](../theory/catalyst_and_aqe.md) next.
- **6–8 correct** → solid. Practice salting on real data.
- **< 6 correct** → re-read [data_skew.md](../theory/data_skew.md), then watch [Daniel Tomes' skew talk](https://www.youtube.com/watch?v=6zg7NTw-kTQ).

**Related:** [quiz/joins.md](joins.md), [quiz/catalyst_and_aqe.md](catalyst_and_aqe.md)
