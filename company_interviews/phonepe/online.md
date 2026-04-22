# PhonePe — Data Engineer Interview

> **Sources:** Medium (candidate blog posts), dataengineerthings.org, Naukri, community experiences
> **Last updated:** 2026-04-22
> **Roles covered:** Data Engineer (L3/L4), Senior Data Engineer

---

## Round Breakdown

| Round | Format | Duration | Focus |
|---|---|---|---|
| Round 1 | Online Assessment | ~60–90 min | SQL + DSA |
| Round 2 | Technical Interview I | ~60 min | SQL (deep dive) |
| Round 3 | Technical Interview II | ~60 min | Spark internals + pipeline design |
| Round 4 | Hiring Manager | ~45 min | Projects + system design |

---

## Round 1 — Online Assessment

Mix of SQL and DSA problems.

### SQL (Medium–Hard)

1. From a transactions table `(txn_id, user_id, merchant_id, amount, txn_time)`:
   - Find the top 5 merchants by total transaction volume this month.
   - Find users who have transacted at least once every day in the last 7 days.

2. Calculate the **running total** of transactions per user ordered by time.

3. Find the **median** transaction amount per merchant — without using `MEDIAN()` (use window functions).

4. Rank users by total spend, handling ties such that equal ranks are assigned and no ranks are skipped (`DENSE_RANK`).

### DSA

1. Find the longest consecutive sequence in an array.
2. Given an array of transaction amounts, find all pairs that sum to a target (Two Sum).
3. Serialize and deserialize a binary tree.

---

## Round 2 — Technical Interview I (SQL Deep Dive)

PhonePe deals with **massive financial transaction data** — SQL questions are practical and business-oriented.

### Window Functions (heavily tested)

1. Write a query to find each user's **3rd transaction** using `ROW_NUMBER()`.
2. Calculate a **7-day moving average** of daily transaction amounts per user.
   ```sql
   SELECT
     user_id,
     txn_date,
     amount,
     AVG(amount) OVER (
       PARTITION BY user_id
       ORDER BY txn_date
       ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
     ) AS moving_avg_7d
   FROM transactions;
   ```
3. Find **consecutive days** of transactions for each user (gap-and-island problem).
4. For each user, find the transaction just before and just after a failed transaction (`LAG` / `LEAD`).
5. Rank merchants by revenue **within each city** — return only top 3 per city.

### Conceptual SQL

1. Explain the difference between `ROWS BETWEEN` and `RANGE BETWEEN` in window functions.
2. How does indexing work? When does an index hurt performance?
3. Explain query execution order: `FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY`.
4. What is a correlated subquery? When would you avoid it?
5. How do you deduplicate records while keeping the most recent?

---

## Round 3 — Technical Interview II (Spark)

### Spark Internals

1. Explain the difference between RDDs, DataFrames, and Datasets.
2. What is the DAG (Directed Acyclic Graph) in Spark? How does it help with fault tolerance?
3. What is lazy evaluation? When does Spark actually execute?
4. What is a shuffle? Why is it expensive? How do you minimize shuffles?
5. Difference between `groupByKey` and `reduceByKey` — why is `reduceByKey` preferred?
6. What is data skew? How does it manifest? How do you fix it? (salting, broadcast joins)
7. How does the Catalyst optimizer work? What is predicate pushdown?
8. How do you handle Out-of-Memory (OOM) errors in Spark?
   - Adjust `spark.executor.memory`
   - Repartition to avoid large partitions
   - Avoid `collect()` on large datasets to the driver

### Spark Coding

1. Given a nested JSON file of transactions, write a Spark job to:
   - Flatten the nested structure
   - Calculate total spend per user per day
   - Write the result partitioned by date

2. Write a Spark job to deduplicate records based on `(user_id, txn_id)` keeping the latest `txn_time`.

3. When would you use a broadcast join vs. a shuffle hash join?

### Pipeline Design

1. Design an incremental ETL pipeline that processes new PhonePe transactions every hour.
   - How do you track what's already been processed?
   - How do you handle late-arriving data?
   - How do you ensure idempotency?

2. How would you design for exactly-once processing in a Kafka → Spark Streaming pipeline?

---

## Round 4 — Hiring Manager (Projects + System Design)

### Project deep-dive

- "Describe the largest data pipeline you've built — volume, latency, tools."
- "How did you handle a data quality incident in production?"
- "What's the most complex SQL problem you've solved at work?"

### System Design

1. Design PhonePe's transaction analytics platform — real-time fraud detection + daily reporting.
2. How would you build a data lineage system to track where each piece of data came from?
3. Design a reconciliation pipeline to validate that no transactions are lost between source and warehouse.

---

## Tips

- SQL is the **strongest focus** at PhonePe — window functions are non-negotiable, practice hard.
- PhonePe deals with financial data so **data integrity, idempotency, and reconciliation** come up a lot.
- In Spark rounds, explain the "why" not just the "what" — they want depth on internals.
- They hire for people who have dealt with **real production incidents** — prepare concrete stories.
- Practice Medium → Hard SQL on LeetCode (specifically window functions category).

---

## Related

- [Spark memory management → pyspark/theory/memory_management.md](../pyspark/theory/memory_management.md)
- [Data skew → pyspark/theory/data_skew.md](../pyspark/theory/data_skew.md)
- [Catalyst optimizer → pyspark/theory/catalyst_and_aqe.md](../pyspark/theory/catalyst_and_aqe.md)
- [Window functions quiz → pyspark/quiz/window_functions.md](../pyspark/quiz/window_functions.md)
