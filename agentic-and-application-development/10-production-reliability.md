# Chapter 10 — Production Reliability

## 10.1 Conceptual picture

An evaluated application (Chapter 9) is one you know works on your test distribution; a *reliable* one keeps working on live traffic, under adversaries, at a cost you can pay, while you watch. This chapter is the operational layer for agentic systems — observability, guardrails wired into the loop, recovery, and cost/latency control. It parallels the inference series' production chapter (Ch 11) one level up: that chapter keeps the *model endpoint* healthy; this one keeps the *system built on it* healthy. The two failure characters differ: endpoints fail loudly (OOM, latency cliffs); applications fail *semantically* — the agent does the wrong thing fluently — which is why observability here means capturing meaning (trajectories, decisions, claims), not just metrics.

## 10.2 Observability: traces are the unit

The foundational practice, on which Chapter 9's whole production loop depends: **capture a full trace of every trajectory** — the nested span tree of model calls (with assembled contexts), tool calls (with arguments and results), decisions, retries, and hand-offs (across agents, Chapter 6.5). OpenTelemetry-based LLM tracing plus a platform (LangSmith, Braintrust, Phoenix) is the standard stack. What traces buy you:

- **Debuggability** — a semantic failure is diagnosed by reading the trajectory: where did the context go wrong, which tool result misled, where did the plan derail. Without the trace there is nothing to read.
- **The eval flywheel** — labeled traces become eval tasks (Chapter 9.6).
- **The metrics that matter**, aggregated from traces: task success/escalation rates, steps-per-task versus baseline, loop frequency, tool-error rates, token cost per task, latency per task, and per-turn classifier labels. Alert on *distribution shifts* in these — a rising step count or tool-error rate is an early warning of semantic degradation that endpoint metrics never see.
- **Audit** — for gated actions and compliance (cross-cutting Ch 7.5), the trace is the record of what the agent did and why.

Sample-read trajectories weekly even when metrics look healthy — the read-the-transcripts discipline from post-training, now applied to production. Metrics summarize; reading catches what you didn't think to measure.

## 10.3 Guardrails wired into the loop

Cross-cutting Chapter 6 gave the security defense stack; here is where it physically attaches to the agent loop of Chapter 4.2:

- **Before the model call** — input screening (injection/jailbreak detection) on user input *and* on retrieved/tool-returned content entering the context (indirect injection rides the data path, not the user path).
- **After the model call, before execution** — the action gate: validate every tool call against policy (allowed tool? allowed arguments? within tier? — Chapter 3.5) in *code*. This is the single highest-value guardrail placement, because it intercepts text becoming action.
- **At consequence boundaries** — HITL approval for irreversible/outward-facing actions, implemented as a durable pause (Chapter 5.4), not a prompt-level plea.
- **After generation** — output screening (policy, PII, grounding checks) before anything reaches the user, with interpretability probes as cheap upstream signals where available (cross-cutting Ch 5.3).
- **Around the whole loop** — the budget guardrail (steps/tokens/time/cost caps) and anomaly cutoffs (kill a trajectory whose cost or step count exceeds percentile bounds).

The design principle, inherited and operationalized: **prompts request, code enforces.** Every safety property you actually rely on must live in a gate the model cannot talk its way past.

## 10.4 Recovery and graceful degradation

Agents run long and touch flaky things (tools, APIs, models); reliability is mostly designed recovery:

- **Retries with discrimination** — transient tool/API failures retry with backoff; *semantic* failures (the model misused the tool) retry with the error-as-instruction (Chapter 3.2) so the next attempt differs; repeated semantic failure escalates rather than loops.
- **Checkpoint-and-resume** — durable state at step boundaries (Chapter 5.4) turns a crash from a lost task into a resumed one; idempotent-or-logged actions make the resume safe.
- **Fallback chains** — model unavailable or degraded → fall back to an alternate model (the Agent SDK's fallback chains; the portfolio idea from cross-cutting Ch 2.2); tool down → degrade to a reduced-capability mode with the user informed, rather than a hard failure.
- **Escalation as a first-class outcome** — the agent (or its guardrails) hands off to a human with the trajectory context attached. A well-designed escalation is a success mode, not a failure: an agent that reliably knows when to stop and hand off outperforms, in production value, one that pushes through and false-succeeds (Chapter 9.5). Design the hand-off path with the same care as the happy path, and measure escalation rate as a health metric in both directions (too high = incapable; too low = overconfident).

## 10.5 Cost and latency control

The application layer multiplies the inference series' economics: one user request can fan out into dozens-to-hundreds of model calls (agent turns, sub-agents, retrieval, judges). The controls:

- **Per-task budgets** enforced in the loop (10.3) — cost caps are a reliability control *and* a financial one; runaway trajectories are both.
- **Model routing by step difficulty** — the hybrid-portfolio idea (cross-cutting Ch 2.2) inside a single trajectory: cheap/small models for classification, extraction, labeling, and summarization steps; the frontier model for the reasoning-heavy steps. Most steps in most trajectories don't need the big model, and per-step routing routinely halves task cost.
- **Cache discipline** (Chapter 2.4) — the 41–80% swing makes prompt-cache-aware context layout the single largest cost lever in long-horizon agents.
- **Latency shaping** — parallelize independent steps (Chapter 4.3), stream intermediate progress to the user (perceived latency is what users feel), and pre-compute what you can (warm retrieval, cached sub-results). An agent that takes ninety silent seconds reads as broken; the same agent narrating its steps reads as working.
- **Watch cost-per-task, not cost-per-token** — the unit economics of an agentic product are per *task completed within SLO*, the application-layer goodput. Track it per feature and per cohort (10.2's metrics), and feed it to the FinOps loop (inference Ch 12.5).

## 10.6 Decisions

1. **Trace every trajectory** (spans for every model/tool call, stitched across agents) and alert on distribution shifts in semantic metrics — steps-per-task, loop rate, tool errors, escalations; sample-read trajectories weekly regardless.
2. **Wire guardrails at the loop's joints** — screen inputs and ingested content, gate every tool call in code, HITL at consequence boundaries, screen outputs, cap budgets. Prompts request; code enforces.
3. **Design recovery, not just success** — discriminating retries, checkpoint-and-resume with idempotent actions, model/tool fallback chains, and escalation as a measured, first-class outcome.
4. **Control cost structurally** — per-task budgets, per-step model routing, cache-aware context layout, parallelism and streaming for latency; track cost-per-completed-task as the unit economic.
5. **Treat semantic degradation as the primary production risk** — endpoint metrics won't see it; your traces, labels, and weekly reads will.

Next: playbooks — the whole series assembled into concrete builds by scale.
