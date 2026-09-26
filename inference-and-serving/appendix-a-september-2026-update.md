# Appendix A — September 2026 Update

*This series was compiled in early July 2026. This appendix records what changed between July and September 2026, plus one serving technique the main chapters missed: **block-diffusion speculative drafting** (A.3). The architecture shifts in pretraining series Appendix B (hybrid linear attention, compressed sparse attention, FP4) are what drive most of the serving changes. Models are now co-designed with their KV cache, and the serving stack is following.*

## A.1 KV cache: co-designed with the model

Chapter 3's rule, "KV cache is often the binding constraint; quantize it to FP8 almost always," still holds for the models most teams serve. At the frontier, KV-cache size is now designed into the architecture:

- **DeepSeek-V4.1-Flash** (September 2026) reports a global KV footprint of about **890 bytes per token**, roughly a quarter of V4-Flash's. It gets there with three techniques together. **Compressed sparse attention**: many tokens are compressed into one KV entry, and a learned indexer selects the top-k entries. **Cross-layer sharing (CSA2)**: each layer is statically assigned Full, Reindex, or Reuse mode, so layers share the main KV, the indexer keys, and even the top-k selections. **FP4 KV storage**: E2M1 values with one E4M3 scale per 16 channels. For scale, a Llama-3-70B-shaped GQA model in BF16 needs about 320 KB per token. That is a ~370× difference, which turns 1M-token contexts from a memory problem into a bandwidth problem.
- **Kimi K3** gets its savings from the other direction. About three quarters of its layers are linear attention (KDA), which keeps a fixed-size state instead of a growing cache. Moonshot reports up to 75% less KV memory and up to 6× decode throughput at 1M context, relative to full attention at matched scale.

**What changes for you:**

1. **FP4 KV cache is real, but only where the model was trained for it.** V4.1's FP4 KV is part of the architecture. Applying FP4 KV quantization after the fact to an arbitrary model is still not the safe default (Chapter 6). Keep FP8 as the default and FP4 as a model-specific option. Engines differ here too: vLLM's first V4.1-Flash support stored the whole KV in MXFP8.
2. **Capacity planning for hybrids needs two numbers.** A hybrid model's per-request memory is a *fixed recurrent state* (linear layers) plus a *growing KV* (the minority of attention layers). Chapter 12's "KV bytes per token × context" formula undercounts short requests and overcounts long ones unless you split it.
3. **Prefix caching must understand non-KV state.** Radix and prefix caches were built for KV blocks. For hybrid and sliding-window models, the engine must also snapshot recurrent state at cache boundaries. SGLang made its UnifiedRadixTree the default for SWA/Mamba/DSA models (v0.5.16). vLLM added Mamba prefix caching and reported a 9–25% improvement in time to first token (v0.29). **Check hybrid prefix-cache support before choosing an engine for a hybrid model.** Without it, agentic workloads with long shared prefixes lose most of Chapter 3's caching benefit.

## A.2 Engine updates (Chapter 5)

The two open engines shipped roughly every two weeks through the summer. The changes that matter for engine choice:

**vLLM (v0.27–v0.30, August–September 2026)**
- **Model Runner V2 became the default** for all models (v0.29), adding dual-batch overlap and full CUDA graphs (v0.30).
- **Encoder/prefill/decode (E/P/D) disaggregation** matured. This extends Chapter 4's prefill/decode split with a third tier for vision encoders (multimodal series Appendix A).
- **Tiered KV offloading** now reaches disk, and **HiSparse** adds a host-resident KV tier for sparse-attention models. This is Chapter 3's offload hierarchy, shipped in the engine.
- **A Rust frontend with a gRPC control plane** (health, abort, discovery, KV-event sources) and `vllm-bench` in the CLI. The routing and control layer of Chapter 11 is moving into the engine.
- **"Fast Start"**, a persistent GPU weight-cache daemon over IPC, targets the cold-start storms of Chapter 14.
- **Built-in watermarking** (Gumbel-max generation plus detection), a direct response to the EU AI Act Article 50 marking obligation (cross-cutting Appendix A).
- Early **Rubin (sm_107)** and AMD gfx1250 enablement.

**SGLang (v0.5.15–v0.5.20, July–September 2026)**
- **Spec V2** (zero-overhead speculative scheduling) and **breakable CUDA graphs** became the defaults.
- **Decode context parallelism** for MLA and sparse-attention models (DeepSeek V4, Kimi K3) splits one long request's decode across GPUs. It is the tool for 1M-context decode, where a single request's KV exceeds one device.
- **DeepEP v2** with CUDA-graph execution across nodes, W4A8 MoE on Hopper, and NVFP4 checkpoints running on AMD hardware.
- A **session-aware unified radix cache** for agentic workloads.
- **CUDA 12 support ends with v0.5.20.** Check your driver and base images before upgrading.

**Engine-choice guidance is unchanged** (vLLM for breadth and ecosystem, SGLang for structured, agentic, and prefix-heavy workloads). Both now ship day-0 support for new frontier open models. The practical differentiators are **hybrid-state caching** and **long-context decode parallelism**, so benchmark both on your model.

## A.3 Block-diffusion speculative drafting (previously missed)

Chapter 7 names EAGLE-family drafting as the 2026 default. It missed **DFlash** (arXiv:2602.06036, ICML 2026), which by summer had shipped in both engines:

