# Modeling Approaches — Self-Check Quiz

**How to use this:** Read the question. Say your answer out loud. Then click to expand.

**Paired with:** [`../theory/modeling_approaches.md`](../theory/modeling_approaches.md)

**Difficulty mix:** 🟢 basics → 🟡 intermediate → ⚡ senior

---

## 🟢 Basics

<details>
<summary><strong>Q1.</strong> Kimball vs Inmon in one sentence each.</summary>

<br>

**Kimball:** bottom-up — build denormalized star schemas per business process, deliver value fast.

**Inmon:** top-down — build a normalized (3NF) enterprise data warehouse first, then create denormalized marts from it.

Most modern analytics teams default to Kimball. Inmon when enterprise governance is the priority.

</details>

<details>
<summary><strong>Q2.</strong> What is Data Vault? Name its three table types.</summary>

<br>

Data Vault is a modeling methodology built for auditability and flexibility. Three table types:

1. **Hubs** — business keys (the core identifiers that don't change). One row per entity.
2. **Links** — relationships between hubs (many-to-many by default).
3. **Satellites** — descriptive attributes with full history. Every change creates a new row with `load_date` and `record_source`.

Not directly queryable by analysts — you build star-schema marts on top.

</details>

<details>
<summary><strong>Q3.</strong> What is the One Big Table (OBT) pattern?</summary>

<br>

Pre-join all facts and dimensions into one wide, fully denormalized table. Every query hits one table with zero joins.

**Works well:** flat exports, data science feature tables, BI tools that can't handle joins, downstream of a governed star schema.

**Fails:** when dimensions change (updating a product name means updating millions of rows), when you need DRY governance across multiple consumers.

**Use it downstream, not as your primary model.**

</details>

---

## 🟡 Intermediate

<details>
<summary><strong>Q4.</strong> What's the difference between medallion architecture and dimensional modeling?</summary>

<br>

**Medallion** (bronze → silver → gold) is a **data flow pattern** — how data moves from raw to refined.

**Dimensional modeling** (Kimball, star schemas) is a **data structure pattern** — how the gold layer is organized.

They're complementary, not competing. Medallion tells you the pipeline stages. Kimball/Inmon/Data Vault tells you how to model within the gold stage.

**Interview signal:** calling this out unprompted shows you understand architecture vs modeling — most candidates conflate them.

</details>

<details>
<summary><strong>Q5.</strong> When would you pick Data Vault over Kimball?</summary>

<br>

**Pick Data Vault when:**
- Regulated industry (banking, insurance, healthcare) requiring full audit trails
- Multiple conflicting source systems (Data Vault handles source-of-truth conflicts well)
- Source schemas change frequently
- You have a dedicated data engineering team

**Pick Kimball when:**
- Analytics-first team with clear business questions
- Small-to-mid team that needs fast time-to-value
- BI-heavy environment (dashboards, reports)
- You don't need full auditability at the raw layer

**The tradeoff:** Data Vault trades simplicity for flexibility and auditability. Kimball trades flexibility for speed and simplicity.

</details>

<details>
<summary><strong>Q6.</strong> Why is OBT becoming more popular in 2025-2026, and what's the catch?</summary>

<br>

**Why now:** modern columnar engines (BigQuery, Snowflake, Databricks) make wide tables cheap — column pruning means you only read the columns you reference, so width doesn't hurt performance. Storage is cheap. Compression handles redundancy.

**The catch:** OBT is easy to build, hard to maintain.
- When a dimension changes (customer name, product category), you have to update every row in the OBT.
- No DRY — if two teams need customer data, they each maintain their own join logic.
- No governed "single source of truth" dimension.

**The consensus:** use OBT downstream of a star schema, not instead of one. Star schema = governed foundation. OBT = materialized convenience for specific consumers.

</details>

---

## ⚡ Senior / interview judgment

<details>
<summary><strong>Q7.</strong> You're building a data warehouse for Zephyr Coffee from scratch. Which approach do you pick and why?</summary>

<br>

**Answer: Kimball, with medallion architecture for the pipeline.**

**Why:**
- Zephyr has clear business processes: orders, loyalty, store ops → each maps to a star schema
- Small data team (you're the first DE) → Kimball's bottom-up approach lets you deliver value in weeks, not months
- BI consumers (Jen, Marcus, Dev) need dashboards → star schemas work natively with their tools
- Not a regulated industry → Data Vault's auditability overhead isn't justified
- Inmon requires a larger team and more upfront planning than Zephyr can afford

**Pipeline:** Bronze (raw POS dumps, loyalty JSON) → Silver (cleaned, typed, deduped) → Gold (star schemas per business process: orders star, loyalty star, store ops star).

**Key design decisions:**
- Conformed `dim_customer`, `dim_store`, `dim_product` shared across stars
- SCD Type 2 on `dim_customer.tier` and `dim_store.store_type`
- Transactional fact for orders, periodic snapshot for monthly store performance

**Proactive callout:** "If Zephyr scales to 1000+ stores with regulatory requirements, I'd re-evaluate Data Vault for the raw layer. But at 200 stores with a small team, Kimball is the right tradeoff."

</details>

<details>
<summary><strong>Q8.</strong> A company uses Data Vault in their raw layer but analysts complain queries are too complex. What do you do?</summary>

<br>

**Build Kimball-style star schema marts on top of the Data Vault.**

The Data Vault stays as the governed, auditable raw-to-clean layer. On top of it, materialize star schemas for specific business processes — these are what analysts and BI tools query.

```
Sources → Data Vault (hubs/links/satellites) → Star Schema Marts → BI Tools / Analysts
```

This is the standard Data Vault deployment pattern — Data Vault was never designed for direct analyst access. If analysts are querying hubs and satellites directly, the implementation is incomplete, not wrong.

**The fix is a mart layer, not abandoning Data Vault.**

</details>

<details>
<summary><strong>Q9.</strong> Your team uses Kimball star schemas. A data scientist says "just give me one big table with everything joined." How do you respond?</summary>

<br>

**Don't say no. Don't say "star schemas are better." Build them an OBT — downstream of the star schema.**

```sql
CREATE TABLE obt_orders AS
SELECT
  f.order_id, f.amount, f.quantity,
  d_c.customer_id, d_c.name AS customer_name, d_c.tier,
  d_p.product_id, d_p.name AS product_name, d_p.category,
  d_s.store_id, d_s.store_name, d_s.store_type,
  d_d.date, d_d.year, d_d.month
FROM fact_orders f
JOIN dim_customer d_c ON f.customer_sk = d_c.customer_sk
JOIN dim_product d_p ON f.product_sk = d_p.product_sk
JOIN dim_store d_s ON f.store_sk = d_s.store_sk
JOIN dim_date d_d ON f.date_key = d_d.date_key;
```

Materialize it as a table or view, refresh on a schedule.

**Watch the join, not just the columns:** join on the surrogate key (`customer_sk`) and don't add `WHERE d_c.is_current = true` — the fact's surrogate key already points at the dimension version that was current *when the order happened*, so that row already has the right point-in-time attributes. Filtering to `is_current = true` on top would silently drop every historical order whose customer/store/product has since changed, which is the opposite of "current state for simplicity." (If the ask were instead "each order's *current* customer attributes," you'd join via the natural key with an `is_current = true` filter — a different, narrower request.) Also drop the `f.*, d_c.*, ...` wildcard select — most engines reject `CREATE TABLE AS SELECT` with duplicate column names across the joined tables.

**Why this works:**
- The data scientist gets their flat table, zero joins
- The star schema remains the governed source of truth
- Dimension changes propagate through the refresh
- You maintain ONE set of dimension logic, not two

**The wrong answer:** building the OBT directly from raw sources, bypassing the star schema. That creates a second ungoverned copy.

</details>

<details>
<summary><strong>Q10.</strong> "Which modeling approach is best?" — how do you answer this in an interview without sounding like you're dodging?</summary>

<br>

**"It depends on three things: team size, governance needs, and time-to-value requirements."**

Then give concrete mappings:

- **Startup with 2 DEs, needs dashboards next month:** Kimball. Ship one star schema this sprint.
- **Bank with 50 source systems and regulatory audits:** Data Vault for the raw layer, Kimball marts on top.
- **Enterprise with 10 departments sharing the same customer data:** Inmon — normalized DW prevents "50 definitions of revenue."
- **Data science team that just needs a flat feature table:** OBT downstream of whatever governs the dimensions.

**Close with:** "I'd never pick an approach without understanding the team, the consumers, and the regulatory context first."

That's not dodging — it's engineering judgment. Which is what they're testing.

</details>

---

## 🎯 How did you do?

- **9–10 correct** → you can reason about modeling tradeoffs at a senior level.
- **6–8 correct** → solid. Re-read the [modeling approaches doc](../theory/modeling_approaches.md), especially the comparison table.
- **< 6 correct** → focus on Kimball first (it's the default). Learn the others after.

**Related:** [quiz/dimensional_modeling.md](dimensional_modeling.md), [quiz/slowly_changing_dimensions.md](slowly_changing_dimensions.md)
