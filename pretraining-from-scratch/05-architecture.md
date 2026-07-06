# Chapter 5 — Architecture

## 5.1 Conceptual picture

Architecture in 2026 is a story of **convergence plus efficiency engineering**. The core has been stable since ~2023: a decoder-only, pre-norm transformer with RMSNorm, SwiGLU MLPs, rotary position embeddings, and some form of KV-efficient attention. What has changed since sits on top of that skeleton: **MoE became the default at scale**, attention diversified into **latent (MLA), sparse, sliding-window, and linear/hybrid** variants aimed at long-context cost, and auxiliary tricks like **multi-token prediction** joined the mainstream. The dominant pattern across serious 2026 open-weight releases (DeepSeek V4, Kimi K2.x, Qwen3.5, GLM-5, MiniMax, Llama 4) is MoE, increasingly combined with hybrid attention.

Strategy for a first-timer: **copy a proven open recipe at your exact scale, change at most one thing, and validate that one thing on your scaling ladder.** Novel architecture is a research program, not a first pretraining run. The interesting genuine decisions are (a) dense vs. MoE and (b) the attention variant, where 2026 is honestly unsettled.

## 5.2 The modern dense baseline, component by component

This is the "Llama-3-class" recipe shared by nearly every strong dense model of 2024–2026 (Llama-3, Qwen, Mistral, Gemma). It is the correct default from 100M to ~70B:

