# Chapter 9 — Structured Output and Adapters

## 9.1 Conceptual picture

Two serving features that don't fit the memory-budget story but are essential in production: **constrained decoding** (forcing the model's output to conform to a schema — JSON, a grammar, a tool-call format) and **multi-LoRA serving** (running many fine-tuned adapters on one shared base model). Both are about *deployment flexibility* rather than raw performance, and both are now first-class in the major engines. Structured output is what makes LLMs usable as reliable components in software (an API that returns malformed JSON 5% of the time is unusable); multi-LoRA is what makes per-customer or per-task customization economical (one base model, many cheap adapters, one fleet).

## 9.2 Constrained decoding: how it works

An LLM outputs a probability distribution over the vocabulary at each step. **Constrained decoding** enforces structure by *masking* — at each step, it zeroes out the probability of any token that would violate the target grammar, so the model can only sample tokens that keep the output valid. Building a valid JSON object? Mask every token that isn't legal at this position given the JSON grammar and what's been emitted so far. The model still chooses *among* valid tokens by its own preferences, but it *cannot* produce invalid output. This guarantees schema-conformant output structurally, not probabilistically — 100% valid JSON, not 95%.

The engineering challenge is doing the masking *fast enough* to not bottleneck generation. Computing the legal-token mask per step, for a complex grammar, over a 100K+ vocabulary, must happen in microseconds or it dominates decode time.

## 9.3 The structured-output engines

| Backend | Approach | Speed | Notes |
|---|---|---|---|
| **XGrammar** | Precomputed grammar automaton + token-mask caching | <40 μs/token | The **default backend** for vLLM, SGLang, TensorRT-LLM as of 2026 |
| **llguidance** (Microsoft) | Rust Earley parser | ~50 μs/token | Negligible startup; flexible grammars |
| **Outlines** | Regex/JSON-schema → FSM | fast | Popular, widely integrated |
| **Pre³ / pushdown-automata methods** | Deterministic PDA | very fast | Research edge, faster complex grammars |

XGrammar won the default slot by making per-token masking nearly free (<40 μs, near-zero JSON overhead) through precomputation and caching. You typically don't choose the backend directly — you pass a **JSON schema, regex, or grammar** to the engine's API (`response_format`, `guided_json`, `guided_grammar`) and it uses its backend. What you *do* choose is the engine, because structured-output *throughput* differs sharply: **SGLang overlaps grammar-mask generation with GPU inference** so structured output "barely impacts throughput," while **vLLM shows noticeable degradation at batch ≥8** with guided decoding. If your workload is structured-output-heavy (lots of concurrent JSON/tool calls), that difference alone can justify SGLang (Chapter 5).

## 9.4 Function/tool calling

Tool calling is structured output with a convention: the model emits a JSON object naming a tool and its arguments, conforming to the tool's schema. Serving-side, it's constrained decoding against the tool schemas plus a parser that extracts the call. Practical notes:

- **The OpenAI function-calling JSON schema is the de facto standard** — vLLM, SGLang, TensorRT-LLM, and Ollama all speak it, so tool-calling client code is portable across engines and hosted APIs (the same portability hedge as Chapter 5).
- **Constrained decoding makes tool calls reliable** — without it, models occasionally emit malformed calls; with grammar enforcement against the tool schema, the call is structurally valid every time. Enable it for any agentic/tool workload.
- **Model support varies** — the model must be *trained* for tool use (post-training series, Chapter 9) for the calls to be *sensible*; constrained decoding only guarantees they're *well-formed*. A model that wasn't tool-trained will emit valid-but-useless calls.
- **A caution**: grammar-constrained generation can interact badly with reasoning (forcing structure too early suppresses the model's reasoning) and has a small security surface (maliciously crafted schemas as a control channel — the "grammar guides the attack" line of work). Let the model reason *then* constrain the final answer, and validate schemas you accept from untrusted sources.

## 9.5 Multi-LoRA serving

LoRA adapters (post-training series, Chapter 4; midtraining Chapter 4) are tiny — 1–2% of the base model's parameters — and *merge into the same base at inference*. This enables **multi-LoRA serving**: load one base model into HBM once, and serve many adapters (per-customer fine-tunes, per-task specializations, different capabilities) from it simultaneously, applying the right adapter per request. Instead of N full model deployments you run one base + N cheap adapters — a massive cost saving for anyone offering customization.

- **How it works** — the base weights are shared across the batch; each request's LoRA delta is applied via extra low-rank matmuls. Requests using *different* adapters can be batched together (the engine applies each request's adapter within the batched compute), so you don't lose batching efficiency.
- **Engine support** — vLLM and SGLang both serve multi-LoRA; **SGLang's scheduling is more efficient at high adapter counts** and batches across adapters natively. If you serve dozens-to-hundreds of adapters, that efficiency matters.
- **The economics** — this is what makes "a fine-tune per customer" viable. Hundreds of customers, one base model in memory, hundreds of small adapters swapped per request. Without multi-LoRA you'd need hundreds of full deployments; with it, one fleet.
- **Limits** — adapters must share the same base model and be small (LoRA-scale); a customer needing a full fine-tune (large-corpus SFT or CPT, midtraining Chapter 4) can't be served this way and needs its own deployment. Adapter loading/swapping has some overhead; hot adapters stay resident, cold ones load on demand.

## 9.6 Decisions

1. **Use constrained decoding for any structured-output or tool-calling workload** — it makes output structurally valid, not probabilistically valid.
2. **Pass a JSON schema/grammar to the engine's guided-decoding API**; you get XGrammar/llguidance under the hood. Choose the *engine* (SGLang for heavy structured-output throughput) rather than the backend.
3. **Standardize on the OpenAI tool-calling schema** for portability; enable constrained decoding for reliable calls, but ensure the model is tool-*trained* for sensible ones.
4. **Let the model reason before constraining** the final answer, and validate untrusted schemas.
5. **Serve customizations via multi-LoRA** — one base, many adapters, one fleet — using SGLang at high adapter counts; fall back to dedicated deployments only for full fine-tunes.

Next: hardware — the GPUs underneath all of this, chosen (unlike training) for memory, not FLOPS.
