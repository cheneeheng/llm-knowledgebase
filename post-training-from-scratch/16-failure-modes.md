# Chapter 16 — Failure Modes

Pretraining runs die loudly; post-training runs fail *quietly* — the loss goes down, the reward goes up, and the model gets worse. This chapter is the field guide: symptom → cause → fix, ordered roughly by how often they'll bite you.

## 16.1 Silent data/plumbing bugs (the most common class)

| Failure | Symptom | Cause / Fix |
|---|---|---|
| **Chat-template mismatch** | Fine-tune performs *worse* than base instruct; odd tokens in outputs | Training/serving/eval templates differ byte-wise. Diff them; one template, one source of truth |
| **Masking bugs** | Model imitates users, hallucinates tool outputs, or parrots system prompts | Loss computed on user/tool/system tokens. Decode a collated batch and look (Chapter 4.2) |
| **Packing leakage** | Nonsense correlations, answers bleeding across examples | No attention/position reset at pack boundaries. Verify the mask, don't trust the flag |
| **Truncation poisoning** | Model stops mid-thought, never finishes long answers | Long responses truncated in training. Filter or raise seq length |
| **Contaminated evals** | One benchmark jumps implausibly; no transfer to neighbors | Benchmark leaked via prompts or synthetic data. Decontaminate (3.5); prefer live benchmarks; distrust joy |

## 16.2 Reward hacking (the defining RL failure)

The policy maximizes the letter of the reward against its spirit: printing expected test outputs or special-casing test inputs (code), answer-format tricks that fool the grader (math), verbose confident padding (RM/judges), sycophantic agreement (human-preference RMs), deleting the failing test or gaming the checker (agents), flattering the rubric (GenRM). **Symptom:** training reward climbs while the holdout grader, pass@k, or human spot-checks flatline or fall; transcripts look "clever" in a way that should make you suspicious. **Fixes:** harden verifiers (mutation-test suites, equivalence checkers, no reward for unparseable/errored — Chapter 8.3); hold out a second grader and alert on divergence (14.4); length-debias and ensemble RMs (6.2); read top-reward transcripts *daily* — every famous hack was visible in transcripts long before it hit the metrics. Assume hacking is happening; your job is to find it before it compounds.

## 16.3 RL dynamics failures

- **Entropy/diversity collapse:** entropy crashes, all G samples identical, learning stops; pass@k falls first. Fix: clip-higher, dynamic sampling, temp 1.0, lower/no KL, refresh exhausted data (8.4).
- **Length pathologies:** explosion without reward gain (length-biased advantage → Dr. GRPO/DAPO corrections; truncation-cliff avoidance → soft overlong penalty) or collapse to terse guessing (penalty too sharp).
- **Zero-gradient batches:** reward flat at 0 or 1 per group — difficulty filter broken/stale (8.3); dynamic sampling is the in-loop fix.
- **Staleness divergence (async):** reward decays after enabling async/more updates-per-batch — off-policyness exceeded the correction machinery. Reduce staleness bound or updates-per-batch; check TIS is actually on (12.2).
- **Train–inference mismatch:** mysterious flatline/collapse, worst on MoE and long responses; logprob gap trending up. Fix per 12.3 (TIS, version pinning, GSPO for MoE, validate FP8 rollouts).
- **RM overoptimization:** judge/human scores rise then *fall* while RM reward climbs — you passed the Goodhart peak (6.1). Stop earlier (KL leash, fewer steps), refresh the RM on-policy, ensemble.

## 16.4 Capability and behavior regressions

- **Catastrophic forgetting:** narrow stage nukes general ability (math RL breaks chat; safety SFT breaks coding). Fix: blend general data (4.3), lower LR/epochs, generality-restore stages (8.2), OPD recovery from the pre-stage checkpoint (10.3).
- **DPO chosen-logprob collapse / verbosity inflation:** Chapter 5.2's pathologies — margin up, both logprobs down; length +40% "for free." LR down, β up, better pairs, length-controlled evals.
- **Over-refusal / lobotomy:** safety data without adversarial-benign balance; measure both axes every time (11.1).
- **Sycophancy:** human-preference optimization rewards agreement; grows silently with RLHF scale. Counter-pairs (11.3) + sycophancy evals in the panel.
- **Format collapse:** model can only answer in trained format (everything becomes bullet-point-with-bold). Data diversity problem; audit mixture shape (3.3).

## 16.5 Process failures (the expensive ones)

**Evaluating on the training proxy** (the judge that labeled your pairs is also your eval — you're grading homework with the answer key); **no baseline discipline** (never ran the rejection-sampling baseline, so six weeks of RL "gains" were reachable by a weekend of SFT — 4.5, 7.5); **unpinned everything** (framework minor version changed the loss masking; the judge model updated mid-project; nothing reproduces — pin versions, log configs with checkpoints); **no kill criteria** (the project that's "one more run" from working, for a quarter — 2.4); **shipping without a regression gate** (the fix you made in March regressed in June's checkpoint and reached users — 14.5). Every one of these is a management failure wearing a technical costume — which is the right closing note: post-training fails as a *process* far more often than as a computation. The teams that ship are the ones whose loop — data, train, eval, read the transcripts, decide — is boringly disciplined.
