# Cross-Cutting Topics: The Concerns That Span the Whole Lifecycle

*Model selection and licensing, small and on-device models, embeddings and retrieval, interpretability, production security, privacy and compliance, and the org/compute/improvement loop — the topics that don't belong to one training stage because they touch all of them.*

*Compiled July 2026. The capstone to the [pretraining](../pretraining-from-scratch/README.md), [midtraining](../midtraining-and-continued-pretraining/README.md), [post-training](../post-training-from-scratch/README.md), [inference](../inference-and-serving/README.md), and [multimodal](../multimodal-training/README.md) series. Assumes familiarity with those — this series is the connective tissue between them, not a from-scratch introduction.*

---

## Why this series exists

The other series each own a stage of the LLM lifecycle. This one owns the concerns that refuse to stay in a single stage. Whether to build a model or buy an API shapes every downstream decision and belongs to no one stage. A model's *license* is decided at selection but governs deployment, distillation, and revenue. *Interpretability* touches training (understanding what the model learned), safety (monitoring behavior), and serving (real-time steering). *Security* and *compliance* are properties of the deployed system that constrain data collection, training data, and serving. *Embeddings and retrieval* are the models you train alongside your LLM to make RAG work. *Small and on-device models* are a deployment target that reaches back and changes how you train and distill. And the *organization, compute budget, and improvement loop* wrap the entire enterprise.

These topics are where teams that understand each individual stage still fail — because they optimized a stage in isolation and missed a constraint that lived between stages. This series is the map of those between-stage constraints.

## How to read this report

Every chapter is **layered**, like the companion series: conceptual picture first, then the practitioner layer — concrete decisions, numbers, and tradeoffs. Because these topics are broad, the chapters are more self-contained than in the stage-specific series; read the one you need. But the first two chapters — the cross-cutting mental models and the build-vs-buy/licensing decision — frame everything else and are worth reading first regardless.

A recurring theme: **the hardest LLM decisions are not technical, they are decisions about constraints that span stages** — legal, economic, organizational, and safety constraints that no amount of training-stage excellence resolves. The teams that ship reliable, defensible, affordable AI are the ones that treat these cross-cutting concerns as first-class from day one, not as afterthoughts bolted on before launch.

## Chapters

| # | File | Contents |
|---|---|---|
| 1 | [The Big Picture](01-big-picture.md) | What "cross-cutting" means, the constraints that span stages, the core mental models |
| 2 | [Model Selection and Licensing](02-model-selection-and-licensing.md) | Build vs buy, open-weight vs open-source, the 2026 license landscape, commercial-use traps, the hybrid default |
| 3 | [Small and On-Device Models](03-small-and-on-device-models.md) | Why small won, distillation as the path, edge quantization, phone/embedded deployment, the specialization frontier |
| 4 | [Embeddings and Retrieval Models](04-embeddings-and-retrieval.md) | Training embedding models, contrastive learning, rerankers, Matryoshka, RAG stacks, evaluation |
| 5 | [Interpretability](05-interpretability.md) | Probes, sparse autoencoders, steering vectors, persona/concept monitoring, what's usable in production |
| 6 | [Production Security](06-production-security.md) | Prompt injection, jailbreaks, agentic exploitation, guardrails, red-teaming, the defense stack |
| 7 | [Privacy, Compliance, and Governance](07-privacy-compliance-governance.md) | GDPR + EU AI Act, training-data provenance and copyright, data lineage, documentation, PII |
| 8 | [Team, Compute, and the Improvement Loop](08-team-compute-and-improvement-loop.md) | Roles and org structure, compute budgeting, the data flywheel, eval-driven development, what to prioritize |
| 9 | [Reading List and References](09-reading-list-and-references.md) | Curated sources by topic with provenance notes |

## Scope

This series covers the concerns that touch multiple stages and the deployment-adjacent model types (embeddings, small/edge models) that the stage-specific series don't own. It deliberately does *not* re-cover material owned elsewhere: training safety/alignment lives in the post-training series (Chapter 11), serving economics in the inference series (Chapter 12), and multimodal in its own series. Where a topic here overlaps one there, this series takes the *cross-stage* view and points to the deep treatment.

## The through-line

Cross-cutting concerns reward **deciding constraints before capabilities**. The instinct is to build the best model first and handle licensing, security, privacy, and cost later — and it is always a mistake, because each of those is a *constraint* that, discovered late, invalidates work already done: a license that forbids your use case, a compliance rule that bans your training data, a security hole that forces a redesign, a cost structure that makes the whole thing unviable. The teams that win establish the constraints first — what can we legally use, what must we protect, what can we afford, who does the work — and then build the best model *within* them. Constraints first, capabilities second; the reverse order is how good models become unshippable products.
