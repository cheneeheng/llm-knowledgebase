# Chapter 6 — Multimodal Data

## 6.1 Conceptual picture

If there is one lever in this series, it is data. The most repeated 2026 finding is that with a *fixed* architecture, better data curation improves VLM quality more than architectural changes — "a prescription for better VLMs through data curation alone." The architecture (Chapters 2–4) is a near-solved recipe; the assembly of the data mixture, its quality, and its staging is where VLMs are won or lost. This chapter is the taxonomy of multimodal data, how to source and synthesize it, and how to mix it.

The mental model: a VLM must learn many distinct skills — describe an image, answer questions about it, read text in it, locate objects, interpret charts, reason step-by-step over it — and *each skill needs its own data type*. A mixture heavy on captions produces a model that describes but can't read; one heavy on VQA produces one that answers but can't ground. Coverage across data types, balanced so none dominates, is the craft.

## 6.2 The data types (and what each teaches)

| Data type | What it is | Skill it teaches |
|---|---|---|
| **Image-caption pairs** | Image + a description | Basic vision-language alignment (Stage 1); the foundation |
| **Interleaved image-text** | Documents/web pages with images inline in text | In-context multimodal learning, multi-image reasoning, few-shot |
| **Visual instruction (VQA)** | Image + question + answer | Instruction-following over images; the core capability |
| **OCR data** | Text-heavy images + transcriptions | Reading text in images — documents, signs, screenshots |
| **Grounding data** | Image + text + bounding boxes/locations | Locating and referring to objects; spatial understanding |
| **Chart/table/document** | Charts, infographics, tables + Q&A | Chart reasoning, document understanding (a *discriminating* axis, Ch 1.3) |
| **STEM / diagram** | Scientific figures, math, diagrams | Expert reasoning (MMMU-style); multimodal math |
| **GUI / screenshot** | UI screenshots + actions/descriptions | Agentic/computer-use grounding |
| **Text-only** | Pure language data | Preventing text forgetting (Chapter 4.6) |

A strong VLM's mixture spans essentially all of these. The Qwen-VL-lineage corpora are explicitly "image caption + interleaved image-text + OCR + grounding + STEM + GUI data, each carefully curated." **Missing a type means missing a capability** — no OCR data, no document reading; no grounding data, no spatial ability. Audit your mixture for coverage of the skills you need.

## 6.3 Captioning and the synthetic-data engine

Raw web image-text pairs (alt-text) are noisy and short — "dog" for a rich photo. The 2026 workhorse is **synthetic recaptioning**: use a strong VLM to generate long, detailed, accurate captions for images, replacing or augmenting the noisy alt-text. "Synthetic data at scale improves multimodal pre-training; synthetic captions are longer and more descriptive." The recaptioning insight (from image generation, but general): a caption that fully describes the image teaches far more alignment than a terse alt-text.

This generalizes into a synthetic-data engine, the multimodal analogue of the text series' synthetic data:
- **Recaption** images with a strong VLM for rich alignment data.
- **Generate VQA** by prompting a VLM to ask-and-answer questions about images.
- **Synthesize OCR/grounding** by rendering text/boxes with known ground truth (GrIT-style synthetic grounding, rendered documents with known transcriptions).
- **Convert** existing datasets (detection, segmentation) into instruction format.

The caution, straight from the text series: **synthetic data inherits the generator's errors and hallucinations.** A VLM that hallucinates objects will write captions asserting those objects, training your model to hallucinate them. Filter and verify synthetic data (quality classifiers, consistency checks), and never let a single generator's quirks propagate unchecked (Chapter 13).

## 6.4 OCR, grounding, and the discriminating skills

Because image-QA is saturated and the frontier moved (Chapter 1.3), the *discriminating* data types deserve extra attention:

- **OCR / documents** — transcribe document images (public OCR to bootstrap, then verified) at scale; this is what lets a VLM read a contract or a receipt. Needs high resolution (Chapter 3.3) *and* the data. Document-heavy products live or die here.
- **Grounding** — synthetic caption datasets with location labels (bounding boxes) for major visual elements teach the model to point, not just describe. Essential for agentic/computer-use and spatial reasoning.
- **Charts/infographics** — a current frontier axis; chart-QA and chart-reasoning data are scarce and valuable. Synthesize by rendering charts from known data (you have ground truth) + generated questions.

If your product is on one of these axes, over-weight its data relative to generic captions/VQA — that's where quality and differentiation now live.

## 6.5 Curation and quality

Curation is where the "data beats architecture" leverage is realized:

- **Caption concreteness / quality filtering** — score captions for how well they actually describe their image (concreteness metrics like ICC) and drop vague or mismatched pairs. Garbage pairs teach misalignment.
- **Image-text matching** — filter pairs where the text doesn't match the image (a CLIP/SigLIP score threshold is a cheap first pass).
- **Deduplication** — as in text pretraining; duplicate images/captions waste compute and risk memorization.
- **Balance** — ensure no data type or domain dominates; a mixture 90% captions learns to caption and little else. Deliberately weight toward coverage of your target skills.
- **Decontamination** — remove images/questions overlapping your eval benchmarks (MMMU, POPE, etc.), including *near*-duplicates (the same image recaptioned). Multimodal contamination is easy and silent (Chapter 10).

## 6.6 The mixture and staging

Tie it to the recipe (Chapter 4):

- **Stage 1 (alignment):** mostly image-caption pairs — large-scale, the simple alignment task. Recaptioned data shines here.
- **Stage 2 (joint tuning):** the *full balanced mixture* — VQA, OCR, grounding, charts, interleaved, STEM, GUI, *plus 10–30% text-only* for forgetting. This is where coverage and balance matter most.
- **Stage 3 (instruction/preference):** highest-quality, human-or-strong-model-verified instruction and preference data; smaller, cleaner, hallucination-reducing (Chapter 9).

Resolution interacts with data: OCR/chart data needs the high-resolution/tiling stages (Chapter 3.3), so schedule detail-heavy data into the high-res stages.

## 6.7 Decisions

1. **Treat data curation as the primary quality lever** — better data beats a better connector; invest here first.
2. **Cover every skill you need with its data type** — captions, interleaved, VQA, OCR, grounding, charts, STEM, GUI; a missing type is a missing capability.
3. **Run a synthetic-data engine** (recaptioning, VQA generation, rendered OCR/grounding) — but filter and verify it; synthetic data propagates the generator's hallucinations.
4. **Over-weight the discriminating skills** (OCR/documents, grounding, charts) if your product is on those axes — that's where the frontier and differentiation now live.
5. **Curate for quality and balance** (concreteness filtering, image-text matching, dedup, no-type-dominates) and **decontaminate against benchmarks including near-duplicates**; stage detail-heavy data into high-resolution stages, keep 10–30% text-only for forgetting.

Next: audio, speech, and omni models — extending everything so far beyond vision.
