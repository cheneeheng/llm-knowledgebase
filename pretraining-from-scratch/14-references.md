# Chapter 14 — References and Sources

Merged source list from the two underlying drafts (`../tmp/fable/`, `../tmp/opus/`), grouped by the chapter they inform. Where a claim in this report rests on a specific number (MMLU deltas, token counts, efficiency multipliers, failure rates, prices), the underlying source is listed here. Foundational pre-2024 knowledge (the transformer, Adam, standard mixed precision) is treated as background and not individually cited.

**Provenance note:** primary technical claims are anchored to arXiv papers, official model technical reports, or vendor engineering blogs (NVIDIA, AMD ROCm). Secondary syntheses (Raschka's galleries, framework round-ups, pricing aggregators) are used for landscape/trend framing and cross-checked against the primary reports they cite. Prices and availability are mid-2026 snapshots and will drift — re-verify anything you act on financially against the primary source at time of use; the field moves monthly.

## Ch. 1–2 — Big picture, scaling laws, budgets

- Kaplan et al., *Scaling Laws for Neural Language Models*, 2020 — arXiv:2001.08361. Power-law scaling; C ≈ 6ND accounting.
- Hoffmann et al., *Training Compute-Optimal Large Language Models* (Chinchilla), 2022 — arXiv:2203.15556. ~20 tokens/param compute-optimal result.
- Muennighoff et al., *Scaling Data-Constrained Language Models*, 2023 — arXiv:2305.16264. "Repetition fine up to ~4 epochs."
- Meta AI, *The Llama 3 Herd of Models*, 2024 — arXiv:2407.21783. Llama-3 8B at ~15T tokens; data-mix disclosure; 405B failure-rate figures (Ch. 3).
- *The Hitchhiker's Guide to Agentic AI: From Foundations to Systems*, 2026 — arXiv:2606.24937. Pretraining best-practices section; CLM loss form; dedup methodology.
- Nursena Kok, *Pre-training Phase of Large Language Models*, Medium, Feb 2026 — https://medium.com/@nursena_kok/pre-training-phase-of-large-language-models-the-foundation-of-modern-ai-111b377f0a33. 2026 decoder-only CLM convergence; "well-curated 15–30T beats poorly filtered 50T+."
- *Pretraining: Breaking Down the Modern LLM Training Pipeline*, MLOps Community / Comet, June 2026 — https://mlops.community/blog/pretraining-breaking-down-the-modern-llm-training-pipeline.

## Ch. 3 — Infrastructure

- Meta AI, Llama 3 report (above) — ~1 interruption/3 hours; 400+ GPU failures over 54 days on 16K H100s.
- NVIDIA — Blackwell platform press release: https://nvidianews.nvidia.com/news/nvidia-blackwell-platform-arrives-to-power-a-new-era-of-computing · DGX B200: https://www.nvidia.com/en-us/data-center/dgx-b200/ · GB200 NVL72: https://www.nvidia.com/en-us/data-center/gb200-nvl72/
- B200/GB200 specs and mid-2026 pricing (secondary aggregators): Spheron https://www.spheron.network/blog/nvidia-b200-complete-guide/ · Inworld https://inworld.ai/resources/nvidia-b200-gpu-cloud · Runpod https://www.runpod.io/articles/guides/nvidia-b200 · JarvisLabs https://jarvislabs.ai/ai-faqs/nvidia-b200-specs · Thunder Compute https://www.thundercompute.com/blog/nvidia-b200-pricing · GetDeploying https://getdeploying.com/gpus/nvidia-gb200
- AMD ROCm blogs, June 2026 — MLPerf Training v6.0 reproduction: https://rocm.blogs.amd.com/artificial-intelligence/mlperf-training6.0-repro/README.html · technical dive: https://rocm.blogs.amd.com/artificial-intelligence/mlperf-training-v6.0/README.html. Primus (Megatron-LM + torchtitan backends); MI325X/MI350/MI355X; MXFP4 on MI355X.
- *Sarus Suite: Cloud-native Containers for HPC*, 2026 — arXiv:2604.17064. Megatron Llama-3 8B Slurm scaling reference (8–1024 GPUs).
- NVIDIA/nccl-tests (GitHub) — interconnect validation/burn-in methodology.

## Ch. 4 — Data

- Penedo et al., *The FineWeb Datasets*, NeurIPS 2024 — arXiv:2406.17557. FineWeb/FineWeb-Edu; datatrove.
- Li et al., *DataComp-LM (DCLM)*, 2024 — arXiv:2406.11794. Classifier-based quality filtering.
- Su et al., *Nemotron-CC*, 2024 — arXiv:2412.02595. 6.3T tokens (4.4T unique + 1.9T synthetic); +5.6 MMLU (HQ) over DCLM; ~4× more unique tokens; ~80% fuzzy-duplicate note for FineWeb-Edu/DCLM; classifier ensembling.
- NVIDIA blogs — *Announcing Nemotron-CC*: https://developer.nvidia.com/blog/announcing-nemotron-cc-a-trillion-token-english-language-dataset-for-llm-pretraining/ · *Building Nemotron-CC using NeMo Curator*: https://developer.nvidia.com/blog/building-nemotron-cc-a-high-quality-trillion-token-dataset-for-llm-pretraining-from-common-crawl-using-nvidia-nemo-curator/
- *Nemotron-CC-Math*, 2025 — arXiv:2508.15096. Lynx-based layout-aware extraction; 133B/52B tokens; +4.8–12.6 MATH.
- Open-sci / OpenEuroLLM reference models, Aug 2025 — https://openeurollm.eu/blog/open-sci-oellm-reference-models-release. Independent Nemotron-CC-HQ > DCLM > FineWeb-Edu confirmation.
- Lozhkov et al., *StarCoder 2 and The Stack v2*, 2024 — arXiv:2402.19173.
- Math corpora: OpenWebMath (arXiv:2310.06786), FineMath (HF), MegaMath, proof-pile.
- Soldaini et al., *Dolma*, 2024 — arXiv:2402.00159. Corpus + toolkit + datasheet practice.
- Component corpora: RedPajama-v2 (Together), TxT360, C4 (arXiv:1910.10683), The Pile (arXiv:2101.00027).
- DatologyAI, *BeyondWeb* — https://www.datologyai.com/blog/beyondweb. Synthetic-data scaling.
- Gujiakai, *The Industrial Recipe for Synthetic Data: HuggingFace's 90 Experiments*, Mar 2026 — https://blog.gujiakai.me/en/2026/03/huggingface-finephrase-synthetic-data/. Classifier mis-scoring of synthetic data.
- Visalytica, *LLM Training Data in 2026*, Dec 2025 — https://www.visalytica.com/blog/llm-training-data.
- FineWeb-Edu overview — https://www.emergentmind.com/topics/fineweb-edu-dataset.
- Tooling: datatrove (HF), NeMo Curator (NVIDIA), Dolma toolkit (AI2); trafilatura/resiliparse; MinHash-LSH.

## Ch. 5 — Architecture

- DeepSeek-AI, *DeepSeek-V3 Technical Report*, 2024 — arXiv:2412.19437. MLA; fine-grained MoE; aux-loss-free balancing; MTP; block-wise FP8 at 671B.
- Kimi Team, *Kimi K2: Open Agentic Intelligence*, 2025 — arXiv:2507.20534. 1T/32B MoE; MuonClip; WSD; 4K-window pretraining + long-context activation.
- Component papers: RMSNorm arXiv:1910.07467 · SwiGLU arXiv:2002.05202 · RoPE arXiv:2104.09864 · GQA arXiv:2305.13245 · YaRN arXiv:2309.00071 · muP arXiv:2203.03466 · Mistral 7B / Mixtral (sliding window; MoE).
- Sebastian Raschka — *The Big LLM Architecture Comparison*: https://magazine.sebastianraschka.com/p/the-big-llm-architecture-comparison · *A Dream of Spring for Open-Weight LLMs* (Apr 2026): https://magazine.sebastianraschka.com/p/a-dream-of-spring-for-open-weight · *A Visual Guide to Attention Variants* (Mar 2026): https://magazine.sebastianraschka.com/p/visual-attention-variants · *Hybrid Attention* gallery: https://sebastianraschka.com/llm-architecture-gallery/hybrid-attention/ · *Beyond Standard LLMs* (MiniMax M2 revert): https://magazine.sebastianraschka.com/p/beyond-standard-llms · *LLM Research Papers: The 2026 List*: https://magazine.sebastianraschka.com/p/llm-research-papers-2026-part1
- Maxime Labonne, *Qwen3.5: Nobody Agrees on Attention Anymore* — https://medium.com/@mlabonne/qwen3-5-nobody-agrees-on-attention-anymore-4709e1bd014b. Qwen3.5 397B-A17B hybrid.
- Yugank Aman, *Inside the Architecture of Every Frontier Model*, Apr 2026 — https://medium.com/@yugank.aman/inside-the-architecture-of-every-frontier-model-what-22-open-weight-llms-reveal-b054ae601980. Ling 2.5 7:1 hybrid; StepFun MTP-3.
- *HySparse*, 2026 — arXiv:2602.03560. Hybrid/sliding-window/sparse survey.
- *Hybrid Linear Attention Done Right*, 2026 — arXiv:2601.22156 · *Liger*, 2025 — arXiv:2503.01496.
- AI21, *Attention was never enough*, Mar 2026 — https://www.ai21.com/blog/rise-of-hybrid-llms/. SSM/hybrid lineage.
- MoE census (secondary): Labellerr — https://www.labellerr.com/blog/top-open-source-moe-llms/ · TechTwitter Jan–Feb 2026 releases — https://www.techtwitter.com/articles/10-open-weight-llm-releases-in-january-and-february-2026 · Atlas Cloud model comparison — https://www.atlascloud.ai/blog/guides/kimi-k2-6-vs-glm-5-1-vs-qwen-3-6-plus-vs-minimax-m2-7-coding-2026

## Ch. 6–7 — Frameworks, parallelism, precision

- Shoeybi et al., *Megatron-LM*, 2019 — arXiv:1909.08053 · Narayanan et al., pipeline parallelism at scale, 2021 · Korthikanti et al., *Reducing Activation Recomputation*, 2022 — arXiv:2205.05198.
- Rajbhandari et al., *ZeRO*, 2019/2020 — arXiv:1910.02054.
- Zhao et al., *PyTorch FSDP*, VLDB 2023 · Liang et al., *TorchTitan*, 2024 — arXiv:2410.06511 (FSDP2+TP+PP+CP composition; float8; distributed checkpointing).
- Liu et al., *Ring Attention with Blockwise Transformers*, 2023 — context parallelism.
- Dao et al., *FlashAttention-2* — arXiv:2307.08691 · Shah et al., *FlashAttention-3* — arXiv:2407.08608.
- Micikevicius et al., *Mixed Precision Training*, 2017; *FP8 Formats for Deep Learning*, 2022 · Peng et al., *FP8-LM*, 2023.
- NVIDIA, *Recipes for Pre-training LLMs with MXFP8*, 2025 — arXiv:2506.08027. BF16-parity at 8B/15T; ~2× on GB200.
- *InfiR2: A Comprehensive FP8 Training Recipe*, 2025 — arXiv:2509.22536.
- *To FP8 and Back Again*, 2024 — arXiv:2405.18710. Late-training FP8 instabilities.
- FP4: Ling Team, *Rethinking Shrinkage Bias in FP4 Pretraining / UFP4*, 2026 — arXiv:2606.20381 · *Elucidating the Design Space of FP4 Training* — arXiv:2509.17791 · The Salt, *End-to-End FP4 Training with Blackwell* — https://thesalt.substack.com/p/end-to-end-fp4-training-for-llms
- NVIDIA Transformer Engine docs — FP8 recipes on Hopper/Blackwell.
- Framework landscape (secondary): Megatron-LM G2 page (2B–462B range; ~47% MFU) — https://www.g2.com/products/megatron-lm/reviews · FutureAGI, *Continued LLM Pretraining in 2026* — https://futureagi.com/blog/continued-llm-pretraining/ · TipTinker, *The 2026 AI Engineering Stack* — https://www.tiptinker.com/llm-frameworks/ · *Modalities*, 2026 — arXiv:2602.08387.
- Open trainers: OLMo/OLMo-2 (AI2, arXiv:2501.00656), GPT-NeoX (EleutherAI), nanotron (HF), MaxText & Levanter (JAX), litGPT, nanoGPT/modded-nanoGPT.

## Ch. 8 — Optimization

- Loshchilov & Hutter, *Decoupled Weight Decay Regularization* (AdamW), 2017 — arXiv:1711.05101.
- Jordan et al., *Muon*, 2024 — modded-nanoGPT speedrun lineage.
- Liu et al. (Moonshot), *Muon is Scalable for LLM Training*, 2025 — arXiv:2502.16982. ~2× compute efficiency vs. AdamW; Moonlight; open distributed Muon.
- Kimi K2 report (above) — MuonClip; 15.5T tokens spike-free at 1T params.
- Muon analysis (2026): *Spectral Scaling Laws of Muon* — arXiv:2606.04058 · *Can Muon Fine-tune Adam-Pretrained Models?* — arXiv:2605.10468 · *Optimizer-Model Consistency* — arXiv:2605.06654 · *River-Valley Perspective* — arXiv:2606.21514 · variants: MuonEq arXiv:2603.28254, HTMuon arXiv:2603.10067, Pion arXiv:2605.19282.
- KingNish, *Muon vs MuonClip vs Muon+AdamW*, HF blog, Dec 2025 — https://huggingface.co/blog/KingNish/optimizer-part1.
- Chowdhery et al., *PaLM*, 2022 — arXiv:2204.02311. Rewind-and-skip; β2=0.95; z-loss.
- Hu et al., *MiniCPM*, 2024 — arXiv:2404.06395. WSD/trapezoid schedule and extendability.
- Yang et al., *Tensor Programs V* (muP), 2022 — arXiv:2203.03466.
- McCandlish et al., *An Empirical Model of Large-Batch Training*, 2018 — critical batch size.
- Wortsman et al., *Small-scale proxies for large-scale Transformer training instabilities*, 2023 — QK-norm/z-loss/spike behavior.

## Ch. 9, 12 — Evaluation and operations

- EleutherAI lm-evaluation-harness (GitHub).
- Benchmarks: MMLU/MMLU-Pro, GSM8K, MATH, HumanEval, MBPP, LiveCodeBench, HellaSwag, ARC, WinoGrande, BBH, RULER.
- Zhang et al., *OPT*, 2022 — and its public training logbook.
- BigScience *BLOOM* + Megatron-DeepSpeed engineering notes.
