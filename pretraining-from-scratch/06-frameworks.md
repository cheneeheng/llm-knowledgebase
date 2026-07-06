# Chapter 6 — Frameworks

## 6.1 Conceptual picture

You will not write a pretraining stack from scratch — you will use a battle-tested framework that already solves parallelism, checkpointing, fused kernels, mixed precision, and (increasingly) fault tolerance. Choosing well saves months. The choice is really three questions: (a) how much scale do you need, (b) how much will you customize, and (c) which ecosystem are you hiring from.

What a framework actually owns: distributed process groups and collectives (NCCL/RCCL), sharding of weights/activations/optimizer state across parallelism axes, fused and FlashAttention kernels, BF16/FP8 plumbing, activation checkpointing, distributed/async checkpointing, dataloader integration, and restart logic. Your job is the **configuration** — model shape, parallelism degrees, batch size, optimizer, schedule, data — plus the data pipeline and evaluation harness. Getting the config right (Chapter 7) is most of the systems battle.

## 6.2 The 2026 shortlist

| Framework | One-liner | Sweet spot |
|---|---|---|
| **Megatron-LM / Megatron-Core** (NVIDIA) | The industrial-strength reference: TP/PP/CP/EP, FP8 via Transformer Engine, up to ~47% MFU on H100 fleets; the substrate under NeMo and many frontier trainers. Steeper learning curve, config-heavy, more rigid | Serious runs ≥ tens of B params, thousands of GPUs, maximum throughput, battle-tested MoE at extreme scale |
| **torchtitan** (PyTorch team) | PyTorch-native, clean and compact: composable FSDP2 + TP + PP + CP ("4D parallelism"), torch.compile, distributed checkpointing, FP8. Much easier to read and modify than Megatron at modest MFU cost; rapidly maturing toward frontier-capable | 1B–100B-class runs where hackability matters; the best *learning* codebase that also scales |
| **NVIDIA NeMo (2.0)** | Enterprise packaging of Megatron-Core: recipes, config system, data curation (NeMo Curator), cluster tooling, multimodal | Teams that want a supported product and NVIDIA hand-holding |
| **DeepSpeed** (Microsoft) | Pioneered ZeRO (sharded optimizer/grads/params — the idea behind FSDP) and ZeRO-Offload/Infinity. Still capable and widely deployed; for the largest runs most teams have shifted to FSDP2/Megatron/torchtitan | Memory-tight setups, offload, heterogeneous hardware |
| **MaxText / Levanter** (JAX) | JAX/XLA-native, the reference for TPU (also runs on GPU); very high MFU, clean scaling | TPU commitment |
| **GPT-NeoX, OLMo-core, nanotron, litGPT, Modalities** | Open research trainers with published recipes (OLMo especially: data + code + checkpoints, fully reproducible) | Reproducing open science; mid-scale research; readable mid-size codebases |
| **nanoGPT / modded-nanoGPT** | Minimal educational trainers — read the whole thing in an afternoon. The modded-nanoGPT "speedrun" lineage is also where optimizer innovation (Muon) was born | Learning the internals; ladder experiments |
| **HF Transformers + Accelerate / Axolotl / Unsloth** | Ergonomic fine-tuning-first stacks; good at *continued* pretraining | Not the tool for from-scratch cluster pretraining |

AMD note: **Primus** wraps Megatron-LM and torchtitan backends for MI300-class GPUs with MLPerf-validated results, so the same skills transfer.

## 6.3 How to choose by scale

- **Learning / ~1B ladder runs:** nanoGPT/modded-nanoGPT, litGPT, or nanotron — readable, fast to iterate, enough to fit your scaling law.
- **Serious 1–8B on NVIDIA:** torchtitan (PyTorch-native, hackable) or NeMo/Megatron-Core (maximum optimization and MoE support).
- **70B–frontier dense/MoE on NVIDIA:** Megatron-Core (or NeMo on top). This is what the ecosystem, kernels, and published recipes assume — and where its advanced pipeline schedules and extreme-scale MoE/EP pay for the learning curve.
- **Anything on TPU:** MaxText (or Levanter).
- **Anything on AMD:** Primus / ROCm builds of Megatron and torchtitan.

## 6.4 The recommended first-timer path

1. **Weekend one:** read and run **nanoGPT/modded-nanoGPT** on a single node. Learn what a training step, a dataloader, a LR schedule, and a checkpoint actually are in code.
2. **Real runs:** **torchtitan.** Best-documented modern codebase whose abstractions match how PyTorch distributed actually works; scales to serious sizes; proven Llama-3-style reference configs.
3. **Graduate to Megatron-Core/NeMo** when pushing ≥50–100B scale, extreme-scale MoE, or when you need the last 5–10% MFU and the most advanced pipeline schedules.

Whatever you pick, budget real time for the unglamorous glue: dataloader integration, checkpoint conversion (training format → Hugging Face for evals), and eval-harness hookup. That glue is always underestimated.
