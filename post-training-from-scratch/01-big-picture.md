# Chapter 1 — The Big Picture

## 1.1 What post-training is

A base model is a text-completion engine. Prompt it with "What is the capital of France?" and it may answer, continue with nine more geography questions, or write a Reddit thread arguing about it — all are plausible continuations of the pretraining distribution. Post-training is everything done to that artifact to turn it into something with a *job*: an assistant that follows instructions, a reasoner that thinks before answering, an agent that operates tools, a model that refuses what it should refuse. Concretely, it is a sequence of much smaller training stages — supervised fine-tuning, preference optimization, reinforcement learning, distillation — each consuming orders of magnitude fewer tokens than pretraining but exerting outsized control over *behavior*.

The single most important mental model: **post-training mostly elicits and sharpens what pretraining put in; it rarely creates capability from nothing.** The base model is a vast library of behaviors and knowledge; post-training selects, amplifies, formats, and stabilizes a narrow slice of it. This is why base model choice (Chapter 2) dominates everything downstream, why RL on verifiable rewards works so much better on models pretrained heavily on math and code, and why no amount of SFT will teach an 8B model things it never saw. (The frontier complicates this — at sufficient RL scale models do appear to acquire genuinely new strategies — but as a planning assumption at every scale you will operate at, elicitation is the right prior.)

## 1.2 The modern pipeline

The canonical 2026 pipeline, of which every real recipe is a subset or permutation:

1. **(Midtraining — upstream of you.)** Late pretraining now includes instruction-like and synthetic reasoning data, annealed in before any "post-training" begins. The boundary moved; know what your base model already saw (see pretraining series, Chapter 10).
2. **SFT (supervised fine-tuning).** Imitate curated demonstrations: chat format, instruction following, tool syntax, initial reasoning style. 10⁴–10⁷ examples. The foundation everything else builds on.
3. **Preference optimization.** Usually DPO or a variant: cheap offline optimization on chosen/rejected response pairs. Shapes style, helpfulness, harmlessness.
4. **Reinforcement learning.** The heavyweight stage, two reward regimes that mix in practice:
   - **RLHF** — reward model trained from human/AI preferences scores responses; policy-gradient RL (PPO/GRPO family) optimizes against it. For open-ended qualities: helpfulness, tone, chat.
   - **RLVR** — reinforcement learning with *verifiable* rewards: math checkers, code tests, constraint validators, task environments. This is what created the thinking-model era (o1, DeepSeek-R1) and, extended to tool-using environments, agentic models.
5. **Distillation.** Small models are no longer post-trained from scratch: the big model is trained hard, then its behavior is distilled into the rest of the family (off-policy SFT on its traces, or on-policy distillation against its logits).
6. **Safety integration throughout** — refusal data in SFT, safety preferences in the RM, constitution/spec-based RLAIF, deliberative alignment for reasoners — plus a final regression-and-red-team gate.

Stages are not strictly ordered in practice: labs interleave and loop (rejection-sampling SFT feeding RL feeding new SFT data), run multiple RL stages (reasoning RL, then general RLHF, then agentic RL), and merge model variants. But learn the pipeline in this order — each stage's tooling and failure modes build on the previous one's.

## 1.3 A short history, because the vocabulary requires it

- **2022 — InstructGPT/ChatGPT:** the founding recipe: SFT → reward model → PPO. "RLHF" enters the vocabulary. Anthropic's HH work and Constitutional AI (AI feedback replacing human feedback, RLAIF) land the same era.
- **2023–24 — the DPO era:** Direct Preference Optimization shows you can skip the RM and PPO machinery for preference data; a zoo of offline variants follows (IPO, KTO, ORPO, SimPO). Open post-training (Zephyr, Tulu) becomes reproducible on academic budgets. Llama 2/3 document industrial RLHF: iterative rejection sampling + PPO/DPO at scale.
- **Late 2024–2025 — the reasoning break:** OpenAI's o1, then DeepSeek-R1 with an open recipe: GRPO (critic-free policy gradient) against verifiable rewards produces long chain-of-thought, self-correction, and huge gains on math/code. RLVR displaces preference-only pipelines at the capability frontier; a wave of algorithm papers (DAPO, GSPO, Dr. GRPO, CISPO…) hardens the method. Tulu 3 and Olmo publish complete open recipes ending in RLVR.
- **2025–2026 — RL eats post-training:** RL moves from single-turn answers to multi-turn *agents* in executable environments (SWE agents, computer use, deep research). RL compute at frontier labs grows from a rounding error toward parity with pretraining. On-policy distillation matures into the standard way to make small models good. Reward modeling shifts toward generative judges and rubric rewards for non-verifiable domains. The open-source RL-infrastructure ecosystem (verl, OpenRLHF, slime, NeMo-RL, TRL…) explodes and semi-consolidates.

## 1.4 The core mental models

- **Proxy optimization, everywhere.** Every stage optimizes a stand-in for what you actually want: a teacher's demonstrations (SFT), a preference model (RLHF), a verifier (RLVR), a judge (evals). Optimize any proxy hard enough and you get the proxy, not the goal — Goodhart's law is the physics of post-training. The craft is keeping proxies honest and knowing when to stop.
- **The KL budget.** Post-training drags the policy away from the base/reference distribution. Drift a little: behavior changes. Drift a lot: capabilities, calibration, and diversity degrade ("alignment tax", catastrophic forgetting, entropy collapse). Most algorithms have an explicit or implicit knob (KL penalty, DPO's β, clipping, LR, epochs) controlling how far you move per unit of reward. Think of every stage as *spending* divergence from a trusted distribution and demand reward for it.
- **On-policy beats off-policy, at a cost.** Training on the model's *own* outputs (RL, on-policy distillation, rejection sampling) avoids the compounding-error problem of imitating someone else's distribution, and is why RL generalizes better than SFT on the same data. The cost is infrastructure: on-policy methods need fast generation inside the training loop, which is most of Chapter 12.
- **FLOPs are cheap; signal is expensive.** The binding constraints are high-quality prompts, reliable reward signal, and eval coverage. A mediocre team with 10× compute loses to a good team with 10× better evals and data. Budget accordingly.
- **Read the transcripts.** Aggregate metrics hide every failure mode that matters (reward hacking, sycophancy, format collapse). Every strong post-training team has a culture of reading raw model outputs daily. No dashboard replaces it.
