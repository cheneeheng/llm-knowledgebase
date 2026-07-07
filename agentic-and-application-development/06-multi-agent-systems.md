# Chapter 6 — Multi-Agent Systems

## 6.1 Conceptual picture

A multi-agent system uses several model instances — often with different roles, tools, or prompts — coordinating on a task, instead of one agent doing everything. The appeal is intuitive: specialization (a researcher, a coder, a critic), parallelism (many sub-tasks at once), and context isolation (each agent gets a clean window for its concern). The reality is more sober: multi-agent adds a **coordination tax** — communication overhead, error propagation between agents, harder debugging, and multiplied cost — that often exceeds the benefit. This chapter is about the narrow set of cases where multiple agents genuinely beat one well-built agent, and how to structure them when they do. The governing rule (Chapter 1.2, restated): **one agent until measurement proves you need more.**

## 6.2 When multiple agents actually help

The honest list, because the default should be *one* agent:

- **Context isolation for parallel sub-tasks.** The strongest case. When a task splits into independent sub-tasks each needing substantial, *different* context (research five topics, each with its own documents and exploration), giving each a sub-agent with a fresh window keeps any one window from bloating and rotting (Chapter 2). This is the *isolate* move at the agent level, and it's where multi-agent earns its keep — Anthropic's research-agent system is the canonical example (an orchestrator spawning sub-agents that each research independently and report back concise findings).
- **Genuine parallelism for latency.** Independent sub-tasks run concurrently across agents, cutting wall-clock time where a single agent would serialize them.
- **Strong specialization with verification.** A generator agent and a separate critic/verifier agent (the evaluator-optimizer pattern across instances) — separation helps when the critic benefits from a fresh perspective and its own tools/context.

The cases where it *doesn't* help, despite temptation: sequential tasks with shared context (the coordination overhead buys nothing — a single agent with the whole context does better); tasks small enough to fit one window (isolation solves a problem you don't have); and anything where the sub-agents must tightly share evolving state (passing it between agents is lossy and expensive — keep it in one context).

## 6.3 The patterns

Two structures cover most production multi-agent systems:

**Orchestrator-worker (the dominant pattern).** A lead agent decomposes the task, spawns worker sub-agents for sub-tasks, and synthesizes their results. Workers have isolated contexts and report back *concise* results (not their full trajectories — the orchestrator's window must not accumulate every worker's raw work). This is orchestrator-workers (Chapter 4.3) escalated from calls to agents, and it's what Claude Agent SDK's "hierarchical subagent spawning (up to 3 levels deep)" formalizes. Depth is a cost: each level adds coordination and error surface, so keep hierarchies shallow (2–3 levels max) and justified.

**Role-based collaboration.** Several agents with fixed roles (researcher, writer, reviewer) pass work between them, often in a defined sequence or a shared "conversation." CrewAI-class frameworks specialize in this for prototyping. It reads naturally and prototypes fast, but the fixed-role conversation is often a *workflow* wearing a multi-agent costume — and if the roles are a known sequence, an explicit workflow (Chapter 4.3) is more reliable than agents chatting. Use role-based collaboration when the interaction is genuinely dynamic; use a workflow when it's a fixed pipeline.

## 6.4 The coordination tax

What you pay, and why it caps how far to scale agents:

- **Error propagation.** One agent's mistake becomes another's input; the compounding-error arithmetic (Chapter 1.3) now compounds *across* agents too. A wrong result from a worker, trusted by the orchestrator, corrupts the synthesis. Verification between agents (does this worker's result make sense?) is the mitigation, at added cost.
- **Communication overhead.** Every inter-agent message is tokens and latency; agents summarizing for each other lose information at each hop. Rich shared state is expensive to pass and lossy — which is *why* isolation works only when sub-tasks are genuinely independent.
- **Cost multiplication.** N agents is roughly N× the model calls (often more, with coordination overhead). Multi-agent systems are markedly more expensive per task — justified only when the quality or latency gain is measured and worth it.
- **Debuggability collapse.** A single agent's trajectory is hard enough to debug (Chapter 9); interacting agents produce a tangle of trajectories where failures are hard to localize. Observability (Chapter 10) must span agents, and even then multi-agent systems are the hardest thing in this series to operate.

## 6.5 Designing multi-agent systems that work

When you've earned it by measurement, the practices:

- **Isolate contexts, share concise results.** Each agent gets a clean, task-scoped window; hand-offs are compressed summaries, not raw trajectories. This is the entire point — preserve it.
- **Keep hierarchies shallow and structured.** Orchestrator-worker, 2–3 levels, clear responsibility per agent. Avoid free-for-all agent "societies"; they're research-interesting and production-fragile.
- **Verify at hand-offs.** Treat a worker's result as untrusted until checked (echoing tool-result hygiene, Chapter 3.3, and inter-agent security, cross-cutting Ch 6).
- **Bound the whole system.** Per-agent *and* system-wide budgets (steps, cost, time) — a multi-agent system has more ways to run away than a single agent.
- **Trace across agents** (Chapter 10) — you cannot operate what you can't see, and multi-agent observability must stitch the sub-trajectories into one view.

## 6.6 Decisions

1. **Default to one well-built agent**; adopt multi-agent only when measurement shows a single agent can't meet the bar — usually because sub-tasks need isolated, different, substantial contexts.
2. **Use orchestrator-worker with concise result hand-offs and isolated worker contexts** as the primary pattern; keep hierarchies shallow (2–3 levels).
3. **Prefer an explicit workflow to role-based agent collaboration** when the interaction is a fixed sequence — reserve collaborating agents for genuinely dynamic interaction.
4. **Pay the coordination tax knowingly** — verify inter-agent hand-offs, budget the whole system, and expect N×+ cost and much harder debugging.
5. **Trace across agents** and treat multi-agent as the highest-complexity rung — climb it last and only with the evals to prove it paid.

Next: RAG systems — the retrieval architecture that grounds agents and applications in your data, with its own earn-the-complexity ladder.
