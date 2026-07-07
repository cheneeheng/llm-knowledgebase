# Chapter 8 — Serving Reasoning Models

## 8.1 Conceptual picture

Reasoning models (the thinking models of the post-training series) broke the serving economics that the rest of this series assumes, and they did it through one change: they emit an enormous **chain of thought** — 5,000 to 50,000 tokens of internal reasoning — before the short user-visible answer. Everything expensive in serving is the decode phase (Chapter 2), and reasoning models do 10–100× more of it per request. This chapter is about what that does to your system and how to serve it without going bankrupt.

The scale of the shift, in the words of the 2026 inference-economics literature: test-time compute "quietly changed the economics of inference" — queries that used to cost a tenth of a cent now cost tens of cents, and reasoning output lengths are growing ~5× per year. Inference went from a third of AI compute to two-thirds largely because of this. If you serve reasoning models, decode efficiency isn't one concern among many; it's the concern.

## 8.2 What long decode does to the system

Trace the consequences through the earlier chapters:

- **KV cache explodes (Chapter 3).** KV grows with sequence length, and reasoning sequences are long. A request generating 30K reasoning tokens holds a KV cache that dwarfs a chat request's. This slashes how many concurrent reasoning requests fit in HBM — "long sequences inflate per-request memory pressure, forcing batch sizes down and collapsing throughput-per-GPU." Smaller batches mean worse utilization means higher cost-per-answer, "by an order of magnitude or more."
- **Decode dominates totally (Chapter 2).** The prefill (reading the prompt) is a rounding error next to a 30K-token decode. Your system is almost pure decode — memory-bandwidth-bound, batch-starved. This makes reasoning models the *ideal* case for decode-optimized hardware (high HBM bandwidth) and for **speculative decoding** (Chapter 7), whose latency win applies to every one of those thousands of decode steps.
- **Latency is measured differently.** For a reasoning model, TTFT is nearly meaningless (the user waits through the whole think), and even TPOT matters less per-token than **total time to final answer** and **total tokens generated**. You optimize for time-and-cost-to-*answer*, not to first token.

## 8.3 Serving strategies specific to reasoning

- **Prioritize decode throughput and KV capacity.** Choose hardware for HBM bandwidth and capacity (H200/B200/MI300X — Chapter 10); enable FP8 KV cache aggressively (Chapter 3) because KV capacity directly caps reasoning concurrency; and lean on KV offloading for parked long contexts.
- **Speculative decoding is a bigger win here than anywhere.** Long single-request decodes with spare compute are exactly the regime where speculation shines (Chapter 7.4), and reasoning traces are often self-similar/repetitive (making EAGLE and even n-gram drafting effective). Enable it.
- **Disaggregation leans decode-heavy.** If you disaggregate (Chapter 4), reasoning traffic needs a *large decode pool* and a *small prefill pool* — the opposite of RAG traffic. Size the pools to your generation-heavy profile.
- **Batch aggressively where latency allows.** Since per-answer cost collapses with batch and reasoning is latency-tolerant (users expect thinking to take time), pack the batch hard — reasoning serving tolerates the higher TPOT that big batches bring because the user is already waiting for a long think.

## 8.4 Controlling the thinking budget

The most important *product* lever: reasoning length is only loosely correlated with answer quality, and unbounded thinking is where the cost goes. Modern reasoning models and serving stacks expose **thinking-budget controls**:

- **Reasoning-effort / thinking-budget parameters** — cap the reasoning tokens (or set low/medium/high effort), trading a little quality for a lot of cost on easy queries. Most 2026 reasoning models and APIs support this; use it to match compute to difficulty rather than paying max thinking for "what's 2+2."
- **Adaptive / router approaches** — route easy queries to a non-reasoning (or low-effort) model and hard queries to full reasoning. A cheap classifier in front of the fleet can cut cost dramatically because most real traffic is easy. This is the single biggest cost lever for a reasoning product.
- **Early-stopping / budget-forcing** — stop the reasoning when confidence is reached or a budget is hit. The efficiency-of-reasoning literature (OckBench and related) shows large token savings at small quality cost from smarter stopping.
- **Stripping the chain of thought from output** — you generally don't return (and shouldn't be billed downstream for) the raw reasoning; serve the answer, keep or discard the trace per policy. (Note: some providers keep reasoning traces server-side for safety/quality; decide your retention policy deliberately.)

## 8.5 The economics reframed

For reasoning models, the cost equation shifts from *per-request* to *per-reasoning-token*, and your optimization targets shift accordingly:

- **Cost is dominated by output tokens**, which are produced sequentially and can't be parallelized within a request — so total generated tokens is the cost driver. Every token of reasoning you don't need is money.
- **Throughput-per-GPU collapses under long context**, so the same GPU serves far fewer reasoning requests than chat requests — plan capacity for a fraction of the concurrency you'd get on chat.
- **The biggest wins are algorithmic, not systems**: routing easy queries away from reasoning, and capping thinking budgets, beat any systems optimization because they cut the token count at the source. Do these *first*, then optimize the serving of what remains.

Batch/offline reasoning (relaxed latency) unlocks the ~50% batch-API discount economics (Chapter 1) — if your reasoning workload tolerates minutes of latency (report generation, offline analysis), batch it hard for a large cost cut.

## 8.6 Decisions

1. **Optimize for time-and-cost-to-final-answer, not TTFT** — reasoning traffic is decode-dominated and latency-tolerant.
2. **Prioritize HBM bandwidth/capacity and FP8 KV cache** — KV capacity caps reasoning concurrency; plan for far lower concurrency than chat.
3. **Enable speculative decoding** — reasoning's long, repetitive, low-batch decodes are its best case.
4. **Control the thinking budget as a product lever** — reasoning-effort caps plus a difficulty router that sends easy queries to a cheaper path is the single biggest cost win.
5. **Batch aggressively (latency-tolerant) and batch offline reasoning** for the discount economics; size disaggregated pools decode-heavy.

Next: structured output and adapters — making reasoning and non-reasoning models emit reliable JSON and tool calls, and serving many fine-tunes from one fleet.
