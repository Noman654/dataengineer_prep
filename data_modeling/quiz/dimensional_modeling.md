# Dimensional Modeling — Self-Check Quiz

**How to use this:** Read the question. Say your answer out loud. Then click to expand.

**Paired with:** [`../theory/dimensional_modeling.md`](../theory/dimensional_modeling.md)

**Difficulty mix:** 🟢 basics → 🟡 intermediate → ⚡ senior

---

## 🟢 Basics

<details>
<summary><strong>Q1.</strong> What's the difference between a fact table and a dimension table?</summary>

<br>

**Fact table** = what happened. Stores measurable events — transactions, clicks, rides. Contains measures (numeric: revenue, quantity) + foreign keys to dimensions.

**Dimension table** = the context. Stores descriptive attributes — who, what, where, when. Contains names, categories, dates, hierarchies.

Rule: if it's a number you'd SUM or AVG, it's a fact. If it's a label or description, it's a dimension.

</details>

<details>
<summary><strong>Q2.</strong> What are the three types of fact tables?</summary>

<br>

1. **Transactional** — one event at a point in time (one sale, one click). Most common.
2. **Periodic snapshot** — state at a regular interval (monthly revenue by store).
3. **Accumulating snapshot** — lifecycle of a process with milestones (order placed → paid → shipped → delivered).

Default to transactional unless the question implies a lifecycle or regular reporting.

</details>

<details>
<summary><strong>Q3.</strong> Star schema vs snowflake — what's the difference and when do you pick each?</summary>

<br>

**Star:** fact in the center, dimensions around it. Dimensions are denormalized (flat). Fewer joins, simpler queries, BI tools love it.

**Snowflake:** same, but dimensions are normalized into sub-dimensions. Less redundancy, more joins.

**Pick star almost always** for analytics. On modern columnar engines (Snowflake, BigQuery, Redshift), compression handles the redundancy, and fewer joins = faster queries + simpler code. Snowflake schema only when a dimension has a genuinely deep hierarchy AND storage/update cost is a real concern.

</details>

---

## 🟡 Intermediate

<details>
<summary><strong>Q4.</strong> What is the grain and why does it matter?</summary>

<br>

The grain is **what one row in the fact table represents** — one transaction, one line item, one daily snapshot.

**Why it matters:** every aggregate (`SUM`, `COUNT`, `AVG`) assumes a consistent grain. If your grain is "one row per order line item" but some rows are order-level (duplicated across items), `SUM(amount)` double-counts.

**The senior move:** when asked to design a schema, say *"Before I draw anything — what's the grain?"* first. That question alone signals you understand modeling.

</details>

<details>
<summary><strong>Q5.</strong> Surrogate keys vs natural keys — when do you use each?</summary>

<br>

**Surrogate key:** system-generated integer (auto-increment or hash). Stable, compact, handles SCD Type 2 (same natural key, multiple rows).

**Natural key:** business identifier (`customer_id`, `sku`, email). Meaningful to humans, easier to debug.

**Rule:** dimensions in a warehouse → surrogate keys. Natural keys can change (email), can be recycled (old SKU reused), and can't handle Type 2 versioning. The surrogate key is the stable join target.

</details>

<details>
<summary><strong>Q6.</strong> What's a degenerate dimension? Give an example.</summary>

<br>

A dimension attribute that lives in the **fact table** instead of its own dimension table — because it doesn't have enough descriptive attributes to justify a separate table.

**Example:** `order_number` or `tx_id` in a transactional fact table. It's an identifier (dimension-like), but there's nothing else to say about it — no name, no category, no hierarchy. So it stays in the fact table as a degenerate dimension.

</details>

<details>
<summary><strong>Q7.</strong> What's the difference between additive, semi-additive, and non-additive facts?</summary>

<br>

| Type | Can you SUM across all dimensions? | Example |
|---|---|---|
| **Additive** | Yes | Revenue, quantity |
| **Semi-additive** | Some dimensions, not all | Account balance — sum across accounts but NOT across time |
| **Non-additive** | No | Unit price, ratios, percentages — use AVG or MAX instead |

