# Pre-Training Data Engineering — Quick Review

**Audience:** data/research engineers building LLM pre-training corpora, and DEs interviewing for AI-lab data roles. This is the **data engineering** behind training an LLM — collection, cleaning, dedup, quality, scale — not the model.

**Read time:** 20 minutes (it's a big topic — skim the section headers and dive where you need depth).

---

## TL;DR — the one-sentence mental model

> Pre-training data engineering is **ETL at web scale where the output is a tokenized corpus instead of a warehouse table.** You ingest the web (Common Crawl), clean it (extract, dedup, quality-filter, remove PII, decontaminate), decide what mix to train on, then tokenize and pack it into a binary format the training run can stream. Every decision is validated by **training small proxy models and benchmarking** — that ablation loop is the scientific method of the field.

If you remember one thing: **the model is only as good as the data, and "good data" is an empirical question you answer with ablations, not opinions.**

---

## Where DE ends and ML begins

| ✅ In scope (this doc) | ❌ Out of scope (ML) |
|---|---|
| Collection, scraping, extraction | Transformer architecture, attention |
| Dedup, quality filtering, PII/safety | Training loop, optimizer, loss |
| Decontamination, data mixing (construction) | The mixing *optimization algorithm* (DoReMi internals) |
| Tokenizer training, sequence packing | Learning-rate schedules |
| Infrastructure, data formats, provenance | Model eval beyond data-ablation benchmarking |

Rule: if it's about the **data feeding the model**, it's in. If it's about the **model itself**, it's out.

---

## Just enough model context (so the data decisions make sense)

- **Tokenization (BPE):** raw text → UTF-8 bytes → byte-pair encoding merges frequent adjacent pairs into subword tokens → a fixed vocabulary. The model only ever sees token IDs. Rough rule: **~4 characters / ~0.75 words per token** (varies by tokenizer and language).
- **Next-token prediction:** the entire self-supervised training signal is "predict the next token." That's why corpus quality is everything — the model learns the patterns in whatever you feed it.
- **Scaling law (Chinchilla):** compute-optimal training is **~20 tokens per parameter** ([Hoffmann et al., 2022](https://arxiv.org/abs/2203.15556)). A 70B model → ~1.4T tokens to be compute-optimal. Modern models deliberately "over-train" smaller models on far more tokens to cut *inference* cost — so token budgets often exceed the Chinchilla point.

---

## The pipeline

```
Web (Common Crawl) + licensed + code
        │
        ▼
   Extract (WARC → clean text)
        │
        ▼
   Dedup (exact → fuzzy → sequence)
        │
        ▼
   Quality filter (heuristic + model-based)
        │
        ▼
   PII + safety filter
        │
        ▼
   Decontaminate (remove benchmark data)
        │
        ▼
   [ Ablate: train proxy model, benchmark, decide ]  ← the loop that validates every step above
        │
        ▼
   Mix domains (web/code/books/math %)
        │
        ▼
   Tokenize + pack → binary shards → training run
```

This is bronze → silver → gold with extra rigor. The ablation loop is what makes it science instead of guesswork.

---

## Stage 1 — Collection & scraping

**Common Crawl** is the starting point for nearly every open corpus. Nonprofit, running since 2008, ~monthly snapshots (~2–2.5B pages, ~350–400 TiB each). Three formats:

| Format | Contains |
|---|---|
| **WARC** | Raw HTTP response + headers + full HTML |
| **WET** | Extracted plaintext only |
| **WAT** | Metadata — links, headers, status codes |

**Critical practitioner detail — this is a free quality win:** serious dataset builders **re-extract text from WARC with Trafilatura**, they don't trust WET. WET keeps too much menu/nav/boilerplate.

FineWeb ablated this directly (2019-18 dump, identical downstream processing): WET produced **~254B tokens** vs Trafilatura-on-WARC's **~200B tokens** — the WET version was ~25% larger but the extra tokens were mostly boilerplate, and the Trafilatura-WARC model **clearly outperformed** on the benchmark suite. The cleaner extraction beats the bigger corpus.

> 💡 **From experience:** WARC + Trafilatura over WET lifts the aggregate benchmark by **~3 points**. It's one of the highest-ROI decisions in the whole pipeline — you change *nothing* else and the model gets better, because you stopped feeding it nav menus and cookie banners.

*(Source: the [FineWeb blogpost](https://huggingfacefw-blogpost-fineweb-v1.static.hf.space/index.html) shows this as a chart — the token counts are stated in text, the exact point-delta is read off the plot.)*

**Extraction toolchain:** **Trafilatura** is the de-facto standard for boilerplate removal (used by FineWeb, RefinedWeb); Resiliparse is faster with lower text retention. Pipeline order is typically: URL filtering (block NSFW/malicious domains) → text extraction → language ID (fastText) → heuristic filters → dedup.

**Other sources:** licensed corpora (book/news deals), code repos (filtered per-repo by license — see Code & Math below).

**Legal/ethical scraping:** most training crawlers honor `robots.txt`. Research shows robots-compliant pre-training is feasible and reduces copyright memorization. Many top sites now explicitly block AI crawlers — the legal landscape is unsettled (see Licensing below).

---

## Stage 2 — Deduplication

**Why it matters:** duplicates inflate memorization, cause train/test contamination, and waste compute. The foundational paper ([Lee et al., 2021](https://arxiv.org/abs/2107.06499)) found a 61-word sentence repeated 60,000+ times in C4; deduplication cut memorized output ~10× and improved training efficiency.

| Method | Mechanism | When to use |
|---|---|---|
| **Exact** | Hash the full doc/string, drop identical | Cheap first pass |
| **Fuzzy / near (MinHash + LSH)** | Shingle into n-grams → MinHash signature → bucket via LSH → match on bucket collision | **The standard at scale.** FineWeb: 5-grams, 112 hashes, 14 buckets × 8 |
| **Sequence / substring (suffix array)** | Find long repeated substrings *within* documents | Removes repeated passages inside otherwise-unique docs |
| **Semantic (embedding)** | Embed + cluster by cosine similarity | Catches paraphrases string methods miss; higher false-positive rate |

**Document-level** removes whole near-duplicate pages; **sequence-level** removes repeated spans inside docs. FineWeb found per-snapshot dedup actually beat global cross-snapshot dedup — counterintuitive, worth knowing.

---

## Stage 3 — Quality filtering

**Heuristic filters (the Gopher / MassiveText rules):** word count bounds, mean word length, **symbol-to-word ratio < 0.1**, excessive bullets/ellipses, stop-word presence, and repetition filters (duplicate lines/paragraphs/n-grams). Cheap, fast, catches obvious garbage.

**Model-based filters:** **FineWeb-Edu** trained a classifier on 450K Llama-3-70B annotations scoring "educational value" 0–5; thresholding at 3 removed **~92%** of FineWeb, and the remaining 1.3T tokens **beat datasets ~10× larger** on MMLU/ARC/OpenBookQA ([FineWeb, 2024](https://arxiv.org/abs/2406.17557)).

This is the **2024–25 "quality over quantity" shift**: aggressive filtering of a huge corpus beats raw scale.

---

## Stage 4 — PII & safety filtering

Now a standard pipeline stage, not optional:

- **PII redaction:** regex/NER for emails, phone numbers, IDs. Dolma redacts emails and phone numbers; The Stack v2 does PII redaction on code.
- **Toxic/unsafe content:** classifier-based removal. Dolma drops content a Jigsaw fastText classifier scores as toxic above a threshold.
- **CSAM and illegal content:** hash-matching against known databases, removal.
- **Opt-out compliance:** respecting `robots.txt` and rights-reservation signals at crawl time.

This is legally critical and squarely DE. ([Dolma, AI2](https://allenai.org/blog/dolma-3-trillion-tokens-open-llm-corpus-9a0ff4b8da64))

---

## Stage 5 — Decontamination

**What:** removing benchmark/test examples from the training data. **Why:** overlap inflates eval scores, making evaluation dishonest. **How:** n-gram matching against known benchmark sets (EleutherAI's `lm-eval-harness` builds n-gram dictionaries and flags matches).

**Limitation:** string-based n-gram matching misses paraphrased, translated, or "soft semantic" duplicates. LLM-based decontaminators are an emerging fix.

---

## How you decide what works — ablation methodology

**This is the part that separates "I ran a filter" from "I validated a filter," and it's the most under-appreciated DE skill in the field.**

You don't pick filtering rules or data mixes by opinion. You:

1. Build several variants of the dataset (e.g., filter A vs filter B vs no filter)
2. Train a **small proxy model** on each (FineWeb used a 1.71B-param model on 350B tokens per variant)
3. Benchmark each proxy on a fixed eval suite (MMLU, ARC, HellaSwag, etc.)
4. Keep the variant that wins

Every filtering and mixing decision in a serious corpus is backed by an ablation like this. The infrastructure to run dozens of these cheaply *is* the data engineering. ([FineWeb methodology](https://arxiv.org/abs/2406.17557))

---

## Data mixing / domain weighting

How you set the proportion of web vs code vs books vs math vs academic. The published reference point is the **LLaMA mixture**: CommonCrawl 67%, C4 15%, GitHub/Wikipedia/Books ~4.5% each, arXiv 2.5%, StackExchange 2% ([LLaMA, 2023](https://arxiv.org/abs/2302.13971)).

**DoReMi** ([Xie et al., NeurIPS 2023](https://arxiv.org/abs/2305.10429)) automates weight selection by training a small proxy with Group DRO; reported +6.5% downstream and 2.6× fewer steps. *Note:* the optimization algorithm is ML — as a DE, your job is **constructing and resampling the mixture** to hit the target weights, not the DRO math.

---

## Multilingual balancing

Low-resource languages need upsampling or they vanish. Techniques: **temperature sampling** (flatten the language distribution) and **UniMax** ([2023](https://arxiv.org/abs/2304.09151)) — cap per-language repeats to avoid over-training on small languages. Also note the multilingual **"tokenization tax"**: underrepresented languages get fewer BPE merges → more tokens per word → higher cost and smaller effective context.

---

## Data repetition / epochs

How many times can you repeat data before it stops helping? "Scaling Data-Constrained Language Models" ([Muennighoff et al., NeurIPS 2023](https://arxiv.org/abs/2305.16264)) found up to **~4 epochs** of repetition is nearly as good as fresh unique data; returns decay sharply after that. Useful when you're data-constrained and can't get more unique tokens.

---

## Tokenizer training (distinct from tokenization)

Tokenization is *applying* a tokenizer. Tokenizer **training** is building one on your corpus:

- **Vocab size** is a tradeoff (commonly 32k–256k). Larger vocab = fewer tokens per doc but a bigger embedding table.
- You train the tokenizer on a representative *sample* of the corpus — research suggests ~150–180 GB captures 90%+ of the eventual vocabulary, with diminishing returns after ([2025](https://arxiv.org/abs/2502.20273)).
- Get this wrong (tokenizer trained on a non-representative sample) and you pay a fertility tax on the real corpus forever.

*(Note: specific vocab-size numbers for individual models vary — verify against the model card if citing exact figures.)*

---

## Infrastructure at scale — the most DE-native part

Processing petabytes is an engineering problem, and it's where your distributed-systems skills transfer directly:

- **datatrove** (HuggingFace) — the framework behind FineWeb. Composable pipeline blocks, runs on Slurm or locally.
- **NVIDIA NeMo Curator** — GPU-accelerated curation (dedup, filtering). HuggingFace and NVIDIA partnered in 2025 to accelerate datatrove with it.
- **Spark / Ray** — general distributed processing for the heavy stages (dedup, filtering at scale).
- **Sharding & streaming** — the corpus is split into thousands of shards; processing and training both stream shards rather than loading everything.

The skills from [`../../pyspark/`](../../pyspark/) — partitioning, shuffles, skew handling, memory management — apply directly here. A skewed dedup join at petabyte scale is the same problem as a skewed join on Zephyr's data, just bigger.

---

## Training data formats + sequence packing

How tokenized data is stored and served to the training run:

- **Formats:** Megatron `.bin/.idx`, **WebDataset** (tar shards), **MosaicML MDS / StreamingDataset** (stream from object storage), Parquet. The goal: stream shards efficiently to GPUs without I/O bottlenecks.
- **Sequence packing:** concatenate multiple documents into fixed-length sequences (e.g., 8192 tokens) to eliminate padding waste. Requires attention masking so the model doesn't attend across document boundaries. This is pure dataloader/DE work and a real efficiency lever. ([sequence packing explainer](https://huggingface.co/blog/sirluk/llm-sequence-packing))

---

## Metadata & provenance

Track per-document metadata — it's what makes the corpus filterable, mixable, and auditable:

| Field | Why it matters |
|---|---|
| Source URL / crawl date | Provenance, license determination, audit |
| Language + confidence | Mixing, filtering |
| Quality score | Threshold filtering, mixing |
| Dedup cluster ID | Reproduce / adjust dedup decisions |
| License | Legal compliance, commercial-use eligibility |
| Token count | Length filters, budget control |

Provenance tracking is increasingly **required by regulators** (EU AI Act training-data disclosure, mandatory since Aug 2025). *(The exact canonical tooling for lineage here is evolving — verify current best practice before standardizing on a tool.)*

---

## Synthetic data

Model-generated training data. The three landmark approaches:

1. **"Textbooks Are All You Need" (phi, Microsoft 2023)** — phi-1 (1.3B params) hit 50.6% on HumanEval using ~7B tokens of filtered + synthetic "textbook-quality" data. Quality can substitute for scale. ([arXiv 2306.11644](https://arxiv.org/abs/2306.11644))
2. **Self-Instruct (2022)** — bootstrap instruction data from the model itself; inspired Alpaca (52K examples for ~$600). ([arXiv 2212.10560](https://arxiv.org/abs/2212.10560))
3. **Distillation** — train a smaller student on a larger teacher's outputs (check the teacher's license).

**The critical risk — model collapse:** training recursively on AI-generated data causes *irreversible* degradation; the distribution's tails vanish ([Shumailov et al., *Nature* 2024](https://www.nature.com/articles/s41586-024-07566-y)).

**The right way (2025–26 consensus):** **mix, don't replace.** Roughly 1/3 synthetic + 2/3 real web is a documented sweet spot; apply identical dedup/PII/quality filtering to synthetic data; verify it; guard against benchmark contamination (synthetic data can leak eval content).

---

## Licensing ⚠️

**The core problem:** most corpora are scraped copyrighted material; fair use is unsettled.

**Key cases (status as of late 2025 — re-verify before relying on these, they move fast):**
- **NYT v OpenAI** — dismissal denied, claims proceed, no fair-use ruling yet
- **Authors Guild v OpenAI** — motion to dismiss denied (Oct 2025), court declined to rule on fair use
- **Getty v Stability (UK)** — Getty largely lost, narrow trademark win, under appeal
- **Bartz v Anthropic** — *the pivotal one:* training on legally-acquired books = fair use, but downloading from pirate "shadow libraries" = not. Settled Sept 2025 for **$1.5B**. **Lesson: how you acquire data is legally decisive.**

**What this means for you as a DE:** capture **license + provenance + acquisition-source** metadata per record, respect `robots.txt`/opt-outs at crawl time, exclude pirated sources, and be ready to produce the disclosure summaries regulators now require.

---

## The canonical datasets (reference)

| Dataset | Creator | Base | Key move | Why it mattered |
|---|---|---|---|---|
| **C4** | Google (T5) | 1 CC snapshot | First "clean CC" recipe | Standard baseline (~750 GB) |
| **The Pile** | EleutherAI | 22 sources | Curated multi-source mix | Diverse curation helps (825 GiB) |
| **RefinedWeb** | Falcon/TII | CC only | Aggressive filter + dedup | Proved web-only beats curated |
| **RedPajama** | Together AI | LLaMA's mix | Open reproduction | First open LLaMA-recipe repro (1.2T tokens) |
| **Dolma** | AI2 | CC + 6 sources | Fully documented pipeline | Largest transparent corpus (~3T tokens, powers OLMo) |
| **FineWeb / FineWeb-Edu** | HuggingFace | 96 CC snapshots | Quality classifier | 1.3T quality subset beat sets ~10× larger |

*(Sizes vary by version — FineWeb is cited as both 15T and 18.5T tokens across versions. Cite the specific version you mean.)*

---

## Must-know papers

- **Chinchilla** (2022) — the ~20 tokens/parameter scaling rule. [arXiv 2203.15556](https://arxiv.org/abs/2203.15556)
- **LLaMA** (2023) — the canonical published data mixture. [arXiv 2302.13971](https://arxiv.org/abs/2302.13971)
- **Deduplicating Training Data Makes LMs Better** (2021) — the dedup case. [arXiv 2107.06499](https://arxiv.org/abs/2107.06499)
- **Textbooks Are All You Need** (2023) — quality over scale. [arXiv 2306.11644](https://arxiv.org/abs/2306.11644)
- **FineWeb** (2024) — quality filtering + the ablation methodology. [arXiv 2406.17557](https://arxiv.org/abs/2406.17557)
- **Scaling Data-Constrained LMs** (2023) — the ~4-epoch repetition result. [arXiv 2305.16264](https://arxiv.org/abs/2305.16264)

---

## Common pitfalls

1. **Trusting Common Crawl's WET files.** Re-extract from WARC — WET keeps boilerplate.
2. **Skipping dedup or doing only exact dedup.** Near-duplicates (MinHash) are where the real gains are.
3. **Picking filters by opinion instead of ablation.** Train proxy models and benchmark — that's the whole methodology.
4. **Ignoring decontamination.** Benchmark leakage makes your eval numbers a lie.
5. **Pure synthetic data.** Model collapse is real. Mix with real data, filter, verify.
6. **No provenance/license metadata.** The Bartz $1.5B settlement proves acquisition source is legally decisive — track it from day one.
7. **Forgetting sequence packing.** Padding waste at scale is enormous; packing recovers it.
8. **Treating this as ML.** The model is ML. The corpus is data engineering. Stay on your side of the line.

---

## Further reading

- [FineWeb: decanting the web](https://arxiv.org/abs/2406.17557) — the single best end-to-end modern reference, includes the ablation methodology
- [datatrove (HuggingFace)](https://github.com/huggingface/datatrove) — the actual pipeline framework
- [NVIDIA NeMo Curator](https://github.com/NVIDIA-NeMo/Curator) — GPU-accelerated curation
- [Dolma (AI2)](https://allenai.org/blog/dolma-3-trillion-tokens-open-llm-corpus-9a0ff4b8da64) — fully documented open pipeline
- [Common Crawl WARC/WAT/WET formats](https://commoncrawl.org/blog/web-archiving-file-formats-explained)

Related in this repo: [`../../pyspark/`](../../pyspark/) (the distributed-processing skills transfer directly), [`sft_data.md`](sft_data.md) (the fine-tuning data counterpart).

---

*Last revised: 2026-05. Lawsuit statuses and dataset sizes change — re-verify time-sensitive claims before relying on them.*
