# Dimensional Modeling — Quick Review

**Audience:** DE interview prep. Assumes you've seen a database schema before but haven't formally studied Kimball.

**Read time:** 15 minutes.

---

## TL;DR — the one-sentence mental model

> Dimensional modeling organizes data into **fact tables** (what happened — events, transactions, measurements) and **dimension tables** (the context — who, what, where, when). The **grain** defines what one row in the fact table represents. Get the grain wrong and everything downstream breaks.

---

## Why this matters in interviews

Every company in the `company_interviews/` folder asks data modeling questions. The pattern is always:

> *"Design a schema for [business process]."*

Juniors list table names. **Seniors follow the Kimball 4-step process:**

1. **Pick the business process** — what are we modeling? (orders, rides, logins)
2. **Declare the grain** — what does one row represent? (one order line item, one ride, one login event)
3. **Identify the dimensions** — who, what, where, when, how? (customer, product, store, date, payment method)
4. **Identify the facts** — what are we measuring? (amount, quantity, duration, distance)

If you remember one thing from this doc: **start with the grain**.

---

## Fact tables

A fact table stores **measurable events** — things that happened. Each row is one event at the declared grain.

### Three types of fact tables

| Type | What one row represents | Example (Zephyr Coffee) | When to use |
|---|---|---|---|
| **Transactional** | One event at a point in time | One line item in a sale (`tx_id, sku, qty, price, ts`) | When you need the full history of every event |
| **Periodic snapshot** | State at a regular interval | Monthly store performance (`store_id, month, total_revenue, total_orders`) | When you need "what did things look like at time T?" |
| **Accumulating snapshot** | Lifecycle of a process with milestones | Order fulfillment (`order_id, placed_ts, paid_ts, shipped_ts, delivered_ts`) | When a process has a defined start → end with stages |

**Transactional facts** are by far the most common in interviews. If the interviewer says "design a schema for X," default to transactional unless the question implies a lifecycle or periodic reporting.

### What goes in a fact table

- **Measures** (numeric, additive) — amount, quantity, duration, distance. These are what you `SUM`, `AVG`, `COUNT`.
- **Foreign keys** to dimension tables — `customer_id`, `store_id`, `product_id`, `date_key`.
- **Degenerate dimensions** — identifiers that live in the fact table because they don't deserve their own dimension (e.g., `tx_id`, `order_number`).

**What does NOT go in a fact table:** descriptive attributes. If it's a name, a category, a region, an address — it goes in a dimension.

### Additive, semi-additive, non-additive facts

| Type | Can you SUM across all dimensions? | Example |
|---|---|---|
| **Additive** | Yes | Revenue, quantity — `SUM(revenue)` by store, by month, by product all make sense |
| **Semi-additive** | Sum across some dimensions, not all | Account balance — you can sum across accounts but NOT across time (balance on Jan + balance on Feb is meaningless) |
| **Non-additive** | Cannot sum meaningfully | Unit price, ratio, percentage — `AVG` or `MAX` instead |

---

## Dimension tables

A dimension table stores **descriptive context** — the "who, what, where, when, how" of each fact.

### Characteristics

- **Wide** — many columns (20-100+ is normal). A `product` dimension might have: `sku, name, category, subcategory, brand, supplier, cost, price, is_active, launch_date, ...`
- **Short** — relatively few rows compared to facts. Zephyr has ~200 stores, ~80 products, ~50K customers vs ~500K transactions.
- **Denormalized** — in a star schema, the dimension is flat (no foreign keys to other dimensions). `store` has `city, region, state` directly — not a separate `city` table.
- **Slowly changing** — attributes change over time (a store changes type from kiosk to cafe, a customer changes tier). How you handle this is the SCD question — covered in [slowly_changing_dimensions.md](slowly_changing_dimensions.md).

### The date dimension

Almost every star schema has a **date dimension** — a row per calendar day with pre-computed attributes:

```
date_key (int, e.g., 20240315)
full_date (date)
day_of_week (string: "Monday")
is_weekend (boolean)
week_number (int)
month_name (string: "March")
quarter (int: 1)
fiscal_year (int)
is_holiday (boolean)
```

