# Chapter 7 — Parallelism and Systems

## 7.1 Conceptual picture: memory is the forcing function

One GPU cannot hold or train your model. For N parameters trained in mixed precision with Adam, per-GPU memory before any parallelism is roughly:

- Weights (BF16): 2N bytes
- Gradients (BF16): 2N bytes
- Adam optimizer state: FP32 master weights (4N) + two FP32 moments (4N + 4N) = 12N bytes
- **≈ 16N bytes for model + optimizer state, before activations**

So an 8B model needs ~130GB for state alone — already more than an 80GB H100. A 70B model needs ~1.1TB. Activations add more, scaling with batch × sequence length × d_model × layers. Parallelism exists to spread this across devices.

Each axis of parallelism trades a different resource: **data parallelism** splits the *batch* (costs gradient-sync bandwidth); **tensor parallelism** splits individual *matrices* (costs activation communication every layer — needs NVLink-class bandwidth); **pipeline parallelism** splits the model by *layers* (costs "bubble" idle time); **context parallelism** splits the *sequence* (for long contexts); **expert parallelism** splits *MoE experts* (costs all-to-all routing traffic). Real runs compose several — "3D/4D/5D parallelism" — chosen to (1) fit memory, (2) keep the slow inter-node network off the critical path, and (3) keep every GPU busy.

## 7.2 The five axes in detail

**1. Data parallelism (DP) and its sharded forms.** Vanilla DDP replicates the whole model per GPU and all-reduces gradients — simple, but doesn't help memory. **ZeRO/FSDP** shard optimizer state, gradients, and (ZeRO-3 / FSDP full-shard) the parameters themselves across DP ranks, materializing weights layer-by-layer via all-gather just in time and reduce-scattering gradients. PyTorch **FSDP2** (per-parameter sharding) is the current native standard and what torchtitan composes with everything else. For models up to ~10–30B on a fast network, FSDP2 alone (plus activation checkpointing) is often all you need — that simplicity is worth real money.

**2. Tensor parallelism (TP).** Split attention heads and MLP matrices across (usually) 2–8 GPUs; each computes part of every matmul; activations are all-reduced (or all-gathered/reduce-scattered) **twice per transformer block**. Extremely communication-heavy → **TP lives inside the NVLink domain, never across node boundaries** (TP ≤ 8 on standard nodes; up to 72 inside a GB200 NVL72 rack — which is exactly why NVL72 changes strategy). **Sequence parallelism** (Megatron sense) extends TP by sharding the norm/dropout activations along the sequence, saving memory TP leaves replicated.

**3. Pipeline parallelism (PP).** Layers → stages → microbatches flow through. Communication is cheap (activations at stage boundaries, point-to-point), so **PP is the axis you place across nodes/racks** when the network is the constraint. The cost is the **pipeline bubble** — idle time filling/draining the pipe; you need global batch ≫ pipeline depth. Interleaved 1F1B ("virtual pipeline") and newer schedules (zero-bubble, DualPipe-style) shrink the bubble at the cost of scheduling complexity.

**4. Context/sequence parallelism (CP).** Shards the sequence dimension for the attention computation itself (ring/all-gather attention passing KV blocks between GPUs), so no single GPU holds activations for the whole sequence. Essential for the long-context phase (128K+), where even one sequence's activations exceed a GPU.

**5. Expert parallelism (EP).** MoE experts distributed across GPUs; tokens routed to their experts via **all-to-all** and back. Bandwidth-hungry and latency-sensitive — the hardest communication pattern you'll run on Hopper-class clusters, and why large NVLink domains (NVL72) are transformative for MoE. Modern MoE at scale (DeepSeek/Kimi-class) runs wide EP (64+ way) with carefully overlapped all-to-all.

**Activation checkpointing (recomputation).** Don't store most activations; recompute them during backward. Full recomputation costs ~30% extra FLOPs; **selective** recomputation (recompute cheap-but-large tensors like attention internals) gets most of the memory for much less overhead. On by default in most large-model configs; the activation-memory equation is also why bulk-phase context stays at 4–8K.

## 7.3 Composing a config — recipes by scale

Total GPUs = DP × TP × PP × CP × EP. The standard shapes (H100/H200-class, 8-GPU nodes):

