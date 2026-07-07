# Chapter 14 — Reading List and References

The [Essential Reading](13-essential-reading.md) chapter is the twelve-source cut. This chapter is the path deeper, organized by topic, followed by the full source list with provenance notes. Where a URL is given as an arXiv id, prepend `https://arxiv.org/abs/`.

---

## Reading list by topic

### The stage itself — surveys and framing
- *Mid-Training of Large Language Models: A Survey* — arXiv:2510.06826. The primary framing, taxonomy, theory, and mixture tables.
- *A Survey on LLM Mid-Training* — arXiv:2510.23081. Companion survey; data-distribution / optimization / long-context decomposition.
- *Predicting Training Re-Evaluation Curves* — OpenReview (2026). How annealing/decay-phase gains can be predicted from partial runs.

### Annealing, WSD, and mixture selection
- Hu et al., *MiniCPM* — arXiv:2404.06395. The original careful WSD analysis and the "decay phase does the work" observation.
- OLMo 2 / OLMo 3 technical reports (AllenAI). Microanneals; the Dolmino annealing mixture; open costs.
- *RegMix: Data Mixture as Regression* — arXiv:2407.01492. Fit mixture→loss on tiny runs, then optimize.
- *Towards Efficient LLM Annealing with Principled Sample Selection* — arXiv:2605.31175. Sample selection for the decay phase.
- *Capacity-Aware Mixture Law* — arXiv:2603.08022. Mixture optimization accounting for model capacity.

### Continued pretraining and forgetting
- Ibrahim et al., *Simple and Scalable Strategies to Continually Pre-train LLMs* — arXiv:2403.08763. The re-warmup + replay recipe.
- Gu et al., *Efficient Continual Pre-training by Mitigating the Stability Gap* — arXiv:2406.14833. The stability gap, measured and mitigated.
- *ADEPT: Continual Pretraining via Adaptive Expansion and Dynamic Decoupled Tuning* — arXiv:2510.10071. Parameter-expansion anti-forgetting.
- *Context-aware Continual Pretraining* — OpenReview (id Mg6pVmTWlo). Regularization-based forgetting mitigation.
- *Replaying pre-training data improves fine-tuning* — arXiv:2603.04964. Replay as Pareto-improving, not just defensive.
- *Balancing Synthetic Data and Replay* — arXiv:2510.11842. Replay/synthetic trade for task-specific capability.
- *The Finetuner's Fallacy: When to Pretrain with Your Finetuning Data* — arXiv:2603.16177. Early inclusion vs late adaptation.
- *Bring Your Own Knowledge: A Survey of LLM Knowledge Expansion* — arXiv:2502.12598. The map of knowledge-injection methods.

### Domain adaptation
- Rozière et al., *Code Llama* — arXiv:2308.12950. 500B-token code CPT.
- Colombo et al., *SaulLM-7B* — arXiv:2403.03883; *SaulLM-54B/141B* — NeurIPS 2024. Legal domain CPT+SFT at scale.
- *Domain-Specific Pretraining of Language Models: A Comparative Study in the Medical Field* — arXiv:2407.14076. Medical CPT comparison.
- *TransformLLM / AdaptLLM (reading-comprehension conversion)* — arXiv:2410.21479, arXiv:2309.09530. Restructuring raw domain text into RC exercises.
- *The interplay between domain specialization and model size* — arXiv:2501.02068. The specialization-vs-size frontier.

### Long-context extension
- Peng et al., *YaRN* — arXiv:2309.00071. The dominant extension method.
- Ding et al., *LongRoPE* — arXiv:2402.13753. Extension beyond 2M tokens.
- Chen et al., *Position Interpolation* — arXiv:2306.15595. The foundational interpolation idea.
- *A Controlled Study on Long Context Extension and Generalization* — arXiv:2409.12181. Apples-to-apples comparison of extension methods.
- Hsieh et al., *RULER* — arXiv:2404.06654. Effective vs nominal context length.
- *Needle-in-a-Haystack* (community benchmark) and *∞Bench* — arXiv:2402.13718. Long-context evaluation.
- amaarora, *How LLMs Scaled from 512 to 2M Context* — blog (2025). The accessible technical walkthrough of RoPE scaling.

