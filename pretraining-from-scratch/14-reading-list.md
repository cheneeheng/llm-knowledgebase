# Chapter 14 — Reading List

A curated path deeper, by topic. Dates reflect mid-2026; check for the latest revisions. If you want only the must-reads, see [Essential Reading](13-essential-reading.md); the full source list with URLs and provenance notes is in [References](15-references.md).

## Scaling laws and scoping

- Kaplan et al., *Scaling Laws for Neural Language Models* (2020) — the original power laws and the 6ND accounting.
- Hoffmann et al., *Training Compute-Optimal Large Language Models* (Chinchilla, 2022) — the ~20 tokens/param reference.
- Muennighoff et al., *Scaling Data-Constrained Language Models* (2023) — repetition value; the ~4-epoch finding.
- The overtraining / inference-aware scaling literature — the modern corrections to Chinchilla.

## Data

- Penedo et al., *The FineWeb Datasets* (2024) + the datatrove toolkit — the reference open CC pipeline.
- Li et al., *DataComp-LM (DCLM)* (2024) — model-based quality filtering.
- Su et al., *Nemotron-CC* (2024) and *Nemotron-CC-Math* (2025) + NeMo Curator — the long-horizon + synthetic recipe; layout-aware math extraction.
- Soldaini et al., *Dolma* (2024) — corpus, toolkit, and datasheet/governance practice.
- Lozhkov et al., *StarCoder 2 and The Stack v2* (2024) — the open code layer and license-filtering template.
- Synthetic-data scaling work: BeyondWeb (DatologyAI), Hugging Face's rephrasing experiments — the "fourth paradigm."

## Architecture

- Meta, *The Llama 3 Herd of Models* (2024) — the canonical modern dense recipe at scale.
- DeepSeek-AI, *DeepSeek-V3 Technical Report* (2024) — fine-grained MoE, MLA, aux-loss-free balancing, FP8, MTP. The single most influential open architecture report of the era.
- Kimi Team, *Kimi K2 Technical Report* (2025) — trillion-param MoE, MuonClip, WSD at scale.
- Component papers: RMSNorm (Zhang & Sennrich 2019), SwiGLU (Shazeer 2020), RoPE (Su et al. 2021), GQA (Ainslie et al. 2023), YaRN (Peng et al. 2023).
- Sebastian Raschka's architecture comparisons and attention-variant guides (2025–2026) — the best running map of the hybrid-attention landscape (Qwen3.5, Kimi Linear, Ling 2.5, MiniMax M2 revert).

## Frameworks and systems

- Shoeybi et al., *Megatron-LM* (2019); Narayanan et al. (2021) — tensor and pipeline parallelism foundations; Korthikanti et al. (2022) — sequence parallelism + selective recomputation.
- Rajbhandari et al., *ZeRO* (2020); Zhao et al., *PyTorch FSDP* (2023); Liang et al., *TorchTitan* (2024) — the sharded-DP lineage and 4D composition.
- Liu et al., *Ring Attention* (2023) — context parallelism.
- MaxText / Levanter repos (JAX/TPU).

## Precision and kernels

- Dao et al., *FlashAttention* v1–v3 (2022–2024).
- Micikevicius et al., *Mixed Precision Training* (2017), *FP8 Formats for Deep Learning* (2022); *FP8-LM* (2023); NVIDIA, *Recipes for Pre-training LLMs with MXFP8* (2025).
- FP4 frontier: Quartet (MXFP4); 2026 shrinkage-bias analyses and UFP4 — read before believing FP4 claims.

## Optimization and stability

- Loshchilov & Hutter, *Decoupled Weight Decay Regularization* (AdamW, 2017).
- Jordan et al., *Muon* (2024); Liu et al., *Muon is Scalable for LLM Training* / Moonlight (2025); MuonClip in the Kimi K2 report.
- Yang et al., *Tensor Programs V* (muP, 2022) — hyperparameter transfer.
- Hu et al., *MiniCPM* (2024) — the WSD/cooldown analysis.
- McCandlish et al., *An Empirical Model of Large-Batch Training* (2018) — critical batch size.
- Chowdhery et al., *PaLM* (2022) — z-loss, β2=0.95, rewind-and-skip; Wortsman et al., *Small-scale proxies for large-scale training instabilities* (2023).

## Evaluation

- EleutherAI **lm-evaluation-harness** — the standard.
- Benchmark papers: MMLU/MMLU-Pro, GSM8K, MATH, HumanEval/MBPP/LiveCodeBench, HellaSwag/ARC/WinoGrande, BBH, RULER.

## Operations — the honest picture

- The **OPT logbook** (Meta, 2022) and BLOOM/Megatron-DeepSpeed engineering notes — public chronicles of what actually goes wrong.
- *The Llama 3 Herd of Models* infrastructure sections — the failure-rate data behind Chapter 3.
- 2024–2026 lab infrastructure write-ups (goodput, resiliency, torchft).

## Codebases worth reading, in order

1. **nanoGPT / modded-nanoGPT** — the whole training loop in an afternoon; the speedrun lineage that produced Muon.
2. **torchtitan** — modern PyTorch distributed done right; read the parallelism composition.
3. **OLMo-core** (+ its data and reports) — the most complete open, reproducible pretraining release.
4. **Megatron-Core** — the industrial reference; read after torchtitan, with the papers alongside.
5. **datatrove / NeMo Curator** — what a real data pipeline looks like.
