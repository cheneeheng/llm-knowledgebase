# Chapter 14 — Failure Modes

Serving fails differently from training. Training dies loudly on one long job; serving degrades *under load*, on a stream of requests, often only at the tail and only at peak — so failures are intermittent, load-dependent, and easy to miss in a quiet test. Each entry is symptom → cause → fix. Most trace to two roots: **running out of KV memory under concurrency**, and **optimizing the wrong metric** (throughput/averages instead of goodput/tails).

---

## 14.1 OOM under load (but fine in testing)

**Symptom:** works perfectly in low-traffic testing, then OOM-crashes or evicts requests at peak concurrency.
**Cause:** KV cache scales with `batch × sequence_length` (Chapter 2.4, 3.2); your test had a small batch and short outputs, production has neither. You sized HBM for weights, not for weights + KV-at-real-concurrency.
**Fix:** size memory for peak concurrent requests at realistic output lengths, with headroom; enable FP8 KV cache (Chapter 3) and paged KV (default); cap max concurrent sequences so the scheduler queues rather than OOMs; load-test at realistic concurrency and output length, not with toy prompts.

---

## 14.2 KV-cache thrash / preemption storms

**Symptom:** throughput collapses under sustained high load; latency spikes erratically; logs show constant preemption/recompute.
**Cause:** more requests admitted than KV cache can hold, so the scheduler constantly preempts (pages out) and resumes (recomputes/reloads) sequences, spending compute on churn instead of progress.
**Fix:** lower the max-concurrency/admission cap so admitted requests fit in KV; enable KV offloading (Chapter 3) to add capacity; scale out replicas (Chapter 11); admit-and-queue rather than admit-and-thrash (Chapter 4.6 goodput over throughput).

---

## 14.3 Prefill blocks decode — tail-latency cliff

**Symptom:** most requests are fast, but p99 TPOT spikes badly and users report the model "freezing" mid-answer.
**Cause:** a long prompt's prefill runs in one big step and stalls all ongoing decodes (Chapter 4.3). Averages look fine; the tail is ruined.
**Fix:** enable **chunked prefill** (Chapter 4.4) — the direct fix; interleaves prefill chunks with decode. At scale, prefill/decode disaggregation. Monitor **percentile** latency (p95/p99), not averages, or you'll never see this.

---

## 14.4 High throughput, terrible goodput

**Symptom:** dashboards show great tokens/sec, but users complain and SLO compliance is low.
**Cause:** you batched so aggressively that per-user latency exceeds SLO — throughput you can't sell (Chapter 1.3). Optimizing the wrong metric.
**Fix:** measure and optimize **goodput** (SLO-compliant throughput), not raw throughput; cap batch size to hold TPOT within SLO; shed/deprioritize load you can't serve in SLO rather than accepting it (Chapter 4.6).

---

## 14.5 Silent quality regression from quantization

**Symptom:** you quantized to save cost; benchmarks you happened to check look fine; weeks later users report worse math/code/reasoning.
**Cause:** quantization damage is task-dependent (Chapter 6.5) — chat evals pass while math/code/reasoning degrade, and offline perplexity misses it entirely.
**Fix:** evaluate the **sensitive tasks** (math/code/reasoning/long-context) on the **served, quantized endpoint** against the FP16 baseline before shipping; set an explicit accuracy budget and reject trades that exceed it; keep INT4 away from reasoning-critical products.

---

## 14.6 Cold-start storm

**Symptom:** after a scale-up (or scale-from-zero), a wave of requests times out for 30–90+ seconds.
**Cause:** model load time (Chapter 11.6) — new replicas aren't ready to serve for the load duration, but traffic is already routed to them (or you scaled from zero into a spike).
**Fix:** minimum ≥2 replicas (never scale to zero for latency-sensitive prod); pre-warm replicas before shifting traffic; readiness probes that only pass when the model is loaded *and* warm; scale up *ahead* of predicted load, not reactively into it.

---

## 14.7 Autoscaling that doesn't (or oscillates)

**Symptom:** the fleet doesn't scale up under load, or flaps up and down wastefully.
**Cause:** scaling on CPU/GPU-utilization HPA, which is meaningless for LLM inference (Chapter 11.3) — GPU sits at 100% whether healthy or saturated; or thresholds tuned so tightly the fleet oscillates.
**Fix:** scale on **queue depth / KV utilization / tail-latency-vs-SLO**; add stabilization windows/cooldowns to prevent flapping; account for cold-start lag in scale-up triggers (scale before saturation, not at it).

---

## 14.8 Prefix cache not hitting

**Symptom:** TTFT is high despite lots of shared system prompts / repeated documents; prefill compute is higher than expected.
**Cause:** prefix caching disabled, or **routing sends same-prefix requests to different replicas** so no single replica's cache warms (Chapter 11.4).
**Fix:** enable prefix caching; use **prefix/KV-aware routing** to send shared-prefix traffic to the same replica; verify cache hit rate is actually high in metrics.

---

## 14.9 Runaway reasoning generation

**Symptom:** cost per request spikes; some requests generate tens of thousands of tokens and hog KV/slots; latency for others suffers.
**Cause:** uncapped reasoning/thinking budget (Chapter 8.4) — the model thinks far longer than the query warrants, and long generations starve the batch.
**Fix:** set thinking-budget / reasoning-effort caps; route easy queries to a non-reasoning path; enforce max-output-token limits; alert on output-token counts per request; batch-tier latency-tolerant reasoning.

---

## 14.10 Structured output kills throughput

**Symptom:** enabling JSON/grammar mode tanks throughput at moderate batch.
**Cause:** guided-decoding mask generation not overlapped with GPU work — vLLM degrades at batch ≥8 with guided decoding (Chapter 9.3).
**Fix:** use SGLang (overlaps grammar masking, near-zero throughput hit) for structured-output-heavy workloads; ensure the XGrammar backend is active; simplify overly-complex grammars.

---

## 14.11 The meta-lesson

Nearly every serving failure is one of two things: **you ran out of KV memory because you sized for weights instead of weights-plus-KV-at-real-concurrency**, or **you optimized an average/throughput number and got killed at the tail/goodput.** The prevention for the first is the memory budget (Chapter 2.4) applied at *peak* load; for the second, it's measuring percentiles and goodput under *realistic* concurrency and output length. Load-test with production-shaped traffic (real lengths, real concurrency, real prefix patterns), watch the tail, and size for peak — and serving is far more mechanical than it feels.

Next: essential reading and references.
