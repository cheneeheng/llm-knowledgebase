# Chapter 9 — Frameworks and Infrastructure

## 9.1 Conceptual picture

Midtraining runs on the *pretraining* stack, not the post-training stack — it is self-supervised next-token prediction at billions-of-tokens scale, so it wants a pretraining trainer (Megatron/NeMo/torchtitan class) with the same parallelism, checkpointing, and data-pipeline machinery the pretraining series covered. What changes is the *shape* of the workload: runs are shorter (days, not months), more numerous (you'll do many CPT/annealing experiments), often smaller-scale (one node or a few, not the whole cluster), and they *start from an existing checkpoint* rather than random init. Those differences reshape which framework you pick and where the operational friction lands.

If you read the pretraining series' infrastructure and frameworks chapters, most of it carries over verbatim — memory budget, the five parallelism axes, precision, MFU, storage, fault tolerance. This chapter covers only the *deltas* for midtraining.

## 9.2 The framework map for midtraining

| Framework | Best for | Midtraining notes |
|---|---|---|
| **Megatron-Core / Megatron-LM** | Largest-scale CPT, custom kernels, frontier midtraining stages | Same as pretraining; use when your midtraining is a real multi-node capability run (100B+ tokens). Load the base checkpoint, adjust schedule to WSD decay or CPT re-warmup. |
| **NVIDIA NeMo** | Enterprise CPT at scale, managed pipelines | The heavyweight enterprise default; strong for large domain-adaptation runs with data curation (NeMo Curator) attached. |
| **torchtitan** | Clean PyTorch-native mid-scale runs | Good when you want a hackable, modern trainer for annealing/CPT experiments up to a few nodes without Megatron's weight. |
| **Axolotl** | Config-driven CPT/domain adaptation, 1–8 nodes | The ergonomic sweet spot for most teams doing domain CPT: YAML config, supports full CPT and LoRA, handles packing/replay mixing, chat templates. The default recommendation for a small lab's domain run. |
| **Unsloth** | Fastest single-node QLoRA CPT | When your whole midtraining is a LoRA/QLoRA domain adaptation on 1–2 GPUs; 2× faster, half the memory. Entry point for individuals. |
| **DeepSpeed** | Memory-constrained distributed CPT | ZeRO offload when you must fit a big model CPT on limited HBM; often used under Axolotl/NeMo. |

**The selection rule mirrors the pretraining series' but shifted down-scale:** most midtraining is *not* frontier-scale, so most teams land on **Axolotl (full CPT or LoRA, few nodes)** or **Unsloth (single-node LoRA)**, and only reach for Megatron/NeMo when the run is genuinely large (a code-scale 100B-token capability CPT, or a frontier lab's midtraining stage). Do not stand up a Megatron cluster to continue-pretrain a 7B on 10B tokens — Axolotl on one node does it in days.

## 9.3 What changes operationally

**Checkpoint loading and conversion.** The defining new task: you start from someone's (or your own) checkpoint, and checkpoint formats differ across trainers (HF Transformers, Megatron distributed, NeMo, DCP). Converting a HF base model into your trainer's sharded format — and back for release — is a real step that eats time and occasionally corrupts silently. Verify a converted checkpoint by running a few eval prompts *before* you start burning GPU-hours on it. Mismatched RoPE config, tied vs untied embeddings, or a dtype flip in conversion are classic quiet bugs.

**Data pipeline is smaller but mixture-heavier.** Midtraining datasets are 10–1000× smaller than pretraining, so you don't need the same streaming-from-object-store heroics — the data often fits on local NVMe. But the *mixture logic* is more intricate: you're blending domain data + replay + synthetic + long-context data at controlled ratios, sometimes with per-stage schedules. Get the mixer right (deterministic, resumable, ratio-accurate) and decode batches to verify (the recurring ritual). Tools like NeMo Curator, or Axolotl's dataset config, handle this; a hand-rolled concatenation usually gets the ratios subtly wrong.

**Schedule flexibility matters more.** You need WSD decay, CPT re-warmup, and multi-stage long-context schedules — schedules the pretraining defaults (one cosine) don't cover. Confirm your trainer supports a custom LR schedule with re-warmup from a loaded checkpoint; some assume warmup-from-zero and fight you.

**Shorter runs change the fault-tolerance calculus.** A 3-day CPT run doesn't need the elaborate auto-restart/hot-spare machinery a 3-month pretraining run does — but it still needs checkpointing every epoch or few thousand steps, because midtraining checkpoints are *the deliverables you're comparing* (best checkpoint is often not the last; Chapters 2, 8). Checkpoint frequently for *selection*, not just for crash recovery.

## 9.4 Cost and hardware

Midtraining is cheap relative to pretraining, which is much of its appeal. Anchored numbers (H100 at ~$2–3.50/hr as of early 2026):

| Run | Scale | Rough cost |
|---|---|---|
| LoRA domain CPT, 7B, 5B tokens | 4–8 H100, 1–3 days | Hundreds to low thousands of $ |
| Full domain CPT, 7B, 30B tokens | 1–4 nodes, ~week | Low tens of thousands of $ |
| Long-context extension, 8–70B | few nodes, days | Cheap in tokens; days of cluster |
| Frontier midtraining stage | full cluster, weeks | Millions (OLMo 3's full 32B pipeline incl. mid+long-context: ~$2.75M on 1024 H100 over ~47 days) |

The OLMo 3 number is a useful public anchor for the top end: a fully-open 32B including midtraining and long-context ran ~47 days on 1024 H100s for ~$2.75M — but that includes the base pretraining. A *midtraining-only* stage bolted onto an existing base is a small fraction of that. For most teams, midtraining is a per-experiment cost of thousands to tens of thousands of dollars, which is exactly why the iterate-cheaply discipline (microanneals, multiple decay forks) is affordable.

## 9.5 Decisions

1. **Use the pretraining stack, down-scaled**: Axolotl (few-node full CPT/LoRA) or Unsloth (single-node LoRA) for most teams; Megatron/NeMo only for genuinely large capability CPT or frontier stages.
2. **Treat checkpoint conversion as a first-class, verified step** — eval a few prompts on the converted base before spending GPU-hours; watch RoPE/embedding/dtype mismatches.
3. **Invest in the mixture pipeline** (ratio-accurate, resumable, decode-verified) more than in streaming throughput; midtraining data fits locally but the blend is intricate.
4. **Confirm your trainer supports WSD decay, re-warmup-from-checkpoint, and multi-stage schedules** before committing.
5. **Checkpoint frequently for selection**, not just crash recovery — the best checkpoint is often not the last.

Next: evaluation — the before/after harness that turns midtraining from guesswork into a measured trade.
