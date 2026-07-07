# Chapter 3 — Continued Pretraining

## 3.1 Conceptual picture

Continued pretraining (CPT, also "continual pretraining") is taking an existing base model and running *more* self-supervised next-token prediction on it — usually on a corpus that differs from the original pretraining data (a domain, a language, more recent data, more of a capability). It is the same objective as pretraining, at pretraining scale relative to fine-tuning (billions of tokens, not thousands of examples), on a model that already works.

The distinction that trips people up: **CPT vs midtraining vs fine-tuning.** They form a spectrum by data scale and intent.
- *Fine-tuning (SFT)* — thousands to millions of *labeled* (prompt, response) examples, loss masked to responses, teaches behavior and format.
- *Midtraining annealing* — the decay stage of your own run, on a curated *general* quality mixture (Chapter 2).
- *CPT* — billions of *unlabeled* tokens on a *shifted* distribution, teaches knowledge and domain fluency, full next-token loss on everything.

CPT is the right tool when the thing you want the model to gain is **knowledge or fluency it cannot absorb from a handful of demonstrations** — a whole programming language, a medical literature, a legal corpus, a low-resource human language, or simply the world after the base model's knowledge cutoff. If you can *demonstrate* the target behavior in a few thousand examples, SFT is cheaper and forgets less. If the target is a large body of tacit knowledge, only CPT's scale reaches it.

## 3.2 When CPT beats the alternatives

Decision rule, in order of increasing commitment:

1. **RAG / long context first.** If the knowledge is retrievable and fits in context, do not train at all — retrieve it. CPT is for knowledge you need *internalized* (fluency, reasoning over the domain, latency-sensitive recall), not for facts you can look up.
2. **SFT if you can demonstrate it.** Narrow behavior, format, style, a bounded skill — SFT on 10³–10⁵ examples. Forgets less, costs dollars.
3. **CPT if the target is a corpus, not a behavior.** New language, new domain vocabulary and reasoning patterns, post-cutoff knowledge, a capability requiring billions of tokens of exposure. The canonical evidence: Code Llama continued Llama 2 on **500B code tokens** and the 7B matched vanilla Llama 2 70B on code — a capability no SFT could have installed.

Common real pairing: **CPT then SFT.** SaulLM continued Mistral-7B on 30B legal tokens (CPT), *then* instruction-tuned on legal instructions (SFT). The CPT installs the domain; the SFT installs the behavior. This two-phase shape is standard for serious domain models.

## 3.3 The re-warmup problem

Here is the first place CPT bites. Your base model finished its run at near-zero LR (it was annealed). To make progress on new data you must raise the LR again — but raising it too high, too fast, re-shocks the model and induces a **loss spike** and a burst of forgetting. Too low and you barely learn. The literature converges on:

- **Re-warm to a fraction of the original peak LR**, not to the original peak. Typical: re-warm to **~1e-4 to 3e-4 for a 7B** (roughly 1/3 to 1/10 of a from-scratch peak), over a short warmup (a few hundred to low thousands of steps), then decay again (WSD or cosine) over the CPT run.
- **The more your target distribution differs from the original, the higher the LR you can justify** (there is more to learn) — but also the more forgetting you risk, so replay (Chapter 8) rises with LR.
- **Match the schedule shape to the token budget.** For a 5–50B-token CPT run, a short re-warmup + linear/cosine decay to near-zero is standard. End near zero so the CPT run has its own annealing tail on the domain data.

## 3.4 The stability gap

A subtle, well-documented CPT phenomenon (Guo et al. and others): when you start CPT, performance on the *target* domain often **gets briefly worse before it gets better** — a dip in the first fraction of training as the model destabilizes from its old optimum before re-converging toward the new distribution. Meanwhile general performance drops. Three mitigations, all cheap:

1. **Warm up the LR gently** — a sharp LR jump deepens the gap.
2. **Use a higher replay ratio early**, tapering it — keeps the model anchored while it re-stabilizes.
3. **Multiple smaller CPT epochs over a subset** can reach target performance with fewer total tokens than one pass over everything, by keeping the model near a good optimum (the "efficient continual pretraining" result). Counterintuitive but robust for small domain corpora.

The practical consequence: **do not judge a CPT run by its first checkpoints.** Eval at several points; the domain metric commonly dips then climbs past baseline.

## 3.5 Data scale and epochs

How many tokens? Anchored by published runs:

| Target | Tokens (CPT) | Notes |
|---|---|---|
| Domain flavor / recency | 1–10B | LoRA-viable; days on 4–8 H100 |
| Serious domain competence | 20–50B | SaulLM legal: 30B; medical HEAL: ~15B |
| New capability (e.g. code) | 100–500B | Code Llama: 500B; effectively a second pretraining |
| New language | 10–100B | 10–30% replay preserves source language |

**Epochs and repetition.** For small high-quality domain corpora, 2–4 epochs is fine and often better than one pass (stability-gap result). Past ~4 epochs on the same tokens you risk memorization and forgetting with little gain — the pretraining series' data-repetition ceiling applies. If your domain corpus is small, augment with **reading-comprehension conversion** (Chapter 4) or synthetic expansion rather than looping raw text many times.

## 3.6 CPT vs the "finetuner's fallacy"

A 2026 result worth internalizing: sometimes the right move is not to CPT the finished base model at all, but to have included the target data in pretraining (or midtraining) in the first place — "the finetuner's fallacy" is assuming late adaptation recovers what early inclusion would have given. You usually cannot rewind pretraining, so this is mostly a *planning* lesson: **if you know your domain in advance and you are training the base model yourself, weave domain data into the annealing mixture (Chapter 2) rather than bolting on a separate CPT stage later.** CPT is the tool when you inherited a base model or learned the target after the fact.

## 3.7 Decisions

1. **RAG → SFT → CPT** in ascending order of commitment; only CPT when the target is a corpus of tacit knowledge, not a demonstrable behavior.
2. **Re-warm to ~1/3–1/10 of from-scratch peak LR**, short warmup, decay to near-zero.
3. **Expect and plan for the stability gap** — gentle warmup, front-loaded replay, eval at multiple checkpoints.
4. **Budget tokens by ambition:** ~5B for flavor, ~30B for domain competence, ~500B for a new capability.
5. **2–4 epochs on small corpora; augment rather than loop** past that.
6. **If you own the base model and know the domain, prefer early inclusion** (annealing mixture) over a bolt-on CPT stage.

Next: domain adaptation makes this concrete with code/medical/legal case studies and the specialization-vs-size tradeoff.
