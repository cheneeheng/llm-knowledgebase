# Chapter 10 — Evaluation

## 10.1 Conceptual picture

Midtraining evaluation answers one question the training loss cannot: **did this edit improve the model on the axis I care about without degrading the axes I don't want to lose?** Because midtraining is fundamentally a *trade* (Chapter 1, 8), every evaluation is a *paired* measurement — target metric up, general metric held — run against a *baseline* (the model before the edit). A midtraining eval report that shows only the target metric going up is not an evaluation; it is marketing, and it hides the forgetting that a real report would surface.

This chapter assembles the harness the previous chapters kept referring to, plus the midtraining-specific concerns (long-context evals, contamination, checkpoint selection).

## 10.2 The four-axis harness

Every midtraining run should report four things at every checkpoint, against baseline:

1. **Target capability** — the reason you did the run. Domain benchmark (LegalBench, MedQA, HumanEval for code), long-context suite (RULER, ∞Bench), reasoning eval (GSM8K, MATH, MMLU-Pro), or your own held-out domain task. This is the metric you're *buying*.
2. **General retention** — MMLU/MMLU-Pro, HellaSwag, ARC, a general reasoning eval. The metric you must not *sell*. Budget: ≤1–2 point drop (Chapter 8).
3. **Behavioral integrity** — does it still follow instructions, stop at EOS, handle short context, produce well-formed output. Cheap spot-checks that catch the qualitative breakage benchmarks miss (a domain CPT that turned the model terse and robotic).
4. **A fast proxy** — held-out perplexity on general *and* target distributions. Moves before the benchmarks, cheap to run every few hundred steps, your early-warning system.

Run 1–3 at every saved checkpoint; run 4 continuously. Establish the **baseline before step zero** — you cannot claim a gain without the pre-edit number, and "the base model's published MMLU" is not a valid baseline because your eval harness/prompt format will differ. Re-measure the base yourself.

## 10.3 Long-context evaluation specifically

Repeating Chapter 5.5 because it is the most-botched midtraining eval:

- **Never use perplexity as your long-context metric.** It falls smoothly while retrieval ability stays broken.
- **RULER for effective length.** It reveals that a model "trained to 128K" is often reliable only to 32–64K. Report *effective* context length (the length at which task accuracy stays above threshold), not the nominal trained length. This is the honest number.
- **NIAH (multi-needle) as a floor check** — a model failing single-needle retrieval is not long-context, full stop.
- **A realistic suite** (LongBench, HELMET, ∞Bench) for the quality bar.
- **Always pair with a short-context regression check** — extension frequently costs short-context quality, and that trade is usually bad.

## 10.4 Contamination and decontamination

Midtraining data — especially synthetic, reading-comprehension-converted, and instruction-formatted data (Chapters 2, 4, 7) — is *high-risk for contamination* because it's often generated or curated from sources that overlap benchmarks. A contaminated midtraining set produces a model that scores well and is actually worse, which is the exact silent-failure mode the post-training series warns about, one stage earlier.

- **Decontaminate against your eval sets** before training: n-gram or embedding-overlap filtering of the midtraining corpus against every benchmark you'll report. The pretraining and post-training series both cover the mechanics; apply them here.
- **Be especially suspicious of synthetic data** — an LLM generating "textbook" or "reasoning" data will happily regurgitate benchmark items it memorized. Filter synthetic data as aggressively as scraped data.
- **Hold out a private eval the training loop has never seen** — the ultimate contamination defense (the post-training series' core discipline). If your private eval and your public eval disagree, trust the private one.

## 10.5 Checkpoint selection

Midtraining, like SFT, overfits, and **the best checkpoint is frequently not the last** (Chapters 2, 8, 9). Selection is a real decision, not an afterthought:

- **Save frequently** (every epoch or few thousand steps) so you have candidates.
- **Select on the paired trade**, not the target metric alone: the best checkpoint is the one with the best target metric *subject to* general retention staying within budget. A checkpoint that's +6 on target but -5 on general loses to one that's +5 target, -1 general.
- **Watch for the stability-gap shape** (Chapter 3.4): target metric dips then climbs — don't select an early trough, but also don't assume monotonic improvement to the end.
- **For a decay/annealing stage**, the endpoint (LR≈0) is usually best *for that mixture* — but if you forked multiple decay experiments (Chapter 2.2), you're selecting across forks, and microanneal rankings should have predicted the winner.

## 10.6 Statistical hygiene

The post-training series' eval-statistics discipline applies unchanged and is worth restating: benchmark deltas of 1–2 points on small eval sets are often noise. Use enough eval items, report confidence intervals or multiple seeds where feasible, and don't over-interpret a single run's single-point movement — especially when deciding whether a forgetting drop is "real." When the target gain and the general drop are both within noise, you have learned that the edit did little, which is itself a useful (and common) result.

## 10.7 Decisions

1. **Report four axes against a self-measured baseline** at every checkpoint: target, general retention, behavioral integrity, fast proxy.
2. **Establish the baseline before step zero**; don't trust published numbers as your baseline.
3. **Evaluate long context with RULER/NIAH, report effective length**, never perplexity, always with a short-context regression check.
4. **Decontaminate the midtraining corpus** (especially synthetic/converted data) against every eval, and keep a private held-out eval.
5. **Select checkpoints on the paired trade**, not the target metric alone; save frequently, expect the best is not the last.
6. **Apply statistical hygiene** — treat sub-2-point movements on small sets as possibly-noise.

Next: playbooks — the previous ten chapters assembled into concrete end-to-end recipes by scale.
