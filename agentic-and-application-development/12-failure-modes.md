# Chapter 12 — Failure Modes

Application-layer failures have a distinctive character: the model is usually fine, and the *system* around it is what broke — the wrong thing in context, a tool the model couldn't use correctly, a loop nobody bounded, an eval that measured the wrong thing. They are also predominantly *semantic* — the system runs green while doing the wrong thing — so most entries below are invisible to infrastructure monitoring and caught only by the tracing, labeling, and trajectory evals of Chapters 9–10. Each entry: symptom → cause → fix. The three roots most trace to: **unmanaged context, unearned autonomy, and unverified success.**

---

## 12.1 Context rot

**Symptom:** the agent gets worse as the session grows — forgets constraints stated early, repeats itself, gets distracted by stale information, quality degrades late in long tasks.
**Cause:** the window filled with unmanaged accumulation — old tool dumps, dead-end exploration, duplicate history — burying the goal and constraints (Chapter 2.1).
**Fix:** the Chapter 2.5 playbook — compact at phase boundaries, clear stale tool results first, keep goals/constraints/decisions verbatim at the top, write-before-compress, and prefer sub-task isolation over heroic summarization. Diagnose by reading assembled contexts from failing sessions; the rot is visible the moment you look.

---

## 12.2 Tool confusion

**Symptom:** wrong tool chosen, valid-but-nonsensical arguments, the same tool failing repeatedly, or the agent avoiding a tool that would trivially solve the step.
**Cause:** tool-design failure (Chapter 3) — vague descriptions, endpoint-grained tools that require assembly, an oversized in-context catalog, inconsistent parameter naming, or error messages that teach nothing.
**Fix:** rewrite descriptions as docstring-quality interfaces with when-to-use guidance; consolidate to intention-level tools; shrink or dynamically select the catalog; make errors corrective instructions. Diagnose from traces: turn-level tool-choice scoring (Chapter 9.2) localizes exactly which tool confuses.

---

## 12.3 Runaway loops

**Symptom:** trajectories with 40 steps where 6 would do; the agent retrying a failing action verbatim; cost per task spiking; tasks that never terminate.
**Cause:** no budget guardrail (Chapter 4.2), no loop detection, and retries that don't vary (the model re-attempts the identical failing call because nothing in context tells it why it failed).
**Fix:** hard caps on steps/tokens/cost/time in code; loop detection (same action twice → intervene); errors-as-instructions so retries differ (Chapter 3.2); escalate after N semantic failures instead of persisting. Monitor steps-per-task and loop rate as standing metrics (Chapter 10.2).

---

## 12.4 False success

**Symptom:** the agent reports the task complete; the ticket is closed, the summary is confident — and the work wasn't actually done. Discovered later via user complaints or downstream contradictions.
**Cause:** success scored from the agent's self-report rather than verified state (Chapter 9.5); no verification step in the loop; outcome-only evals that trusted the transcript.
**Fix:** state-based ground-truth checks everywhere (in the loop as verification gates, Chapter 4.4, and in evals); agent-as-judge audits of "successful" trajectories; production monitoring for contradiction signals (reopened tickets, failing follow-ups). The single most damaging agent failure because it converts failure into silent data corruption — build your harness around it.

---

## 12.5 The compounding cliff

**Symptom:** the agent aces short tasks and collapses on long ones; success rate falls off a cliff past some step count.
**Cause:** the Chapter 1.3 arithmetic — per-step reliability compounding multiplicatively — usually aggravated by context rot deepening as the trajectory lengthens (12.1 and 1.3 reinforce each other).
**Fix:** structural, not model-level — decompose into shorter verified phases (milestones, Chapter 9.2), checkpoint at boundaries, isolate sub-tasks into fresh contexts, add mid-trajectory verification gates that catch errors before they compound. If long-horizon performance is the requirement, architect for it; don't expect it from a longer loop.

---

