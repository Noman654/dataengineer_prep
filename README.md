# Data Engineer Prep

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Last Commit](https://img.shields.io/github/last-commit/Noman654/dataengineer_prep)](https://github.com/Noman654/dataengineer_prep)
[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)](CONTRIBUTING.md)

Prep for your next data engineering interview. Work through PySpark notebooks framed as real problems from **Zephyr Coffee Co.** (a fictional 200-store chain with messy data), review the theory docs before senior rounds, drill the quizzes the night before.

Built in the open. Contributions welcome — see below.

---

## 📚 Navigate

- **[pyspark/](pyspark/)** — the PySpark module (start here)
- **[data_modeling/](data_modeling/)** — dimensional modeling, SCDs, star vs snowflake, grain
- **[ai_for_data_engineering/](ai_for_data_engineering/)** — ⭐ the DE work behind LLMs: RAG/agents (using LLMs) + pre-training & SFT data (building LLMs)
- **[company_interviews/](company_interviews/)** — company-wise DE interview patterns (Ola, Flipkart, Swiggy, PhonePe, Jio)
- **[ZEPHYR.md](ZEPHYR.md)** — the fictional company whose data runs through every notebook
- **[Roadmap](#roadmap)** — what's coming next
- **[Resources](#resources-for-data-engineers)** — thought leaders + resume examples
- **[Contributing](#contributing)**

---

## What's in here now

### [pyspark/](pyspark/)

The PySpark module. READMEs inside guide you through it based on your level (beginner / intermediate / senior).

**Hands-on notebooks** (each framed as a Slack message from a Zephyr colleague asking you to solve a realistic problem):
- [Syntax cheatsheet](pyspark/syntax_cheatsheet.ipynb) — 30-min flat reference
- [Window functions](pyspark/window_functions.ipynb) — consecutive months, churn detection, top-N per group
- [Joins](pyspark/joins.ipynb) — type mismatches, broadcast, skew detection, salting

**Theory docs** (10-min night-before-interview reviews):
- [Shuffle & partitioning](pyspark/theory/shuffle_and_partitioning.md)
- [Memory management](pyspark/theory/memory_management.md)
- [Catalyst & AQE](pyspark/theory/catalyst_and_aqe.md)
- [Data skew playbook](pyspark/theory/data_skew.md)
- [Spark UI & `.explain()` debugging](pyspark/theory/spark_ui_debugging.md)

**Self-check quizzes** (collapsible Q&A, 🟢 basics → ⚡ senior judgment):
- [Window functions](pyspark/quiz/window_functions.md) · [Joins](pyspark/quiz/joins.md) · [Memory](pyspark/quiz/memory_management.md) · [Catalyst/AQE](pyspark/quiz/catalyst_and_aqe.md) · [Skew](pyspark/quiz/data_skew.md) · [Spark UI](pyspark/quiz/spark_ui_debugging.md)

---

## Roadmap

Phase 2 (next, no dates):
- Null handling & deduplication notebook (Zephyr's 2023 POS duplicate incident)
- Nested data notebook (exploding loyalty event structs)
- Structured streaming notebook
- Delta Lake notebook
- Quiz + theory coverage for each

Phase 3:
- SQL module (window functions in SQL, gaps-and-islands, SCDs, query optimization)
- Python for DE module (collections, generators, pandas↔Spark, testing)
- System design scenarios for DE interviews
- DE interview question bank

The repo aims to be honest about what's built and what's not. No fake timelines.

---

## Resources for Data Engineers

### Thought leaders worth following
1. [Sumit Mittal](https://www.linkedin.com/in/bigdatabysumit/) — Founder of BigDataBySumit
2. [Joe Reis](https://www.linkedin.com/in/josephreis/) — Co-author of *Fundamentals of Data Engineering*
3. [Zach Wilson](https://www.linkedin.com/in/eczachly/) — Data engineering specialist 
4. [Shashank Mishra](https://www.linkedin.com/in/shashank219/) — Data engineer & educator
5. [Gowtham SB](https://www.linkedin.com/in/sbgowtham/) — Big data & cloud
6. [Manish Kumar](https://www.linkedin.com/in/manish-kumar-data-engineer/) - For questions and interview experience
7. [Darshil Parmar](https://www.youtube.com/@DarshilParmar) - For Crisp DE Videos
8. [Ansh Lamba](https://www.youtube.com/@AnshLambaJSR) - Best for Azure and Databricks

## Resource That I love 
1. [Data Pathshala](https://datapathsala.com/) Preparation of Data Engineering by Manish Kumar
2. [Data Engineer Handbook](https://github.com/DataExpert-io/data-engineer-handbook) DE Concepts
   
### Resume examples
- [Manish's DE resume](https://github.com/manisnitt/myresume/blob/main/manish_resume_github.pdf) — well-structured, shows skills/projects/experience clearly
- [My resume](https://docs.google.com/document/d/10e79n92zj-s92Ss55H8A7fM80UhWeeD5euWAX6AUrbA/edit?usp=sharing)
- [Jake Overleaf](https://www.overleaf.com/latex/templates/jakes-resume/syzfjbzwjncs) - Open as template then edit it as your resume on latex

---

## Contributing

Typo fixes, clearer explanations, new quiz questions, Zephyr scenario ideas, and blog-link additions (with a one-line justification for why it beats what's already linked) are all welcome. Open an issue or a PR.

**Please don't send:** random link dumps, self-promotional content, or AI-generated filler. The curation is the point — every external link in this repo was added because it's genuinely the best free resource for that topic, not because it exists.

