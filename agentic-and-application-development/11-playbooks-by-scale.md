# Chapter 11 — Playbooks by Scale

Concrete end-to-end builds assembling the series, ordered by the complexity ladder — each playbook is the previous one plus earned rungs. Starting points to adapt, not laws; Chapters 2–10 hold the *why*. The meta-instruction that governs all four: **build the eval harness at step one of each, and climb to the next playbook only when your metrics prove this one insufficient.**

---

## 11.1 A grounded Q&A product (RAG chatbot)

**Goal:** a chatbot that answers accurately from your documents, with citations. The most common LLM application; no agent required.

- **Architecture:** a *workflow* (Chapter 4.3) — retrieve → assemble → generate. No loop.
- **Retrieval:** the Chapter 7 baseline — recursive chunking (300–500 tokens, 10–15% overlap), hybrid search (dense + BM25) + reranker. Strong open embedder + reranker (cross-cutting Ch 4), self-hosted.
- **Context:** stable system prompt (cite sources; say "I don't know" when context lacks the answer) as the cached prefix; retrieved chunks in the volatile suffix (Chapter 2.4).
- **Eval first:** a golden set of question → answer-bearing-chunk pairs; Recall@K for retrieval, RAGAS faithfulness for answers (Chapter 7.6). Gate every chunking/embedder/prompt change on it.
- **Production:** trace every request; label failures; route them into the golden set (Chapter 9.6). Guardrails: input screening + grounding check on output.
- **Escalate only on evidence:** multi-hop failures in the eval set → query decomposition, then agentic retrieval for the hard slice (Chapter 7.4). Not before.

---

## 11.2 A single production agent

**Goal:** an agent that completes a bounded class of tasks with tools — a support agent that looks up orders and issues refunds, a data agent that answers questions against your warehouse.

- **Architecture:** one agent loop (Chapter 4.2) with a hard budget; planning only if tasks are multi-phase. Consider first whether a *workflow* covers 80% of traffic with an agent handling the remainder (the least-autonomy rule).
- **Tools:** 5–15 intention-level tools (Chapter 3), MCP-packaged, tiered by consequence — reads free, writes logged, refund-class actions gated behind HITL approval (durable pause, Chapter 5.4).
- **Context:** stable prefix (system + policies-in-structured-fields + tools), append-only turns, compaction at phase boundaries (Chapter 2).
- **Framework:** hand-rolled loop or a thin framework if no durable-state/multi-agent needs; LangGraph/Agent SDK once HITL and resumability matter (Chapter 8.4).
- **Eval first:** 30–100 task scenarios with *state-based* ground-truth checks (never self-report — Chapter 9.5); run each 4–8×, ship on pass^k, not pass@1. τ-bench-style simulated-user harness if conversational.
- **Production:** full tracing, per-turn labeling, action-gate + budget guardrails, escalation-to-human as a designed, measured outcome (Chapter 10).
- **Cost:** per-step model routing (small model for classification/extraction turns), cache discipline.

---

## 11.3 An agentic RAG / research system

**Goal:** answers to hard, multi-hop questions over a large corpus — research assistance, due diligence, investigation. Where 11.1's pipeline measurably fails.

- **Architecture:** retrieval-as-a-tool inside an agent loop (Chapter 7.5): the agent decomposes the question, searches (hybrid + rerank underneath, unchanged), evaluates results, iterates. Router in front: easy questions → the 11.1 pipeline; hard ones → the agent (most traffic is easy — pay agent costs only on the hard slice).
- **Context:** aggressive Chapter 2 hygiene — searches are bulky; compress each round's results to findings before the next round; write intermediate findings to a scratchpad.
- **Graph RAG** only if your eval shows relationship-heavy questions failing chunk retrieval (Chapter 7.4) — it adds a graph-maintenance burden; earn it.
- **Sub-agents** (Chapter 6) only when single-agent context demonstrably rots on broad questions: orchestrator + parallel research workers with isolated windows, concise findings handed back, shallow hierarchy.
- **Eval:** multi-hop golden questions with ground-truth answers; trajectory metrics (search count vs necessary, loop rate); faithfulness scoring; pass^k on the hard slice.
- **Cost watch:** this is the playbook where per-task cost explodes quietly — per-task budgets and cost-per-completed-task tracking from day one (Chapter 10.5).

---

## 11.4 A multi-agent platform

**Goal:** many agents, many tools, many teams — an internal platform where agentic capabilities are built, shared, and operated org-wide. The frontier of this series, and mostly an *organizational* engineering problem.

- **Standardize the substrate:** MCP for every tool integration, A2A for inter-agent contracts, one traced observability pipeline spanning all agents (Chapters 3.4, 8.5, 10.2) — the portability hedge is what lets teams choose frameworks without fragmenting the platform.
- **Framework:** one blessed production framework past the three thresholds (durable state + HITL, MCP, A2A — Chapter 8.2); committed for a year+; benchmark before committing (the 30-point spread).
- **Patterns:** orchestrator-worker with shallow hierarchies as the sanctioned multi-agent shape (Chapter 6); workflow-first policy for new use cases; an architecture-review gate for anyone requesting more autonomy (spend-autonomy governance made organizational).
- **Central guardrails:** the Chapter 10.3 stack (input/action/output gates, budgets) as *platform middleware*, not per-team reimplementation; tool-permission tiers and credential least-privilege administered centrally; third-party MCP servers vetted like dependencies (Chapter 3.4).
- **Eval as infrastructure:** shared harness, per-agent suites with pass^k gates in CI, production labeling feeding per-team eval sets, a platform-level held-out audit suite (Chapter 9.6).
- **The org loop:** this playbook is cross-cutting Ch 8 applied to agents — data/eval ownership, the improvement flywheel per agent, cost attribution per team, and red-teaming (cross-cutting Ch 6.4) as a scheduled platform function.

---

## 11.5 The cross-cutting checklist

Before shipping any of the four:

- [ ] Eval harness exists *before* the build; changes gate on it; a held-out slice is protected.
- [ ] Ground-truth (state-based) success checks — never the agent's self-report.
- [ ] pass^k measured (4–8 runs) on the tasks that matter; product bar set in pass^k terms.
- [ ] Least autonomy that solves the task; every rung of complexity justified by a metric.
- [ ] Budgets (steps/tokens/cost/time) enforced in code on every loop.
- [ ] Tool permissions tiered; irreversible actions behind HITL; execution sandboxed.
- [ ] Stable-prefix context layout; cache hit rate verified; compaction at boundaries.
- [ ] Full tracing on; failures labeled and routed into the eval set weekly.
- [ ] Escalation path designed, measured, and tested — in both directions.
- [ ] Cost-per-completed-task tracked from day one.

Next: failure modes — the characteristic ways these systems break.
