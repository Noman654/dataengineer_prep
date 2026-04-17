# 03 — PySpark

The heart of this repo. PySpark is what most DE interviews probe hardest, and Zephyr Coffee Co.'s messy data is where you actually get to practice against realistic problems instead of toy datasets.

> **New here?** Read [ZEPHYR.md](../ZEPHYR.md) first — that's the fictional company whose data you'll be working with throughout. Every notebook is framed as a Slack message from a Zephyr colleague (Jen in marketing, Marcus in finance, Dev in store ops) asking you to solve a realistic problem.

---

## 🧭 How this module works

Every topic below has some mix of four things:

- 📖 **Read first** — the single best free external resource. We don't duplicate what already exists well.
- 📓 **Hands-on notebook** — our own, framed around a Zephyr scenario. These exist only where external content is weak or where Zephyr gives you something realistic to practice against.
- 📝 **Theory doc** — a dense quick-review summary in [`theory/`](theory/). Written for senior interview prep — no fluff, no roleplay.
- 🏋️ **Practice** — where to drill problems.

**If a topic has no notebook, it's not missing — it's that external content already covers it well.** Follow the read-first link, then practice.

---

## 👋 Pick your path

### 🟢 Beginner (0–1 yr, never touched Spark)
Estimated: 3–4 weeks.

1. Open [`00_syntax_cheatsheet.ipynb`](00_syntax_cheatsheet.ipynb) — 30 min flat reference, no story
2. Walk through topics 1 → 7 in order
3. Skip ⚡ sections for now — come back when you have 1 year of hands-on experience

### 🟡 Intermediate (1–3 yrs, comfortable with DataFrames)
Estimated: 1–2 weeks.

1. Skim the syntax cheatsheet to fill gaps
2. Jump straight to the 🟡 notebooks: [window functions](02_window_functions.ipynb), [joins](03_joins.ipynb), null handling, nested data
3. Do every Boss Level — those push you where the walkthrough stops

### ⚡ Senior interview prep (4+ yrs, building pipelines in production)
Estimated: ~1 week focused review.

1. Start in [`theory/`](theory/) — shuffle, AQE, memory, skew
2. Read the ⚡ articles in "Advanced Topics" below
3. Speedrun the Boss Levels in every notebook
4. Go to `../interview_prep/system_design_scenarios.md` (coming in Phase 3)

---

## 📚 Topics

### 0. Syntax Reference 🟢 📓
A flat, hands-on reference of the stuff you actually type every day: DataFrame creation, schemas, UDFs (standard / pandas / SQL-based), `cache` vs `persist`, and more. **Start here if you've never touched PySpark** — no narrative, just syntax.

