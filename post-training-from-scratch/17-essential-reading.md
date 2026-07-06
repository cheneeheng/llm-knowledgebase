# Chapter 17 — Essential Reading: The Twelve Sources That Matter Most

The full, topic-organized path is in the [Reading List](18-reading-list.md) and the complete source list in [References](19-references.md). This chapter is the ruthless cut: if you read only these twelve, you will have seen every load-bearing idea in this report from its primary source. Ordered as a reading sequence, not by importance — early entries build the frame later ones assume.

Selection criteria: primary sources plus one textbook and one codebase-shaped resource; each must anchor a *decision* you will actually make; overlapping sources lose to the one with the best signal-to-page ratio.

| # | Source | One-line reason | Chapters |
|---|---|---|---|
| 1 | Lambert, *The RLHF Book* (2024–, living text) | The one coherent map of the whole field, by the person who ran Tulu | all |
| 2 | Ouyang et al., *InstructGPT* (2022) | The founding recipe: SFT → RM → PPO, and why it exists | 1, 4, 6, 7 |
| 3 | Rafailov et al., *DPO* (2023) | The derivation that made preference optimization a commodity | 5 |
| 4 | Lambert et al., *Tulu 3* (2024) | The complete, ablated, reproducible open recipe — SFT mixes to RLVR | 3, 4, 5, 15 |
| 5 | Gao et al., *Scaling Laws for Reward Model Overoptimization* (2022) | The Goodhart curves that govern every proxy you'll ever optimize | 6, 16 |
| 6 | Bai et al., *Constitutional AI* (2022) | AI feedback + principles: the template for scalable safety and RLAIF | 3, 11 |
| 7 | DeepSeek-AI, *DeepSeek-R1* (2025) | The thinking-model recipe, open: GRPO, cold start, four stages, distillation table | 7, 8, 10 |
| 8 | Shao et al., *DeepSeekMath* (2024) | Where GRPO actually comes from, with the math spelled out | 7 |
| 9 | Yu et al., *DAPO* (2025) | The hardened open RLVR recipe: clip-higher, dynamic sampling, and why each trick exists | 7, 8 |
| 10 | Sheng et al., *HybridFlow / verl* (2024) | The RL-systems problem — trainer vs rollout engine — and the design that won | 12, 13 |
| 11 | Thinking Machines, *On-Policy Distillation* (2025) | Dense-reward distillation: the small-model workhorse, with honest compute accounting | 10 |
| 12 | *A Practitioner's Guide to Multi-Turn Agentic RL* (2025) | What actually changes when RL meets environments, from people who ran it | 9 |

## Why each, and what to extract

**1. The RLHF Book — rlhfbook.com.** Read it as the textbook spine for this report: notation, derivations (BT models, PPO/GRPO, DPO), and sober assessments of what works. It is continuously updated, which no paper on this list can claim.

**2. InstructGPT — arXiv:2203.02155.** Read for: the three-stage structure everyone still uses, the human-data methodology (annotator instructions are the ancestor of model specs), and the first honest measurement of an alignment tax.

**3. DPO — arXiv:2305.18290.** Read for: the closed-form trick (reward as log-ratio) — once you see it, half the preference-optimization literature becomes corollaries. The loss-curve pathologies in its wake (Chapter 5.2) are the practical footnote the paper lacks.

**4. Tulu 3 — arXiv:2411.15124.** The Llama-3-Herd of post-training: every mixture, every ablation, every hyperparameter, all reproducible. Read for: how mixture decisions get made empirically, and what a complete modern recipe looks like end-to-end. Follow with the Olmo 3 report for the 2025-26 evolution (Dolci, thinking variants).

**5. RM Overoptimization — arXiv:2210.10760.** Read for: the peak-then-fall curves against a gold RM, and the scaling of the peak with RM size/data. This is the quantitative backbone of "every proxy can be gamed" — the report's central warning.

**6. Constitutional AI — arXiv:2212.08073.** Read for: critique-and-revise data generation and RLAIF — the mechanism by which principles scale into training data. Pair with OpenAI's deliberative-alignment paper for the reasoning-era version.

**7. DeepSeek-R1 — arXiv:2501.12948.** The document that reset the field. Read for: R1-Zero's emergence evidence (the "aha moment"), the four-stage pipeline, *and* the distillation-beats-small-RL table — the most consequential single table in post-training.

**8. DeepSeekMath — arXiv:2402.03300.** Read for: GRPO's actual formulation (group baseline, the KL variant) and the data-side story everyone skips — the math-corpus construction that made the RL possible. The substrate lesson of Chapter 8 in primary form.

**9. DAPO — arXiv:2503.14476.** Read for: each stabilization trick tied to the specific failure it fixes (entropy collapse, zero-gradient groups, truncation bias) with ablations, plus a fully open reproduction. The best single document on *operating* RLVR.

**10. HybridFlow/verl — arXiv:2409.19256.** Read for: why RL post-training is a distributed-dataflow problem, the colocated/disaggregated tradeoff, and the resharding machinery every framework now copies. Chapter 12 in primary form.

**11. On-Policy Distillation — thinkingmachines.ai/blog/on-policy-distillation.** Read for: reverse-KL-on-student-rollouts explained cleanly, the compute accounting versus RL, and the capability-recovery use case. The clearest writing on why on-policy beats off-policy — which illuminates RL too.

**12. Practitioner's Guide to Multi-Turn Agentic RL — arXiv:2510.01132.** Read for: environment/reward design pitfalls, credit assignment across turns, and the infra ladder (sync→async, partial rollouts) grounded in real runs — Chapter 9's field manual.
