# Chapter 11 — Infrastructure and Frameworks

## 11.1 Conceptual picture

Multimodal training runs on the text-training stack (pretraining/post-training series) with a set of specific deltas that come from one root cause: **images and video are big, variable-sized, and turn into wildly variable-length token sequences.** A text batch is uniform-ish; a multimodal batch mixes a text-only example, an image example costing 700 tokens, and a 12-tile document costing 9,000 tokens. That variability, plus the extra data-pipeline burden of decoding images/video and running a vision encoder in the loop, is what changes about the infrastructure. This chapter covers those deltas and the framework landscape; everything shared with text training (parallelism, precision, checkpointing) carries over from the earlier series.

## 11.2 What changes from text training

- **The data pipeline gets heavier.** You're not just reading tokens — you're decoding images (JPEG/PNG), decoding/sampling video frames, resizing/tiling, and often running the vision encoder to produce features. This is CPU/IO-intensive and can starve the GPUs if not pipelined well. Options: precompute-and-cache vision features (fast, but fixes resolution and precludes encoder training), or encode on-the-fly (flexible, needs enough data-loading parallelism). Video decoding especially is a bottleneck — frame extraction is expensive.
- **Sequence lengths are highly variable.** Visual tokens make sequence length jump around by 10× within a batch (Chapter 2.5). This breaks naive batching and hurts efficiency (padding waste, or straggler examples). Solutions: length-based bucketing/sorting, sequence packing with careful attention masking (don't let one example's tokens attend to another's — and don't split an image), and dynamic batch sizing. This is the multimodal analogue of the packing discipline from the post-training series, harder because of the variance.
- **Multi-component optimization.** You have encoder + connector + LLM, each possibly at a different LR and freeze state (Chapter 4). The trainer must support differential learning rates and per-component freezing cleanly — not all text trainers do.
- **Memory profile shifts.** Long visual-token sequences inflate activations and KV; a "small" VLM can have large per-example memory from a high-res image. Budget as in the inference series, but for training activations.
- **Checkpoint complexity.** You're saving three components (and their optimizer states); conversions and releases must keep encoder/connector/LLM consistent (a mismatched encoder-connector pair is silently broken — the multimodal version of the inference series' conversion-corruption failure).

## 11.3 The framework landscape

| Framework | Best for | Notes |
|---|---|---|
| **LLaVA / LLaVA-NeXT codebases** | Learning, reproducing the canonical recipe | The reference adapter-VLM training code; start here to understand the recipe |
| **LLaMA-Factory** | Config-driven VLM fine-tuning, many models | Broad model support, ergonomic; good for SFT/LoRA of existing VLMs |
| **Axolotl** | Config-driven multimodal fine-tuning | Added multimodal support; familiar if you used it for text |
| **ms-swift (ModelScope)** | Wide VLM/omni model coverage, training + infer | Strong for Qwen-VL/Omni-family and many others |
| **Unsloth** | Fast single-GPU VLM LoRA | Fastest/cheapest entry point for adapting a VLM on one card |
| **NeMo / Megatron (multimodal)** | Large-scale / native / omni training | The heavyweight path for frontier VLM/native training; multimodal support in NeMo |
| **LLaVA-OneVision / open frameworks** | Fully-open reproducible VLM training | End-to-end open data+code for democratized VLM training |
| **Native model training** (custom) | Omni / any-to-any | Frontier native models often use custom Megatron/NeMo-based stacks |

The selection rule mirrors the text series, shifted to VLMs: **Unsloth/LLaMA-Factory/Axolotl for fine-tuning or LoRA-adapting an existing VLM on modest hardware** (most teams); **LLaVA-lineage or LLaVA-OneVision to train an adapter VLM from an LLM + encoder**; **NeMo/Megatron for large-scale, native, or omni training** (frontier). Don't reach for a Megatron native-multimodal setup to LoRA a Qwen-VL — Unsloth does it in an afternoon.

## 11.4 Cost

Multimodal training cost anchors, relative to the text series:

- **LoRA-adapting an existing VLM** (domain instruction tuning) — hours to a day on 1–8 GPUs; tens to hundreds of dollars. The common case.
- **Training an adapter VLM from LLM + encoder** (align + joint tune on a good mixture) — the connector-alignment stage is cheap (small data, connector-only); the joint stage is the cost, comparable to an SFT-scale run scaled by the visual-token inflation. An 8B VLM: low thousands of dollars on a node for a few days.
- **Frontier / native / omni** — a full pretraining-scale undertaking (native models train jointly at scale), so pretraining-series economics apply, times modalities. Millions for a frontier omni model.

The multimodal-specific cost driver is the **visual-token inflation**: every image multiplies the effective sequence length, so a VLM run processes more tokens per example than a text run on the same "number of examples." Budget by *tokens* (including visual), not by number of images.

## 11.5 Decisions

1. **Run on the text stack with multimodal deltas** — the shared machinery (parallelism, precision, checkpointing) carries over; budget for the new burdens.
2. **Engineer the data pipeline** for image/video decode + optional on-the-fly encoding; precompute-and-cache features when you won't train the encoder, encode on-the-fly when you will.
3. **Handle sequence-length variance** with bucketing and careful packing (never split an image, never cross-attend examples) — it's the main efficiency delta.
4. **Use a trainer that supports per-component freezing and differential LRs** (encoder/connector/LLM) and keep the three components consistent across checkpoints.
5. **Pick the framework by scale:** Unsloth/LLaMA-Factory/Axolotl to adapt an existing VLM, LLaVA-lineage to build an adapter VLM, NeMo/Megatron for native/omni; **budget by total tokens including visual-token inflation**, not by image count.

Next: playbooks — the series assembled into concrete recipes by scale.
