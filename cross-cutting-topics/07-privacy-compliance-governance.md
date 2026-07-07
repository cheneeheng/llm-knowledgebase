# Chapter 7 — Privacy, Compliance, and Governance

## 7.1 Conceptual picture

This is the chapter most engineers want to skip and most often regret skipping. Legal and regulatory constraints — data privacy (GDPR), AI-specific regulation (the EU AI Act), copyright, and data governance — are cross-cutting constraints (Chapter 1.2) that reach *backwards* from your deployed product into the data you collect and train on, and getting them wrong doesn't degrade your model, it makes it *illegal to ship* (or exposes you to fines up to 7% of global revenue under the AI Act, 4% under GDPR). This chapter is the practitioner's map of what applies, what it requires, and — crucially — why you must handle it *before* training, not before launch.

The framing that makes this tractable for a technical reader: compliance is mostly about **provenance and documentation.** Regulators increasingly don't just ask "is your model safe," they ask "can you *show* where your training data came from, what legal basis you had to use it, and how you documented the whole pipeline." Data lineage — knowing and recording the origin, rights, and transformations of every dataset — is the technical practice underneath most of the legal requirements.

## 7.2 The regulatory stack (2026)

An AI product operates "inside an overlapping regulatory stack," not under one law. The load-bearing pieces:

**GDPR (and global equivalents — CCPA, etc.).** Governs *personal data*. Applies whenever your model trains on or processes data about identifiable people. Core obligations: lawful basis for processing, data minimization, transparency, individual rights (access, deletion — which is genuinely hard once data is baked into weights), and purpose limitation. You're typically a *data processor* for your clients under GDPR while being an AI *provider* under the AI Act — both hats at once.

**The EU AI Act — the big 2026 addition.** Risk-tiered regulation of AI systems, phasing in:
- **General-Purpose AI (GPAI) obligations** became applicable **August 2, 2025** — foundation-model providers must meet **training-data transparency and copyright-policy requirements** (document what you trained on, have a copyright-compliance policy). If you train a foundation model, this is you *now*.
- **High-risk systems** (Annex III — recruitment, credit scoring, law enforcement, etc.) face the heaviest obligations; after the May 2026 Omnibus agreement, the main high-risk deadline was deferred to **December 2, 2027**, but the direction is set.
- **Article 10** requires high-risk providers to document training/validation/test data's "origin, relevance, representativeness and potential biases" — i.e. **data lineage as a legal requirement.**

**Copyright.** Whether training on copyrighted data is permitted is contested and jurisdiction-dependent, and the AI Act now requires a copyright policy and training-data transparency for GPAI. Your training data's copyright status is a real constraint (and your *base model's* training data may carry its own exposure — Chapter 2.4).

**Sector and regional rules** — DSA, export controls, sector-specific (health/finance), content-provenance mandates. The stack is deep; know which apply to *your* domain and geography.

## 7.3 Data governance: the technical practice

The engineering work that satisfies most of the above is **data lineage as a first-class asset**: "every dataset carrying metadata about source, legal basis, consent status, retention period, transformations, and links to the models that consume it." Concretely:

- **Track provenance** — for every training/fine-tuning dataset, record where it came from, under what license/consent, and its copyright status. You must be able to answer "why were you allowed to use this?" for any data in your model.
- **Link data to models** — know which datasets went into which model version, so that if a dataset is found non-compliant (a copyright claim, a consent withdrawal), you know which models are affected.
- **Document the pipeline** — the transformations, filtering, and decisions (the same documentation the training series recommended for reproducibility now doubles as compliance evidence).
- **Retention and deletion** — track retention periods and handle deletion requests. The hard problem: deleting personal data from *trained weights* is not straightforward (it's memorized, not stored in a row you can drop); mitigations include not training on data you may need to delete, machine-unlearning research (immature), and keeping deletable data in *retrieval* (RAG, Chapter 4) rather than weights.

This is why compliance is a *before-training* concern: you cannot reconstruct provenance you didn't record, and you cannot un-train data you shouldn't have used. Build the lineage system before the training run, not before the audit.

## 7.4 PII and privacy in practice

Beyond documentation, the operational privacy work:

- **Scrub PII from training data** — filter personal data out of pretraining/fine-tuning corpora (unless you have a lawful basis and need it), reducing both compliance exposure and memorization/leakage risk.
- **Prevent memorization/leakage** — models can regurgitate training data verbatim; this is a privacy *and* security issue (a model trained on customer data can leak it to other users). Deduplication (training series) reduces memorization; test for it.
- **Keep sensitive data out of third-party APIs** — the privacy driver for self-hosting (Chapters 2, 3): if data can't legally leave your environment, you can't send it to a hosted API, which forces the build side of build-vs-buy. This is one of the strongest reasons to self-host a small/open model.
- **Privacy-preserving techniques** — differential privacy in training (bounds memorization at a quality cost), federated approaches, and PII-detection guardrails (Chapters 5–6) at inference. Used where the sensitivity justifies the overhead.

## 7.5 Documentation and accountability

Regulators and enterprise customers increasingly require *artifacts*:

- **Model cards / documentation** — what the model is, its intended use, limitations, training data summary, evaluation results, known biases. The AI Act formalizes much of this for high-risk systems.
- **Data documentation** (datasheets) — the lineage above, presented.
- **Risk assessments** — for high-risk uses, documented analysis of risks and mitigations.
- **Audit trails** — who did what, which model version served which decision (ties to the inference series' versioning, Ch 11).

Treat documentation as a deliverable produced *alongside* the work, not reconstructed before an audit — the same discipline as the training series' reproducibility logs, now legally load-bearing.

## 7.6 Decisions

1. **Establish compliance constraints before training, not before launch** — provenance you didn't record can't be reconstructed, and data you shouldn't have trained on can't be un-trained.
2. **Build data lineage as a first-class asset** — track origin, legal basis, consent, retention, and dataset→model links for every dataset; this satisfies most of GDPR Article 10 / AI Act requirements.
3. **Know which of the regulatory stack applies** to your domain and geography — GPAI training-data transparency and copyright policy are live *now* if you train foundation models; high-risk obligations are coming.
4. **Handle PII deliberately** — scrub it from training data, prevent memorization/leakage, and keep un-exportable data out of third-party APIs (a driver to self-host) and in retrieval rather than weights where deletion may be required.
5. **Produce documentation (model cards, datasheets, risk assessments, audit trails) alongside the work** — as a deliverable, not an audit-eve reconstruction.

Next: team, compute, and the improvement loop — the organizational system that produces all of the above.
