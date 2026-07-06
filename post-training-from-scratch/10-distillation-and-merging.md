# Chapter 10 — Distillation and Merging

## 10.1 Conceptual picture

Distillation transfers a strong model's behavior into a cheaper one; merging combines several trained models' weights into one. Both matter because the 2026 economics of model families changed: you post-train one flagship *hard* (all of Chapters 4–9), then propagate — nobody runs the full pipeline independently per size anymore. Distillation is also, quietly, how most "SFT data" is made (Chapter 3), how small reasoning models are made (Chapter 8.5), and how capability lost to a narrow stage gets bought back.

## 10.2 Off-policy distillation

What almost everyone means by "distillation": generate responses from the teacher, SFT the student on them (sequence-level knowledge distillation). Everything in Chapter 4 applies; the distillation-specific decisions are teacher sampling (rejection-sample with a verifier/judge — distill the teacher's *good* behavior), coverage of your prompt distribution, and license/ToS on the teacher. A refinement when teacher and student share a tokenizer: train on the teacher's full **logits** (forward-KL per token) rather than sampled text — denser signal, rarely worth cross-tokenizer surgery otherwise.

Its structural weakness is distribution mismatch: the student trains exclusively on *teacher* prefixes, so when its own generation drifts somewhere the teacher never went, it has no idea how to continue — errors compound token by token (the imitation-learning exposure-bias problem). Mild for short chat responses; severe for long CoT and agent trajectories.

## 10.3 On-policy distillation — the 2025–26 workhorse

On-policy distillation (OPD; GKD is the academic root) fixes the mismatch: the **student samples** its own responses, and the **teacher grades every token** — the training signal is per-token reverse KL toward the teacher's distribution *on the student's own trajectories*. Think of it as RL where the reward is dense (every token, not one scalar per response) and the "reward model" is a teacher that can't be hacked in the usual way, or as SFT that follows the student wherever it goes. Consequences, now well replicated (Qwen 3's small models, the Thinking Machines write-ups, subsequent lab reports): compared to RLVR it reaches similar or better student quality at a fraction of the compute (dense signal ⇒ far fewer episodes; commonly ~1–2 orders of magnitude cheaper); compared to off-policy SFT it generalizes off the teacher's happy path. Infrastructure sits between SFT and RL: you need student generation plus teacher forward passes (Chapter 12's machinery, minus reward plumbing — and it works well with LoRA students).

When OPD is the answer: making the small sizes of a family match the flagship's post-training; restoring capability after a narrowing stage (safety training, domain SFT) using *the pre-stage model as its own teacher*; continual-learning recovery; cheap approximation of an RL result you can't afford. When it isn't: no teacher meaningfully better than the student (that's what RL is for), or cross-family tokenizer mismatch (fall back to sequence-level).

## 10.4 Model merging

Weight-space arithmetic on models sharing an architecture and lineage: checkpoint averaging within a run; **model soups** (average fine-tunes from different hyperparameters/data); task arithmetic and its robust descendants (TIES, DARE — trim/sign-resolve/sparsify task vectors) to combine specialist fine-tunes; SLERP for pairs. It shouldn't work as well as it does, but it's load-bearing in real recipes: Llama 3 averaged models trained on different data-mix variants; WARM merges reward models for hack-resistance; WARP-style RLHF merges the policy back toward reference as a KL-control device; several open recipes (Cohere's, various Chinese labs') explicitly merge per-domain post-trains (code/math/safety specialists) into the shipping model. Uses, in decreasing reliability: (1) checkpoint/seed averaging — nearly free variance reduction, do it; (2) merging same-recipe variants trained on different mixes — usually recovers the best of each; (3) merging genuinely different specialists — real but fiddly, evaluate hard for interference; (4) merging across different base models — don't. Merging is a complement to distillation, not a substitute: it combines *siblings*; distillation moves capability *down* a family. Always eval the merge like a new model — including safety regressions, which merging is notorious for reintroducing.

## 10.5 Decisions

Small model, strong teacher available: off-policy distill for the cold start, OPD to finish — RL only if you need to exceed the teacher on a verifiable domain. Flagship: RL (Chapters 7–9), then OPD it into every other size. Post-stage capability dips: OPD from the pre-stage checkpoint. Multiple specialist fine-tunes: TIES/DARE-merge, then a light general stage on top. And run seed/checkpoint averaging by default — it's the cheapest quality you'll buy anywhere in this report.
