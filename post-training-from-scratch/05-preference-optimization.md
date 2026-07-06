# Chapter 5 — Preference Optimization

## 5.1 Conceptual picture

Preference data says "response A over response B" — it encodes *relative* judgments about qualities you can't write a checker for: helpfulness, tone, clarity, harmlessness. The classical route (Chapters 6–7) trains a reward model on those pairs and runs RL against it. **DPO (Direct Preference Optimization, 2023)** collapsed that: for the standard RLHF objective (maximize reward with a KL leash to a reference policy), the optimal policy has a closed form in which the reward is proportional to `log π(y|x) − log π_ref(y|x)`. Substituting into the Bradley–Terry preference likelihood yields a simple classification loss directly on the policy: push up the margin by which the model prefers chosen over rejected, *relative to a frozen reference model*, with temperature **β** controlling the implicit KL leash (low β ≈ move far; typical 0.05–0.3).

What you give up versus RL: DPO is **offline** — it optimizes on a fixed set of pairs rather than the model's own evolving outputs, so it's one step of policy improvement, not an optimization loop. What you get: no reward model, no rollouts, no RL infra — an SFT-shaped job that runs anywhere SFT runs, at ~2× SFT memory (the frozen reference forward pass).

## 5.2 Mechanics and hyperparameters

Data per example: prompt, chosen, rejected — generated on-policy with an AI judge as in Chapter 3. Defaults that work (8B): **LR 5e-7–1e-6** (10–30× lower than SFT — the #1 beginner error is SFT-scale LRs, which lobotomize the model), **1 epoch**, β 0.1, global batch 64–256 pairs, applied *after* SFT on a model that has already seen the format.

Monitor three things: reward margin (chosen minus rejected implicit reward — should grow), **chosen log-prob** (see below), and output length (DPO on judge-labeled data reliably inflates verbosity — track mean response length on a fixed prompt set and compare judge win-rates *length-controlled*, Chapter 14).

The known pathology: DPO only needs the *margin* to grow, and often achieves it by pushing the chosen response's probability down more slowly than the rejected's — both fall. Mild versions are normal; a collapsing chosen log-prob means the model is draining mass off your good responses (and everything similar to them) toward who-knows-where. Fixes: lower LR, higher β, better (more on-policy) pairs, or a chosen-NLL auxiliary term (DPOP/"DPO-positive"; Tulu 3 shipped with a length-normalized DPO variant for related reasons).

## 5.3 The variant zoo

| Variant | One-line change | When it earns its place |
|---|---|---|
| **DPO** | The reference formulation | Default; best-understood, best-supported |
| **IPO** | Bounded objective instead of logistic | When DPO overfits preference noise |
| **cDPO / robust DPO** | Assume a label-noise rate | Noisy judge labels |
| **KTO** | Per-response thumbs-up/down (prospect-theoretic), no pairs needed | You have binary feedback from production, not pairs |
| **ORPO** | Odds-ratio penalty folded into SFT; no reference model, single stage | Compute-tight; combined SFT+preference in one pass |
| **SimPO** | Length-normalized implicit reward, no reference model | Strong AlpacaEval-class results, cheaper; less leash — watch drift |
| **Iterative/online DPO** | Regenerate pairs from the current policy each round, repeat | Recovers much of online RL's edge; the best cheap upgrade |

Guidance: the variants matter less than the data. Published head-to-heads (Tulu 3's ablations among them) show DPO-with-good-on-policy-pairs beating every clever objective on off-policy pairs. Pick DPO (or ORPO if you want one stage), invest the savings in pair quality, and iterate rather than exotify.

## 5.4 Where preference optimization sits in 2026

DPO lost the capability frontier to online RL — labs push reasoning/agentic gains with GRPO-family methods, and use RM-scored RL rather than offline pairs for the final polish of flagship models. But it holds three durable niches: (a) the **cheap workhorse stage** in open recipes (Tulu 3, Olmo 3, SmolLM 3 all run SFT → DPO → RLVR) — it reliably buys chat quality, instruction following, and safety tone for a few hundred GPU-hours; (b) **style/safety shaping** where verifiable rewards don't exist and RM+RL is overkill; (c) **the practitioner's ceiling**: for a team without RL infrastructure, SFT + iterative on-policy DPO is 80–90% of what's reachable, at 5% of the systems complexity. Run it before you contemplate anything in Chapter 7; if DPO on good pairs didn't move your eval, RL on the same signal probably won't either — your bottleneck is the signal.
