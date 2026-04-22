# Review Log

Tracks what's been reviewed by Claude so future sessions can skip already-vetted files.

**How to use:** when reviewing files, look at the table below first. Skip rows marked `✅ Clean` unless the file has been modified since the review date. Re-review rows marked `🟡 Issues open`.

---

## Status legend

- ✅ **Clean** — reviewed, no issues, or all issues fixed
- 🟡 **Issues open** — reviewed, issues found, not yet fixed
- ⏸️ **Skipped** — intentionally not reviewed (e.g., data files)
- ❌ **Stale** — reviewed previously, but file has been substantially modified since

---

## Files

| File | Last reviewed | Status | Notes |
|---|---|---|---|
| `README.md` | 2026-04-22 | ✅ Clean | Minor: verify resume Google Doc is view-only |
| `ZEPHYR.md` | 2026-04-22 | ✅ Clean | Priya defined but unused in current notebooks |
| `pyspark/README.md` | 2026-04-22 | ✅ Clean | Section 12.5 fractional numbering kept by choice |
| `pyspark/syntax_cheatsheet.ipynb` | 2026-04-22 | ✅ Clean | Fixed: cell-6 rdd→data, cell-15 modernized Pandas UDF |
| `pyspark/window_functions.ipynb` | 2026-04-22 | ✅ Clean | — |
| `pyspark/joins.ipynb` | 2026-04-22 | ✅ Clean | — |
| `pyspark/theory/shuffle_and_partitioning.md` | 2026-04-22 | ✅ Clean | — |
| `pyspark/theory/memory_management.md` | 2026-04-22 | ✅ Clean | — |
| `pyspark/theory/catalyst_and_aqe.md` | 2026-04-22 | ✅ Clean | Fixed: Shikha Gupta → Bhatia, Optimiser → Optimizer |
| `pyspark/theory/data_skew.md` | 2026-04-22 | ✅ Clean | Fixed: replaced Syeed Hasan with verified Subham Khandelwal article |
| `pyspark/theory/spark_ui_debugging.md` | 2026-04-22 | ✅ Clean | Fixed: removed Jules Damji misattribution + unverified Databricks session URL |
| `pyspark/quiz/window_functions.md` | 2026-04-22 | ✅ Clean | Fixed: replaced "coming soon" with Related section |
| `pyspark/quiz/joins.md` | 2026-04-22 | ✅ Clean | — |
| `pyspark/quiz/memory_management.md` | 2026-04-22 | ✅ Clean | — |
| `pyspark/quiz/catalyst_and_aqe.md` | 2026-04-22 | ✅ Clean | — |
| `pyspark/quiz/data_skew.md` | 2026-04-22 | ✅ Clean | — |
| `pyspark/quiz/spark_ui_debugging.md` | 2026-04-22 | ✅ Clean | Fixed: replaced unverified URL with official Spark Web UI docs |
| `assets/sample_data/` | — | ⏸️ Skipped | Data files, not reviewable code |

---

## Review history (chronological)

*(Each review session adds an entry below — what was reviewed, what was found, what was fixed.)*

### 2026-04-22 — full repo review (Claude session)

**Scope:** all 17 content files (READMEs, theory docs, quizzes, notebooks).

**Clean (10 files):** root README, ZEPHYR.md, pyspark/README, shuffle_and_partitioning.md, memory_management.md, quiz/joins.md, quiz/memory_management.md, quiz/catalyst_and_aqe.md, quiz/data_skew.md, window_functions.ipynb, joins.ipynb.

**Issues found (7 items across 6 files) — ALL FIXED same session:**

1. ✅ **catalyst_and_aqe.md** + **pyspark/README.md** — "Shikha Gupta" → "Shikha Bhatia", "Optimiser" → "Optimizer".
2. ✅ **data_skew.md** — replaced fabricated Syeed Hasan URL with verified Subham Khandelwal article.
3. ✅ **spark_ui_debugging.md** — removed Jules Damji misattribution (was Julius Krah's handle).
4. ✅ **spark_ui_debugging.md** — removed unverified Databricks session URL.
5. ✅ **quiz/spark_ui_debugging.md** — replaced unverified URL with official Spark Web UI docs link.
6. ✅ **quiz/window_functions.md** — replaced stale "coming soon" with Related section.
7. ✅ **syntax_cheatsheet.ipynb** — cell-6: `rdd` → `data`; cell-15: modernized Pandas UDF with type hints.

---
