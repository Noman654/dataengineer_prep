# Slowly Changing Dimensions (SCDs) — Quick Review

**Audience:** DE interview prep. SCDs are one of the most frequently asked modeling topics — knowing only Type 1 and Type 2 signals junior. Knowing Types 0 through 6 signals senior.

**Read time:** 10–15 minutes.

---

## TL;DR — the one-sentence mental model

> A dimension attribute changes over time (a customer upgrades tier, a store changes type). **How you handle that change in the warehouse determines whether downstream queries see history or just current state.** Each SCD type is a different tradeoff between simplicity, storage, and historical accuracy.

---

## Why interviewers love this topic

Because it forces you to ask the right question before picking a solution:

> *"How often does this attribute change? Do downstream consumers need point-in-time accuracy, or is current state sufficient?"*

Juniors say "Type 2" reflexively. Seniors ask the question, then pick the type with justification. That's the signal.

---

## The SCD Types

### Type 0 — Retain Original

**Rule:** never update. The original value is preserved forever.

**Zephyr example:** `customer.signup_date` — the date a customer first registered. It never changes, even if they re-register.

**When to use:** attributes that are definitionally immutable — birth dates, original signup, first-purchase date.

**Implementation:** just don't update the column. Ever.

---

### Type 1 — Overwrite

**Rule:** update in place. Old value is lost.

```
BEFORE:  store_id=42, name="Zephr Portland"    ← typo
AFTER:   store_id=42, name="Zephyr Portland"   ← fixed, old value gone
```

**Zephyr example:** fixing a misspelled store name. The history of the typo has no business value.

**When to use:**
- Correcting data errors (typos, wrong codes)
- Attributes where history is irrelevant (email format normalization)
- When simplicity matters more than audit trail

**Tradeoff:** simple, but you lose all history. If anyone ran a report last week with the old value, the numbers won't reproduce.

**Implementation:** plain `UPDATE` statement.

---

### Type 2 — Add New Row (Versioned History)

**Rule:** insert a new row with a new surrogate key. Mark the old row as expired.

```
sk  | store_id | store_type | effective_date | expiry_date  | is_current
101 | 42       | kiosk      | 2019-01-01     | 2023-06-30   | false
102 | 42       | cafe       | 2023-07-01     | 9999-12-31   | true
```

**Zephyr example:** store 42 converts from kiosk to cafe. Old transactions still join to `sk=101` (kiosk), new transactions join to `sk=102` (cafe). Revenue reports by store type are historically accurate.

**When to use:**
- Attributes where history matters for analytics (customer tier, store type, product category)
- When downstream queries need point-in-time accuracy ("revenue by tier *at time of purchase*")

**Tradeoff:** more storage (multiple rows per entity), more complex queries (`WHERE is_current = true` for current state, join on surrogate key for historical).

**Implementation columns:**
- `surrogate_key` — auto-increment, unique per version
- `effective_date` / `expiry_date` — version validity window
- `is_current` — boolean flag for quick "current state" queries
- `natural_key` — the business identifier (stays the same across versions)

**The 9999-12-31 convention:** the current row's `expiry_date` is set to a far-future date. When a new version arrives, the current row's `expiry_date` is set to `new_version_effective_date - 1` and `is_current` flipped to false.

---

### Type 3 — Add Previous-Value Column

**Rule:** add a column to store the previous value alongside the current value.

```
store_id | current_region | previous_region | region_change_date
42       | Central        | West            | 2024-01-15
```

**Zephyr example:** Zephyr reorganizes its regions — store 42 moves from West to Central. You keep both values so reports can compare "before reorg" vs "after reorg."

**When to use:**
- One-time organizational changes (region reorg, department restructure)
- When you only need to look back one version (not full history)

**Tradeoff:** only preserves **one** historical value. If the region changes again, you lose the original. For full history, use Type 2.

**Implementation:** add `previous_[attribute]` and `[attribute]_change_date` columns.

---

### Type 4 — Mini-Dimension

**Rule:** split rapidly changing attributes into a separate small dimension table.

**Zephyr example:** customer demographics — `age_band`, `income_band`, `distance_from_store`. These change frequently (birthdays, income updates). Instead of creating a new Type 2 row every time any demographic changes, pull these into a `dim_customer_demographics` mini-dimension.

```
dim_customer:  customer_sk, customer_id, name, signup_date       ← stable
dim_customer_demo: demo_sk, age_band, income_band, distance_band  ← volatile

fact_transactions: tx_id, customer_sk, demo_sk, store_sk, ...
```

**When to use:** a dimension has some stable attributes (name, signup date) and some volatile attributes (age band, income bracket) that change at very different rates. Type 2 on the full dimension would create too many rows.

**Tradeoff:** extra join in queries, but avoids dimension table bloat.

---

### Type 6 — Hybrid (1 + 2 + 3)

**Rule:** combine Type 2 (new row) + Type 1 (overwrite current) + Type 3 (keep previous). Called Type 6 because 1+2+3 = 6.

```
sk  | store_id | store_type | previous_type | effective_date | expiry_date | is_current
101 | 42       | cafe       | kiosk         | 2019-01-01     | 2023-06-30  | false
102 | 42       | cafe       | kiosk         | 2023-07-01     | 9999-12-31  | true
```

**Notice:** both rows show `store_type = cafe` (Type 1 overwrite on all rows), but `previous_type = kiosk` (Type 3 column), and there are two rows (Type 2 versioning).

