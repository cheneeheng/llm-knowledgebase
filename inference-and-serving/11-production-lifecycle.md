# Chapter 11 — Production Lifecycle

## 11.1 Conceptual picture

A tuned replica on one node is a demo. A production service is many replicas behind routing, autoscaling on the right signals, instrumented so you know your latency and cost in real time, deployed so you can ship a new model without an outage, and resilient to the failure modes of GPUs-as-a-service. This chapter is the operational layer — mostly standard SRE/platform engineering, but with several LLM-specific twists that trip up teams applying web-service habits to GPU inference. The twists are where the value is.

## 11.2 The Kubernetes-native stack

By 2026 the standard production substrate is Kubernetes with LLM-aware extensions:

- **KServe / vLLM-on-K8s / Ray Serve** — deploy and manage inference replicas as scalable services.
- **llm-d** — a Kubernetes-native distributed-inference stack (disaggregation, hierarchical KV offloading, cache-aware routing, scale-to-zero) validated at large B200 topologies.
- **The Gateway API Inference Extension (GA Feb 2026)** — the important one: it adds **model-aware routing, KV-cache-aware scheduling, and traffic splitting by model name** to standard Kubernetes ingress. This is the piece that makes LLM routing (11.4) first-class infrastructure rather than a bespoke proxy.

You don't need all of it. A single-model, moderate-scale service can be a vLLM Deployment behind a service with custom-metric autoscaling. The disaggregated, KV-aware, scale-to-zero machinery is for large multi-model fleets. Match the stack to your scale (Chapter 13's playbooks).

## 11.3 Autoscaling on the right metrics

**The single most important operational lesson, and the most common mistake:** the default Kubernetes autoscaler (CPU/memory HPA) is *useless* for LLM inference. "GPU utilization sits at 100% whether your instance is efficiently serving 50 concurrent requests or saturated and dropping them" — so scaling on CPU or even GPU-utilization tells you nothing about whether you're meeting SLO. Scale on **inference-specific signals**:

- **Queue depth / waiting requests** — the clearest overload signal; requests waiting means you need capacity now.
- **KV-cache utilization** — when KV cache fills, you can't admit more requests regardless of compute; a leading indicator of concurrency saturation.
- **TTFT / TPOT percentiles vs SLO** — scale up when tail latency approaches the SLO, not when a coarse resource metric moves.
- **Batch size / running requests** — how full the batch is relative to capacity.

Configure HPA (or KEDA, or the Inference Extension's predicted-latency scheduling) on these custom metrics. Scaling on the wrong metric is why "deploying vLLM on Kubernetes without proper GPU scheduling leads to 40% idle cost and request queuing under load."

## 11.4 Routing

With multiple replicas (and possibly multiple models/adapters), routing decides which replica serves each request, and LLM-aware routing beats round-robin substantially:

- **KV-cache-aware / prefix-aware routing** — send requests that share a prefix (same system prompt, same conversation, same RAG document) to the *same* replica so they hit its warm prefix cache (Chapter 3). This can dramatically cut TTFT and prefill compute — llm-d reports order-of-magnitude TTFT reduction vs round-robin. The highest-value routing decision for prefix-heavy traffic.
- **Model/adapter-aware routing** — route by model name or LoRA adapter to the replica that has it loaded (Chapter 9), avoiding adapter cold-loads.
- **Load/latency-aware routing** — send to the replica with the shortest queue / best predicted latency, not blindly round-robin.
- **Traffic splitting** — by model name for A/B tests and canary rollouts (11.6).

## 11.5 Observability

You cannot run an inference fleet you can't see. Instrument, at minimum:

- **The four metrics (Chapter 1)** — TTFT, TPOT, throughput, and goodput — as percentiles (p50/p95/p99), not averages. Tail latency is what users feel and what SLOs are written against.
- **Cost per token / per request** (Chapter 12) — the business metric; derived from throughput and GPU cost. Watch it like a latency SLO.
- **KV-cache utilization, batch fullness, queue depth** — the capacity signals that drive autoscaling and predict saturation.
- **Token counts** — input/output tokens per request, especially for reasoning models where output length *is* the cost (Chapter 8). Alert on runaway generation.
- **Quality signals** — sampled output logging, structured-output validity rate, refusal/error rates, and (where possible) online quality evals. Serving-time quality regressions (a bad quantization, a model swap) are silent without this.
- **Errors and saturation** — OOMs, preemptions, rejected/queued requests, replica health.

## 11.6 Deployment and rollout

- **Cold starts are the LLM-specific deployment pain.** Loading an 8B model takes 30–90 seconds (much longer for big models) — far slower than a stateless web service. Consequences: keep a **minimum replica count** (≥2 for production) so you're never at zero when traffic arrives; be cautious with scale-to-zero (great for cost on spiky/dev traffic, painful for latency on the first request after idle); pre-warm replicas before shifting traffic to them.
- **Canary / blue-green rollouts** — never swap a model in place. Route a small traffic percentage to the new model/version, watch the four metrics *and* quality signals, then ramp. A new checkpoint or quantization can regress quality invisibly; the canary with quality monitoring is your safety net (same discipline as the training series' "measure against baseline").
- **Model versioning** — pin and track exactly which checkpoint, quantization, and engine version is serving; reproducibility matters when you're debugging a quality regression.

## 11.7 Reliability

- **Health checks that reflect real serving** — a replica whose KV cache is thrashing or whose queue is exploding is "up" by a TCP check but failing users; health-check on serving metrics.
- **Graceful degradation under overload** — shed or queue load to protect SLO for admitted requests (Chapter 4.6); a degraded-but-SLO-compliant service beats one that accepts everything and misses every deadline.
- **Redundancy** — active-active replicas across zones for HA; the disaggregated stacks add transport resilience for the KV-transfer path.
- **Spot/preemptible resilience** — if you serve on cheaper preemptible GPUs, design for eviction (drain, checkpoint in-flight requests, fast reschedule).

## 11.8 Decisions

1. **Autoscale on inference-specific metrics** — queue depth, KV utilization, tail-latency-vs-SLO — never CPU/GPU-utilization HPA.
2. **Use prefix/KV-aware routing** for prefix-heavy traffic (the biggest TTFT win) and model/adapter-aware routing for multi-model fleets; adopt the Gateway API Inference Extension where it fits.
3. **Instrument the four metrics as percentiles, plus cost-per-token, KV/queue signals, token counts, and quality** — tail latency and silent quality regressions are what hurt you.
4. **Plan for cold starts** — minimum ≥2 replicas, cautious scale-to-zero, pre-warming; **roll out via canary with quality monitoring**, never in-place swaps.
5. **Degrade gracefully under overload** and build zone/preemption redundancy sized to your availability target.

Next: economics — turning all of this into a defensible cost per token, and knowing when to build versus buy.
