# Chapter 6 — Tokenizer and Vocabulary Surgery

## 6.1 Conceptual picture

The tokenizer is the one pretraining decision the pretraining series called "irreversible," and mostly it is. But there is a class of midtraining interventions that *do* modify it: adding vocabulary for a new language or domain, replacing a badly-fitting tokenizer, or pruning an oversized one. These are surgery — high-risk edits to a load-bearing part — and this chapter is about when the operation is worth it and how not to kill the patient.

The reason anyone touches the tokenizer: **fertility.** Fertility is tokens-per-word (or per-character) for your target text. A tokenizer optimized for English produces catastrophically long sequences for other languages and scripts — up to **13× longer** for some non-English text, and badly for domains full of novel strings (code identifiers, chemical formulas, gene names, log lines). Long sequences mean: higher inference cost (you pay per token), shorter effective context (the window fills with fragments), and worse modeling (the model spends capacity reassembling words from sub-word rubble). If your target domain or language tokenizes at 2–3× the fertility of English, the tokenizer is a real tax and surgery may pay for itself.

## 6.2 The decision: should you touch it at all?

Touching the tokenizer is expensive and dangerous — new embeddings start untrained, and getting them wrong forgets or destabilizes the model. Default to **not** touching it. Touch it only when *all* of these hold:

- Fertility on your target text is genuinely bad (say >1.5–2× a reasonable baseline).
- The domain/language is a *large fraction* of your deployment traffic (the inference savings are real and recurring).
- You are already doing substantial CPT anyway (the new embeddings need billions of tokens to train in — you get them "for free" if you were continuing pretraining regardless).

If you only need a bit of a new language and traffic is light, **skip it** — live with the fertility, or use the base tokenizer's byte-fallback. The most common tokenizer-surgery mistake is doing it when CPT alone would have sufficed, and eating months of instability for a few percent of sequence length.

## 6.3 Vocabulary extension: the mechanics

The standard, lowest-risk operation is *adding* tokens (not replacing the tokenizer):

1. **Train a new tokenizer on your target corpus**, then take the tokens that are *not* already in the base vocabulary (the non-overlapping ones). Add a bounded set — commonly a few thousand to ~30K new tokens; adding too many dilutes the embedding table and wastes softmax compute.
2. **Resize the embedding and un-embedding (LM head) matrices** to fit the new vocabulary size.
3. **Initialize the new embeddings well** — this is the crux (6.4).
4. **Continue-pretrain** so the new embeddings train in. Optionally warm-start by freezing the transformer and training *only* the new embeddings (and LM head rows) at a high LR for a short phase, then unfreeze for full CPT. This lets the new tokens "find their place" before you perturb the whole model.

## 6.4 Embedding initialization — where it lives or dies

New tokens with random embeddings are pure noise injected into a trained model; they cause a loss spike and slow, forgetting-prone convergence. Never random-init. The methods, in ascending sophistication:

- **Mean-of-constituents (the baseline you should always at least do).** Initialize a new token's embedding as the *mean of the embeddings of the sub-word tokens the base tokenizer would have split it into.* A new token "tokenization" gets the average of `token`, `ization`, etc. Cheap, robust, and captures most of the value — a new token starts life meaning roughly what its pieces meant.
- **Subword-interpolation / weighted** (FOCUS, EEVE-style) — weight the constituents by similarity/frequency rather than a flat mean. A modest improvement over the mean.
- **Learned / hypernetwork init** (HYPEROFA, Learned Embedding Propagation) — a small network predicts good embeddings for new tokens from multilingual signals. Best convergence, most machinery; worth it only for large multilingual expansions (100s+ languages, à la Omnilingual-scale work).

Whatever you choose, **initialize the LM head (un-embedding) rows too**, symmetrically — an untrained output projection for the new tokens is as damaging as an untrained input embedding.

## 6.5 Replace or prune, not just extend

Two rarer operations:

- **Full replacement** — swap in a better-fit tokenizer entirely (e.g. a domain- or script-specialized one like an Indic-capable drop-in). This orphans *every* embedding, not just new ones, so it needs substantial CPT to re-anchor — closer to re-pretraining the embedding layer. Justified only when the base tokenizer is a severe, pervasive mismatch (e.g. adapting a heavily English model to a non-Latin script as the primary language). Cross-tokenizer transfer methods can init the new full vocabulary from the old one to soften the blow.
- **Pruning** — *removing* rarely-used tokens to shrink the embedding table and softmax (useful for a domain-specialized deployment that will never see most of the base vocabulary). Cuts parameters and speeds inference; low-risk since you are removing, not adding, but only worth it when the deployment is genuinely narrow.

## 6.6 The verification ritual

Because tokenizer bugs are silent and catastrophic, before any real run:

- **Round-trip test**: encode then decode a sample of target text; confirm byte-exact reconstruction. Broken merges corrupt data invisibly.
- **Fertility measurement**: compute tokens-per-character on held-out target text, before and after — confirm the surgery actually bought the reduction you did it for.
- **Special-token audit**: ensure your new tokens don't collide with reserved/special tokens (role markers, `<think>`, EOS) — a collision is a downstream disaster.
- **Decode a few training batches** (the same ritual as the pretraining/SFT series): look at real tokenized examples with the new vocabulary and confirm they read correctly.
- **Freeze it early and forever.** As with chat templates, a tokenizer change after you have generated data or trained downstream invalidates everything. The tokenizer is the contract; sign it once.

## 6.7 Decisions

1. **Default to not touching the tokenizer.** Touch it only when fertility is genuinely bad *and* the target is a large traffic fraction *and* you're doing CPT anyway.
2. **Prefer extension (add tokens) over replacement**; prune only for narrow deployments.
3. **Bound new vocabulary** to a few thousand–30K tokens; more dilutes the table.
4. **Never random-init**; use mean-of-constituents at minimum, learned init for large multilingual expansions. Initialize the LM head symmetrically.
5. **Optionally warm-start** by training only new embeddings first, then full CPT.
6. **Run the verification ritual** (round-trip, fertility, special-token audit, decode batches) and freeze the tokenizer permanently.

Next: capability and reasoning injection — using midtraining's self-supervised span to install the reasoning and instruction priors that used to be post-training's exclusive job.