**Why not just use a timestamp?** Because `GROUP BY month_name` or `WHERE is_holiday = true` is much cleaner than extracting from a raw timestamp every time. The date dimension pre-computes these so every query is simpler.

**Interview tip:** if you're designing a schema on a whiteboard, always include a date dimension. Interviewers notice when you skip it.

---

## Star schema vs Snowflake schema

### Star schema

Fact table in the center, dimension tables around it. Dimensions are **denormalized** — flat, no further normalization.

```
                    dim_customer
                         |
dim_product --- fact_transactions --- dim_store
                         |
                    dim_date
```

**Pros:**
- Simple queries — fewer joins (fact + dimension, no chains)
- BI tools (Tableau, Power BI, Looker) work best with star schemas
- Query performance is excellent on columnar engines (Snowflake, BigQuery, Redshift) because each dimension join is a single hop

**Cons:**
- Denormalized dimensions have some data redundancy (every product row stores the `category` name, not a category_id)
- Updates to shared attributes (e.g., renaming a category) must update every product row

### Snowflake schema

Same as star, but dimensions are **normalized** — split into sub-dimensions.

```
dim_category --- dim_product --- fact_transactions --- dim_store --- dim_region
                                        |
                                   dim_date
```

`dim_product` has a `category_id` foreign key → `dim_category` has `category_name, department`.

**Pros:**
- Less redundancy — category name stored once
- Easier to update shared attributes

**Cons:**
- More joins — queries are more complex
- BI tools handle it less naturally
- On modern columnar engines, the storage savings are negligible (compression handles redundancy well)

### When to pick which

**Star schema is almost always the right answer in a DE interview.** Here's why:

- Modern columnar engines (Snowflake, BigQuery, Redshift, Spark) make the storage cost of denormalization trivial
- Query simplicity matters more than storage savings at almost every scale
- BI tools assume star schemas
- Snowflake schemas only win when a dimension is genuinely huge AND highly normalized (rare)

**The interview answer:** *"I'd use a star schema. Snowflake saves storage on redundant dimension attributes, but on a columnar engine the compression already handles that, and star gives us simpler queries + better BI tool compatibility. I'd only snowflake a dimension if it's very large and has a deep hierarchy — like a geographic hierarchy with 7 levels."*

---

## Grain — the concept that separates senior from junior

The grain is **what one row in the fact table represents**. It's the single most important decision in dimensional modeling.

### Zephyr example

Zephyr's `transactions` table — what's the grain?

- **One row per transaction?** Then `tx_id` is unique. Revenue = `SUM(total_amount)`. But you lose product-level detail.
- **One row per transaction line item?** Then `(tx_id, sku)` is the composite key. You can analyze by product. Revenue = `SUM(unit_price * quantity)`.

These are different grains. Both are valid. The choice depends on what business questions you need to answer.

### Why getting it wrong is catastrophic

If your grain is "one row per transaction" but you accidentally have multiple rows per transaction (duplicates, or you joined with line items), then `SUM(total_amount)` double-counts revenue. **Marcus will notice within an hour.**

**The interview move:** when asked to design a schema, the first thing you say is:

> *"Before I draw anything — what's the grain? For this order system, I'd declare the grain as one row per order line item, so we can analyze at the product level. Does that match the business questions we're answering?"*

That question alone signals senior thinking.

### Mixed grain = disaster

Never mix grains in the same fact table. If some rows are "one per order" and others are "one per line item," every aggregate is wrong. Separate them into different fact tables.

---

## Surrogate keys vs natural keys

| | Surrogate key | Natural key |
|---|---|---|
| **What** | System-generated integer (auto-increment or hash) | Business identifier (`customer_id`, `sku`, email) |
| **Example** | `dim_customer.sk = 1, 2, 3, ...` | `dim_customer.customer_id = "CUST_42"` |
| **Why use it** | Stable, compact, handles SCDs (Type 2 creates new rows with new SKs) | Meaningful to the business, easier to debug |
| **When** | Always for dimensions in a warehouse | OK for operational/OLTP databases |

