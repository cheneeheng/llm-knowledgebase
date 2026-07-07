# Chapter 12 — Playbooks by Scale

Concrete end-to-end multimodal recipes assembling the series. Starting points to adapt, not laws. Read Chapters 2–11 for the *why*.

---

## 12.1 Single GPU — LoRA-adapt an existing VLM

**Goal:** specialize an open VLM (Qwen-VL, InternVL, LLaVA) to your domain/task on one 24–48GB card.

- **Framework:** Unsloth (fastest) or LLaMA-Factory.
- **Start from** a strong pretrained VLM (encoder + connector + LLM already aligned) — don't build from parts for this.
- **Method:** LoRA the LLM (all linear layers), keep encoder frozen, keep/lightly-tune the connector. Visual instruction data for your domain (Chapter 6) + some general VQA + a little text-only for retention.
- **Resolution:** inherit the base VLM's; raise tiling only if your task is OCR/document/chart-heavy and the base under-reads.
- **Eval:** your domain task + POPE (hallucination) + a text eval, before and after (Chapter 10).
- **Cost:** hours to a day; tens to low-hundreds of dollars.
- **When this is enough:** domain VLM, format/style adaptation, a bounded new skill on top of a capable base. The common case.

---

## 12.2 One node — build an 8B adapter VLM from parts

**Goal:** turn a text LLM into a general VLM (the LLaVA path).

- **Framework:** LLaVA-NeXT / LLaVA-OneVision lineage, or ms-swift.
- **Components:** SigLIP2 encoder (frozen) + MLP connector + your 8B LLM. Dynamic tiling for resolution.
- **Stage 1 (align):** connector-only, ~1e-3 LR, 1 epoch on large image-caption data (recaptioned for quality). Encoder + LLM frozen.
- **Stage 2 (joint):** train connector + LLM (LoRA or full), ~2e-5 LLM LR, on the *balanced mixture* — VQA, OCR, grounding, charts, interleaved, STEM + **10–30% text-only**. This is where the capability and cost are.
- **Stage 3 (instruction):** high-quality verified visual instructions; optionally mDPO against hallucination (Chapter 9).
- **Eval:** the discriminating axes for your target + POPE + paired text retention, every checkpoint (Chapter 10).
- **Cost:** low thousands of dollars, a few days on a node.
- **Guard:** differential LRs (connector high, LLM low), text-replay for forgetting, decode batches to verify template/masking/tiling.

---

## 12.3 Few nodes — a strong general or reasoning VLM

**Goal:** a competitive VLM across the discriminating axes, possibly with visual reasoning.

- **Framework:** NeMo/Megatron-multimodal or a scaled LLaVA-OneVision.
- **Components:** SigLIP2 (frozen early, **unfrozen late** at very low LR for OCR/grounding gains), MLP connector, a strong LLM (or reasoning LLM base).
- **Stages:** multi-stage (Chapter 4 extended) — align → joint tune → **high-resolution stage** (aggressive tiling / native res for OCR/charts) → instruction/preference. Schedule detail-heavy data into the high-res stages.
- **Data:** large curated mixture with heavy investment in the discriminating skills (OCR, charts, grounding, video if targeted); synthetic-data engine with filtering (Chapter 6).
- **Reasoning (optional):** cold-start visual-CoT SFT + RLVR on verifiable visual tasks (visual math, counting, charts) — Chapter 9.5; guard against reasoning over misperceived images.
- **Eval:** full paired harness including hallucination, contamination defense (image-level decontam + private eval), video/audio benchmarks if in scope.
- **Cost:** tens of thousands to low hundreds of thousands; days-to-weeks on a few nodes.

---

## 12.4 Frontier — a native omni model

**Goal:** unified text + image + audio + video in, text + speech out — the Qwen-Omni shape.

- **Approach:** native early fusion (Chapter 5), not glued adapters. Custom Megatron/NeMo-based stack.
- **Tokenization:** decide understanding-vs-generation → continuous (understanding) / discrete (any-to-any generation) / hybrid-decoupled (both). Thinker-talker for streaming speech out (Chapter 7).
- **Stage 1:** LLM locked; train vision + audio encoders on image-text and audio-text pairs for semantic understanding.
- **Stage 2:** unfreeze all; train on the wide multimodal mixture — **crucially mixing unimodal AND cross-modal data early** (the parity ingredient, Chapter 5.4) — with modality-aware normalization to prevent text from dominating.
- **Stage 3+:** specialization, then RL/preference for generation quality and hallucination.
- **Video:** frame sampling + token compression (Chapter 8); use audio track as supervision.
- **Eval:** all modalities' discriminating axes; parity check (no modality degraded); hallucination; streaming latency for speech.
- **Cost:** pretraining-scale × modalities — millions; full cluster, weeks.
- **Reality check:** this is a frontier-lab undertaking. Most "we need audio/video" needs are met by 12.1–12.3 (adapter understanding + optional TTS), not a native omni model.

---

## 12.5 The cross-cutting checklist

Before any multimodal training run:

- [ ] Adapter-vs-native decided (almost always adapter unless you need generation/frontier fusion).
- [ ] Encoder chosen (SigLIP2 default), resolution/tiling set to your token budget and task detail needs.
- [ ] Freeze schedule set (connector always, encoder frozen-by-default/unfrozen-late, LLM frozen-in-align).
- [ ] Data mixture audited for skill coverage (captions/VQA/OCR/grounding/charts/STEM/GUI) + 10–30% text-only.
- [ ] Synthetic data filtered/verified; benchmarks decontaminated at the *image* level; private eval reserved.
- [ ] Differential LRs configured (connector high, LLM low, encoder very low if unfrozen).
- [ ] Template/masking/tiling verified by decoding real batches.
- [ ] Paired eval harness ready: target axes + hallucination (POPE) + text retention + prior-multimodal.
- [ ] Data pipeline handles image/video decode without starving GPUs.

Next: failure modes — the specific ways multimodal training goes wrong.
