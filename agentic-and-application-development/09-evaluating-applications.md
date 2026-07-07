# Chapter 9 — Evaluating Applications

## 9.1 Conceptual picture

Everything in the companion series about evaluation — held-out sets, contamination, judge biases, statistical hygiene — carries over, but applications add a dimension single-response evals can't measure: **the system acts over many steps, and the trajectory matters, not just the final answer.** An agent can reach the right answer by a lucky path that won't repeat, take forty steps where four sufficed, silently violate a policy en route, or — the most insidious — *report* success while the task is actually unfinished. Application evaluation is therefore trajectory evaluation, and it is the discipline that separates a demo (worked once, on stage) from a system (works repeatedly, unattended). Per the through-line, it comes *before* the capability: build the eval harness before the agent, because without it every architecture decision in Chapters 4–8 is a guess.

## 9.2 The levels of granularity

Agent evaluation happens at three levels, and a production harness uses all three:

- **Turn level** — is each individual step sound? Did the model pick the right tool, with the right arguments, and interpret the result correctly? Cheap to score (per-step, often rule-based or classifier-based), and the level at which failures are *diagnosable* — a trajectory-level failure tells you something broke; turn-level scoring tells you *where*.
- **Milestone level** — did the agent reach the intermediate states the task requires (found the relevant file, obtained the customer record, produced a plan)? Milestones decompose long tasks into checkable waypoints and localize failures to a phase without requiring a full reference trajectory.
- **Trajectory level** — did the whole run succeed, efficiently and within policy? Final-outcome correctness plus trajectory-quality metrics: step count versus the achievable minimum, loops and repeated actions, recovery after errors, tool-call accuracy, policy adherence, cost and latency.

The trap is evaluating only the top level: outcome-only scoring hides *how* success happened and gives you nothing to fix when it doesn't. Score turns and milestones too, and the trajectory metrics become explicable.

## 9.3 pass^k: the reliability metric

The single most important conceptual upgrade from model evals to agent evals. Standard pass@k asks "does at least one of k attempts succeed?" — the right question for *capability* (can it be done at all?). Agents in production face the opposite question: the same task arrives repeatedly, unattended, and every failure costs. τ-bench introduced **pass^k** — the probability that the agent succeeds on **all** of k independent attempts — "precisely because agents can pass once but fail on repeats."

The numbers are sobering and are the point: an agent with a healthy-looking 90% single-run success rate has pass^8 ≈ 43%. The gap between pass@1 and pass^k is exactly the demo-production gap of Chapter 1.6, quantified — and it's the Chapter 1.3 compounding arithmetic surfacing in evaluation. Practical consequences:

- **Run every eval task multiple times** (k of 4–8 is common) and report pass^k alongside pass@1. A single run per task measures capability plus luck.
- **Treat the variance itself as a signal.** High run-to-run variance on the same task means the agent's behavior is unstable — usually a context, tool-ambiguity, or planning problem worth diagnosing before it becomes a production incident.
- **Set product bars in pass^k terms.** "Works 90% of the time" sounds shippable; "fails at least once in most 8-task batches" is the same number and usually isn't.

## 9.4 Judging: rules first, then models, then agents

How trajectories get scored, in order of preference:

- **Rule-based / ground-truth checks first.** Wherever the task permits verification — the test suite passes, the database row has the right value, the API returned 200, the final state matches — check it mechanically. Deterministic, cheap, ungameable. This is the RLVR insight (post-training Ch 8) applied to evaluation, and it should carry as much of the load as the task allows.
- **LLM-as-judge for the unverifiable**, with all the post-training series' cautions intact: length, position, and self-preference biases; non-determinism; prompt sensitivity. Ground the judge with a rubric and reference, use pairwise comparison over absolute scores, and calibrate against a sample of human labels before trusting it.
- **Agent-as-judge for trajectories.** A plain LLM judge reads a transcript; an *agent* judge — with tools, multi-step reasoning, and state — can actually verify a trajectory: re-run the reproduction step, inspect the artifact the agent claimed to produce, check intermediate actions. Agent-as-judge evaluates "the entire reasoning trajectory, capturing decision-making processes and intermediate actions," not just the final output, and is the emerging standard for deep trajectory review. It costs a full agentic run per judgment — reserve it for offline evaluation and auditing samples, not per-turn production scoring.

The cost gradient is the design constraint: rules run everywhere, LLM judges run on samples and offline suites, agent judges run on the slices that matter most.

## 9.5 False success: the failure your evals must target

The characteristic agent-evaluation failure, now named in the literature: **false success** — the agent *reports* completion while the task is not actually done. It closed the ticket without fixing the issue; it claimed the file was updated but wrote to the wrong path; it summarized a refund as issued that was never issued. False success is worse than honest failure because it passes outcome-only evals that trust the agent's self-report, and in production it silently converts task failures into data corruption and broken promises.

Defenses, layered: **never score success from the agent's self-report** — verify against ground truth (the environment's actual state, not the transcript's claims); **prefer state-based checks over transcript-based checks** in your rule layer; **use agent-as-judge to audit** samples of "successful" trajectories specifically hunting for unverified claims; and **instrument production for downstream contradiction signals** (the user reopens the ticket, the follow-up API call 404s) that reveal false successes after the fact. If your eval harness has one specialty, make it this.

## 9.6 The production loop: benchmarks are not enough

Public agent benchmarks (τ-bench and τ²-bench for tool-use with simulated users, SWE-bench for software tasks, GAIA for general assistance, AgentBench) are useful for orientation and model selection — but they measure their tasks, not yours, and they're static while your traffic isn't. The harness that actually protects a product is built from production, mirroring the data flywheel (cross-cutting Ch 8.4):

1. **Trace everything** (Chapter 10) — every trajectory captured with its spans: model calls, tool calls, arguments, results.
2. **Label production turns** — a cheap per-turn classifier labels every production interaction (success/failure/escalation/policy-flag), turning raw traffic into a continuously-scored stream.
3. **Route labels into the eval set** — failures and interesting cases become new eval tasks; your suite grows to cover reality rather than your launch-day imagination.
4. **Gate changes on the suite** — every prompt edit, tool change, model swap, or framework upgrade runs the eval suite (with pass^k) before shipping. Applications change weekly; without a regression gate, every change is a gamble.
5. **Keep a held-out slice** the optimization loop never sees — the same honesty discipline as every other series, because tuning prompts against your full eval set overfits it just like training does.

Tooling — LangSmith, Braintrust, Phoenix, DeepEval, RAGAS (for the RAG layer, Chapter 7.6) — provides the trace-capture, scoring, and suite-management machinery; the *policy* (what to label, what to gate on, what pass^k bar to hold) is yours.

## 9.7 Decisions

1. **Build the eval harness before the agent** — every architecture choice in this series is a guess without one.
2. **Score at all three levels** — turn, milestone, trajectory — so failures are diagnosable, and track trajectory-quality metrics (steps vs minimum, loops, recovery), not just outcomes.
3. **Report pass^k, not just pass@1** — run tasks 4–8 times; set product bars on all-k reliability, and treat run-to-run variance as a defect signal.
4. **Layer judges by cost** — ground-truth rules everywhere, calibrated LLM judges on samples, agent-as-judge for deep trajectory audits — and **never score success from the agent's self-report** (false success is the failure your harness must specialize in).
5. **Grow the suite from labeled production traffic and gate every change on it**, keeping a held-out slice the tuning loop never sees.

Next: production reliability — the observability, guardrails, and recovery machinery that keeps a measured system running.
