# Chapter 19 — References and Sources

Full source list, grouped by the chapter they inform. Where a claim in this report rests on a specific number (hyperparameters, costs, benchmark deltas, dataset sizes), the underlying source is listed here.

**Provenance note:** primary technical claims are anchored to arXiv papers, official model technical reports, lab blogs, and framework repositories. Secondary syntheses (Raschka, framework round-ups, the RLHF Book) are used for landscape framing and cross-checked against the primary reports they cite. Sources dated 2026 were gathered in a July 2026 research pass; costs and framework details are mid-2026 snapshots and will drift — re-verify anything you act on against the primary source at time of use. This field moves monthly.

## Ch. 1–2 — Big picture, scoping, recipes end-to-end

- Ouyang et al., *Training language models to follow instructions with human feedback* (InstructGPT), 2022 — arXiv:2203.02155.
- Bai et al., *Training a Helpful and Harmless Assistant with RLHF* (HH-RLHF), 2022 — arXiv:2204.05862.
- Touvron et al., *Llama 2: Open Foundation and Fine-Tuned Chat Models*, 2023 — arXiv:2307.09288. Dual RMs; iterative rejection sampling + PPO.
- Meta AI, *The Llama 3 Herd of Models*, 2024 — arXiv:2407.21783. Post-training sections: SFT+DPO rounds, data mix, model averaging.
- Lambert et al., *Tülu 3: Pushing Frontiers in Open Language Model Post-Training*, 2024 — arXiv:2411.15124. 939k-prompt SFT mix; DPO ablations; RLVR introduction. Companion: AI2 blog https://allenai.org/blog/tulu-3-technical · 405B scaling https://allenai.org/blog/tulu-3-405b
- OLMo Team, *2 OLMo 2 Furious*, 2025 — arXiv:2501.00656 · AI2, *Olmo 3* (Nov 2025) — https://allenai.org/blog/olmo3. The Dolci post-training suite; fully open "model flow." Secondary: C. Wolfe, *Olmo 3 and the Open LLM Renaissance* — https://cameronrwolfe.substack.com/p/olmo-3
- Qwen Team, *Qwen3 Technical Report*, 2025 — arXiv:2505.09388. Thinking-mode fusion; distillation-built family.
- Lambert, *The RLHF Book* — https://rlhfbook.com. The field's textbook; living document.
- llm-stats, *Post-Training in 2026: GRPO, DAPO, RLVR & Beyond* — https://llm-stats.com/blog/research/post-training-techniques-2026. 2026 landscape synthesis (pipeline stages, Nemotron 3 Super SFT scale, self-play).

## Ch. 3 — Data

- Zhou et al., *LIMA: Less Is More for Alignment*, 2023 — arXiv:2305.11206.
- Wang et al., *Self-Instruct*, 2022 — arXiv:2212.10560 · Xu et al., *WizardLM/Evol-Instruct*, 2023 — arXiv:2304.12244 · Ge et al., *Scaling Synthetic Data Creation with 1B Personas* (PersonaHub), 2024 — arXiv:2406.20094 · Xu et al., *Magpie*, 2024 — arXiv:2406.08464.
- Cui et al., *UltraFeedback*, 2023 — arXiv:2310.01377. On-policy + AI-judge preference pattern.
- Tulu 3 datasets — https://huggingface.co/collections/allenai/tulu-3-datasets · SmolTalk (HF) · WildChat, LMSYS-Chat-1M · NuminaMath.
- Contamination: Yao et al., *Detecting Data Contamination from RL Post-training*, 2025 — arXiv:2510.09259 · *The Impact of Post-training on Data Contamination*, 2026 — arXiv:2601.06103.

## Ch. 4 — SFT

- Hu et al., *LoRA*, 2021 — arXiv:2106.09685 · Dettmers et al., *QLoRA*, 2023 — arXiv:2305.14314.
- Thinking Machines, *LoRA Without Regret*, 2025 — https://thinkingmachines.ai/blog/lora/. All-layer LoRA, ~10× LR, capacity conditions.
- Muennighoff et al., *s1: Simple test-time scaling*, 2025 — arXiv:2501.19393. 1k traces; budget forcing.
- Yuan et al., *RAFT/rejection-sampling fine-tuning lineage*, 2023 — arXiv:2304.06767; STaR — arXiv:2203.14465.

