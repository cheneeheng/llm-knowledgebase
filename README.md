# llm-knowledgebase

A complete, practitioner-oriented map of building large language models, covering the full lifecycle from cluster procurement to a deployed, served, governed product. Five layered report series, each written for someone who knows what a transformer is but has never done the thing.

## The lifecycle series

These four follow the training-to-serving lifecycle in order. Read them in sequence, or jump to the stage you're working on.

1. [Training an LLM From Scratch: The Complete Pretraining Playbook](pretraining-from-scratch/README.md) — cluster procurement to your first trillion tokens; everything up to (but not including) post-training.
2. [Midtraining and Continued Pretraining: The Missing Middle](midtraining-and-continued-pretraining/README.md) — the stage between a base model and a post-trained one: annealing, continued pretraining, domain adaptation, long-context extension, tokenizer surgery, reasoning injection.
3. [Post-Training an LLM: The Complete Playbook](post-training-from-scratch/README.md) — base model to assistant, reasoner, and agent: SFT, preference optimization, reward models, RLHF, RLVR, agentic RL, distillation, safety.
4. [Serving an LLM: The Complete Inference Playbook](inference-and-serving/README.md) — trained checkpoint to production endpoint: inference anatomy, KV cache, batching, serving engines, quantization, speculative decoding, reasoning-model serving, hardware, and economics.

## The cross-cutting series

These two cover concerns and model types that span multiple stages rather than living in one.

5. [Training a Multimodal Model: The Complete Playbook](multimodal-training/README.md) — text LLM to a model that sees, hears, and speaks: vision encoders, connectors, native fusion, image/audio/video data, omni models, multimodal post-training, and evaluation.
6. [Cross-Cutting Topics: The Concerns That Span the Whole Lifecycle](cross-cutting-topics/README.md) — model selection and licensing, small and on-device models, embeddings and retrieval, interpretability, production security, privacy and compliance, and the org/compute/improvement loop.

## How the series fit together

Pretraining produces a **base model**; midtraining refines it into a strong, long-context, RL-ready base; post-training turns it into an **assistant, reasoner, or agent**; inference serves it affordably at scale. The multimodal series runs parallel to all four (a model that sees/hears is trained and served with the same machinery plus modality-specific deltas). The cross-cutting series wraps the whole thing with the constraints — legal, economic, security, organizational — that decide whether a good model becomes a shippable product.

The deepest recurring lesson across all five: **everything downstream of data is a proxy for what you actually want, every proxy can be gamed, and the discipline that separates teams that ship from teams that thrash is keeping the proxies honest** — decontaminated data, audited rewards, red-teamed guardrails, and held-out evals the training loop has never seen.

*Compiled July 2026. Each series is grounded in primary technical reports and current tooling, with an essential-reading chapter and full references. Numbers reflect mid-2026 practice and are defaults to validate on your own workload, not constants.*
