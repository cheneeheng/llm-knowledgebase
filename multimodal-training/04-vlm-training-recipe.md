# Chapter 4 — The VLM Training Recipe

## 4.1 Conceptual picture

Building an adapter VLM is a **staged** process, and the stages are defined by *what you freeze and what you train* at each step. The logic is alignment-before-ability (Chapter 1.4): the connector begins random and meaningless, so you first train *only* it to align vision with language, then progressively unfreeze more of the model to build capability, using a data curriculum that goes from simple captions to rich instructions. Get the staging wrong — train everything at once from random, or skip alignment — and you either destabilize the LLM or never teach the connector to speak. This chapter is the recipe.

## 4.2 The canonical multi-stage recipe

The LLaVA-lineage recipe, refined through 2026, in the order you run it:

**Stage 1 — Connector alignment (pretraining the projector).**
- *Frozen:* vision encoder, LLM. *Trained:* connector only.
- *Data:* large-scale image-caption pairs (hundreds of thousands to millions). Simple task: given an image, produce its caption.
- *Goal:* teach the connector to map visual features into the LLM's embedding space — nothing more. The LLM and encoder are untouched, so nothing can be damaged; you're just building the bridge.
- *Why alone:* if you trained the LLM here on captions, you'd degrade its text ability on trivial data. Keep it frozen until the bridge exists.

**Stage 2 — Joint tuning (the main event).**
- *Frozen:* often the encoder (or unfrozen late, Chapter 3). *Trained:* connector + LLM (full or LoRA).
- *Data:* a rich mixture — visual instruction data, VQA, OCR, grounding, charts, interleaved image-text, *plus text-only data to prevent forgetting* (Chapter 6).
- *Goal:* build actual multimodal capability — instruction-following over images, reasoning, reading. This is where the VLM becomes useful and where most of the compute and data go.

**Stage 3 — Instruction / preference tuning (post-training).**
- *Trained:* connector + LLM. *Data:* high-quality multimodal instructions, then optionally multimodal preference/RL (Chapter 9).
- *Goal:* polish behavior, reduce hallucination, align to your use case. This is the multimodal analogue of the post-training series' SFT + preference stages.

Larger models add stages (a high-resolution stage, an encoder-unfreezing stage, a long-context/video stage) — Qwen and InternVL run 3–4 pretraining stages — but the skeleton is always **align the connector → jointly build capability → post-train**, with resolution and unfreezing increasing across stages.

## 4.3 What to freeze, summarized

| Stage | Vision encoder | Connector | LLM |
|---|---|---|---|
| 1 — Alignment | ❄️ frozen | 🔥 train | ❄️ frozen |
| 2 — Joint tuning | ❄️ frozen (or 🔥 late) | 🔥 train | 🔥 train (full or LoRA) |
| 3 — Instruction/preference | ❄️/🔥 | 🔥 train | 🔥 train |

The invariant: **the connector is always trained; the encoder is frozen by default and unfrozen latest; the LLM is frozen only during alignment.** Everything else is scale-dependent elaboration.

## 4.4 Hyperparameters

Defaults that work for an 8B-class adapter VLM:

| Knob | Stage 1 (align) | Stage 2 (joint) | Notes |
|---|---|---|---|
| LR | ~1e-3 (connector only) | ~2e-5 (LLM), ~1e-3 (connector) | Connector tolerates high LR; LLM wants low, as in SFT |
| Epochs | 1 | 1–3 | Overfits fast on instruction data; checkpoint & eval each epoch |
| Encoder LR (if unfrozen) | — | ~2e-6 (very low) | Encoder is delicate; low LR only |
| Batch | large (simple task) | moderate | Visual tokens inflate sequence length — watch memory |
| Seq length | cover captions | cover instructions + visual tokens | Visual tokens can dominate; budget accordingly (Ch 2.5) |
| Optimizer | AdamW | AdamW | Nothing exotic |

Use **differential learning rates** — the connector (random, needs to move fast) at a high LR, the LLM (pretrained, delicate) at an SFT-level low LR, the encoder (most delicate) at a very low LR if unfrozen. A single global LR is a common mistake that either under-trains the connector or damages the LLM.

## 4.5 Full fine-tune vs LoRA

Same logic as the text series (post-training Chapter 4), applied to the LLM backbone:

- **LoRA the LLM** (with a fully-trained connector) — the efficient default for most teams. Trains the connector fully (it's small and must move a lot) and LoRA-adapts the LLM (preserving text ability, cheap). Fits VLM training on modest hardware; forgets less text. Apply LoRA to all LLM linear layers.
- **Full fine-tune the LLM** — for flagship VLMs, large diverse datasets, or when LoRA's capacity binds. More text forgetting risk (mitigate with text-replay, 4.6), more compute.

Decision: **LoRA for a first VLM, a domain VLM, or limited data; full FT for flagship general VLMs with large mixtures.** The connector is always fully trained regardless.

## 4.6 Preventing text forgetting

The new-eye-dims-old-tongue problem (Chapter 1.4). Training on image-text data pulls the LLM off its text distribution. Defenses, all from the midtraining playbook:

- **Text-replay in the mixture** — include a meaningful fraction (10–30%) of *text-only* instruction/pretraining data in Stage 2, so the LLM keeps practicing pure language. The single most effective defense.
- **LoRA the LLM** — frozen base weights can't forget (Chapter 4.5).
- **Freeze the LLM longer / lower its LR** — less movement, less forgetting.
- **Keep running the text evals** — MMLU, instruction-following, coding — *alongside* the vision evals, on every checkpoint. A VLM that gained 10 points of VQA and lost 8 of MMLU is often a bad trade. This paired measurement is as non-negotiable here as in midtraining (Chapter 10, 13).

## 4.7 Decisions

1. **Follow the staged recipe:** align the connector (encoder+LLM frozen) → joint-tune connector+LLM on a rich mixture → post-train. Never train everything from random at once.
2. **Freeze the encoder by default, the LLM only during alignment, the connector never** — unfreeze the encoder latest and at very low LR.
3. **Use differential learning rates** — connector high, LLM low (SFT-level), encoder very low.
4. **LoRA the LLM for a first/domain VLM; full FT for flagship**; always fully train the connector.
5. **Prevent text forgetting** with 10–30% text-replay in the joint stage and by running text evals alongside vision evals on every checkpoint.

Next: native multimodal models — what changes when you abandon the frozen-encoder composition and train all modalities jointly from the start.
