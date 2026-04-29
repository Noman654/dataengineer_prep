# Modeling Approaches — Kimball vs Inmon vs Data Vault vs OBT

**Audience:** DE interview prep. "Which modeling approach would you use?" is a senior-level question that tests whether you understand the tradeoffs, not just the definitions.

**Read time:** 10–15 minutes.

---

## TL;DR — the one-sentence mental model

> **Kimball** = bottom-up, denormalized star schemas per business process. **Inmon** = top-down, normalized enterprise data warehouse then denormalized marts. **Data Vault** = hub-link-satellite pattern for auditability and agility. **OBT** = one wide denormalized table, modern shortcut that works in specific cases. **None of them is "best" — they solve different problems.**

---

## Kimball (Dimensional Modeling)

**Philosophy:** build the warehouse bottom-up. Pick one business process at a time, model it as a star schema, deliver value fast. The warehouse is the collection of conformed star schemas.

**How it works:**
1. Pick a business process (orders, rides, logins)
2. Declare the grain
3. Build fact + dimension tables (star schema)
4. **Conformed dimensions** ensure consistency across stars — the same `dim_customer` is used by the orders star and the loyalty star

**Strengths:**
- Fast time-to-value — ship one star schema, it's immediately queryable
- BI tools (Tableau, Power BI, Looker) are built for star schemas
- Well-understood by analysts — intuitive queries, few joins
- Works naturally with dbt (staging → intermediate → mart pattern)

**Weaknesses:**
- Requires upfront modeling decisions (grain, dimensions) — hard to change later
- Denormalized dimensions can drift if not governed (same customer, different attributes in two stars)
- SCD Type 2 on large dimensions gets expensive

**When to pick Kimball:**
- Analytics-first teams with clear business questions
- BI-heavy environments (dashboards, reports, ad-hoc queries)
- Small-to-mid teams without a dedicated data modeling function
- Most DE interviews default to Kimball — it's the industry standard for OLAP

---

## Inmon (Enterprise Data Warehouse)

**Philosophy:** build the warehouse top-down. First, create a single normalized (3NF) enterprise data warehouse as the "single source of truth." Then build denormalized data marts from it for specific departments.

**How it works:**
```
Source systems → ETL → Enterprise DW (3NF, normalized) → Data Marts (star schemas per department)
```

The enterprise DW stores everything in 3NF. Department-specific marts are built as views or materialized tables — these are where analysts query.

**Strengths:**
- Single source of truth — one canonical version of every entity
- Less data redundancy — normalized storage
- Handles complex, cross-functional analysis well
- Easier to enforce enterprise-wide data governance

**Weaknesses:**
- Slow time-to-value — you build the entire normalized DW before any department gets a mart
- More complex ETL (source → 3NF → mart = two transformation layers)
- Requires strong data governance and a dedicated modeling team
- Analysts don't query 3NF directly — they need marts built for them

**When to pick Inmon:**
- Large enterprises with multiple departments sharing the same entities
- Regulatory environments requiring strict audit trails
- Organizations with dedicated data modeling teams
- When cross-functional consistency matters more than speed

---

## Kimball vs Inmon — the interview answer

| Dimension | Kimball | Inmon |
|---|---|---|
| **Approach** | Bottom-up (star schemas) | Top-down (normalized DW → marts) |
| **Storage** | Denormalized (some redundancy) | Normalized 3NF (less redundancy) |
| **Time to value** | Fast (ship one star) | Slow (build DW first) |
| **Query complexity** | Simple (few joins) | Complex in DW, simple in marts |
| **Best for** | Analytics, BI, dashboards | Enterprise governance, cross-functional |
| **Team size** | Works with small teams | Needs dedicated modeling team |
| **dbt fit** | Natural (staging → mart) | Possible but more layers |

**The interview answer:** *"Kimball for most analytics use cases — faster to deliver, BI-friendly, works with dbt. Inmon when you need enterprise-wide consistency across many departments and have the team to maintain it. In practice, most modern data teams start with Kimball and only go Inmon if governance demands it."*

---

