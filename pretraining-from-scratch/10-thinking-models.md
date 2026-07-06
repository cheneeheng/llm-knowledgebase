# Chapter 10 — Thinking / Reasoning Models and What Pretraining Owes Them

"Reasoning models" (the o-series, DeepSeek-R1-style, Gemini/Claude thinking modes, Qwen/Kimi reasoning variants) are primarily made by **post-training** — reinforcement learning on verifiable rewards (RLVR) that teaches long chain-of-thought — plus distillation of those traces into smaller models. All of that is out of scope for this report. But pretraining determines the *ceiling* of what that post-training can achieve, and 2026 practice has shifted accordingly. This chapter is about exactly that interface: the pretraining decisions a future reasoning model will cash in.

## 10.1 What pretraining contributes to reasoning

- **Reasoning is latent in the base model.** RLVR largely *elicits and sharpens* reasoning behaviors the base model already has in distribution — it amplifies; it does not create from nothing. If the base model has weak math, code, and logical structure, RL has little to work with. The reasoning ceiling is set at pretraining.
- **Data composition is the lever.** Heavy, high-quality **math and code** in pretraining — especially concentrated in the annealing phase — is the strongest pretraining predictor of downstream reasoning. This is a major reason 2026 mixtures carry large code fractions and invest in layout-preserving math extraction (Nemotron-CC-Math and kin): standard extractors mangle LaTeX/MathJax, and malformed equations teach malformed reasoning. Recall from Chapter 4 that Llama-3's 2024 mix was already ~25% math/reasoning + 17% code; the reasoning-relevant share has only grown.
- **Synthetic reasoning traces in mid/late pretraining.** Labs increasingly inject large volumes of synthetic step-by-step reasoning, worked solutions, and structured problem-solving data late in pretraining or in a distinct "midtraining" stage — priming the model for chain-of-thought before formal post-training begins. This is the clearest example of the pretraining/post-training boundary moving: data that looks instruction-like or CoT-like now routinely appears *before* any RL or SFT, as part of the token stream.

## 10.2 Architectural implications

- **Long context matters more.** Long chains of thought and tool-use traces need room — thinking models routinely generate tens of thousands of tokens per answer. Budget the context-extension phase (Chapter 5.6) accordingly, and consider the final context target when picking RoPE θ from day one.
- **Multi-token prediction and strong code/math grounding** help the substrate (Chapter 5.7).
- **Efficiency architectures interact with reasoning — validate carefully.** The MiniMax episode (Chapter 5.5) is directly relevant: linear attention that looked fine on ordinary prompts underperformed on *reasoning and multi-turn/agentic* tasks — exactly the regime reasoning models live in. If reasoning is the goal, any efficiency-oriented attention variant must be ablated on reasoning-style and long-context-retrieval evals, not just perplexity. (The counterpoint — Kimi Linear, Qwen3.5 hybrids — shows it can be done, with care.)
- **Inference economics shape the architecture choice.** Reasoning models pay their compute at *inference* (long generated traces), which amplifies the value of cheap-per-token architectures: high-sparsity MoE, MLA/hybrid attention, and speculative decoding via retained MTP heads. This is part of why the 2026 frontier converged on sparse-MoE-plus-efficient-attention.

## 10.3 What this means for your run

Even though you stop at pretraining, if the eventual goal is a reasoning model, make four choices deliberately:

1. **Weight math and code heavily** in the mixture, with layout-preserving extraction.
2. **Plan a strong annealing phase** rich in high-quality reasoning-adjacent and synthetic reasoning data (decontaminated against your evals).
3. **Extend context enough** to hold long chains of thought — 32K minimum, 128K standard in 2026.
4. **Evaluate reasoning-relevant signals during pretraining** — GSM8K/MATH few-shot curves across checkpoints tell you whether the substrate is forming — and hand your post-training team permanent checkpoints and a documented data recipe so they know what the base model saw.
