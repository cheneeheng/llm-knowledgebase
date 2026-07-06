# Chapter 9 — Running the Job and Evaluating It

You've scoped, built the data, chosen architecture, framework, parallelism, and optimizer. Now you actually run — for weeks. This chapter is operations: the discipline that turns a good plan into a finished model.

## 9.1 The pre-flight checklist

Before the real run, in order:

1. **Overfit a tiny batch.** Can the model memorize a handful of sequences to near-zero loss? If not, something is broken (data, masking, loss computation). The cheapest bug-catch there is.
2. **Short proxy run at target config.** Run your ~1B (or scaled-down target) end-to-end for a few billion tokens with the *exact* production stack — same framework, parallelism pattern, dataloader, precision. Confirm loss decreases smoothly, MFU is healthy, checkpointing works, and resume-from-checkpoint reproduces the loss curve exactly.
3. **Verify the data pipeline end-to-end.** Decode packed batches back to text — is it what you think? Are document boundaries masked correctly? Is the mixture ratio actually what you configured? (Measure it; don't trust the config.)
4. **Test failure recovery.** Kill a node mid-run on purpose. Does the job detect it, restart from checkpoint, and continue at the same loss? If you haven't tested this, you don't have it.
5. **Confirm determinism knobs and seeds are logged**, and the full config is version-controlled and attached to the run.
6. **Sanity-check throughput and cost.** Tokens/sec × price → does projected wall-clock and dollar cost match the plan? If MFU is far below expectation, fix it *now*, not at token 3T.

## 9.2 Monitoring during the run

Watch continuously (wire into a live dashboard — W&B/MLflow + Prometheus/Grafana — with alerts; a human should be able to glance and know the run is healthy):

- **Training loss** and **held-out validation loss** — the primary signal; should decrease smoothly. Per-domain loss localizes data problems.
- **Gradient norm** — sudden growth precedes divergence (Chapter 8.5).
- **Learning rate** — confirm the schedule is doing what you think.
- **Activation/logit statistics** — max attention logit, output-logit magnitude: early warning for the explosions QK-norm/z-loss prevent.
- **MFU / tokens-per-second / goodput** — throughput regressions signal hardware or data-loading problems.
- **Per-rank step time** — an outlier rank is a straggler or failing GPU.
- **Hardware telemetry** — temperatures, ECC error counts, NIC health, power.
- **Periodic downstream evals** at checkpoints (9.3) — capability emerging, not just loss dropping.

Alert on: loss spike/NaN, grad-norm explosion, step-time regression >5%, ECC bursts, heartbeat loss.

## 9.3 Evaluating a base model

You are evaluating a *base* model — pretrained, not instruction-tuned — which constrains method:

- **Perplexity / loss on held-out sets.** The cleanest signal of language-modeling quality; ideal for tracking progress and comparing checkpoints on the *same* data. Not comparable across different tokenizers or datasets.
- **Few-shot benchmarks.** Base models aren't instruction-followers; probe them few-shot with a standard harness (**lm-evaluation-harness**) so numbers are comparable and reproducible. The standard basket:
  - Knowledge: **MMLU / MMLU-Pro**
  - Math: **GSM8K / MATH**
  - Code: **HumanEval / MBPP / LiveCodeBench**
  - Commonsense/reasoning: **HellaSwag / ARC / WinoGrande**, **BBH**
- **The evaluation ladder.** Early in training (and for small ladder models), most benchmarks sit at noise floor; use "easy-mode" signals — perplexity on curated sets, cloze-style accuracy, benchmark variants with continuation scoring — that move smoothly before emergent benchmarks do. Design your eval suite so *something* is informative at every checkpoint.
- **Long-context evals** if you did extension: needle-in-a-haystack, **RULER**, and longer-form retrieval/reasoning variants.
- **Decontamination check.** Confirm (and state) that eval data wasn't in training. Contaminated numbers are worthless (Chapter 4.3, stage 7).
- **Track capability curves** — benchmark scores vs. tokens across checkpoints. Capabilities emerge non-linearly; the curves tell you whether the annealing phase is paying off and whether you're saturating.

A caution: benchmark obsession misleads. Base-model benchmarks predict post-training *potential*, imperfectly. Loss curves, a broad benchmark basket, and qualitative generation inspection together give a truer picture than any single number.

## 9.4 The day-to-day of babysitting a run

- **A run owner is on call.** Someone owns the dashboard, triages alerts, and executes the spike playbook (8.5) and restart procedures (3.7). Rotate the duty; write runbooks so 3 a.m. decisions are pre-made.
- **Change discipline.** Mid-run changes (data mixture at phase boundaries, batch ramp, LR decisions) are planned and logged, never improvised. Every intervention gets a timestamped note attached to the run.
- **Weekly health review.** Loss vs. prediction from your scaling-law fit; eval curve trajectory; goodput and failure statistics; storage burn rate; budget burn vs. plan. If loss is off the predicted curve, investigate — don't hope.

## 9.5 Reproducibility and record-keeping

Log everything needed to reproduce or debug: full config, code commit, data snapshot/version and mixture weights per phase, tokenizer version, hardware and topology, every hyperparameter, all interventions, and the full loss/eval history. Keep permanent checkpoints at regular intervals (Chapter 3.6) — they are the raw material for post-hoc analysis, for the annealing-branch experiments WSD enables, and for whoever post-trains the model. Months of compute deserve a record you can stand behind and learn from.
