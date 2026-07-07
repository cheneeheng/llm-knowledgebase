# llm-knowledgebase

This repository contains information about LLM training, covering the full lifecycle from cluster procurement to a deployed, served model.

## Reports

The reports below follow the training lifecycle in order. Read them in sequence, or jump to the stage you're working on.

1. [Training an LLM From Scratch: The Complete Pretraining Playbook](pretraining-from-scratch/README.md) — cluster procurement to your first trillion tokens; everything up to (but not including) post-training.
2. [Midtraining and Continued Pretraining: The Missing Middle](midtraining-and-continued-pretraining/README.md) — the stage between a base model and a post-trained one: annealing, continued pretraining, domain adaptation, long-context extension, tokenizer surgery, reasoning injection.
3. [Post-Training an LLM: The Complete Playbook](post-training-from-scratch/README.md) — base model to assistant, reasoner, and agent: SFT, preference optimization, reward models, RLHF, RLVR, agentic RL, distillation, safety.

Further series (inference and serving, multimodal training, and cross-cutting topics) are in progress.
