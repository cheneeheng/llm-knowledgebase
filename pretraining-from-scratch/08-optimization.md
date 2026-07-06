# Chapter 8 — Optimization: Optimizers, Schedules, Batch Size, Stability

The optimizer, learning-rate schedule, batch size, and stability controls determine whether your run converges efficiently or wastes tokens and diverges. They interact — tune them together, on the ladder.

## 8.1 The optimizer

**AdamW is the reference and the safe choice.** Adam with decoupled weight decay; nearly every published model used it or a close variant. The LLM-specific settings matter:

- β1 = 0.9, **β2 = 0.95** (not the 0.999 default — LLMs want faster second-moment adaptation; it is also a stability measure)
- ε = 1e-8; weight decay ~0.1, applied **only to weight matrices** (not norms or biases; practices vary on embeddings)
- Gradient clipping at global-norm 1.0 (universal)
- Cost: two FP32 moments per parameter dominate optimizer memory (Chapter 7.1)

**Muon / MuonClip — the optimizer story of 2025–2026, and it crossed into production.** Muon orthogonalizes the momentum-smoothed gradient of each 2D weight *matrix* via Newton–Schulz iterations (spectral-norm steepest descent), producing balanced updates that use all directions of the weight space; AdamW handles the non-matrix parameters (embeddings, norms, output head). The evidence:

- Moonshot's scaled recipe (adding weight decay + per-matrix update-scale matching) reported **~2× computational efficiency vs. AdamW** in scaling-law experiments, validated by pretraining Moonlight (3B/16B MoE, 5.7T tokens).
- **Kimi K2 (~1T params, 15.5T tokens) was pretrained with MuonClip — Muon plus QK-clip — with zero loss spikes.** QK-clip rescales the query/key projections whenever attention logits exceed a threshold, eliminating the attention-logit blowups that were Muon's instability mode at scale.
- Distributed implementations are open (Moonshot's memory-optimal version; torchtitan integrations), and a large share of notable 2026 open releases used Muon-family optimizers.

Caveats: extra per-step compute (Newton–Schulz), one more nonstandard component, and benefits are best-documented for *pretraining* — evidence suggests keeping the optimizer family consistent across later training stages, which is a consideration if someone else will post-train your base model.

**Practical call:** AdamW for your first run — the known-good baseline removes a variable. Muon/MuonClip is a legitimate, production-validated upgrade to ablate on your ladder; the ~2× claims justify the experiment. (Also in the ecosystem: AdamW-8bit for memory, Shampoo/SOAP lineage, Lion, Adafactor — none displaces the AdamW-default/Muon-challenger picture for pretraining in 2026.)

## 8.2 Learning rate — the hyperparameter

Peak LR is the single most important scalar. Being 2× too high risks divergence; 2× too low quietly wastes compute. It shrinks with model scale — reference points (AdamW, standard parametrization): ~1B: 3e-4–1e-3 · 8B: ~3e-4 · 70B: ~1.5e-4 — with batch-size and width dependencies. When in doubt, take the published value from the recipe you are copying at your exact scale.

**muP (Maximal Update Parametrization)** formalizes the dependencies: parametrize init scales and per-layer LRs so the optimal LR *transfers across width* — tune once on a ~40M proxy, use at 8B. One of the highest-leverage tricks in the playbook; strongly recommended if you'll do any hyperparameter search. Fallback: published standard-parametrization recipes.

### Schedule

- **Warmup — always.** Linear ramp from ~0 over ~0.1–1% of training (500–2,000 steps), letting Adam's second moments stabilize before large updates. Skipping warmup is a classic early-divergence cause.
- **Cosine decay (the classic):** decay to ~10% of peak over the run. Works well; weakness is that the shape bakes in the total token count in advance — extending the run later is awkward.
- **WSD — Warmup-Stable-Decay, a.k.a. trapezoid (the modern default):** hold peak LR flat for most of training, then decay sharply (linear or 1-sqrt) over the final ~10–40%. Advantages: (1) the stable phase is *extendable* — decide the endpoint late, continue training later; (2) it pairs perfectly with the **annealing data mixture** (Chapter 4.5) — LR decay and the quality-data shift happen together and compound; (3) you can branch *multiple* decay runs off one stable trunk (ideal for mixture-annealing experiments). Kimi K2 used WSD: ~10T tokens at constant 2e-4, then decay to 2e-5 over the final ~5.5T.

Either is defensible; **WSD is the recommended default for a new run.**

## 8.3 Batch size

- Measured in **tokens** (sequences × seq_len × DP degree): ~0.5–4M for ≤8B, 4–16M+ at 70B, tens of millions at frontier scale (Kimi K2 held ~67M).
- **Critical batch size:** beyond it, bigger batches stop improving loss-per-token — you spend more compute per step for diminishing returns. It *grows* over training, which is why **batch-size ramps** (small early, when the model learns fastest per update; larger later) are common at scale. Staying near but below critical is the efficient regime — another thing the ladder can measure.
- **LR–batch coupling.** Peak LR and batch size are linked (LR scales sub-linearly with batch). Changing the batch via more GPUs without re-examining LR is a frequent source of "why did my scaled-up run diverge."
- **Gradient accumulation** decouples global batch from per-GPU memory.

## 8.4 Initialization and small-but-load-bearing settings

- Init ~N(0, 0.02) (or ~1/√d) with depth-scaled residual-output projections — or muP rules if using muP.
- **z-loss** ~1e-4 on the final softmax log-normalizer if you see output-logit drift; cheap insurance (PaLM lineage). Logit soft-capping is the alternative.
- **QK-norm** (architectural, Chapter 5.2) — counters attention-logit explosion at the source.
- MoE-specific: router in FP32; bias-based (aux-loss-free) load balancing; watch per-expert load continuously.
- No dropout. Weight decay off for norms/biases.

## 8.5 Stability: loss spikes and what to do at 3 a.m.

Loss spikes are a fact of large-run life. Causes range from pathological data batches and numeric edge cases (attention-logit or output-logit blowups, late-training FP8 issues) to genuine hardware corruption.

**Defense in depth (preventive):** conservative LR with full warmup; β2 = 0.95; QK-norm and/or QK-clip; z-loss; grad clipping at 1.0; higher-precision sensitive ops (especially under FP8); careful init/muP; per-domain loss tracking (localizes data-caused spikes); frequent checkpoints. Track **gradient norm** continuously — it usually foreshadows the loss spike.

**Reactive playbook:**

1. **Classify.** Small spikes that self-heal within a few hundred steps are common and fine. A spike that keeps climbing is a divergence — act.
2. **Rewind** to the last good checkpoint before the spike.
3. **Skip the offending data window** (hundreds of steps) — a single pathological batch is a common trigger; the rewind-and-skip practice has been normalized since PaLM.
4. **If spikes recur across data windows**, suspect in order: LR too high → numerics (drop to a more conservative FP8 recipe, or BF16) → a sick node emitting bad values (SDC).
5. **Strengthen controls** (z-loss, QK-norm/QK-clip, tighter clipping) if the problem is systematic, and investigate root cause before resuming at full speed.