## Data Vault 2.0

**Philosophy:** separate business keys, relationships, and descriptive attributes into distinct table types. Optimized for auditability, flexibility, and handling change — not for direct querying.

**How it works — three table types:**

### Hubs
Store **business keys** — the core identifiers that don't change.

```
hub_customer:  hub_customer_sk, customer_id (business key), load_date, record_source
```

### Links
Store **relationships** between hubs — many-to-many by default.

```
link_order:  link_order_sk, hub_customer_sk, hub_product_sk, hub_store_sk, load_date, record_source
```

### Satellites
Store **descriptive attributes** with full history (every change creates a new row).

```
sat_customer_details:  hub_customer_sk, name, email, tier, load_date, record_source
sat_customer_demographics:  hub_customer_sk, age_band, income_band, load_date, record_source
```

**Strengths:**
- Full auditability — every change is tracked with load_date and record_source
- Highly flexible — adding new sources or attributes means adding satellites, not restructuring
- Handles schema evolution gracefully
- Parallel loading — hubs, links, and satellites can be loaded independently
- Works well for regulated industries (banking, insurance, healthcare)

**Weaknesses:**
- Not directly queryable by analysts — you build star-schema "business vaults" or marts on top
- More tables = more complex architecture
- Steeper learning curve for the team
- Overkill for small-to-mid analytics teams

**When to pick Data Vault:**
- Highly regulated industries requiring full audit trails
- Multiple source systems with conflicting data (Data Vault handles this well)
- Rapidly changing source schemas
- Large teams with dedicated data engineers

**When NOT to pick:** small teams, pure analytics use cases, when time-to-value matters most. In these cases, Kimball is simpler and sufficient.

---

## One Big Table (OBT)

**Philosophy:** skip the star schema entirely. Join everything into one wide, fully denormalized table. Every query hits one table with zero joins.

**Zephyr example:**
```
obt_transactions:
  tx_id, ts, total_amount, quantity, unit_price,
  customer_id, customer_name, customer_tier, customer_signup_date,
  store_id, store_name, store_city, store_region, store_type,
  sku, product_name, product_category, product_price,
  payment_method, is_holiday, day_of_week, month, quarter
```

One table. 25+ columns. Every dimension pre-joined.

**Strengths:**
- Zero joins — simplest possible queries (`SELECT ... FROM obt WHERE ...`)
- Works well with BI tools that struggle with relationships
- Excellent for flat exports (CSV dumps for finance, Google Sheets)
- On columnar engines (BigQuery, Snowflake), column pruning keeps it fast despite the width
- Great for data science teams who just want a flat feature table

**Weaknesses:**
- SCD handling is painful — when a customer's tier changes, do you update every historical row?
- Redundancy — customer name is stored in every transaction row
- Updates are expensive — changing a product name means updating millions of fact rows
- No DRY reuse — if two OBTs need customer data, you're maintaining the same join logic twice
- Doesn't scale governance-wise — no single authoritative dimension to govern

**When to pick OBT:**
- Downstream of a star schema — as a materialized mart for a specific use case
- Data science feature tables
- Small datasets where maintenance cost is low
- When your BI tool can't handle joins (some tools genuinely can't)

**When NOT to pick:** as your primary warehouse model. Dimensions change. Updates become nightmares. Use it downstream, not upstream.

**The 2025-2026 consensus:** hybrid approach. Kimball star schema as the governed foundation, OBTs as materialized views downstream for specific consumers who need flat tables.

---

## Medallion Architecture (Bronze / Silver / Gold)

**Important:** medallion is a **data architecture pattern**, not a data model. But interviewers conflate them, so know the difference.

```
Bronze (raw)  → Silver (cleaned, conformed)  → Gold (business-ready, modeled)
```

| Layer | What it contains | Modeling |
|---|---|---|
| **Bronze** | Raw ingested data, as-is from source. JSON, CSV, nested. | No modeling — schema-on-read |
| **Silver** | Cleaned, deduplicated, type-cast, conformed. | Lightly normalized or event-level |
| **Gold** | Business-ready. Star schemas, OBTs, aggregated marts. | This is where Kimball / OBT lives |

