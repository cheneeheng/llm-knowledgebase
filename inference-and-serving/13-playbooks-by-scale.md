# Chapter 13 — Playbooks by Scale

Concrete end-to-end serving recipes assembling the series. Each is a starting point to adapt to your traffic, not a law. Read Chapters 2–12 for the *why* behind each choice.

---

## 13.1 Local / single user — a model on your machine

**Goal:** run an open model locally for development, privacy, or embedding in a desktop app.

- **Engine:** Ollama (friendly) or llama.cpp (control). GGUF quantization.
- **Model + quant:** pick a model that fits your RAM/VRAM at Q4_K_M or Q5 (Chapter 6); an 8B fits comfortably on a laptop, a 70B needs a big Mac or a 24–48GB GPU at INT4.
- **Hardware:** consumer GPU (RTX 4090/5090), Apple Silicon (Metal), or CPU (slow but works).
- **What you skip:** batching, autoscaling, disaggregation — single-user serving doesn't need them.
- **Tool calling / JSON:** Ollama's native tool calling + constrained decoding for structured output.
- **When this is enough:** local dev, air-gapped/offline, single-user apps, prototyping.

---

## 13.2 One node — a small production API

**Goal:** serve one open model to real users/customers behind an API on a single multi-GPU node.

- **Engine:** vLLM (default) — OpenAI-compatible server. SGLang if prefix-heavy or structured-output-heavy.
- **Model + quant:** FP8 on Hopper/Blackwell (near-free quality, Chapter 6); size so weights + KV-for-your-concurrency fit with headroom (Chapter 10 fit math).
- **Batching:** continuous batching (default) + **chunked prefill** on (mixed prompt lengths). **Prefix caching** on (system prompts, multi-turn).
- **Hardware:** one H100/H200 for ≤70B (with quant); TP=2/4 within the node if it doesn't fit one card.
- **Serving features:** FP8 KV cache; multi-LoRA if you serve customizations; constrained decoding for JSON/tools.
- **Ops:** run ≥2 replicas (cold-start floor) even at this scale if uptime matters; basic observability (the four metrics as percentiles + cost/token); canary new models.
- **Cost:** benchmark goodput on your traffic; compute cost/token at real utilization (Chapter 12). Compare to an API before scaling up.

---

## 13.3 Multi-node — a production fleet

**Goal:** serve one or several models at scale with SLOs and elastic load.

- **Stack:** Kubernetes + vLLM/SGLang replicas; KServe or Ray Serve; the Gateway API Inference Extension for model/KV-aware routing.
- **Batching/scheduling:** continuous batching + chunked prefill per replica; **prefix/KV-aware routing** to hit warm caches (biggest TTFT win); priority scheduling if mixing realtime + batch.
- **Autoscaling:** on **queue depth / KV utilization / tail-latency-vs-SLO** (never CPU HPA); min ≥2 replicas, max = budget; pre-warm before traffic shift.
- **Quantization:** FP8 weights + FP8 KV as default; INT4 (AWQ) where memory-constrained; validate quality on the served endpoint against FP16 baseline.
- **Hardware:** H100/H200/MI300X sized to model + KV; replicate whole replicas to scale throughput; TP within node, PP only across nodes.
- **Observability:** four metrics (p50/p95/p99), cost/token per endpoint, KV/queue/token-count signals, quality sampling; alert on tail latency and validity-rate regressions.
- **Rollout:** canary + quality monitoring, never in-place swaps; pinned model/quant/engine versions.
- **Economics:** FinOps loop; segment latency-tolerant traffic to a batch tier (~half cost); right-size models to tasks.

---

## 13.4 Frontier — a reasoning-model fleet at scale

**Goal:** serve large reasoning and/or MoE models at high volume with strict SLOs — the hardest case.

- **Stack:** llm-d / NVIDIA Dynamo over vLLM/SGLang/TensorRT-LLM; **prefill/decode disaggregation** (Chapter 4) with a **decode-heavy pool** (reasoning is decode-dominated, Chapter 8); hierarchical KV offloading.
- **Hardware:** B200/GB200 rack-scale or MI300X; NVLink domains for TP and expert parallelism (MoE); bandwidth + KV capacity prioritized (Chapter 10).
- **Reasoning specifics (Chapter 8):** aggressive FP8 KV cache; **speculative decoding** (EAGLE) for the long decodes; **thinking-budget controls** and a **difficulty router** (easy queries → cheaper/non-reasoning path) as the top cost levers; batch aggressively (latency-tolerant).
- **Quantization:** FP8 standard; evaluate FP4 on Blackwell for the largest models (validate hard — math/reasoning are sensitive, Chapter 6).
- **Structured output:** SGLang for high-throughput constrained decoding if tool/JSON-heavy (Chapter 9).
- **Scheduling:** SLO-aware admission and priority/preemption across realtime and batch classes; scale prefill and decode pools independently by traffic profile.
- **Economics:** utilization obsession (Chapter 12); reasoning has structurally worse throughput/GPU, so routing easy traffic away and capping thinking budgets dominate all systems optimizations; batch-tier the offline reasoning.
- **Ops:** active-active HA across zones, KV-transport resilience, predicted-latency scheduling, robust cold-start/pre-warm strategy for slow-loading large models.

---

## 13.5 The cross-cutting checklist

Before any production serving deployment, regardless of scale:

- [ ] Workload characterized (rate, prompt/output length distributions, concurrency, prefix-sharing, SLO).
- [ ] Single-stream decode speed estimated (`bandwidth / model_bytes`) — sanity-check your latency promises.
- [ ] Fit math done: weights + KV-for-concurrency + overhead ≤ HBM, with headroom.
- [ ] Engine chosen and **benchmarked on your replayed traffic** (goodput at SLO, not peak throughput).
- [ ] Precision chosen and **quality-validated on the served endpoint** vs FP16 baseline (sensitive tasks).
- [ ] Continuous batching + chunked prefill + prefix caching on.
- [ ] Autoscaling on inference metrics (queue/KV/latency), min ≥2 replicas.
- [ ] Observability: four metrics as percentiles + cost/token + quality sampling.
- [ ] Canary rollout with quality monitoring; pinned versions.
- [ ] Cost/token computed at real utilization; build-vs-buy and batch-tiering considered.

Next: failure modes — the specific ways serving systems break under real load.
