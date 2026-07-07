# Chapter 13 — Essential Reading

The full topic-organized path and complete source list are in the [Reading List and References](14-reading-list-and-references.md). This chapter is the ruthless cut: the sources that, read in order, give you every load-bearing idea in this series from its primary source. They are ordered as a reading sequence — early entries build the frame later ones assume.

Selection criteria: primary sources (surveys, technical reports, method papers) that each anchor a *decision* you will actually make in midtraining; overlapping sources lose to the one with the best signal-to-page ratio.

| # | Source | One-line reason | Chapters |
|---|---|---|---|
| 1 | *Mid-Training of Large Language Models: A Survey* (arXiv:2510.06826) | The framing, taxonomy, and theory for this entire stage in one place | 1, 2, all |
| 2 | *A Survey on LLM Mid-Training* (arXiv:2510.23081) | The companion survey; complementary recipes and data-mixture tables | 1, 2, 7 |
| 3 | Meta, *The Llama 3 Herd of Models* (arXiv:2407.21783) | The best worked example of annealing + six-stage long-context in a real run | 2, 5 |
| 4 | Qwen Team, *Qwen3 Technical Report* (arXiv:2505.09388) | The clearest modern three-stage general→reasoning→long-context recipe | 2, 5, 7 |
| 5 | DeepSeek-AI, *DeepSeek-V3 Technical Report* (arXiv:2412.19437) | The canonical two-phase YaRN long-context bolt-on, with numbers | 5 |
| 6 | Rozière et al., *Code Llama* (arXiv:2308.12950) | The reference domain-CPT: 500B tokens turning a general model into a code model | 3, 4 |
| 7 | Colombo et al., *SaulLM-7B* (arXiv:2403.03883) | Domain CPT then domain-SFT at the affordable 30B-token scale, with forgetting control | 3, 4 |
| 8 | Peng et al., *YaRN* (arXiv:2309.00071) | The context-extension method most 2026 models use; how RoPE frequencies are remapped | 5 |
| 9 | Ibrahim et al., *Simple and Scalable Strategies to Continually Pre-train LLMs* (arXiv:2403.08763) | The re-warmup + replay recipe that makes CPT work without forgetting | 3, 8 |
| 10 | AllenAI, *OLMo 2 / OLMo 3 Technical Reports* | Microanneals, the Dolmino annealing mix, and a fully-open end-to-end midtraining pipeline with costs | 2, 9, 10 |
| 11 | *Front-Loading Reasoning* (ICLR 2026) + Meta RAM *Thinking in Midtraining* | Why reasoning belongs in midtraining, and how much to front-load | 7 |
| 12 | Gu et al., *Efficient Continual Pre-training by Mitigating the Stability Gap* (arXiv:2406.14833) | The stability gap named, measured, and mitigated | 3, 8 |

## Why each, and what to extract

**1. Mid-Training of LLMs: A Survey (arXiv:2510.06826).** Read first. It defines the stage, gives the annealing/LR-schedule/long-context taxonomy, the SmolLM2/Qwen3/Llama-3 mixture tables, and the three theoretical framings (gradient noise scale, information bottleneck, curriculum). It is the skeleton this series puts muscle on. Extract: the mixture-ratio tables and the WSD-vs-cosine argument.

**2. A Survey on LLM Mid-Training (arXiv:2510.23081).** The parallel survey. Read for the additional recipes and the data-distribution/optimization/long-context decomposition. Extract: OLMo 2 microanneals and RegMix as mixture-selection methods.

**3. Llama 3 Herd (arXiv:2407.21783).** The single best worked example on this list. Extract: the annealing upsample of high-quality sources, the six-stage 8K→128K extension, and the honest data-composition percentages.

**4. Qwen3 (arXiv:2505.09388).** The cleanest modern staging. Extract: the general (30T) → reasoning (5T, STEM/code upweighted) → long-context (32K, length-mixed) three-stage structure — the template most teams should imitate.

**5. DeepSeek-V3 (arXiv:2412.19437).** Extract just the long-context section: two YaRN phases of ~1000 steps each, 4K→32K→128K. The minimal-cost extension recipe.

**6. Code Llama (arXiv:2308.12950).** The proof that domain competence is a capability wanting near-pretraining-scale CPT (500B tokens), not fine-tuning. Extract: the token budget and the long-context + instruction phases layered on top.

**7. SaulLM-7B (arXiv:2403.03883).** The affordable domain shape: 30B legal tokens CPT + legal-instruction SFT, with forgetting mitigation. Extract: the token budget most domain teams should target and the CPT→SFT two-phase pattern.

**8. YaRN (arXiv:2309.00071).** The method behind Chapter 5. Extract: the three-frequency-band treatment (extrapolate high, interpolate low, blend middle) and the 400–600-step extension efficiency claim.

**9. Simple and Scalable Continual Pre-training (arXiv:2403.08763).** The operational recipe for CPT: how much to re-warm the LR, how much replay, and the demonstration that a simple warmup+replay recipe matches full retraining. Extract: the re-warmup fraction and the replay-ratio guidance.

**10. OLMo 2 / OLMo 3 (AllenAI technical reports).** The only fully-open end-to-end midtraining pipelines, with real costs (~$2.75M / 1024 H100 / ~47 days for the 32B incl. base). Extract: microanneals for mixture selection, the Dolmino annealing mix, and the operational timeline.

**11. Front-Loading Reasoning (ICLR 2026) + Meta RAM "Thinking in Midtraining."** The evidence for Chapter 7: reasoning data yields more final capability spent in midtraining than in post-training SFT. Extract: the argument for where to place the reasoning prior and the moved cold-start boundary.

**12. Mitigating the Stability Gap (arXiv:2406.14833).** Names and fixes the dip-before-climb of CPT. Extract: the three mitigations (gentle warmup, higher-quality subset, multiple epochs over a subset) and why early checkpoints mislead.

## Near misses

Cut only for overlap: **LongRoPE** (arXiv:2402.13753 — read if you need 500K+ context), **RegMix** (mixture-ratio-as-regression; the surveys cover it), **AdaptLLM / reading-comprehension conversion** (arXiv:2309.09530 — read before adapting a small domain corpus), **EEVE / vocabulary-expansion** papers (read before tokenizer surgery, Chapter 6), and **RULER** (arXiv:2404.06654 — read before you report a long-context number).
