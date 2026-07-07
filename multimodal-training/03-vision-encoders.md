# Chapter 3 — Vision Encoders

## 3.1 Conceptual picture

The vision encoder is your VLM's eye — it converts pixels into the semantic vectors the connector maps into the LLM. Its quality bounds everything downstream: no connector or LLM can recover detail the encoder threw away. The good news is that this component is largely *solved* in 2026 — a handful of strong pretrained encoders (SigLIP2 chief among them) are the near-universal choice, so you rarely train one from scratch. The decisions that remain and that materially change your VLM are: which encoder, at what **resolution**, and **frozen or unfrozen**. This chapter is those decisions.

## 3.2 What a vision encoder is

Almost universally a **Vision Transformer (ViT)**: split an image into fixed patches (e.g. 14×14 pixels), linearly embed each patch, add position embeddings, run transformer layers, output a grid of patch embeddings. A 384×384 image at patch-14 gives a ~27×27 grid ≈ 729 patch vectors — the "visual tokens" of Chapter 2.

The crucial part is *how it was pretrained*, because that determines what the vectors mean:

- **CLIP** — contrastive image-text pretraining (match images to captions across a batch). The original workhorse; ViT-L/14 CLIP powered early LLaVA. Learns broadly useful, language-aligned features.
- **SigLIP / SigLIP2 — the 2026 default.** SigLIP replaced CLIP's softmax contrastive loss with a **sigmoid** loss (pairwise, no giant batch needed), training more efficiently and to higher quality. SigLIP2 adds captioning loss, self-distillation, and masked prediction on top, plus multilingual data and native-resolution support — making it the strongest general-purpose encoder for VLMs. Default to SigLIP2 unless you have a reason.
- **DINOv2 (self-supervised) and mixtures.** DINOv2 (no text supervision) captures fine spatial/geometric detail CLIP-style encoders miss; some VLMs *mix* encoders (SigLIP for semantics + DINOv2 for detail) for grounding-heavy work. Adds complexity; worth it for spatial/grounding-critical products.

## 3.3 Resolution and the token budget

Resolution is the highest-leverage encoder decision because it directly sets both what the model can *see* and what it *costs* (the Chapter 2.5 budget). The problem: ViTs are pretrained at a fixed low resolution (224 or 384), but real tasks — reading a document, a dense chart, a small sign — need far more pixels than 384×384 provides. Three approaches:

- **Fixed low resolution** (early VLMs) — resize everything to 336/384. Cheap, but blind to detail; hopeless for OCR/documents.
- **Dynamic tiling / AnyRes (the dominant 2026 approach).** Split a high-resolution image into an aspect-ratio-aware grid of native-resolution tiles (e.g. 384×384 tiles, up to ~12, in the InternVL style) plus a downsized thumbnail for global context. Each tile is encoded separately; the visual-token count scales with image size. This is how VLMs read documents and dense charts — you give the encoder the pixels by tiling. Cost: token budget explodes (12 tiles ≈ 8,700+ tokens for one image), so cap tiles per your task and budget.
- **Native-resolution encoders** (NaViT-style, SigLIP2's native mode) — encoders trained to handle variable resolutions/aspect ratios directly, avoiding the artifacts of forcing images into square tiles. Increasingly the clean solution; SigLIP2 supports it.

The practical rule: **spend resolution (tokens) on the tasks that need it.** A document/OCR/chart product uses aggressive tiling or native high-res; a general chat VLM caps tiles to keep the budget sane. This is the single decision that most separates a VLM that can read a form from one that can't.

## 3.4 Frozen vs unfrozen encoder

Whether to train the vision encoder during VLM training, one of the most-debated staging choices:

- **Frozen encoder (common, safe).** Keep the pretrained encoder fixed; train only the connector (and LLM). Advantages: cheaper, stable, preserves the encoder's broad pretrained knowledge, can't be damaged by a small/narrow VLM dataset. The finding: "freezing SigLIP2 and fine-tuning the language backbone achieves SOTA in VQA and multimodal benchmarks at comparable scales." **Default to frozen**, especially with strong encoders like SigLIP2 and limited data.
- **Unfrozen / fine-tuned encoder.** Train the encoder (usually at a low LR, in a later stage) to adapt its features to your tasks. Advantages: can meaningfully improve OCR, grounding, and domain-specific perception (medical images, documents) where the pretrained encoder is weak. Risks: forgetting the encoder's general features, instability, needs more data and care. Unfreeze late, at low LR, when you have the data and a task the frozen encoder demonstrably underperforms on (fine detail, specialized domains).

The 2026 nuance: many strong VLMs (Qwen-VL line, InternVL) *do* train the encoder in later stages, because at scale it helps — but they do it carefully, staged, with lots of data. For a first VLM or a small dataset, **frozen is the correct, robust default**; unfreeze only when you've hit the frozen encoder's ceiling on a task you care about and have the data to train it safely.

## 3.5 Choosing an encoder — decision guide

| Situation | Encoder choice |
|---|---|
| General-purpose VLM, default | **SigLIP2** (frozen), dynamic tiling |
| OCR / documents / dense charts | SigLIP2 native-res or aggressive tiling; consider unfreezing late |
| Grounding / spatial / detection | SigLIP2 + DINOv2 mix, or unfrozen encoder |
| Domain-specific perception (medical, satellite) | Domain-pretrained or fine-tuned (unfrozen) encoder + domain data |
| Tight token/compute budget | Lower resolution / capped tiles / resampler connector (Ch 2) |
| Multilingual | SigLIP2 (multilingual pretraining) |

## 3.6 Decisions

1. **Use SigLIP2** as the default encoder — it's the strongest general-purpose VLM eye in 2026; CLIP only for legacy compatibility.
2. **Use dynamic tiling / native resolution** to handle high-res images; **cap tiles to your token budget** and spend resolution on detail-critical tasks (OCR/charts/grounding).
3. **Keep the encoder frozen by default** — strong encoders + frozen is SOTA at comparable scale and robust with limited data.
4. **Unfreeze the encoder late, at low LR, with ample data**, only when the frozen encoder demonstrably caps a task you care about (fine detail, specialized domains).
5. **Mix encoders (SigLIP2 + DINOv2)** only for grounding/spatial-critical work where the added complexity pays off.

Next: the training recipe that ties encoder, connector, and LLM together — the multi-stage schedule of what to freeze and train when.