**The interview answer:** *"Dimensions in a warehouse should have surrogate keys. Natural keys can change (a customer's email changes), can be recycled (an old SKU reassigned), and can't handle SCD Type 2 (where the same natural key has multiple historical rows). The surrogate key is the stable join target."*

---

## Normalization (1NF → 3NF) — just enough for interviews

Interviews rarely go beyond 3NF. Know the definitions and one example each.

| Form | Rule | Violation example | Fix |
|---|---|---|---|
| **1NF** | Every column has atomic (single) values; no repeating groups | `phone_numbers = "555-1234, 555-5678"` in one cell | Split into a separate `phones` table |
| **2NF** | 1NF + every non-key column depends on the **entire** primary key (no partial dependencies) | `(order_id, product_id) → product_name` — `product_name` depends only on `product_id`, not the full key | Move `product_name` to a `products` table |
| **3NF** | 2NF + no transitive dependencies (non-key column depends on another non-key column) | `employee → department → department_head` — `department_head` depends on `department`, not directly on `employee` | Move `department_head` to a `departments` table |

**The interview answer when asked "normalized or denormalized?":**

> *"For OLTP (operational systems) — normalized (3NF) to avoid update anomalies. For OLAP (analytics/warehouse) — denormalized (star schema) for query performance and simplicity. In a modern lakehouse, the raw/bronze layer is often normalized or semi-structured, and the gold/mart layer is denormalized star schemas."*

---

## Interview one-liners (memorize these)

- **"Walk me through designing a schema."** → Kimball 4-step: identify the business process, declare the grain, pick the dimensions (who/what/where/when), identify the facts (measures). Start with the grain — everything follows from it.
- **"Star vs snowflake?"** → Star: denormalized dimensions, simpler queries, BI-friendly. Snowflake: normalized dimensions, less redundancy. Star wins on modern columnar engines because compression handles redundancy and fewer joins = faster queries.
- **"What's the grain?"** → What one row in the fact table represents. The most important decision. Wrong grain = wrong aggregates = wrong business decisions.
- **"Surrogate vs natural keys?"** → Surrogate for warehouse dimensions (stable, handles SCDs, compact joins). Natural keys for OLTP or when business users need to query directly.
- **"Three types of fact tables?"** → Transactional (one event), periodic snapshot (state at regular interval), accumulating snapshot (process lifecycle with milestones).
- **"Normalized or denormalized?"** → OLTP = normalized (3NF, avoid update anomalies). OLAP = denormalized (star schema, query speed). Modern lakehouses: raw layer normalized, gold layer denormalized.

---

## Common pitfalls

1. **Skipping the grain.** Starting to draw tables without declaring what one row means. Everything is wrong from this point.
2. **Putting descriptive attributes in the fact table.** `store_name` in the fact table = redundancy + update nightmares. It belongs in `dim_store`.
3. **Using natural keys as the primary key in dimensions.** Breaks when the natural key changes or when you add SCD Type 2 versioning.
4. **Snowflaking unnecessarily.** On modern columnar engines, the storage savings rarely justify the extra joins.
5. **Mixing grains.** Some rows are order-level, some are line-item-level, in the same table. Every aggregate is silently wrong.
6. **Forgetting the date dimension.** Interviewers notice. Always include one.
7. **Over-normalizing for analytics.** 3NF is for OLTP. If you propose 3NF for a warehouse, you've missed the point.

---

## Further reading

- **Kimball Group Design Tips** (free): [kimballgroup.com](https://www.kimballgroup.com/category/design-tips/) — Design Tip #51 (grain), #111 (bridge tables), #152 (SCD types). The primary source.
- **"The Data Warehouse Toolkit" by Ralph Kimball** (3rd ed.) — the definitive book. Not free, but the single best resource on dimensional modeling.
- **"Cracking the Data Modeling Interview" by Sean Coyne** (Medium series) — specifically targeting DE interview modeling questions.

Related in this repo: [slowly_changing_dimensions.md](slowly_changing_dimensions.md), [modeling_approaches.md](modeling_approaches.md).

---

*Last revised: 2026-04*
