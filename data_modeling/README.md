# Data Modeling

Data modeling is tested in nearly every DE interview — from "design a schema for X" to "how would you handle slowly changing dimensions?" This module covers the concepts that actually get asked.

---

## How to use this module

### If you're prepping for a specific company
Check [`company_interviews/`](../company_interviews/) first — the modeling questions vary by company. Then come back here for the concepts.

### If you're doing a general review
1. Start with [dimensional_modeling.md](theory/dimensional_modeling.md) — the foundation
2. Then [slowly_changing_dimensions.md](theory/slowly_changing_dimensions.md) — the most-probed topic
3. Drill the quizzes

---

## Theory docs

| Doc | What it covers | Read time |
|---|---|---|
| [Dimensional Modeling](theory/dimensional_modeling.md) | Fact vs dimension tables, star vs snowflake, grain, surrogate keys, normalization, Kimball's 4-step process | 15 min |
| [Slowly Changing Dimensions](theory/slowly_changing_dimensions.md) | SCD Types 0–6 with Zephyr examples, Type 2 SQL implementation, interview flow for SCD questions | 10–15 min |
| [Modeling Approaches](theory/modeling_approaches.md) | Kimball vs Inmon vs Data Vault vs OBT, medallion architecture, when to pick which | 10–15 min |

## Quizzes

| Quiz | Questions | Difficulty |
|---|---|---|
| [Dimensional Modeling](quiz/dimensional_modeling.md) | 10 | 🟢 → ⚡ |
| [Slowly Changing Dimensions](quiz/slowly_changing_dimensions.md) | 10 | 🟢 → ⚡ |
| [Modeling Approaches](quiz/modeling_approaches.md) | 10 | 🟢 → ⚡ |

---

## External resources (curated)

- **Kimball Group Design Tips** (free): [kimballgroup.com](https://www.kimballgroup.com/category/design-tips/) — the primary source. Start with Design Tips #51 (grain) and #152 (SCD types).
- **"The Data Warehouse Toolkit" by Ralph Kimball** (3rd ed.) — the definitive book on dimensional modeling. Not free, but the single best resource.
- **"Cracking the Data Modeling Interview" by Sean Coyne** (Medium) — multi-part series targeting DE interview questions.
- **Joe Reis's "Practical Data Modeling" Substack** — covers event modeling and modern patterns.