**Why it matters:** if you `SUM(account_balance)` across months, you get a meaningless number. Knowing which facts are semi-additive prevents this.

</details>

---

## ⚡ Senior / interview judgment

<details>
<summary><strong>Q8.</strong> Walk through designing a schema for Zephyr Coffee's orders. What's your process?</summary>

<br>

**Kimball 4-step process:**

**Step 1 — Business process:** order transactions at Zephyr stores.

**Step 2 — Grain:** one row per order line item (`tx_id + sku`). Not one per transaction — we want product-level analysis.

**Step 3 — Dimensions:**
- `dim_customer` (customer_id, name, tier, signup_date, home_store_id)
- `dim_product` (sku, name, category, price, cost)
- `dim_store` (store_id, city, region, store_type, opened_date)
- `dim_date` (date_key, full_date, day_of_week, month, quarter, is_holiday)
- `dim_payment` (payment_method — card, cash, mobile, gift_card)

**Step 4 — Facts:**
- `quantity` (additive)
- `unit_price` (non-additive)
- `discount_amount` (additive)
- `line_total` = quantity * unit_price - discount (additive)

**Things to call out proactively:**
- `customer_id` is nullable (walk-ins) — use a "Unknown Customer" row in `dim_customer` instead of NULL foreign keys
- `tx_id` stays in the fact as a degenerate dimension
- Date dimension includes `is_holiday` for marketing analysis

</details>

<details>
<summary><strong>Q9.</strong> An interviewer asks: "normalized or denormalized?" What's your answer?</summary>

<br>

**Don't answer with one word.** Context matters:

- **OLTP (operational systems)** → normalized (3NF). Avoid update anomalies, enforce data integrity, minimize redundancy.
- **OLAP (analytics/warehouse)** → denormalized (star schema). Query speed, simpler joins, BI tool compatibility.
- **Modern lakehouse** → raw/bronze layer is often semi-structured or normalized. Gold/mart layer is denormalized star schemas.

**The nuance that signals senior:** "The answer depends on the workload. For a production database serving the app — normalized. For an analytics warehouse serving dashboards — denormalized. In a lakehouse, we do both: ingest raw, model into star schemas in the gold layer."

</details>

<details>
<summary><strong>Q10.</strong> Your periodic snapshot fact table shows monthly revenue by store. The March numbers don't match the transactional fact table. What went wrong?</summary>

<br>

**Most likely cause: grain mismatch or timing.**

Check in this order:

1. **Late-arriving facts.** Transactions from March arrived after the snapshot was built. The snapshot shows data as-of the snapshot date, not all data with a March timestamp.

2. **Grain mismatch.** The snapshot sums `total_amount` from the transactional table, but the transactional table's grain is line-item-level while `total_amount` is order-level. Summing it across line items double-counts.

3. **Filtered differently.** The snapshot excludes returns (`total_amount < 0`) or walk-ins (`customer_id IS NULL`) while the transactional table includes them.

4. **SCD timing.** A store changed regions in March. The snapshot uses the current dimension state; the transactional table joined to the historical state. Same store, different attributes, different grouping.

**The senior answer:** "I'd start by comparing the raw row counts and sums between both tables for March, filtered to the same grain and the same dimension version. The mismatch is almost always a grain issue, a late-arriving data issue, or a filter inconsistency — not a bug."

</details>

---

## 🎯 How did you do?

- **9–10 correct** → you can design schemas on a whiteboard. Drill [SCDs](../theory/slowly_changing_dimensions.md) next.
- **6–8 correct** → solid grasp. Re-read the [dimensional modeling doc](../theory/dimensional_modeling.md) sections you missed.
- **< 6 correct** → start with Kimball's 4-step process and the grain concept. Those two unlock everything else.

**Related:** [quiz/slowly_changing_dimensions.md](slowly_changing_dimensions.md) (coming soon)
