# Chapter 6 — Reward Models

## 6.1 Conceptual picture

A reward model (RM) turns judgment into a number: given prompt and response, emit a scalar the RL loop can maximize (or a ranking Best-of-N can select with). RMs exist because the thing you want — human judgment — is too slow and expensive to sit inside a training loop, so you distill it into a model and query that instead. Everything important about RMs follows from one fact: **the RM is a lossy, exploitable proxy trained on finite data, and the policy is an adversary with millions of gradient steps to find its bugs.** Gao et al.'s overoptimization curves made this quantitative: as you optimize against a fixed RM, *true* quality rises, peaks, and falls while RM score keeps climbing. You never get to optimize an RM hard; you get to optimize it *carefully, for a while* (bigger RMs and more RM data move the peak later — they don't remove it).

## 6.2 The classical recipe: Bradley–Terry RMs

Architecture: take a strong *instruct* checkpoint (not the raw base — you want it to already understand quality), replace the LM head with a scalar value head reading the final token's hidden state. Loss: Bradley–Terry on pairs, `−log σ(r(x, y_chosen) − r(x, y_rejected))`. Training: **1 epoch** (RMs overfit pairs brutally fast), LR 1e-6–9e-6, batch 64–256 pairs, with chosen/rejected always in the same batch. Data: the preference pairs of Chapter 3 — diversity of prompts and of *failure types* matters more than raw count; 100k–1M pairs is the industrial range, 10k–100k is workable. Held-out pair accuracy of 70–80% is normal and fine (human inter-annotator agreement isn't much higher); what matters is Best-of-N and downstream policy quality, not accuracy.

Practical hardening: ensemble or at least seed-average RMs (disagreement flags hackable regions — WARM formalized merged-RM robustness); add explicit length features or length-balanced pairs, or you will train a verbosity detector; include safety pairs unless a separate safety RM handles that axis (Llama 2 ran separate helpfulness and safety RMs); refresh the RM on pairs drawn from the *current* policy every RL round or two — a stale RM meets an out-of-distribution adversary.

## 6.3 Generative reward models and rubrics — the 2025–26 shift

The scalar-head RM is being displaced for many uses by **generative reward models (GenRMs)**: an LLM that *reasons in text* about the response — critique, compare, check against criteria — then emits a judgment. Reasoning before judging buys accuracy and, crucially, *auditability*: you can read why a reward was assigned, which turns reward-hacking hunts from archaeology into reading. RM-R1/JudgeLRM-style work trains judges with RLVR on verifiable judgment tasks; strong 7–32B trained judges now beat much larger untrained ones on RewardBench-class evals.

**Rubric rewards** are the leading answer for non-verifiable domains (writing, advice, open analysis): instead of one opaque scalar, score against an explicit checklist — per-prompt generated rubrics or curated rubric banks (OpenRubrics; production recipes reportedly run banks of 10k+ rubrics), each item checked by a judge, aggregated to a reward. Rubrics decompose judgment into harder-to-game pieces, are inspectable, and let domain experts contribute *criteria* rather than labels. Cost is real: a GenRM/rubric reward is an LLM inference (possibly several) per training sample — Chapter 12's rollout problem, squared. Common compromise: GenRM/rubrics for a periodic refresh and audit; a distilled scalar RM (trained on GenRM verdicts) inside the hot loop.

## 6.4 Process reward models

PRMs score each *step* of a reasoning trace rather than the outcome ("Let's Verify Step by Step"; Math-Shepherd automated the step labels via rollout value estimates). Dense signal in theory; in practice two things happened: (a) PRMs proved excellent for **inference-time search/reranking** (step-level beam search, Best-of-N), and (b) frontier RL recipes largely *declined* to use them as the RL reward — DeepSeek-R1 explicitly dropped PRMs after finding step-labeling ill-defined and the models prone to reward hacking under optimization. Treat PRMs as a search/verification tool and an evaluation aid, not your RL objective, unless you have the audit machinery to police them.

## 6.5 Evaluating reward models

RewardBench (v1, then the harder v2) is the standard pair-accuracy benchmark family; use it for smoke-testing, not selection — **benchmark accuracy correlates weakly with downstream policy quality**, and published results repeatedly find high-scoring RMs that train worse policies. The evaluations that predict RL outcomes: **Best-of-N lift** on *your* prompt distribution scored by a trusted judge/humans; robustness probes (does reward rise under length padding, confident tone, sycophantic agreement, keyword stuffing?); and small **proxy RL runs** watching for early hacking. An afternoon of BoN evaluation on your own prompts beats any leaderboard.

## 6.6 Decisions

Small team, verifiable domain: skip the RM entirely — verifiers are better rewards (Chapter 8). Chat/style axis on a budget: judge-labeled DPO (Chapter 5) before any RM. Building real RLHF: start from an instruct model + BT head on on-policy pairs, length-debias, 1 epoch, ensemble if you can; evaluate by BoN on your prompts; refresh per RL round. Non-verifiable domains at 2026 standards: rubric/GenRM rewards with a distilled scalar in the loop. Always: keep a holdout judge the policy never optimizes against, and read the top-reward transcripts weekly — the RM's bugs are your model's future personality.
