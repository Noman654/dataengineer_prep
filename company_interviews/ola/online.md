# Ola — Data Engineer Interview

> **Sources:** Naukri, YouTube interview experiences, community reports, Levels.fyi
> **Last updated:** 2026-04-22
> **Roles covered:** Data Engineer (L3/L4), Senior DE

---

## Round Breakdown

| Round | Format | Duration | Focus |
|---|---|---|---|
| Round 1 | Recruiter Screen | ~20–30 min | Background, experience |
| Round 2 | DSA / Problem Solving | ~60 min | Arrays, Trees, Heaps, DP |
| Round 3 | SQL + Data Modeling | ~60 min | Complex queries, schema design |
| Round 4 | System Design | ~60–75 min | Pipeline architecture |
| Round 5 | Hiring Manager | ~45 min | Projects, behavioural |
| Round 6 | (Senior only) Bar Raiser | ~45 min | In-depth tech + culture |

---

## Round 2 — DSA / Problem Solving

Expect **Medium–Hard** LeetCode level. Interviewers ask for basic solution first → then optimize.

### Questions asked

1. Find the maximum sum subarray (Kadane's algorithm).
2. Given a matrix, find the longest path of consecutive numbers.
3. Merge K sorted linked lists (heap-based approach).
4. Sliding window: maximum of all subarrays of size K.
5. Two-pointer: container with most water.
6. Design a data structure that supports `insert`, `delete`, `getRandom` in O(1).
7. Find the median from a data stream (using two heaps).

> **Tip:** Always clarify edge cases before coding. Talk through your approach, then code. Optimize after your first working solution.

---

## Round 3 — SQL + Data Modeling

SQL is **heavily tested** — expect incrementally harder questions.

### SQL questions asked

1. Given a `rides` table with columns `(ride_id, driver_id, user_id, city, fare, ride_date)`:
   - Find the top 3 drivers by revenue in each city this month.
   - Find users who have taken more than 5 rides in the last 7 days.
   - Calculate the 7-day rolling average fare per city.

2. Find all drivers who have had **no rides** in the last 30 days but had rides in the 30 days before that.

3. Write a query to rank drivers by number of rides, handling ties with `DENSE_RANK()`.

4. Given an `events` table `(user_id, event_type, event_time)`, find users who performed events A → B → C in order within a 1-hour window.

5. Explain the difference between `WHERE` and `HAVING`. When does each execute in the query lifecycle?

6. How do you optimize a slow SQL query? What would you check first?

7. When would you use NoSQL over SQL at Ola's scale?

### Data Modeling questions

1. Design a schema for Ola's ride-sharing system. What are your fact and dimension tables?
2. How would you handle schema evolution in a data warehouse without breaking existing pipelines?
3. Star schema vs Snowflake schema — which would you pick for an analytical dashboard? Why?

---

## Round 4 — System Design (Pipeline Architecture)

### Questions asked

1. **Design a real-time anomaly detection pipeline** for Ola driver GPS data.
   - How do you ingest GPS pings (Kafka)?
   - How do you detect anomalies (Spark Streaming / Flink)?
   - How do you alert in real-time vs batch report overnight?

2. **Design a daily batch pipeline** that calculates driver earnings, city-level revenue, and surge pricing adjustments.
   - Tools: Kafka → Spark → S3/HDFS → Hive/Redshift → Dashboard
   - How do you handle late-arriving data?
   - How do you backfill 6 months of historical data?

3. **How would you handle data quality?**
   - Null checks, schema validation, deduplication
   - What happens when bad data reaches production?

4. **What is the trade-off between batch and streaming?**
   - Latency vs. throughput
   - Exactly-once vs at-least-once semantics

### Key concepts to know

- Kafka producer/consumer, consumer groups, partitions
- Spark Structured Streaming — watermarks, triggers, output modes
- Airflow for orchestration — DAG design, retries, SLAs
- Delta Lake / Iceberg for ACID transactions on data lakes
- Backfilling strategies

---

## Round 5 — Hiring Manager (Projects + Behavioural)

### Project questions

- "Describe the most complex data pipeline you've built."
- "How did you debug a production issue in your pipeline?"
- "What would you redesign if you could start over?"
- "How do you ensure your pipelines are maintainable by others?"

### Behavioural questions

- "Tell me about a time you dealt with ambiguous requirements."
- "How did you manage competing priorities from different stakeholders?"
- "Describe a data quality issue you discovered and how you fixed it."
- "How do you keep up with new data engineering tools and frameworks?"

---

## Tips

- Ola focuses heavily on **scale** (millions of rides/day) — always think about scalability in your answers.
- In system design, **clarify requirements before designing** — latency SLA? data volume? structured or unstructured?
- SQL round is genuinely hard — practice Medium/Hard window function problems on LeetCode.
- Don't rush to code in DSA rounds — interviewers prefer a well-explained brute force over a silent optimal.
- Know your resume cold — they will challenge every technical choice you've made.

---

## Related

- [Spark concepts → pyspark/theory/](../pyspark/theory/)
- [Window functions quiz → pyspark/quiz/window_functions.md](../pyspark/quiz/window_functions.md)
- [Joins quiz → pyspark/quiz/joins.md](../pyspark/quiz/joins.md)
