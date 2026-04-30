# Mondee — My Interview

> **Author:** Mohd Nauman
> **Role:** Data Engineer
> **Rounds:** 2
> **Focus:** Project deep-dive (LLM inference infra) + Hypothetical system design

---

## Round 1 — Project Deep-Dive

**Duration:** ~45-60 min
**Format:** conversational, interviewer-driven

### Topics covered

The entire round was about my project involving LLM inference infrastructure. They went deep on:

**vLLM & SGLang:**
- Why we chose vLLM over other inference engines (TGI, Triton, etc.)
- How vLLM's PagedAttention manages KV-cache memory — why this matters for throughput at scale
- What SGLang adds on top — structured generation, constrained decoding, batching control
- The tradeoffs: vLLM for raw throughput vs SGLang for structured output control

**Inference control & serving:**
- How we controlled inference parameters (temperature, top-p, max tokens, stop sequences)
- Batching strategy — continuous batching vs static batching, why continuous batching wins for variable-length requests
- How we served the model — API endpoint design, request queuing, timeout handling
- Monitoring: latency percentiles (p50/p95/p99), throughput (tokens/sec), GPU utilization, queue depth

**Architecture decisions they probed:**
- Why self-hosted inference vs managed API (cost, latency control, data privacy)
- How we handled model versioning and A/B testing between model versions
- What happens when GPU memory is exhausted — fallback strategy, request rejection vs queuing

### How it went

Went well — they wanted to see depth of understanding, not just "I used vLLM." The key was explaining *why* each decision was made, not just *what* we used. They pushed hard on the tradeoffs between vLLM and SGLang and seemed satisfied when I explained the specific use case that drove the choice.

### What to prep if you get this

- Know vLLM internals: PagedAttention, continuous batching, KV-cache management
- Know when SGLang wins over vLLM (structured/constrained output, multi-turn, complex prompting)
- Be ready to explain your architecture decisions as tradeoffs, not just choices
- Have latency/throughput numbers from your project ready (they asked for specifics)

---

## Round 2 — Hypothetical System Design

**Duration:** ~45-60 min
**Format:** open-ended scenario, whiteboard-style

### The question

> "You have unstructured data flowing in real-time — logs, free-text, semi-structured JSON blobs. How would you stream this into structured, queryable tables in a warehouse? And how do you make sure it doesn't break when the source format changes?"

### Topics they were probing

This wasn't just "design a pipeline." They wanted to see how I think about:

**Ingestion layer:**
- Kafka for the streaming backbone — partition strategy, serialization format (Avro vs JSON vs Protobuf)
- How to handle mixed/unknown schemas at ingestion time (schema-on-read at bronze layer)

**Parsing & structuring:**
- How to extract structure from unstructured data — regex, NLP, LLM-based extraction, JSON path parsing
- Typing and casting — handling dirty types (strings that should be ints, mixed date formats)
- The "what if the source adds a new field?" question — schema evolution strategy

**Schema enforcement & safety:**
- Schema registry (Confluent, or even Pydantic for validation) at the boundary between raw and cleaned layers
- Backward + forward compatibility rules — what changes are safe (adding a nullable field) vs breaking (renaming a field, changing a type)
- Dead-letter queues for records that fail validation — don't drop silently, don't crash the pipeline
- Data contracts between producer and consumer teams

**Making sure it won't break:**
- Contract tests: validate incoming schema against expected schema before processing
- Schema evolution policy: only allow backward-compatible changes; breaking changes require a new topic/table version
- Monitoring + alerting: volume anomalies (sudden drop = source is broken), schema drift detection, null-rate spikes
- Idempotent processing: if you replay the stream, the output doesn't duplicate

### How it went

They were less interested in specific tools and more interested in the failure modes I anticipated. The strongest signal was when I talked about dead-letter queues and schema evolution — "what happens when it breaks" mattered more than "how you build it."

### What to prep if you get this

- Know the medallion pattern (bronze = raw/unstructured, silver = cleaned/typed, gold = modeled)
- Have a clear schema evolution strategy (backward/forward compat, versioned topics)
- Dead-letter queue pattern — what goes there, how you monitor it, when you replay
- Data contracts — what they are, how to enforce at the producer boundary
- Be ready for "what happens when X breaks?" follow-ups at every layer

---

## Tips

- This was a 2-round format — no separate DSA or SQL round. The focus was entirely on system thinking and project depth.
- If you've built LLM inference infra, know your numbers cold (latency, throughput, cost per request). They will ask.
- For the hypothetical round, don't jump to tools. Start with: "What's the SLA? What's the volume? What does 'break' mean to this business?" — then design.
- The "make sure it won't break" part was 50% of Round 2. Failure handling > happy path.

---

## Related prep

- Streaming unstructured → structured: connects to [`pyspark/theory/catalyst_and_aqe.md`](../pyspark/theory/catalyst_and_aqe.md) (schema handling) and [`data_modeling/theory/dimensional_modeling.md`](../data_modeling/theory/dimensional_modeling.md) (how to structure the output)
- Schema evolution + data contracts: emerging topic — see `company_interviews/README.md` for context
