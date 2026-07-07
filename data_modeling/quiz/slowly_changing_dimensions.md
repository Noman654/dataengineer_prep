# Slowly Changing Dimensions — Self-Check Quiz

**How to use this:** Read the question. Say your answer out loud. Then click to expand.

**Paired with:** [`../theory/slowly_changing_dimensions.md`](../theory/slowly_changing_dimensions.md)

**Difficulty mix:** 🟢 basics → 🟡 intermediate → ⚡ senior

---

## 🟢 Basics

<details>
<summary><strong>Q1.</strong> What is a slowly changing dimension?</summary>

<br>

A dimension attribute that changes over time — a customer upgrades tier, a store changes type, a product gets recategorized. "Slowly" means it changes occasionally (not every transaction), but when it does, you need a strategy for how the warehouse handles it.

The SCD type you choose determines whether you keep history or overwrite it.

</details>

<details>
<summary><strong>Q2.</strong> Explain SCD Type 1 and Type 2 in one sentence each.</summary>

<br>

**Type 1:** overwrite the old value. Simple, but history is lost.

**Type 2:** insert a new row with a new surrogate key, mark the old row as expired. Full history preserved, but more storage and complex queries.

</details>

<details>
<summary><strong>Q3.</strong> Zephyr's store 42 had a typo — "Zephr Portland" instead of "Zephyr Portland." Which SCD type do you use to fix it?</summary>

<br>

**Type 1 — overwrite.** The typo has no business value as history. Just fix it.

If you used Type 2 for this, you'd create a new version row for a spelling correction — unnecessary complexity, and downstream queries would see two versions of the same store for no reason.

</details>

---

## 🟡 Intermediate

<details>
<summary><strong>Q4.</strong> A Zephyr customer upgrades from Bronze to Gold tier. Which SCD type, and what columns does the dimension table need?</summary>

<br>

**Type 2** — tier changes matter for analytics ("revenue by tier at time of purchase").

Required columns:
- `customer_sk` — surrogate key (unique per version)
- `customer_id` — natural key (same across versions)
- `tier` — the changing attribute
- `effective_date` — when this version became active
- `expiry_date` — when it was superseded (`9999-12-31` for current)
- `is_current` — boolean for quick current-state filtering

After the change, the old row has `is_current=false, expiry_date=yesterday`. The new row has `is_current=true, expiry_date=9999-12-31`.

</details>

<details>
<summary><strong>Q5.</strong> What's SCD Type 3 and when would you use it instead of Type 2?</summary>

<br>

**Type 3:** add a `previous_value` column alongside the current value. Only stores one level of history.

```
store_id | current_region | previous_region | region_change_date
42       | Central        | West            | 2024-01-15
```

**Use instead of Type 2 when:**
- The change is a one-time event (organizational restructure)
- You only need "before vs after" comparison, not full version history
- If it changes again, you'd lose the original value (only one `previous_` column)

If the attribute changes more than once, Type 2 is better.

</details>

<details>
<summary><strong>Q6.</strong> What's SCD Type 6 and why is it called that?</summary>

<br>

**Type 6 = hybrid of Type 1 + Type 2 + Type 3.** Called "6" because 1+2+3=6.

It combines:
- **Type 2:** new row for each version (full history)
- **Type 1:** overwrite the `current_value` column on ALL rows (easy current-state access)
- **Type 3:** add a `previous_value` column (quick before/after comparison)

**When to use:** enterprise warehouses where consumers need all three access patterns. Most complex to maintain — only justify if genuinely needed.

</details>

<details>
<summary><strong>Q7.</strong> Write the SQL to implement a Type 2 change: store 42 changes from "kiosk" to "cafe" today.</summary>

<br>

```sql
-- Step 1: Expire the current row
UPDATE dim_store
SET expiry_date = CURRENT_DATE - 1,
    is_current = false
WHERE store_id = 42
  AND is_current = true;

-- Step 2: Insert the new version
INSERT INTO dim_store 
  (store_id, store_name, store_type, effective_date, expiry_date, is_current)
VALUES 
  (42, 'Zephyr Portland', 'cafe', CURRENT_DATE, '9999-12-31', true);
```

Key points:
- The surrogate key auto-increments — the new row gets a new `sk`
- Old row keeps its `sk` — existing fact rows still join to it correctly
- `9999-12-31` is the convention for "currently active"

</details>

---

## ⚡ Senior / interview judgment

<details>
<summary><strong>Q8.</strong> An interviewer asks: "How would you handle slowly changing dimensions for Zephyr's customer table?" What's your process?</summary>

<br>

**Don't jump to "Type 2."** Ask questions first:

1. **Which attributes change?**
   - `name` — rarely, usually corrections → **Type 1**
   - `email` — occasionally → **Type 1** (old email has no analytics value)
   - `tier` (bronze/silver/gold) — changes matter for "revenue by tier at purchase time" → **Type 2**
   - `home_store_id` — might change → depends on whether analysts need historical home-store

2. **How often?** If tier changes daily (unlikely but ask), Type 2 creates too many rows. Consider Type 4 (mini-dimension) for volatile attributes.

3. **Do consumers need point-in-time accuracy?** If yes → Type 2 for that attribute. If no → Type 1 is simpler.

**The answer:** "I'd use Type 1 for name and email corrections, Type 2 for tier (analytics need point-in-time), and I'd ask the business about home_store before deciding."

**Why this is senior:** you didn't pick one type for the whole table. You picked per attribute based on business need. That's the signal.

</details>

<details>
<summary><strong>Q9.</strong> Your Type 2 dim_customer has 50M rows because customers change tier frequently. Queries are slow. How do you fix it?</summary>

<br>

**Options, in order:**

1. **Partition the dimension by `is_current`.** Most queries only need current state. Partitioning by `is_current` lets the engine skip all historical rows for current-state queries.

2. **Create a current-only view.** `CREATE VIEW dim_customer_current AS SELECT * FROM dim_customer WHERE is_current = true`. Analysts use the view, historical queries go to the base table.

3. **Consider Type 4 (mini-dimension)** for the volatile attribute. If `tier` is the only thing creating versions, split it into `dim_customer_tier` with its own surrogate key. The base `dim_customer` stays small.

4. **Archive old versions.** If history older than N years isn't queried, move expired rows to a cold-storage table. Keep the main dimension lean.

**Don't do:** drop Type 2 and go to Type 1. You'd lose the historical analysis that justified Type 2 in the first place. Fix the performance, don't remove the feature.

</details>

<details>
<summary><strong>Q10.</strong> You're using Delta Lake / Iceberg. How does MERGE change your SCD Type 2 implementation?</summary>

<br>

In traditional SQL, Type 2 is a two-step process (UPDATE to expire + INSERT new row). `MERGE` can collapse it into one atomic statement — but there is a trap that makes this a great interview question.

**The trap:** a naive MERGE with `ON target.store_id = source.store_id AND target.is_current = true` cannot implement Type 2 on its own. For a changed entity, the source row *matches* the current target row, so only the `WHEN MATCHED` expire fires — `WHEN NOT MATCHED` can never fire for that same source row, and the new version is silently never inserted. (Brand-new entities are the one case that works: they have no current row, so they fall through to the insert.)

**The standard fix** is to make changed entities appear *twice* in the source — once under their real key (matches, expires the current row) and once under a `NULL` merge key (matches nothing, inserts the new version):

```sql
MERGE INTO dim_store AS target
USING (
  -- every staging row once, under its real key
  SELECT s.store_id AS merge_key, s.* FROM staging s
  UNION ALL
  -- changed entities a second time, under a NULL key
  SELECT NULL AS merge_key, s.*
  FROM staging s
  JOIN dim_store t
    ON s.store_id = t.store_id AND t.is_current = true
  WHERE s.store_type != t.store_type
) AS source
ON target.store_id = source.merge_key AND target.is_current = true

WHEN MATCHED AND target.store_type != source.store_type THEN
  UPDATE SET
    target.expiry_date = CURRENT_DATE - 1,
    target.is_current = false

WHEN NOT MATCHED THEN
  INSERT (store_id, store_type, effective_date, expiry_date, is_current)
  VALUES (source.store_id, source.store_type, CURRENT_DATE, '9999-12-31', true);
```

**Why this matters:**
- **Atomic** — with the union trick, expire and insert genuinely happen in one transaction: no window where the old row is expired but the new version doesn't exist. (The naive version has the opposite problem — for changed entities that window never closes.)
- **Idempotent** — re-running with the same staging data is a no-op: the `!=` guard skips the expire, and the union's `WHERE` produces no NULL-key rows, so nothing gets inserted twice.
- **The readable alternative** — MERGE for the expire plus a separate INSERT in the same transaction, which is the pattern in the [theory doc](../theory/slowly_changing_dimensions.md). Both are correct; the union trick is one statement, the two-step version is easier to review.
- **Interview signal:** explaining *why* the naive single MERGE cannot work (one source row cannot hit both branches) shows you've implemented this, not just read about it.

</details>

---

## 🎯 How did you do?

- **9–10 correct** → SCDs are your strong suit. Drill [modeling approaches](../theory/modeling_approaches.md) next.
- **6–8 correct** → solid. Re-read the [SCD theory doc](../theory/slowly_changing_dimensions.md), especially Types 4 and 6.
- **< 6 correct** → start with Types 1 and 2 only. Master those before learning the others.

**Related:** [quiz/dimensional_modeling.md](dimensional_modeling.md)
