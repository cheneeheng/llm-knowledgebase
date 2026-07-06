# Chapter 13 — Essential Reading: The Twelve Sources That Matter Most

The full, topic-organized path is in the [Reading List](14-reading-list.md) and the complete source list in [References](15-references.md). This chapter is the ruthless cut: if you read only these twelve, you will have seen every load-bearing idea in this report from its primary source. They are ordered as a reading sequence, not by importance — early entries build the frame the later ones assume.

Selection criteria: primary sources only (technical reports and papers, plus one codebase and one logbook); each must anchor a *decision* you will actually make; overlapping sources lose to the one with the best signal-to-page ratio.

| # | Source | One-line reason | Chapters |
|---|---|---|---|
| 1 | nanoGPT / modded-nanoGPT (codebase) | See the entire training loop before reading about scale | 1, 6 |
| 2 | Hoffmann et al., *Chinchilla* (2022) | The N-vs-D framework every scoping decision starts from | 2 |
| 3 | Muennighoff et al., *Data-Constrained LMs* (2023) | The correction to Chinchilla you'll actually operate under | 2, 4 |
| 4 | Meta, *The Llama 3 Herd of Models* (2024) | The single best end-to-end account of a real large run | all |
| 5 | Penedo et al., *FineWeb* (2024) | Data-pipeline craft, stage by stage, fully reproducible | 4 |
| 6 | Su et al., *Nemotron-CC* (2024) | The long-horizon + synthetic data recipe; why filtering harder isn't the answer | 4 |
| 7 | DeepSeek-AI, *DeepSeek-V3 Technical Report* (2024) | The most influential open architecture+systems report of the era: MoE, MLA, FP8, MTP | 5, 7 |
| 8 | Kimi Team, *Kimi K2 Technical Report* (2025) | MuonClip and WSD validated at 1T params / 15.5T tokens, spike-free | 5, 8 |
| 9 | Yang et al., *Tensor Programs V / muP* (2022) | Hyperparameter transfer — what makes the scaling ladder work | 2, 8 |
| 10 | Shoeybi/Narayanan et al., *Megatron-LM* (2019/2021) | Where tensor and pipeline parallelism come from and why placement rules exist | 7 |
| 11 | Liang et al., *TorchTitan* (2024) | How the modern PyTorch stack composes all the parallelism axes in practice | 6, 7 |
| 12 | Zhang et al., *OPT* + its public logbook (2022) | The unfiltered operational reality of a large run — what dashboards don't show | 3, 9, 12 |

## Why each, and what to extract

**1. nanoGPT / modded-nanoGPT — github.com/karpathy/nanoGPT, github.com/KellerJordan/modded-nanogpt.**
Not a paper, and deliberately first. Every abstraction in the other eleven sources (a step, a schedule, a shard, a checkpoint) is ~600 lines of readable code here. Run it on one node before reading anything about 10,000. modded-nanoGPT adds the speedrun lineage where Muon was born — reading its commit history is a compressed course in what actually moves training efficiency.

**2. Chinchilla — arXiv:2203.15556.**
Read for: the compute-optimal framework (C ≈ 6ND, ~20 tokens/param), and the *method* — how to fit a scaling law from a sweep of small runs, which is exactly what your ladder does. Skim the isoFLOP analysis; that's the part you'll reuse.

**3. Scaling Data-Constrained Language Models — arXiv:2305.16264.**
The bridge from Chinchilla's world (infinite data) to yours (finite data). Read for: the value of repeated epochs (~4 near-free, then decay), which sets your unique-token budget math and justifies the overtraining regime everyone actually trains in.

**4. The Llama 3 Herd of Models — arXiv:2407.21783.**
If you read only one document on this list, it's this one. A frontier lab describing, in one place: the dense recipe at 8B/70B/405B, the data mix, scaling-law usage, the parallelism layout, and — uniquely — honest failure statistics (an interruption every ~3 hours at 16K GPUs). It grounds every chapter of this report in one worked example.

**5. FineWeb — arXiv:2406.17557.**
The best-documented web-data pipeline: extraction, filtering, dedup, and — critically — *ablations for every stage*, so you see which filtering decisions move benchmarks and by how much. Pairs with the datatrove code, so nothing is left as hand-waving.

**6. Nemotron-CC — arXiv:2412.02595.**
The counterpoint to FineWeb-Edu-style aggressive filtering: classifier ensembling, global dedup, and synthetic rephrasing to keep ~4× more unique tokens at equal-or-better quality — the recipe for 10T+ horizons. Read for: why unique-token supply is the binding constraint, and how synthetic data entered the mainstream.

**7. DeepSeek-V3 Technical Report — arXiv:2412.19437.**
Four chapters of this report trace to this one document: fine-grained MoE with aux-loss-free balancing, MLA, multi-token prediction, and block-wise FP8 training at 671B — plus the systems co-design (DualPipe, expert-parallel overlap) that makes it run. The most-copied open recipe in the field.

**8. Kimi K2 Technical Report — arXiv:2507.20534.**
The production proof for the two biggest optimization shifts of the era: MuonClip (Muon + QK-clip) and WSD, at 1T parameters over 15.5T tokens with zero loss spikes. Read for: the QK-clip mechanism, the exact WSD shape (~10T stable at 2e-4, decay to 2e-5), and the 4K-window-then-extend context strategy.

**9. Tensor Programs V (muP) — arXiv:2203.03466.**
The heaviest read on the list; skim the theory, internalize the result: with the right parametrization, optimal LR and init *transfer across width*, so you tune on a 40M proxy and trust it at 8B. This is what turns the scaling ladder from a hope into a method. (The MiniCPM paper is the gentler practical companion.)

**10. Megatron-LM — arXiv:1909.08053 (+ Narayanan et al. 2021 for pipeline scheduling).**
Where tensor parallelism and the modern pipeline schedules come from. Read for: *why* TP costs two collectives per layer and must live inside NVLink, and why PP's bubble shrinks with microbatching — the physics behind every placement rule in Chapter 7.

**11. TorchTitan — arXiv:2410.06511.**
The modern synthesis: how FSDP2, TP, PP, and CP compose in idiomatic PyTorch, with distributed checkpointing and float8. Read alongside its codebase — it is the reference implementation for everything Chapters 6–7 recommend for sub-frontier scale.

**12. OPT-175B and its public logbook — arXiv:2205.01068 + the logbook in facebookresearch/metaseq.**
Dated as a recipe, timeless as testimony: hardware failures, loss spikes, mid-run LR surgery, restarts — recorded in real time by the people on call. The only document on this list that conveys what Chapters 9 and 12 feel like in practice. Read it last, right before you launch.

## Near misses

Would-be #13–16, cut only for overlap: **DCLM** (arXiv:2406.11794 — quality-classifier filtering; FineWeb covers the craft), **MiniCPM** (arXiv:2404.06395 — the WSD analysis; K2 shows it at scale), **FlashAttention-2** (arXiv:2307.08691 — read if you touch kernels), and **Wortsman et al.** (small-scale proxies for instabilities — read before your first spike, arXiv:2309.14322).
