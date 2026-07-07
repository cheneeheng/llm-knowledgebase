# Serving an LLM: The Complete Inference Playbook

*From a trained checkpoint to a production endpoint — the anatomy of inference, KV cache, batching, serving engines, quantization, speculative decoding, reasoning-model serving, hardware, and the economics that decide whether any of it is affordable.*

*Compiled July 2026. The third act after [Training an LLM From Scratch](../pretraining-from-scratch/README.md), [Midtraining](../midtraining-and-continued-pretraining/README.md), and [Post-Training](../post-training-from-scratch/README.md). Assumes you have a model (trained or downloaded) and now have to serve it to users at a cost that makes sense. Assumes you know what a transformer is; assumes nothing about GPUs-as-a-service or serving systems.*

---

## Why this series exists

Training gets the papers; inference gets the bill. **Inference is roughly two-thirds of all AI compute in 2026, up from about a third in 2023**, and the fraction is still climbing because reasoning models emit 5–50× more tokens per answer than the chat models they replaced. The training series produce an artifact; this series is about the far larger, ongoing cost of *running* that artifact — and about the fact that a model which is excellent and unservably expensive is, commercially, a model you do not have.

Serving is its own engineering discipline with almost no overlap in day-to-day concerns with training. Training is throughput over months on a fixed job; serving is latency and cost over a stream of unpredictable requests, forever. The bottleneck flips from compute to **memory bandwidth**; the unit of work flips from a step to a **request**; the thing you optimize flips from loss to a three-way tension between **latency, throughput, and cost** that you can never fully win. This series is about navigating that tension.

## How to read this report

Every chapter is **layered**, like the companion series: it opens with the *conceptual picture* — what the thing is and why it exists — then descends into the *practitioner layer*: concrete latency numbers, memory math, config flags, and the specific decisions you'll face. If you want the map, read the opening of each chapter and the "Decisions" summaries. If you are about to size a GPU fleet or pick a serving stack, read everything.

A recurring theme: **inference is an exercise in filling GPU memory bandwidth without blowing the latency budget.** Every technique — batching, KV-cache paging, quantization, speculative decoding, disaggregation — is ultimately a way to do more useful work per unit of the scarce resource (HBM bandwidth) while keeping time-to-first-token and time-per-token inside what users tolerate. Get the mental model of *why inference is memory-bound* (Chapter 2) and every later chapter follows from it.

## Chapters

| # | File | Contents |
|---|---|---|
| 1 | [The Big Picture](01-big-picture.md) | What serving is, the two phases, the three metrics, the latency/throughput/cost triangle, the 2026 landscape |
| 2 | [The Anatomy of Inference](02-inference-anatomy.md) | Prefill vs decode, why decode is memory-bound, arithmetic intensity, the roofline, batch size as the master knob |
| 3 | [KV Cache and Memory](03-kv-cache-and-memory.md) | What the KV cache is, PagedAttention, prefix caching, KV quantization, offloading, why GQA/MLA exist |
| 4 | [Batching and Scheduling](04-batching-and-scheduling.md) | Continuous batching, chunked prefill, prefill/decode disaggregation, the scheduler, SLO-aware serving |
| 5 | [Serving Engines](05-serving-engines.md) | vLLM, SGLang, TensorRT-LLM, TGI, llama.cpp/Ollama — architecture bets and which to pick when |
| 6 | [Quantization for Inference](06-quantization-for-inference.md) | FP8/FP4/INT4, AWQ/GPTQ/GGUF, calibration, the accuracy trade, weight vs KV vs activation quant |
| 7 | [Speculative Decoding and Parallelism](07-speculative-decoding-and-parallelism.md) | Draft models, EAGLE/Medusa/n-gram, acceptance rates, tensor/pipeline parallelism for serving |
| 8 | [Serving Reasoning Models](08-serving-reasoning-models.md) | Long outputs, test-time compute economics, why reasoning re-priced inference, thinking-budget control |
| 9 | [Structured Output and Adapters](09-structured-output-and-adapters.md) | Constrained decoding, XGrammar, function/tool calling, JSON mode, multi-LoRA serving |
| 10 | [Hardware](10-hardware.md) | The 2026 inference GPU landscape, memory-bound sizing, H100/H200/B200/MI300X/TPU, single-GPU-fit math |
| 11 | [Production Lifecycle](11-production-lifecycle.md) | Kubernetes, autoscaling on the right metrics, routing, observability, cold starts, canaries, SLOs |
| 12 | [Economics and Capacity](12-economics-and-capacity.md) | Cost per token, utilization, batch vs realtime, capacity planning, build vs buy, the FinOps loop |
| 13 | [Playbooks by Scale](13-playbooks-by-scale.md) | Concrete recipes: local single-GPU, one-node API, multi-node production, frontier reasoning fleet |
| 14 | [Failure Modes](14-failure-modes.md) | OOM under load, KV thrash, latency cliffs, quantization quality regressions, cold-start storms |
| 15 | [Essential Reading and References](15-essential-reading-and-references.md) | The must-read sources, ordered, plus the full source list |

## Scope

This series covers everything from "I have a checkpoint" to "I have a production endpoint at a defensible cost." It covers open-model self-hosting in depth (the case where you control the stack) and treats API consumption (calling someone else's endpoint) as the build-vs-buy alternative in Chapter 12. Training, midtraining, and post-training are out of scope — this is what happens after. Edge/on-device inference is touched in Chapters 5 and 10 (llama.cpp, GGUF) but a full mobile/embedded treatment belongs to the cross-cutting series.

## The through-line

Serving rewards **matching the system to the workload**. There is no globally best serving configuration — there is the one that fits *your* request distribution (prompt lengths, output lengths, concurrency, latency SLO, traffic shape) and *your* budget. The teams that win here are not the ones running the fanciest engine; they are the ones who measured their actual traffic, sized memory to it, chose the batching and quantization that hit their SLO at the lowest cost, and instrumented the fleet so they know their real cost-per-token and can watch it. Measure the workload, fill the bandwidth, guard the tail latency, and never quantize or disaggregate without proving it paid off on your traffic.
