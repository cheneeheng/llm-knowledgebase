# Appendix B — Automatic Prompt Optimization

*A companion to Chapters 2 and 9. The series treats prompts as something you write and evaluations as something you run. It missed the technique that connects the two: **optimizing prompts automatically against your evals.** Tools for this (DSPy's optimizers, and GEPA in particular) matured in 2025–2026. GEPA was an ICLR 2026 oral and reports beating RL fine-tuning on some tasks with a small fraction of the rollouts. If you have an eval set and a metric but cannot or should not fine-tune, this is the next step up.*

## B.1 The idea: prompts are parameters

Every LLM application has trainable parameters that almost nobody trains: the **instructions and few-shot examples** in its prompts. Hand-tuning them has three problems. It does not scale across a multi-step pipeline, where changing one prompt shifts the inputs to the next. It overfits to the examples the engineer happens to look at. And it has to be redone for every model migration, because a prompt tuned for one model is not tuned for its successor.

Automatic prompt optimization treats the prompt text as the object of optimization. Given:

1. **a program**: one or more LLM calls, possibly with tools and retrieval,
2. **a metric**: your Chapter 9 evals, as code checks, LLM judges, or both,
3. **a training set** of inputs (with labels, or with a judge that needs none),

an optimizer proposes prompt variants, runs the program, scores the results, and keeps what works. It is the same loop as fine-tuning, except that the parameters are text and the "gradient" comes from search or from an LLM's reflection on what went wrong.

## B.2 The methods, from oldest to current

- **Instruction search (APE, OPRO; 2022–2023).** An LLM proposes candidate instructions, each is scored on a dev set, and the best feed the next round. OPRO frames the LLM as the optimizer, with a running history of (prompt, score) pairs in its context. Simple, and effective for single prompts.
- **Demonstration bootstrapping (DSPy's BootstrapFewShot).** Run the program and keep traces that pass the metric as few-shot examples. This automates the most reliable prompt improvement there is: good examples.
- **Joint instruction and demonstration search (DSPy MIPROv2).** Proposes instructions grounded in the data and the program, bootstraps demonstration sets, and uses Bayesian optimization to choose combinations across *all* modules of a pipeline at once. This was the strong default through 2025.
- **Reflective evolution (GEPA; Agrawal et al., arXiv:2507.19457, ICLR 2026 oral).** The current state of the art, and the method to learn. Instead of reducing each run to a scalar score, GEPA reads the **execution traces**: reasoning, tool calls, tool outputs, error messages, and the judge's rationale. It then asks an LLM to diagnose *why* a candidate failed and propose a targeted edit. It keeps a **Pareto frontier** of candidates, meaning prompts that are best on at least one training example rather than only the top average scorer. That preserves diverse strategies and avoids premature convergence. Reported results: about **10% better than GRPO fine-tuning on average (up to 20%) with ~35× fewer rollouts**, and double-digit gains over MIPROv2 with much shorter prompts. It ships as `dspy.GEPA` and as a standalone library that can also optimize code and other text artifacts.
- **Textual-gradient methods (TextGrad and kin)** backpropagate LLM-written "feedback" through a computation graph. They are conceptually close to GEPA's reflection, but less common in production.

**Why reflection wins:** a scalar reward says *that* a run failed. A trace says *why*. When the feedback is rich (a stack trace, a judge's explanation, a failed unit test), an LLM can make a targeted fix that would take RL thousands of samples to find. The same lesson runs through Chapter 3.2: errors written as instructions teach, and bare scores do not.

## B.3 Where it sits on the improvement ladder

Given an application that is underperforming, climb in this order:

1. **Fix the context and tools** (Chapters 2–3). Most failures are information or interface problems, and no optimizer can fix a missing document or a confusing tool.
2. **Optimize the prompts automatically** (this appendix). This is cheap: hundreds of program runs, no GPUs, and it works on closed APIs.
3. **Fine-tune or RL** (post-training series), when the model lacks a capability rather than an instruction, when you need a smaller or cheaper model to match a bigger one, or when the prompt cost at your volume exceeds a training run.

Steps 2 and 3 are complementary, not rival approaches. Optimized prompts make a strong starting point, and a good source of trajectories, for later fine-tuning. Methods that combine prompt optimization with weight updates exist for this reason.

## B.4 How to run it well

- **The metric is everything.** The optimizer will exploit whatever you measure. With an LLM judge, it will find prompts that please the judge. Use Chapter 9.4's order (rules first, then models), keep a **held-out test set the optimizer never sees**, and read a sample of optimized outputs before shipping. Prompt optimization can reward-hack just as RL can (post-training series Chapter 16).
- **Give the optimizer rich feedback.** For GEPA, return the judge's rationale, the failing assertion, and the error text, not just a pass/fail bit. The quality of the feedback bounds the quality of the fix.
- **Data size.** Tens to a few hundred training examples are typical, with a separate validation set. Too few and the optimizer overfits to them, so check how the optimized prompt does on the held-out set.
- **Optimize the whole pipeline, not one call.** Multi-step programs (RAG → reason → format; planner → executor) are where manual tuning fails and joint optimization pays off most. Multi-agent systems are harder, and recent benchmarks (MAS-PromptBench, arXiv:2606.23664) study when optimization helps them. Measure end-to-end, not per agent.
- **Optimize cheap, verify expensive.** Optimization runs cost many model calls. Research on cross-tier transfer (arXiv:2608.10694) explores optimizing against a cheaper model and deploying on a stronger one. Treat transferred prompts as candidates, and always re-validate on the deployment model.
- **Keep the output cache-friendly.** The optimized prompt becomes part of your stable prefix. Freeze it, version it, and keep it byte-identical between releases so Chapter 2.4's prompt-cache savings survive.
- **Re-optimize on model migration.** When you change models or tiers (Appendix A.5), re-run the optimizer from your current prompts before comparing. Otherwise you are comparing a tuned prompt against an untuned one.

## B.5 Decisions

1. **Treat prompts as trainable parameters** whenever you have a metric and a few hundred examples. Optimize them rather than hand-tuning.
2. **Default to GEPA-style reflective optimization**, fed with rich textual feedback. Use MIPROv2-style joint search when you only have scalar scores.
3. **Climb the ladder in order:** context and tools → prompt optimization → fine-tuning/RL.
4. **Guard against metric exploitation** with a held-out test set and human review of optimized outputs.
5. **Optimize whole pipelines end-to-end, freeze and version the result, and re-optimize on every model migration.**

## B.6 Sources

- Agrawal et al., *GEPA: Reflective Prompt Evolution Can Outperform Reinforcement Learning* (ICLR 2026 oral) — arXiv:2507.19457; code — https://github.com/gepa-ai/gepa
- Khattab et al., *DSPy: Compiling Declarative Language Model Calls into Self-Improving Pipelines*, 2023 — arXiv:2310.03714; Opsahl-Ong et al., *Optimizing Instructions and Demonstrations for Multi-Stage Language Model Programs* (MIPROv2), 2024 — arXiv:2406.11695
- Zhou et al., *Large Language Models Are Human-Level Prompt Engineers* (APE), 2022 — arXiv:2211.01910; Yang et al., *Large Language Models as Optimizers* (OPRO), 2023 — arXiv:2309.03409
- Yuksekgonul et al., *TextGrad: Automatic "Differentiation" via Text*, 2024 — arXiv:2406.07496
- *MAS-PromptBench: When Does Prompt Optimization Improve Multi-Agent LLM Systems?*, 2026 — arXiv:2606.23664
- *Optimize Cheap, Deploy Strong: Cost-Aware Cross-Tier Transfer for Evolutionary Optimization*, 2026 — arXiv:2608.10694
