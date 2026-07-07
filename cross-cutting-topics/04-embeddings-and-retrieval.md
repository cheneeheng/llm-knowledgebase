# Chapter 4 — Embeddings and Retrieval Models

## 4.1 Conceptual picture

An embedding model turns a piece of text (or an image) into a dense vector such that *semantically similar things land near each other* in vector space. This is the engine of retrieval — search, RAG, deduplication, clustering, recommendation — and it's a *different model* from your generative LLM, trained differently (contrastive, not next-token) for a different job (represent, not generate). Almost every LLM product needs one: RAG (retrieval-augmented generation) is how you give an LLM knowledge it wasn't trained on without retraining (inference series Ch 3 covered serving RAG; this covers the retrieval model itself), and RAG quality is bounded by retrieval quality, which is bounded by your embedding model. So embeddings are a parallel training track most LLM guides skip — and this chapter fills.

The mental model: your generative LLM *reasons*; your embedding model *finds*. A great generator over bad retrieval hallucinates confidently from irrelevant context; the embedding model is the unglamorous component that decides whether the right information ever reaches the generator.

## 4.2 How embedding models are trained

Embedding models are trained with **contrastive learning**: pull semantically-related pairs (a question and its answer, a query and a relevant document) *together* in vector space and push unrelated pairs *apart*. The mechanics:

- **Base:** modern embedding models are increasingly *LLM-derived* — take a strong pretrained LLM (Qwen3, Mistral) and adapt it into an embedder, inheriting its language understanding. The Qwen3-Embedding series (0.6B/4B/8B) exemplifies this — an LLM foundation turned into a top embedder.
- **Contrastive objective:** InfoNCE-style loss over batches — for each query, its positive (relevant) document should score higher than the *in-batch negatives* (other documents). **Hard negatives** (documents that are similar but *not* relevant) are the key training-data ingredient; mining good hard negatives is where embedding quality is won.
- **Multi-stage training** (the 2026 recipe): large-scale weakly-supervised contrastive *pretraining* (on massive noisy pairs — web title-body, Q-A, etc.), then supervised *fine-tuning* on curated high-quality pairs with hard negatives, then optionally *distillation from a reranker* (4.4). Qwen3-Embedding's pipeline is exactly this: contrastive pretraining → reranker distillation → high-quality vectors.
- **Data is (again) the lever:** the quality and diversity of the pairs, and the hardness of the negatives, determine the embedder. Synthetic pair generation (an LLM writes queries for documents) is a major 2026 source, filtered for quality (the data-curation through-line).

## 4.3 Matryoshka: one model, many dimensions

A practically important 2026 technique: **Matryoshka Representation Learning (MRL)** trains the embedding so that its *prefixes* are also valid embeddings — the first 256 dimensions of a 1024-dim vector are themselves a usable (lower-quality) embedding, like nested dolls. Why it matters:

- **Storage/compute flexibility without retraining** — truncate to 256 dims to cut storage 4× with only 2–3% precision loss, or keep full dims for max quality, *from one model*. You pick the dimension per your storage/latency budget at query time.
- **Coarse-to-fine retrieval** — retrieve with cheap short vectors, rerank with full vectors. Cuts the cost of large vector databases dramatically.

Prefer MRL-trained embedders (Qwen3-Embedding and most 2026 models support it); it's a free flexibility win.

## 4.4 Rerankers

Embeddings enable *fast* retrieval (approximate nearest-neighbor over millions of vectors) but at limited precision — a single vector can't capture full query-document relevance. **Rerankers** fix the top results: after the embedder retrieves ~100 candidates, a reranker (a cross-encoder that reads the query *and* each candidate *together*, not as separate vectors) rescores them for precise relevance, and you keep the top few. The two-stage pattern — **embed-and-retrieve (fast, recall) → rerank (slow, precision)** — is the standard 2026 RAG retrieval stack.

- Rerankers are more accurate (they attend across query and document jointly) but far slower (one forward pass per candidate), so they only run on the retrieved shortlist, not the whole corpus.
- They're trained similarly (contrastive/ranking loss) and often distilled into the embedder (reranker-distillation, 4.2) to close the gap.
- "Most production RAG stacks in 2026 default to a strong embedder plus a reranker" (BGE-M3 + BGE-reranker, or the Qwen3 embed+rerank pair). Use both; embedding-only retrieval leaves precision on the table.

## 4.5 The RAG stack and choosing components

The full retrieval side of a RAG system:

1. **Chunk** documents into retrievable units (size/overlap is a real tuning knob — too big dilutes relevance, too small loses context).
2. **Embed** chunks with your embedding model; store in a vector database (FAISS, Milvus, pgvector, etc.).
3. **Retrieve** top-K by vector similarity (ANN search) — fast, recall-oriented.
4. **Rerank** the top-K with a cross-encoder — slow, precision-oriented.
5. **Assemble** the top few into the LLM's context (mind the visual/context budget) and generate.

Choosing an embedder in 2026: benchmark on **MTEB** (Massive Text Embedding Benchmark) as a starting point — but, as with LLM leaderboards, **evaluate on your own retrieval task**, because MTEB rank doesn't guarantee your-domain performance. Strong open options (Qwen3-Embedding-8B tops MTEB multilingual and beats OpenAI/Google API embedders; BGE-M3 for a smaller footprint) mean self-hosting retrieval is usually the right call — embeddings are cheap to serve and keeping them in-house is a privacy win (Chapter 7). Consider: multilinguality, max sequence length (long-document retrieval), dimension (MRL flexibility), and domain fit.

## 4.6 Evaluation

Retrieval has its own metrics, distinct from generation:
- **Recall@K** — fraction of relevant documents retrieved in the top K. The dominant retrieval metric; if the right doc isn't in the top K, the generator never sees it.
- **nDCG / MRR** — rank-aware metrics rewarding relevant docs ranked higher (reranker quality).
- **End-to-end RAG eval** — ultimately, does retrieval improve the *generated answer*? Measure the full pipeline (retrieval + generation) on your task, because good retrieval metrics don't guarantee good answers (the chunking, context assembly, and generator all matter).
- **Contamination/freshness** — retrieval corpora drift; monitor that the index stays current and relevant.

## 4.7 Decisions

1. **Train/choose a dedicated embedding model** — it's a separate, contrastively-trained model from your LLM, and RAG quality is bounded by it.
2. **Prefer a strong LLM-derived, MRL-trained open embedder** (Qwen3-Embedding, BGE-M3 class) and self-host — they beat API embedders, are cheap to serve, and keep data private.
3. **Use the two-stage stack: embed-and-retrieve (recall) → rerank (precision)** — embedding-only retrieval under-performs; add a reranker.
4. **Exploit Matryoshka** — truncate dimensions to trade storage/speed for a small precision loss, and do coarse-to-fine retrieval, from one model.
5. **Evaluate retrieval on your own task** (Recall@K, nDCG) *and* end-to-end (does it improve the answer) — MTEB rank is a starting point, not a guarantee; tune chunking and context assembly.

Next: interpretability — understanding and steering what the model actually does inside.
