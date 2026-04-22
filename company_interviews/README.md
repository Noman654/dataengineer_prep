# Company Interviews

Real DE interview experiences — personal + community-sourced patterns — organized by company with prep recommendations linked to this repo.

---

## How this section works

Each company folder has two files:

- **`online.md`** — community-sourced patterns synthesized from GFG, Glassdoor, Medium, Reddit, YouTube. Includes round structure, commonly asked topics, and **"What a Strong Answer Looks Like"** skeletons for the hardest questions.
- **`my_interview.md`** — first-person experience from someone who actually interviewed there. Includes a **"Delta from online.md"** section showing where community intel matched reality vs where it didn't.

**Why both?** GFG gives you questions. Glassdoor gives you reviews. Neither tells you what a *strong* answer looks like or how the real thing compares to what the internet says. The cross-reference is the point.

---

## NDA Guidelines

Most companies consider exact interview questions confidential. To stay safe:

- **Do share:** topic areas, round structure, difficulty, what they were testing, your prep strategy
- **Do share:** question *patterns* (e.g., "window functions on ride data", "design a surge pricing pipeline")
- **Don't share:** exact question text copied verbatim from a live interview, proprietary datasets, take-home code
- **Don't share:** interviewer names, internal eval rubrics, specific offer numbers (unless you're comfortable)

When in doubt, describe the *type* of question, not the question itself.

---

## Companies

| Company | Rounds | Focus Areas | Interview Patterns |
|---|---|---|---|
| [Ola](ola/) | 4–6 rounds | SQL, System Design, Spark | [online.md](ola/online.md) |
| [Flipkart](flipkart/) | 5 rounds | Spark coding, Data Modeling | [online.md](flipkart/online.md) |
| [Swiggy](swiggy/) | 4–5 rounds | SQL, Spark, Pipeline Design | [online.md](swiggy/online.md) |
| [PhonePe](phonepe/) | 3–4 rounds | SQL (heavy), Spark internals | [online.md](phonepe/online.md) |
| [Jio](jio/) | 4 rounds | DSA, Big Data, Projects | [online.md](jio/online.md) |

> **Personal experiences** will be added once filled using the [`_template_personal.md`](_template_personal.md) format. Contributions welcome.

---

## How to use

1. Open the **company folder** you're targeting
2. Read `online.md` — round breakdown, question patterns, tips
3. Cross-reference with [`pyspark/quiz/`](../pyspark/quiz/) and [`pyspark/theory/`](../pyspark/theory/) for concept practice

---

## Contributing

**Add your own experience:**
1. Fork the repo
2. Copy [`_template_personal.md`](_template_personal.md) into the company folder
3. Fill it in honestly — follow the NDA guidelines above
4. Open a PR

**Add community research:**
1. Copy [`_template_online.md`](_template_online.md) into a company folder
2. Synthesize from public sources (GFG, Glassdoor, Reddit) — cite every source
3. Don't copy-paste verbatim from other sites (copyright)
4. Open a PR

**Add a new company:** create a folder, add at least one of the two files, update the table above.

---

## Status

| Company | `online.md` | Last updated |
|---|---|---|
| `ola/` | ✅ Sourced | 2026-04 |
| `flipkart/` | ✅ Sourced | 2026-04 |
| `swiggy/` | ⏸️ Partial | 2026-04 |
| `phonepe/` | ✅ Sourced | 2026-04 |
| `jio/` | ✅ Sourced | 2026-04 |
