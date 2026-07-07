# Chapter 14 — Essential Reading and References

Multimodal moves fast; this list weights the foundational papers that defined the architecture, the strong 2025–2026 technical reports that show current recipes, and a few surveys. Read the essential cut in order; the topic list and provenance follow.

## The essential cut

| # | Source | One-line reason | Chapters |
|---|---|---|---|
| 1 | Liu et al., *Visual Instruction Tuning* (LLaVA, arXiv:2304.08485) + *LLaVA-1.5* (arXiv:2310.03744) | The founding adapter-VLM recipe; MLP connector, two-stage training | 2, 4 |
| 2 | Zhai et al., *SigLIP* (arXiv:2303.15343) + *SigLIP2* (arXiv:2502.14786) | The default vision encoder and why sigmoid pretraining won | 3 |
| 3 | *An Introduction to Vision-Language Modeling* (arXiv:2405.17247) | The best single survey of VLM architecture and training | 1–4 |
| 4 | Bai et al., *Qwen2-VL / Qwen3-VL Technical Reports* (arXiv:2409.12191, 2511.21631) | The strongest open VLM recipe: native resolution, dynamic tiling, multi-stage, encoder training | 3, 4, 6 |
| 5 | *InternVL* series (arXiv:2312.14238 and successors) | Dynamic tiling / AnyRes and scaled multi-stage VLM training | 3, 4 |
| 6 | *Toward Native Multimodal Modeling: A Roadmap* (arXiv:2605.25343) | The taxonomy and roadmap for adapter-vs-native and early fusion | 1, 5 |
| 7 | *Qwen3-Omni Technical Report* (arXiv:2509.17765) | The reference omni recipe: joint training, parity, thinker-talker streaming speech | 5, 7 |
| 8 | Li et al., *POPE: Object Hallucination in LVLMs* (arXiv:2305.10355) | The hallucination measurement that must run every checkpoint | 9, 10 |
| 9 | Yue et al., *MMMU* (arXiv:2311.16502) | The (now-saturated) flagship multimodal reasoning benchmark and its scope | 10 |
| 10 | *20/20 Vision Language Models: Better VLMs through Data Curation Alone* (arXiv:2605.11405) | The through-line: data curation beats architecture | 6 |
| 11 | Alayrac et al., *Flamingo* (arXiv:2204.14198) | Interleaved data, cross-attention fusion, and few-shot multimodal ICL | 2, 6 |
| 12 | *Molmo* (arXiv:2409.17146) + *LLaVA-OneVision* (arXiv:2408.03326) | Fully-open VLM data + recipe; reproducible end-to-end training | 4, 6, 11 |

## Why each, and what to extract

**1. LLaVA + LLaVA-1.5.** The founding recipe. Extract: the two-stage (align connector → instruction tune) structure and why the MLP connector replaced fancier ones. Everything in Chapters 2 and 4 descends from here.

**2. SigLIP / SigLIP2.** Your default eye. Extract: why sigmoid contrastive pretraining is more efficient than CLIP's softmax, and SigLIP2's added objectives (captioning, self-distillation, masked prediction) and native-resolution support.

**3. Introduction to Vision-Language Modeling (survey).** The clearest overview. Extract: the taxonomy of encoders, connectors, fusion strategies, and training approaches — the map for Chapters 1–4.

**4. Qwen2-VL / Qwen3-VL.** The strongest open VLM recipe. Extract: native dynamic resolution, the multi-stage pretraining (including encoder training), and the curated data mixture (OCR + grounding + STEM + GUI).

**5. InternVL.** Extract: dynamic high-resolution tiling (AnyRes) — how VLMs read documents — and scaled multi-stage training with progressive resolution.

**6. Native Multimodal Roadmap.** The framing for Chapter 5. Extract: adapter-vs-native tradeoffs, discrete-vs-continuous tokenization, early fusion, and the design principles (modality-aware norm, shared space, symmetric paths).

**7. Qwen3-Omni.** The reference omni model. Extract: LLM-frozen-first staging, the mix-unimodal-and-cross-modal-early ingredient for parity, and the thinker-talker architecture for streaming speech.

**8. POPE.** The hallucination probe. Extract: the polling-based object-presence method and why hallucination must be measured separately from accuracy.

**9. MMMU.** Extract: the benchmark's scope (expert reasoning across disciplines and image types) and, from Chapter 10, its saturation — why it's now a floor, not a differentiator.

