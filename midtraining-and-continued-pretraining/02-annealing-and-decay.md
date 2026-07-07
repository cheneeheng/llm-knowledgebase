# Chapter 2 — Annealing and the Decay Phase

## 2.1 Conceptual picture

Annealing is the practice of ending a training run (or a training stage) by decaying the learning rate toward zero while simultaneously shifting the data mixture toward your highest-quality, most capability-dense sources. The two moves are separate but almost always done together, and the reason they belong together is the central insight of this chapter: **as the learning rate falls, the model stops exploring and starts settling, and whatever it is looking at as it settles is what it becomes.**

During the high-LR stable phase the model is bouncing around a wide region of the loss landscape; individual batches barely dent it, and data quality is averaged over trillions of tokens. During the low-LR decay phase the steps are small and directional — the model is walking downhill into a specific minimum. Feed it curated math, code, reasoning, and clean web text during that walk and it settles into a minimum that is good at those things. Feed it the same noisy web slop it saw during pretraining and you waste the most valuable tokens in the entire run.

## 2.2 WSD vs cosine

The classic pretraining schedule is **cosine decay**: LR rises through warmup then follows a cosine curve down to a floor (typically 10% of peak) over a *predetermined* number of steps. Its fatal flaw for midtraining thinking is that it bakes the endpoint in at step zero — you cannot cleanly "add an annealing stage" to a cosine run because the schedule already spent its decay budget smoothly across the whole run.

**WSD (Warmup-Stable-Decay)**, also called trapezoidal or WSD-S, fixes this. Three phases:

- **Warmup** — LR ramps to peak (as usual).
- **Stable** — LR held *constant* at peak for the bulk of training. No predetermined end; you can train as long as you have tokens.
- **Decay** — LR drops rapidly to near-zero over the final ~10–20% of tokens, on the annealing mixture.

WSD's advantages are why it has largely won for anything with a distinct annealing stage: (1) the stable phase has no fixed length, so you can decide the total token budget late; (2) you can **branch** — checkpoint at the end of the stable phase and run *multiple* different decay experiments (different mixtures, different lengths) from the same trunk; (3) the decay phase is an explicit, isolated capability lever you can iterate on cheaply. MiniCPM, SmolLM2, OLMo 2, and most 2025–2026 open models use WSD or a close variant.

Cost of the decay: it is not free — a WSD decay over the last 10% of tokens costs 10% of your compute. But that 10% buys most of your annealing gains, and the ability to fork cheap decay experiments off a frozen trunk is worth more than the smoothness cosine gives you.

## 2.3 The decay-phase mixture swap

This is where annealing becomes a capability lever rather than just an LR schedule. Concrete published mixtures for the *annealing/decay* stage (contrast with the stable-phase web-heavy mix):

| Model | Stable-phase mix (approx) | Annealing-phase mix |
|---|---|---|
| SmolLM2 | 90% web / 10% code | 58% web / 24% code / 14% math / 4% synthetic (1T tokens) |
| Llama 3 405B | web-dominant, 15T total | upsample highest-quality sources; ~50% general / 25% math-reasoning / 17% code / 8% multilingual |
| Qwen3 | 30T general | "reasoning stage": 5T tokens upweighting STEM, code, reasoning, synthetic |
| OLMo 2 | DCLM/web | Dolmino mix — curated web + math (incl. synthetic) + academic + instruction-formatted |

The pattern is consistent: **upweight math, code, reasoning, and instruction-formatted data; downweight raw web; introduce a few percent of synthetic textbook/QA data.** A common heuristic (from Llama 3) is roughly **70% continuity data / 30% novel high-quality data** during the swap, chosen so the distribution shift is a nudge rather than a shock (distributional continuity, Chapter 1). Swapping 100% of the mixture at the decay boundary produces a loss spike and wastes early decay tokens re-stabilizing.

**Instruction-formatted data in the decay phase** is the quiet workhorse of the moved boundary (Chapter 1, 7). Sprinkling QA-style, chat-style, and CoT-style examples into annealing — *not* as masked SFT, just as ordinary next-token data — measurably improves the base model's zero-shot instruction-following and gives post-training a warmer start. This is why 2026 "base" models are noticeably more steerable out of the box than 2023 ones.

## 2.4 Microanneals: predicting a mixture's value cheaply

The problem with the decay-phase mixture being a capability lever is that evaluating a candidate mixture seems to require running the whole decay. **Microanneals** (OLMo 2's technique) solve this: run an *abbreviated* decay — a short LR decay on a small token budget (a few billion) from a fixed stable-phase checkpoint — for each candidate mixture, eval the result, and use the ranking to extrapolate which full-scale mixture will win. Because you branch off one frozen trunk, each microanneal is cheap and they are embarrassingly parallel.

This is the annealing analogue of the pretraining series' scaling-ladder discipline: **validate the mixture decision at small scale before spending the real decay budget.** RegMix formalizes the same idea as a regression problem (fit mixture-ratio → downstream-loss on many tiny runs, then optimize). Either way, the rule is: never pick your annealing mixture by intuition when a day of microanneals will rank your candidates for you.

## 2.5 Practitioner defaults

For an annealing/decay stage on a dense model you control:

| Knob | Default | Notes |
|---|---|---|
| Schedule | WSD | Cosine only if you cannot re-tool your trainer |
| Decay fraction | 10–20% of total tokens | Longer decay = more annealing effect, diminishing past ~20% |
| Decay shape | Linear or 1-sqrt to ~0 | 1-sqrt (used by some) decays faster early; linear is fine |
| Peak→floor | Peak LR down to 0 (or ~1% of peak) | Annealing genuinely wants near-zero, unlike cosine's 10% floor |
| Mixture shift | ~70% continuity / 30% novel-quality | Interpolate over first few % of decay, don't step-change |
| Synthetic fraction | 2–5% | Textbook/QA/CoT synthetic; more risks style collapse |
| Validation | Microanneals per candidate mixture | Rank before committing the full decay |

**If you downloaded a base model** that was already annealed (most open weights), you generally do *not* re-anneal from scratch — you either continue-pretrain (Chapter 3) with its own small decay, or go straight to post-training. Re-annealing someone else's annealed model tends to overfit the decay mixture and forget. Know whether your base is a stable-phase checkpoint (rare, e.g. some OLMo intermediate releases) or a fully-annealed release (the norm).

## 2.6 Decisions

1. **Use WSD** unless your framework forces cosine; the fork-cheap-decay-experiments property is worth more than smoothness.
2. **Spend 10–20% of tokens on decay**, LR to near-zero, on a curated mixture.
3. **Shift the mixture ~30% toward quality/capability data, interpolated, not step-changed.**
4. **Rank candidate mixtures with microanneals** before committing the full decay.
5. **Include a few percent of instruction/CoT-formatted data** in the decay to warm-start steerability.
6. **Don't re-anneal an already-annealed downloaded base** — continue-pretrain or post-train instead.

Next: continued pretraining, which reuses this machinery but points it at a domain or language corpus rather than a general quality mixture — and inherits the harder forgetting problem.
