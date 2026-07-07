# Chapter 8 — Team, Compute, and the Improvement Loop

## 8.1 Conceptual picture

Every technical decision in this knowledge base is made by an *organization* spending a *compute budget*, running an *improvement loop*. This final substantive chapter is about that human-and-resource system — who does the work, how compute is allocated, and how a deployed model gets *better over time*. It's the most-neglected cross-cutting concern because it's not technical, yet it determines whether the technical work compounds or thrashes. The through-line (Chapter 1.4): **the loop matters more than the launch** — the teams that win optimize how fast they can measure a problem and ship a fix, not just the quality of the initial model.

## 8.2 Roles and org structure

Building and running LLM systems needs a spread of skills that rarely live in one person. The core roles (a small team wears several hats; a large one specializes):

- **Data engineering / curation** — sourcing, cleaning, deduplicating, mixing, and documenting data. The most underrated role, because data is the dominant quality lever across every series (pretraining data, midtraining mixtures, post-training demonstrations, multimodal curation, embedding pairs). Under-invest here and everything downstream suffers.
- **Training / research** — running the training stages, the scaling ladder, hyperparameters, architecture. The glamorous role, but a smaller share of the work than newcomers expect.
- **Inference / platform engineering** — serving, autoscaling, cost, the production stack (inference series). Where two-thirds of the compute and much of the ongoing cost live.
- **Evaluation** — building and maintaining evals, the honesty-check on every proxy. Deserves a dedicated owner; when eval is everyone's part-time job it's no one's.
- **Safety / security / compliance** — red-teaming, guardrails, governance (Chapters 6–7). Increasingly non-optional and increasingly its own function.
- **Product / domain experts** — who define what "good" means for the actual use case and supply domain judgment for data and evals.

The organizing insight: **data and evaluation are the roles most often under-staffed and most determinative of success.** The instinct is to hire researchers; the leverage is often in a great data-curation and eval capability. Structure the team so data and eval have clear owners, not just the training and serving that feel like "the real work."

## 8.3 Compute budgeting

Compute is the dominant cost and a scarce, contended resource. Allocating it well:

- **Know the split.** Inference is ~two-thirds of AI compute in 2026 and growing (inference series Ch 1); training is the visible cost but often the smaller ongoing one. Budget for the *serving* lifetime, not just the training run.
- **Spend on experiments, not just the big run.** The training series' discipline: most compute value comes from *de-risking* the expensive run via small-scale experiments (the scaling ladder, microanneals, ablations). A budget that funds only the flagship run and no experiments wastes the flagship run. Allocate a meaningful fraction to small, fast, parallel experiments.
- **Right-size ruthlessly** (Chapters 2–3). The biggest compute waste is running a bigger model than the task needs. Distillation, small models, and task-appropriate routing cut compute more than any efficiency tweak.
- **Buy/rent/serverless by load shape** (inference Ch 10, 12) — reserved for steady base load, on-demand for bursts, serverless/API for spiky or low volume. Match the procurement to the utilization or bleed money on idle GPUs.
- **The FinOps loop** (inference Ch 12) — continuously measure cost per token/task, attribute it, optimize the biggest line item. Compute budgeting is a loop, not a one-time plan.

## 8.4 The data flywheel

The mechanism by which deployed models improve: **production generates data, data improves the model, the better model generates better production data.** This flywheel is the central engine of a maturing LLM product, and building it is often more valuable than any single training improvement:

1. **Collect** — log production interactions (inputs, outputs, user feedback, thumbs, corrections, tool results, failures). This is proprietary data no competitor has.
2. **Mine** — find the failures, the edge cases, the high-value examples; identify where the model is weak from *real* usage, not guessed distributions.
3. **Curate** — turn production signal into training data: failures into new SFT/preference examples, corrections into demonstrations, common queries into eval cases. (Mind privacy/consent — Chapter 7 — production data is often personal data.)
4. **Improve** — fine-tune / post-train / update retrieval on the curated data.
5. **Redeploy and measure** — ship, and confirm the improvement on the very failures that motivated it.

The teams that win build this loop *tightly* — a short path from "a user hit a problem" to "the fix is deployed and verified." The flywheel compounds: each turn produces better data, which produces a better model, which attracts more usage, which produces more data.

## 8.5 Eval-driven development

The discipline that makes the flywheel (and all iteration) trustworthy, and the single most repeated lesson across this entire knowledge base: **your evaluation suite is the thing you actually optimize, so invest in it above almost everything else.** Concretely:

- **Evals are the product spec.** "Good" is whatever your evals measure; if they measure the wrong thing, you'll optimize the wrong thing perfectly. Build evals that reflect real use, including the failures the flywheel surfaces.
- **Hold out honest evals.** A private eval the training loop never sees is the defense against every proxy-gaming failure (contamination, reward hacking, overfitting) across all series. When public metrics and the private eval disagree, trust the private one.
- **Measure the trade, not just the target.** The paired-eval discipline (midtraining forgetting, multimodal hallucination, quantization quality) generalizes: every change is a trade, and you must measure what you might be *losing*, not just what you're gaining.
- **Fast eval = fast iteration.** The flywheel's speed is bounded by how fast you can measure whether a change helped. Cheap, fast, automated evals are what let a team iterate hundreds of times (the post-training economics); slow evals throttle the whole loop.

Eval-driven development is the organizational expression of the knowledge base's deepest theme: **everything is a proxy, every proxy is gamed, and keeping proxies honest is the real job.**

## 8.6 What to prioritize

For a team starting out, the priority order that follows from the whole knowledge base:

1. **Evaluation and data capability first** — the two most-determinative, most-under-staffed functions. Without them, nothing else compounds.
2. **The improvement loop** — the collect→curate→improve→measure flywheel, built tight, beats any one-time model win.
3. **Constraints early** (Chapters 2, 6, 7) — licensing, security, compliance established before they invalidate work.
4. **Right-sizing and hybrid routing** (Chapters 2–3) — the biggest cost/latency lever.
5. **The flashy stuff last** — the biggest model, the fanciest technique. It's the smallest share of what makes a product succeed.

## 8.7 Decisions

1. **Staff data curation and evaluation as first-class, owned functions** — they're the most determinative and most under-staffed roles; don't let them be everyone's part-time job.
2. **Budget compute for the serving lifetime and for experiments**, not just the flagship run — and right-size ruthlessly (the biggest waste is an oversized model).
3. **Build the data flywheel tightly** — a short path from production failure to deployed, verified fix; production data is your compounding proprietary advantage (mind privacy, Ch 7).
4. **Practice eval-driven development** — evals are your product spec and the thing you actually optimize; hold out honest private evals, measure the trade not just the target, and keep evals fast to keep iteration fast.
5. **Prioritize eval/data/loop/constraints over the flashy model work** — the order that makes technical excellence compound into a shippable, defensible product.

Next: the reading list and references.
