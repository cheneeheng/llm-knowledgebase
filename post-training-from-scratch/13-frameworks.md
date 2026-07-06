# Chapter 13 — Frameworks

## 13.1 Conceptual picture

Two distinct tool categories, often conflated. **Fine-tuning stacks** (SFT/DPO/LoRA) are mature commodities — pick by ergonomics. **RL stacks** own Chapter 12's welded trainer+rollout+reward system — pick by scale and architecture, and expect to read source code. The RL ecosystem exploded post-R1 (a dozen-plus credible libraries by 2026) and only semi-consolidated; the practical shortlist below is opinionated. One warning that applies to all of them: RL frameworks carry *silent correctness* risk — masking bugs, template mismatches, logprob-gap issues — that fine-tuning stacks mostly don't. Whatever you adopt, run its smallest end-to-end example, decode and read the batches (Chapter 4), and check the logprob gap (Chapter 12) before believing any training curve.

## 13.2 Fine-tuning stacks (SFT / DPO / LoRA)

| Framework | One-liner | Sweet spot |
|---|---|---|
| **TRL** (Hugging Face) | The ecosystem default: SFTTrainer/DPOTrainer/GRPOTrainer, every PEFT method, vLLM integration; benefits from the Hub gravity well | Default for 1 GPU–few nodes; also the gentlest RL on-ramp |
| **Axolotl** | YAML-config fine-tuning over HF stack: multi-GPU (FSDP/DeepSpeed), packing, multimodal, every dataset format | Production fine-tuning without writing trainers |
| **LLaMA-Factory** | Web UI + CLI, enormous model/method coverage, zero-code | Fastest first fine-tune; broad method experimentation |
| **Unsloth** | Hand-fused Triton kernels: ~2× speed, ~½ memory on single GPU; QLoRA champion | Consumer GPUs / Colab; the hobbyist standard |
| **torchtune → PyTorch post-training (forge lineage)** | PyTorch-native recipes, hackable; original torchtune wound down in 2025 with successor tooling under the PyTorch org | PyTorch purists; check current repo status before adopting |
| **OpenRLHF / verl (SFT modes)** | The RL stacks also do SFT/DPO | Convenient when already using them for RL |

## 13.3 RL stacks

| Framework | One-liner | Sweet spot |
|---|---|---|
| **TRL (GRPOTrainer)** | Simplest credible GRPO: colocated vLLM, LoRA-friendly, HF-native | First RLVR experiments, ≤8–16 GPUs, short-to-medium CoT |
| **verl** (ByteDance lineage) | The open production standard: HybridFlow programming model, FSDP *or* Megatron trainer, vLLM/SGLang rollouts, colocated + disaggregated + async, MoE support, most algorithm recipes (PPO/GRPO/DAPO/GSPO/…) land here first; steep config surface | Serious RLVR/agentic RL, 16–1000s of GPUs; the safe "nobody got fired" choice |
| **OpenRLHF** | Ray + DeepSpeed + vLLM; readable, research-friendly, full method set incl. PRM/iterative-DPO paths | Multi-node research without verl's complexity appetite |
| **NeMo-RL** (NVIDIA) | Megatron-core-native RL, enterprise support, NeMo Gym environments | NVIDIA-committed enterprises; very large MoE on Megatron |
| **slime** (THUDM/Zhipu lineage) | Megatron + SGLang bridged by a clean data buffer; lightweight, async-first, GLM-proven | Large-model RL with Megatron muscle minus framework sprawl |
| **vime** (vLLM project, 2026) | slime's design philosophy rebuilt vLLM-native as the official vLLM-ecosystem RL layer | Watch it; likely consolidation point for the vLLM world |
| **AReaL** | Fully-async specialist (in-flight updates, interruptible generation) with the strongest published async ablations | Long-horizon/agentic where staleness engineering is the point |
| **SkyRL, ROLL, RAGEN, rllm** | Agentic-RL-focused stacks (trajectory abstractions, env integration, disaggregated rollout services) | Agent training research; evaluate against verl+your-envs first |
| **Verifiers + Prime Intellect environments hub** | Standard interface for envs/rubric rewards + a community hub of ready environments; trains via its own GRPO or plugs into other stacks | Building/sharing environments; small-scale agentic RL |
| **Tinker** (Thinking Machines) | Managed training-as-API: you write the loop (SFT/RL/OPD primitives), they run distributed LoRA training on their fleet | Full algorithmic control with zero infra ownership; also the reference OPD implementation |

## 13.4 How to choose

- **Learning / single GPU:** Unsloth or LLaMA-Factory for SFT/DPO; TRL GRPOTrainer for a first taste of RLVR on a 1.5–4B model.
- **Practitioner, 1 node:** TRL or Axolotl for SFT/DPO (full-param 8B); TRL GRPO or verl-small for RL. Tinker if you'd rather rent the infra problem away.
- **Small lab, 16–128 GPUs, reasoning/agentic:** **verl** by default — the ecosystem's recipes, algorithms, and bug-fixes concentrate there. OpenRLHF if you want readable internals; slime if you're Megatron-comfortable and async-hungry.
- **Enterprise/NVIDIA-locked:** NeMo-RL. **Frontier-adjacent MoE:** verl-Megatron, slime, or a fork you now own.
- **Agentic:** whichever RL core you chose + Verifiers-style env interfaces; adopt an agent-focused stack only when trajectory infrastructure (partial rollouts, env fleets) becomes your actual bottleneck.

Ecosystem realities: APIs churn fast — pin versions per project and record exact configs with checkpoints; algorithm-of-the-month lands as a config flag within weeks (you rarely need to switch frameworks to chase papers); and the glue — data converters, eval-harness hookup, checkpoint→HF export for serving — is, as ever, underestimated. Budget for it.
