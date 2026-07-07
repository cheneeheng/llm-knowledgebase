# Chapter 3 — Small and On-Device Models

## 3.1 Conceptual picture

The dominant surprise of 2025–2026 is that **small won more than anyone expected.** A well-distilled, well-specialized 2–4B model can beat a 671B general model *on a specific task* — a 2.6B SLM outperforming DeepSeek-R1's full 671B MoE on targeted enterprise reasoning is a documented 2026 result. This inverts the "bigger is better" intuition for *deployment*: the giant model is the *teacher* and the *fallback*, but the thing you ship is often small. Small models are cheaper (Chapter inference-12), faster, private (they run on-device, no data leaves), and — when specialized — competitive or better on their target task. This chapter is why small won, how to build small models (distillation), and how to deploy them at the edge.

The reframing: model size is a *deployment* decision driven by task and constraints, not a *quality* decision. The question is never "how big can we make it" but "what's the *smallest* model that clears the task's quality bar" — because every parameter you don't deploy is cost, latency, and privacy you save (Chapter 1.4, right-sizing).

## 3.2 Why small won

Several forces converged:

- **Specialization beats generality per-task** (midtraining series Ch 4, the size-vs-specialization frontier). A small model focused on one task, with the right data, doesn't need the giant model's breadth. Most *products* are narrow.
- **Distillation got good** (3.3) — you can transfer a large model's competence on a task into a small one efficiently, so the small model inherits much of the teacher's quality where it counts.
- **Small models unlock on-device** — privacy (data never leaves the device), zero inference cost (runs on the user's hardware), offline capability, and no per-token API bill. For many products this is transformative.
- **The economics are overwhelming** — a 4B model serves at a fraction of a 70B's cost and latency (inference series Ch 2, 12); on-device it's ~free. When a small model clears the bar, deploying big is burning money.

The result: the 2026 stack is often **a big teacher (train/rent) → a small student (distill) → deploy the student**, with the big model kept only as a fallback for the hardest queries (the hybrid router, Chapter 2.2).

## 3.3 Distillation as the path to small

The primary way to build a high-quality small model (deep mechanics in the post-training series Ch 10; here, the strategic view):

- **Why distill vs train-small-from-scratch:** a small model trained from scratch on raw data underperforms a small model *distilled* from a strong teacher, because "the teacher's soft probability distributions carry richer signal than raw labeled data alone." The teacher effectively curates and enriches the training signal.
- **The recipe:** generate outputs (or logits/soft targets) from a strong teacher on your task distribution, train the small student to match them (off-policy distillation), optionally with on-policy refinement. For reasoning, distill the teacher's *chains of thought* (the R1-distill recipe — post-training Ch 10) so the small model learns to reason.
- **Base-model choice matters:** systematic 2026 benchmarks find some small bases distill far better than others (Qwen3-4B emerged as a strong distillation base). Pick a small base known to fine-tune/distill well, not just the smallest available.
- **Specialize hard:** distillation shines when the student only needs the teacher's competence on a *narrow* task — you're not compressing the whole teacher, just its skill on your distribution. The narrower the target, the more the small model can match the big one.

## 3.4 Quantization for the edge

To fit and run on constrained hardware, small models are quantized aggressively — the deployment-quantization story from the inference series (Ch 6), pushed harder because edge memory is tiny:

- **The converged recipe:** "train in 16-bit, quantize to 4-bit for deployment." 4-bit PTQ (GPTQ/AWQ) "preserves most model quality with 4× memory reduction" and is now standard practice for edge.
- **INT4 for edge:** compressing 16-bit → 4-bit cuts model size ~75% and speeds inference 3–4×, making a 3–4B model fit and run on a phone. GGUF (llama.cpp) is the standard format for on-device (inference series Ch 5).
- **The quality caveat (inference Ch 6):** INT4 hurts math/code/reasoning more than chat; validate on your task, and for reasoning-critical edge tasks consider a slightly larger model at higher precision over a tiny one at INT4.
- **Hardware-aware:** modern phone SoCs (Snapdragon, Apple Neural Engine) and NPUs accelerate quantized inference; target the deployment chip's supported formats.

## 3.5 On-device deployment

Where small + quantized models actually run in 2026:

- **Phones:** models like Llama 3.2 1B/3B, Phi-family, Gemma-small run on flagship phone chips (Snapdragon 8-class, Apple Silicon) at useful speed, reaching GPT-3.5-class quality on-device for many tasks.
- **Edge/embedded:** Raspberry Pi-class boards, IoT devices, even microcontrollers (ESP32-class for the smallest models) run SLMs for offline, private, low-power inference.
- **Laptops/desktops:** consumer hardware runs up to ~30B models locally (inference series Ch 10, 13) via llama.cpp/Ollama.
- **Runtimes:** llama.cpp/GGUF, ONNX Runtime, MLC-LLM, Apple's on-device frameworks, and vendor NPU SDKs are the on-device serving layer (the local-serving story from inference Ch 5).

The design implications reach back into training: if you're *targeting* the edge, you plan for it — distill to a size that fits the target device at INT4, specialize hard to the task (edge devices serve narrow apps), and test on the actual hardware early (the token/latency budget is brutal on a phone).

## 3.6 When small is wrong

Balance: small isn't always right. Stay big (or route to a frontier model) when:
- The task genuinely needs broad knowledge or hard general reasoning (the small model's narrowness bites).
- You can't get good distillation data or a good teacher for the task.
- The quality gap on your task exceeds what the cost/latency savings justify.
- The task is open-ended enough that "narrow specialization" doesn't apply.

The hybrid router (Chapter 2.2) resolves this: small/specialized for the common narrow cases, big/frontier for the hard open-ended ones. You rarely need *only* small or *only* big.

## 3.7 Decisions

1. **Right-size to the task** — deploy the *smallest* model that clears the quality bar; treat size as a deployment decision, not a quality one.
2. **Distill from a strong teacher** rather than training small from scratch — soft targets carry richer signal; pick a base known to distill well (e.g. Qwen3-4B class) and specialize hard.
3. **Quantize to 4-bit (INT4/GGUF) for the edge** — the converged 16-bit-train / 4-bit-deploy recipe — validating quality on your task, and prefer a larger model at higher precision for reasoning-critical edge work.
4. **Target the deployment hardware early** — test on the actual phone/edge chip; the token/latency/memory budget is the binding constraint.
5. **Keep a big fallback and route** — small/specialized for common narrow cases, frontier for hard open-ended ones (Chapter 2.2 hybrid).

Next: embeddings and retrieval — the models you train alongside your LLM to make it recall.
