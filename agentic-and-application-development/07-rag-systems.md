# Chapter 7 — RAG Systems

## 7.1 Conceptual picture

Retrieval-augmented generation (RAG) grounds a model in data it wasn't trained on — your documents, current facts, private knowledge — by *retrieving* relevant content at query time and placing it in context (the *select* move, Chapter 2). It is "the default architecture for enterprise AI applications in 2026," because it's how you give a model knowledge without retraining, with citations, and with the ability to update the knowledge by updating the index rather than the weights. This chapter is the *system* — the retrieval pipeline, its quality determinants, and the ladder of sophistication from naive vector search to agentic and graph RAG. The retrieval *model* (the embedder and reranker) is the cross-cutting series' Chapter 4; the serving of RAG (prefix caching shared documents) is inference Chapter 3. Here we assemble them into a working system.

The one sentence that governs RAG quality: **the generator can only be as good as what retrieval puts in front of it.** A frontier model over bad retrieval confidently answers from irrelevant context (or hallucinates when nothing relevant was found); a mediocre model over excellent retrieval answers well and cites sources. RAG engineering is mostly retrieval engineering.

## 7.2 The baseline pipeline

The pipeline every RAG system starts from, in order:

1. **Chunk** — split documents into retrievable units. The 2026 default: recursive chunking at **300–500 tokens with 10–15% overlap**, aiming for chunks that are "semantically complete — each chunk should answer a question on its own." Too large dilutes relevance (the right sentence buried in a page); too small loses context (a fact without its referent). Chunking is a real, under-appreciated quality lever — bad chunks cap the whole system.
2. **Embed and index** — embed chunks with your embedding model (cross-cutting Ch 4), store in a vector database (FAISS, Milvus, pgvector, Qdrant, ...).
3. **Retrieve** — embed the query, find top-K nearest chunks by vector similarity (fast, recall-oriented).
4. **Rerank** — rescore the top-K with a cross-encoder reranker for precision (cross-cutting Ch 4.4).
5. **Assemble and generate** — place the top few reranked chunks in context (mind the budget and position, Chapter 2) and generate, ideally with instructions to cite and to say "I don't know" when the context lacks the answer.

## 7.3 The production baseline: hybrid + rerank

The single most repeated 2026 RAG recommendation, and where every system should *start* after the naive baseline: **hybrid search + reranking.**

- **Hybrid search** combines **dense** retrieval (embeddings — semantic similarity, "find conceptually related content") with **sparse/lexical** retrieval (BM25 — exact keyword match, "find the document with this precise term/code/name"). Each covers the other's weakness: dense misses exact identifiers and rare terms; sparse misses paraphrase and synonymy. Combined (via score fusion like RRF), they retrieve better than either alone across almost all workloads.
- **Reranking** then takes the fused candidates and precisely orders them with a cross-encoder.

"For most production use cases, Hybrid + Rerank offers the best quality-to-cost ratio." Treat this as the default target architecture — get it working, measure it (7.6), and only climb higher if the metrics prove it insufficient. Most teams that think they need agentic or graph RAG actually need a working hybrid+rerank pipeline with good chunking first.

## 7.4 The escalation ladder

When hybrid+rerank is measured insufficient — and only then — the higher rungs:

- **Query transformation** — rewrite/expand/decompose the query before retrieval (a vague question becomes a good search; a multi-part question becomes several searches). Cheap, often the next step up from baseline.
- **Agentic RAG.** The model "decomposes a complex query into sub-queries, decides which retrieval tools to call, runs them (sometimes in parallel), evaluates what came back, and iterates until it has enough context." Retrieval becomes an agentic loop (Chapter 4) with search as a tool. It wins on **multi-hop questions** (answers requiring chained retrieval — "who is the CEO of the company that acquired X") and when accuracy is non-negotiable, at the cost of more model calls and latency. It's the agent patterns of this series applied to retrieval.
- **Graph RAG.** Retrieve over a **knowledge graph of entities and relationships** instead of flat chunks. It excels where answers require *connecting* information across documents (relationships, aggregations, "how are these related") that chunk retrieval can't assemble, and many strong 2026 systems pair it with an agent that queries the graph. The cost is building and maintaining the graph (entity/relation extraction, an upkeep burden), so it's justified for genuinely relationship-heavy, high-value domains, not general Q&A.

Each rung adds cost, latency, and failure surface (the through-line). Climb one at a time, measuring at each: query transformation before agentic, agentic before graph, and never adopt a rung on architecture-blog enthusiasm rather than your own retrieval metrics.

## 7.5 RAG and agents together

RAG and agents converge in 2026. An agent *is* a system that retrieves into its own context; agentic RAG *is* an agent whose tool is search. In a full application, retrieval is typically a **tool the agent calls** (Chapter 3) rather than a fixed pre-step — the agent decides when it needs to look something up, searches, evaluates, and searches again. This is more flexible than pipeline RAG (the agent retrieves adaptively) and inherits the agent tradeoffs (cost, reliability). The design choice mirrors Chapter 4: pipeline RAG (you control retrieval flow) for predictable Q&A; agentic RAG (the model controls it) for open-ended, multi-hop research. Most production systems are pipeline RAG with an agentic escalation for hard queries — the hybrid router idea again.

## 7.6 Evaluating RAG (before adding complexity)

The discipline that makes the ladder navigable, and the step most teams skip: **instrument retrieval evaluation before adding any sophistication.** You cannot know whether a rung helped without measuring, and RAG has two evaluation layers:

- **Retrieval quality** — Recall@K (is the answer-bearing chunk in the top K?), precision, nDCG (cross-cutting Ch 4.6). If retrieval doesn't surface the right content, no generator can fix it — diagnose here first.
- **End-to-end / answer quality** — does the generated answer, given the retrieved context, actually answer correctly and faithfully? **RAGAS** and similar frameworks score faithfulness (is the answer grounded in the retrieved context, or hallucinated?), answer relevance, and context relevance. This catches the failure where retrieval was fine but the generator ignored it or embellished.

"Measure your retrieval quality with RAGAS, and only add complexity when your metrics prove the simpler approach isn't enough." Build the eval harness first; it's what turns the escalation ladder from guesswork into engineering.

## 7.7 Decisions

1. **Get chunking right first** — recursive, ~300–500 tokens, 10–15% overlap, semantically complete chunks; it caps everything downstream.
2. **Default to hybrid (dense + BM25) + reranking** — the best quality-to-cost ratio for most workloads and the target baseline after naive vector search.
3. **Instrument retrieval and answer evaluation (Recall@K, RAGAS faithfulness) before adding complexity** — you can't navigate the ladder without metrics.
4. **Climb the ladder one rung at a time, by measurement** — query transformation → agentic RAG (multi-hop, accuracy-critical) → graph RAG (relationship-heavy) — never on blog enthusiasm.
5. **Prefer retrieval-as-a-tool for agents and pipeline RAG for predictable Q&A**, escalating to agentic RAG only for the hard queries — the hybrid-router pattern.

Next: orchestration frameworks — the software that implements the loops, memory, and coordination of the last five chapters.