- **Pre-norm residual blocks with RMSNorm.** Normalize *before* each sublayer (attention/MLP) — pre-norm is what makes deep transformers trainable without warmup fragility. RMSNorm over LayerNorm: cheaper, no mean-centering, equally good. A final RMSNorm before the output head. Increasingly common stability additions: **QK-norm** (RMSNorm on queries and keys before the attention dot-product — cheap, meaningfully reduces attention-logit blowups; used by Gemma, Qwen, OLMo 2) and occasionally extra post-norms (OLMo-2 style).
- **SwiGLU MLP.** Gated linear unit with SiLU: `down( silu(gate(x)) * up(x) )`. Universally adopted over vanilla GELU MLPs. Because it uses three matrices instead of two, hidden dim ≈ 8/3 × d_model (rounded to hardware-friendly multiples) keeps parameters matched.
- **Rotary position embeddings (RoPE).** Relative position via rotation of Q/K by position-dependent angles. The base frequency θ is a key knob and matters for context: 10K (old) → 500K (Llama-3) → 1M+ for long-context variants. Context extension (5.5) works by increasing θ and/or frequency-dependent scaling — plan θ with your final context target in mind.
- **Grouped-Query Attention (GQA).** Full query heads, K/V heads shared across groups (e.g., 32 Q heads, 8 KV heads). Near-MHA quality with 4–8× smaller KV cache. The default for dense models.
- **No biases** in linear layers or norms (don't help, cost a little).
- **Tied vs. untied embeddings:** tie input/output embeddings below ~1–2B to save parameters; untie at ≥7B.
- **Initialization:** scaled init (e.g., std ~0.02 or ~1/√d, with depth-scaled residual-output projections); under muP (Chapter 8), init and LR follow the muP prescription instead.
- **Optional logit stabilizers:** z-loss (~1e-4) or logit soft-capping on the output head (Chapter 8).

That is a strong, boring, correct model. Ship it for a first run.

## 5.3 Choosing the shape (depth, width, heads)

Given a parameter budget:

- **Aspect ratio.** There is a broad sweet spot for d_model/n_layers between roughly 64 and 128; extremely deep-thin or shallow-wide models train worse. Follow a known-good family's proportions.
- **Reference shapes:** 1B ≈ 16–24 layers × d2048; 8B ≈ 32 × 4096 (32 Q / 8 KV heads); 70B ≈ 80 × 8192 (64 Q / 8 KV).
- **Head dimension** 64–128; total attention dim = n_heads × head_dim = d_model for standard attention.
- **Hardware- and parallelism-friendliness.** Make every dimension divisible by your tensor-parallel degree and friendly to tensor cores (multiples of 64/128). Wider is friendlier to tensor parallelism; deeper is friendlier to pipeline parallelism but adds bubble (Chapter 7). Shape and parallelism plan should be chosen together.

## 5.4 Mixture of Experts (MoE)

**Concept.** Replace each MLP (in some or all blocks) with E parallel expert MLPs plus a learned router that sends each token to the top-k experts. Parameters scale with E; compute scales with k. You decouple *knowledge capacity* from *per-token cost*: DeepSeek-V3-class models carry ~671B parameters while spending ~37B per token; Kimi K2 is 1T total / 32B active. At fixed training FLOPs, MoE reliably beats dense — which is why essentially every serious large open-weight release in 2026 is MoE.

**The modern fine-grained recipe** (DeepSeek-style, widely copied):

- **Many small experts** (64–256+) rather than a few big ones — better specialization and capacity/compute trade-off.
- **Top-k routing** with k ≈ 2–8 depending on granularity, plus one or a few **shared experts** every token uses (capturing common patterns).
- **Load balancing without an auxiliary loss.** The router collapses (sends everything to a few experts) unless prevented. The classic fix was an auxiliary load-balancing loss, which fights the main objective; the 2026 standard (DeepSeek-V3) is **auxiliary-loss-free, bias-based balancing** — per-expert routing-bias adjustments that equalize load — sometimes with a tiny residual aux loss. Use the loss-free approach if your framework supports it.
- **Router in FP32**, sigmoid or softmax gating with normalization among selected experts.
- **Sparsity trend:** 2026 pushes higher total/active ratios (K2 ~32×) — more capacity per active FLOP, at the cost of more communication and router stress (Chapter 2.4).

**Costs.** MoE buys FLOP-efficiency with systems complexity: all experts occupy memory even though few fire (you need expert parallelism and all-to-all communication, Chapter 7); load imbalance wastes capacity; training has extra failure modes (router collapse, dead experts); MFU runs lower than dense; inference deployment is harder. **Recommendation: do your first run dense; adopt MoE on your second run** or whenever you target frontier-style efficiency at mid-to-large scale.

## 5.5 The 2026 attention landscape

For long contexts, standard attention's costs — quadratic compute, linear-in-length KV cache — dominate, and the field has responded with a menu. This is the most active architecture area right now:

- **Multi-head Latent Attention (MLA)** (DeepSeek). Compress K/V into a small per-token latent vector, decompress per head. Slashes KV cache below even GQA while *improving* quality in DeepSeek's ablations; requires RoPE handled via a decoupled path. Adopted well beyond DeepSeek (Kimi K2.x, GLM-5, Ling 2.5). A strong choice when KV-cache/long-context memory dominates.
- **Sliding-window / local-global interleaving** (Gemma-3, GPT-OSS, Mistral heritage). Most layers attend within a window (as small as 128–4K tokens); every Nth layer is global. Simple, robust, big KV savings.
- **Sparse attention** (DeepSeek DSA/NSA lineage). Learned token selection per query — near-full-attention quality at a fraction of long-context cost; appearing in V3.2/V4-class and GLM-5 models.
- **Linear/recurrent hybrids — the headline 2026 trend.** Interleave cheap linear-attention or state-space blocks (Gated DeltaNet, Mamba-2, Lightning Attention) with periodic full-attention (or MLA) layers, typically at 3:1 to 7:1 ratios. Qwen3-Next validated the pattern; Qwen3.5 promoted it to the flagship line (3:1 Gated DeltaNet : gated attention); Kimi Linear refines both halves (Kimi Delta Attention + gated MLA); Ling 2.5 runs Lightning Attention + MLA at 7:1 at 1T scale with ~3.5× the 32K-context throughput of same-size Kimi K2; NVIDIA's Nemotron-3 line interleaves Mamba-2 with sparse MoE. The intuition: cheap recurrent mixers handle most sequence modeling; periodic exact-attention layers preserve retrieval. Driven hard by agentic workloads' hunger for cheap long contexts.

**A cautionary tale worth knowing.** In late 2025, MiniMax shipped M2 *without* linear attention — reverting to standard attention — publicly noting linear attention was tricky in production: fine on ordinary prompts, weaker on reasoning and multi-turn/agentic tasks. Then Kimi Linear showed a well-designed hybrid *can* work. The lesson: hybrid linear attention is real and increasingly production-validated, but **not a free lunch** — it needs careful design and evaluation on reasoning and long-context retrieval evals, not just perplexity.

**Guidance:** GQA (or GQA + sliding-window interleave) for a first dense run. MLA if you're building a DeepSeek-style MoE or long-context memory is a priority. Hybrids are proven enough to adopt *from open reference implementations* if long-context efficiency is a primary product goal — but they add novelty risk; ablate on your ladder with long-context retrieval and reasoning evals.

## 5.6 Long-context handling

You extend context in a dedicated phase, not from scratch:

1. Train the bulk at short length (4K–8K) — attention cost and activation memory make full-length training wasteful, and most documents are short.
2. **RoPE scaling:** increase the base frequency and/or apply frequency-dependent interpolation (position interpolation, NTK-aware scaling, YaRN/ABF-style) so positions beyond the trained length remain in-distribution; then continue training on tens-to-hundreds of billions of tokens of genuinely long data at 32K → 128K → 256K+.
3. Architecture helps: sliding-window/MLA/hybrid layers keep extended-context cost manageable; context parallelism (Chapter 7) handles the activation memory.
4. Evaluate with long-range retrieval tests (needle-in-a-haystack, RULER, and harder long-form variants) — not just loss.

## 5.7 Auxiliary objectives and other components

- **Multi-token prediction (MTP).** Small extra head(s) predicting token t+2 (or t+2..t+4) during training. Densifies the training signal (small quality gains) and, retained at inference, powers self-speculative decoding (DeepSeek-V3; StepFun's MTP-3 predicts three ahead). Cheap, increasingly standard at scale; skip for a minimal first run.
- **muP (Maximal Update Parametrization).** Technically a parametrization, but it's an architecture-level decision: init scales and per-layer learning rates set so optimal hyperparameters transfer from small proxies to the big model — tune LR on a 40M model, use it at 8B. Used in various forms by many labs; strongly recommended if you will do any hyperparameter search (Chapter 8), with published standard-parametrization recipes as the fallback.
- **What's gone:** learned absolute position embeddings, post-norm, dropout (pretraining uses none — you see each token roughly once), encoder-decoder for general LLMs, and the loss-hurting auxiliary-loss MoE balancing.

## 5.8 Decisions summary

| Scale / goal | Recipe |
|---|---|
| ≤1B (learning) | Dense Llama-3-style: GQA, 32–64K vocab, tied embeddings, 4K context |
| 7–8B | Same skeleton: 128K vocab, untied, QK-norm, consider MTP; 8K context then extend |
| 70B dense | Proven Llama-3-70B shape; nothing exotic |
| Frontier-style efficiency | Fine-grained MoE (DeepSeek/Kimi-family recipe) + MLA or a validated hybrid, MTP, FP8 — plus significantly more systems investment (Chapters 7–8) |