**Zephyr example:** store 42 converts from kiosk to cafe. You want: (a) full version history (Type 2), (b) easy "current type" filtering without `is_current` (Type 1 updates old rows), (c) quick comparison of previous vs current (Type 3 column).

**When to use:** when you need all three: historical versions, easy current-state access, and before/after comparison. Most common in enterprise warehouses.

**Tradeoff:** most complex to implement and maintain. Only use when the business genuinely needs all three access patterns.

---

## Quick comparison table

| Type | History preserved? | Storage impact | Query complexity | When to use |
|---|---|---|---|---|
| **0** | N/A (immutable) | None | Simplest | Birth dates, signup dates |
| **1** | No — lost | None | Simple | Error corrections, irrelevant history |
| **2** | Full history | High (N rows per entity) | Medium (`is_current`, surrogate key joins) | Tier changes, status changes, anything needing point-in-time |
| **3** | One previous value | Low (one extra column) | Simple | One-time reorgs, before/after comparisons |
| **4** | Via mini-dimension | Medium | Extra join | Volatile demographics alongside stable attributes |
| **6** | Full + current + previous | Highest | Most complex | Enterprise warehouses needing all access patterns |

---

## The interview flow for SCD questions

When an interviewer asks *"How would you handle [changing attribute]?"*:

**Step 1 — Ask questions:**
- How often does this change? (Daily? Yearly? Once?)
- Do consumers need point-in-time accuracy or just current state?
- How many attributes change at different rates?

**Step 2 — Pick with justification:**
- Changes rarely + history matters → **Type 2**
- Error correction / history irrelevant → **Type 1**
- One-time reorg → **Type 3**
- Some attributes volatile, others stable → **Type 4** (mini-dimension)
- Need all access patterns → **Type 6** (rare, acknowledge complexity)

**Step 3 — Mention the tradeoff:**
- "Type 2 gives us full history but increases dimension size and query complexity."
- "Type 1 is simpler but we lose the ability to reproduce last month's report."

---

## SCD Type 2 implementation in SQL

This comes up in coding rounds. Know the pattern:

```sql
-- Step 1: Expire the current row
UPDATE dim_store
SET expiry_date = CURRENT_DATE - 1,
    is_current = false
WHERE store_id = 42
  AND is_current = true;

-- Step 2: Insert the new version
INSERT INTO dim_store (store_id, store_name, store_type, effective_date, expiry_date, is_current)
VALUES (42, 'Zephyr Portland', 'cafe', CURRENT_DATE, '9999-12-31', true);
```

In Delta Lake / Iceberg, this is often done via `MERGE`:

```sql
MERGE INTO dim_store AS target
USING staging_store AS source
ON target.store_id = source.store_id AND target.is_current = true
WHEN MATCHED AND target.store_type != source.store_type THEN
  UPDATE SET target.expiry_date = CURRENT_DATE - 1, target.is_current = false;

-- Then insert new rows for changed records
INSERT INTO dim_store
SELECT store_id, store_name, store_type, CURRENT_DATE, '9999-12-31', true
FROM staging_store s
WHERE EXISTS (
  SELECT 1 FROM dim_store d 
  WHERE d.store_id = s.store_id 
  AND d.is_current = false 
  AND d.expiry_date = CURRENT_DATE - 1
);
```

---

## Interview one-liners (memorize these)

- **"What's an SCD?"** → A dimension attribute that changes over time. The SCD type determines whether you overwrite (lose history) or version (keep history). The choice depends on whether downstream consumers need point-in-time accuracy.
- **"Type 1 vs Type 2?"** → Type 1 overwrites in place — simple, no history. Type 2 adds a new row with effective/expiry dates — full history, more complex queries. Pick based on whether history matters.
- **"What's Type 6?"** → Hybrid of 1+2+3 (hence "6"). New row for history (Type 2), overwrite current value on old rows (Type 1), keep previous-value column (Type 3). Most complex, used when you need all three access patterns.
- **"How do you implement Type 2?"** → Expire the current row (set `expiry_date`, flip `is_current`), insert a new row with the updated attribute and a new surrogate key. In Delta/Iceberg, use `MERGE` for atomic expire+insert.

---

## Common pitfalls

1. **Defaulting to Type 2 for everything.** Type 2 on a 50M-row customer dimension with 10 changing attributes creates billions of rows. Ask which attributes actually need history.
2. **Forgetting `is_current`.** Without it, every "current state" query needs `WHERE expiry_date = '9999-12-31'` — fragile and easy to forget.
3. **Using natural keys as join keys in Type 2.** After Type 2, the same `customer_id` has multiple rows. Facts must join on the surrogate key, not the natural key.
4. **Not handling the "no change" case.** If an incoming record matches the current row exactly, don't create a new version. Compare before inserting.
5. **Mixing Type 1 and Type 2 on the same attribute.** Pick one strategy per attribute, not per mood.

---

## Further reading

- **Kimball Group Design Tip #152** — [SCD Types 0, 4, 5, 6, 7](https://www.kimballgroup.com/2013/02/design-tip-152-slowly-changing-dimension-types-0-4-5-6-7/) — the definitive reference for the extended types.
- **"Cracking the Data Modeling Interview, Part 3: SCDs" by Sean Coyne** (Medium) — interview-focused walkthrough.
- **"The Data Warehouse Toolkit" by Ralph Kimball** — Chapter 5 covers SCDs in full detail.

Related in this repo: [dimensional_modeling.md](dimensional_modeling.md), [modeling_approaches.md](modeling_approaches.md).

---

*Last revised: 2026-04*
