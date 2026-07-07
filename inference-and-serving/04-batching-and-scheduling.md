# Chapter 4 — Batching and Scheduling

## 4.1 Conceptual picture

Chapter 2 established that batching is the master throughput knob and that prefill (compute-bound) and decode (memory-bound) interfere. The scheduler is the component that resolves both: it decides, every step, which requests run, how new requests join, and how prefill and decode share the GPU. Good scheduling is the difference between a system that posts high throughput on a benchmark and one that delivers high *goodput* — SLO-compliant throughput — on real, bursty traffic. This chapter is the ladder of scheduling techniques, from the one everybody uses to the one only large fleets need.

## 4.2 Continuous (in-flight) batching

The foundation. **Static batching** — the naive approach — groups N requests, runs them together, and waits for *all* to finish before starting the next batch. Because outputs have wildly different lengths, the whole batch stalls on the slowest request while finished slots sit idle. Catastrophic GPU waste.

**Continuous batching** (a.k.a. in-flight or iteration-level batching) instead makes scheduling decisions *every decode step*: the moment any request finishes and frees a slot, a waiting request joins the running batch immediately. No request waits for the batch to drain; the GPU stays maximally full. This is the single most important serving technique after paged KV, and — like paging — it's on by default in every modern engine (vLLM, SGLang, TensorRT-LLM, TGI). You don't build it; you get it. But it's *why* those engines exist and why you should never hand-roll a serving loop.

## 4.3 The prefill/decode interference problem

Continuous batching alone has a flaw. When a new request arrives, its prefill (processing a long prompt in one big compute-heavy pass) must run — and a naive scheduler runs that whole prefill in one step, during which *all ongoing decodes stall*. A single 32K-token prompt arriving can freeze every other user's token stream for the duration of that prefill, spiking their TPOT and blowing tail latency. This is the central scheduling tension, and the next two techniques address it.

## 4.4 Chunked prefill

**Chunked prefill** (a.k.a. dynamic/piecewise prefill) breaks a long prompt's prefill into fixed-size chunks (e.g. 8K tokens) and interleaves those chunks with ongoing decode steps rather than running the whole prefill at once. A 32K prompt processes as four chunks, with everyone's decode steps running between chunks. Result: prefill no longer monopolizes the GPU; decodes keep flowing; tail latency stays bounded. It also *improves* overall utilization by mixing compute-bound prefill chunks with memory-bound decode in the same batch — the two phases' complementary resource profiles (Chapter 2.3) fill the GPU better together than either alone.

Chunked prefill runs on a single node with no extra hardware, which is why it's the **default recommendation for most deployments** that see mixed prompt lengths. Tune the chunk size: smaller chunks protect decode latency more but add scheduling overhead; 8K is a common starting point. It's a config flag in vLLM and SGLang.

## 4.5 Prefill/decode disaggregation

The heavyweight solution, for large fleets with strict *simultaneous* TTFT and TPOT SLOs. **Disaggregation** physically separates the phases onto different GPU pools: **prefill nodes** (compute-dense, e.g. B200) do prompt processing; **decode nodes** (bandwidth-optimized) do token generation; the KV cache is transferred from prefill to decode nodes over fast interconnect when prefill completes.

Why it wins at scale:
- Each pool is sized and tuned for its phase — prefill for compute, decode for bandwidth — instead of compromising one machine for both.
- The phases stop interfering by construction: a prefill surge scales the prefill pool without touching decode latency.
- You scale them **independently** — if your traffic is prompt-heavy (RAG, long inputs) you add prefill nodes; if it's generation-heavy (reasoning, long outputs) you add decode nodes.

The cost: complexity, and the KV-cache transfer overhead between pools (FlowKV, NIXL, and llm-d's transport layers exist to make this transfer fast and reliable). Disaggregation pays off **only at scale, under strict dual SLOs** — public results show ~2× throughput and order-of-magnitude TTFT improvements on large B200 topologies, but for a single node or a small deployment, **continuous batching + chunked prefill is enough** and disaggregation is over-engineering. Reach for it when you're running many nodes and can't hit both latency targets any other way.

## 4.6 SLO-aware and priority scheduling

Beyond the batching mechanics, production schedulers make policy decisions:

- **Priority / preemption** — high-priority (interactive) requests preempt low-priority (batch) ones; preempted requests have their KV cache paged out and resumed later. Lets one fleet serve both realtime and batch traffic at different price points.
- **SLO-aware admission** — under overload, the scheduler must decide whether to queue, reject, or degrade requests rather than accept everything and violate everyone's SLO. Goodput (Chapter 1) is maximized by *shedding* load you can't serve within SLO, not by accepting it and missing every deadline.
- **Length-aware / predicted scheduling** — estimating output length to pack the batch better and avoid a few very long requests hogging slots. The Gateway API Inference Extension's predicted-latency scheduling (GA 2026) brings this to Kubernetes routing (Chapter 11).

## 4.7 The scheduling ladder

Turn these on in order as scale and SLO pressure grow:

1. **Continuous batching** — always (default). If your engine doesn't do it, change engines.
2. **Chunked prefill** — turn on for mixed prompt lengths; the default for most single-to-few-node deployments. Protects decode latency from big prefills.
3. **Prefix caching** (Chapter 3) — scheduling and memory overlap here; reused prefixes skip prefill entirely.
4. **Priority/preemption** — when one fleet serves multiple traffic classes.
5. **Prefill/decode disaggregation** — only at multi-node scale under strict dual SLOs, when 1–4 can't hit both targets.

## 4.8 Decisions

1. **Never use static batching**; rely on the engine's continuous batching. This is settled.
2. **Enable chunked prefill** for any traffic with variable/long prompts — it's the cheapest fix for prefill-induced tail latency.
3. **Reserve disaggregation for scale** — many nodes, strict simultaneous TTFT+TPOT SLOs; otherwise it's complexity you don't need.
4. **Scale the bottleneck phase independently** once disaggregated: prefill pool for prompt-heavy traffic, decode pool for generation-heavy.
5. **Optimize goodput, not throughput** — shed or deprioritize load you can't serve in SLO rather than accepting it and missing every deadline.

Next: the serving engines that implement all of this, and how to choose among them.