## Ch. 5 — Preference optimization

- Rafailov et al., *DPO*, 2023 — arXiv:2305.18290 · Azar et al., *IPO*, 2023 — arXiv:2310.12036 · Ethayarajh et al., *KTO*, 2024 — arXiv:2402.01306 · Hong et al., *ORPO*, 2024 — arXiv:2403.07691 · Meng et al., *SimPO*, 2024 — arXiv:2405.14734.
- Tunstall et al., *Zephyr: Direct Distillation of LM Alignment*, 2023 — arXiv:2310.16944.
- Pal et al., *Smaug/DPOP* (chosen-logprob collapse), 2024 — arXiv:2402.13228.

## Ch. 6 — Reward models

- Gao et al., *Scaling Laws for Reward Model Overoptimization*, 2022 — arXiv:2210.10760.
- Ramé et al., *WARM: Weight-Averaged Reward Models*, 2024 — arXiv:2401.12187.
- Lightman et al., *Let's Verify Step by Step*, 2023 — arXiv:2305.20050 · Wang et al., *Math-Shepherd*, 2023 — arXiv:2312.08935.
- Lambert et al., *RewardBench*, 2024 — arXiv:2403.13787 · Malik et al., *RewardBench 2*, 2025 — arXiv:2506.01937 (incl. benchmark≠downstream findings).
- Zhang et al., *Generative Verifiers*, 2024 — arXiv:2408.15240 · *Generative Reward Models*, 2024 — arXiv:2410.12832 · Chen et al., *RM-R1: Reward Modeling as Reasoning*, 2025 — arXiv:2505.02387.
- Rubrics: *OpenRubrics*, 2025 — arXiv:2510.07743 · *RUBRIC-ARROW*, 2026 — arXiv:2605.29156 · rubric-based reward overview — https://www.emergentmind.com/topics/rubric-based-reward-modeling-rlrr

## Ch. 7 — RL algorithms

- Schulman et al., *PPO*, 2017 — arXiv:1707.06347 · *GAE*, 2015 — arXiv:1506.02438.
- Shao et al., *DeepSeekMath* (GRPO), 2024 — arXiv:2402.03300.
- Ahmadian et al., *Back to Basics: RLOO*, 2024 — arXiv:2402.14740 · Hu, *REINFORCE++*, 2025 — arXiv:2501.03262.
- Yu et al., *DAPO*, 2025 — arXiv:2503.14476 · Liu et al., *Understanding R1-Zero-Like Training* (Dr. GRPO), 2025 — arXiv:2503.20783 · Zheng et al., *GSPO*, 2025 — arXiv:2507.18071 · MiniMax, *MiniMax-M1* (CISPO), 2025 — arXiv:2506.13585 · Yuan et al., *VAPO*, 2025 — arXiv:2504.05118 · *GMPO*, 2025 — arXiv:2507.20673.

## Ch. 8 — RLVR and thinking models

- OpenAI, *Learning to Reason with LLMs* (o1), 2024 — https://openai.com/index/learning-to-reason-with-llms/
- DeepSeek-AI, *DeepSeek-R1*, 2025 — arXiv:2501.12948 · Kimi Team, *Kimi k1.5*, 2025 — arXiv:2501.12599 (length control; partial rollouts).
- Yue et al., *Does RLVR Incentivize New Reasoning Beyond the Base Model?*, 2025 — arXiv:2504.13837 · NVIDIA, *ProRL*, 2025 — arXiv:2505.24864 · *BroRL*, 2025 — arXiv:2510.01180 · *Echo Chamber: RL Amplifies Pretraining Behaviors*, 2025 — arXiv:2504.07912.
- Cui et al., *The Entropy Mechanism of RL for LLM Reasoning*, 2025 — arXiv:2505.22617.
- Luo et al., *DeepScaleR*, 2025 — https://github.com/agentica-project (blog + repo); staged context growth.
- Raschka, *The State of RL for LLM Reasoning*, 2025 — https://magazine.sebastianraschka.com/p/the-state-of-llm-reasoning-model-training
- *Language Models that Think, Chat Better*, 2025 — arXiv:2509.20357 (RLVR-style training beyond verifiable domains).

