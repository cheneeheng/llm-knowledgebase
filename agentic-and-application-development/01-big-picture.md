# Chapter 1 — The Big Picture

## 1.1 From models to compound systems

A model, however capable, is a function: text in, text out. A **product** is a system: it knows your data (retrieval), acts on the world (tools), remembers (memory), follows a process (orchestration), stays within bounds (guardrails), and recovers when a step fails (reliability engineering). The 2026 term for this is the **compound AI system** — an application in which one or more model calls are components in a larger designed structure, and the structure contributes as much to the outcome as the models do.

The claim that the structure matters this much is empirical, not rhetorical. Princeton's HAL benchmark data shows the *same* model scoring 64.9% versus 57.6% on GAIA depending only on the orchestration scaffold around it, and framework choice swings agentic benchmarks by up to 30 absolute points on identical models and tasks. That spread is larger than the gap between model generations. For a team that consumes models rather than trains them, **the scaffold is the model-quality lever you actually control.**

This reframes the skill. The model-building series teach you to move a model's capability; this series teaches you to move a *system's* capability with the model held fixed — by engineering what the model sees (Chapter 2), what it can do (Chapter 3), how it loops (Chapter 4), what it remembers (Chapter 5), how instances coordinate (Chapter 6), what it retrieves (Chapter 7), and how you know any of it works (Chapter 9).

## 1.2 The workflow-vs-agent spectrum

The most important design axis in this series, and the source of the most expensive mistakes. Every LLM application sits somewhere on a spectrum of **who controls the control flow**:

- **A workflow** — *you* control the flow. The application is a fixed sequence (or graph) of steps, some of which are model calls: classify, then retrieve, then draft, then check. The model fills in steps; the code decides what happens next. Predictable, debuggable, cheap.
- **An agent** — the *model* controls the flow. You give it a goal, tools, and a loop; it decides which tool to call, interprets the result, and decides what to do next, for as many turns as the task takes. Flexible, powerful on open-ended tasks, and expensive in cost, latency, and failure surface.

The spectrum between them (a workflow with one agentic step; an agent constrained to a phase structure) is where most good systems live. The design rule, which recurs in every serious 2026 engineering guide: **use the least autonomy that solves the task.** Workflows beat agents wherever the task's structure is known in advance — they are more reliable *because* they encode your knowledge of the process instead of asking the model to rediscover it each time. Agents earn their cost only when the path genuinely cannot be predetermined: open-ended research, debugging, multi-step tasks whose shape depends on what each step reveals.

## 1.3 The reliability arithmetic

The single most sobering fact about agents, worth internalizing before building one: **errors compound multiplicatively over steps.** An agent whose individual steps are 95% reliable — an excellent per-step number — completes a 20-step task with probability 0.95²⁰ ≈ **36%**. At 50 steps, 8%. This arithmetic explains most of what you'll observe in this series:

- Why **workflows beat agents** on structured tasks (fewer model-controlled decisions = fewer chances to compound error).
- Why **short trajectories beat long ones** (Chapters 4–5: compaction, sub-task isolation, and checkpointing all exist to reset the compounding).
- Why **verification steps matter disproportionately** (a check that catches errors converts a multiplicative chain into something closer to independent retries).
- Why **pass^k evaluation** (Chapter 9) exists: an agent that passes a task once may fail it on repeats, and production runs the repeats.

Improving per-step reliability — better tools, better context, better models — helps, but the leverage is usually structural: fewer steps, isolated sub-tasks, verification gates, and recovery paths.

## 1.4 The layers of the stack

The application stack this series covers, bottom to top, with each chapter's home:

```
┌─────────────────────────────────────────────────┐
│ Evaluation & observability (ch 9–10)            │  ← wraps everything
├─────────────────────────────────────────────────┤
│ Orchestration framework (ch 8)                  │  LangGraph, Agent SDK, ...
├──────────────┬──────────────┬───────────────────┤
│ Agent loop & │ Multi-agent  │ Memory & state    │  ch 4, 6, 5
│ planning     │ coordination │                   │
├──────────────┴──────────────┴───────────────────┤
│ Tools (ch 3)          │ Retrieval / RAG (ch 7)  │
├───────────────────────┴─────────────────────────┤
│ Context engineering (ch 2)                      │  ← every model call
├─────────────────────────────────────────────────┤
│ The served model      (inference series)        │
└─────────────────────────────────────────────────┘
```

Context engineering is at the bottom deliberately: every layer above it ultimately works by shaping what lands in the model's context window on each call. An agent *is* a loop that repeatedly re-engineers its own context; memory *is* a policy for what enters context; RAG *is* retrieval into context. Master Chapter 2 and the rest of the stack becomes variations on a theme.

## 1.5 Core mental models

- **The context window is the interface.** The model has no state, no memory, no knowledge of your system beyond what's in the current call's context. Everything — tools, memory, retrieval, orchestration — is machinery for putting the right tokens in that window (and keeping the wrong ones out).
- **Autonomy is spent, not maximized.** Each increment of model-controlled decision-making buys flexibility and costs reliability, money, and debuggability. Spend it where the task demands it; hoard it everywhere else.
- **Complexity must be earned by measurement** (the through-line). The eval comes before the capability: you cannot know whether agentic RAG beats hybrid-plus-rerank, or two agents beat one, except by measuring on your task. Teams that add complexity on vibes accumulate failure surface with no measured gain.
- **The demo-production gap is the whole game.** Getting an agent to work once is easy and misleading; the discipline of Chapters 9–10 (trajectory evals, pass^k, tracing, guardrails) is what separates a demo from a system.

## 1.6 Decisions this series will force

1. **Where on the workflow-agent spectrum does each task belong?** (Ch 4)
2. **What does the model see per call, and what's cached?** (Ch 2)
3. **Which tools, designed how, with what permissions?** (Ch 3)
4. **What's remembered, compacted, persisted?** (Ch 5)
5. **One agent or several, coordinated how?** (Ch 6)
6. **Which RAG rung — hybrid+rerank, agentic, graph?** (Ch 7)
7. **Framework or hand-rolled, and which framework?** (Ch 8)
8. **How do you evaluate trajectories and label production?** (Ch 9)

Start with Chapter 2: context engineering is the substrate every other chapter builds on.
