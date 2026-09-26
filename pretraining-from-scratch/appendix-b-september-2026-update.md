# Appendix B — September 2026 Update

*This series was compiled in early July 2026. This appendix covers two things: what changed between July and September 2026, and a few June 2026 results the main chapters missed. The chapters' guidance still holds. What moved is the frontier around it. Three reference designs shipped (Kimi K3, DeepSeek V4/V4.1, Qwen3.8), and each one challenges a "not yet" in Chapters 5, 7, and 8. Read the chapter first, then the matching section here.*

## B.1 The three reference designs of mid-2026

Chapter 5 described the 2026 architecture as "convergence plus efficiency engineering." The three largest open releases of the summer confirm the trend and push it further. All three are sparse MoE models with 1M-class context, and none of them uses full attention in every layer.

| | Kimi K3 (Moonshot, July 2026) | DeepSeek-V4 (June 2026) / V4.1-Flash (Sept 2026) | Qwen3.8 (Alibaba, Aug 2026) |
|---|---|---|---|
| Scale | 2.8T total / 104B active | V4: 1.6T / 49B; V4.1-Flash: 552B backbone | 2.4T-A95B MoE, plus a 27B dense model |
| Sequence mixer | 69 Kimi Delta Attention (linear) layers + 24 gated-MLA layers (≈3:1) | Compressed Sparse Attention + Heavily Compressed Attention (CSA/HCA); V4.1 adds CSA2 layer-sharing modes | Gated DeltaNet + attention hybrid (the Qwen3.5 lineage) |
| Residual path | **Attention Residuals** (learned attention over earlier layers' outputs) | **Manifold-Constrained Hyper-Connections (mHC)** | Standard |
| Experts | "Stable LatentMoE": 16 of 896 routed experts + 2 shared | DeepSeekMoE lineage | Sparse MoE |
| Optimizer / numerics | Released in MXFP4 weights (see the report for the training recipe) | **Muon on a trillion-plus MoE**; FP4 QAT on experts and the indexer path | — |
| Context | 1M, native vision | 1M; pretraining sequence length ramps from 4K to 1M | 262K in the deployment examples |
| License | Custom "Kimi K3 License" (revenue-gated; see cross-cutting Appendix C) | MIT | Apache 2.0 |

Read the table as a set of design options the frontier has validated, not a template to copy. The Chapter 5 guidance still stands: **your first run is dense GQA with AdamW.** Treat each row as a candidate for your ablation ladder once you have a stable baseline.

## B.2 Linear-attention hybrids reached the frontier

Chapter 5.5 called hybrids "the headline 2026 trend" and warned, citing MiniMax M2's reversion, that they are "not a free lunch." The evidence for them has now moved up a scale class:

- **Kimi K3** is the largest open model to date, and roughly three quarters of its layers are linear-attention (KDA). It interleaves one gated-MLA layer for every three KDA layers to keep a path for exact global retrieval. That is the 3:1 ratio Chapter 5 described, at 2.8T parameters. At matched scale, Moonshot reports up to 75% less KV cache and up to 6× decoding throughput at 1M context, while matching or beating full-attention baselines, including on RL-style post-training tasks. That last result is the one the MiniMax cautionary tale said to watch. Moonshot also open-sourced the kernel (FlashKDA, a CUTLASS implementation).
- **Qwen3.8** keeps Gated DeltaNet hybrids at the flagship tier (2.4T). Two of the three largest labs now ship linear-attention hybrids as their flagship.
- **DeepSeek took the other branch.** V4 keeps softmax attention but makes it sparse and compressed. CSA compresses every *m* tokens of KV into one entry, and a DSA-style indexer then selects the top-*k* compressed entries for each query. HCA compresses more heavily. The two layer types are interleaved.

**What this changes.** "Hybrids are proven enough to adopt from open reference implementations" (Chapter 5.5) is now true at every scale. A frontier open model and its kernels are available to copy from. The MiniMax lesson remains a lesson about *evaluation*, not a verdict on the architecture. Ablate hybrids on long-context retrieval and on multi-turn agentic tasks, not on perplexity.

## B.3 The residual stream is now a design axis

Chapter 5 treated the residual connection as fixed. Two independent lines of work now make it learnable, and both shipped in frontier models:

- **Attention Residuals** (Moonshot, arXiv:2603.15031, March 2026). Each layer normally adds its output to a running sum. Under AttnRes, each layer instead *attends over* the outputs of earlier layers, using learned, input-dependent softmax weights. The motivating analogy: fixed accumulation over depth has the same bottleneck that RNNs had over sequence length. **Block AttnRes** attends over block-level summaries to bound memory, and it reports about a 1.25× compute advantage (it matches a baseline trained with 25% more compute). The largest gains were on multi-step reasoning. Kimi K3 uses it in production.
- **mHC** (DeepSeek, arXiv:2512.24880). This widens the residual stream into several parallel streams, as hyper-connections do, and constrains the mixing matrix to a manifold so that signal propagation stays stable at depth. DeepSeek-V4 uses it.

**Practical call:** both are drop-in at the block level, but both add a novel component. Leave them out of a first run. Adopt one only when you pretrain in the ≥100B-token regime with a reference implementation, and only after it wins a ladder ablation.

## B.4 MoE: latent experts

**LatentMoE** (NVIDIA, arXiv:2601.18089; used in Nemotron 3 Super and Ultra) projects each token from the model width *d* down to a smaller latent width *ℓ* before routing and expert computation. That cuts expert parameter loads and all-to-all traffic by a factor of *d/ℓ*. The savings are then spent on *more* experts and a *higher* top-k at roughly constant inference cost. Kimi K3's "Stable LatentMoE" (16 of 896 experts) is the same idea at 2.8T scale. Chapter 5.4 noted the sparsity trend of K2's ~32× total/active ratio; K3 reaches ~27× at a much larger absolute size, and routes at a far finer granularity.

This matters for your design because all-to-all communication is the tax MoE imposes (Chapter 7). A design that shrinks the routed payload is a systems win before it is a quality win.

## B.5 FP4 pretraining: from research bet to shipped recipe

Chapter 7.4 said: "FP4 is the bleeding edge — not yet a safe default… FP4 only as a research bet." That was right for a first-timer, and it still is. But the chapter missed a June 2026 result that changes the frontier picture:

- **Nemotron 3 Ultra** (NVIDIA, June 2026; 550B total / 55B active, hybrid Mamba-Transformer with LatentMoE) was **pretrained in NVFP4 from the first gradient update, on 20T tokens.** The training ran in two phases: 15T tokens for breadth, then 5T for quality, under a WSD schedule. Most linear layers ran weights, activations, and gradients in NVFP4. Sensitive layers stayed in BF16 or MXFP8: latent projections, MTP layers, QKV/attention projections, and embeddings. The team branched from checkpoints at 5T, 10T, and 16T tokens and continued each branch in BF16 for comparison. The average train-loss gap was **under 0.4%**, smaller than the gap they measured on smaller variants.
- **DeepSeek-V4** used FP4 quantization-aware training for MoE expert weights and the sparse-attention indexer path. This is a narrower use: FP4 for the parts that dominate memory and bandwidth.

**Updated ladder:** BF16 first run → FP8/MXFP8 once you have a stable baseline → **FP4 on Blackwell-class hardware as a defensible option at scale.** Adopt it only by copying a published recipe exactly, including which layers stay in high precision, and run your own BF16 branch comparison as NVIDIA did. It is still not for your first run. The fact that the gap *shrank* with scale in NVIDIA's data suggests FP4 is more forgiving at frontier scale than small-scale ablations imply. Validate that claim on your own ladder before relying on it.

## B.6 Optimizer: Muon at trillion scale

Chapter 8.1 recorded MuonClip on Kimi K2 (1T parameters). **DeepSeek-V4 (1.6T MoE, 32T+ tokens) trained with Muon** alongside a custom hybrid ZeRO sharding strategy. It reported faster convergence and better stability than AdamW-class baselines at that size, plus two targeted stability fixes: "anticipatory routing" and SwiGLU clamping against loss spikes. Two of the largest open labs now pretrain with Muon, so it is the frontier default for new large runs. The chapter's practical call is unchanged: **AdamW for your first run; Muon is the first optimizer ablation to run after that.** The 2026 evidence strengthens the case for running that ablation early.

## B.7 Long context moved into pretraining

Chapter 5.6 and the midtraining series describe context extension as a *dedicated late phase*: train the bulk of tokens at 4–8K, then extend. DeepSeek-V4 instead ran a **sequence-length schedule from 4K to 1M across pretraining**. That is affordable only because CSA/HCA keep attention cost sub-quadratic. The general principle: when the architecture makes long sequences cheap, the extension phase can dissolve into the main schedule. With a standard-attention model, keep the phased approach.

## B.8 Hardware and benchmarks (Chapter 3)

- **NVIDIA Rubin** (Vera Rubin NVL144 platform) entered full production in mid-2026, with shipments scheduled from Q3. NVIDIA's per-package figures: 288GB HBM4 at ~22 TB/s, ~35 PFLOPs NVFP4 for training and ~50 PFLOPs NVFP4 for inference, and NVLink 6. vLLM had already added an `sm_107` Rubin build target by August. Rubin Ultra (HBM4e) follows in 2H 2027.
- **AMD Helios** (72× MI455X per rack, 31TB HBM4, ~2.9 EFLOPs FP4) is in production, with shipments from the end of Q3 2026. Announced commitments: Microsoft (Azure), OpenAI (a first 1GW of MI450 in 2H 2026), and Anthropic (up to 2GW, from 1H 2027). Chapter 3 called AMD "genuinely credible now"; frontier labs are now committing gigawatts to it.
- **MLPerf Training v6.0** (June 2026) added the first frontier-MoE benchmark, DeepSeek-V3 671B, plus a GPT-OSS-20B entry point: 95 systems from 24 organizations on 13 accelerator types. The fastest DeepSeek-V3 time-to-train was 2.02 minutes on 8,192 GB300 GPUs. Blackwell led every workload it entered, and AMD finished within a few percent. When comparing vendors for MoE pretraining, use the MoE benchmark rather than dense-model results.

**The first-timer recommendation is unchanged:** H100/H200 on the mainstream PyTorch stack. Rubin and Helios matter for procurement decisions made now for 2027 runs.

## B.9 Decisions

1. **Hybrid linear attention is frontier-validated** (Kimi K3, Qwen3.8). Adopt it from open reference code if long context is a product goal, and evaluate on agentic and retrieval tasks, not perplexity.
2. **Learned residual paths (AttnRes, mHC) are an ablation candidate**, not a first-run component.
3. **LatentMoE-style routing** is the current answer to MoE all-to-all cost. Consider it when you design a second-generation MoE.
4. **FP4 pretraining has a shipped 20T-token recipe** (Nemotron 3 Ultra). It is still not for your first run. Copy published recipes exactly, and verify against a BF16 branch.
5. **Muon is the frontier optimizer default.** Keep AdamW for run one, and schedule Muon as the first optimizer ablation.
6. **Long context can move into pretraining** when the architecture makes it cheap. Otherwise keep the dedicated extension phase.

## B.10 Sources

- Kimi Team, *Kimi K3: Open Frontier Intelligence*, 2026 — arXiv:2607.24653; Kimi K3 tech blog — https://www.kimi.ai/blog/kimi-k3
- Kimi Team, *Attention Residuals*, 2026 — arXiv:2603.15031; code — https://github.com/MoonshotAI/Attention-Residuals
- DeepSeek-AI, *DeepSeek-V4: Towards Highly Efficient Million-Token Context Intelligence*, 2026 — arXiv:2606.19348
- DeepSeek-AI, *DeepSeek-V4.1-Flash: Pushing the Limits of KV Cache Compression*, 2026 — arXiv:2609.19969
- DeepSeek-AI, *mHC: Manifold-Constrained Hyper-Connections*, 2025 — arXiv:2512.24880
- Qwen Team, Qwen3.8 repository — https://github.com/QwenLM/Qwen3.8
- NVIDIA, *LatentMoE*, 2026 — arXiv:2601.18089; *Nemotron 3 Ultra* technical report, 2026 — arXiv:2606.15007, https://research.nvidia.com/labs/nemotron/Nemotron-3-Ultra/
- MLCommons, *MLPerf Training v6.0 Results*, June 2026 — https://mlcommons.org/2026/06/mlperf-training-v6-0-results/
- AMD Helios / MI450 shipping and customer announcements, 2026 (Next Platform, Feb 2026; AMD Advancing AI 2026 coverage)
- NVIDIA Rubin specifications as announced at GTC 2026 (vendor figures; validate against independent benchmarks when available)

*Provenance note: several figures above come from vendor or lab announcements and secondary coverage compiled in September 2026, and some had no independent replication yet. Treat them the way Chapter 15 treats 2026-dated preprints: credible and directionally important, but numbers to verify.*
