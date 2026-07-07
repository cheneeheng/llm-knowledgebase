# Chapter 6 — Quantization for Inference

## 6.1 Conceptual picture

Quantization stores the model's numbers in fewer bits — FP16 → FP8 → FP4/INT4 — trading a little accuracy for a lot of memory and speed. It is the highest-ROI serving optimization after batching, and for two reasons that both trace to Chapter 2: (1) smaller weights mean *fewer bytes streamed from HBM per token*, and since decode is memory-bandwidth-bound, fewer bytes means faster decode; (2) smaller weights free HBM for a bigger KV cache and bigger batch, which is where throughput/cost lives. The often-quoted headline: on a single H100, BF16 might serve ~4 concurrent users at 4K context while INT4 serves ~47 — a **10×+ concurrency swing**, mostly from freed memory. Quantization is not a niche optimization; in 2026 it's the default.

The catch is the accuracy trade, and it's non-uniform: some tasks (chat, summarization) barely notice INT4, while others (math, code, multi-step reasoning) degrade visibly. So quantization is never "set the flag and ship" — it's "set the flag and *measure on your tasks*."

## 6.2 The precision ladder

| Precision | Bits | Memory vs FP16 | Quality hit | Status in 2026 |
|---|---|---|---|---|
| **BF16/FP16** | 16 | 1× | none (baseline) | Leaving performance on the table for pure serving |
| **FP8** | 8 | 0.5× | ~0.5–2% on benchmarks | **The default production precision** on Hopper/Blackwell |
| **INT8** | 8 | 0.5× | small, task-dependent | Solid where FP8 hardware absent; W8A8 kernels mature |
| **FP4 (NVFP4/MXFP4)** | 4 | 0.25× | more pronounced than FP8 | **Emerging** on Blackwell; tooling maturing through 2026 |
| **INT4 (AWQ/GPTQ/GGUF)** | 4 | 0.25× | noticeable; task-dependent | Widely used for VRAM-constrained fit; avoid for math/code/reasoning |

**FP8 is the sweet spot and the default.** The quality drop from calibrated FP8 is minimal (~0.5–2%), it's a config flag in every serving engine, and native FP8 tensor cores on Hopper/Blackwell make it genuinely faster, not just smaller. If your hardware supports FP8, serve in FP8 unless you've measured a quality problem. **FP4** is the frontier: 4× smaller and fast on Blackwell, but calibration tooling is still maturing in early 2026 and the quality hit is more pronounced — expect it to become common through the year, but validate hard today. **INT4** is the escape hatch for fitting a model into constrained VRAM (a 70B on one 48GB card) — accept the quality hit knowingly and keep it away from reasoning-heavy workloads.

## 6.3 What gets quantized: weights, activations, KV cache

Three distinct things, often quantized to different precisions (the "WxAy" notation: W=weights, A=activations):

- **Weights** — the big fixed memory cost; quantizing these is the main VRAM/bandwidth win. Weight-only INT4 (W4A16) keeps activations in FP16 for quality while shrinking the dominant memory term — a popular balance. W4A8 (LiquidGEMM-style kernels) pushes activations to 8-bit for more speed.
- **Activations** — the transient signals; harder to quantize without quality loss (outliers), which is why activation quantization lagged weight quantization. FP8 activations are now robust on modern hardware.
- **KV cache** — covered in Chapter 3; FP8 KV cache is near-free quality and a direct concurrency win. Quantize this almost always on modern hardware.

A common production stack: **FP8 weights + FP8 activations + FP8 KV cache** on Blackwell/Hopper, or **INT4 weights (AWQ) + FP16 activations + FP8 KV** when you need to fit a big model on small VRAM.

## 6.4 The methods (how the bits are chosen)

You'll see these format/method names; here's what to know:

- **AWQ (Activation-aware Weight Quantization)** — protects the salient weight channels (those multiplied by large activations) from quantization error using a calibration dataset. The go-to INT4 weight-only method for quality; widely supported.
- **GPTQ** — layer-wise second-order weight quantization, also calibration-based. Mature, well-supported, similar niche to AWQ; AWQ often edges it on quality, GPTQ has broad tooling.
- **GGUF (llama.cpp)** — the quantization *format* for the llama.cpp/Ollama world, with a family of levels (Q4_K_M, Q5, Q8, etc.). Runs on CPU and GPU; the standard for local/edge (Chapter 5). Distinct from hardware FP4 — it's software INT quantization portable across devices.
- **FP8 / FP4 via TensorRT Model Optimizer (ModelOpt) or native engine flags** — hardware-native float formats; ModelOpt and the serving engines provide calibration for these. FP8 often needs little or no calibration; FP4 needs careful calibration.
- **SmoothQuant, and outlier-handling variants** — techniques to make activation quantization robust by migrating quantization difficulty from activations to weights. Under the hood of many W8A8 paths.

**Calibration** — most sub-FP8 methods need a small representative dataset (a few hundred samples) to choose scales that minimize error. Calibrate on data resembling your *serving traffic*, not a generic corpus, for the best quality-at-bits.

## 6.5 The measurement discipline (non-negotiable)

Because the quality hit is task-dependent and offline metrics miss serving-time effects:

1. **Run your real eval set through the quantized, *served* endpoint** — not just an offline perplexity check. Perplexity barely moves while task accuracy on math/code/reasoning drops; measure the tasks you care about.
2. **Test the sensitive tasks specifically** — math (GSM8K/MATH), code (HumanEval), long-context recall, multi-step reasoning. These reveal quantization damage that chat evals hide.
3. **Compare against the FP16 baseline on the same harness** — like midtraining's forgetting budget, you're measuring a *delta*, and you need the un-quantized number.
4. **Budget the trade explicitly** — decide up front what accuracy drop is acceptable for the concurrency/cost gain, and reject a quantization that exceeds it. A 10× concurrency win that costs 5 points of code accuracy is a bad trade for a coding product and a fine one for a chat toy.

## 6.6 Decisions

1. **Serve in FP8 by default** on Hopper/Blackwell — minimal quality hit, real speedup, one flag. BF16 for pure serving is leaving throughput unclaimed.
2. **Use INT4 (AWQ) to fit a model into constrained VRAM**, knowingly, and keep it away from math/code/reasoning-critical products.
3. **Approach FP4 as the emerging frontier** — great on Blackwell, but calibrate and validate hard in early 2026.
4. **Quantize the KV cache to FP8** as a near-free concurrency win (Chapter 3).
5. **Always measure task accuracy on the served endpoint against the FP16 baseline**, focusing on the sensitive tasks, and reject trades that exceed your explicit accuracy budget.

Next: speculative decoding — the one technique that beats the fundamental one-token-per-memory-read ceiling.
