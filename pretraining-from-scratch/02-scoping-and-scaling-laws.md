# Chapter 2 — Scoping and Scaling Laws

This is where you decide, on paper, what you are going to build — before spending a cent on the real run. Get this right and the rest is execution. Before touching a GPU you must answer three questions: how many parameters, how many tokens, and therefore how much compute (and money).

## 2.1 The fundamental compute equation

For a dense decoder-only transformer, training compute is well approximated by:

$$C \approx 6\,N\,D$$

where *N* is the number of parameters, *D* is the number of training tokens, and *C* is in FLOPs. The 6 comes from ~2 FLOPs per parameter per token for the forward pass and ~4 for backward. This one formula drives all budgeting (the cost table in Chapter 1.4 is just 6ND divided by cluster throughput).

The central scoping question follows immediately: you have a budget C (GPU-hours convert directly to FLOPs); how do you split it between **model size N** and **data D**? Too big for your data and you waste capacity (undertrained weights); too small and you waste data (the model saturates). There is an optimal ratio, and scaling laws find it.

For MoE models, use **active** parameters in 6ND for compute, but remember total parameters still set memory cost (see 2.4).

## 2.2 Chinchilla, stated plainly

The **Chinchilla scaling laws** (Hoffmann et al., 2022) answered: for a fixed compute budget, what is the optimal split between parameters and tokens? Answer: parameters and tokens should scale in roughly equal proportion, landing at roughly **20 tokens per parameter**. A compute-optimal 1B model wants ~20B tokens; an 8B wants ~160B; a 70B wants ~1.4T.

Chinchilla mattered because models before it (like the 175B GPT-3) were badly *undertrained* — a much smaller model on more data beat them, which reframed the field.

## 2.3 Why nobody actually trains Chinchilla-optimal anymore

The critical nuance: Chinchilla answers "what is the best model for a fixed *training* budget?" But that is rarely the real question. The real questions are:

- **"What is the best model I can serve cheaply?"** Inference cost scales with model size and runs for the model's entire deployed life. A slightly worse but much smaller model served billions of times is often far better business than a compute-optimal larger one. This pushes you to deliberately **overtrain** smaller models far past Chinchilla-optimal. Llama 3 8B was trained on 15T tokens — nearly **1,900 tokens per parameter**, ~90× the Chinchilla ratio. "Wasteful" of training compute, and completely correct.
- **"What is the best model that fits this memory/latency budget?"** Same logic.

By 2026 this is the norm: small and mid-size models are routinely trained on 10–20T+ tokens, and community consensus is that well-curated 15–30T-token datasets outperform poorly filtered 50T+ sets — competition has moved from raw quantity to curation, synthetic augmentation, and mixture design.

**Practical guidance:** treat 20 tokens/param as a *floor*, not a target. For a model you intend to actually use or ship, plan for hundreds to ~2,000 tokens per parameter, limited by budget and data availability. For a pure research/learning exercise where the artifact doesn't matter, Chinchilla-optimal gives the best loss per dollar.

Other limitations to keep in mind: Chinchilla was fit for dense models on one data distribution; optimal coefficients shift with data quality (better data changes the exponents); it says nothing about MoE; and it ignores repetition effects (2.5).

## 2.4 Scaling laws for MoE

For MoE, the single "N" splits in two:

- **Total parameters** — capacity; determines memory footprint and (partly) how much the model can know.
- **Active parameters** — what fires per token; determines training and inference FLOPs.

The training-compute law tracks *active* parameters; quality tracks something between active and total, improving with **sparsity** (total/active ratio) up to a point. The 2026 trend is toward higher sparsity — more total experts, fewer activated — because it buys capacity cheaply: Kimi K2 is ~1T total / 32B active (~32× sparsity); Qwen3.5-397B is 397B/17B active; MiniMax pushes to ~10B active. The tension: very high sparsity strains the router and the all-to-all communication (Chapter 7) and eventually hurts stability. If you go MoE, choose the total/active ratio from your own ablations, not fashion.

## 2.5 The data wall: repetition and synthetic data

A hard constraint the pure laws ignore: **high-quality text is finite.** High-quality, deduplicated English web text tops out at roughly 4–6T unique tokens from the best open pipelines (Chapter 4); adding code, math, multilingual, books, and papers gets an open effort to ~10–20T unique tokens. At 15T-token runs you are approaching the ceiling. Two consequences:

