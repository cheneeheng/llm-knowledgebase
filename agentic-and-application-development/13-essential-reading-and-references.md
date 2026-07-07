# Chapter 13 — Essential Reading and References

The application layer moves faster than any other part of the stack, and its best sources are engineering guides, framework documentation, and benchmark papers rather than a settled academic canon — so this list weights primary engineering writing and the few papers that named the load-bearing concepts. Read the essential cut in order; the topic list and provenance notes follow.

## The essential cut

If you read only these, you'll have every load-bearing idea in this series from a primary source. Ordered as a sequence — early entries frame what later ones assume.

| # | Source | One-line reason | Chapters |
|---|---|---|---|
| 1 | Anthropic, *Building Effective Agents* | The workflow-vs-agent taxonomy and the least-autonomy principle; the most-cited engineering guide of the era | 1, 4 |
| 2 | Anthropic, *Effective Context Engineering for AI Agents* + the managed-agents compaction guidance | The write/select/compress/isolate framing and the compaction playbook | 2, 5 |
| 3 | Yao et al., *ReAct* (arXiv:2210.03629) | The reason-act-observe loop underneath every agent | 4 |
| 4 | Anthropic, *Writing Tools for Agents* | Tool design with a model as the caller — descriptions, consolidation, result hygiene | 3 |
| 5 | The *Model Context Protocol* specification (modelcontextprotocol.io) | The tool-interoperability standard and the portability hedge | 3, 8 |
| 6 | Anthropic, *How We Built Our Multi-Agent Research System* | The orchestrator-worker pattern with context isolation, from the canonical production example | 6 |
| 7 | Yao et al., *τ-bench* (arXiv:2406.12045) + *τ²-bench* | pass^k and the agent-reliability evaluation frame | 9 |
| 8 | Zhuge et al., *Agent-as-a-Judge* (arXiv:2410.10934) | Trajectory-level judging beyond output-only LLM judges | 9 |
| 9 | Lewis et al., *RAG* (arXiv:2005.11401) + the 2026 hybrid+rerank production guides | The retrieval-augmentation foundation and the modern baseline architecture | 7 |
| 10 | Es et al., *RAGAS* (arXiv:2309.15217) | Faithfulness and the RAG evaluation layer that gates the escalation ladder | 7, 9 |
| 11 | The Princeton *HAL* (Holistic Agent Leaderboard) analyses | The evidence that scaffold choice swings identical-model performance by tens of points | 1, 8 |
| 12 | The 2026 framework comparisons (LangGraph/Agent SDK/CrewAI/Microsoft Agent Framework) + LangGraph docs | The three maturity thresholds and the current framework map | 8 |

## Why each, and what to extract

**1. Building Effective Agents.** Read first. It defines the workflow patterns (chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer), draws the workflow-vs-agent line, and argues the least-autonomy principle this series adopts as law. Extract: the taxonomy and the discipline of not building an agent when a workflow suffices.

**2. Context engineering + compaction guidance.** The Chapter 2 substrate from its source. Extract: the four techniques, the context-as-workspace framing, goals-at-the-top, clear-stale-tool-results-first, and compaction-at-boundaries.

**3. ReAct.** The paper that put reason-act-observe into one loop and made tool-using agents work. Skim the benchmarks; extract the loop and why interleaving reasoning with action beats plan-everything-then-act.

**4. Writing Tools for Agents.** Tool design as API-design-for-a-model. Extract: descriptions as the interface, intention-level consolidation, and error messages as corrective instructions.

**5. The MCP specification.** Extract: the server/tool/resource model, and strategically, why building on the protocol decouples your integration investment from your framework choice.

**6. The multi-agent research system post.** The honest production account of orchestrator-worker: sub-agents with isolated contexts researching in parallel, concise findings handed back, and the coordination costs stated plainly. Extract: when isolation justifies multi-agent, and the hand-off discipline.

**7. τ-bench / τ²-bench.** Extract: pass^k — why an agent that passes once may fail on repeats, and how to set product bars in all-k terms. The evaluation upgrade this series considers non-negotiable.

**8. Agent-as-a-Judge.** Extract: why output-only judging misses trajectory failures, and how an agentic judge with tools verifies claims (the false-success defense) rather than reading transcripts credulously.

