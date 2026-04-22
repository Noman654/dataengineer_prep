# Swiggy — Data Engineer Interview

> **Sources:** JoinTaro, community experiences, Naukri
> **Last updated:** 2026-04-22
> **Status:** Partial — limited public interview data available. Will be updated as more reports surface.
> **Roles covered:** Data Engineer (L3/L4)

---

## Round Breakdown

| Round | Format | Duration | Focus |
|---|---|---|---|
| Round 1 | Online Assessment | ~60 min | SQL + DSA |
| Round 2 | Technical Interview I | ~60 min | SQL, Data Modeling |
| Round 3 | Technical Interview II | ~60 min | Spark, Pipeline Design |
| Round 4 | System Design | ~60 min | End-to-end pipeline architecture |
| Round 5 | Hiring Manager | ~45 min | Projects, behavioural |

---

## Round 1 — Online Assessment

### SQL

1. From an `orders` table `(order_id, user_id, restaurant_id, delivery_partner_id, total_amount, order_time, delivered_time)`:
   - Find the average delivery time per restaurant.
   - Find delivery partners who have completed > 50 orders with an average rating > 4.5.
   - Find users who ordered from the same restaurant more than 3 times in a month.

2. Rank restaurants by number of orders in each city — return top 3 per city.

3. Find the first order placed by each user (use `ROW_NUMBER()` or `MIN()`).

### DSA

1. Two Sum — find pairs that sum to target.
2. Valid parentheses (stack problem).
3. Binary search on a rotated sorted array.

---

## Round 2 — Technical Interview I (SQL + Data Modeling)

### SQL questions

1. Given `delivery_events(event_id, partner_id, event_type, event_time)` where `event_type` is `PICKUP` or `DELIVERED`:
   - Calculate the time taken for each delivery (PICKUP → DELIVERED).
   - Find partners who have had more than 2 deliveries take > 60 minutes today.

2. Find restaurants with **revenue drop > 20%** compared to previous week.

3. Write a query to calculate the **7-day rolling average** of daily orders per city.

4. Find users who have not ordered in the last 30 days but ordered in the 30 days before (lapsed users).

5. Gap-and-island: Find delivery partners who have had **3 or more consecutive days** with 0 deliveries.

### Data Modeling

1. Design a schema for Swiggy's order management system.
   - Entities: Users, Restaurants, Dishes, Orders, Order Items, Delivery Partners
   - How do you handle promotions/discounts in the schema?
   - How do you model a dish that belongs to multiple categories?

2. How would you structure a data warehouse for Swiggy's operational analytics?
   - What are the key fact tables?
   - How do you handle SCD (Slowly Changing Dimensions) for restaurant menus that change frequently?

---

## Round 3 — Technical Interview II (Spark + Pipeline Design)

### Spark

1. How does data skew affect a Spark job processing restaurant orders? How would you fix it if `restaurant_id` is skewed?
2. Write a Spark job to compute the daily active users (DAU) from a clickstream events table.
3. Explain partitioning strategies — how many partitions would you use for a 100GB dataset?
4. How do you write incremental Spark jobs (only process new data since last run)?
5. What is Delta Lake? What problems does it solve over plain Parquet?

### Pipeline Design

1. How would you build a pipeline that tracks restaurant availability in real-time?
2. Design a batch job to compute Swiggy's daily business metrics (GMV, orders, avg delivery time per city).
3. How do you implement idempotency in an ETL pipeline?

---

## Round 4 — System Design

### Questions asked

1. **Design Swiggy's real-time delivery tracking system.**
   - Delivery partner sends GPS ping every 10 seconds for millions of active orders.
   - How do you ingest, process, and deliver real-time ETA to users?
   - Tools: Kafka → Flink/Spark Streaming → Redis → API

2. **Design a system to detect fraudulent orders** (e.g., abnormal refund patterns, fake delivery confirmations).
   - Batch anomaly detection vs. real-time? Trade-offs?

3. **Design Swiggy's data warehouse** for business analytics (GMV, ARPU, restaurant performance, delivery metrics).
   - Source systems → ingestion → transformation → serving layer
   - How do you handle late-arriving order updates (order status can change hours later)?

---

## Round 5 — Hiring Manager

- "What's the most impactful data project you've shipped?"
- "How do you handle stakeholders who want data faster than your pipeline can deliver?"
- "Tell me about a time your pipeline had a bug that affected business decisions."
- "How do you approach on-call / incident management for data pipelines?"

---

## Tips

- Swiggy focuses on **food delivery scale** — think real-time (ETA, tracking, availability) in your system design.
- SQL is important but pipeline design is **equally tested** here — know Airflow, Kafka, Spark Streaming.
- GPS/location data and streaming scenarios come up a lot — know real-time architectures.
- Behavioural round focuses on **ownership and impact** — frame answers around business outcomes.

> ⚠️ **Note:** Swiggy interview data is limited in public sources. If you have a recent experience, please contribute to this file!

---

## Related

- [Spark internals → pyspark/theory/](../pyspark/theory/)
- [Shuffle and partitioning → pyspark/theory/shuffle_and_partitioning.md](../pyspark/theory/shuffle_and_partitioning.md)
- [Window functions quiz → pyspark/quiz/window_functions.md](../pyspark/quiz/window_functions.md)