- **Repetition.** Up to ~4 epochs over the same data is close to "free" (behaves almost like fresh data, per the data-constrained scaling literature — Muennighoff et al. 2023); beyond that, value decays and eventually repetition hurts. Budget tokens against your *unique* supply accordingly. If your token budget exceeds your unique-token supply by more than ~4×, revisit the plan.
- **Synthetic data.** The biggest data-side shift of 2025–2026. Rather than only filtering the web down (FineWeb-Edu and DCLM discard ~90% of raw data), labs now **rephrase and regenerate** web content into higher-quality forms and synthesize reasoning traces. NVIDIA's Nemotron-CC rewrote roughly 2T tokens of web text this way — a recognized "fourth paradigm" of pretraining data after the Wikipedia, C4/Pile, and FineWeb/DCLM eras. Details in Chapter 4.

## 2.6 The scaling ladder — the methodology that protects you

The deeper use of scaling laws isn't budgeting — it's **de-risking decisions**. You do not pick hyperparameters for a 70B run by intuition. You climb a ladder:

1. **Fit your own scaling law.** Train a series of small models (e.g., 50M → 100M → 300M → 700M → 1.5B → 3B), each compute-optimally, on *your* data with *your* stack. Loss on a held-out set follows a remarkably clean power law in compute:

   $$L(C) = L_\infty + a\,C^{-b}$$

   Fit the curve. Published coefficients are a starting guess, not gospel — the exponents *for your setup* are what predict your big run.
2. **Establish hyperparameter transfer.** Use muP ("maximal update parametrization", Chapter 8) so learning rate and initialization transfer across width — tune LR once on a ~40M proxy and it stays near-optimal as you scale width. One of the highest-leverage tricks in the playbook.
3. **Predict and commit.** Extrapolate the fitted law to target compute. Predict final loss. If it isn't good enough, change the plan *now*, on paper.
4. **Run proxy ablations at ~1B.** Every data-mixture change, architecture tweak, and optimizer choice is validated at ~1B first, where a full run is hours-to-days and cheap. If option A beats option B consistently across the ladder, it will almost certainly win at scale. If two options *cross over* as scale increases, that's a red flag requiring a bigger pilot. Only changes that clearly help at proxy scale earn a place in the big run.

Rules of thumb: spend **1–10% of total compute** on ablations and ladder runs. Every hyperparameter you are unsure about at target scale should come from a scaling-law fit, from muP transfer, or from a published recipe at your exact scale. This discipline is the difference between a team that ships and one that burns its compute grant on a diverging run at token 3T. Treat the ladder as mandatory.

## 2.7 A worked scoping example

Budget ~$2M, H100s at an effective ~$2/GPU-hour reserved → ~1M GPU-hours. At 40% MFU that is ~1M × 3600 s × 4×10¹⁴ FLOP/s ≈ 1.4×10²⁴ FLOPs.

- **Compute-optimal:** C = 6ND with D ≈ 20N ⇒ N ≈ √(C/120) ≈ 1.1×10¹¹ → a ~100B model on ~2T tokens. Best loss for the money; expensive to serve.
- **Deployable ~8B:** fix N = 8×10⁹ ⇒ D = C/6N ≈ 2.9×10¹³ → ~29T tokens. That exceeds clean unique supply, so cap at ~15T (light repetition + synthetic augmentation) and bank the leftover compute — or train a second model.

Most teams choose the second plan, which is the whole point of 2.3: the compute-optimal answer and the *right* answer diverge, and inference economics breaks the tie.

## 2.8 Wall-clock and the decisions to lock before proceeding

Wall-clock estimate:

$$\text{days} = \frac{6ND}{\text{GPUs} \times \text{peak FLOPs} \times \text{MFU} \times 86400}$$

Example: 8B × 15T tokens on 1,024 H100s at 40% MFU → 7.2×10²³ / (1024 × 9.9×10¹⁴ × 0.4 × 86400) ≈ **21 days**. Aim for a run that finishes in 2–8 weeks; longer runs compound operational risk. If the number that comes out is six months, buy more GPUs or shrink the model.

By the end of scoping you should have written down:

- Target parameter count (and dense vs. MoE; if MoE, total and active)
- Target token count and unique-token supply check
- Context-length schedule (bulk length + extension targets)
- Total FLOPs, GPU type and count, estimated wall-clock
- Total budget including a **20–30% contingency** for failures, restarts, and ablations
