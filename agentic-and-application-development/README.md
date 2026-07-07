# Building With LLMs: The Agentic and Application Playbook

*From a served model to a working product — context engineering, tool design, agent architectures, memory, multi-agent systems, RAG systems, orchestration frameworks, application evaluation, and production reliability.*

*Compiled July 2026. The application layer atop the [pretraining](../pretraining-from-scratch/README.md), [midtraining](../midtraining-and-continued-pretraining/README.md), [post-training](../post-training-from-scratch/README.md), [inference](../inference-and-serving/README.md), [multimodal](../multimodal-training/README.md), and [cross-cutting](../cross-cutting-topics/README.md) series. Assumes you have (or can call) a capable served model; assumes nothing about building agents or RAG systems. This is the series for the far larger population that never trains a model at all.*

---

## Why this series exists

The other six series answer "how do I build a model?" This one answers the question most teams actually face: **"I have a model — how do I build something useful with it?"** By 2026 the unit of value has shifted from the model to the **compound AI system**: an application that wraps one or more models with retrieval, tools, memory, orchestration, and guardrails. The evidence that the system layer matters as much as the model: on identical models and identical tasks, framework/scaffold choice alone swings agent benchmark scores by up to **30 absolute percentage points** — the same Claude model scores 64.9% on GAIA in one orchestration scaffold and 57.6% in another. The application layer is not plumbing; it is a capability multiplier (or divider) as large as a model-generation jump.

This layer has its own disciplines with their own names now. **Context engineering** replaced prompt engineering as the practice of designing everything the model sees per call. **Agent harness engineering** covers tools, memory, permissions, and orchestration. **Agentic RAG** and **graph RAG** matured past naive vector search. And **agent evaluation** grew trajectory-level methods (pass^k, agent-as-judge) because single-response evals can't measure a system that acts over many steps. This series covers all of it.

## How to read this report

Every chapter is **layered**, like the companion series: conceptual picture first, then the practitioner layer — concrete patterns, numbers, and the decisions you'll face. Chapters 2–8 follow the order you'd build a system (context → tools → agent loop → memory → multi-agent → RAG → framework); Chapters 9–10 are the evaluation and reliability machinery underneath; Chapter 11 assembles playbooks.

A recurring theme, inherited from the whole knowledge base and sharpened here: **complexity must be earned by measurement.** The application layer offers an endless ladder of sophistication — agents, multi-agent, graph RAG, elaborate memory — and every rung adds cost, latency, and failure surface. The winning teams climb only when their evals prove the simpler rung insufficient: hybrid retrieval before agentic RAG, a workflow before an agent, one agent before many. Most production failures in this layer are self-inflicted complexity, not model limitations.

## Chapters

| # | File | Contents |
|---|---|---|
| 1 | [The Big Picture](01-big-picture.md) | Compound AI systems, the workflow-vs-agent spectrum, why the scaffold matters as much as the model |
| 2 | [Context Engineering](02-context-engineering.md) | The discipline that replaced prompting: write/select/compress/isolate, system prompts, prompt caching economics |
| 3 | [Tool Design](03-tool-design.md) | Designing tools agents use well, schemas and descriptions, MCP, permissions, tool-result hygiene |
| 4 | [Agent Architectures](04-agent-architectures.md) | The agent loop, ReAct lineage, planning, workflow patterns vs autonomous agents, when to use which |
| 5 | [Memory and State](05-memory-and-state.md) | Short vs long-term memory, compaction, durable state, session persistence, the four techniques applied |
| 6 | [Multi-Agent Systems](06-multi-agent-systems.md) | When multiple agents beat one, orchestrator-worker and subagent patterns, context isolation, the coordination tax |
| 7 | [RAG Systems](07-rag-systems.md) | The full retrieval architecture: chunking, hybrid search + reranking, agentic RAG, graph RAG, the escalation ladder |
| 8 | [Orchestration Frameworks](08-orchestration-frameworks.md) | LangGraph, Claude Agent SDK, CrewAI, Microsoft Agent Framework and friends; MCP/A2A; framework vs hand-rolled |
| 9 | [Evaluating Applications](09-evaluating-applications.md) | Trajectory evaluation, pass^k, agent-as-judge, judge biases, production labeling, statistical hygiene |
| 10 | [Production Reliability](10-production-reliability.md) | Observability and tracing, guardrails and HITL integration, durable execution, cost/latency control |
| 11 | [Playbooks by Scale](11-playbooks-by-scale.md) | Recipes: a RAG chatbot, a single production agent, an agentic RAG system, a multi-agent platform |
| 12 | [Failure Modes](12-failure-modes.md) | Context rot, tool confusion, runaway loops, false success, coordination collapse, eval theater |
| 13 | [Essential Reading and References](13-essential-reading-and-references.md) | The must-read sources, ordered, plus the full source list |

## Scope

This series covers building applications on models — yours or an API's. It leans on the companion series for everything below the application line: training an agentic *model* (post-training Ch 9), serving and structured output (inference Ch 8–9), the retrieval *model* itself (cross-cutting Ch 4), and securing the deployed system (cross-cutting Ch 6 — read it together with Chapters 3 and 10 here; agent security is where the two series meet). Model training appears here only as a pointer: when your application's failures trace to the model rather than the scaffold, the fix lives in the other series.

## The through-line

Application building rewards **earned complexity and measured trust**. Every capability you add — a tool, a memory, an agent, a second agent — is something the model can now do *wrong*, and the system's reliability is the product of every step's reliability: a 95%-reliable step, run twenty times, succeeds 36% of the time. The teams that ship agents that work start simple (a workflow, hybrid RAG, one agent), instrument everything (traces, trajectory evals, production labels), extend autonomy only as far as measurement justifies trusting it, and keep a human in the loop exactly where the blast radius demands one. Build the eval before the agent, earn each rung of complexity, and never confuse a demo that worked once with a system that works.
