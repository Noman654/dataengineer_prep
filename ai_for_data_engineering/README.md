# AI for Data Engineering

Data engineering doesn't stop at warehouses and dashboards anymore. LLMs are now both a **consumer** of data pipelines and a **product** that data pipelines build. This module covers the data engineering work on both sides.

**The framing:** this is about *pipelines*, not models. We don't teach prompt engineering, model architecture, or how transformers work. We teach the data engineering — ingestion, transformation, quality, scale — that LLM systems depend on.

---

## Two tracks

### Track A — Using LLMs (every data engineer should know this)

The data engineering behind LLM *applications* — RAG, agents, and LLM-powered pipelines.

| Topic | What it covers |
|---|---|
| RAG ingestion pipelines | Extract → chunk → embed → load to vector store; freshness; failure modes |
| Agents and data | The data plumbing behind agentic systems — tool I/O, state, memory stores |
| Vector stores as infrastructure | Vector stores as a data system — schema, metadata, hygiene, scaling |

### Track B — Building LLMs (for data/research engineers at AI labs)

The data engineering behind *training* LLMs. This is heavy, web-scale pipeline work — and it's almost completely absent from DE prep resources.

| Topic | What it covers |
|---|---|
| Pre-training (PT) data | Web-scale ingestion, deduplication, quality filtering, decontamination, tokenization, dataset mixing |
| SFT data | Supervised fine-tuning dataset construction — sourcing, formatting, quality control, diversity |

---

## Why this module exists

Every other module in this repo (PySpark, data modeling) teaches skills that many prep resources cover. **This module is the differentiator.**

- **Track A** is rising fast — "data engineering for AI" is now a real interview category.
- **Track B** is severely under-served — almost no prep resource covers data engineering for LLM *training*, even though it's some of the highest-leverage data work being done in 2026.

The content here is written from hands-on experience building both pre-training and SFT datasets — not summarized from blog posts.

---

## Where DE ends and ML begins

This module stays on the data engineering side of the line:

| ✅ In scope (data engineering) | ❌ Out of scope (ML) |
|---|---|
| Ingestion pipelines for documents / web data | Model architecture, transformers |
| Chunking, embedding orchestration | Prompt engineering, retrieval tuning |
| Vector store schema + hygiene | Reranking model selection |
| PT/SFT dataset construction, dedup, quality | Training loop, loss functions, hyperparameters |
| Data quality for LLM inputs and outputs | Inference kernel optimization (vLLM internals) |

If a topic is about the *model*, it's out. If it's about the *data feeding or flowing around the model*, it's in.

---

## Docs

The module is split by how LLMs show up in data engineering work:

- [`using_llms/`](using_llms/) — Track A: data pipelines behind LLM applications.
- [`building_llms/`](building_llms/) — Track B: data pipelines for training and fine-tuning LLMs.

**Track A — Using LLMs:**
- `rag_pipelines.md` — extract → chunk → embed → load; freshness; failure modes
- `agents_and_data.md` — data plumbing behind agentic systems
- `vector_stores.md` — vector stores as a data system

**Track B — Building LLMs:**
- [`pretraining_data.md`](building_llms/pretraining_data.md) — web-scale ingestion, dedup, quality filtering, decontamination, data mixing, infrastructure at scale, training formats, PII/safety, ablation methodology
- [`sft_data.md`](building_llms/sft_data.md) — instruction dataset construction, formatting, quality control, diversity

## Status

| Track | Theory docs | Quizzes |
|---|---|---|
| A — Using LLMs | Planned in `using_llms/` | Planned |
| B — Building LLMs | `pretraining_data.md` drafted; `sft_data.md` in progress | Planned |

*Theory docs are being written. Check back, or watch the repo.*
