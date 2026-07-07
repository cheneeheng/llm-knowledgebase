# Chapter 10 — Evaluation

## 10.1 Conceptual picture

Multimodal evaluation in 2026 has a specific shape imposed by one fact: **the classic benchmarks are saturated.** MMMU-Pro — the flagship multimodal reasoning benchmark — is topped out, with every frontier model scoring ~80–83%. So evaluating a VLM by its MMMU score no longer discriminates between good and great models; the field "moved past the pure image-QA era." Effective evaluation now means measuring on the axes that *still* separate models — video, audio, long-document OCR, chart reasoning — and measuring the *failure* that accuracy benchmarks hide: hallucination. This chapter is the multimodal eval stack, built around what still discriminates.

## 10.2 The eval axes that still matter

| Axis | What it tests | Representative benchmarks | Status |
|---|---|---|---|
| **General multimodal reasoning** | Expert-level image+reasoning | MMMU, MMMU-Pro | **Saturated** (~80%+); a floor, not a differentiator |
| **Hallucination** | Describing absent objects | POPE, HallusionBench, CHAIR | The critical failure axis; measure always |
| **OCR / documents** | Reading text in images | DocVQA, OCRBench, ChartQA | **Discriminating**; Claude Opus leads |
| **Chart / infographic** | Reasoning over charts/tables | ChartQA, InfographicVQA | **Discriminating** frontier axis |
| **Video** | Temporal understanding | TempCompass, PerceptionTest, VideoMME | **Discriminating**; Gemini leads |
| **Audio** | ASR + audio reasoning | ASR sets, audio-QA | **Discriminating** for omni models |
| **Grounding** | Locating objects | RefCOCO, grounding benchmarks | Important for agentic/spatial |
| **General VQA** | Basic image Q&A | VQAv2, GQA | Largely saturated; sanity check |

The practical takeaway: **report the discriminating axes, and use the saturated ones only as floors** (a model that *fails* MMMU is broken, but passing it says little). Weight your eval effort toward the axes your product lives on and the axes where frontier models still differ.

## 10.3 Hallucination evaluation — the non-negotiable

Because hallucination is the characteristic multimodal failure (Chapter 9) and accuracy benchmarks hide it, you must measure it explicitly:

- **POPE (Polling-based Object Probing).** Asks yes/no about object *presence* ("is there a clock?"), including objects that aren't there. Directly measures whether the model asserts absent objects — the core hallucination. Cheap, standard, run it always.
- **HallusionBench, CHAIR, and successors** — richer hallucination probes (visual illusions, caption-level object hallucination rates). CHAIR measures the fraction of mentioned objects that are actually present.
- **The discipline:** track a hallucination metric on *every* checkpoint alongside accuracy, and treat a model that gained accuracy while gaining hallucination as suspect. A VLM that answers more questions correctly *and* invents more objects has not obviously improved. This is the paired-measurement discipline (like midtraining's forgetting budget) applied to the multimodal-specific failure.

## 10.4 Contamination — worse in multimodal

Multimodal contamination is *easier and more silent* than text contamination, for reasons worth internalizing:

- **Images are recaptioned and reused** — the same benchmark image appears across datasets with different captions, so a naive text-only decontamination misses it. You must decontaminate on the *image*, not just the question text.
- **Synthetic data regurgitates benchmarks** — a strong VLM generating training data will happily reproduce benchmark images/questions it memorized (Chapter 6.5).
- **Popular images are everywhere** — iconic benchmark images (COCO, etc.) permeate web data.

Defenses: decontaminate on image similarity (perceptual hashing / embedding overlap), not just text; filter synthetic data against benchmark images; and — the ultimate defense, straight from the text series — **hold a private eval the training pipeline has never seen.** When your private eval and public benchmarks disagree, trust the private one.

## 10.5 LLM/VLM-as-judge and its multimodal biases

For open-ended multimodal outputs (image description quality, visual reasoning), automated judging uses a strong VLM as judge — inheriting all the text LLM-judge biases (post-training series Chapter 14: position, verbosity, self-preference) *plus* multimodal-specific ones:

- **The judge hallucinates too** — a VLM judge evaluating whether a description matches an image can itself misperceive the image, producing wrong judgments. Judge reliability is bounded by judge perception.
- **Prompt sensitivity** — multimodal models are notably sensitive to prompt phrasing (Promptception-style findings); judge and judged both wobble with prompt changes.
- **Mitigations:** ground the judge with the reference answer, use it for *relative* comparison not absolute scoring, spot-check judge decisions against human labels, and prefer *verifiable* evaluation (exact-match on VQA, IoU on grounding, value-match on charts) over judged evaluation wherever the task allows.

## 10.6 The paired harness (assembled)

Every multimodal training run should report, on every checkpoint, against a self-measured baseline:

1. **Target capability** on the discriminating axes for your product (OCR, charts, video, audio, grounding).
2. **Hallucination** (POPE minimum) — the failure axis.
3. **Text retention** — MMLU/instruction-following, because the new eye can dim the old tongue (Chapters 1.4, 4.6). Non-negotiable.
4. **Prior-multimodal retention** — if you're adding a modality/capability, check you didn't break earlier ones.
5. **A fast proxy** and a **private held-out eval** for contamination defense.

Run 1–4 per checkpoint; select on the paired trade (target up, hallucination down, retention held), not target alone — a checkpoint that's +5 on ChartQA, +3 on hallucination, -4 on MMLU is usually worse than a more balanced one.

## 10.7 Decisions

1. **Report the discriminating axes** (OCR/documents, charts, video, audio, grounding, hallucination); use saturated benchmarks (MMMU, VQAv2) as floors, not differentiators.
2. **Measure hallucination explicitly on every checkpoint** (POPE minimum) — accuracy hides it; treat accuracy-up-with-hallucination-up as suspect.
3. **Decontaminate on images (perceptual/embedding), not just text**, filter synthetic against benchmarks, and hold a private eval — multimodal contamination is silent and easy.
4. **Use VLM-as-judge cautiously** (it hallucinates and is prompt-sensitive); prefer verifiable metrics (exact-match, IoU, value-match) where the task allows.
5. **Run the paired harness** (target + hallucination + text retention + prior-multimodal + private eval) and select on the balanced trade, not the target metric alone.

Next: infrastructure and frameworks — what changes in the training stack when images and video enter.
