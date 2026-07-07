# Appendix A — Knowledge Editing and Unlearning

*A companion to Chapters 3 and 8. Continued pretraining adds knowledge in bulk; this appendix covers the two surgical inverses — **editing** (change one fact) and **unlearning** (remove specific knowledge) — which sound like small versions of CPT and are actually much harder. Read this before promising anyone that a fact can be "fixed" or a datum "deleted" from a trained model.*

## A.1 The problem: knowledge lives in weights, not rows

A trained model stores knowledge diffusely across its parameters — memorized, not filed. There is no row to update when the CEO changes, and no row to delete when a GDPR right-to-be-forgotten request arrives (cross-cutting Ch 7). The honest baseline for both operations is *remove/fix the data and retrain*, which "is not practical for LLMs as their training may take months." Everything in this appendix is an attempt to approximate that baseline cheaply — and the headline is that the approximations are real but brittle, so the *architectural* answer (keep volatile and deletable knowledge out of weights in the first place) usually beats the *surgical* one.

## A.2 Knowledge editing: locate-and-edit

**The methods.** ROME (Rank-One Model Editing) treats a fact as an association stored in specific MLP layers, locates the responsible weights via causal tracing, and applies a rank-one update that rewrites the association ("the Eiffel Tower is in *Rome*" — hence the name). MEMIT scales the idea to thousands of edits across multiple layers; AlphaEdit and successors refine the update math to reduce collateral damage. Editing is fast (seconds per fact, no retraining) and precise in the best case.

**The limitations, which are the real content here:**
- **Edits are shallow.** The edited fact often holds only in phrasings close to the edit; paraphrases, reversals ("who is the CEO of X" vs "X's CEO is who?"), and multi-hop uses of the fact frequently still produce the old answer. The model's *reasoning with* the fact is not reliably updated — only the direct association.
- **Edits ripple.** Rank-one surgery on shared weights perturbs neighboring knowledge; batch editing (MEMIT-scale) accumulates degradation, and general capability measurably erodes as edit counts grow. The forgetting budget of Chapter 8 applies to edits too.
- **Edits don't compose with later training.** Subsequent fine-tuning or CPT can silently revert or scramble edits.

**Practical position (2026):** editing is a research tool and an emergency patch, not a maintenance strategy. For keeping a model current, the durable answers are the ones this series already covered — **RAG for volatile facts** (don't put them in weights at all — Chapter 3.2's decision rule) and **CPT with replay for bulk refresh** (Chapter 3). Reach for a locate-and-edit method only for a small number of point fixes where redeployment is impossible, and eval the edited model on the full Chapter 10 harness before trusting it.

## A.3 Machine unlearning

**The goal.** Remove the influence of specific training data — a person's PII, copyrighted text, harmful capability — so the model behaves *as if it had never seen it*. Driven by GDPR's right to be forgotten, copyright exposure, and safety (cross-cutting Ch 7); unlike differential privacy (which bounds influence but "cannot fully guarantee the right to be forgotten"), unlearning targets elimination after the fact.

**The methods, roughly in order of use:** gradient ascent on the forget set (push the model away from the data, cheap and blunt — over-applied, it lobotomizes); preference/relabeling approaches (train the model to respond as if ignorant); regularized variants (marginal-information regularization and kin, constraining damage to retained knowledge); and — a notable 2025–2026 finding — **knowledge-editing methods repurposed as unlearning** (ROME/MEMIT with merged queries perform competitively with dedicated unlearning methods). Every method fights the same two-front war: forget *thoroughly* (the knowledge shouldn't resurface under paraphrase, jailbreak, or fine-tuning) while forgetting *narrowly* (retained capability intact — the Chapter 8 trade again, inverted).

**The limitations:**
- **Verification is unsolved.** Proving the knowledge is *gone* — not just suppressed — is the hard part; "unlearned" facts are routinely recovered by adversarial prompting or a few steps of fine-tuning. Suppression that survives your test set is not erasure.
- **You must know what to forget.** Right-to-be-forgotten requests name a person, not the individual–fact associations the model actually stores; identifying what a model knows about an individual is itself an open problem (the 2025 quantifying-personal-data line of work).
- **Cost scales badly with scope.** Unlearning one document is feasible; unlearning a person's diffuse presence across a web-scale corpus approaches retraining.

**Practical position (2026):** treat unlearning as **immature and unauditable** for compliance purposes. The load-bearing defenses are *upstream*: PII scrubbing and deduplication before training, provenance tracking so you know which models a dataset touched, and keeping deletable personal data in **retrieval stores rather than weights** — exactly the cross-cutting Ch 7 guidance, which this appendix now grounds: the reason "keep it in RAG" is the standard advice is that deletion from an index is real and deletion from weights is, today, best-effort.

## A.4 Decisions

1. **Architect so you rarely need either operation** — volatile facts in RAG, deletable personal data in retrieval stores, provenance tracked so affected models are known. Surgery is the fallback, not the plan.
2. **Use locate-and-edit (ROME/MEMIT-class) only for small, urgent point fixes** where redeployment is impossible; expect paraphrase/multi-hop leakage and ripple damage, run the full eval harness after, and re-apply after any subsequent training.
3. **Maintain currency with the series' main tools** — CPT + replay for bulk knowledge refresh, annealing-mixture inclusion when you control the run — not with accumulated edits.
4. **Treat unlearning as best-effort suppression, not certified erasure** — verify adversarially (paraphrase, jailbreak, fine-tuning probes), and don't promise regulators weight-level deletion the field can't yet audit.
5. **Watch the editing-as-unlearning convergence** — the two literatures merged in 2025–2026, and a single mature toolkit may emerge; today, both remain research-grade.
