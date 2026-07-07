# Chapter 2 — The Anatomy of Inference

## 2.1 Conceptual picture

This is the most important chapter in the series, because every optimization later is a corollary of one fact: **generating text token-by-token is memory-bandwidth-bound, not compute-bound.** If you understand why, you can derive batching, quantization, KV-cache paging, and speculative decoding from first principles. If you don't, they're a bag of tricks.

Here is the fact, concretely. To generate one token in the decode phase, the GPU must read *every weight of the model* out of high-bandwidth memory (HBM) and multiply it by a single token's activation vector. For a 70B model in FP16, that's 140GB of weights read to produce *one token*. The multiply itself is trivial for a modern GPU — it finishes long before the next batch of weights arrives from memory. So the GPU spends decode *waiting for memory*, with its expensive matrix-multiply units mostly idle. The token rate is set by how fast weights stream from HBM, i.e. by **memory bandwidth**, divided by model size.

Rough decode-speed estimate for a single request: `tokens/sec ≈ memory_bandwidth / model_size_in_bytes`. A 70B FP16 model (140GB) on an H100 (3.35 TB/s) tops out around `3350/140 ≈ 24 tokens/sec` for a single stream — and that's a *ceiling*, before overheads. This back-of-envelope is the single most useful calculation in serving; do it before you promise anyone a latency number.

## 2.2 Why batching is the master knob

Here is the beautiful consequence. When you read those 140GB of weights, you can multiply them by *one* token's activations or by *many* tokens' activations (from different requests) at almost the same memory cost — because the weights are read once and reused across the batch. Compute goes up with batch size; memory traffic (for weights) does not. So **batching converts wasted memory-bound idle time into useful compute**, dramatically increasing throughput per GPU at almost no latency cost — until the batch gets big enough that you become compute-bound or run out of memory for KV cache.

This is why:
- **Throughput scales with batch size** (up to a point), and cost-per-token falls with it. Serving one user on an H100 is wildly inefficient; serving 50 concurrent users is where the economics work.
- **Latency per user is roughly flat** as batch grows (in the memory-bound regime) — you're using idle time, not stealing from anyone.
- **The whole serving stack is built to keep the batch as full as possible** — continuous batching (Chapter 4) exists to never let a memory read go to waste on a half-empty batch.

The regime flips at large batch or long context: eventually the arithmetic catches up to the memory read and you become compute-bound, at which point adding batch stops helping. But most single-model serving lives in the memory-bound regime where batching is close to a free lunch.

## 2.3 Prefill vs decode, quantified

The two phases sit on opposite sides of the roofline:

| | Prefill | Decode |
|---|---|---|
| Tokens processed per pass | All prompt tokens (hundreds–thousands) | One (per request) |
| Bottleneck | **Compute** (matrix units saturated) | **Memory bandwidth** (weights streamed per token) |
| Arithmetic intensity | High (many tokens reuse each weight) | Low (one token per weight read) |
| Sets the metric | TTFT | TPOT |
| Scales with batch? | Already compute-saturated | Yes — batching is the win here |
| Wants hardware with | High FLOPS | High HBM bandwidth |

**Arithmetic intensity** — FLOPs per byte read — is the formal way to say this. Prefill has high intensity (each weight read does work for many tokens), so it sits on the compute-bound side of the roofline. Decode with batch size 1 has intensity ~1 (each weight read does work for one token), deep in the memory-bound regime. *Batching decode raises its arithmetic intensity* — with batch size B, each weight read serves B tokens — which is precisely why batching moves decode rightward toward the compute-bound regime where the GPU is efficient.

The practical upshot, which drives Chapter 4: prefill and decode want different things and interfere when mixed naively (a big prefill blocks ongoing decodes, spiking everyone's TPOT). The solutions — chunked prefill, prefill/decode disaggregation — all follow from this tension.

## 2.4 The memory budget

At serving time your GPU HBM holds three things, and they compete:

1. **Model weights** — fixed: `params × bytes_per_param`. 70B in FP16 = 140GB; in FP8 = 70GB; in INT4 = 35GB. This is why quantization (Chapter 6) is about *fitting the model and freeing room for batch*, not mainly about speed.
2. **KV cache** — grows with `batch_size × sequence_length`. This is the *variable* that determines how many concurrent requests (and how long their contexts) you can hold. Often the binding constraint (Chapter 3).
3. **Activations and overhead** — transient working memory, comparatively small.

The serving memory equation: `HBM = weights + KV_cache + overhead`. Weights are fixed; overhead is small; so **KV cache gets whatever's left, and that determines your maximum batch size and context length.** Every memory optimization — quantizing weights, quantizing the KV cache, paging it, offloading it — is ultimately about fitting more concurrent requests (bigger batch = better throughput/cost) into the HBM you have. Hold this equation in your head; it's the budget every later chapter spends.

## 2.5 What this means for the rest of the series

Every technique ahead is now derivable:

- **Continuous batching (Ch 4)** — keep the batch full so no memory read is wasted. Direct consequence of 2.2.
- **PagedAttention / KV paging (Ch 3)** — pack the KV cache tightly so you fit more requests in the leftover HBM. Direct consequence of 2.4.
- **Quantization (Ch 6)** — shrink weights and KV cache to fit more batch and stream fewer bytes per token (which *also* speeds decode, since decode is bytes-bound). Consequence of both.
- **Speculative decoding (Ch 7)** — the one trick that beats the memory-bound ceiling: verify several tokens per weight-read pass, raising tokens-per-memory-read above 1 without more requests. Consequence of 2.1.
- **Disaggregation (Ch 4)** — put compute-bound prefill and memory-bound decode on hardware suited to each. Consequence of 2.3.

## 2.6 Decisions

1. **Estimate single-stream decode speed** as `bandwidth / model_bytes` before promising latency; it's your ceiling.
2. **Treat batch size as the master throughput/cost knob** — serving is only economical batched; size your system to keep batches full.
3. **Budget HBM as `weights + KV + overhead`** and recognize KV cache (hence max concurrency) usually binds — this is what quantization buys back.
4. **Expect prefill and decode to interfere**; the scheduling chapter exists to manage it.
5. **When someone proposes an optimization, classify it**: does it fill the batch, shrink the bytes, or beat the one-token-per-read ceiling? If it does none of those, it won't help a memory-bound system.

Next: the KV cache — the variable in the memory budget, and the object half of serving's cleverness is spent managing.
