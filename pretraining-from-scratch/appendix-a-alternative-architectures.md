# Appendix A — Alternative Architectures

*A companion to Chapter 5. The main series is deliberately transformer-centric, because the decoder-only transformer is what you should pretrain in 2026. This appendix covers the credible alternatives — state-space models, hybrid attention-SSM designs, and diffusion language models — what they buy, what they cost, and when they should actually change your architecture decision.*

## A.1 Why alternatives exist at all

Every alternative to the transformer attacks one of two costs baked into attention: **quadratic compute in sequence length** (every token attends to every other) and **the KV cache** (memory that grows linearly with context and, at serving time, is usually the binding constraint — inference series Ch 3). The transformer pays these costs in exchange for exact, content-based recall over the whole context. The alternatives trade some of that recall for constant-memory or parallel generation, and the interesting 2026 question is no longer "will X replace the transformer" (it hasn't) but "which layers of a model, or which workloads, are better served by X."

## A.2 State-space models (Mamba and kin)

**What they are.** SSMs (S4 → Mamba → Mamba-2) process sequences recurrently through a fixed-size hidden state with input-dependent (selective) dynamics. Compute is linear in sequence length; generation needs only the constant-size state — **no KV cache**. Training parallelizes via convolutional/scan formulations, so they train efficiently too.

**What they buy.** Linear-time long sequences, constant-memory decode (a huge serving win — the entire KV-cache chapter of the inference series largely disappears), and strong throughput at long context.

**What they cost.** A fixed-size state is a lossy summary: pure SSMs are measurably worse at **exact recall** — retrieving a precise fact from far back in context, in-context learning that depends on verbatim copying, and needle-style retrieval. The 2025–2026 memory-recall literature confirms the gap is fundamental to the architecture, not a training artifact. For tasks where the model must *look things up* in its context, attention wins.

## A.3 Hybrids: the practical winner

The field's resolution of that trade is not either/or but **interleaving**: mostly SSM/linear-attention layers for efficiency, with a minority of full-attention layers for exact recall. **Jamba** (AI21) was the first production-grade example — Transformer + Mamba + MoE in one stack, 256K context — and the pattern now appears across the industry (Jamba-1.5, IBM Granite 4, Zamba, various frontier-lab hybrids). Typical ratios put attention in roughly 1-in-4 to 1-in-8 layers; the memory-recall research shows a small attention fraction recovers most of the recall gap while keeping most of the efficiency gain.

**When a hybrid should change your Chapter 5 decision:** your product is dominated by *very* long contexts (hundreds of K to millions of tokens) where KV-cache economics break the serving budget, or by high-throughput long-context serving where the constant-state decode pays directly. **When it shouldn't:** general-purpose models at ordinary context lengths — the dense transformer's ecosystem advantage (kernels, serving engines, tooling, hiring, the entire inference series) still outweighs the efficiency delta, and every framework/serving assumption in this knowledge base holds without translation. Hybrids are a considered bet for long-context-first products, not a new default.

## A.4 Diffusion language models

**What they are.** Instead of generating left-to-right one token at a time, diffusion LMs (dLLMs) start from a fully-masked/noised sequence and iteratively denoise **all positions in parallel**, refining the whole output over a fixed number of steps.

**What they buy: speed.** Because each denoising step updates many tokens at once, generation escapes the sequential-decode bottleneck that makes autoregressive inference memory-bound (inference series Ch 2). The 2025–2026 production proof points: Inception's **Mercury** (the first commercial dLLM) generates 1,000+ tokens/second on a single H100 — 5–10× comparable AR models; **Gemini Diffusion** runs ~1,479 tok/s, ~5× its Flash-Lite sibling; Google's DiffusionGemma claims ~4× faster generation. By mid-2026 dLLMs power real-time code completion and latency-critical workloads that AR models can't serve.

**What they cost.** Quality still trails at the top end — Mercury 2's stated trade is ~10× speed for **5–15% lower quality on reasoning tasks** (near-parity on structured output, translation, classification) — and the training recipe, tooling, and serving stack are young and largely separate from the AR ecosystem this knowledge base describes. Streaming UX also changes (text appears in refining waves, not a growing stream).

**The 2026 position:** dLLMs are a *workload-specific* choice — ultra-low-latency completion, high-volume structured generation — usually consumed as a vendor model rather than pretrained in-house. Pretraining one yourself is frontier-research territory; adapting an existing AR model to diffusion decoding (an active research line) is the plausible middle path.

## A.5 Linear attention and efficient-attention variants

A large family (linear attention, GLA, RWKV, RetNet, sliding-window + global hybrids) linearizes or localizes attention for the same linear-cost goal as SSMs — the efficient-attention surveys catalogue dozens. Two practical notes: first, **sliding-window attention in most layers + a few global layers** (Gemma-style) is the *quietly mainstream* efficiency hybrid already inside "ordinary" transformers, and you likely adopt it via Chapter 5's baseline without ceremony; second, **distilling a pretrained transformer into a linear-attention hybrid** (rather than pretraining one from scratch) matured through 2026 as the low-cost route to these architectures. Treat the exotic variants as serving-cost optimizations to evaluate against quantization and caching (inference series Ch 6, 3), not as pretraining decisions.

## A.6 Decisions

1. **Pretrain a dense (or MoE) transformer by default** — the Chapter 5 guidance stands; the ecosystem advantage still dominates for general-purpose models.
2. **Consider an attention-SSM hybrid** (Jamba-class, ~1:4–1:8 attention ratio) only when very-long-context economics are the product — the KV-cache savings are real, the exact-recall gap is real, and the small attention fraction is what reconciles them.
3. **Treat diffusion LMs as a workload-specific serving choice** (latency-critical completion, structured generation, a 5–15% reasoning-quality trade), consumed from vendors — not a pretraining default.
4. **Adopt sliding-window + global attention as ordinary practice** inside your transformer; evaluate more exotic linear-attention variants as serving optimizations, preferably via distillation from your AR model rather than from-scratch pretraining.
5. **Re-evaluate yearly** — this is the fastest-moving architecture question in the field, and the hybrid share of new frontier models is rising; the default may genuinely shift.
