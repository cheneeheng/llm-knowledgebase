# Chapter 1 — The Big Picture

## 1.1 What pretraining actually is

Conceptually, pretraining is the simplest part of the whole LLM lifecycle. You take a randomly initialized transformer, feed it an enormous stream of text (and increasingly code, math, and multimodal data), and train it with a single objective: **causal language modeling (CLM)** — predict the next token given everything before it. The loss is cross-entropy averaged over positions:

$$\mathcal{L} = -\frac{1}{T}\sum_{t=1}^{T} \log P_\theta(x_t \mid x_{<t})$$

That's it. No labels, no reward models, no human preferences. Every document is its own supervision. Everything the model "knows" — grammar, facts, reasoning patterns, coding ability, in-context learning, the raw material that post-training later shapes into a helpful assistant — is a side effect of getting good at this one objective at scale.

This is worth internalizing because it explains why data and scale dominate everything else. You are not teaching the model tasks. You are **compressing a corpus into weights**, and the model's capabilities are whatever compression at that scale happens to produce. Scaling laws (Chapter 2) are, at bottom, statements about how this compression improves as you add parameters and tokens.

The paradigm has fully converged: every frontier model in 2026, open or closed, dense or MoE, is a decoder-only autoregressive transformer (or transformer-hybrid) trained with next-token prediction, sometimes augmented with multi-token prediction as an auxiliary objective.

The simplicity of the objective is deceptive, because everything *around* the objective is hard. Pretraining is the most expensive, most operationally brutal phase of LLM development. A serious run consumes thousands to millions of GPU-hours, and the difference between a well-executed run and a sloppy one — in data curation, hyperparameters, parallelism efficiency, and fault handling — is easily a 2–5× difference in effective compute. You cannot patch a bad pretraining run afterward: post-training can shape a model's behavior but cannot inject capabilities the base model never learned.

## 1.2 Anatomy of a modern pretraining run

A 2026-era pretraining run is not one homogeneous pass over data. It is structured as **phases**:

**Phase 1 — Main pretraining (the "stable phase", ~80–90% of tokens).** Large-scale, diverse, web-heavy data at moderate context length (4K–8K tokens), with the learning rate at or near its peak. This is where the model learns language, world knowledge, and general competence. For an 8B model this might be 10–15T tokens; for a frontier MoE, 15T+.

**Phase 2 — Midtraining / annealing (~10–20% of tokens).** As the learning rate decays toward zero, the data mixture shifts toward high-quality, high-density sources: curated educational content, math, code, textbooks, and increasingly synthetic data. Because the learning rate is decaying, what the model sees last gets "baked in" disproportionately. This phase — variously called the *decay phase*, *annealing phase*, or *midtraining* — is where a surprising fraction of benchmark performance is won, and its data recipe is the part labs guard most closely. It is also where reasoning-relevant data gets emphasized (Chapter 10). It pairs naturally with Warmup-Stable-Decay learning-rate schedules (Chapter 8).

**Phase 3 — Long-context extension.** Most of training happens at 4K–8K context because attention cost and activation memory grow with sequence length, and most documents are short anyway. Near the end, you continue training on a smaller number of tokens (tens to hundreds of billions) at progressively longer contexts — 32K, then 128K, sometimes 256K–1M — with adjusted RoPE parameters and a data mix enriched in genuinely long documents. This is how models advertise "128K context" without paying long-context prices for the whole run. (Kimi K2, for example, pretrained at a 4,096-token window and did a long-context activation stage near the end.)

**Phase 4 — "Midtraining" as its own named stage (a fuzzy, growing category).** Some 2026 labs treat the injection of large volumes of reasoning traces, synthetic instruction-like data, and domain data — after bulk pretraining, before formal post-training — as a distinct stage with its own recipe. This report covers it where it bleeds into pretraining decisions (Chapters 4 and 10).

After Phase 3 you have a **base model**. It completes text; it doesn't chat, follow instructions, or refuse anything. Everything after this point is post-training and outside our scope.

## 1.3 The mental model: five coupled workstreams

Practically, a pretraining project decomposes into five parallel workstreams, and this report has a chapter for each:

- **Data system** — sourcing, cleaning, dedup, tokenization, mixing, and streaming trillions of tokens without stalling the GPUs. Usually the largest engineering effort by headcount at serious labs. (Chapter 4)
- **Model system** — the architecture, its shape, and its initialization. (Chapter 5)
- **Optimization system** — optimizer, learning-rate schedule, batch size, numerical precision, stability controls. (Chapter 8)
- **Distributed system** — how the model and data are split across hundreds or thousands of accelerators, and how they communicate. (Chapters 6–7)
- **Operational system** — checkpointing, failure detection, restart, monitoring, and evaluation on a cluster that *will* have hardware failures. (Chapters 3, 9, 12)

Pretraining is hard not because any one of these is hard, but because they are **coupled**. Batch size interacts with learning rate, which interacts with parallelism strategy, which interacts with interconnect bandwidth, which interacts with data-loading throughput. Change one and you often re-tune the others. The scaling-ladder methodology (Chapter 2) exists precisely to find a configuration where all five are jointly healthy at small scale before committing to the expensive large run.

A useful compression of the whole report: **data determines what your model can know, architecture determines the ceiling of what it can compute, systems determine how much of your money becomes gradient updates instead of heat, and operations determine whether you actually finish.**

## 1.4 What "from scratch" costs, roughly

Calibrate expectations before going deep. Using the standard approximation that training a dense transformer costs about **6 × parameters × tokens** FLOPs (Chapter 2), and that a good cluster realizes ~40% of peak GPU throughput (MFU — "model FLOPs utilization"):

| Model | Tokens | Compute (FLOPs) | H100-hours @ 40% MFU | Rough cloud cost @ $2.5/H100-hr |
|---|---|---|---|---|
| 1B | 25B (Chinchilla-optimal) | 1.5 × 10²⁰ | ~105 | ~$300 |
| 1B | 100B | 6.0 × 10²⁰ | ~420 | ~$1.1K |
| 1B | 1T (overtrained) | 6.0 × 10²¹ | ~4,200 | ~$11K |
| 8B | 5T | 2.4 × 10²³ | ~168,000 | ~$420K |
| 8B | 15T | 7.2 × 10²³ | ~505,000 | ~$1.26M |
| 70B | 15T | 6.3 × 10²⁴ | ~4.4M | ~$11M |
| ~700B MoE (~40B active) | 15T | ~3.6 × 10²⁴ | ~2.5M | ~$6.3M |

*An H100 delivers ~990 TFLOPs peak dense BF16; at 40% MFU that's ~400 TFLOPs sustained. FP8 and Blackwell-class hardware change the arithmetic (Chapters 3, 7).*

Three lessons jump out:

1. **MoE is a FLOP-efficiency play.** For sparse models, the *N* in 6ND is **active** parameters per token, not total. A ~700B-total MoE with ~40B active costs roughly what a 40B dense model costs per token while holding far more capacity. That asymmetry — total parameters set memory cost and knowledge capacity, active parameters set compute cost — is the entire reason the frontier is almost all MoE now (Chapter 5).
2. **The gap between "learning run" and "frontier run" is four to five orders of magnitude in cost** — exactly why you validate everything at the cheap end first.
3. **A 1B learning run costs less than a conference ticket.** There is no excuse for making your first mistakes at 8B.