| Scale | Typical layout |
|---|---|
| ≤1B dense | Plain DDP or FSDP2 on 8–64 GPUs. Nothing else |
| 7–8B dense | FSDP2 (full shard) across 64–1,024 GPUs; or Megatron TP=2–4 × DP. Selective activation checkpointing. Global batch ~4M tokens |
| 70B dense | TP=8 (intra-node) × PP=4–8 (inter-node) × sharded-DP for the rest; CP added for the long-context phase. Global batch ~8–16M tokens |
| Frontier MoE (~600B+ total / ~40B active) | Small TP (MLA-friendly) × EP=64+ × PP × DP, CP for long context; FP8; expert-comm/compute overlap. Megatron/DeepSeek-systems territory |

Rules that always hold: **TP inside NVLink; DP/FSDP and PP across the network; EP wherever all-to-all bandwidth is highest.** Keep the model-parallel product (TP×PP) as small as memory allows — DP scales more efficiently.

Practical tuning loop: pick the smallest TP that fits memory → add PP only when TP=8 + sharded-DP still doesn't fit (or inter-node DP traffic dominates) → set microbatch size to the largest that fits → profile (torch profiler / nsys) to verify communication overlaps computation → measure MFU and iterate.

**MFU targets:** ≥45% for dense ≤8B with FSDP2; ~40–47% for well-tuned 70B Megatron; MoE lower (25–40% is normal); long-context phases lower still. If you're at 20%, you have a communication, data-loading, or recomputation problem — profile before you scale.

## 7.4 Numerics: BF16, FP8, FP4

**BF16 mixed precision is the safe default:** BF16 compute/weights/grads, FP32 master weights and optimizer state, FP32 (or carefully scaled) gradient all-reduce accumulation. BF16's wide exponent range makes FP16-style loss scaling unnecessary — and **never use FP16 for LLM pretraining** (range issues).

**FP8 training is production-mature in 2026.** GEMMs in FP8 with per-tensor or (better) block/micro-scaled scaling, keeping sensitive ops — norms, softmax, router, embeddings, final logits, optimizer — in higher precision. Worth ~1.3–2× throughput on Hopper/Blackwell. Proof points: DeepSeek-V3 trained with fine-grained block-wise FP8 at 671B scale; NVIDIA's **MXFP8** recipe (hardware-scaled micro-block FP8 on Blackwell) matched BF16 loss on an 8B/15T run at ~2× throughput on GB200, and is simpler than earlier per-tensor recipes. Known failure modes appear *late* in training (SwiGLU weight correlations, logit growth after hundreds of billions of tokens) — so adopt FP8 only after you have a stable BF16 baseline and a loss-matching validation on your ladder (train the same small model both ways; curves should overlay).

**FP4 is the bleeding edge — not yet a safe default.** Blackwell exposes native FP4 (E2M1); MI355X supports MXFP4. Recipes (Quartet/MXFP4, UFP4) show near-lossless 4-bit pretraining becoming viable, and 2026 releases use FP4-QAT on MoE expert weights — but E2M1's non-uniform grid has a documented "shrinkage bias" instability and recipes are still being worked out. Watch; don't bet a first run on it.

**Recommendation ladder: BF16 first run → FP8/MXFP8 once you have a stable baseline (and ideally Blackwell) → FP4 only as a research bet.**

## 7.5 Kernels and throughput

- **FlashAttention (v2/v3)** — IO-aware fused attention, memory-linear in sequence length; v3 exploits Hopper features. Non-negotiable; also the varlen kernels that make intra-document masking free (Chapter 4.6).
- **Fused kernels** for RMSNorm, SwiGLU, RoPE, cross-entropy, and the optimizer step — cut memory traffic and launch overhead.
- **torch.compile** over the transformer block for fusion in PyTorch stacks.
- **Communication/compute overlap** — the biggest systems win after correct placement: FSDP all-gathers, gradient reduce-scatters, TP collectives, and EP all-to-alls should hide behind matmuls. Good frameworks do this; **verify it's actually happening in profiles** rather than silently falling back.
- **Determinism.** Fix seeds; prefer deterministic kernels where cheap; make restarts bitwise-reproducible if you can. At minimum, ensure the *data order* is exactly reproducible from a checkpoint — it turns mystery divergences into diffable bugs and is your main SDC defense at small scale.
