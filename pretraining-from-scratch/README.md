# Training an LLM From Scratch: The Complete Pretraining Playbook

*From cluster procurement to your first trillion tokens — everything up to (but not including) post-training.*

*Compiled July 2026. Assumes you know what a transformer is (attention, MLP blocks, residual stream, backprop) but have never trained an LLM. Covers all scales, oriented around cloud GPU clusters. Synthesized from two independent drafts (see `../tmp/`) plus their source lists.*

---

## How to read this report

Every chapter is **layered**. It opens with the *conceptual picture* — what the thing is and why it exists — then descends into the *practitioner layer*: concrete numbers, configs, and the specific decisions you will face. If you want the map, read the opening section of each chapter and the "Decisions" summaries. If you are about to spend money on GPUs, read everything, twice.

A recurring theme: **pretraining is an exercise in not wasting compute.** A serious run costs hundreds of thousands to tens of millions of dollars, and a single bad decision made on day one (wrong tokenizer, wrong data mix, wrong learning rate) can silently cost a large fraction of final quality with no way to fix it short of restarting. Almost everything below is ultimately about de-risking one long, expensive, irreversible bet.

## Chapters

| # | File | Contents |
|---|---|---|
| 1 | [The Big Picture](01-big-picture.md) | What pretraining is, the phases of a modern run, the five coupled workstreams |
| 2 | [Scoping and Scaling Laws](02-scoping-and-scaling-laws.md) | 6ND, Chinchilla and why nobody follows it, MoE scaling, the data wall, the scaling ladder, worked cost examples |
| 3 | [Infrastructure](03-infrastructure.md) | Accelerators, interconnect, topology, storage, checkpointing, fault tolerance, orchestration, procurement |
| 4 | [Data](04-data.md) | Sources, the refinement pipeline, deduplication, tokenization, mixtures and curricula, synthetic data, packing, legal |
| 5 | [Architecture](05-architecture.md) | The modern dense baseline, model shape, MoE, the 2026 attention landscape, long context, auxiliary objectives |
| 6 | [Frameworks](06-frameworks.md) | Megatron-Core, torchtitan, NeMo, DeepSpeed, MaxText, research trainers — which to pick at which scale |
| 7 | [Parallelism and Systems](07-parallelism-and-systems.md) | The memory budget, the five parallelism axes, composing configs by scale, activation recomputation, precision (BF16/FP8/FP4), kernels, MFU |
| 8 | [Optimization](08-optimization.md) | AdamW vs Muon/MuonClip, LR schedules (cosine vs WSD), batch size, muP, initialization, stability controls |
| 9 | [Running the Job and Evaluating It](09-running-and-evaluating.md) | Pre-flight checklist, monitoring, base-model evaluation, reproducibility |
| 10 | [Thinking Models](10-thinking-models.md) | What reasoning models change about pretraining specifically; the moved pretraining/post-training boundary |
| 11 | [Playbooks by Scale](11-playbooks-by-scale.md) | Concrete end-to-end recipes: 1B, 8B, 70B dense, frontier MoE |
| 12 | [Failure Modes](12-failure-modes.md) | The ways runs die — divergence, SDC, stragglers, data starvation, checkpoint rot, silent recipe bugs |
| 13 | [Essential Reading](13-essential-reading.md) | The twelve must-read sources, as an ordered reading sequence |
| 14 | [Reading List](14-reading-list.md) | A curated path deeper, by topic |
| 15 | [References](15-references.md) | Full source list with provenance notes |

## Scope

This report stops at the end of pretraining. You finish with a **base model**: it completes text; it does not chat, follow instructions, or refuse anything. Supervised fine-tuning, RLHF, RLVR, preference optimization, reward modeling — all of post-training — is out of scope. Chapter 10 explains where the pretraining/post-training boundary sits for reasoning models, because that boundary has moved.

## The through-line

Pretraining rewards discipline before scale. The decisions that determine your model's quality — data composition, scoping, tokenizer, architecture baseline, scaling-ladder validation — are cheap to make well and catastrophic to get wrong, and they nearly all happen *before* the expensive run begins. The large run itself is mostly operational excellence: keeping thousands of accelerators fed, healthy, and making forward progress, and catching a spike before it becomes a divergence. Start small, climb the ladder, validate every change at proxy scale, and treat the big run as confirmation of a plan you already trust — not the experiment itself.