### Tokenizer and vocabulary surgery
- *EEVE: Efficient and Effective Vocabulary Expansion* — arXiv:2402.14714. Freeze-most, warm-start-embeddings recipe.
- *FOCUS / mean-of-constituents initialization* — arXiv:2305.14481. Subword-composition embedding init.
- *HYPEROFA: Hypernetwork-based Embedding Initialization* — arXiv:2504.21018. Learned init for large multilingual expansion.
- *Learned Embedding Propagation* (Russian adaptation) — arXiv:2412.21140. Practical language-adaptation vocabulary work.
- *Exploring Design Choices for Building Language-Specific LLMs* — arXiv:2406.14670. When vocabulary surgery pays off.

### Reasoning and capability injection
- *Front-Loading Reasoning* — ICLR 2026 (OpenReview VmEkhV2yCX). Reasoning-data placement across the pipeline.
- Meta RAM, *Thinking in Midtraining* — facebookresearch.github.io/RAM blog. Reasoning as a midtraining phenomenon.
- Qwen Team, *Qwen3 Technical Report* — arXiv:2505.09388. The reasoning-stage recipe.
- DeepSeek-AI, *DeepSeek-R1* — arXiv:2501.12948. The cold-start boundary midtraining now shares (cross-reference the post-training series).

### Frameworks and infrastructure
- *Megatron-LM* — arXiv:1909.08053; *NeMo* (NVIDIA docs). Large-scale CPT trainers.
- *torchtitan* — arXiv:2410.06511. PyTorch-native mid-scale trainer.
- Axolotl (github.com/axolotl-ai-cloud/axolotl), Unsloth (github.com/unslothai/unsloth). Ergonomic CPT/LoRA.
- *Continued LLM Pretraining in 2026: Frameworks, Strategies, Evaluation* — futureagi.com blog. Practical 2026 framework survey.

---

## Full source list with provenance notes

**Surveys (secondary, high-value):** The two mid-training surveys (2510.06826, 2510.23081) are the backbone of this series and were published October 2025; treat their recipe tables as authoritative summaries of the primary technical reports they cite. Provenance: peer-reviewed-adjacent arXiv surveys with extensive citation.

**Technical reports (primary):** Llama 3 (2407.21783), Qwen3 (2505.09388), DeepSeek-V3 (2412.19437), Code Llama (2308.12950), SaulLM (2403.03883), OLMo 2/3 (AllenAI). These are first-party accounts by the teams that ran the training; their numbers (token budgets, mixtures, LR, RoPE bases) are the load-bearing facts in Chapters 2–7 and were quoted directly. Caveat: technical reports underreport failures and negative results — read them for what worked, not for what didn't.

**Method papers (primary):** YaRN (2309.00071), LongRoPE (2402.13753), Position Interpolation (2306.15595), the continual-pretraining recipe (2403.08763), the stability-gap paper (2406.14833), RULER (2404.06654), and the vocabulary-expansion papers. Each introduces or rigorously evaluates a specific technique; cited for mechanism, not just result.

**2026-dated sources:** Several entries (Front-Loading Reasoning ICLR 2026; the replay/finetuner's-fallacy/mixture-law arXiv 2603–2606 preprints) postdate the January 2026 knowledge cutoff and were surfaced by mid-2026 web search; they are included because they sharpen recipes in Chapters 3, 7, and 8, but as recent preprints their conclusions are less settled than the 2024–2025 technical reports. Weight them accordingly.

**Blogs and secondary explainers:** The amaarora RoPE walkthrough, the futureagi frameworks survey, and the Meta RAM blog are pedagogically useful and current but are secondary sources; verify any specific number against the primary report before you build a run around it.

**A note on numbers:** Every concrete figure in this series (replay 10–30%, decay 10–20% of tokens, re-warm to ~1/3–1/10 peak, 30B tokens for domain competence, RoPE bases per target length, ~$2.75M for OLMo 3) is drawn from the sources above and reflects mid-2026 practice. They are defaults to start from and validate at small scale (microanneals, a baseline eval), not constants — the field moves, and your model, data, and budget will move the optimum.
