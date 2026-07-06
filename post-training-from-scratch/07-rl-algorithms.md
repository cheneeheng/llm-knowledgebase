# Chapter 7 — RL Algorithms

## 7.1 Conceptual picture

RL for LLMs is, at its core, one loop: **sample** responses from the current policy at temperature ~1.0; **score** them (RM, verifier, environment); **estimate advantages** (how much better was this response/token than expected); take a few **clipped policy-gradient steps**; repeat. Formally it's usually a contextual bandit (one action = one whole response) rather than deep multi-step RL — the "steps" inside a response are tokens sharing one reward. Two design axes generate the whole algorithm zoo: *how you estimate the baseline/advantage* (learned critic vs group statistics) and *at what granularity you correct for off-policyness* (token vs sequence importance ratios, and how you clip them). Everything else is stabilization detail — and in LLM RL, stabilization detail is most of the difference between published algorithms.

A note on why online RL beats the offline methods of Chapter 5 on the same signal: it trains on the model's own current outputs (no distribution mismatch), it *searches* — discovering high-reward behaviors no dataset contained — and it keeps pressure on as the policy moves. The price is that every pathology in Chapters 6 and 16 is also discovered by that same search.

## 7.2 PPO — the founding recipe

PPO maximizes a clipped surrogate: per token, ratio `ρ = π(a|s)/π_old(a|s)` times advantage, clipped to `[1−ε, 1+ε]` (ε≈0.2), so no single update moves the policy far. Advantages come from GAE using a **learned value model** — a second full-size network trained alongside the policy — and the reward is typically RM score minus a KL penalty to the reference. PPO is what InstructGPT and Llama 2 used and it remains a strong algorithm *when the value model is good* — but that's the rub: you carry 4 models (policy, reference, RM, value), and a poorly initialized or poorly trained value model quietly ruins advantage estimates, which is exactly what happened across a hundred failed academic replications. VAPO (2025) showed value-based methods *done right* (value pretraining, decoupled GAE λ, length-adaptive tricks) beat critic-free methods on long-CoT reasoning — but "done right" is real engineering. PPO is the right choice when you have the infra maturity and want the ceiling; it is the wrong first algorithm.

## 7.3 GRPO and the critic-free family

**GRPO** (DeepSeekMath, then R1) replaced the value model with a *group baseline*: sample G responses (G=8–64) per prompt, advantage of each = its reward standardized within the group (`(r − mean)/std`). No critic, half the memory, and the group structure fits verifiable rewards perfectly (a group's pass/fail spread *is* the signal). This is the algorithm that trained R1 and became the open-source default.

Its known biases, and the fixes that define the successor zoo:

| Algorithm | Change vs GRPO | Why |
|---|---|---|
| **Dr. GRPO** | Drop the per-group std division and the length normalization | Std-scaling upweights near-solved/near-impossible prompts; length norm biases toward long wrong answers |
| **DAPO** | Clip-higher (asymmetric ε, e.g. 0.2/0.28), dynamic sampling (resample until group has mixed outcomes), token-level loss over the batch, overlong-response penalty, no KL | The battle-tested open recipe for long-CoT RLVR; fixes entropy collapse and zero-gradient groups |
| **GSPO** (Qwen 3) | Importance ratio at the *sequence* level (length-normalized), clip sequences not tokens | Token ratios accumulate variance over long sequences; sequence-level clipping notably stabilizes MoE RL |
| **CISPO** (MiniMax-M1) | Clip the IS *weights*, never zero out tokens' gradients | Keeps learning signal from low-probability but pivotal tokens (the "wait, let me reconsider" tokens) |
| **RLOO / REINFORCE++** | Leave-one-out baseline / PPO-tricks-on-REINFORCE | Simpler than GRPO; fine when you don't need its extras |

Advice: these differ far less than their papers imply; data and reward quality dominate algorithm choice. The 2026 pragmatic default is **GRPO with the DAPO tricks** (clip-higher, dynamic sampling, no/low KL, token-level loss) — most frameworks expose exactly this — or **GSPO if training MoE**.

## 7.4 The knobs that actually matter

- **KL to reference: decide, don't default.** In-loss KL (weight β≈1e-3–0.1) or in-reward penalty keeps the policy leashed — necessary when optimizing a hackable RM (RLHF), and it protects general capability. Pure-RLVR recipes (DAPO, R1-Zero-style) usually drop KL entirely: the verifier can't be fooled the way an RM can, and the leash slows learning. If you keep KL, know your estimator (the k3 estimator is the usual low-variance choice).
- **Staleness: how off-policy is each gradient step.** You generate a big rollout batch, then take several minibatch updates on it; later updates use data from a policy that no longer exists. Fewer updates per batch (1–4) = more on-policy = more stable. This same knob, turned further, becomes async RL (Chapter 12) — and its correction machinery (clipping, truncated importance sampling) is shared.
- **Sampling:** temperature ~1.0 for training rollouts (exploration lives here; low temperature starves it), top-p 1.0 — and make sure *eval* sampling params are whatever you'll serve with.
- **Batch sizes:** rollout batch 256–1024 prompts × G samples; minibatch such that you take 1–4 steps; LR ~1e-6 (constant, no fancy schedule).
- **What to watch, every run:** reward (up), response length (explained moves only), **token entropy** (slow decline fine; collapse fatal; clip-higher and dynamic sampling are the levers), clip fraction (<~0.2), KL/logprob gap vs the rollout engine (Chapter 12's mismatch signal), and pass@k on a holdout (pass@1 up while pass@16 flat/down = you're sharpening, not learning — Chapter 8).

## 7.5 What to actually use

Single-turn verifiable tasks: GRPO+DAPO tricks (or GSPO on MoE), no KL, in verl/TRL/OpenRLHF defaults. RM-scored open-ended RLHF: same family but *keep* the KL leash and refresh the RM (Chapter 6). Multi-turn agents: Chapter 9's variants. You have a mature platform team and want the last few points on long-horizon reasoning: consider value-based (VAPO-style) PPO. And always run the Chapter 4 rejection-sampling baseline first — if RL can't beat it on your task, your reward signal, not your algorithm, is the problem.
