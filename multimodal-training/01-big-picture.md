# Chapter 1 — The Big Picture

## 1.1 What multimodal training is

Multimodal training turns a model that only reads and writes text into one that also *perceives* other modalities — images, documents, audio, video — and, for the most advanced systems, *generates* them. In 2026 this is the default expectation of "an LLM," not an add-on. The mechanics are an extension of everything the text series taught: you still train transformers with next-token prediction and post-train with SFT and RL. What's new is the front of the pipeline — getting a non-text signal (pixels, audio waveforms) into a form the transformer can consume — and the data and evaluation disciplines that come with it.

The key reframing: a transformer eats a sequence of vectors (token embeddings). Text becomes vectors via a tokenizer + embedding table. **Multimodal training is, at its heart, the problem of turning images/audio/video into vectors the same transformer can attend over, and teaching the model to relate those vectors to language.** Everything else — which encoder, which connector, what to freeze, what data — is engineering around that one idea.

## 1.2 The two paradigms

The spine of this series is a single architectural choice, and it's worth understanding before any detail.

**The adapter (compositional) VLM.** Take a pretrained vision encoder (it already turns images into useful vectors) and a pretrained text LLM (it already reasons in language), and connect them with a small learned **connector** that maps the encoder's image vectors into the LLM's embedding space. Train mostly the connector (and optionally the LLM), and you have a model that "sees." This is cheap (you reuse two pretrained models), modular (swap encoders, swap LLMs), and how the overwhelming majority of open VLMs — LLaVA, Qwen-VL, InternVL, Molmo — are built. It's where you should start.

**The native (early-fusion) multimodal model.** Instead of bolting modalities onto a text-centric core, train a single model over all modalities from the beginning, so vision, audio, and text have "equal architectural standing" in one shared representation space. This enables tighter cross-modal reasoning and *any-to-any generation* (text in, image or speech out) that adapter models struggle with, at the cost of far more compute and less modularity. The frontier (Gemini, GPT, Qwen-Omni, Emu, Janus) is moving this way, and Chapter 5 covers it — but it's not where most teams begin.

| | Adapter / compositional | Native / early-fusion |
|---|---|---|
| Build | Reuse pretrained encoder + LLM | Train jointly from (near) scratch |
| Compute | Low (train a connector) | High (full joint optimization) |
| Modularity | High (swap parts) | Low (integrated) |
| Cross-modal reasoning | Good, LLM-centric | Superior, genuine fusion |
| Generation | Usually text-out only | Any-to-any (text/image/audio out) |
| Who uses it | Most teams, most open VLMs | Frontier labs, omni models |
| This series | Chapters 2–4 | Chapter 5 |

## 1.3 The moved frontier

Where the field sits as you enter it, because it shapes what's worth training:

- **Image-QA is saturated.** MMMU-Pro — the flagship multimodal reasoning benchmark — is topped out: every frontier model (GPT-5.5, Gemini 3, Claude Opus 4.7, Qwen3.5-Omni) scores ~80–83% as of early 2026. Basic "look at this image and answer" is solved. The differentiating axes moved to **video, audio, long-document OCR, and chart/infographic reasoning** (Chapter 10). Train for where the frontier *is*, not where it was.
- **Native and omni models are ascendant.** Qwen3-Omni and peers demonstrate joint training across text/image/audio/video with *no per-modality degradation* and real-time speech output — the native paradigm delivering on its promise (Chapter 7). The adapter approach still dominates by count, but native is where the capability ceiling is rising.
- **Data curation beats architecture.** The most repeated 2026 finding: with the *same* architecture, better data curation moves VLM quality more than architectural changes ("a prescription for better VLMs through data curation alone"). This is the through-line — the architecture is nearly a solved recipe; the leverage is in data (Chapter 6).
- **Vision encoders consolidated.** SigLIP2 and native-resolution ViTs (with dynamic tiling) are the default eyes; the encoder question is largely settled, and the open decisions are resolution/token-budget and frozen-vs-unfrozen (Chapter 3).

## 1.4 Core mental models

Five ideas recur across the series.

- **Modalities become tokens.** Whatever the modality, it ends up as a sequence of vectors in the transformer's stream. Images become a few hundred to a few thousand "visual tokens"; audio becomes audio tokens; video becomes a *lot* of tokens. Reasoning about the **visual-token budget** — how many tokens an image costs and how that trades resolution against sequence length and compute — is as central to multimodal work as the KV-cache budget was to serving.

- **Alignment before ability.** The connector starts random; it speaks neither the encoder's nor the LLM's language. The first training stage does nothing but *align* — teach the connector to map image vectors into the LLM's space (usually via captioning). Only after alignment do you teach *ability* (instruction-following, reasoning). Skipping or rushing alignment is a top failure mode.

- **The new eye can dim the old tongue.** Training on image-text data pulls the LLM toward the multimodal distribution and, if you're careless, degrades its pure-text capability — catastrophic forgetting, exactly as in midtraining. The defense is the same: keep text data in the mixture, freeze judiciously, and *keep running the text evals* (Chapter 4, 13).

- **Hallucination is the characteristic failure.** VLMs confidently describe objects that aren't in the image, misread text, and invent chart values. Hallucination is to multimodal what reward-hacking is to RL: the silent failure you must measure explicitly (POPE and friends, Chapter 10), because accuracy benchmarks hide it.

- **Curation over cleverness.** Say it again because it's the through-line: the leverage is in the data mixture and its staging, not in a novel connector. Reach for better data first.

## 1.5 Decisions this series will force

1. **Adapter or native?** (Ch 2, 5 — almost always start adapter.)
2. **Which vision encoder, at what resolution, frozen or not?** (Ch 3)
3. **Which connector — MLP, Q-Former, Perceiver?** (Ch 2)
4. **What multi-stage training recipe — freeze what, when?** (Ch 4)
5. **What data mixture — captioning, interleaved, OCR, grounding, charts, text-replay?** (Ch 6)
6. **Which modalities beyond vision — audio, video, omni?** (Ch 7, 8)
7. **How to post-train and reduce hallucination?** (Ch 9)
8. **How to evaluate on the axes that still discriminate?** (Ch 10)

Start with Chapter 2: the adapter VLM architecture is where 90% of multimodal work happens and the frame the rest of the series builds on.
