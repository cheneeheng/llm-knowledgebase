# Appendix A — Watermarking and Content Provenance

*A companion to Chapter 7, which mentioned content-provenance mandates in one line. That line became a legal obligation with teeth: from **August 2, 2026**, the EU AI Act's Article 50 requires machine-readable marking of AI-generated content, with penalties up to €15M or 3% of global turnover. If you deploy a generative system, this appendix is now part of your compliance surface — and it also covers the flip side, why *detecting* AI content remains unreliable.*

## A.1 The regulatory driver

Article 50 of the EU AI Act imposes transparency duties on generative AI: **AI-generated or -manipulated content must be marked in a machine-readable way**, deepfakes and synthetic media must be disclosed, and users interacting with an AI system must be told so. Timeline: obligations bite **August 2, 2026** for new systems; systems already on the market have until **December 2, 2026**. This converts watermarking from a nice-to-have into a deployment requirement for anyone serving EU users — and, per the constraints-before-capabilities rule (Chapter 1.3), it's a constraint to design in, not bolt on.

## A.2 The two-layer standard: C2PA + invisible watermarks

The industry converged on a **dual** approach, and 2026 best practice is to implement both on the same content:

- **C2PA Content Credentials** — a *signed metadata manifest* attached to the content, recording provenance (what tool made it, edits applied) with a cryptographic tamper-evidence chain. The coalition spans 6,000+ members (Google, Microsoft, Adobe, Meta, OpenAI, Amazon, BBC...), and major generators (Firefly, DALL-E 3, Sora, Imagen) already embed manifests. Strength: verifiable, rich, standardized. Weakness: **metadata strips** — a screenshot, a re-encode, or a hostile platform removes it.
- **Imperceptible watermarks (SynthID-class)** — a statistical signal embedded *in the content itself* (pixel patterns for images; for text, biased token sampling that a detector key can verify). Strength: survives metadata stripping. Weakness: weaker semantics (it says "AI-made by this provider," not the full provenance story), and it degrades under transformation.

The layers cover each other's failure mode — C2PA for verifiable provenance when intact, the watermark as the survivor when metadata is gone — which is why "both on the same content" is the converged recommendation and the sensible Article 50 implementation.

## A.3 Text watermarking and its limits

Text is the hard case, because text is trivially rewritten. The state of the art (SynthID-Text and academic kin, in the Kirchenbauer lineage) watermarks by **biasing token sampling** — imperceptibly skewing generation toward a keyed "green list" so a detector with the key can spot the statistical signature. It works impressively on unmodified output (~99.5% first-party detection) and degrades under attack: **paraphrase plus translation round-trips drop detection to ~75–85%**, and determined rewriting removes it. Short texts carry too little signal to detect at all. Design consequence: text watermarking satisfies the *marking* obligation and supports honest-actor provenance; it is **not** a mechanism that reliably catches an adversary laundering your model's output.

## A.4 Detection: the direction that doesn't work

The reverse problem — "is this content AI-generated?" without a watermark key — deserves a blunt statement because organizations keep buying it: **post-hoc AI-text detectors are unreliable and unfit for consequential decisions.** They false-positive on human text (non-native writers especially), false-negative on lightly edited output, and degrade with every model generation. Watermark-key detection works only on content from participating tools with the signal intact — "no watermark system covers the whole internet." Policy implication: treat provenance as something you *attach at creation* (A.2), never something you can *recover after the fact*; and never sanction a student, employee, or author on a detector score alone.

## A.5 Implementation checklist

For a team deploying generative features under Article 50:

1. **Inventory outputs** — which of your features generate or materially manipulate content (text, image, audio, video)?
2. **Attach C2PA manifests** to generated media at creation (the SDKs are mature; your image/video pipeline signs what it makes).
3. **Enable provider watermarks** — if you serve via API vendors, their watermarking (SynthID et al.) may already satisfy marking; confirm and document it. If you self-host generation, integrate a watermarking layer yourself.
4. **Disclose interactions and deepfakes** per Article 50's other prongs (chatbot disclosure, synthetic-media labels in the UI).
5. **Preserve marks through your pipeline** — a post-processing step that strips metadata undoes your compliance; test end-to-end that marks survive to the user.
6. **Document the scheme** as part of the Chapter 7.5 compliance artifacts, and re-check guidance — the GPAI Code of Practice and Article 50 implementation guidance are still evolving.

## A.6 Decisions

1. **Treat Article 50 marking as a launch requirement** for generative features touching EU users — deadlines August/December 2026, and the fine structure is real.
2. **Implement both layers** — C2PA manifests for verifiable provenance plus an imperceptible watermark that survives metadata stripping.
3. **Know what text watermarking can and can't do** — it satisfies marking and honest-actor provenance; it does not survive determined paraphrase, and short outputs are unmarkable.
4. **Never rely on post-hoc detection** — attach provenance at creation; refuse detector-score-only enforcement decisions.
5. **Test mark survival end-to-end and document the scheme** alongside your other compliance artifacts (Chapter 7.5).
