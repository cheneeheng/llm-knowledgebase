# Chapter 10 — Hardware

## 10.1 Conceptual picture

Choosing inference hardware is a different exercise from choosing training hardware, and the difference is one word: **memory.** Training is compute-bound and you buy FLOPS. Inference decode is memory-bandwidth-bound (Chapter 2) and constrained by memory *capacity* (the model + KV cache must fit — Chapter 2.4), so you buy **HBM bandwidth and HBM capacity**, and raw TFLOPS is almost a distraction. A GPU with mediocre FLOPS but huge fast memory serves LLMs better than the reverse. Internalize that and the hardware landscape sorts itself out.

Two numbers dominate every inference GPU decision:
- **HBM capacity (GB)** — determines whether the model fits, and how much room is left for KV cache (hence batch size, hence throughput/cost). Chapter 2.4's budget.
- **HBM bandwidth (TB/s)** — determines single-stream decode speed (Chapter 2.1) and thus your latency floor.

## 10.2 The 2026 GPU landscape

| GPU | HBM | Bandwidth | Inference niche |
|---|---|---|---|
| **NVIDIA H100** | 80GB HBM3 | 3.35 TB/s | Still the most-deployed inference GPU; unmatched ecosystem/optimization |
| **NVIDIA H200** | 141GB HBM3e | 4.8 TB/s | The sweet spot: same CUDA ecosystem, 1.75× memory → fits bigger models/KV, higher tok/s |
| **NVIDIA B200 (Blackwell)** | 192GB HBM3e | 8 TB/s | 4–5× H100 inference throughput; native FP4; serves 70B FP16 on one GPU with KV headroom |
| **NVIDIA GB200 (Grace-Blackwell)** | 192GB/GPU + NVLink domain | very high | Rack-scale disaggregated topologies; frontier fleets |
| **AMD MI300X** | 192GB HBM3 | ~5.3 TB/s | Strongest non-NVIDIA option; 192GB fits 70B FP16 on one card with ~52GB KV headroom; ROCm maturing |
| **NVIDIA L40S** | 48GB | ~0.86 TB/s | Cost-efficient for <30B models; lower bandwidth caps decode speed |
| **Google TPU v5/v6, AWS Trainium2, Groq LPU** | varies | high | Managed-cloud / specialized; Groq LPU targets ultra-low-latency decode |

The practical hierarchy for 2026: **H100** is the safe, ecosystem-rich default; **H200** is the value pick (more memory, same software, better tok/s per dollar for memory-bound work); **B200** is the performance leader (and the FP4 path); **MI300X** is the credible NVIDIA alternative when its 192GB and price/performance win and ROCm supports your stack. Verify your serving engine + quantization + model combo is well-supported on any non-NVIDIA choice before committing — the software ecosystem gap is the real cost of leaving CUDA.

## 10.3 The fit calculation

The single most useful hardware exercise: **does the model + a useful KV cache fit, and how much batch does that leave?** Worked example for a 70B model:

- **FP16 weights**: 140GB → needs 2× H100 (TP=2), or *one* H200/B200/MI300X (fits with room).
- **FP8 weights**: 70GB → fits on one H100 (80GB) with ~10GB for KV (tight), comfortably on H200/B200/MI300X with large KV.
- **INT4 weights**: 35GB → fits on one 48GB L40S with KV room, or one H100 with huge KV/batch headroom.

The lesson: **quantization (Chapter 6) and GPU memory are the same decision.** You either buy more memory or shrink the model to fit the memory you have — and you want to fit with *enough headroom for a healthy KV cache*, because a model that fits with only 2GB of KV room serves a tiny batch and terrible economics. Size for `weights + target_KV_for_your_concurrency + overhead`, not just for weights.

The parallelism corollary (Chapter 7.5): if the model doesn't fit one GPU even quantized, use **tensor parallelism within a node** (TP=2/4/8 over NVLink) to pool memory and bandwidth; use **pipeline parallelism across nodes** only when it exceeds a single node. Use the *least* parallelism that fits with KV headroom — each added GPU per replica adds communication cost.

## 10.4 Matching hardware to workload

- **Small models (<30B)** — L40S or a single H100/H200 for cost efficiency; these fit comfortably and you're optimizing $/token.
- **Mid models (30–70B)** — H100 (with FP8/INT8) for cost, or H200/MI300X/B200 when you need FP16 quality or long-context KV headroom on a single card.
- **Large models (70–405B+)** — MI300X or H200/B200 for memory capacity; multi-GPU TP; MoE models want expert parallelism and rack-scale interconnect (GB200).
- **Reasoning models (Chapter 8)** — prioritize **bandwidth and KV capacity** above all; H200/B200/MI300X; the long decode makes bandwidth the latency floor and KV capacity the concurrency cap.
- **Long-context serving** — capacity-first (KV grows with length); H200/B200/MI300X, plus KV offloading (Chapter 3).
- **Local/edge** — consumer GPUs (RTX 4090/5090), Apple Silicon (Metal), or CPU via llama.cpp/GGUF (Chapter 5).

## 10.5 Buy, rent, or serverless

Same three-way choice as the training series' procurement, shifted to inference's always-on profile:

- **Reserved/owned GPUs** — cheapest per hour at high, steady utilization; you eat idle time. Right for predictable base load.
- **On-demand cloud** — flexible, pricier per hour; right for variable load and bursts. H100 stabilized around ~$2–3.50/hr across providers in early 2026.
- **Serverless / per-token inference APIs (for open models)** — someone else runs the fleet and you pay per token; zero ops, highest per-token price, right for spiky or low volume and for avoiding the whole of this series. The build-vs-buy math is Chapter 12.

The inference-specific wrinkle: **utilization is the whole game economically** (Chapter 12). An owned GPU at 20% utilization is more expensive per token than on-demand; the autoscaling and routing of Chapter 11 exist to keep utilization high enough that owning pays off.

## 10.6 Decisions

1. **Choose inference GPUs by HBM capacity and bandwidth, not TFLOPS** — decode is memory-bound.
2. **Default to H100 (ecosystem) or H200 (value); B200 for peak performance and FP4; MI300X as the NVIDIA alternative** when its memory/price wins and your stack is ROCm-supported.
3. **Do the fit calculation**: size for `weights + KV-for-your-concurrency + overhead`, and treat quantization and GPU-memory as one decision — fit with KV headroom, not just weights.
4. **Use the least parallelism that fits with KV headroom** — TP within a node, PP only across nodes, EP for MoE; replicate to scale throughput.
5. **Match the tier to the workload** (small→L40S, reasoning/long-context→bandwidth+capacity), and choose buy/rent/serverless by load shape and utilization.

Next: the production lifecycle — turning tuned replicas into a reliable, autoscaled, observable fleet.
