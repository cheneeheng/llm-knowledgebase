# Chapter 13 — Failure Modes

Multimodal training fails in a register between the text stages: not the loud divergence of pretraining, but silent quality problems — a model that scores well and hallucinates, or that gained an eye and lost its tongue. Each entry is symptom → cause → fix. Most trace to three roots: **weak alignment/grounding**, **modality imbalance**, and **the-new-eye-dims-the-old-tongue forgetting**.

---

## 13.1 Object hallucination

**Symptom:** the model confidently describes objects, text, or chart values not present in the image.
**Cause:** language-prior dominance (the LLM is far better trained than the vision pathway, so it guesses plausible-sounding answers), SFT data that asserts unsupported visual claims, or insufficient resolution so the model literally can't see the detail (Chapters 9.3, 3).
**Fix:** ground SFT data strictly (drop answers not visually supported), add negative/counterfactual examples, train mDPO/RLVR against hallucination (Chapter 9.4), raise resolution for detail tasks, and **measure POPE every checkpoint** — this failure is invisible in accuracy. The characteristic multimodal failure; prevention is a whole discipline.

---

## 13.2 Catastrophic text forgetting

**Symptom:** the VLM answers image questions well but its pure-text ability (MMLU, coding, instruction-following) dropped noticeably.
**Cause:** joint training pulled the LLM off its text distribution; no text-replay, LLM LR too high, or full-FT without safeguards (Chapters 1.4, 4.6).
**Fix:** include 10–30% text-only data in the joint stage; LoRA the LLM (frozen base can't forget); lower the LLM LR; **run text evals alongside vision evals on every checkpoint** and hold the forgetting budget. The new eye must not dim the old tongue.

---

## 13.3 Connector collapse / failed alignment

**Symptom:** after training, the model ignores the image — answers from text alone, or produces generic captions unrelated to the specific image.
**Cause:** the alignment stage was skipped, too short, or the connector under-trained; or Stage 2 started before the connector could map vision into the LLM's space. The bridge never got built.
**Fix:** run a proper Stage 1 alignment (connector-only, enough caption data, encoder+LLM frozen) *before* joint tuning; give the connector a high enough LR to move; verify on captioning before proceeding to instructions. Alignment before ability (Chapter 4.1) — non-negotiable.

---

## 13.4 Modality imbalance (native/omni)

**Symptom:** in a native or omni model, one modality is strong and others are weak, or adding a modality degraded the others.
**Cause:** the data-rich modality (usually text) dominated training dynamics, its gradients swamping the scarce modalities; or pure-cross-modal training without unimodal data (Chapter 5.4).
**Fix:** modality-aware normalization to balance dynamics; **mix unimodal AND cross-modal data in early training** (the Qwen-Omni parity ingredient); balance the data mixture across modalities; check per-modality parity on every checkpoint. Don't let the loud modality drown the quiet ones.

---

## 13.5 Resolution starvation

**Symptom:** the model fails at OCR, small text, dense charts, or small-object grounding despite lots of relevant training data.
**Cause:** resolution too low / tiling too capped — the encoder never received enough pixels to see the detail, so no amount of data helps (Chapter 3.3).
**Fix:** raise resolution via dynamic tiling or native-resolution encoding; spend the visual-token budget on detail-critical tasks; schedule OCR/chart data into high-resolution stages. You can't train a skill the encoder can't perceive.

---

## 13.6 Visual-token budget blowout

**Symptom:** OOM or extreme slowdown during training/inference; sequences far longer than expected; tiny effective batch.
**Cause:** aggressive tiling or high frame counts inflated visual tokens (one high-res image → thousands of tokens; video → tens of thousands), blowing the sequence length (Chapters 2.5, 8.2).
**Fix:** cap tiles/frames to your budget, use token compression (per-frame pooling, temporal merging), bucket by length, and budget training *by total tokens including visual inflation*, not image count. Video especially needs sampling + compression to be tractable.

---

## 13.7 Template/masking/tiling mismatch

**Symptom:** the fine-tune performs far worse than expected, or breaks in serving despite good training metrics.
**Cause:** the image-placeholder token, chat template, loss masking, or tiling scheme used in training doesn't byte-for-byte match serving/eval (Chapter 9.2) — the multimodal version of the text series' #1 fine-tuning bug.
**Fix:** freeze the template (including how images are referenced and tiled) early; make training match serving exactly; **decode real collated batches and look at them** (template, masks, tiling boundaries, that no image is split) before any real run. Five minutes; catches the majority of silent failures.

---

## 13.8 Synthetic-data hallucination propagation

**Symptom:** the model hallucinates in specific, consistent ways that mirror a quirk of the VLM that generated its training data.
**Cause:** synthetic captions/VQA generated by a VLM inherited that generator's hallucinations, and SFT cloned them (Chapters 6.3, 9.2).
**Fix:** filter synthetic data (image-text matching, consistency checks), verify a sample by hand, use multiple generators or human verification for critical data, and never let one generator's quirks propagate unchecked. Synthetic data is a force multiplier for both quality and errors.

---

## 13.9 Contamination-inflated scores

**Symptom:** benchmark scores look great; real-world use and private evals don't match.
**Cause:** benchmark images (recaptioned, or iconic and web-pervasive) leaked into training; text-only decontamination missed them (Chapter 10.4).
**Fix:** decontaminate at the *image* level (perceptual hash / embedding), filter synthetic against benchmark images, and hold a private eval the pipeline never saw. Trust the private eval when it disagrees with public benchmarks.

---

## 13.10 Reasoning over a misperceived image

**Symptom:** a reasoning VLM produces long, confident chains of thought that are internally coherent but wrong because they're based on a misreading of the image.
**Cause:** strong reasoning over weak perception — the model hallucinated the image content, then reasoned flawlessly about the hallucination; reasoning VLMs are *less* robust to visual distraction (Chapter 9.5).
**Fix:** strengthen grounding and hallucination control *before and during* reasoning training; verify RLVR rewards against the actual image ground truth, not just answer plausibility; test robustness to visual distractors. Reasoning amplifies perception errors, so grounding matters more, not less.

---

## 13.11 The meta-lesson

Multimodal failures cluster around three habits to build: **align and ground before you specialize** (proper Stage 1, resolution, hallucination control), **balance the modalities and keep the text alive** (replay, modality-aware training, paired evals), and **verify artifacts and measure the silent failure** (decode batches, POPE every checkpoint, image-level decontamination, private eval). None of these is exotic. Multimodal training is the text discipline plus "watch what the new modality does to hallucination and to the modalities you already had" — watch those two things, and it's mechanical.

Next: essential reading and references.