## Ch. 9 — Agentic post-training

- *The Landscape of Agentic RL for LLMs: A Survey*, 2025 — arXiv:2509.02547 · *A Practitioner's Guide to Multi-Turn Agentic RL*, 2025 — arXiv:2510.01132.
- Pan et al., *SWE-Gym*, 2024 — arXiv:2412.21139 · *SWE-smith*, 2025 — arXiv:2504.21798 · Kimi Team, *Kimi-Dev: Agentless Training as Skill Prior*, 2025 — arXiv:2509.23045 · Kimi Team, *Kimi K2: Open Agentic Intelligence*, 2025 — arXiv:2507.20534.
- Jin et al., *Search-R1*, 2025 — arXiv:2503.09516 · *RAGEN/StarPO*, 2025 — arXiv:2504.20073 · *AgentRL*, 2025 — arXiv:2510.04206 · *Training Long-Context, Multi-Turn SWE Agents with RL*, 2025 — arXiv:2508.03501 · *ProRL Agent: Rollout-as-a-Service*, 2026 — arXiv:2603.18815 · *SENTINEL: Failure-Driven RL for Tool Agents*, 2026 — arXiv:2606.12908.
- Environments/benchmarks: τ-bench — arXiv:2406.12045; τ²-bench; OSWorld — arXiv:2404.07972; terminal-bench; NeMo Gym (NVIDIA); Prime Intellect Environments Hub + `verifiers` — https://github.com/willccbb/verifiers

## Ch. 10 — Distillation and merging

- Agarwal et al., *GKD: Generalized Knowledge Distillation*, 2023 — arXiv:2306.13649.
- Thinking Machines, *On-Policy Distillation*, 2025 — https://thinkingmachines.ai/blog/on-policy-distillation/ · SDFT recipe — https://tinker-docs.thinkingmachines.ai/cookbook/recipes/sdft/
- *A Survey of On-Policy Distillation for LLMs*, 2026 — arXiv:2604.00626 · *Rethinking On-Policy Distillation*, 2026 — arXiv:2604.13016 · curated list — https://github.com/chrisliu298/awesome-on-policy-distillation
- *Nemotron-Cascade: Cascaded RL*, 2025 — arXiv:2512.13607 · *Nemotron-Cascade 2* (cascade RL + multi-domain OPD), 2026 — arXiv:2603.19220.
- Wortsman et al., *Model Soups*, 2022 — arXiv:2203.05482 · Yadav et al., *TIES-Merging*, 2023 — arXiv:2306.01708 · Yu et al., *DARE*, 2024 — arXiv:2311.03099 · Ramé et al., *WARP*, 2024 — arXiv:2406.16768.

## Ch. 11 — Safety, alignment, character

- Bai et al., *Constitutional AI*, 2022 — arXiv:2212.08073 · Lee et al. (Google), *RLAIF vs RLHF*, 2023 — arXiv:2309.00267.
- OpenAI, *Deliberative Alignment*, 2024 — arXiv:2412.16339, https://openai.com/index/deliberative-alignment/ · OpenAI Model Spec — https://model-spec.openai.com
- Anthropic — *Claude's Constitution* (Jan 2026 revision) and character-training posts — https://www.anthropic.com/news
- *STAR-1: Safer Alignment of Reasoning LLMs with 1K Data*, 2025 — arXiv:2504.01903.
- Sharma et al., *Towards Understanding Sycophancy in Language Models*, 2023 — arXiv:2310.13548 · Röttger et al., *XSTest*, 2023 — arXiv:2308.01263 · *OR-Bench*, 2024 — arXiv:2405.20947.
- Andriushchenko et al., *AgentHarm*, 2024 — arXiv:2410.09024 · *LLM Safety: A Holistic Survey*, 2024 — arXiv:2412.17686.

