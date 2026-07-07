# Chapter 15 — Essential Reading and References

Serving moves fast and much of the best material is engine documentation, engineering blogs, and benchmarks rather than papers — so this list weights primary docs and a few foundational papers over a long academic bibliography. Read the essential cut in order; the topic list and provenance notes follow.

## The essential cut

If you read only these, you'll have every load-bearing idea in this series from a primary source. Ordered as a sequence.

| # | Source | One-line reason | Chapters |
|---|---|---|---|
| 1 | Kwon et al., *Efficient Memory Management for LLM Serving with PagedAttention* (vLLM, arXiv:2309.06180) | The paper that defined modern serving; KV paging and continuous batching | 2, 3, 4 |
| 2 | *vLLM documentation* (docs.vllm.ai) | The reference for how a real engine exposes every technique in this series | 3–9 |
| 3 | Zheng et al., *SGLang: Efficient Execution of Structured LM Programs* (arXiv:2312.07104) | RadixAttention prefix reuse and overlapped structured decoding | 3, 5, 9 |
| 4 | Agrawal et al., *Sarathi-Serve / chunked prefill* (arXiv:2403.02310) | The prefill/decode interference fix that most deployments run | 4 |
| 5 | Zhong et al., *DistServe: prefill/decode disaggregation* (arXiv:2401.09670) | Why and when to split the phases across hardware | 4, 8 |
| 6 | Leviathan et al., *Fast Inference via Speculative Decoding* (arXiv:2211.17192) | The exact-verification trick that beats the memory-bound ceiling | 7 |
| 7 | Li et al., *EAGLE* (arXiv:2401.15077) + *Medusa* (arXiv:2401.10774) | The speculative-decoding methods you'll actually deploy | 7 |
| 8 | Lin et al., *AWQ* (arXiv:2306.00978) + Frantar et al., *GPTQ* (arXiv:2210.17323) | The weight-quantization methods and their accuracy/calibration trade | 6 |
| 9 | Dong et al., *XGrammar* (arXiv:2411.15100) | The structured-generation engine behind the major serving stacks | 9 |
| 10 | Epoch AI, *Inference Economics of Language Models* (epoch.ai) | The cost model — utilization, batch size, and the economics that govern everything | 12 |
| 11 | *Test-Time Compute Changed the Economics of Inference* (2026) + *OckBench* (arXiv:2511.05722) | Why reasoning models re-priced serving and how to measure reasoning efficiency | 8, 12 |
| 12 | *llm-d* + NVIDIA *Deploying Disaggregated Inference on Kubernetes* | The production Kubernetes-native serving stack, disaggregation, KV-aware routing | 11 |

## Why each, and what to extract

**1. PagedAttention / vLLM (arXiv:2309.06180).** Read first. It introduces paged KV cache and popularized continuous batching — the two techniques underneath every modern engine. Extract: why fragmentation caps concurrency and how paging removes it.

**2. vLLM docs.** The living reference: how continuous batching, prefix caching, chunked prefill, quantization, multi-LoRA, guided decoding, and the OpenAI-compatible server are actually configured. Read the config for each technique as you adopt it.

**3. SGLang (arXiv:2312.07104).** RadixAttention (automatic prefix reuse via a radix tree) and the overlapped scheduler that makes structured output nearly free. Extract: when prefix-heavy and structured-output-heavy workloads justify SGLang over vLLM.

**4. Sarathi-Serve / chunked prefill (arXiv:2403.02310).** The technique that stops long prefills from stalling decode. Extract: the chunk-size trade and why mixing prefill chunks with decode improves utilization.

**5. DistServe (arXiv:2401.09670).** The case for prefill/decode disaggregation and when its complexity pays off. Extract: how independent scaling of the two pools hits dual TTFT/TPOT SLOs — and why it's overkill below scale.

**6. Speculative Decoding (arXiv:2211.17192).** The foundational exact-speedup method. Extract: how verification preserves the target distribution, and why acceptance rate governs the speedup.

**7. EAGLE (arXiv:2401.15077) + Medusa (arXiv:2401.10774).** The deployable speculative methods. Extract: EAGLE's feature-level drafting (highest acceptance) and Medusa's extra-heads simplicity; when each fits.

**8. AWQ (arXiv:2306.00978) + GPTQ (arXiv:2210.17323).** The weight-quantization workhorses. Extract: activation-aware protection of salient channels (AWQ), layer-wise second-order quantization (GPTQ), and the role of calibration.

