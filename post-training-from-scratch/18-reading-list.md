# Chapter 18 — Reading List

A curated path deeper, by topic. Dates reflect mid-2026; check for the latest revisions. If you want only the must-reads, see [Essential Reading](17-essential-reading.md); the full source list with URLs and provenance notes is in [References](19-references.md).

## Foundations and full recipes

- Lambert, *The RLHF Book* (living) — the textbook; start here.
- Ouyang et al., *InstructGPT* (2022); Bai et al., *HH-RLHF* (2022) — the founding recipes, human-data methodology.
- Touvron et al., *Llama 2* (2023) — industrial RLHF documented: dual RMs, iterative rejection sampling + PPO.
- Lambert et al., *Tulu 3* (2024); AI2, *Olmo 3* (2025) — the open, ablated, end-to-end recipes (SFT → DPO → RLVR; Dolci mixes, thinking variants).
- Qwen Team, *Qwen 3* (2025) — thinking-mode fusion, GSPO lineage, distillation-heavy family building.

## SFT, LoRA, and data

- Zhou et al., *LIMA* (2023) — quality-over-quantity for demonstrations.
- Hu et al., *LoRA* (2021); Dettmers et al., *QLoRA* (2023); Thinking Machines, *LoRA Without Regret* (2025) — when adapters match full FT and the config that makes it true.
- Wang et al., *Self-Instruct* (2022) → Xu et al., *Evol-Instruct/WizardLM* (2023) → Ge et al., *PersonaHub* (2024) → Xu et al., *Magpie* (2024) — the synthetic-data lineage.
- Muennighoff et al., *s1: Simple test-time scaling* (2025) — 1k traces, budget forcing; the minimal long-CoT SFT result.

## Preference optimization

- Rafailov et al., *DPO* (2023); Azar et al., *IPO* (2023); Ethayarajh et al., *KTO* (2024); Hong et al., *ORPO* (2024); Meng et al., *SimPO* (2024).
- Cui et al., *UltraFeedback* (2023) — the on-policy + AI-judge pair-generation pattern.
- Tunstall et al., *Zephyr* (2023) — distilled DPO popularized; the first open proof it works.

## Reward models and judges

- Gao et al., *RM Overoptimization* (2022); Ramé et al., *WARM* (2024) — Goodhart curves; merged RMs.
- Lightman et al., *Let's Verify Step by Step* (2023); Wang et al., *Math-Shepherd* (2023) — PRMs and automated step labels.
- Lambert et al., *RewardBench* (2024) + RewardBench 2 (2025) — and the benchmark≠downstream caveat.
- Zhang et al., *Generative Verifiers/GenRM* (2024); Chen et al., *RM-R1* (2025); *OpenRubrics* (2025) — reasoning judges and rubric rewards.

## RL algorithms

- Schulman et al., *PPO* (2017) + *GAE* (2015) — the ancestors, worth the primary read.
- Shao et al., *DeepSeekMath/GRPO* (2024); Ahmadian et al., *RLOO* (2024); Hu, *REINFORCE++* (2025).
- Yu et al., *DAPO* (2025); Liu et al., *Dr. GRPO* (2025); Zheng et al., *GSPO* (2025); MiniMax, *M1/CISPO* (2025); Yuan et al., *VAPO* (2025) — the stabilization zoo, each tied to a failure mode.

## RLVR and thinking models

- OpenAI, *Learning to Reason with LLMs* (o1, 2024); DeepSeek-AI, *R1* (2025); Kimi Team, *K1.5* (2025) — the founding trio (K1.5 for length control and partial rollouts).
- Yue et al., *Does RLVR Incentivize New Reasoning?* (2025) vs. NVIDIA, *ProRL* (2025) — the elicitation-vs-creation debate, both sides.
- Luo et al., *DeepScaleR* (2025) — small-model RLVR done carefully; staged context growth.
- Cui et al., *Entropy Mechanism of RL for LLMs* (2025) — entropy collapse formalized.

## Agentic RL

- *The Landscape of Agentic RL for LLMs: A Survey* (2025); *A Practitioner's Guide to Multi-Turn Agentic RL* (2025).
- Pan et al., *SWE-Gym* (2024); *SWE-smith* (2025) — training environments for code agents.
- Jin et al., *Search-R1* (2025); ByteDance, *RAGEN* (2025) — tool-use RL patterns and multi-turn stability (StarPO).
- Kimi Team, *Kimi K2* (2025) — agentic-data synthesis at scale; *Kimi-Dev* (2025) — agentless skill priors.

## Distillation and merging

- Agarwal et al., *GKD* (2023); Thinking Machines, *On-Policy Distillation* (2025); survey arXiv:2604.00626 (2026).
- Wortsman et al., *Model Soups* (2022); Yadav et al., *TIES* (2023); Yu et al., *DARE* (2024); Ramé et al., *WARP* (2024).

## Safety, alignment, character

- Bai et al., *Constitutional AI* (2022); Anthropic, *Claude's Constitution* (2026 revision) and character-training posts.
- OpenAI, *Deliberative Alignment* (2024) + *Model Spec*; *STAR-1* (2025) — safety for reasoning models.
- Sharma et al., *Towards Understanding Sycophancy in LMs* (2023); Röttger et al., *XSTest* (2023) — the two failure axes.
- Andriushchenko et al., *AgentHarm* (2024) — agent safety evaluation.

## Infrastructure and frameworks

- Sheng et al., *HybridFlow/verl* (2024); Hugging Face, *Keep the Tokens Flowing* (2026) — the async-RL landscape across the open libraries.
- Fu et al., *AReaL* (2025) — fully-async RL with ablations; *Your Efficient RL Framework Secretly Brings You Off-Policy RL* (2025) + TIS follow-ups — the train–inference mismatch literature.
- Framework repos as documentation: verl, OpenRLHF, TRL, slime, NeMo-RL, Tinker docs — the READMEs and recipe folders are the real spec sheets.

## Evaluation

- Zheng et al., *Judging LLM-as-a-Judge (MT-Bench/Arena)* (2023); Dubois et al., *Length-Controlled AlpacaEval* (2024); Li et al., *Arena-Hard* (2024).
- Zhou et al., *IFEval* (2023); Jain et al., *LiveCodeBench* (2024); *SWE-bench Verified* (2024); *τ-bench / τ²-bench* (2024–25).
- Singh et al., *The Leaderboard Illusion* (2025) — arena gaming; Chapter 14's skepticism, sourced.
