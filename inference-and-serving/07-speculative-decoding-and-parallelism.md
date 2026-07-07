# Chapter 7 — Speculative Decoding and Parallelism

## 7.1 Conceptual picture

Speculative decoding is the one trick that beats the fundamental limit from Chapter 2. Recall: decode reads *all* the model's weights from HBM to produce *one* token, and that memory read is the bottleneck while the compute units sit idle. Speculative decoding exploits that idle compute: a small, fast **draft model** cheaply guesses the next several tokens, and the big **target model** verifies all of them in a *single* forward pass (the same pass that would have produced one token can *check* many, because verification is parallel over the guessed tokens). If the draft guessed right, you got several tokens for one target-model memory read. You've raised tokens-per-memory-read above 1 without adding requests — the only technique that does.

Crucially, speculative decoding is **exact**: the verification step accepts a draft token only if it matches what the target model would have sampled, so the output distribution is *identical* to running the target alone. It's a pure latency/throughput win with zero quality cost — which is why it's ubiquitous in 2026 for latency-sensitive serving.

## 7.2 How verification stays exact

The draft proposes tokens t1…tk. The target model runs one forward pass over the prompt + all k drafted tokens, producing its own probability distribution at each position. A rejection-sampling scheme accepts the longest correct prefix of the draft (each token accepted with a probability that preserves the target's distribution) and corrects the first wrong token from the target's own distribution. Net effect: the sequence is distributed exactly as if the target generated it alone, but you often advance several tokens per target pass. The **acceptance rate** — how many drafted tokens survive on average — is the whole ballgame: high acceptance means big speedup, low acceptance means you did extra draft work for little gain.

## 7.3 The methods

| Method | Draft source | Notes |
|---|---|---|
| **Draft model (classic)** | A separate small model (e.g. 1B drafting for 70B) | Simple; needs a well-matched small model; draft must be much cheaper than target |
| **EAGLE / EAGLE-2/3** | A lightweight head trained on the target's own features | Highest acceptance rates in 2026; the go-to for serious speculative serving |
| **Medusa** | Multiple extra decoding heads on the target predicting several future tokens | No separate model; moderate acceptance; simple to bolt on |
| **n-gram / prompt lookup** | Copy likely continuations from the prompt/context | Free (no model); great for repetitive/structured output, RAG, code (long verbatim spans) |
| **Self-speculation / layer-skip** | The model drafts with a subset of its own layers | No extra params; hardware-friendly |

**EAGLE-family methods** dominate 2026 for general speculative serving because their acceptance rates are highest — they draft using the target's own hidden features, so the guesses align well. **n-gram/prompt-lookup** is a free win specifically when output repeats input (summarization that quotes, code that echoes identifiers, structured output) — no model needed, just copy and verify. Modern engines (vLLM, SGLang, TensorRT-LLM) support several of these as config; you enable and tune, you don't implement.

## 7.4 When speculative decoding helps (and when it doesn't)

The economics depend on regime (Chapter 2):

- **Low batch / latency-critical (memory-bound, idle compute)** — speculative decoding shines. There's spare compute to burn on verification, and cutting a single user's latency is the goal. Interactive single-user or low-concurrency serving benefits most.
- **High batch (compute-bound, saturated)** — the benefit shrinks or reverses. When the GPU is already compute-saturated by a large batch, there's no idle compute for verification, and the extra draft work competes with real requests. At very high concurrency, speculative decoding can *hurt* throughput.

So the rule: **speculative decoding is a latency optimization for the memory-bound regime, not a throughput optimization for saturated fleets.** Use it for low-latency, low-to-moderate-concurrency serving (including reasoning models, where long single-user decodes benefit hugely — Chapter 8); be cautious about enabling it under high batch, and measure goodput with it on and off. It composes with everything else (quantization, batching) but its benefit is regime-dependent in a way those aren't.

## 7.5 Parallelism for serving

Distinct from training parallelism (pretraining series, Chapter 7), because serving optimizes latency/memory-fit, not training throughput.

- **Tensor parallelism (TP)** — split each layer's weights across GPUs; every GPU holds a shard and they communicate every layer. For serving, TP is the primary way to (a) *fit* a model too big for one GPU and (b) *speed up* decode by splitting the per-token memory read across GPUs' combined bandwidth. Must live inside a fast interconnect (NVLink) — the per-layer collectives are latency-critical. Default for multi-GPU single-node serving. TP=2/4/8 within a node.
- **Pipeline parallelism (PP)** — split layers across GPUs/nodes; used to fit very large models across *nodes* when they exceed a single node's memory. Adds latency (pipeline bubble) so it's a fit-necessity, not a latency win; prefer TP within a node and PP only across nodes.
- **Expert parallelism (EP)** — for MoE models, distribute experts across GPUs. Essential for serving large MoE (DeepSeek-scale) models efficiently; the routing makes serving MoE its own discipline (uneven expert load, all-to-all communication).
- **Data parallelism / replication** — just run N independent replicas behind a load balancer. This is how you scale *throughput* (more requests/sec) once a single replica is tuned — the horizontal-scaling story of Chapter 11.

The serving sizing rule: **use the least parallelism that fits the model with room for a healthy KV cache and batch.** Every GPU you add to a replica for TP adds communication overhead and cost; you split across GPUs to fit and to hit latency, then *replicate* whole replicas to scale throughput. Chapter 10 does the fit math.

## 7.6 Decisions

1. **Enable speculative decoding for latency-sensitive, low-to-moderate-concurrency serving** (including reasoning models); it's exact, so no quality cost.
2. **Prefer EAGLE-family drafts** for general use (highest acceptance); add **n-gram/prompt-lookup** for repetitive/structured/RAG output as a free win.
3. **Measure acceptance rate and goodput with speculation on vs off** — it can hurt at high batch where compute is saturated.
4. **Use tensor parallelism within a node** to fit and speed models, kept inside NVLink; **pipeline parallelism only across nodes** when a model exceeds one node.
5. **Use expert parallelism for large MoE**, and **replicate whole replicas** to scale throughput — split to fit, replicate to grow.

Next: reasoning models, which stress every technique in this series because they turn a short answer into a long, expensive decode.
