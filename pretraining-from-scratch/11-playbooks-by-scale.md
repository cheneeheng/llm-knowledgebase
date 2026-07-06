# Chapter 11 — Playbooks by Scale

Concrete end-to-end starting points. Treat these as informed defaults to validate on the ladder, not gospel — the right numbers come from your own proxy runs. Costs assume H100-class cloud pricing (~$2–2.5/hr reserved) at ~40% MFU; see Chapter 1.4 for the arithmetic.

## 11.1 The ~1B "first real run"

- **Purpose:** learn the whole pipeline end-to-end; doubles as your scaling-ladder proxy.
- **Architecture:** dense modern baseline — RMSNorm, SwiGLU, RoPE, GQA. ~d_model 2048, 16–24 layers, 16 heads / 4–8 KV heads. Tied embeddings, 4K context.
- **Tokenizer:** 32–64K, or reuse a proven open one (saves effort, makes numbers comparable).
- **Data:** 25B tokens is Chinchilla-optimal (~$300 of compute); train longer (100B–1T) to get a genuinely useful model and exercise the stack. A clean FineWeb-Edu/DCLM slice plus some code is plenty.
- **Hardware/parallelism:** a single 8-GPU node (up to a few nodes); pure DDP/FSDP2, no TP/PP. BF16.
- **Optimizer/schedule:** AdamW (β2 0.95, wd 0.1), WSD, grad-clip 1.0. Peak LR ~3e-4–1e-3 (validate; muP if you'll ladder from here).
- **Framework:** nanoGPT/litGPT/nanotron for learning; torchtitan if you want production patterns from the start.
- **Timeline/cost:** hours to a few days; hundreds to low-thousands of dollars. **This is where you make — and cheaply fix — your mistakes.**

## 11.2 The ~8B "serious small model"

- **Purpose:** a genuinely useful, deployable base model — the workhorse size.
- **Architecture:** dense baseline; d_model 4096 × 32 layers, 32 Q / 8 KV heads, QK-norm, 128K vocab, untied embeddings. Consider MTP; consider MLA if long-context memory is a priority. 8K bulk context, then extend to 128K.
- **Data:** 10–15T tokens (deliberately overtrained ~1,000–1,900 tok/param). Nemotron-CC-class base for the long horizon; 10–20% code, 5–10% math; strong annealing mixture in the last ~10–20%; long-context data for the extension phase. Full dedup + decontamination discipline.
- **Hardware/parallelism:** 256–1,024 H100/H200. FSDP2 across the cluster; TP=2–4 within nodes only if activations/context demand it; selective activation checkpointing; CP for the long-context phase. Global batch ~4M tokens. BF16 (FP8/MXFP8 once you have a stable baseline).
- **Optimizer/schedule:** AdamW + WSD (long constant phase, sharp final decay coinciding with the annealing mix); muP-transferred LR (~3e-4 class); batch ramp optional. Muon is a defensible ablation if your ladder confirms it.
- **Framework:** torchtitan or NeMo/Megatron-Core.
- **Timeline/cost:** ~3 weeks on 1,024 H100s for 15T tokens; order ~$1.3M of compute plus 20–30% contingency, data pipeline, and ablations.

## 11.3 The ~70B dense model

- **Architecture:** proven Llama-3-70B shape — d_model 8192 × 80 layers, 64 Q / 8 KV heads, 128K vocab, untied. Nothing exotic.
- **Data:** 15T+ tokens; serious multi-stage annealing; long-context extension.
- **Parallelism:** real 3D — TP=8 (intra-node) × PP=4–8 (inter-node, interleaved schedule) × sharded-DP for the rest; CP for long context; activation checkpointing throughout. Global batch 8–16M tokens. This is where systems complexity jumps.
- **Hardware:** 1,000–4,000 GPUs; fault tolerance and goodput now dominate operational effort (Chapter 3.7).
- **Framework:** Megatron-Core (or NeMo).
- **Reality check:** ~4.4M H100-hours ≈ **~$11M at reserved cloud rates** for a 15T-token run. Few teams train 70B *dense* from scratch anymore — at this capability level the economics increasingly favor MoE (11.4).

## 11.4 The frontier MoE

- **Architecture:** DeepSeek/Kimi-family fine-grained MoE — hundreds of small experts, shared expert(s), top-k routing with auxiliary-loss-free (bias-based) balancing, FP32 router; high sparsity (e.g., ~1T total / ~30B active), ratio chosen from MoE scaling ablations. MLA or a validated hybrid attention for long context. MTP head.
- **Data:** 15T+ tokens; heavy code/math; large synthetic component; strong multi-stage annealing; long-context extension to 128K–1M.
- **Parallelism:** the full stack — wide EP (64+ way, all-to-all overlapped; GB200 NVL72 domains shine here) × small TP (MLA-friendly) × PP (across nodes) × sharded-DP × CP. FP8/MXFP8 throughput. MFU expectations lower than dense (25–40%).
- **Optimizer/schedule:** AdamW *or* MuonClip (production-validated at exactly this scale — Kimi K2, 15.5T tokens, zero spikes); WSD; batch in the tens of millions of tokens; z-loss/QK-clip stability controls.
- **Hardware:** 10,000–100,000+ accelerators; goodput, fault tolerance, and communication engineering are the whole game.
- **Reality check:** a specialist, well-resourced-lab endeavor — everything in Chapters 3, 7, and 12 at maximum intensity. **Do not make this your first run.**

## 11.5 The graduation path

1B (learn everything, break everything) → ladder ablations (data mixtures, optimizer, any architecture deviation) → 8B with the validated recipe (first real product) → then *either* 70B-dense-class capability via MoE at similar active-parameter cost, *or* frontier MoE if you have the team and the cluster. Each step reuses the previous step's data pipeline, configs, and scaling-law fits — that accumulated validation is the real asset, more than any single checkpoint.
