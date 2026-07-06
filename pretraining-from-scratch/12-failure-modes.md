# Chapter 12 — Failure Modes and Recovery

A field guide to what goes wrong. Most expensive pretraining disasters are one of these eight. The pattern to notice: the loud failures (crashes, spikes) are the *cheap* ones — detection is free and recovery is a checkpoint away. The silent ones (SDC, a subtly wrong recipe) are the expensive ones, and every defense against them is something you set up *before* the run.

## 12.1 Divergence / loss spikes

Loss suddenly climbs and doesn't recover. **Causes:** attention-logit explosion, a pathological data batch, too-high LR, FP8/FP16 numerical issues, a failing GPU emitting NaNs. **Response:** the playbook in Chapter 8.5 — classify (self-healing blip vs. climbing divergence), rewind to the last good checkpoint, skip the offending data window, lower LR if it recurs, strengthen stability controls (z-loss, QK-norm/QK-clip), then root-cause. **Prevention beats cure:** stability controls from day one are cheap relative to lost weeks; gradient norm usually foreshadows the spike.

## 12.2 Silent data corruption (SDC)

A GPU keeps running but computes subtly wrong values — no crash, just quietly degraded or poisoned training. **The scariest failure**, because you may not notice for a long time. **Response:** per-rank statistical monitoring for anomalies; periodic numeric self-checks; bitwise-reproducible restarts (a diffable divergence localizes the bad rank); willingness to health-check and swap suspect hardware. At scale, assume some SDC will happen and instrument for it.

## 12.3 Stragglers

One slow GPU or node drags every collective — all-reduce waits for the slowest participant — tanking throughput cluster-wide. One degraded GPU among 1,024 sets the pace for all 1,024. **Response:** per-rank step-time monitoring; automated eviction; pre-job burn-in (per-GPU GEMM benchmarks catch "silently slow" GPUs before they join the job).

## 12.4 Data-loading bottleneck

GPUs idle waiting for data — MFU mysteriously low, utilization dipping at step boundaries. **Response:** profile the input pipeline; more prefetch/workers; pre-tokenized binary shards staged on node-local NVMe; verify streaming sustains (global batch tokens ÷ step time) with margin. Never let a multi-million-dollar cluster wait on I/O.

## 12.5 Hardware failures (routine)

GPUs fall off the bus, HBM ECC errors, NICs flap, nodes reboot, switches hiccup — constantly, at scale (Llama-3 405B: ~one interruption every 3 hours on 16K H100s). **Response:** the fault-tolerance stack from Chapter 3.7 — automatic detection, async checkpoints, warm spares, sub-10-minute automated restart. Measured in goodput. This is normal operations, not an emergency, *if* you built for it.

## 12.6 Checkpoint problems

A corrupt checkpoint, or one that can't resume on a different GPU count/parallelism layout — discovered exactly when you need it most. **Response:** keep multiple rolling checkpoints (the latest may be the corrupt one) plus periodic permanent ones; use a resharding-aware distributed format (PyTorch DCP, Megatron distributed checkpoint); include dataloader and RNG state; **test restore before you rely on it** — a checkpoint you've never restored from is a hope, not a backup.

## 12.7 The expensive silent failure: a subtly wrong recipe

The run completes, loss looks fine — but a bad mixture choice, a tokenizer flaw (digits, whitespace, multilingual fertility), unmasked document boundaries, or a contaminated eval means the model is quietly worse than it should be, or your numbers are inflated. No alarm ever fires. **This is the failure the scaling ladder exists to prevent:** disciplined proxy ablations, held-out validation, decoding actual training batches back to text, and rigorous decontamination are the only detection mechanisms. Respect the ladder.

## 12.8 Over/under-training and mixture mistakes

Wrong N/D split (Chapter 2) or a poor mixture (Chapter 4) — recoverable only by retraining. Prevented on paper and at proxy scale; never fixed at token 10T.

---

Which is the theme of the whole report: **the cheap decisions are the important ones, and they all happen before the expensive run starts.** The run itself is operational excellence plus the humility to rewind when the evidence says so.