- 📓 [`00_syntax_cheatsheet.ipynb`](00_syntax_cheatsheet.ipynb)
- 📖 Companion reading: [Spark Quick Start](https://spark.apache.org/docs/latest/quick-start.html) (15 min, official)

---

### 1. DataFrame Basics & Transformations 🟢
No custom notebook — external content is already excellent. Don't waste time.

- 📖 **Read first (pick one):**
  - [Spark SQL Getting Started](https://spark.apache.org/docs/latest/sql-getting-started.html) — rigorous, ~30 min
  - [Spark by Examples: PySpark DataFrame Tutorial](https://sparkbyexamples.com/pyspark/pyspark-dataframe-tutorial/) — more code examples
  - [Learning Spark, 2nd Edition — Chapter 3](https://www.databricks.com/resources/ebook/learning-spark-2nd-edition) (free PDF from Databricks)
- 🏋️ **Practice:** Re-do the tutorial examples against Zephyr's transactions — same operations, realistic data.

---

### 2. Window Functions 🟡 📓
**Why it matters:** The single most-asked topic in DE interviews. Every pipeline uses them — running totals, rank, lag/lead, gaps-and-islands, deduplication.

- 📖 **Read first:**
  - [Databricks: Introducing Window Functions in Spark SQL](https://www.databricks.com/blog/2015/07/15/introducing-window-functions-in-spark-sql.html)
  - [Spark SQL window syntax reference](https://spark.apache.org/docs/latest/sql-ref-syntax-qry-select-window.html)
- 📓 **Hands-on:** [`02_window_functions.ipynb`](02_window_functions.ipynb)
  Jen from marketing needs to find loyal regulars (3+ consecutive months), detect churn (3 months of declining spend), and rank top customers per month. Teaches `lag` / `row_number` / `rank` / `dense_rank` through real problems.
- 🏋️ **Practice:**
  - [LeetCode SQL Study Plan](https://leetcode.com/studyplan/top-sql-50/) — problems 185, 262, 601 (same patterns in SQL)
  - [StrataScratch](https://www.stratascratch.com/) — filter by "Window Functions"
  - [DataLemur](https://datalemur.com/) — free DE-specific interview questions

---

### 3. Joins (all types + broadcast + skew) 🟡 📓
**Why it matters:** *"Explain broadcast joins vs shuffle joins"* and *"how do you handle skew in a join"* are guaranteed interview questions. Skew handling is the single biggest differentiator between junior and senior DE.

- 📖 **Read first:**
  - [Spark SQL join types reference](https://spark.apache.org/docs/latest/sql-ref-syntax-qry-select-join.html)
  - [Spark Performance Tuning: Join hints](https://spark.apache.org/docs/latest/sql-performance-tuning.html#join-strategy-hints-for-sql-queries)
- 📓 **Hands-on:** [`03_joins.ipynb`](03_joins.ipynb)
  Dev from Store Ops needs you to fix the loyalty ↔ POS reconciliation. Teaches: type mismatches across systems, left-anti-join for finding orphans, broadcast hint for small lookups, skew detection, salting the hot key.
- 📝 **Deep dive:** [`theory/shuffle_and_partitioning.md`](theory/shuffle_and_partitioning.md)

---

### 4. GroupBy & Aggregations 🟢
No custom notebook — syntax is clean and external tutorials are thorough.

- 📖 **Read:** [Spark by Examples: PySpark groupBy explained](https://sparkbyexamples.com/pyspark/pyspark-groupby-explained-with-example/)
- 🏋️ **Practice:** StrataScratch aggregation problems (Easy → Medium)

---

### 5. Null Handling & Deduplication 🟡 📓 *(Phase 2)*
**Why it matters:** Real data is messy. Zephyr's 2023 POS outage left duplicate transactions, walk-ins leave ~60% of `customer_id` NULL, and people naively filter nulls in ways that silently drop half their revenue. This is where "writes PySpark" differs from "understands data."

- 📖 **Read first:**
  - [Spark DataFrame na functions](https://spark.apache.org/docs/latest/api/python/reference/api/pyspark.sql.DataFrame.na.html)
- 📓 **Notebook:** *coming in Phase 2 — Marcus catches a revenue bug caused by an inner join dropping walk-ins, and the 2023 duplicate incident needs dedup.*

---

### 6. Nested Data (explode, structs, arrays) 🟡 📓 *(Phase 2)*
**Why it matters:** JSON payloads, event streams, loyalty events — nested data is everywhere in production pipelines.

- 📖 **Read first:**
  - [Spark by Examples: explode + selectExpr on nested JSON](https://sparkbyexamples.com/pyspark/pyspark-explode-array-and-map-columns-to-rows/)
- 📓 **Notebook:** *coming in Phase 2 — flatten Zephyr's `loyalty_events.payload` struct which varies by event type.*

---

### 7. Reading & Writing Data 🟢
**When to worry about this:** Once your data is bigger than ~1 GB. Before that, CSV is fine. Know Parquet's advantages (columnar, compression, predicate pushdown).

- 📖 **Read:**
  - [Spark Data Sources](https://spark.apache.org/docs/latest/sql-data-sources.html)
  - [Databricks: Parquet vs CSV vs JSON benchmarks](https://www.databricks.com/glossary/what-is-parquet)
- 🏋️ **Practice:** Write Zephyr's transactions as partitioned Parquet (partition by `store_id` or `month`), read back, compare file sizes and query speeds.

---

## ⚡ Advanced — Senior Interview Prep

These topics are where senior DE interviews actually live. External docs are scattered; this is where our `theory/` quick-review docs earn their keep.

### 8. Spark Architecture & DAG ⚡
- 📖 **Read first:**
  - [**Spark's Secret Ingredient: The DAG**](https://www.linkedin.com/pulse/sparks-secret-ingredient-dag-directed-acyclic-graph-revealed-nauman-wzyec/) — by Mohd Nauman *(author of this repo)*
  - [Understanding Spark DAGs](https://medium.com/plumbersofdatascience/understanding-spark-dags-b82020503444)

### 9. Shuffle & Partitioning ⚡ 📝
- 📖 **Read first:**
  - [Spark Performance Tuning — official](https://spark.apache.org/docs/latest/sql-performance-tuning.html)
  - [Mastering Spark Internals: Shuffle](https://books.japila.pl/apache-spark-internals/shuffle/) by Jacek Laskowski (free gitbook, dense)
- 📝 **Quick-review:** [`theory/shuffle_and_partitioning.md`](theory/shuffle_and_partitioning.md) — the 10-min revision doc for the night before an interview.

### 10. Memory Management ⚡ 📝
- 📖 **Read first:**
  - [**The Ultimate Guide to Spark Memory Management**](https://www.linkedin.com/pulse/ultimate-guide-spark-memory-management-data-engineers-mohd-nauman-4mjec/) — by Mohd Nauman *(author of this repo)*
  - [**Deep Dive into Spark Memory Management** by Luminousmen](https://luminousmen.com/post/dive-into-spark-memory/) — the most-shared memory post on LinkedIn; clear diagrams of the unified memory model
  - [**Why Your Spark Applications Are Slow or Failing, Part 1: Memory Management** by Rishitesh Mishra](https://dzone.com/articles/common-reasons-your-spark-applications-are-slow-or) — classic troubleshooting framework
  - [**Part 3: Cost-Efficient Executor Configuration** by Brad Caffey (Expedia)](https://medium.com/expedia-group-tech/part-3-efficient-executor-configuration-for-apache-spark-b4602929262) — best writeup on the 5-cores rule and executor sizing math
  - [Spark Memory Management — official docs](https://spark.apache.org/docs/latest/tuning.html#memory-management-overview)
- 📝 **Quick-review:** [`theory/memory_management.md`](theory/memory_management.md) — unified memory model, PySpark-specific OOM patterns, executor sizing, the 10-min revision doc.

### 11. Catalyst Optimizer & Adaptive Query Execution (AQE) ⚡
- 📖 **Read first:**
  - [Catalyst Optimiser: The Power of Spark SQL](https://medium.com/@Shkha_24/catalyst-optimizer-the-power-of-spark-sql-cad8af46097f)
  - [Databricks: Adaptive Query Execution](https://www.databricks.com/blog/2020/05/29/adaptive-query-execution-speeding-up-spark-sql-at-runtime.html) — the canonical AQE post
  - [Databricks Engineering blog — AQE deep dive](https://www.databricks.com/blog/2020/05/29/adaptive-query-execution-speeding-up-spark-sql-at-runtime.html)

### 12. Handling Data Skew ⚡
- 📖 **Read first:**
  - [Databricks: What is data skew and how to handle it](https://www.databricks.com/glossary/data-skew)
  - The salting pattern (covered in `03_joins.ipynb` Boss Level)

### 13. Structured Streaming ⚡ *(Phase 2)*
- 📖 **Read first:**
  - [Spark Structured Streaming Programming Guide](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html) — official, dense

### 14. Delta Lake ⚡ *(Phase 2)*
- 📖 **Read first:**
  - [Delta Lake docs](https://docs.delta.io/latest/index.html)
  - [Databricks: Delta Lake Transaction Log](https://www.databricks.com/blog/2019/08/21/diving-into-delta-lake-unpacking-the-transaction-log.html) — explains MERGE, time travel, OPTIMIZE

---

## 🏆 Boss Levels

Every hands-on notebook ends with a **Boss Level** — a harder, more ambiguous version of the core problem, meant to push you beyond the walkthrough. If you're interview-prepping, do all of them before anything else. They're intentionally designed to mirror the ambiguity of real interview questions (including the ones where there's no single right answer).

---

## 📝 Notes

- **Links checked:** 2026-04. If a link rots, grep this file and fix it — don't let it linger.
- **Colab badges** at the top of each notebook open the notebook directly in Google Colab with zero setup. Currently pointing to the `test` branch; update to `main` when this gets merged.
- **Phase 2** adds: null handling, nested data, structured streaming, Delta Lake notebooks. **Phase 3** adds SQL and Python modules + interview prep folder.
