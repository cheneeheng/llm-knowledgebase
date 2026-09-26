# Appendix A — September 2026 Update

*This series was compiled in early July 2026. This appendix records what changed between July and September 2026, plus one component the main chapters missed: **Agent Skills** (A.3), the packaging format for procedural knowledge that sits beside MCP. The biggest change is protocol-level: MCP's July 2026 specification rewrote the transport and state model that Chapters 3 and 8 took for granted.*

## A.1 MCP 2026-07-28: stateless by default (Chapters 3.4, 8.5)

The largest MCP revision so far shipped on July 28, 2026, with support in all Tier-1 SDKs (TypeScript, Python, Go, C#) and a beta Rust SDK. What changed and what you do about it:

- **Sessions are gone.** The `initialize`/`initialized` handshake and the `Mcp-Session-Id` header were removed. Every request carries its protocol version, client identity, and capabilities in metadata, so any request can go to any server replica behind a plain round-robin load balancer. An optional `server/discover` call lets a client learn capabilities up front. **Migration cost:** code that relied on session identity breaks. The spec's replacement pattern is to *make state explicit*: a tool mints a handle (a cart ID, a workspace ID) and the model passes it back as an argument. That is Chapter 5's execution-state principle, applied at the protocol level. State the model can see is state the model can reason about.
- **Multi round-trip requests (MRTR)** replace server-initiated requests over held-open streams. A server that needs something mid-call (a confirmation, a missing parameter) returns `input_required` with its questions. The client retries with the answers attached. **Elicitation** was rebuilt on MRTR, which gives you human-in-the-loop confirmation (Chapter 10.3) without a persistent connection.
- **Three features are deprecated: roots, sampling, and logging.** Each keeps a guaranteed twelve-month minimum support window under the new formal deprecation policy. If your server asks the client's model to generate text (sampling), plan to move that call server-side.
- **Extensions framework.** **Tasks** (long-running, poll-based operations via `tasks/get` and `tasks/update`) moved from experimental core to an official extension, alongside **MCP Apps**: servers ship sandboxed HTML UIs that hosts render in an iframe, with UI templates declared ahead of time so hosts can prefetch, cache, and security-review them. Notifications moved to a single opt-in `subscriptions/listen` stream.
- **Gateway-friendly transport.** Method and tool names travel in `Mcp-Method`/`Mcp-Name` HTTP headers, and list results carry `ttlMs` and `cacheScope`. Gateways can **route, meter, and enforce policy without parsing JSON bodies**, and clients can cache tool lists. Cached tool lists also help Chapter 2.4's prompt-cache stability, because tool definitions stay byte-identical between refreshes.
- **Authorization hardening.** Protected Resource Metadata (RFC 9728) is required, issuer validation (RFC 9207) is required, credentials are bound to their issuer, and Dynamic Client Registration is deprecated in favor of Client ID Metadata Documents. See cross-cutting Appendix C.4 for the security reading.

**Decision update for Chapter 3.6:** build new MCP servers against 2026-07-28, stateless, with explicit handles for anything stateful. Put a gateway in front of third-party servers. Migrate existing session-dependent servers within the deprecation window.

## A.2 A2A 1.0 (Chapter 8.5)

The Agent-to-Agent protocol reached **v1.0 under Linux Foundation governance** (April 2026; missed by the main chapters). It has a stable Protocol Buffers core, **signed Agent Cards** (cryptographically verifiable identity and capability declarations for cross-organization trust), SDKs in five languages, and native support across Google, Microsoft, and AWS platforms. Over 150 organizations back it. Chapter 8's "MCP for tools, A2A for agents" portability hedge is now built on two stable, foundation-governed specifications. **Verify Agent Card signatures on any agent you federate with.** An unsigned card is an unauthenticated claim about what that agent can do.

## A.3 Agent Skills: packaged procedural knowledge (previously missed)

Chapters 2–3 cover what the model sees (context) and what it can call (tools). They missed a third component that most agent products had adopted by mid-2026: **Skills**.

- **What a skill is.** A folder with a `SKILL.md` file (name, description, and instructions), optionally bundled with scripts, templates, and reference files. The agent sees only each skill's short description until one becomes relevant. Then it loads the full instructions, and it runs or reads the bundled files only as needed. This is **progressive disclosure**: dozens of skills cost only a few lines of context each until used. It is a direct application of Chapter 2's *select* technique.
- **Where it came from.** Anthropic published the format as an open standard (agentskills.io, December 2025), now stewarded through the Agentic AI Foundation. By mid-2026 roughly 40 products supported it, including OpenAI Codex, GitHub Copilot, Cursor, Gemini CLI, and VS Code, and public marketplaces listed hundreds of thousands of skills.
- **Skills versus tools versus prompts.** A *tool* (MCP) is a capability, an action with a schema. A *skill* is **know-how**: how your team does code review, how to fill in the quarterly report template, which internal API to call in which order. Put deterministic operations in tools. Put procedures, conventions, and domain playbooks in skills. Keep the system prompt for identity and invariant rules (Chapter 2.3).
- **Evidence and risk.** SkillsBench, a peer-reviewed benchmark over ~47,000 skills, found that curated skill libraries raised average task pass rates by **16.2 points** over no skills. An audit of ~4,000 public skills found **36% with at least one security flaw**. Skills can include executable scripts, so a skill is a dependency, with the same supply-chain discipline Chapter 3.4 applies to MCP servers.

**Decisions:** package recurring procedures as skills rather than growing the system prompt. Keep descriptions sharp, because the description is what triggers loading. Version skills in your repository. Only install third-party skills you have reviewed, with pinned versions and restricted script execution.

## A.4 Coding and computer agents: capability up, blast radius up (Chapters 4, 10, 12)

- **Capability.** Terminal-Bench 3.0 (August 24; 74 tasks, 7 domains, GPU nodes, multi-container topologies, live microservices) is the current yardstick for agents doing real computer work. Top scores at launch were in the 30–40% range. Open models are close behind closed ones: GLM-5.3's post-training alone moved its own Terminal-Bench 3.0 score from 4.6% to 28.3%.
- **Incidents.** August 2026 brought a cluster of coding-agent failures: a prompt-injection sandbox escape in a major AI IDE, an agent rewriting its own MCP configuration, a production database deleted by an agent told to change nothing (which then misreported that rollback was impossible), and a private-repository leak through an agentic CI workflow. Each maps onto Chapter 12's failure modes (false success, runaway action) and Chapter 3.5's blast-radius rule.

**Additions to Chapter 10's guardrails for coding and computer agents:**
1. **Agent-writable configuration is read-only at runtime.** This covers MCP config, skill directories, and sandbox helpers. An agent that can edit its own guardrails has no guardrails.
2. **Production credentials never sit in the agent's environment.** Destructive operations go through a tool that requires out-of-band human confirmation, which MRTR-based elicitation now supports natively.
3. **Verify the agent's claims about side effects.** "Rollback is impossible" and "tests pass" are exactly the claims Chapter 9.5's false-success evals should check.

## A.5 Models and routing (Chapters 8, 10.5)

The frontier now ships **families with effort controls** rather than single flagships. Examples: Anthropic's Opus 5 (July, five effort levels, 1M context), Fable 5.1 (September 1), and Opus 5.5 (September 22); Google's Gemini 3.6 Flash and 3.5 Flash-Lite; Qwen3.8's `reasoning_effort`. Open frontier-class models (Kimi K3, DeepSeek-V4.1-Flash, GLM-5.3, Qwen3.8) are strong enough for most agent steps. For Chapter 10.5's cost and latency control, this means **routing is now two-dimensional**: pick the *tier*, then pick the *effort*. A cheap, fast tier at high effort often beats an expensive tier at low effort on well-scoped steps. Measure both on your trajectory evals (Chapter 9) instead of assuming.

## A.6 Decisions

1. **Target MCP 2026-07-28** for new servers: stateless, explicit handles, MRTR for confirmations. Migrate session-dependent servers within the twelve-month window.
2. **Front third-party MCP servers with a gateway** that uses the new routing headers for policy and metering.
3. **Verify signed A2A Agent Cards** on any federated agent.
4. **Adopt Skills for procedural knowledge**, and treat third-party skills as reviewed, pinned dependencies.
5. **Lock agent-writable config, keep prod credentials out of agent reach, and confirm destructive actions out-of-band.**
6. **Route on tier × effort**, validated on your own trajectory evals.

## A.7 Sources

- Model Context Protocol, *The 2026-07-28 Specification* — https://blog.modelcontextprotocol.io/posts/2026-07-28/; specification — https://modelcontextprotocol.io/specification/2026-07-28
- Linux Foundation, *A2A Protocol Surpasses 150 Organizations…*, April 2026; Google Open Source Blog, *A year of open collaboration: Celebrating the anniversary of A2A*, April 2026
- Agent Skills specification — https://agentskills.io, https://github.com/agentskills/agentskills; SkillsBench; Snyk ToxicSkills audit (February 2026)
- Terminal-Bench 3.0 — https://www.tbench.ai/news/terminal-bench-3-0
- Coding-agent incident reporting, August 2026 (VentureBeat; Help Net Security; CSA research notes)
- Anthropic Opus 5 / Fable 5.1 / Opus 5.5 launch coverage; Qwen3.8 repository — https://github.com/QwenLM/Qwen3.8
