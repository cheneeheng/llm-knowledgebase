# Chapter 2 — Scoping and Base Models

## 2.1 Do you need to post-train at all?

The cheapest post-training run is the one you skip. Work up this ladder and stop at the first rung that meets your bar:

1. **Prompting / system prompts** — a good frontier model with a well-engineered system prompt covers a shocking fraction of "we need a custom model" requests.
2. **RAG / context engineering** — knowledge problems are context problems, not weight problems. Do not fine-tune to inject facts; SFT is bad at adding knowledge and good at changing behavior.
3. **Fine-tune (SFT/LoRA)** — when you need consistent *behavior* prompting can't hold: strict output formats, a house style, a narrow task at low latency/cost, distilling a frontier model's behavior on your task into a small cheap model.
4. **Preference optimization / RL** — when you need to *exceed* what you can demonstrate: optimizing pass rates on verifiable tasks, agent reliability, subtle quality axes with a reward signal. RL is never the first tool.

Legitimate reasons to run the full pipeline: cost/latency (replace a frontier API with a 4–14B specialist), data privacy/on-prem, deep domain behavior, a capability the base ecosystem lacks, research, or because you *are* the model provider. Anti-reason: "fine-tuning will make it know our docs" — that's RAG.

## 2.2 Choosing the base model

Unless you also pretrain, your base model is a download, and this is the highest-leverage decision you will make. Criteria, in order:

- **Ceiling on your target capability.** RL elicits what pretraining installed. For reasoning/code targets, pick bases with heavy math/code pretraining (this is most of why Qwen bases dominate open RLVR research — many RL results replicate on Qwen and fail on weaker bases).
- **Size and family shape.** Pick the smallest size that can carry the capability; prefer families with multiple sizes (develop the recipe small, apply big, distill down). Sub-4B: distill, don't RL. MoE models bring RL-specific instabilities (Chapter 7) — prefer dense until you have infra maturity.
- **Base vs instruct starting point.** Full pipeline from *base* gives you control and a clean substrate (and is required for serious RL recipes). Light-touch work (LoRA on a narrow task, style tuning) on top of a good *instruct* model is fine and much cheaper — you inherit their post-training, including its quirks and refusal behavior.
- **License.** Apache-2.0/MIT (Qwen, OLMo, Mistral, DeepSeek) are clean. Llama licenses carry use restrictions and branding clauses; some "open" licenses restrict outputs or commercial use. Also check the *ToS chain* for distillation: training on outputs of OpenAI/Anthropic/Google APIs violates their terms — relevant when generating synthetic data (Chapter 3).
- **Ecosystem fit.** Tokenizer/chat-template maturity, day-1 support in vLLM/SGLang and your training framework, existing fine-tunes as evidence, published recipe details (OLMo 3 is the gold standard of documentation; the model flow — every checkpoint, dataset, and stage — is public).

The 2026 shortlist: **Qwen 3 / Qwen 3.5** (default for reasoning/agent work, all sizes, Apache), **Llama 3.3 / 4** (strong general dense, license caveats), **DeepSeek-V3.x** (frontier-class open MoE, if you can serve it), **Gemma 3** (strong small dense), **OLMo 3** (fully open flow — best for learning and research), **GLM, Kimi K2** (frontier-class open MoE, agentic emphasis), **SmolLM 3** (sub-2B, fully open recipe).

## 2.3 Cost by ambition level

Orders of magnitude, mid-2026 cloud prices (H100 ≈ $2–3/hr):

| Ambition | Typical shape | Compute cost |
|---|---|---|
| LoRA/QLoRA SFT, 4–8B, narrow task | 1 consumer GPU or 1×A100/H100, hours | $5–50 |
| Full-param SFT + DPO, 8B, general assistant (Tulu-3-class) | 1×8 H100 node, ~1–3 days total | $500–3k |
| RM training + modest RLHF/GRPO, 8B | 1–4 nodes, days | $2k–20k |
| Serious RLVR reasoning model, 14–32B | 16–64 H100s, 1–4 weeks, many iterations | $30k–300k |
| Agentic RL (SWE/computer-use), 32B–MoE | 64–512 GPUs + environment fleet, weeks–months | $200k–several M |
| Frontier full pipeline, model family | thousands of GPUs, months; RL compute now approaching pretraining scale at the top labs | $10M–100M+ |

Two structural notes. First, **iteration multiplies everything**: the table is per *final* run; real projects spend 3–10× the final-run cost on experiments that get thrown away. Second, **people and evals dominate small-scale budgets**: at the $500–20k tiers, the GPU bill is noise next to the engineering time spent on data and evaluation. That inversion — signal over FLOPs — is the defining economics of post-training.

## 2.4 Scope like an empiricist

Before any training: (a) write down the target behavior as an **eval you can run** (Chapter 14) — if you can't measure it, you can't train it; (b) score the base/instruct model and the best API model on that eval to bracket the gap; (c) pick the *cheapest stage* that plausibly closes the gap; (d) define kill criteria. Post-training projects die from unbounded iteration on a vague goal far more often than from any technical failure. The eval comes first; the training run is just the move that improves it.
