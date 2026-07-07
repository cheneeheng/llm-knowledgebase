# Chapter 5 — Serving Engines

## 5.1 Conceptual picture

A serving engine is the software that takes your checkpoint and turns it into a high-throughput, continuously-batched, paged-KV, quantization-aware, OpenAI-API-compatible endpoint. You do not write one. You choose one, configure it for your workload, and operate it. The good news: the field has consolidated to a handful of mature options that all share the fundamentals (continuous batching, paged KV, prefix caching, tensor parallelism, FP8). The choice is about which *architectural bet* matches your priorities — raw performance, flexibility, structured output, or edge simplicity.

The engines make "radically different architectural bets that serve the same purpose," and the right one depends on your scenario, not on benchmarks in the abstract. Below is the 2026 map.

## 5.2 The contenders

**vLLM — the general-purpose default.**
The most widely deployed open engine, the reference implementation of PagedAttention, and the safest starting point. Excellent throughput across a broad range of batch sizes and models, the largest ecosystem, fastest support for new model architectures, native multi-LoRA, broad quantization support, and an OpenAI-compatible server out of the box. Its ceiling: the Python scheduler adds per-step overhead (~200–500 μs per scheduling cycle at high concurrency), so at extreme scale it trails the compiled engines. **Pick vLLM when** you want the broadest model support, the most community knowledge, and strong performance without hardware lock-in — which is most teams, most of the time. It is the correct default; deviate only for a specific reason below.

**SGLang — throughput and structured-output leader.**
Built around RadixAttention (aggressive automatic prefix reuse via a radix tree) and a scheduler that overlaps CPU work (including grammar-mask generation) with GPU inference. Consequences: best-in-class throughput on workloads with heavy prefix sharing (multi-turn, RAG, agents), and structured-output enforcement that "barely impacts throughput" where vLLM degrades at batch ≥8 (Chapter 9). Native, efficient multi-LoRA at high adapter counts. **Pick SGLang when** your workload is prefix-heavy (agents, RAG, multi-turn) or structured-output-heavy (lots of JSON/tool calls), or when you're chasing maximum throughput and willing to run a slightly less universal engine.

**TensorRT-LLM — peak NVIDIA performance.**
Compiles the model into a hardware-specific optimized engine with aggressive kernel fusion. In high-concurrency settings it can outperform vLLM by 30–50% on total throughput and delivers the lowest latency on NVIDIA hardware — at the cost of a compilation/build step per model+GPU+config, less flexibility, NVIDIA-only lock-in, and more operational friction. **Pick TensorRT-LLM when** you're locked to NVIDIA, running a stable set of models at large scale, and the last 30–50% of performance is worth the engineering and inflexibility. Often deployed under NVIDIA's Triton/Dynamo serving layer.

**llama.cpp / Ollama — local and edge.**
The CPU/consumer-GPU path. llama.cpp runs GGUF-quantized models on laptops, CPUs, Macs (Metal), and small GPUs; Ollama wraps it in a friendly local server with model management and (as of 2026) native tool calling. Not for high-concurrency datacenter serving — throughput and batching are far below the datacenter engines — but unbeatable for local development, on-device deployment, air-gapped environments, and single-user workloads. **Pick these when** you're serving one or few users locally, on modest hardware, or embedding a model in a desktop app.

**TGI (Text Generation Inference) and others.**
Hugging Face's TGI remains a solid, well-integrated option especially inside the HF ecosystem; it pioneered continuous batching in the open. NVIDIA Dynamo (the successor serving framework layering disaggregation over engines) and Ray Serve (for orchestrating multi-model/multi-node topologies) are orchestration layers you may run *above* an engine rather than alternatives to it.

## 5.3 The selection table

| Scenario | Engine | Why |
|---|---|---|
| Default / broad model support / getting started | **vLLM** | Widest support, biggest ecosystem, strong all-round performance |
| Prefix-heavy (agents, RAG, multi-turn) | **SGLang** | RadixAttention prefix reuse |
| Structured output / heavy JSON & tool calls | **SGLang** | Overlapped grammar masking, near-zero throughput hit |
| Max NVIDIA performance, stable model set, large scale | **TensorRT-LLM** | Kernel fusion, +30–50% throughput; accept the build step |
| Local / edge / on-device / single user | **llama.cpp / Ollama** | GGUF, CPU/consumer-GPU, simple |
| Multi-node disaggregated production | engine + **Dynamo / llm-d** | Orchestration and disaggregation over a base engine |

## 5.4 What all of them give you

Regardless of choice, expect: an **OpenAI-compatible HTTP API** (so client code is portable across engines and even to/from hosted APIs — a real hedge against lock-in), continuous batching, paged + prefix-cached KV, tensor parallelism across GPUs, FP8 and common weight-quantization formats, streaming, and multi-LoRA. Because the API is standardized, **you can and should benchmark two engines on your own traffic and switch** — the client barely changes. Do not choose on someone else's benchmark; the winner depends on your prompt/output length distribution and concurrency.

## 5.5 The benchmarking discipline

Before committing, run **your** workload through the candidates:
- Replay a representative sample of *your* traffic (real prompt lengths, output lengths, concurrency, prefix-sharing pattern) — synthetic uniform-length benchmarks lie.
- Measure the full quartet: TTFT, TPOT, throughput, and **goodput at your SLO** under realistic concurrency, not just peak throughput.
- Sweep the knobs that matter (batch/concurrency cap, chunked-prefill chunk size, quantization, prefix caching on/off) — an engine's default config is rarely optimal for your traffic.
- Watch quality, not just speed, when quantization is involved (Chapter 6) — run your eval set through the served endpoint, because serving-time quantization can regress quality in ways offline eval misses.

## 5.6 Decisions

1. **Start with vLLM** unless you have a specific reason from the table — it's the correct default and the deepest ecosystem.
2. **Switch to SGLang** for prefix-heavy or structured-output-heavy workloads, or to chase throughput.
3. **Adopt TensorRT-LLM** only when NVIDIA-locked, at scale, with a stable model set, and the extra 30–50% justifies the build step and rigidity.
4. **Use llama.cpp/Ollama** for local, edge, and single-user; don't try to run a datacenter on them.
5. **Rely on the OpenAI-compatible API as your portability hedge** and benchmark candidates on your *own* replayed traffic and SLO before committing.

Next: quantization — the lever that shrinks the model to fit more batch and stream fewer bytes, and the accuracy trade you must measure when you pull it.
