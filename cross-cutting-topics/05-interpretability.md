# Chapter 5 — Interpretability

## 5.1 Conceptual picture

Interpretability is the effort to understand *what a model is actually doing inside* — which internal features it computes, what its activations represent, why it produced a given output — and, increasingly, to *intervene* on those internals to control behavior. It's cross-cutting because it touches training (what did the model learn, is it contaminated/backdoored), safety (is it about to do something harmful, is its "character" drifting), and serving (can we steer it in real time). For a long time interpretability was purely a research pursuit; the 2026 shift this chapter documents is that **some interpretability techniques have become production-usable** — for monitoring, debugging, and steering deployed models — and every serious team should know which.

The honest framing: interpretability is still immature and mostly *not* a substitute for behavioral evaluation. You cannot yet fully explain a frontier model. But specific tools — probes, sparse autoencoders, steering vectors — now offer real, actionable windows into model internals, and knowing what they can and can't do keeps you from both over-trusting and ignoring them.

## 5.2 The techniques, from cheap to deep

**Probes (linear probes) — the cheap, practical entry point.**
Train a simple linear classifier on the model's internal activations to predict some property (is this true/false, toxic/safe, in-domain/out). If a linear probe can read a property off the activations, the model "knows" it internally. Probes are cheap, fast, and *usable in production* — e.g. a probe that reads "is the model about to refuse / hallucinate / go off-policy" from activations, as a monitoring signal. The workhorse of practical interpretability.

**Sparse autoencoders (SAEs) — the 2026 standard research tool.**
The problem: a model's activations are *dense and polysemantic* — each neuron/direction mixes many concepts, so you can't read them directly. SAEs "decompose dense hidden states into human-interpretable directions," learning an overcomplete sparse basis where individual features correspond to interpretable concepts (a feature for "code that's buggy," "deception," "the Golden Gate Bridge"). SAEs have "become a standard tool for mechanistic interpretability," letting you *see* which concepts a model activates on a given input and *intervene* by turning features up/down. The 2026 frontier extends them to new architectures (diffusion LMs, VLMs — transcoders tracing visual hallucination).

**Steering vectors — intervention.**
Once you have a concept direction (from an SAE, or a difference-of-means between contrasting prompts), you can *add* it to the model's activations at inference to *steer* behavior — increase honesty, reduce a persona trait, correct a reasoning path. "SAE-based steering to correct erroneous reasoning paths in real-time" and "persona vectors to monitor character traits" are documented 2026 production-adjacent applications. Steering is powerful but blunt — over-steering breaks fluency ("steering without breaking" is an active research concern), and denoising concept vectors (with SAEs) improves reliability.

**Attention analysis and weight-based methods.**
Examining attention patterns (what attends to what) and analyzing/orthogonalizing weights (e.g. removing a refusal direction, or a backdoor) round out the toolkit — more surgical, more research-y.

## 5.3 What's actually usable in production

The practical filter — what a deployment team can use *today*:

- **Probes for monitoring** — read safety-relevant properties (hallucination risk, refusal, off-policy behavior, PII presence) off activations as a real-time signal feeding guardrails (Chapter 6). Cheap and deployable now.
- **Persona/character monitoring** — "persona vectors" track whether a model's character is drifting (sycophancy, a trait you trained out creeping back) in production. Ties to the post-training series' character-training (Ch 11).
- **SAE-based steering for correction** — nudging a model away from a known failure mode (a bad reasoning pattern, an unwanted trait) via feature steering, where behavioral fixes (retraining) are too slow.
- **Debugging training artifacts** — using SAEs/probes to investigate *why* a fine-tune regressed, whether data contamination created a spurious feature, or whether a backdoor exists.

What's *not* yet production-ready: full mechanistic explanation of frontier behavior, guaranteed detection of deception/misalignment, or replacing behavioral evals. Treat interpretability as a *complement* to evaluation and guardrails, not a replacement.

## 5.4 Interpretability for safety and trust

The strategic reason to invest: as models are given more autonomy (agents, tool use — post-training Ch 9, security Ch 6), *behavioral* testing alone becomes insufficient — you can't test every situation, and a model can behave well in tests while harboring a failure mode. Interpretability offers a *second channel*: instead of only asking "did it act right," ask "what was it computing." A model that gives the right answer for the wrong internal reason (reasoning over a hallucination, using a spurious feature, activating a deception concept) is caught by interpretability where behavior passes. This is why frontier labs invest heavily and why interpretability-based monitoring is moving into high-assurance production.

## 5.5 A caution on over-claiming

Interpretability findings are seductive and easy to over-interpret. A feature that *looks* like "deception" may fire on unrelated inputs; a probe that predicts a property in-distribution may fail out-of-distribution; steering that fixes one behavior may break others. The discipline (echoing the whole knowledge base): **validate interpretability claims with falsification** — the 2026 research move toward "concept extraction and explanation verification through falsification frameworks." Don't trust an interpretation until you've tried to break it. An unfalsified interpretability story is a hypothesis, not a finding.

## 5.6 Decisions

1. **Use linear probes for production monitoring** — cheap, deployable signals for hallucination risk, refusal, PII, off-policy behavior, feeding guardrails.
2. **Use SAEs to understand and debug** — decompose activations into interpretable features to investigate training regressions, contamination, and failure modes.
3. **Use steering vectors (SAE-derived, denoised) for real-time correction** where retraining is too slow — carefully, watching for fluency breakage from over-steering.
4. **Monitor persona/character drift** in production (persona vectors) — ties to character training; catches sycophancy and trait regression.
5. **Treat interpretability as a complement to evals/guardrails, not a replacement, and falsify every interpretation** before trusting it — it's a powerful second channel, not a guarantee.

Next: production security — the adversary's-eye view of your deployed model.