- **The idea.** EAGLE-style drafters are autoregressive, so drafting *k* tokens costs *k* sequential draft steps. DFlash uses a small **block-diffusion** drafter that proposes a whole block of tokens in one forward pass, conditioned on the target model's hidden states through deep KV injection. The target then verifies the block as usual, so output stays *lossless*.
- **The reported numbers.** Over 6× lossless speedup across models and tasks, and up to 2.5× more speedup than EAGLE-3. NVIDIA reports up to 15× on Blackwell in favorable settings.
- **Where it runs.** SGLang's Spec V2 supports DFlash (LMSYS, June 2026). vLLM added **DFlash2** with local convolution (v0.28). SGLang also added **DSpark**, which uses confidence-driven verification windows, and reported 383.7 tok/s on DeepSeek-V4-Pro.

**Updated guidance for Chapter 7.4:** EAGLE-3 remains the safe default because trained heads exist for most popular models. If your model has a published DFlash drafter, or you can train one, benchmark it. Reasoning models (Chapter 8), with long single-request decodes and spare compute, are where block drafting gains the most. As always, measure acceptance and end-to-end latency on *your* traffic, because the advertised numbers come from favorable distributions.

## A.4 Serving the new frontier open models

Self-hosting the largest open models is a rack-scale problem:

- **Kimi K3** (2.8T parameters) ships in MXFP4 (about 4.25 bits per weight including scales), which is **~1.5 TB of weights** before any KV. An 8×B200 node (1.5 TB HBM) cannot hold it. An 8×B300 node (~2.3 TB) holds it with little KV headroom. Realistic deployments are multi-node or rack-scale (GB300 NVL72-class) with wide expert parallelism (Chapter 7).
- **DeepSeek-V4.1-Flash** (552B, MIT) and **GLM-5.3-Flash** (320B total / 18B active, MIT) are the practical "frontier-adjacent" self-hosting tier. Both fit a single 8-GPU Blackwell node with room for long-context KV.
- **The license can constrain the serving plan.** Kimi K3's license requires a separate agreement for model-as-a-service offerings above $20M in trailing-12-month group revenue. The full GLM-5.3's license adds a security review for very large MaaS operators. Check these *before* you build a hosted-API business on them (cross-cutting Chapter 2 and Appendix C).

## A.5 Hardware and economics (Chapters 10, 12)

- **NVIDIA Rubin** began shipping from Q3 2026. NVIDIA's per-package figures: 288GB HBM4 at ~22 TB/s and ~50 PFLOPs NVFP4 for inference. The ~2.7× bandwidth over B200 (~8 TB/s) matters more than the FLOPs for decode-bound serving (Chapter 2). **AMD Helios** (72× MI455X, 31 TB HBM4 per rack) ships from the end of Q3. Microsoft has announced Helios for frontier-model *inference* on Azure. For procurement decisions now, both are 2027 capacity.
- **Prices kept falling.** Trackers show continued steep declines in per-token prices through the summer, and both big frontier releases were priced or tuned for lower cost. Anthropic's Opus 5 (July) came at $5/$25 per million input/output tokens with 1M context. Opus 5.5 (September) is reported to cost about 40% less to run than Opus 5 at default settings on typical workloads. Chapter 12's FinOps loop (re-measure quarterly, 12.5) and build-vs-buy analysis (12.7) matter more when API prices fall faster than your self-hosting costs.
- **Effort levels are a serving lever.** Frontier APIs and open models now expose trained reasoning-effort levels (post-training series Appendix A.2). Route effort per request the way Chapter 8 routes by difficulty. It is the cheapest latency and cost control you have for reasoning models.

## A.6 Decisions

1. **Keep FP8 KV as the default.** Use FP4 KV only where the model was trained for it.
2. **Model hybrid memory as fixed state plus growing KV**, and require hybrid-aware prefix caching from your engine.
3. **Benchmark block-diffusion drafters** (DFlash-class) against EAGLE-3, especially for reasoning models.
4. **Use decode context parallelism** for 1M-context serving on MLA and sparse models.
5. **Treat 2T+ open models as rack-scale deployments**, and read the MaaS clause of the license before planning a hosted business on them.
6. **Re-run the build-vs-buy math**, because API prices and effort-level routing changed the numbers this quarter.

## A.7 Sources

- DeepSeek-AI, *DeepSeek-V4.1-Flash: Pushing the Limits of KV Cache Compression*, 2026 — arXiv:2609.19969
- Kimi Team, *Kimi K3: Open Frontier Intelligence*, 2026 — arXiv:2607.24653
- *DFlash: Block Diffusion for Flash Speculative Decoding* (ICML 2026) — arXiv:2602.06036; LMSYS, *The next generation of speculative decoding: DFlash and Spec V2*, June 2026 — https://www.lmsys.org/blog/2026-06-15-next-generation-speculative-decoding-dflash-v2/; NVIDIA Technical Blog on DFlash on Blackwell
- vLLM releases v0.27–v0.30 — https://github.com/vllm-project/vllm/releases
- SGLang releases v0.5.15–v0.5.20 — https://github.com/sgl-project/sglang/releases
- NVIDIA Rubin (GTC 2026 specifications); AMD Helios / MI455X announcements, 2026
- Anthropic Opus 5 (July 24, 2026) and Opus 5.5 (September 22, 2026) launch coverage; per-token price indices (Silicon Data, BenchLM), September 2026