**9. RAG + the hybrid+rerank guides.** The founding retrieval-augmentation paper, then the 2026 production consensus layered on it. Extract: hybrid (dense+BM25) + reranker as the quality-to-cost baseline, and the chunking defaults (300–500 tokens, 10–15% overlap).

**10. RAGAS.** Extract: faithfulness/answer-relevance/context-relevance as the metrics that make the RAG escalation ladder navigable by measurement instead of enthusiasm.

**11. The HAL analyses.** Extract: the 30-point scaffold spread on identical models — the empirical core of this series' claim that the application layer is a capability lever, not plumbing.

**12. The framework comparisons + LangGraph docs.** Extract: the three production thresholds (durable state + HITL, native MCP, native A2A), the current best-fit map, and the year-long commitment the orchestration layer implies.

## Reading list by topic

- **Agent architectures:** Building Effective Agents; ReAct (2210.03629); Reflexion (2303.11366, self-correction); plan-and-execute patterns; the awesome-harness-engineering collection (github.com/ai-boost/awesome-harness-engineering) as a living index.
- **Context engineering & memory:** Anthropic's context-engineering and managed-agents posts; the 2026 context-engineering field guides; mem0 and framework memory docs; the prompt-caching long-horizon cost study (41–80% savings); "Context Engineering: From Prompts to Corporate Multi-Agent Architecture" (arXiv:2603.09619).
- **Tools & protocols:** Writing Tools for Agents; the MCP spec and ecosystem; the A2A protocol; tool-poisoning/MCP-security analyses (cross-reference cross-cutting Ch 6).
- **Multi-agent:** the Anthropic research-system post; orchestrator-worker analyses; "why multi-agent fails" post-mortems and the coordination-tax literature.
- **RAG systems:** RAG (2005.11401); RAGAS (2309.15217); the 2026 agentic-RAG and GraphRAG guides; Microsoft GraphRAG (2404.16130); hybrid-search/RRF references; chunking-strategy studies.
- **Evaluation:** τ-bench (2406.12045), τ²-bench, SWE-bench (2310.06770), GAIA (2311.12983), AgentBench (2308.03688); Agent-as-a-Judge (2410.10934); "Characterizing False Success in LLM Agents" (arXiv:2606.09863); the LangSmith/Braintrust/Phoenix/DeepEval documentation for harness machinery.
- **Production reliability:** OpenTelemetry LLM/agent tracing conventions; the observability platforms' agent-tracing guides; durable-execution documentation (LangGraph checkpointing, Agent SDK); the agent-evaluation-and-observability 2026 surveys.
- **Frameworks:** LangGraph, Claude Agent SDK, CrewAI, Microsoft Agent Framework, LlamaIndex Workflows, Pydantic AI docs; the 2026 comparison round-ups; HAL (Princeton) for scaffold-sensitivity evidence.

## Provenance notes

**Foundational papers (2020–2024):** RAG, ReAct, Reflexion, τ-bench, SWE-bench, GAIA, Agent-as-a-Judge, RAGAS are peer-reviewed or heavily-cited arXiv work; their concepts (the loop, pass^k, faithfulness, trajectory judging) are settled vocabulary and quoted directly.

**First-party engineering guides (Anthropic and framework vendors):** Building Effective Agents, the context-engineering and tool-writing posts, and the multi-agent research-system account are primary sources for practice but are vendor-authored — their patterns generalize, but verify vendor-specific claims (SDK features, caching behavior) against the stack you actually run.

**2026-dated sources:** The framework comparisons, context-engineering field guides, agentic-RAG guides, the false-success paper (2606.09863), the prompt-caching study, and the HAL scaffold-spread numbers postdate the January 2026 knowledge cutoff and were surfaced by mid-2026 web search. They reflect current practice, but this layer churns fastest of all: framework rankings, MCP ecosystem maturity, and benchmark saturation move quarterly. Re-verify before committing a year to a framework.

**A note on numbers:** Every concrete figure in this series (the 30-point scaffold spread, 0.95²⁰ ≈ 36%, pass^8 ≈ 43% at 90% single-run, 41–80% caching savings, 300–500-token chunks, 2–3-level hierarchies, 4–8 eval runs) is drawn from the sources above and reflects mid-2026 practice. They are defaults and framings to reason from — your task, traffic, and stack will move the optima, and the series' own through-line applies to its numbers: measure on your system before trusting them.