**9. XGrammar (arXiv:2411.15100).** The structured-generation backend. Extract: how precomputed automata + mask caching make per-token grammar enforcement <40 μs, and why engine choice affects structured-output throughput.

**10. Epoch AI, Inference Economics.** The rigorous cost model. Extract: how batch size, utilization, and hardware set cost per token, and where the throughput/latency frontier sits.

**11. Test-Time Compute economics + OckBench (arXiv:2511.05722).** The reasoning-era repricing. Extract: why long decode collapses throughput-per-GPU and how to measure/optimize reasoning-token efficiency.

**12. llm-d + NVIDIA disaggregation-on-K8s.** The production substrate. Extract: KV-cache-aware routing, hierarchical KV offloading, scale-to-zero, and the Gateway API Inference Extension — how the fleet actually runs.

## Reading list by topic

- **Serving fundamentals & engines:** vLLM (2309.06180) and docs; SGLang (2312.07104); TensorRT-LLM docs; TGI docs; Orca (continuous batching, OSDI 2022); the vLLM-vs-SGLang-vs-TensorRT-LLM benchmark write-ups (inferenceengineering.tech, spheron.network, 2026).
- **KV cache & memory:** PagedAttention (2309.06180); RadixAttention (in SGLang); KV-cache quantization surveys; LMCache / hierarchical offloading (llm-d docs).
- **Batching & scheduling:** Sarathi-Serve / chunked prefill (2403.02310); DistServe (2401.09670); Splitwise (2311.18677); the BentoML and Spheron disaggregation guides (2026).
- **Quantization:** AWQ (2306.00978); GPTQ (2210.17323); SmoothQuant (2211.10438); LLM.int8() (2208.07339); FP8 formats (2209.05433); NVIDIA TensorRT Model Optimizer docs; llama.cpp GGUF docs; the FP8-vs-INT8-vs-INT4 benchmark write-ups (2026).
- **Speculative decoding:** Leviathan et al. (2211.17192); Chen et al. (2302.01318); EAGLE 1/2/3 (2401.15077 and successors); Medusa (2401.10774); prompt-lookup/n-gram decoding.
- **Structured output:** XGrammar (2411.15100); llguidance (Microsoft); Outlines (2307.09702); grammar-constrained decoding (2502.05111); the structured-output security work (2503.24191).
- **Reasoning-model serving & economics:** Epoch AI inference economics; test-time-compute economics essays (2026); OckBench (2511.05722); reasoning-effort/thinking-budget API docs.
- **Hardware:** H100/H200/B200/MI300X spec sheets and the 2026 GPU price/performance comparisons (siliconanalysts, gpuadvisor); NVIDIA Blackwell/FP4 documentation.
- **Production / Kubernetes:** llm-d (github.com/llm-d); NVIDIA "Deploying Disaggregated LLM Inference on Kubernetes"; KServe docs; Ray Serve docs; the Gateway API Inference Extension (GA Feb 2026); vLLM-on-K8s production guides (2026).

## Provenance notes

**Primary papers (2022–2024):** PagedAttention, SGLang, Sarathi-Serve, DistServe, speculative decoding, EAGLE/Medusa, AWQ/GPTQ, XGrammar are peer-reviewed or well-cited arXiv papers introducing techniques that are now standard; their mechanisms are settled and quoted directly.

**Engine documentation:** vLLM/SGLang/TensorRT-LLM/llama.cpp docs are authoritative for *how* to configure each technique but version-churn fast — verify flags against the version you deploy.

**2026-dated benchmarks and economics essays:** The engine comparisons, GPU price/performance charts, cost-per-token guides, and test-time-compute economics pieces postdate the January 2026 knowledge cutoff and were surfaced by mid-2026 web search. They reflect current practice and pricing (H100 ~$2–3.50/hr, FP8 as default, XGrammar as default backend, reasoning re-pricing) but as vendor blogs and benchmarks their specific numbers should be re-verified for your hardware and re-measured on your traffic before you build on them.

**A note on numbers:** Every concrete figure here (single-stream `bandwidth/model_bytes`, ~0.5–2% FP8 quality hit, <40 μs XGrammar masking, 10×+ INT4 concurrency swing, ~50% batch discount, 30–90s cold starts, GPU specs) is drawn from the sources above and reflects mid-2026 practice. They are defaults to reason from and *measure on your own workload* — serving optima are workload-specific, and your traffic will move them.
