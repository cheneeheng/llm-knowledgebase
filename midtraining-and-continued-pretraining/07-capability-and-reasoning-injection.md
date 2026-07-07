# Chapter 7 — Capability and Reasoning Injection

## 7.1 Conceptual picture

This is the chapter that most sharply distinguishes 2026 practice from 2023. Reasoning and instruction-following used to be things you *added in post-training* to a base model that had none. The finding that reshaped the pipeline: if you **front-load** reasoning-intensive data — long chain-of-thought traces, math derivations, code with explanations, instruction-formatted examples — into the *self-supervised* midtraining span, the resulting base model is already primed to reason, and post-training's job shrinks from *installing* reasoning to *eliciting and refining* it. The ICLR 2026 "Front-Loading Reasoning" line and Meta's "thinking in midtraining" work both make the case: reasoning is not a post-training-only phenomenon, and the optimal place to seed it is earlier than the field assumed.

Capability injection is the general form: using midtraining to deliberately install *capabilities* (reasoning, math, code, tool-call format, multilingual competence) via curated self-supervised data, as opposed to *knowledge* (Chapter 3/4) or *context length* (Chapter 5). The mechanism is the same annealing machinery from Chapter 2 — you are choosing what the model settles into as the LR decays — but the target is a skill, and the data is chosen to exhibit that skill richly.

## 7.2 Why front-loading beats late installation

Three reasons, each with practical weight:

1. **RL amplifies what exists; it struggles to create.** RLVR (post-training series, Ch 8) works by reinforcing good reasoning the model *sometimes* produces. If the base almost never produces long correct CoT, RL has nothing to grab — hence the cold-start SFT phase that R1 needed. Seed the reasoning behavior in midtraining and the base already produces it at reasonable frequency, so RL converges faster and reaches higher. The "front-loading" result quantifies this: reasoning data spent in midtraining yields more final capability than the same data spent in post-training SFT.
2. **Self-supervised scale beats demonstration scale.** You can put *far* more reasoning tokens through midtraining (billions, unlabeled, cheap) than through SFT (millions, curated, expensive). Breadth of exposure to reasoning patterns transfers.
3. **It warms up steerability for free.** Instruction-formatted data in the annealing mix (Chapter 2.3) makes the base model follow instructions zero-shot — measurably better than a base trained only on web text — which shortens the SFT stage and reduces its forgetting.

## 7.3 What data injects what capability

| Capability | Data to inject in midtraining | Notes |
|---|---|---|
| General reasoning | Long-CoT traces (math, logic, science), verified where possible | The headline lever; even a few % of decay tokens moves reasoning evals |
| Math | Proof corpora, worked solutions (OpenWebMath, synthetic step-by-step) | Dense; small fractions have outsized effect ("disproportionate gains") |
| Code | Code with natural-language explanation, docstrings, tests, whole repos | Explanation-paired code teaches reasoning-about-code, not just syntax |
| Instruction-following | QA-style, chat-formatted, task-formatted text (as plain next-token) | Warms steerability; do *not* mask it, it's not SFT |
| Tool / function-call format | Traces showing the tool-call schema in context | Seeds the format so post-training agentic RL starts warm |
| Multilingual | Balanced multilingual corpus, parallel where available | Preserve with replay when specializing later |

The recurring empirical note from the survey literature: **domain/capability data produces disproportionate gains even in small proportions** during annealing. You do not need the decay mixture to be 50% math to move math evals; 10–15% often does it. This is why the Chapter 2 mixtures upweight but do not dominate with any single capability.

## 7.4 The moved cold-start boundary

The post-training series describes the R1 "cold start" — a small SFT on long-CoT traces to give RL something to amplify. Midtraining reasoning injection **absorbs part of that job**. The boundary now looks like a gradient:

```
             reasoning "installed" here ──────────────►
  pretraining │ midtraining reasoning │ cold-start SFT │ RLVR
   (none)     │ injection (frequency  │ (format,       │ (reliability,
              │  and breadth)         │  polish)       │  length, correctness)
```

The practical consequence for *your* pipeline: **decide where you are putting the reasoning prior.** If you control midtraining, front-load it and expect a lighter cold-start. If you inherited a base model that already had reasoning midtraining (most strong 2026 open bases — Qwen3, OLMo 3 Think lineage, DeepSeek), you may be able to skip cold-start SFT and go closer to straight RL. If your base is a pure web-text base, you carry the full cold-start burden in post-training. Knowing which base you have is the single most useful thing to establish before designing your post-training.

## 7.5 Cautions specific to injection

- **Don't turn midtraining into masked SFT.** The injected instruction/CoT data is *unmasked next-token prediction* — you are shaping the base distribution, not cloning behavior. Masking it (loss only on responses) is a category error that belongs in post-training. Keep it plain LM loss.
- **Verify reasoning traces if you can.** Injecting *wrong* CoT teaches confident wrong reasoning (the SFT-trains-hallucination problem, one stage earlier). Prefer verified math/code traces; for unverifiable reasoning, prefer high-quality sources over volume.
- **Watch for style lock-in.** Heavy CoT injection at low LR can lock the model into a verbose "thinking out loud" style that post-training then has to manage (length control, entropy — post-training series Ch 8). Keep the injection proportionate; you are seeding a capability, not fixing the model's voice.
- **Balance against forgetting.** Reasoning injection is still a distribution shift; the general-eval + replay discipline (Chapter 8) applies. A model that gains 5 points of GSM8K and loses 4 of MMLU has not obviously improved.

## 7.6 Decisions

1. **Front-load reasoning** into midtraining if you control it — it beats spending the same data on post-training SFT and gives RL a warmer start.
2. **Inject via the annealing mixture** (Chapter 2): 10–15% reasoning/math/code + a few % instruction-formatted, unmasked, plain LM loss.
3. **Verify traces where possible**; prefer quality over volume for unverifiable reasoning.
4. **Establish where your reasoning prior lives** — if you inherited a reasoning-midtrained base, lighten or skip cold-start SFT; if a pure base, carry the full cold-start.
5. **Keep injection proportionate** to avoid style lock-in, and hold it to the forgetting budget.

Next: forgetting and stability — the tax that every technique in this series pays, and how to measure and pay it.
