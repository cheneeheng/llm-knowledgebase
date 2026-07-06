# Post-Training an LLM: The Complete Playbook

*From base model to assistant, reasoner, and agent — SFT, preference optimization, reward models, RLHF, RLVR, distillation, safety.*

*Compiled July 2026. Companion to [Training an LLM From Scratch](../pretraining-from-scratch/README.md). Assumes you know what a transformer is and what reinforcement learning is (policy, reward, on/off-policy) but have never post-trained a model. Covers all scales, from a single consumer GPU to a frontier pipeline.*

---

## How to read this report

Every chapter is **layered**, like the pretraining series: it opens with the *conceptual picture* — what the thing is and why it exists — then descends into the *practitioner layer*: concrete numbers, configs, and the specific decisions you will face. Chapters 3–11 follow the pipeline in the order you would run it; Chapters 12–14 cover the systems, frameworks, and evaluation machinery underneath; Chapter 15 assembles everything into end-to-end recipes.

The economics are inverted relative to pretraining. Pretraining is one long, expensive, irreversible bet; post-training is **hundreds of short, cheap, reversible experiments**. A full SFT of an 8B model costs tens of dollars in GPU time; even a serious RL run costs a small fraction of the pretraining that produced the base model. The scarce resources are different: **good evals, good data, and iteration speed** — not FLOPs. The failure modes are different too: pretraining runs die loudly (divergence, hardware); post-training runs fail *silently*, producing a model that scores well and is subtly worse — sycophantic, verbose, reward-hacked, or dumber outside the training distribution. Almost everything below is ultimately about noticing that before your users do.

## Chapters

| # | File | Contents |
|---|---|---|
| 1 | [The Big Picture](01-big-picture.md) | What post-training is, the modern pipeline, a short history (InstructGPT → DPO era → R1 → 2026), the core mental models |
| 2 | [Scoping and Base Models](02-scoping-and-base-models.md) | Do you even need to post-train, choosing a base model, licenses, cost by ambition level |
| 3 | [Data](03-data.md) | Prompts, completions, preferences, verifiable tasks, trajectories; synthetic generation; human annotation ops; decontamination |
| 4 | [Supervised Fine-Tuning](04-sft.md) | Chat templates, loss masking, packing, hyperparameters, full FT vs LoRA/QLoRA, long-CoT SFT, rejection sampling |
| 5 | [Preference Optimization](05-preference-optimization.md) | DPO and its family (IPO, KTO, ORPO, SimPO), data construction, where DPO still fits in 2026 |
| 6 | [Reward Models](06-reward-models.md) | Bradley–Terry RMs, generative/rubric reward models, process reward models, overoptimization, RewardBench and its limits |
| 7 | [RL Algorithms](07-rl-algorithms.md) | PPO → GRPO → DAPO/GSPO/CISPO; KL penalties; advantage estimation; the hyperparameters that matter; what to actually use |
| 8 | [RLVR and Thinking Models](08-rlvr-and-thinking-models.md) | Verifiable rewards, the R1 recipe, verifiers, curriculum, length dynamics, entropy collapse, multi-stage recipes |
| 9 | [Agentic Post-Training](09-agentic-post-training.md) | Multi-turn RL, environments, reward design for agents, credit assignment, the async infra it forces |
| 10 | [Distillation and Merging](10-distillation-and-merging.md) | Off-policy vs on-policy distillation, GKD, when distillation beats RL, model merging in real recipes |
| 11 | [Safety, Alignment, and Character](11-safety-and-character.md) | Refusal training, Constitutional AI / RLAIF, deliberative alignment, model specs, character training, red-teaming |
| 12 | [Infrastructure](12-infrastructure.md) | The trainer+rollout+verifier system, colocated vs disaggregated, sync vs async, weight sync, train–inference mismatch, sandboxes |
| 13 | [Frameworks](13-frameworks.md) | TRL, verl, OpenRLHF, NeMo-RL, slime/vime, Axolotl/LLaMA-Factory/Unsloth, Tinker — which to pick at which scale |
| 14 | [Evaluation](14-evaluation.md) | The eval stack, LLM-as-judge and its biases, contamination, RL-specific monitoring, statistical hygiene |
| 15 | [Playbooks by Scale](15-playbooks-by-scale.md) | Concrete end-to-end recipes: single GPU, one node, small lab, frontier pipeline |
| 16 | [Failure Modes](16-failure-modes.md) | Reward hacking, entropy collapse, forgetting, length explosion, masking bugs, judge gaming — symptom, cause, fix |
| 17 | [Essential Reading](17-essential-reading.md) | The twelve must-read sources, as an ordered reading sequence |
| 18 | [Reading List](18-reading-list.md) | A curated path deeper, by topic |
| 19 | [References](19-references.md) | Full source list with provenance notes |

## Scope

This report starts where the pretraining series ends: you have (or downloaded) a **base model** — it completes text, knows a great deal, and does nothing you ask. It ends with a deployed assistant, reasoner, or agent, and covers everything that happens in between. Pretraining, midtraining data design, and inference serving are out of scope, though Chapter 1 maps the (moving) boundary with midtraining and Chapter 12 necessarily covers inference engines as RL infrastructure.

## The through-line

Post-training rewards **iteration discipline over scale**. The teams that win are not the ones with the biggest RL runs; they are the ones with the best evals, the tightest generate→score→inspect loop, and the habit of actually reading their model's outputs. Every stage of the pipeline optimizes a *proxy* — a teacher's outputs, a preference model, a verifier — and every proxy can be gamed. Your real job is to keep the proxy honest: decontaminate the data, audit the reward, read the transcripts, and hold out evals the training loop has never seen. Do that, and the pipeline itself is surprisingly mechanical.
