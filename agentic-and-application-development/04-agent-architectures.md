# Chapter 4 — Agent Architectures

## 4.1 Conceptual picture

An agent, stripped to essentials, is a loop: the model receives a goal and its context, chooses an action (call a tool, or answer), the action executes, its result re-enters the context, and the loop repeats until the model decides the goal is met. That loop — sometimes called the ReAct pattern (reason, act, observe) — is the whole of agentic architecture; everything else is a constraint on it or a structure around it. This chapter is about *how much* loop to give a task, because the central decision (Chapter 1.2) is not "how do I build an agent" but "does this task want an agent at all, and if so, how constrained a one."

## 4.2 The agent loop

The canonical loop, concretely:

```
context = system_prompt + tools + goal
loop:
    action = model(context)          # reason + decide
    if action is final_answer: return it
    result = execute(action)          # act (tool call)
    context = context + action + result   # observe (append)
    if over_budget(steps, tokens, time): break  # the guardrail
```

Two things make it work in practice and their absence causes most failures: **the append step** (each turn's action and result join the context so the model sees its own history — Chapter 2's append-don't-edit) and **the budget guardrail** (a hard cap on steps/tokens/time so a confused agent halts instead of looping forever — the reliability arithmetic of Chapter 1.3 made operational). An agent without a budget is an outage waiting for a bad trajectory.

## 4.3 The workflow patterns (control-flow you own)

Before reaching for an autonomous loop, the workflow patterns — from the widely-cited "building effective agents" taxonomy — solve most real tasks with the model filling steps you sequenced:

- **Prompt chaining** — decompose a task into a fixed sequence of model calls, each consuming the last (outline → draft → polish). For tasks with a known decomposition.
- **Routing** — a classifier model directs input to one of several specialized paths (a support query routed to billing/technical/general handling). Separates concerns and lets each path be tuned.
- **Parallelization** — run independent model calls concurrently and aggregate (fan-out subtasks, or vote across several attempts). Cuts latency and improves reliability via voting.
- **Orchestrator-workers** — a model decomposes a task and dispatches subtasks to worker calls, then synthesizes. The bridge to multi-agent (Chapter 6) and the pattern for tasks whose *subtasks* are dynamic but whose *structure* (decompose-dispatch-synthesize) is fixed.
- **Evaluator-optimizer** — one model generates, another critiques, loop until the critique passes. Adds a verification gate — the highest-value structural reliability move (Chapter 1.3).

These are **workflows**: you own the graph, the model fills nodes. They dominate production because they're predictable, debuggable, cheap, and encode your knowledge of the process. Reach for them first.

## 4.4 Autonomous agents (control-flow the model owns)

When the path genuinely can't be predetermined — the number and order of steps depend on what each step reveals (open-ended research, debugging a failure, navigating a UI toward a goal, multi-hop investigation) — you give the model the loop and let it drive. What makes an autonomous agent work beyond the bare loop:

- **Planning.** For complex goals, have the agent produce a plan first (explicit sub-goals), then execute against it, re-planning as reality diverges. A plan in context keeps a long trajectory coherent and gives you something to inspect. Plan-then-execute beats pure step-by-step reactivity on long-horizon tasks — but over-planning on simple tasks wastes tokens; match planning depth to task complexity.
- **Reflection / self-correction.** Let the agent evaluate its own progress and recover from errors (retry with adjustment, backtrack, try another approach). This is the evaluator-optimizer pattern internalized into the loop, and it's what converts the multiplicative error chain (Chapter 1.3) into something survivable — an agent that catches and fixes its own mistakes doesn't compound them.
- **Verification against ground truth.** Where the task allows checking (tests pass, the query returns, the value matches), wire the check into the loop as the success signal — the agentic-RLVR insight (post-training Ch 8) applied to inference-time agents. Verification is the difference between an agent that *thinks* it succeeded and one that *did* (the false-success failure, Chapter 12).

## 4.5 Choosing where on the spectrum

The decision procedure, applying Chapter 1's rule (least autonomy that solves the task):

1. **Is the control flow knowable in advance?** If yes → a **workflow** (4.3). Most tasks. Don't use an agent to run a process you already understand.
2. **If not, is it because subtasks are dynamic but the structure is fixed?** → **orchestrator-workers** — the constrained middle, dynamic within a known shape.
3. **If the whole path is genuinely open-ended** → an **autonomous agent** (4.4), with planning, reflection, verification, and a hard budget.
4. **Regardless: what's the blast radius?** Higher autonomy demands tighter permissions and human-in-the-loop gates (Chapter 3.5, cross-cutting Ch 6) — you're trading reliability for flexibility, so contain the downside.

The anti-pattern the whole industry learned in 2024–2025 and codified by 2026: **defaulting to a maximally-autonomous agent because it's impressive.** It's less reliable, more expensive, harder to debug, and usually solves a structured problem worse than a workflow would. Autonomy is a cost you pay when the task forces it, not a feature you show off.

## 4.6 Decisions

1. **Start with a workflow** — prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer — for any task whose control flow you can predetermine. This is most tasks.
2. **Escalate to an autonomous agent only when the path is genuinely unknowable** in advance, and then give it planning, reflection, and verification, not just a bare loop.
3. **Always bound the loop** — hard caps on steps/tokens/time; an unbounded agent is an outage.
4. **Add a verification gate wherever the task allows checking** — it's the highest-leverage structural reliability move against compounding error.
5. **Match autonomy to blast radius** — tighter permissions and HITL gates as you give the model more control; spend autonomy, don't maximize it.

Next: memory and state — what the agent carries across the turns of its loop and across sessions.