## Ch. 12–13 — Infrastructure and frameworks

- Sheng et al., *HybridFlow: A Flexible and Efficient RLHF Framework* (verl), 2024 — arXiv:2409.19256 · repo — https://github.com/verl-project/verl (26Q2 roadmap issue #5836: PD-disaggregated rollout, FP8 sync, hybrid modes).
- Hu et al., *OpenRLHF*, 2024 — arXiv:2405.11143 · TRL — https://github.com/huggingface/trl · NeMo-RL — https://github.com/NVIDIA-NeMo/RL · slime — https://github.com/THUDM/slime · vime (vLLM project, June 2026) — https://github.com/vllm-project/vime, https://vllm.ai/blog/2026-06-09-announcing-vime · AReaL, 2025 — arXiv:2505.24298 · SkyRL — https://github.com/NovaSky-AI/SkyRL · Tinker — https://thinkingmachines.ai/tinker
- Hugging Face, *Keep the Tokens Flowing: Lessons from 16 Open-Source RL Libraries*, 2026 — https://huggingface.co/blog/async-rl-training-landscape
- Guanghan, *RL Infra for Large-Scale Agentic Training*, Feb 2026 — https://blog.guanghan.ai/post/260208_rl_infra/ (GLM-5 production infra: PD disaggregation, FP8/MTP rollouts, DP-aware KV routing).
- Train–inference mismatch: Yao et al., *Your Efficient RL Framework Secretly Brings You Off-Policy RL Training*, 2025 — https://fengyao.notion.site/off-policy-rl (truncated importance sampling); *Jet-RL: On-Policy FP8 RL with Unified Precision Flow*, 2026 — arXiv:2601.14243.
- Systems papers (2025–26): *Seer: Online Context Learning for Fast Synchronous RL*, 2025 — arXiv:2511.14617 · *RollArt: Scaling Agentic RL via Disaggregated Infrastructure*, 2025 — arXiv:2512.22560 · *JigsawRL*, 2026 — arXiv:2604.23838 · *RLBoost: Preemptible Resources for Cost-Efficient RL*, 2025 — arXiv:2510.19225.
- Fine-tuning stacks: Axolotl — https://axolotl.ai · LLaMA-Factory — arXiv:2403.13372, https://llamafactory.readthedocs.io · Unsloth — https://unsloth.ai · comparative round-ups (secondary): https://www.spheron.network/blog/axolotl-vs-unsloth-vs-torchtune/ · https://theaiengineer.substack.com/p/unsloth-vs-axolotl-vs-llama-factory · https://modal.com/blog/fine-tuning-llms

## Ch. 14 — Evaluation

- Zheng et al., *Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena*, 2023 — arXiv:2306.05685.
- Dubois et al., *Length-Controlled AlpacaEval*, 2024 — arXiv:2404.04475 · Li et al., *Arena-Hard / BenchBuilder*, 2024 — arXiv:2406.11939.
- Zhou et al., *IFEval*, 2023 — arXiv:2311.07911 · Jain et al., *LiveCodeBench*, 2024 — arXiv:2403.07974 · OpenAI, *SWE-bench Verified*, 2024 — https://openai.com/index/introducing-swe-bench-verified/
- Singh et al., *The Leaderboard Illusion*, 2025 — arXiv:2504.20879 · *BadJudge* (judge vulnerabilities), 2025 — arXiv:2503.00596 · *PReMISE: Policy Rubrics for LLM Judges*, 2026 — arXiv:2605.30803.
- lm-evaluation-harness (EleutherAI); LMArena — https://lmarena.ai

## Cross-cutting curated indexes

- hscspring, *rl-llm-nlp: post-R1 LLM×RL index* — https://github.com/hscspring/rl-llm-nlp
- *CapTrack: Multifaceted Evaluation of Forgetting in LLM Post-Training*, 2026 — arXiv:2603.06610 (Ch. 4/16 forgetting).
