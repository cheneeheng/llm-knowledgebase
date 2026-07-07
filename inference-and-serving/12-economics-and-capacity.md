# Chapter 12 — Economics and Capacity

## 12.1 Conceptual picture

Every technique in this series exists to move one number: **cost per token** (or its inverse, tokens per dollar). This chapter assembles them into the financial picture — how to compute your real cost per token, why utilization dominates it, how to plan capacity, and the build-vs-buy decision that sits over the whole series. The governing insight: **serving cost is a utilization problem wearing a technology costume.** The same GPU can cost 10× more per token at 10% utilization than at 80%, so most of the economic game is keeping expensive hardware busy with SLO-compliant work — which is why batching (Chapter 2), autoscaling (Chapter 11), and routing exist.

## 12.2 Computing cost per token

The fundamental equation:

```
cost_per_token = GPU_cost_per_hour / (tokens_per_second × 3600 × utilization)
```

Every lever in this series moves a term:
- **`tokens_per_second`** — raised by batching (Ch 2), quantization (Ch 6), speculative decoding (Ch 7), the right engine (Ch 5), and the right hardware (Ch 10). This is the throughput numerator.
- **`utilization`** — raised by autoscaling on the right metrics (Ch 11), routing, and matching capacity to load. The most-neglected term and often the biggest lever.
- **`GPU_cost_per_hour`** — set by buy/rent/serverless choice (Ch 10) and by hardware selection.

Work a concrete example: an H100 at $3/hr serving a batched 8B model at, say, 2,500 tokens/sec at 70% utilization → `3 / (2500 × 3600 × 0.7) ≈ $0.48 per million tokens`. The same GPU at 15% utilization → ~$2.20 per million tokens. Same hardware, same model, **4.5× cost difference from utilization alone.** This is why the FinOps loop (12.5) obsesses over utilization.

## 12.3 Why utilization dominates

Inference load is *bursty and unpredictable* — traffic spikes, diurnal patterns, idle nights — but GPUs cost the same whether busy or idle. The tension:

- **Provision for peak** → you meet SLO but pay for idle capacity most of the time (low utilization, high cost/token).
- **Provision for average** → high utilization but you violate SLO during spikes (low goodput, lost/failed requests).

The resolution is *elastic* capacity: autoscale replicas up for spikes and down for troughs, keeping utilization high while meeting SLO (Chapter 11) — bounded below by cold-start-driven minimum replicas and above by your budget. Where autoscaling can't smooth it (hard latency SLO, slow cold starts), you provision headroom and accept some idle as insurance. The reasoning-model wrinkle (Chapter 8): long, memory-heavy requests force small batches and *low throughput-per-GPU*, so reasoning traffic has structurally worse economics — plan for it.

## 12.4 Batch vs realtime pricing

A major lever: **latency-tolerant work is far cheaper to serve.** Realtime interactive serving must keep batches from getting too big (latency) and capacity ready for spikes (idle headroom). Offline/batch work can be packed into maximally full batches, run on cheaper/preemptible capacity, and time-shifted to fill troughs — which is why OpenAI and others price batch APIs at ~50% of realtime. If any of your workload tolerates minutes-to-hours of latency (report generation, bulk classification, offline reasoning, embeddings backfill), **route it to a batch tier** for roughly half the cost. Segmenting traffic by latency tolerance is one of the highest-ROI economic moves and costs almost nothing to implement.

## 12.5 The FinOps loop

Running inference economically is a continuous loop, not a one-time setup:

1. **Measure** real cost per token per model/endpoint (Chapter 11 observability), broken down by workload.
2. **Attribute** cost to workloads/customers/features — know what's expensive and why (usually: a low-utilization endpoint, a reasoning model with uncapped thinking, or a model bigger than the task needs).
3. **Optimize** the biggest line item — often *not* a systems tweak but a product decision: use a smaller model for an easy task, cap reasoning budgets (Chapter 8), route easy queries to a cheaper path, consolidate low-traffic endpoints, quantize harder.
4. **Right-size the model to the task** — the biggest waste in 2026 inference is serving a frontier-size model for a task an 8B handles. Distillation (post-training series) and model selection are cost levers: the cheapest token is the one a smaller model produces at equal quality.
5. **Repeat** — traffic, models, and hardware prices all move; re-measure quarterly.

## 12.6 Capacity planning

To size a fleet:

1. **Characterize the workload** — request rate (peak and average), prompt-length and output-length distributions, concurrency, prefix-sharing, latency SLO. Reasoning vs chat vs RAG have wildly different profiles.
2. **Benchmark goodput per replica** on your traffic (Chapter 5.5) — SLO-compliant tokens/sec for one tuned replica on your chosen hardware.
3. **Compute replicas needed** = peak SLO-load ÷ per-replica goodput, plus headroom for cold-start latency and failures (Chapter 11).
4. **Layer in autoscaling bounds** — minimum replicas (cold-start floor, ≥2), maximum (budget ceiling), scaling triggers (queue/KV/latency metrics).
5. **Model the cost** at expected utilization and validate it against the business case *before* committing hardware — this is where build-vs-buy gets decided.

## 12.7 Build vs buy, decided

The decision hanging over the whole series. Self-host (build) vs consume an API (buy):

**Buy (hosted API — Claude/GPT/Gemini, or serverless open-model endpoints) when:**
- Volume is low, spiky, or unpredictable (you'd run at low utilization → per-token API price beats your idle-heavy owned cost).
- You want zero ops and fastest time to market.
- You need frontier capability you can't reproduce.
- Your team is small and inference isn't your core competency.

**Build (self-host) when:**
- Volume is high and steady enough that amortized GPU cost per token beats API pricing at your achievable utilization — the crossover is the core calculation, and it favors building only above meaningful, predictable scale.
- You need a specific model (your fine-tune, a customized open model, multi-LoRA per customer — Chapter 9) no API offers.
- Privacy/compliance/data-residency forbids sending data to a third party (Chapter cross-cutting series).
- You need latency, throughput, or control an API can't provide.

The honest default: **most teams should buy until the math clearly says build.** Self-hosting is this entire series' worth of engineering; do it when volume, model-specificity, or compliance makes it necessary, not by default. And it's not all-or-nothing — many mature setups **route by request**: cheap/private/high-volume traffic to self-hosted open models, hard/frontier traffic to an API, batched work to a batch tier. Segment and route.

## 12.8 Decisions

1. **Compute real cost per token** as `GPU_$/hr ÷ (tok/s × 3600 × utilization)` and treat **utilization as the dominant, most-neglected lever**.
2. **Segment traffic by latency tolerance** and route latency-tolerant work to a batch tier for ~half the cost.
3. **Run the FinOps loop** — measure, attribute, optimize the biggest line item (often a product/model-size decision, not a systems tweak), right-size the model to the task.
4. **Plan capacity from characterized workload → per-replica goodput → replicas + autoscaling bounds**, and model the cost before buying hardware.
5. **Buy (API) by default; build (self-host) when volume, model-specificity, or compliance makes the math favor it** — and route by request across self-hosted, API, and batch tiers rather than choosing one globally.

Next: playbooks — the whole series assembled into concrete recipes by scale.
