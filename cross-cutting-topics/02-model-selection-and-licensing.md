# Chapter 2 — Model Selection and Licensing

## 2.1 Conceptual picture

Before you train, fine-tune, distill, or serve anything, you make two coupled decisions that govern everything downstream: **build or buy** (train/host your own model vs call an API) and, if you build on an open model, **under what license.** These are constraint decisions (Chapter 1.3) — get them wrong and completed technical work becomes worthless. A fine-tune of a model whose license forbids your use case is legally undeployable; a product built on an API that changes pricing or terms is at someone else's mercy. This chapter is how to make both decisions well, and why the 2026 answer is almost always a *hybrid*.

## 2.2 Build vs buy

The first fork, expanded from the inference series' serving-side treatment (Ch 12) to the whole lifecycle:

**Buy (call a hosted frontier API — Claude, GPT, Gemini) when:**
- You need frontier capability you can't reproduce.
- Volume is low, spiky, or unpredictable (API per-token beats your idle self-hosted cost).
- Time-to-market and zero-ops matter more than control.
- You lack the team to run training/serving infrastructure.

**Build (train/fine-tune and self-host an open model) when:**
- You need a specific model — your fine-tune, a domain-adapted model, a small specialized one — no API offers.
- Privacy/compliance forbids sending data to a third party (Chapter 7).
- Volume is high and steady enough that amortized cost beats API pricing (the crossover favors building only above meaningful scale).
- You need control over the model's availability, versioning, and behavior.

**The 2026 default is neither — it's hybrid.** "Most teams end up mixing: a self-hosted open-weight model for sensitive data, a cheap API for high-volume tasks, and a frontier model for the hardest work." The winning architecture is a **router over a portfolio** (Chapter 1.4): route each request to the cheapest model that can handle it, self-host what's private/high-volume, call APIs for frontier-hard queries. Don't choose one globally; segment and route. This also hedges every risk — no single vendor lock-in, no single cost structure, no single point of failure.

## 2.3 The spectrum of "openness"

If you build, you build on an open model, and "open" is a spectrum that matters legally and practically:

- **Open source (the strict definition).** Weights *and* training code *and* training data *and* eval pipelines *and* intermediate checkpoints, all public under a permissive license — you can reproduce the model from scratch. As of 2026 this bar is met by very few families: **OLMo** (Allen Institute, Apache 2.0, everything public) and **StarCoder 2** (BigCode) are the canonical examples. Choose these when reproducibility, auditability, or full-stack understanding matters (research, high-assurance, or building deep expertise — as the training series' use of OLMo attests).
- **Open weight (the common case).** The trained *weights* are downloadable, but training data and/or code are proprietary. This covers most "open" models — Llama, Qwen, Mistral, DeepSeek, Gemma. You can run, fine-tune, distill, and serve them, but you can't reproduce or fully audit them. For most teams, open-weight is what "open model" means and is entirely sufficient.
- **Open weight, restricted license.** Weights are available but the *license* restricts use (see 2.4) — the trap layer.
- **Closed (API-only).** No weights; you consume via API. The "buy" side.

## 2.4 The license landscape

The load-bearing detail, because a license clause is a constraint that invalidates work. The 2026 landscape:

| License | Models (2026 examples) | Commercial use | Watch for |
|---|---|---|---|
| **Apache 2.0** | Qwen3/3.5, Gemma 4, Mistral Small 4, OLMo | Yes, unconditional | The gold standard; patent grant included |
| **MIT** | DeepSeek V4/R1, Phi-4 | Yes, unconditional | Gold standard; minimal, permissive |
| **Custom community license** | Llama 4 (Meta) | Yes, *with conditions* | **The 700M-MAU threshold** and acceptable-use/naming clauses |
| **Open-RAIL** | StarCoder 2, some others | Yes, *with use restrictions* | Behavioral use restrictions (no specified harmful uses) |
| **Non-commercial / research** | some releases | **No commercial use** | Deployable for research only — a common trap |

**Apache 2.0 and MIT are the "gold standard" — unconditional commercial use, no strings.** Default to these when license simplicity matters. The traps:

- **Custom community licenses (Llama's)** permit commercial use but with conditions — the famous 700M-monthly-active-user threshold (above which you must negotiate with Meta), acceptable-use policies, and naming/attribution requirements. Fine for most, a landmine for large deployments and specific use cases. *Read the actual license, not the "it's open" headline.*
- **Non-commercial licenses** forbid commercial deployment entirely — deployable for research/eval only. Teams routinely fine-tune on these and discover too late they can't ship.
- **Derived-model and distillation clauses** — some licenses restrict what you can do with *outputs* or *distilled* models (can you train a competitor on this model's outputs?). Critical if your plan is distillation (Chapter 3, post-training series Ch 10). Check the *output* terms, not just the *weight* terms.
- **Training-data license (separate!)** — the model's license and the *data's* license are different things; even an Apache model may have been trained on data with its own constraints, and *your* fine-tuning data has its own license/copyright status (Chapter 7).

## 2.5 The selection checklist

Choosing a base model in 2026, in order:

1. **Constraint filter first** (Chapter 1.3): Does the license permit your commercial use, your scale, your distillation plan? Does it comply with your regulatory environment (Chapter 7)? *Eliminate non-compliant models before evaluating quality* — a better model you can't legally use is not a candidate.
2. **Capability fit**: Does it clear your task's quality bar? Benchmark candidates on *your* tasks, not leaderboards (the eval discipline from every series). For many tasks, a strong open-weight model (Qwen3, DeepSeek, Llama 4) now matches or beats frontier APIs.
3. **Size/deployment fit**: Does it fit your serving hardware and cost target (inference series Ch 10, 12)? Often the right base is the *smallest* that clears the quality bar (Chapter 3).
4. **Ecosystem**: Framework support, community fine-tunes, tooling, quantization support — a well-supported model is cheaper to build on.
5. **Provenance and trust**: Do you trust the training (safety, contamination, backdoors)? Fully-open models (OLMo) let you audit; open-weight models you take on trust.

## 2.6 Decisions

1. **Default to a hybrid portfolio** — self-host open for private/high-volume, API for frontier-hard, route per request; don't choose one model globally.
2. **Know where your model sits on the openness spectrum** — open-source (reproducible, rare) vs open-weight (downloadable, common) vs restricted vs closed — and pick the least-restrictive that meets your needs.
3. **Read the actual license, not the headline** — prefer Apache 2.0 / MIT for unconditional commercial use; scrutinize custom licenses (Llama's MAU threshold), non-commercial traps, and distillation/output clauses.
4. **Separate the model license from the data license** — both constrain you, and your fine-tuning data has its own copyright/compliance status (Chapter 7).
5. **Select by constraint-filter first, then capability/size/ecosystem** — eliminate legally-unusable models before benchmarking quality, and benchmark on your own tasks.

Next: small and on-device models — the deployment target that's reshaping how models are chosen and trained.
