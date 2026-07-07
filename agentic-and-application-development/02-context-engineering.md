# Chapter 2 — Context Engineering

## 2.1 Conceptual picture

Context engineering is "the deliberate design of everything a language model sees on every inference call — system prompt, user input, retrieved documents, conversation history, tool definitions, and long-term memory." It is the 2026 successor to prompt engineering, and the rename marks a real shift in scope: prompt engineering was about wording one message well; context engineering is about *curating an entire information environment* per call, under a hard token budget, across a system that may make thousands of calls. Since the context window is the model's only interface (Chapter 1.5), this discipline is the substrate of everything else in this series.

The core tension: **more context is not better context.** Models degrade as windows fill — relevant facts get lost among irrelevant ones, early instructions get buried, stale tool results mislead — a cluster of failures practitioners call *context rot*. The craft is getting the *right* information in, keeping everything else out, and doing it economically. A useful slogan from the practitioner literature: treat context as a workspace, not a warehouse.

## 2.2 The four techniques

The 2026 consensus decomposes context engineering into four moves; every pattern in this chapter (and most of Chapters 5–7) is one of them:

- **Write** — persist information *outside* the window (files, scratchpads, memory stores, state objects) so it isn't lost when the window turns over. The agent's notebook. (Chapter 5 builds on this.)
- **Select** — retrieve into the window only what's relevant *now*: RAG for documents (Chapter 7), memory retrieval for history, dynamic tool selection when the tool catalog is large. Selection is the defense against warehouse-context.
- **Compress** — summarize or distill to save tokens: replace a completed step's long trace with a short verified summary, compact old conversation turns, trim bulky tool results that stopped mattering. (Compaction, 2.5.)
- **Isolate** — give sub-tasks their own clean context instead of one shared window: sub-agents with fresh windows (Chapter 6), sandboxed side-computations, separate contexts per concern. Isolation is the defense against cross-contamination and rot.

## 2.3 Anatomy of a well-built context

What a production call's context contains, in the order that works (position matters — models weight beginnings and ends most):

1. **System prompt** — identity, task, rules, output format. Keep it *stable* (byte-identical across calls) both for behavior consistency and for prompt caching (2.4). Structured and terse beats eloquent and long; every rule should be one the model demonstrably needs (test by removal).
2. **Policies and permissions in structured fields** — what the agent may/may not do, "in structured fields so they do not get buried by conversation" (the Anthropic managed-agents guidance). Buried permissions are unenforced permissions.
3. **Tool definitions** — the tools relevant to *this* task (Chapter 3), not the whole catalog.
4. **Selected knowledge** — retrieved chunks, memory, prior-step summaries. Selected and compressed, not dumped.
5. **Recent history and current state** — with "active goals, the next action, and the minimum proof needed for that action at the top" of the working section.
6. **The current input** — the user turn or the latest tool result.

The recurring verification ritual, application-layer edition: **log and read the fully-assembled context** for a sample of real calls. Teams are routinely shocked by what's actually in the window — duplicate history, stale tool dumps, a system prompt fighting a retrieved document. You cannot engineer what you don't look at.

## 2.4 Prompt caching economics

Prompt caching (inference series Ch 3) is a serving feature, but *exploiting* it is a context-engineering responsibility, and the numbers make it a first-order design constraint: a 2026 study across 500+ long-horizon agent sessions found caching cut API cost **41–80%** and improved time-to-first-token 13–31%. The design rules that unlock it:

- **Stable prefix, volatile suffix.** Caches match byte-identical *prefixes*. Put everything stable (system prompt, policies, tool definitions, few-shot examples) first and never vary it; put everything volatile (history, retrieval, the user turn) after. One dynamic timestamp at the top of a system prompt destroys the entire cache.
- **Append, don't edit.** In an agent loop, appending new turns preserves the cached prefix; rewriting earlier context invalidates it. This is a genuine tension with compaction (which rewrites) — the resolution is to compact *in batches, at phase boundaries*, paying the cache miss once rather than per-turn.
- **Order tool definitions and documents deterministically** — a shuffled tool list is a cache miss.

For an agent making hundreds of calls per task, cache discipline is frequently the difference between viable and unviable unit economics.

## 2.5 Compaction and long-horizon hygiene

For long-running tasks, the window *will* fill, and the question is what to do about it. The compaction playbook (Anthropic's managed-agents guidance and the practitioner consensus):

- **Compact at boundaries, not continuously** — when a step/phase completes, replace its long trace (tool outputs, exploration, dead ends) with a short verified summary of what was learned and decided. Mid-step compaction loses working state.
- **Clear stale tool results first** — old tool outputs are the bulkiest, lowest-value tokens once the task has moved on; they're the first thing to trim.
- **Preserve the invariants** — goals, constraints, decisions made, and open questions survive every compaction verbatim. Losing a constraint to a summary is how agents violate instructions late in long tasks.
- **Write before you compress** — anything that might matter later goes to external storage (the *write* move) before its in-context form is compacted. Compaction should lose *tokens*, not *information*.
- **Prefer isolation to heroic compaction** — if a task keeps blowing the window, the better fix is usually structural: split it into sub-tasks with their own clean contexts (Chapter 6) rather than ever-more-aggressive summarization of one giant window.

## 2.6 Decisions

1. **Design the whole context, not the prompt** — own everything in the window per call, and read assembled contexts from production regularly.
2. **Apply the four moves deliberately** — write, select, compress, isolate — and name which one you're doing when you change the context design.
3. **Structure for position and stability** — stable prefix (system, policies, tools) first and byte-identical; goals and next-action at the top of the working section; policies in structured fields.
4. **Engineer for the cache** — stable-prefix/volatile-suffix ordering, append-don't-edit, batch compaction at phase boundaries; the 41–80% cost swing makes this non-optional for agents.
5. **Compact at boundaries, preserve invariants, write before compressing** — and reach for context isolation before heroic summarization.

Next: tool design — the other half of what the model sees, and the half that lets it act.
