# Midtraining and Continued Pretraining: The Missing Middle

*The stage between a base model and a post-trained one — annealing, long-context extension, continued pretraining, domain adaptation, tokenizer surgery, and reasoning injection.*

*Compiled July 2026. The bridge between [Training an LLM From Scratch](../pretraining-from-scratch/README.md) and [Post-Training an LLM](../post-training-from-scratch/README.md). Assumes you have read (or could have written) those two — you know what a base model is, what a data mixture is, what a learning-rate schedule does, and what SFT and RL are for. Covers all scales, oriented around cloud GPU clusters.*

---

## Why this series exists

The pretraining series stops at a base model and explicitly punts on "midtraining data design." The post-training series starts from a base model and punts on it too. That punt is this series. In every real 2026 pipeline there is a stage — or a stack of stages — that is neither vanilla next-token prediction on web text nor SFT/RL on demonstrations. It has its own name now (**midtraining**), its own literature, and its own failure modes, and getting it wrong wastes a large fraction of the value locked in your base model.

Concretely, midtraining is where you: **anneal** the learning rate down while swapping the web-heavy pretraining mixture for a curated, capability-dense one; **extend the context window** from 4–8K to 128K+; **continue pretraining** on a domain corpus (code, medicine, law, a new language) without destroying the general model; **operate on the tokenizer** when your target domain tokenizes badly; and **inject reasoning and instruction-following priors** early, because the pretraining/post-training boundary has moved and the "base" model shipped in 2026 is already half-cooked for chat and CoT.

## How to read this report

Every chapter is **layered**, like the companion series: it opens with the *conceptual picture* — what the thing is and why it exists — then descends into the *practitioner layer*: concrete token budgets, learning rates, mixture ratios, and the specific decisions you will face. If you want the map, read the opening section of each chapter and the "Decisions" summaries. If you are about to spend a cluster-week on a continued-pretraining run, read everything.

The economics sit between the two neighbors. Midtraining is far cheaper than pretraining (tens to hundreds of billions of tokens, not trillions) but far more expensive than post-training (a real long-context or CPT run is a multi-node, multi-day job, not a $30 SFT). The characteristic failure is also in between: not the loud divergence of pretraining nor the silent reward-hacking of post-training, but **quiet forgetting** — a model that gains the new capability and loses two points of MMLU you don't notice until a user does.

## Chapters

| # | File | Contents |
|---|---|---|
| 1 | [The Big Picture](01-big-picture.md) | What midtraining is, the three moved boundaries, where each technique sits, the core mental models |
| 2 | [Annealing and the Decay Phase](02-annealing-and-decay.md) | WSD vs cosine, the decay-phase mixture swap, microanneals, data-quality upweighting, why annealing is a capability lever |
| 3 | [Continued Pretraining](03-continued-pretraining.md) | CPT vs midtraining vs fine-tuning, the re-warmup problem, the stability gap, replay, when CPT beats SFT |
| 4 | [Domain Adaptation](04-domain-adaptation.md) | Code/medical/legal/finance case studies, token budgets, LR, reading-comprehension conversion, the specialization-vs-size tradeoff |
| 5 | [Long-Context Extension](05-long-context-extension.md) | RoPE and base-frequency scaling, YaRN/NTK/LongRoPE, progressive multi-stage recipes, long-context data construction, evaluation |
| 6 | [Tokenizer and Vocabulary Surgery](06-tokenizer-surgery.md) | When to touch the tokenizer, vocabulary extension, embedding initialization, the un-embedding, multilingual and domain fertility |
| 7 | [Capability and Reasoning Injection](07-capability-and-reasoning-injection.md) | Front-loading reasoning, CoT and instruction data in midtraining, the moved cold-start boundary, math/code capability seeding |
| 8 | [Forgetting and Stability](08-forgetting-and-stability.md) | Measuring catastrophic forgetting, replay ratios, the stability gap, LR re-warmup, LoRA vs full CPT for retention |
| 9 | [Frameworks and Infrastructure](09-frameworks-and-infrastructure.md) | What changes from the pretraining stack — NeMo, Megatron, torchtitan, Axolotl, Unsloth — checkpointing, data pipelines, cost |
| 10 | [Evaluation](10-evaluation.md) | The before/after harness, forgetting probes, long-context evals, domain evals, contamination, when to promote a checkpoint |
| 11 | [Playbooks by Scale](11-playbooks-by-scale.md) | End-to-end recipes: single-GPU domain LoRA, one-node 8B CPT, long-context extension, frontier midtraining stage |
| 12 | [Failure Modes](12-failure-modes.md) | Loss spikes on re-warmup, silent forgetting, long-context degradation, tokenizer drift, mixture cliffs, decay-phase overfit |
| 13 | [Essential Reading](13-essential-reading.md) | The must-read sources, as an ordered sequence |
| 14 | [Reading List and References](14-reading-list-and-references.md) | A curated path deeper, plus the full source list |

## Scope

This series covers everything between "you have a base model" and "you begin alignment." It assumes the pretraining run itself is done (or you downloaded someone's base model) and that post-training happens after. The boundaries are deliberately fuzzy — Chapter 1 maps exactly where midtraining bleeds into pretraining's annealing phase on one side and into SFT's cold-start on the other, because in 2026 those boundaries are contested and the best teams treat the whole span as one continuous curriculum.

## The through-line

Midtraining rewards **surgical restraint**. You are editing a model that already works, and every edit trades some of what it knows for some of what you want it to know. The teams that win here are not the ones who continue-pretrain on the most domain tokens; they are the ones who measure the trade precisely — a held-out general eval and a domain eval, run on every checkpoint — and who reach for the smallest intervention that moves the domain metric without moving the general one. Anneal gently, replay generously, extend context in stages, touch the tokenizer only when you must, and never promote a checkpoint you have not measured against what it used to be able to do.