## 12.6 Coordination collapse (multi-agent)

**Symptom:** a multi-agent system performs worse than the single agent it replaced — workers duplicate or contradict each other, the orchestrator synthesizes garbage from a bad worker result, nobody can tell where it went wrong.
**Cause:** unearned multi-agent adoption (Chapter 6.1) — sub-tasks weren't independent, hand-offs lost critical state, worker results went unverified, hierarchy too deep, no cross-agent tracing.
**Fix:** first ask whether one agent (or a workflow) does the job — often yes; if multi-agent is earned, enforce Chapter 6.5: isolated contexts, concise verified hand-offs, shallow hierarchy, system-wide budgets, stitched traces. Coordination collapse is almost always a symptom of complexity adopted on vibes.

---

## 12.7 Prompt-level enforcement

**Symptom:** the agent violates a rule it was explicitly given — deletes without asking, contacts a customer it shouldn't, exceeds a spending limit — sometimes after politely acknowledging the rule.
**Cause:** the rule lived only in the prompt. Prompts are probabilistic; long contexts bury instructions (12.1); injection can override them (cross-cutting Ch 6).
**Fix:** *prompts request, code enforces* (Chapter 10.3) — permission tiers and action gates in code, HITL at consequence boundaries, budgets in the loop. Audit: list every rule you rely on and check which are enforced by a gate versus merely requested; the requested ones are the incidents you haven't had yet.

---

## 12.8 Retrieval rot and RAG drift

**Symptom:** a RAG system that launched accurate degrades over months — answers cite outdated documents, misses new content, faithfulness scores slip.
**Cause:** the index diverged from reality — content changed but wasn't re-ingested, chunking/embedder changes were never backfilled, the corpus grew past what the retrieval configuration was tuned for.
**Fix:** treat the index as a production system: ingestion pipelines with freshness monitoring, re-run retrieval evals (Recall@K, RAGAS) on a schedule against a maintained golden set, version the index alongside the embedder (cross-cutting Ch 4.6). RAG quality is perishable; only a standing eval notices the decay.

---

## 12.9 Eval theater

**Symptom:** the eval suite is green; users are unhappy. Changes ship confidently and regress production.
**Cause:** the suite measures the wrong things — outcome-only (misses trajectories), pass@1-only (misses reliability, Chapter 9.3), built from launch-day guesses rather than production reality (Chapter 9.6), or overfit because the tuning loop saw the whole set.
**Fix:** the Chapter 9 stack in full — trajectory + milestone scoring, pass^k bars, suites grown from labeled production failures, a held-out slice the loop never sees, and weekly human reads of real trajectories. An eval suite that never disagrees with your expectations is measuring your expectations.

---

## 12.10 The cost spiral

**Symptom:** unit economics quietly invert — the feature works but each task costs multiples of what it returns; the bill grows faster than usage.
**Cause:** unmonitored fan-out (one request → dozens of calls), frontier-model usage on trivial steps, cache-hostile context layout (a mutating prefix), and runaway trajectories (12.3) — none visible when only cost-per-token is watched.
**Fix:** Chapter 10.5 — track cost-per-completed-task per feature from day one, per-step model routing, cache-discipline (stable prefix; verify the hit rate), per-task budgets. The spiral is a measurement failure before it is a spending failure.

---

## 12.11 The meta-lesson

Ten failures, three disciplines. **Manage the context** (2.5's hygiene prevents rot and half the compounding cliff). **Earn and bound autonomy** (budgets, gates-in-code, workflow-first, multi-agent-last prevent runaways, violations, and coordination collapse). **Verify — never trust self-report** (state-based checks, pass^k, production labeling prevent false success and eval theater). Every entry above is one of these three, unpracticed. The application layer has no exotic failures — it has the predictable consequences of giving a probabilistic component control without measurement, and every one of them is preventable by the machinery this series described.

Next: essential reading and references.
