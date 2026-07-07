# Chapter 9 — Multimodal Post-Training

## 9.1 Conceptual picture

Everything the post-training series taught — SFT, preference optimization, RLHF, RLVR, reasoning — applies to multimodal models, with the image (or audio/video) added to the context. A multimodal SFT example is `(image, prompt) → response` with loss on the response, exactly like text SFT with an image prepended. A multimodal preference pair is two responses to the same image-prompt, one preferred. The algorithms are unchanged. What's *new* and demands its own treatment is the characteristic multimodal failure that post-training must fight: **hallucination** — the model confidently describing things not in the image — and the emergence of **multimodal reasoning** (thinking step-by-step over visual input). This chapter is the multimodal-specific layer over the post-training series.

## 9.2 Visual instruction tuning (multimodal SFT)

The multimodal SFT stage (Stage 3 of Chapter 4's recipe), which turns a jointly-tuned VLM into a helpful assistant:

- **Mechanics identical to text SFT** (post-training series Chapter 4): chat template (now with an image placeholder token marking where visual tokens go), loss masked to the assistant response, packing (careful with images — you can't split an image across a pack boundary). The template must byte-for-byte match serving, including how images are referenced — a top multimodal bug.
- **Data:** high-quality visual instructions (Chapter 6) — diverse tasks, verified answers, spanning your target skills. LLaVA's original insight was that GPT-4-generated visual instructions bootstrap a capable assistant cheaply; the 2026 version uses strong VLMs to generate large, verified instruction sets.
- **The quality-over-quantity rule holds:** a smaller set of excellent, hallucination-free instructions beats a large noisy one, because (as in text) SFT clones the data's behavior *including its errors* — instruction data that asserts unsupported visual claims trains visual hallucination.

## 9.3 The hallucination problem

The defining multimodal failure. VLMs hallucinate in ways text models don't: describing objects absent from the image, misreading text, inventing chart values, over-relying on *language priors* (answering what's *usually* true rather than what the image *shows* — "the grass is green" for a photo of brown grass). Causes:

- **Language prior dominance** — the LLM is vastly better trained (trillions of text tokens) than the vision pathway, so under uncertainty it falls back on textual plausibility over visual evidence.
- **SFT teaching assertion** — instruction data that describes things not clearly in the image trains confident unsupported claims (the text series' "SFT trains hallucination," now visual).
- **Weak visual grounding** — insufficient resolution (Chapter 3) or connector capacity means the model literally can't see the detail, so it guesses.

Hallucination is *the* thing multimodal post-training must reduce, and it's measured explicitly (POPE and friends, Chapter 10) because accuracy benchmarks hide it.

## 9.4 Reducing hallucination

The toolkit, in rough order of adoption:

- **Better, hallucination-free SFT data** — the first defense: ensure instruction answers are strictly grounded in the image; filter data where the answer isn't visually supported. Negative/counterfactual data ("is there a cat? No.") teaches the model to say no.
- **Preference optimization against hallucination (multimodal DPO/RLHF).** Construct preference pairs where the *grounded* response is preferred over the *hallucinated* one, and train with DPO or RLHF. This directly penalizes making things up. Specialized methods (mDPO and successors) address a subtlety: standard DPO on multimodal data can ignore the image and learn from text alone, so multimodal preference methods add image-conditioning safeguards.
- **RLVR for verifiable visual tasks** — for tasks with checkable answers (visual math, counting, chart-value extraction, grounded detection), reward correctness against ground truth (post-training series Chapter 8). The verifier checks the answer matches the image's truth, directly rewarding grounding over guessing.
- **Grounding supervision** — training on data that forces the model to point to *where* in the image its answer comes from (bounding boxes) strengthens the visual pathway against language-prior override.

## 9.5 Multimodal reasoning

The frontier that mirrors text reasoning models (post-training series Chapters 8, and the thinking-models threads). Multimodal reasoning models produce a **chain of thought over visual input** — reasoning step-by-step about an image before answering (visual math, diagram reasoning, multi-step chart analysis). The recipe parallels text:

- **Cold-start / long-CoT SFT** on verified visual reasoning traces (the model reasons about the image, then answers), seeding the thinking behavior.
- **RLVR** on verifiable visual tasks (visual math like MathVista, MMMU-style problems, counting) to reinforce correct reasoning — the R1 recipe with images.
- **The same length/entropy dynamics** as text reasoning (length explosion, entropy collapse — post-training series Chapter 8) apply, plus a multimodal-specific risk: the model can reason well but *lose track of the image*, reasoning about a hallucinated version of it. Keep verifying against the actual visual ground truth.

A 2026 caution worth heeding: reasoning VLMs can be *less robust to visual distractions* — strong reasoning over a misperceived image produces confident wrong chains. Grounding and hallucination control matter *more*, not less, as you add reasoning.

## 9.6 Guarding text and prior capabilities

The paired-eval discipline (Chapter 4.6, midtraining series Chapter 8) persists through post-training: multimodal post-training can degrade the base LLM's text ability and even earlier multimodal skills. Keep text-only and prior-multimodal evals running alongside the new-capability evals; hold the forgetting budget; replay text and prior data. A multimodal reasoning model that gained visual math but lost general instruction-following is often a bad trade.

## 9.7 Decisions

1. **Reuse the post-training algorithms unchanged** (SFT, DPO, RLHF, RLVR) with the image in context — the machinery is the same; the template/masking must match serving exactly.
2. **Treat hallucination as the primary target** — ground SFT data strictly, add negative/counterfactual examples, and measure hallucination explicitly (Chapter 10).
3. **Use multimodal preference optimization (mDPO-style) and RLVR** to penalize hallucination and reward grounding — ensure the method actually conditions on the image, not just the text.
4. **Build multimodal reasoning via cold-start visual-CoT SFT + RLVR on verifiable visual tasks**, and guard against reasoning over a *misperceived* image — grounding matters more with reasoning, not less.
5. **Run paired evals** (text + prior-multimodal + new capability) on every checkpoint and hold the forgetting budget.

Next: evaluation — measuring all of this on the axes that still discriminate.
