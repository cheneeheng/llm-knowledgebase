# Chapter 1 — The Big Picture

## 1.1 What serving is

Serving is the practice of turning a trained model into a system that answers requests — many of them, concurrently, at a latency users tolerate and a cost the business survives. It sounds like "just run the model," and for one request on your laptop it is. The discipline appears the moment you have *many* requests of *different* shapes arriving at *unpredictable* times and a *finite* pool of expensive GPUs to serve them from. Everything in this series is about that mismatch.

The core object is the **request**: a prompt in, a completion out. The core resource is **GPU memory bandwidth** (not, as most newcomers assume, compute — Chapter 2 explains why). The core tension is a triangle you can never fully win:

- **Latency** — how fast a single user gets their answer.
- **Throughput** — how many users you serve per GPU per second.
- **Cost** — dollars per million tokens, which is throughput expressed as money.

Push latency down (small batches, dedicated capacity) and throughput/cost suffer. Push throughput up (big batches) and individual latency suffers. The entire craft is finding the point on this triangle that meets your latency SLO at the lowest cost — and every technique in this series is a way to bend the triangle outward so that point is better than it was.

## 1.2 The two phases (the fact everything hinges on)

An LLM answers in two mechanically distinct phases, and confusing them is the root of most serving mistakes:

- **Prefill** — the model reads the entire prompt in one forward pass, building up its internal state (the KV cache, Chapter 3). This is **compute-bound**: lots of tokens processed in parallel, the GPU's matrix units are busy. It determines **time-to-first-token (TTFT)**.
- **Decode** — the model generates the output one token at a time, each token requiring a full forward pass that reads all the model's weights from memory to produce a *single* token. This is **memory-bandwidth-bound**: the GPU's compute sits mostly idle waiting for weights to stream from HBM. It determines **time-per-output-token (TPOT)**, and thus how long a long answer takes.

These two phases want opposite things from the hardware and the scheduler, which is why 2026 serving increasingly *separates* them (chunked prefill, prefill/decode disaggregation — Chapter 4). Internalize this now: **prefill is compute-bound and fast per token; decode is memory-bound and the reason inference is expensive.** Reasoning models, which emit enormous decode phases, are expensive precisely because decode is the bottleneck and they do a lot of it (Chapter 8).

## 1.3 The metrics that matter

Four numbers define a serving system's quality. Learn to state your targets in these terms:

| Metric | What it measures | Driven by | User-visible as |
|---|---|---|---|
| **TTFT** (time to first token) | Latency to the first output token | Prefill speed, queue wait | "How long until it starts answering" |
| **TPOT / ITL** (time per output token / inter-token latency) | Steady-state generation speed | Decode speed (memory bandwidth) | "How fast it types" |
| **Throughput** | Total tokens/sec across all requests | Batch size, hardware, efficiency | Your cost per token |
| **Goodput** | Throughput *that meets SLO* | All of the above under load | What you can actually sell |

**Goodput** is the one newcomers miss: a system can post huge throughput numbers while violating everyone's latency SLO under load — throughput you cannot sell. The right target is the maximum goodput (SLO-compliant throughput) your fleet delivers, and the whole of Chapter 4 (scheduling) and Chapter 11 (autoscaling) is about maximizing goodput, not raw throughput.

Typical 2026 SLOs for an interactive assistant: TTFT under ~200–500ms, TPOT under ~20–50ms (i.e. 20–50+ tokens/sec, comfortably above reading speed). Batch/offline jobs relax TTFT to seconds and optimize purely for throughput/cost (OpenAI's batch API charges ~50% of realtime precisely because relaxed latency lets them batch hard).

## 1.4 The 2026 landscape

Where the field sits as you enter it:

- **Serving engines have consolidated** around three open-source leaders — **vLLM** (the general-purpose default), **SGLang** (throughput and structured-output leader), **TensorRT-LLM** (peak NVIDIA performance) — plus **llama.cpp/Ollama** for local/edge. Any of them gives you continuous batching and paged KV cache out of the box. Chapter 5 picks between them.
- **FP8 is the default production precision** on Hopper/Blackwell hardware, with FP4 emerging on Blackwell (Chapter 6). Serving in BF16 in 2026 is leaving throughput on the table.
- **Inference is memory-bound**, so the hardware conversation is about **HBM capacity and bandwidth**, not TFLOPS — H200 (141GB), B200 (192GB), MI300X (192GB) matter because they fit bigger models and bigger KV caches per GPU (Chapter 10).
- **Reasoning models re-priced inference.** Test-time compute (long chains of thought) turned queries that cost a tenth of a cent into queries that cost tens of cents, and made decode-phase efficiency the whole ballgame (Chapter 8).
- **The production stack matured.** Kubernetes-native serving (KServe, llm-d, the Gateway API Inference Extension, GA Feb 2026) brought KV-cache-aware routing, disaggregated topologies, and scale-to-zero to standard infrastructure (Chapter 11).

## 1.5 The build-vs-buy fork

Before any of this, the first decision: **do you self-host at all?** Calling a frontier API (Claude, GPT, Gemini) means someone else does everything in this series. You self-host when: you need a model no API offers (your fine-tune, an open model you customized), you have privacy/compliance constraints that forbid sending data out, your volume is high enough that per-token API margins exceed your amortized GPU cost, or you need latency/control an API can't give. Chapter 12 does the math. This series assumes you've decided to self-host (or want to understand the economics well enough to decide); if you're purely an API consumer, read Chapters 1, 8, and 12 and skip the systems chapters.

## 1.6 Decisions this series will force

1. **Self-host or call an API?** (Ch 12)
2. **Which serving engine?** (Ch 5)
3. **What precision — BF16, FP8, FP4, INT4?** (Ch 6)
4. **How to batch and schedule — continuous batching alone, chunked prefill, or full disaggregation?** (Ch 4)
5. **Speculative decoding — worth the complexity?** (Ch 7)
6. **What hardware, and how many GPUs per replica?** (Ch 10)
7. **How to autoscale, route, and observe in production?** (Ch 11)
8. **What's your real cost per token, and how do you drive it down?** (Ch 12)

Start with Chapter 2: until you understand *why decode is memory-bound and batching is the master knob*, none of the later optimizations will make sense.
