# SFT Data Engineering — Quick Review

**Audience:** data/research engineers building supervised fine-tuning (SFT) datasets, and DEs interviewing for AI-lab data roles.

**Status:** 🚧 **Content in progress.** This doc is the fine-tuning-data counterpart to [`pretraining_data.md`](pretraining_data.md). Where pre-training is about web-scale corpus construction, SFT is about building high-quality instruction/response datasets.

---

## What this will cover

- **Instruction dataset construction** — sourcing prompts, generating/collecting responses, formatting (chat templates, system/user/assistant turns)
- **Quality control** — the SFT quality bar is different from pre-training; a small, clean dataset beats a large, noisy one
- **Diversity** — task coverage, instruction variety, why diversity matters more than volume for SFT
- **Human annotation vs synthetic** — when to use labelers, when to use model-generated data (Self-Instruct, Alpaca-style), how to mix
- **Data formatting & templates** — chat templates, special tokens, masking the loss on prompt tokens
- **Decontamination** — keeping eval benchmarks out of the SFT set
- **From experience** — the practitioner reality of building SFT data

---

*This doc is being written. See [`pretraining_data.md`](pretraining_data.md) for the pre-training side, which is complete.*

*Last revised: 2026-05.*