**10. 20/20 VLMs (data curation).** The through-line paper. Extract: the evidence that curation moves quality more than architecture — invest in data first.

**11. Flamingo.** The other architectural lineage. Extract: interleaved image-text data, Perceiver Resampler, cross-attention fusion, and few-shot multimodal in-context learning.

**12. Molmo + LLaVA-OneVision.** Fully-open recipes. Extract: open data curation practices and reproducible end-to-end training — the frameworks/data reference for Chapters 6 and 11.

## Reading list by topic

- **Architecture & connectors:** LLaVA (2304.08485), LLaVA-1.5 (2310.03744), BLIP-2/Q-Former (2301.12597), Flamingo/Perceiver (2204.14198); the MLP-vs-resampler analyses.
- **Vision encoders:** CLIP (2103.00020), SigLIP (2303.15343), SigLIP2 (2502.14786), DINOv2 (2304.07193); mixture-of-encoders (2501.06986); NaViT native resolution (2307.06304).
- **VLM recipes & scaling:** Qwen2-VL/Qwen3-VL (2409.12191, 2511.21631), InternVL series (2312.14238), MM1 (2403.09611), Molmo (2409.17146), LLaVA-OneVision (2408.03326), Aria (2410.05993, native MoE), Gemma 3 (2503.19786).
- **Native / any-to-any:** Native Multimodal Roadmap (2605.25343), Chameleon (2405.09818), Emu3 (2409.18869), Janus (2410.13848), "Lexicalizing Modalities as Discrete Tokens" (2603.27538).
- **Audio / speech / omni:** Qwen3-Omni (2509.17765), Qwen3.5-Omni (2604.15804), MiniMind-O (2605.03937), Whisper (2212.04356), OmniZip audio-guided compression (2511.14582).
- **Video:** temporal-modeling and long-video papers; TempCompass, PerceptionTest, VideoMME benchmarks; token-compression methods.
- **Data:** the 20/20 curation paper (2605.11405), SVIT (2307.04087), recaptioning (2310.16656), GrIT/grounding synthesis, ICC caption concreteness (2403.01306), OCR/document data (TextHawk2, 2410.05261).
- **Post-training & hallucination:** POPE (2305.10355), HallusionBench, mDPO and multimodal preference methods, visual RLVR (MathVista-style), reasoning-VLM robustness (2606.08894).
- **Evaluation:** MMMU (2311.16502) and MMMU-Pro, POPE, ChartQA, DocVQA, OCRBench, video benchmarks; the 2026 multimodal-benchmark surveys (digitalapplied, llm-stats).

## Provenance notes

**Foundational papers (2021–2024):** CLIP, Flamingo, BLIP-2, LLaVA, SigLIP, MMMU, POPE, InternVL are peer-reviewed or heavily-cited and define the field's vocabulary; their mechanisms are settled and quoted directly.

**2025–2026 technical reports:** Qwen2/3-VL, Qwen3-Omni, SigLIP2, Molmo, LLaVA-OneVision, Gemma 3 are first-party accounts of current recipes; their numbers (staging, resolution, data mixtures) are the load-bearing practitioner facts in Chapters 3–7. As technical reports they under-report failures — read for what worked.

**2026-dated preprints:** The native-multimodal roadmap (2605.25343), data-curation paper (2605.11405), Qwen3.5-Omni (2604.15804), and several arXiv 2603–2606 entries postdate the January 2026 knowledge cutoff and were surfaced by mid-2026 web search; included because they sharpen the adapter-vs-native and data-curation arguments, but as recent preprints weight them below the settled 2023–2025 work.

**Benchmark-saturation claims** (MMMU-Pro ~80%+ across frontier models; the moved axes to video/audio/OCR/charts) come from mid-2026 benchmark surveys and reflect the state at compilation — re-check current leaderboards, since saturation and the discriminating axes move quarterly.

**A note on numbers:** Every concrete figure (SigLIP ~729 tokens/384-tile, up to 12 tiles, 10–30% text-replay, differential LRs ~1e-3 connector / ~2e-5 LLM / ~2e-6 encoder, MMMU-Pro ~80%) reflects mid-2026 practice from the sources above. They are defaults to start from and validate on your own data and evals — multimodal optima are data- and task-specific, and the field moves fast.
