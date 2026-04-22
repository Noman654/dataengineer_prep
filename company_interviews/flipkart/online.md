# Flipkart — Data Engineer Interview

> **Sources:** InterviewQuery, InterviewBit, GrowDataSkills, LeetCode Discuss, GeeksForGeeks
> **Last updated:** 2026-04-22
> **Roles covered:** Data Engineer (L3/L4/L5), Senior Data Engineer

---

## Round Breakdown

| Round | Format | Duration | Focus |
|---|---|---|---|
| Round 1 | Machine Coding (Spark) | ~60–90 min | Live Spark coding on dataset |
| Round 2 | Problem Solving + DSA | ~60 min | Arrays, Trees, Heaps |
| Round 3 | Data Modeling + SQL | ~60 min | Schema design + complex queries |
| Round 4 | System Design | ~60–75 min | Pipeline architecture |
| Round 5 | Managerial / Behavioural | ~45 min | Projects, leadership, culture |

---

## Round 1 — Machine Coding (Spark)

This is Flipkart's **signature round** — you are given a raw dataset (usually nested JSON or CSV) and asked to write a Spark program live on a shared screen.

### What they test

- Can you read and parse complex nested structures?
- Can you write clean, generalized, efficient Spark code?
- Do you know how to handle OOM errors mid-coding?
- Can you explain your optimization choices?

### Questions asked

1. Given a nested JSON of orders `(order_id, user_id, items: [{product_id, qty, price}], ts)`:
   - Flatten it to `(order_id, user_id, product_id, qty, price, ts)`
   - Calculate total revenue per product per day
   - Find the top 5 products by revenue in the last 30 days

2. Given a CSV of `(user_id, session_id, page, event_time)`, compute session duration per user (gap > 30 min = new session).

3. Deduplicate a large dataset on `(order_id)` keeping the earliest record — do it efficiently at scale.

4. Write a Spark job that reads from HDFS, performs a join with a small lookup table (broadcast join), and writes output partitioned by date.

### Common follow-ups after coding

- "Your job is running slow — how would you debug it?" (Spark UI, stages, tasks)
- "You're hitting an OOM on the executor — what do you do?"
- "Can you repartition this to reduce skew?"
- "What's the complexity of this join? How would you make it faster?"

---

## Round 2 — Problem Solving + DSA

### Questions asked

1. Find the kth largest element in an unsorted array (heap approach).
2. LRU Cache — implement with O(1) get and put.
3. Binary tree right side view.
4. Merge intervals.
5. Find all permutations of a string.
6. Check if a graph has a cycle (DFS + visited set).

---

## Round 3 — Data Modeling + SQL

### Data Modeling questions

1. **Design a schema for a cricket tournament system** — teams, players, matches, innings, runs.
   - What are your fact and dimension tables?
   - How do you handle slowly changing dimensions (SCD Type 2)?
   - What's your partition key for performance?

2. **Design a schema for Flipkart's order management system.**
   - How do you model order line items?
   - How do you handle returns and refunds in the schema?
   - Normalized vs. denormalized — which do you pick for OLAP?

### SQL questions asked

1. Given `orders(order_id, user_id, product_id, amount, order_date)`:
   - Find users who placed orders in January but NOT in February.
   - Calculate the month-over-month growth in revenue.
   - Find the product with the highest revenue each month.

2. Write a query to find the **running total** of orders per user.

3. Find all users who have placed orders on **3 or more consecutive days**.

4. Given a `sellers` table and `products` table, find sellers who have **no products** listed (anti-join pattern).

5. How do you optimize a query that is scanning a 10TB table? What would you check?

---

## Round 4 — System Design

### Questions asked

1. **Design a near-real-time pipeline for Flipkart's Big Billion Days sale.**
   - Millions of events/second (clicks, add-to-cart, purchases)
   - Kafka for ingestion → Spark Streaming → aggregated metrics → dashboard
   - How do you handle the sudden 10x traffic spike?
   - How do you ensure exactly-once processing?

2. **Design an incremental data pipeline** that syncs Flipkart's MySQL order DB → Redshift every hour.
   - CDC (Change Data Capture) with Debezium/Kafka
   - How do you handle schema changes?
   - How do you backfill 2 years of historical data?

3. **Design a data quality framework** for Flipkart's data warehouse.
   - What checks do you implement (nulls, duplicates, referential integrity, volume anomalies)?
   - How do you alert and quarantine bad data?
   - How do you track lineage?

### Key concepts to know

- Kafka → Spark Structured Streaming → Delta Lake / Iceberg
- Checkpointing and exactly-once semantics
- CDC patterns (Debezium, Maxwell)
- Airflow DAG design for backfill and retry
- Handling late data with watermarks
- Delta Lake ACID properties — MERGE, UPDATE, DELETE

---

## Round 5 — Managerial / Behavioural

### Questions asked

- "Walk me through the most complex pipeline you've built — what was hard about it?"
- "How do you handle a production outage in your pipeline at 2am?"
- "Describe a situation where you pushed back on a requirement — how did it go?"
- "How do you decide when to use streaming vs. batch?"
- "How do you ensure code quality and maintainability in your team?"

---

## Tips

- **Practice the machine coding round** — this is unique to Flipkart. Actually write Spark code on datasets, not just read theory.
- **Know your Spark internals cold** — they probe deeply on shuffles, OOM, Catalyst, and performance tuning.
- **SQL round is schema-first** — design the model before writing queries, they evaluate your modeling instincts.
- In system design, always **ask clarifying questions first** — SLA, data volume, update frequency, consistency requirements.
- Flipkart runs e-commerce at Indian scale — always think about **surge scenarios** (Big Billion Days).

---

## Related

- [Spark internals → pyspark/theory/](../pyspark/theory/)
- [Shuffle and partitioning → pyspark/theory/shuffle_and_partitioning.md](../pyspark/theory/shuffle_and_partitioning.md)
- [Data skew → pyspark/theory/data_skew.md](../pyspark/theory/data_skew.md)
- [Joins → pyspark/joins.ipynb](../pyspark/joins.ipynb)
