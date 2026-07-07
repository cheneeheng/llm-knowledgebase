# Chapter 1 — The Big Picture

## 1.1 What midtraining is

Midtraining is the set of training stages that run *after* the bulk of pretraining and *before* alignment post-training. Where pretraining optimizes for **scale and coverage** across trillions of noisy web tokens, and post-training optimizes for **alignment** on demonstrations and preferences, midtraining optimizes for **quality and specialization** on tens to hundreds of billions of curated tokens. The 2026 survey literature defines it as the phase where a model "undergoes multiple annealing-style phases that refine data quality, adapt optimization schedules, and extend context length," shifting learning dynamics "from memorization toward abstraction."

The plain-language version: pretraining teaches the model *language and world* from a firehose; midtraining teaches it *to be good at the specific things you care about* — reasoning, code, math, long documents, your domain, your language — by feeding it a smaller, denser, cleaner diet while turning the learning rate down. It is still next-token prediction. It is still self-supervised. There are no human preference labels and no reward model. That is what keeps it on the pretraining side of the house even as it starts to install capabilities we used to think of as post-training's job.

It matters because it is the highest-leverage cheap stage in the whole pipeline. A base model is a diamond in the rough; midtraining is the cut. SmolLM2's final trillion tokens of annealing, Qwen3's 5T "reasoning stage," Llama 3's annealing upsample, DeepSeek-V3's context extension — in every case a single-digit-percent slice of total compute produced a large fraction of the benchmark movement. Skip it and you ship a base model that post-training then has to drag uphill from a worse starting point.

## 1.2 The three moved boundaries

Midtraining exists in the gap left by two boundaries that have moved since 2023.

**Boundary 1 — pretraining's tail became a stage of its own.** Pretraining used to be one cosine schedule from warmup to end. Then teams noticed that the *last* few percent of tokens, trained at low LR on the *best* data, did disproportionate work. That observation, plus the WSD (Warmup-Stable-Decay) schedule that made it a first-class design choice (Chapter 2), turned the tail into an explicit **annealing stage** with its own mixture. The pretraining series calls this the annealing phase; this series calls it the first stage of midtraining. Same tokens, different framing — the framing that treats it as a designed capability lever wins.

**Boundary 2 — reasoning moved earlier.** In the InstructGPT era, reasoning and instruction-following were purely post-training phenomena: the base model completed text, post-training taught it to think and obey. By 2026 that separation has collapsed. Front-loading long-CoT, math, and instruction-formatted data into midtraining ("reasoning injection," Chapter 7) produces a base model that is already primed for RL — the cold-start SFT that used to *install* reasoning now merely *awakens* it. The pretraining series' "thinking models" chapter and the post-training series' "RLVR" chapter both point at this seam; midtraining is the seam.

**Boundary 3 — adaptation stopped being fine-tuning.** Taking a general model and specializing it to code or medicine or a new language used to be called "domain fine-tuning" and done with SFT-scale data. It turned out that real domain competence needs *pretraining-style* self-supervised learning on a large domain corpus — continued pretraining (Chapter 3) — not a few thousand instruction pairs. Code Llama continued Llama 2 on 500B code tokens; SaulLM continued Mistral on 30B legal tokens. That is unambiguously training, not fine-tuning, and it lives here.

## 1.3 The map: where each technique sits

```
PRETRAINING                MIDTRAINING                     POST-TRAINING
(trillions of              (tens-hundreds of B tokens)     (demonstrations,
 web tokens)                                                preferences, RL)
                    ┌─────────────────────────────────┐
  stable-phase ───► │ Annealing / decay phase (ch 2)  │ ──► SFT cold-start
  next-token        │   - LR decay to ~0              │      (ch 4, post-training)
  prediction        │   - mixture swap to quality     │
                    │ Continued pretraining (ch 3)    │
                    │   - domain corpus, replay       │
                    │ Domain adaptation (ch 4)        │
                    │ Long-context extension (ch 5)   │
                    │   - RoPE scaling, progressive   │
                    │ Tokenizer surgery (ch 6)        │
                    │ Reasoning injection (ch 7)      │
                    └─────────────────────────────────┘
```

These are not strictly ordered — real recipes interleave them. Qwen3 does reasoning-stage annealing *then* long-context. Llama 3 folds annealing and long-context together. DeepSeek-V3 does long-context as two bolt-on 1000-step phases after the main run. But the dependency structure is real: **tokenizer surgery must precede everything** (it changes the token stream); **long-context extension usually comes last** in the self-supervised span (it is expensive per token and you want your final capability mixture inside the long window); **annealing is the natural container** for the rest because low LR + quality data is the setting in which all of these work best.

## 1.4 Core mental models

Four ideas recur across every chapter.

- **You are editing, not creating.** The base model is a working artifact with a loss landscape you are perturbing. Every gradient step on new data moves it toward the new distribution and away from the old one. The entire discipline is managing that trade — which is why "measure the general eval too" is the one rule you cannot skip.

- **Quality has rising marginal value as LR falls.** During the stable phase, at high LR, the model is churning; data quality matters but is averaged over trillions of tokens. During decay, at low LR, each token is nudging the model toward its final resting minimum — so the *composition* of the last tokens it sees is what it ends up looking like. This is why the decay-phase mixture is a capability lever, not a rounding error (Chapter 2).

- **Forgetting is the tax, replay is the payment.** Any specialization pressure induces catastrophic forgetting of the general distribution. Mixing 10–30% of the original general data back in ("replay") is the near-universal defense, and it is cheap. Under-replaying is the single most common midtraining mistake (Chapter 8).

- **Distributional continuity beats distributional shock.** Abruptly swapping the entire mixture, or spiking the LR back up, or jumping straight to 128K context, all produce loss spikes and stability gaps that waste tokens recovering. Every technique in this series works better staged: interpolate the mixture, re-warm the LR gently, extend context in steps.

## 1.5 Decisions this series will force

By the end you will have made these calls, each covered in its chapter:

1. **Do you even midtrain, or take someone's already-midtrained base?** (Ch 3, 4 — mostly a build-vs-download question, same as post-training's base-model choice.)
2. **WSD or cosine, and how long a decay?** (Ch 2)
3. **Continued pretraining vs domain SFT vs RAG for your specialization?** (Ch 3, 4)
4. **How much replay, of what?** (Ch 8)
5. **What target context length, via which scaling method, in how many stages?** (Ch 5)
6. **Touch the tokenizer or live with bad fertility?** (Ch 6)
7. **Inject reasoning here or leave it to post-training?** (Ch 7)
8. **Which framework, and does your pretraining stack carry over?** (Ch 9)

Everything else is execution. Start with Chapter 2: annealing is both the most universal midtraining technique and the one whose mechanics illuminate all the others.