**The interview answer:** *"Medallion is how data flows through the warehouse (raw → clean → modeled). Kimball/Inmon/Data Vault is how you model the gold layer. They're complementary, not competing."*

---

## Quick comparison table

| | Kimball | Inmon | Data Vault | OBT |
|---|---|---|---|---|
| **Structure** | Star schemas | 3NF DW → star marts | Hub/Link/Satellite | One wide table |
| **Approach** | Bottom-up | Top-down | Parallel loading | Pre-join everything |
| **Time to value** | Fast | Slow | Medium | Fastest (but technical debt) |
| **Query complexity** | Low (few joins) | Low in marts | High (needs mart layer) | Lowest (zero joins) |
| **Handles change** | SCD on dimensions | SCD on 3NF tables | Satellites track all history | Painful (update every row) |
| **Best for** | Analytics, BI | Enterprise governance | Regulated, multi-source | Flat exports, data science |
| **Team size** | Small–medium | Large | Medium–large | Any |
| **dbt fit** | Natural | Possible | Growing (dbt-vault package) | Natural |

---

## Interview one-liners (memorize these)

- **"Kimball vs Inmon?"** → Kimball is bottom-up (star schemas per business process, fast to deliver, BI-friendly). Inmon is top-down (normalized 3NF enterprise DW first, marts later, better governance). Most modern teams start Kimball. Inmon when governance is non-negotiable.
- **"What's Data Vault?"** → Three table types: hubs (business keys), links (relationships), satellites (attributes with history). Built for auditability and flexibility. Not directly queryable — you build star marts on top. Common in banking, insurance.
- **"What about One Big Table?"** → Pre-join everything into one wide table. Zero joins, simplest queries. Works great downstream of a star schema for specific consumers. Fails as a primary model because dimension changes require updating millions of fact rows.
- **"What's medallion architecture?"** → Bronze (raw) → Silver (cleaned) → Gold (modeled). It's a data flow pattern, not a modeling methodology. The gold layer is where Kimball / OBT / Data Vault actually lives.
- **"Which would you pick?"** → Depends on the use case. For a startup analytics warehouse: Kimball. For a regulated enterprise with 50 source systems: Data Vault → Kimball marts. For a flat export to a data science team: OBT downstream of a star schema. Never pick one without asking about the context first.

---

## Common pitfalls

1. **Saying "Kimball" or "Inmon" without justification.** Interviewers want the tradeoff reasoning, not the buzzword.
2. **Treating OBT as a replacement for dimensional modeling.** It's a downstream convenience, not a foundation. Without a governed star schema upstream, OBT maintenance becomes unsustainable.
3. **Confusing medallion with modeling.** Medallion is architecture (how data flows). Kimball/Inmon/Data Vault is modeling (how data is structured). They coexist.
4. **Over-engineering with Data Vault for a small team.** If you have 3 data engineers and one source system, Data Vault is overkill. Kimball gets you value in a week.
5. **Dismissing Inmon entirely.** In large enterprises with strict governance requirements, Inmon's normalized DW prevents the "50 different definitions of revenue" problem that Kimball's conformed dimensions are supposed to solve but often don't in practice.

---

## Further reading

- **"The Data Modeling Wars: Inmon vs Kimball vs Data Vault" by Chengzhi Zhao** (Medium) — concise comparison with diagrams.
- **"Our Hybrid Kimball and OBT Approach" — Brooklyn Data (2025)** — the modern consensus on when to use OBT alongside star schemas.
- **Kimball Group Design Tips** — [kimballgroup.com](https://www.kimballgroup.com/category/design-tips/)
- **Data Vault 2.0 overview** — Dan Linstedt's original methodology. The [dbt-vault package](https://dbtvault.readthedocs.io/) is the modern implementation.

Related in this repo: [dimensional_modeling.md](dimensional_modeling.md), [slowly_changing_dimensions.md](slowly_changing_dimensions.md).

---

*Last revised: 2026-04*
