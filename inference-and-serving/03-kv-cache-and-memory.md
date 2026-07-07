# Chapter 3 — KV Cache and Memory

## 3.1 Conceptual picture

When a transformer generates token N, attention needs the keys and values of all previous tokens 1…N-1. Recomputing them every step would make generation quadratic and hopeless. So we *cache* them: the **KV cache** stores the key and value vectors for every token in the context, for every layer and every attention head, and each new token appends its own K and V and attends over the whole cache. This is what makes decode linear per token instead of quadratic — and it is also the single largest, most dynamic consumer of GPU memory in serving, the "KV" term in the memory budget from Chapter 2.

The KV cache is where most of serving's cleverness lives because it has an awkward property: its size is `2 × layers × kv_heads × head_dim × seq_len × batch × bytes`, and it grows *per token, per request*, unpredictably (you don't know how long an output will be). Managing that growth — packing it, sharing it, shrinking it, spilling it — is the subject of this chapter.

## 3.2 The size of the problem

A concrete calculation, because the numbers surprise people. For a 70B-class model with, say, 80 layers, 8 KV heads (GQA), head_dim 128, in FP16 (2 bytes): per token the KV cache is `2 × 80 × 8 × 128 × 2 ≈ 320 KB`. At 8K context that's ~2.5GB *per request*. Serving 40 concurrent 8K requests needs ~100GB of KV cache — more than an H100's entire 80GB HBM after weights. This is why KV cache, not weights, is usually what limits your concurrency, and why every technique below matters.

Two architectural facts from the pretraining series pay off enormously here:
- **GQA (grouped-query attention)** shrinks KV heads (e.g. 8 instead of 64), cutting KV cache ~8× versus multi-head attention — the main reason it's near-universal in 2026 models.
- **MLA (multi-head latent attention, DeepSeek)** compresses the KV cache into a low-rank latent, cutting it further. A model's attention design is, in large part, a *serving* decision — you inherit its KV footprint.

## 3.3 PagedAttention: the foundational trick

The problem PagedAttention (vLLM's signature contribution) solves: naive KV cache allocation reserves a contiguous block for each request sized to the *maximum* possible length, so a request that only generates 100 tokens still holds a 4K-or-8K reservation. This **internal fragmentation** wastes most of the KV memory and caps concurrency far below the theoretical limit.

PagedAttention borrows virtual memory from operating systems: split the KV cache into fixed-size **blocks** (pages), allocate them on demand as a sequence grows, and keep a block table mapping logical positions to physical blocks. The blocks needn't be contiguous. Result: near-zero fragmentation, so you fit far more concurrent requests in the same HBM — the technique that made high-throughput open serving possible and is now table stakes in every engine. You don't implement it; you get it. But understanding it explains why your effective concurrency is close to the memory-budget maximum instead of a fraction of it.

## 3.4 Prefix caching: reuse across requests

Many requests share a prefix — the same long system prompt, the same few-shot examples, the same document in a RAG pipeline, the same conversation history across turns. **Prefix caching** (automatic prefix caching / APC) stores the KV cache of shared prefixes and reuses it across requests instead of re-prefilling it every time. The payoff is enormous where it applies:

- **System prompts / few-shot templates** — a 2K-token system prompt prefilled once, reused for every request. Cuts TTFT and prefill compute dramatically.
- **Multi-turn conversations** — each turn reuses the cached KV of prior turns instead of re-reading the whole history.
- **RAG with shared documents** — the retrieved context's KV is cached across queries that hit the same document.

SGLang's RadixAttention generalizes this to a radix tree over all cached prefixes, matching and reusing any shared prefix automatically. Enable prefix caching whenever your traffic has shared prefixes (it almost always does); it's one of the highest-ROI flags in serving.

## 3.5 KV cache quantization

Since KV cache is often the binding constraint, shrinking it directly buys concurrency. **KV cache quantization** stores keys and values in FP8 or INT8 (or lower) instead of FP16, cutting KV memory 2× (FP8) or more, freeing that space for a bigger batch. For 32K+ contexts this cuts KV VRAM 30–50% and directly raises how many concurrent long-context requests you can hold.

The trade: KV quantization is generally *safer* than weight quantization for quality (the cache is a transient signal, not the learned weights), and FP8 KV cache is close to free in quality on most models. INT4 KV cache starts to bite on long-context recall and reasoning. Default: **FP8 KV cache is a near-free concurrency win on Hopper/Blackwell**; measure before going lower.

## 3.6 Offloading and hierarchical KV

When even paged, quantized KV won't fit, spill it down the memory hierarchy: **KV offloading** moves cold KV blocks from HBM to CPU RAM (or NVMe, or CXL/disaggregated memory) and pages them back when needed. Modern stacks (llm-d's hierarchical KV offloading, LMCache) do this transparently. It trades bandwidth (fetching from slower memory) for capacity (holding more/longer contexts than HBM allows). Worth it for: very long contexts, high-concurrency prefix reuse across a fleet, and multi-turn sessions parked between turns. Not worth the latency hit for tight-SLO interactive decode where the KV should stay in HBM.

## 3.7 The memory-management stack, assembled

In rough order of "turn it on":

1. **Paged KV cache** — on by default in every engine. Non-negotiable.
2. **Prefix caching** — on whenever traffic shares prefixes (system prompts, multi-turn, RAG). Huge TTFT win.
3. **FP8 KV cache** — near-free concurrency on modern hardware; measure quality on your evals.
4. **KV offloading / hierarchical** — when capacity binds and you can afford the fetch latency; fleet-scale prefix reuse.
5. **Architecture (GQA/MLA)** — inherited from the model, but a real selection criterion when you *choose* which model to serve: a GQA/MLA model is cheaper to serve at long context than an MHA one of equal quality.

## 3.8 Decisions

1. **Compute your KV-per-token and KV-per-request** early; it usually binds concurrency before weights do.
2. **Rely on paged KV** (default) and **enable prefix caching** whenever prefixes repeat — one of the highest-ROI settings.
3. **Turn on FP8 KV cache** on Hopper/Blackwell as a near-free concurrency win; validate on quality-sensitive evals before going to INT4.
4. **Offload KV** only when capacity binds and the SLO tolerates fetch latency; prefer it for long-context and fleet-wide prefix reuse.
5. **Treat KV footprint (GQA/MLA) as a model-selection criterion** — attention design is a serving cost you inherit.

Next: batching and scheduling — how the engine keeps that hard-won KV-cache space full of useful work without letting prefill and decode trample each other.
