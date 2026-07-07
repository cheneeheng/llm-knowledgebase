# Chapter 8 — Orchestration Frameworks

## 8.1 Conceptual picture

An orchestration framework is the software that implements the machinery of the last five chapters — the agent loop, tool invocation, memory, state persistence, multi-agent coordination — so you write application logic instead of plumbing. You don't have to use one (a simple agent loop is a page of code, Chapter 4.2), and for a single tool-using assistant, hand-rolling is often right. But as you add durable state, human-in-the-loop, multi-agent coordination, and production reliability, the plumbing grows, and a framework that provides these as primitives saves real work — at the cost of learning its abstractions and committing to them. This chapter maps the 2026 landscape and the choose-vs-build decision.

The stakes are higher than typical framework choices because of two facts: **the scaffold swings performance by up to 30 points** (Chapter 1.1), so the framework's orchestration quality matters to your product's capability, and **the orchestration layer is framework-specific and requires a rewrite to change** — "plan to commit to one framework for at least the first year of a production system." This is a decision to make deliberately, not by grabbing the most-starred repo.

## 8.2 The three maturity thresholds

The 2026 framework literature converges on three thresholds that separate production-ready frameworks from prototyping toys — use them as your filter:

1. **Durable state and human-in-the-loop as primitives** — can the framework checkpoint an agent's state so it survives interruption, pause for human approval, and resume (Chapter 5.4)? Without this, long-horizon and gated-action agents aren't productionizable.
2. **Native MCP** for tool interoperability (Chapter 3.4) — so your tools port across frameworks. All the serious 2026 frameworks now have it.
3. **Native A2A (agent-to-agent)** for cross-agent interoperability — so agents built in different frameworks/by different teams can coordinate.

A framework that hasn't crossed these is a prototyping tool, fine for experiments and wrong for production. This filter alone narrows the field usefully.

## 8.3 The 2026 landscape

The production-ready field, by best-fit (from the 2026 framework comparisons):

| Framework | Best for | Notes |
|---|---|---|
| **LangGraph** | Complex stateful workflows; the general default | Graph-based control flow, strong durable state + HITL, largest ecosystem (LangSmith for eval/observability). The safe general pick for complex orchestration. |
| **Claude Agent SDK** | Anthropic-native production agents | Hierarchical subagent spawning (to 3 levels), fallback model chains, MCP marketplace. Strong when building on Claude and wanting native agent primitives. |
| **CrewAI** | Fast role-based multi-agent prototypes | Ergonomic role/crew abstractions; fastest to a multi-agent prototype. Watch the workflow-in-disguise trap (Chapter 6.3). |
| **Microsoft Agent Framework** | Enterprise .NET/Microsoft stacks | The unified successor to Semantic Kernel + AutoGen; native MCP and A2A; the enterprise-Microsoft default. |
| **LlamaIndex Workflows** | RAG-grounded agents | Strong where retrieval is central (Chapter 7). |
| **Pydantic AI** | Type-safe Python agents | Type safety and validation as first-class; appeals to Python-typing-heavy teams. |
| **AutoGen / AG2** | Research-style conversational agents | The legacy research path; being superseded for production by the above. |

The selection rule: **filter by the three thresholds, then choose by your stack and interaction shape.** LangGraph is the reasonable general default for complex stateful orchestration; Claude Agent SDK if you're Anthropic-native; Microsoft Agent Framework for enterprise .NET; CrewAI for fast role-based prototyping; LlamaIndex when RAG is the center of gravity. Benchmark two candidates on *your* task if the choice is close — the 30-point scaffold spread means it's worth measuring.

## 8.4 Framework vs hand-rolled

The prior decision: use a framework at all? Guidance:

- **Hand-roll** when the system is simple — a single agent with a handful of tools and no durable-state/HITL/multi-agent needs. The loop is a page of code (Chapter 4.2); a framework adds abstraction you don't need, and you keep full control and debuggability. Many production single-agent systems are hand-rolled deliberately.
- **Use a framework** when you need its primitives — durable execution, HITL checkpointing, multi-agent coordination, or the observability/eval tooling that comes with the ecosystem (LangSmith with LangGraph). Re-implementing durable state and cross-agent coordination well is a large project; the framework has done it.
- **The middle path** — a thin framework or library (not a heavy orchestration engine) for the parts you want (e.g. just tool calling and retries) while keeping your own control flow. Many teams land here.

The trap in both directions: adopting a heavy framework for a task that needed a loop (carrying abstraction weight for nothing), or hand-rolling durable multi-agent execution (rebuilding what a framework provides, badly). Match the framework weight to the system's actual complexity — the earn-the-complexity principle applied to tooling.

## 8.5 MCP and A2A as the portability hedge

The strategic reason the protocols matter, beyond features: they decouple your *investments* from your *framework choice*. Tools built as MCP servers (Chapter 3.4) port across frameworks; agents speaking A2A interoperate across frameworks. Since the orchestration layer requires a rewrite to change (8.1) but the tools and inter-agent contracts don't *have to*, standardizing on MCP/A2A means a framework migration is a rewrite of orchestration logic only — not of your entire integration and agent-coordination surface. This materially lowers the cost of the year-long commitment a framework demands, and it's why "standardize on MCP" is the single most repeated portability advice of 2026. Build on the protocols; treat the framework as the replaceable part.

## 8.6 Decisions

1. **Decide framework-vs-hand-rolled by complexity** — hand-roll simple single-agent systems; adopt a framework when you need durable state, HITL, multi-agent, or ecosystem eval/observability.
2. **Filter frameworks by the three thresholds** — durable state + HITL, native MCP, native A2A — to separate production-ready from prototyping tools.
3. **Choose by stack and interaction shape** — LangGraph as the general default, Claude Agent SDK for Anthropic-native, Microsoft Agent Framework for enterprise .NET, CrewAI for role-based prototypes, LlamaIndex for RAG-centric; benchmark close calls on your task.
4. **Expect to commit for ~a year** — the orchestration layer is rewrite-to-change; choose deliberately, not by star count.
5. **Standardize on MCP/A2A as the portability hedge** — build tools and agent contracts on the protocols so a framework migration touches only orchestration logic.

Next: evaluating applications — how you know any of this works, using trajectory-level methods the model series' single-response evals can't provide.
