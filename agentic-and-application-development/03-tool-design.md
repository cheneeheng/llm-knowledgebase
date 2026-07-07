# Chapter 3 — Tool Design

## 3.1 Conceptual picture

A tool is a function the model can invoke: a name, a description, a parameter schema, and an implementation. Mechanically, tool calling is solved — the model emits a structured call, constrained decoding guarantees it's well-formed (inference series Ch 9), your code executes it and returns the result. What is *not* solved, and what this chapter is about, is designing tools the model uses *well*. Tool design is API design where the client is a language model, and that client has unusual properties: it reads documentation at call time (the description *is* the docs, in context, every call), it can't step through your code, it pattern-matches on names, and it gets confused by ambiguity in ways a human developer would ask about. Most "the agent used the tool wrong" failures are tool-design failures, not model failures.

## 3.2 Designing for a model as the caller

The principles, distilled from the 2026 agent-engineering literature and the tool-use post-training story (post-training series Ch 9):

- **The description is the interface.** The model chooses and uses tools based on the name, description, and schema — nothing else. Write descriptions like excellent docstrings: what it does, when to use it (and when *not* to), what each parameter means, what it returns, one example if the usage is non-obvious. A vague description produces vague usage.
- **Match tools to intentions, not endpoints.** Wrapping your REST API one-endpoint-per-tool gives the model a puzzle to assemble; giving it `search_customer(query)` instead of four CRUD calls gives it an intention it can act on. Consolidate: fewer, higher-level tools that map to the *tasks* the agent actually performs beat many granular ones.
- **Make the schema do the enforcement.** Enums for closed choices, required vs optional marked honestly, tight types, naming consistent across tools (`customer_id` everywhere, never `cust_id` in one place). Every constraint the schema encodes is a mistake the model can't make (the constrained-decoding guarantee); every constraint left to the description is a mistake it eventually will.
- **Keep the catalog small and relevant.** Tool definitions live in context (Chapter 2), and a large catalog both costs tokens and degrades selection — models pick wrong tools from crowded menus. Expose the tools relevant to the task at hand; for large catalogs, select dynamically (retrieve relevant tools like documents — the *select* move).
- **Design errors as instructions.** The model reads error results and decides what to do next. `"error: invalid input"` teaches nothing; `"date must be YYYY-MM-DD; you sent 03/04/2026"` gets a corrected retry on the next turn. Every error message is a prompt.

## 3.3 Tool-result hygiene

The other direction — what tools return — is a context-engineering concern (Chapter 2) with tool-specific rules:

- **Return what the model needs, not what the API returned.** A raw 40KB JSON blob buries the three fields that matter and bloats the window (and later needs compacting). Filter and format results for the *decision the model faces next*.
- **Paginate and truncate deliberately** — return a summary plus a handle ("2,340 rows; showing first 20; call again with `offset`") rather than everything or an arbitrary cutoff.
- **Mark provenance.** Tool results are *untrusted data*, not instructions — the security boundary from cross-cutting Ch 6. Structure results so injected text in a fetched document can't masquerade as a system instruction, and never train or prompt the model to treat tool output as commands.

## 3.4 MCP: the tool interoperability layer

The **Model Context Protocol (MCP)** standardized how tools are packaged and served: an MCP server exposes tools (and resources/prompts) over a standard protocol, and any MCP-capable client — by 2026, effectively every major framework and assistant — can use them without integration code. Why it matters strategically:

- **Portability** — "tool integrations built on MCP port between frameworks without rewriting"; teams that standardized on MCP can swap orchestration frameworks (Chapter 8) without rebuilding the integration layer. Your tools outlive your framework choice.
- **Ecosystem** — thousands of public MCP servers (and now marketplaces) mean common integrations (GitHub, Slack, databases, browsers) are off-the-shelf.
- **The caution** — third-party MCP servers are third-party code with tool-level access: tool poisoning and credential theft via malicious servers is a named 2026 attack surface (cross-cutting Ch 6.2). Vet servers as you'd vet dependencies (cross-cutting's dependency discipline), pin versions, and grant least-privilege credentials.

Default: **build your integrations as MCP servers** unless you have a reason not to — the portability is free and the packaging discipline is healthy. Hand-rolled tool plumbing is legacy the day it's written.

## 3.5 Permissions and blast radius

Tools are where an agent's text becomes action, so tools are where safety becomes concrete (the containment story of cross-cutting Ch 6.3, applied at design time):

- **Least privilege per tool** — each tool gets the narrowest credentials that work; the search tool cannot write; the write tool touches one system, not all of them.
- **Tiered by consequence** — classify tools as read (free to call), write-reversible (call with logging), and write-irreversible/outward-facing (require confirmation — human-in-the-loop or a policy gate). The tier lives in *code*, not in the prompt: a prompt-level "always ask before deleting" is a suggestion; a code-level gate is a control.
- **Sandbox execution tools** — code execution, browsing, and file access run in isolated, revocable environments as a structural property.
- **Log every call** — tool calls are the audit trail of an agent's actions (Chapter 10's observability starts here).

## 3.6 Decisions

1. **Write descriptions as the interface** — docstring-quality, with when-to-use and when-not; the model reads nothing else.
2. **Consolidate to intention-level tools** with schema-enforced constraints and consistent naming; keep the in-context catalog small, selecting dynamically from large ones.
3. **Engineer results and errors** — filtered, formatted, paginated results; error messages written as corrective instructions; tool output treated as untrusted data.
4. **Package integrations as MCP servers** for portability and ecosystem — and vet third-party servers as dependencies with least-privilege credentials.
5. **Enforce permissions in code, tiered by consequence** — read freely, log writes, gate the irreversible behind confirmation, sandbox execution.

Next: agent architectures — the loop that decides which of these tools to call, and when to loop at all.
