# Chapter 4 — Domain Adaptation

## 4.1 Conceptual picture

Domain adaptation is CPT (Chapter 3) pointed at a specific vertical — code, medicine, law, finance, science, a company's internal corpus — with the goal of a model that is materially better *in that vertical* while still being a usable general model. It is the most common reason a team touches midtraining at all, because the value proposition is legible to a business: "our model understands radiology reports / SEC filings / our codebase better than GPT-class general models, and we own it."

The core tension, which every decision below is really about: **specialization trades against generality, and against model size.** A 2025 result made this precise — the smaller the model, the *more* domain specialization helps relative to a bigger general model, but the more you specialize, the more general capability you spend. There is a frontier: for a fixed deployment budget you choose a point on it (very specialized small model vs. lightly-adapted larger model). Domain adaptation is the practice of choosing and reaching that point without falling off the general-capability cliff.

## 4.2 The case studies (what actually worked)

These are the load-bearing published examples; steal their numbers.

**Code — Code Llama.** Continued Llama 2 on **500B code tokens** (plus a small long-context and instruction phase). The 7B reached vanilla Llama 2 70B on code benchmarks. Lesson: code competence is a *capability*, not a flavor — it wanted a near-pretraining-scale corpus, and it paid off enormously. This is the high end of domain adaptation, closer to a second pretraining.

**Legal — SaulLM.** Continued Mistral-7B on **30B legal tokens** (SaulLM-7B), later scaled to 54B/141B models on more. CPT then legal-instruction SFT. Beat general models on LegalBench while remaining a working general model, using explicit forgetting mitigation. Lesson: **30B tokens is the sweet spot for serious domain competence** on a 7B — enough to install the domain, small enough to be affordable and to limit forgetting.

**Medical — HEAL / Apollo / PMC-LLaMA.** HEAL continued LLaMA2-13B on **~15B medical tokens**; Apollo used ~2.5B multilingual medical tokens; the medical-comparative-study line shows domain CPT beating general models on MedQA-style benchmarks. Lesson: medicine is knowledge-dense and the corpora are smaller/cleaner, so **10–15B tokens** goes a long way; multilingual medical needs less because the reasoning transfers.

**Finance — BloombergGPT (the cautionary counterexample).** Trained a 50B model roughly half on financial data *from scratch* — expensive, and later general-model + light-adaptation approaches largely matched it. Lesson: **from-scratch domain pretraining is almost always the wrong call in 2026**; adapt a strong general base instead. This is the domain-adaptation version of "don't pretrain if you can post-train."

## 4.3 Token budget and hyperparameters

Consolidated defaults for adapting a 7–13B dense base:

| Knob | Default | Notes |
|---|---|---|
| Tokens | 10–50B for domain competence | 500B only for a full capability like code |
| LR (full CPT) | re-warm to ~2–3e-4, decay to ~0 | Chapter 3's re-warmup; lower (~1e-4) for 70B+ |
| Optimizer | AdamW, β=(0.9, 0.95), wd 0.01 | Same as pretraining |
| Epochs | 1–4 | More for small corpora, watch memorization |
| Replay | 10–30% general data | Non-negotiable; more for aggressive LR / big shift |
| Seq length | Match eventual use | Extend context separately (Chapter 5) if needed |
| Method | Full CPT for capability; LoRA for flavor | See 4.5 |

A LoRA-only domain CPT on a 7B with ~5B tokens finishes in **1–3 days on 4–8 H100s** — the entry point for a small team. Full CPT on 30B tokens is a multi-node cluster-week.

## 4.4 Reading-comprehension conversion: making a small corpus go further

A recurring problem: your domain corpus is small (a few billion tokens of internal docs, or a niche field) — too small for a rich CPT, too big to hand-label. The **reading-comprehension conversion** technique (AdaptLLM, TransformLLM, and successors) rewrites raw domain text into *reading-comprehension exercises* — inject questions, summaries, and tasks derived from each document — using a capable LLM. Training on the converted corpus installs domain knowledge *and* task-following simultaneously, and consistently beats CPT on the same raw text. It is the domain-adaptation twin of the annealing-phase synthetic-data trick (Chapter 2): **synthetic restructuring extracts more capability per domain token than raw next-token prediction over that token.**

Practical recipe: convert ~30–50% of your domain corpus to reading-comprehension format, keep the rest raw, mix in replay, CPT. This is now the default for small-to-medium domain corpora.

## 4.5 Full CPT vs LoRA for domains

The Chapter 3 spectrum applied to adapters:

- **LoRA/QLoRA** — the default for *domain flavor*: adapting tone, terminology, format, a bounded knowledge boost, per-customer specializations. Fits on one card, forgets far less (base is frozen), and the "LoRA without regret" result (post-training series, Ch 4) says it matches full FT whenever the domain's information content fits the adapter — true for most sub-billion-token domain corpora *if* you apply LoRA to all layers and use ~10× LR.
- **Full CPT** — required when the domain is a genuine *capability* needing 100B+ tokens (code), or when you are building a flagship domain model family and will post-train on top. LoRA's capacity binds on large corpora.

Decision rule: **under ~1–5B domain tokens and a flavor/knowledge goal → LoRA. Above that, or a real new capability → full CPT.**

## 4.6 The forgetting budget, made concrete

The one metric that keeps domain adaptation honest: run your **general eval before and after** (MMLU + HellaSwag + a reasoning eval). A well-tuned domain CPT with ~20% replay should show **no more than a 1–2 point absolute drop** on general benchmarks. **More than ~3 points means your replay is too low or your LR too high** — dial replay up or LR down and rerun. Domain gains that come with a 5+ point general drop are usually a bad trade unless the deployment is truly single-domain (and even then, users find the edges). Chapter 8 covers the mechanics; the discipline is: measure both axes on every checkpoint, always.

## 4.7 Decisions

1. **Adapt a strong general base; never domain-pretrain from scratch** (BloombergGPT's lesson).
2. **Budget 10–50B tokens** for real domain competence on a 7–13B model; 500B only for a full capability.
3. **Convert part of a small corpus to reading-comprehension format** to extract more per token.
4. **LoRA for flavor/knowledge under a few billion tokens; full CPT above that or for new capabilities.**
5. **CPT then domain-SFT** for a shippable domain assistant (SaulLM shape).
6. **Hold the forgetting budget to ≤2 points** on general evals via 10–30% replay; treat a >3-point drop as a bug.

Next: long-context extension — the one midtraining technique with its own distinctive machinery (RoPE surgery) rather than being CPT with a different mixture.
