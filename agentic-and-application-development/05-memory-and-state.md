# Chapter 5 — Memory and State

## 5.1 Conceptual picture

Models are stateless — each call knows only what's in its context (Chapter 1.5). Memory is the machinery that gives an application continuity anyway: within a task (the agent remembers what it just did), across a session (the assistant remembers this conversation), and across sessions (it remembers *you*, your preferences, prior work). All of it reduces to Chapter 2's *write* and *select* moves — persist information outside the window, retrieve the relevant piece back in when needed — but the design choices around what to store, how to retrieve, and when to forget are involved enough to warrant their own chapter. Memory done well is invisible; done poorly it's either amnesia (the agent forgets a constraint mid-task) or context rot (irrelevant history crowds out the current task).

## 5.2 The memory hierarchy

Three timescales, three mechanisms:

- **Working memory (within-task)** — the agent's context window itself plus a scratchpad: the current goal, plan, recent steps, and a notebook for intermediate results (the *write* move for anything that must survive compaction). Managed by context engineering (Chapter 2) and compaction (2.5). Lifespan: the task.
- **Session memory (within-conversation)** — the conversation history for the current session, compacted as it grows. Lifespan: the session. The main tool is compaction — summarize old turns, keep recent ones verbatim, preserve invariants.
- **Long-term memory (across sessions)** — durable knowledge about the user, past interactions, learned facts, and preferences, stored in an external store and *selectively retrieved* into context when relevant. Lifespan: indefinite. This is where "the assistant remembers me" lives, and where most of the design difficulty is.

## 5.3 Long-term memory: store, retrieve, forget

The hard tier. Three operations:

**Store — decide what's worth remembering.** Not everything; a memory of every turn is a warehouse that retrieval can't navigate. Store *salient, durable* facts: stated preferences, key decisions, stable attributes, corrections the user made. The extraction is itself a model call (summarize this session into memory-worthy facts) with a bias toward *few, high-value* memories over many marginal ones. Over-storing is the dominant long-term-memory failure.

**Retrieve — surface the relevant memory now.** Same machinery as RAG (Chapter 7): embed memories, retrieve by similarity to the current context, optionally rerank. The subtlety is *relevance to the moment* — the user asking about a recipe doesn't need their work preferences retrieved. Precision matters more than recall here; a wrongly-retrieved memory is active noise (and a privacy risk if memories cross users).

**Forget — expire and update.** Memory that only grows, decays. Preferences change, facts go stale, corrections supersede. Real systems need update (this new fact replaces that old one) and expiry (retention limits — also a compliance requirement, cross-cutting Ch 7: memory of a user is personal data with deletion obligations). Design forgetting from the start; retrofitting deletion into a memory store is painful.

Frameworks and services (mem0-class memory layers, and framework-native memory in LangGraph/Agent SDK) provide store/retrieve/forget as primitives — use them rather than hand-building, but understand the operations, because the *policy* (what to store, when to forget) is yours and determines whether memory helps or hurts.

## 5.4 Durable state and long-horizon execution

Distinct from memory-as-knowledge is **execution state** — where an agent *is* in a long task, so it can survive interruptions. A task running for minutes or hours (or waiting on a human, or a slow tool) must not lose everything if the process restarts. This is why "durable state and human-in-the-loop as primitives" is a 2026 framework-selection threshold (Chapter 8): the framework checkpoints the agent's state (its context, plan, progress) durably, so execution can pause, resume, and recover.

The design implications:
- **Checkpoint at step boundaries** — after each tool call/decision, persist enough state to resume. Ties to reliability (Chapter 10) and to the compounding-error mitigation of restarting from a good checkpoint rather than the top.
- **Make steps idempotent or logged** — on resume, you must not double-execute a side-effecting action; log completed actions and skip them on replay (the same discipline as the training series' resumable data pipelines, applied to agent actions).
- **Human-in-the-loop is a durable pause** — an agent waiting for approval (Chapter 3.5) is a durably-checkpointed agent that resumes when the human responds, possibly much later. This only works if state is durable.

## 5.5 The unifying view: memory is a context policy

Everything in this chapter is one question: **what enters the context window, from where, when?** Working memory keeps it in; session memory compacts it; long-term memory retrieves it back; durable state reconstructs it after a pause. Seen this way, memory isn't a separate subsystem — it's the *policy layer* over Chapter 2's write/select/compress moves, deciding across time what the model gets to see. The failures are correspondingly the same: too little (amnesia — a dropped constraint, a forgotten decision) or too much of the wrong thing (rot — stale or irrelevant memory as noise). The discipline is the same too: read the assembled context (Chapter 2.3), and check what memory put there.

## 5.6 Decisions

1. **Match the mechanism to the timescale** — working memory (scratchpad + compaction) within-task, session memory (compaction) within-conversation, long-term memory (external store + retrieval) across sessions.
2. **For long-term memory, store few high-value facts, retrieve for present relevance, and design forgetting** (update + expiry) from the start — over-storing and never-forgetting are the dominant failures, and expiry is also a compliance requirement.
3. **Use a memory framework/service for the primitives** but own the policy — what to store and when to forget is yours and determines whether memory helps.
4. **Make long-horizon execution durable** — checkpoint state at step boundaries, make actions idempotent-or-logged to survive resume without double-execution, and treat HITL as a durable pause.
5. **Think of memory as a context policy** — every memory decision is a decision about what enters the window; read assembled contexts to verify it's working.

Next: multi-agent systems — when one context and one loop aren't enough, and the coordination tax you pay for more.
