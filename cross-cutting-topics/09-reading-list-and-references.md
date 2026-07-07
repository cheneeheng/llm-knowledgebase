# Chapter 9 — Reading List and References

This series is broad, so rather than a single essential-twelve, the reading is organized by chapter/topic. Each topic lists the sources that anchor the decisions in that chapter, with provenance notes at the end. Where a topic is covered more deeply in another series, that pointer is given.

## By topic

### Model selection and licensing (Chapter 2)
- The **OSI Open Source AI Definition** and the open-source-vs-open-weight distinction (opensource.org; the 2026 landscape write-ups on huggingface.co and digitalapplied.com).
- **License texts themselves** — Apache 2.0, MIT, the **Llama Community License** (read the MAU threshold and acceptable-use clauses directly), Open-RAIL. Don't rely on summaries.
- **OLMo** technical reports (AllenAI) — the reference fully-open (reproducible) model family.
- The 2026 open-model comparison guides (acecloud, computingforgeeks, onyx) for the current license/capability landscape — treat as starting points, verify licenses at source.

### Small and on-device models (Chapter 3)
- *Small Language Models: Survey, Measurements, and Insights* (arXiv:2409.15790) — the systematic SLM survey.
- *MiniCPM* (arXiv:2404.06395) — scalable small-model training (also the WSD reference in the pretraining series).
- *On-Device LLMs: State of the Union 2026* (v-chandra.github.io/on-device-llms) — the edge-deployment landscape.
- Distillation mechanics: **post-training series Chapter 10** (the deep treatment); Distill Labs' SLM distillation benchmarks (Qwen3-4B as a strong base).
- Edge quantization: **inference series Chapter 6** (FP8/INT4/AWQ/GPTQ/GGUF); Shakti (arXiv:2410.11331) for an edge-optimized SLM.

### Embeddings and retrieval (Chapter 4)
- *Qwen3 Embedding* (arXiv:2506.05176) — the LLM-derived, multi-stage, MRL, embed+rerank reference recipe.
- *Matryoshka Representation Learning* (arXiv:2205.13147) — the nested-dimension technique.
- **MTEB** (arXiv:2210.07316) — the embedding benchmark (use as a starting point, evaluate on your task).
- **BGE / BGE-M3 and BGE-reranker** — the common production RAG default stack.
- Contrastive-learning and hard-negative-mining references; **inference series Chapter 3** for serving RAG (prefix caching).

### Interpretability (Chapter 5)
- **Sparse autoencoder** foundational work (Anthropic's "Towards Monosemanticity" and "Scaling Monosemanticity"; the SAE literature) — the standard mechanistic-interpretability tool.
- *Mechanistic Interpretability of Code Correctness via SAEs* (arXiv:2510.02917, ICLR 2026) — a production-flavored SAE application.
- **Steering / concept vectors**: denoising concept vectors with SAEs (arXiv:2505.15038); persona vectors work; "steering without breaking" (arXiv:2605.10971).
- Transcoders for VLM hallucination (arXiv:2605.22902) — interpretability meets multimodal.
- Linear probing literature — the practical monitoring entry point.

### Production security (Chapter 6)
- **OWASP Top 10 for LLM Applications** (genai.owasp.org) and the **OWASP Top 10 for Agentic Applications 2026** — the canonical threat taxonomies. LLM01 (prompt injection) is required reading.
- *Red-Teaming LLMs 2026: A Practitioner's Guide* (datavlab.ai); the LLM-security-2026 attack-map write-ups (reddogsecurity, zylos, tokenmix).
- Automated red-teaming: PI-Hunter (arXiv:2606.12737), IPI-proxy (arXiv:2605.11868), AgentRedBench (arXiv:2606.02240); DeepTeam framework.
- Guardrail tools: NeMo Guardrails, LLM Guard, Guardrails AI, Lakera Guard (docs).
- **Post-training series Chapter 11** for model-level safety training (the defense-layer that lives in training).

### Privacy, compliance, governance (Chapter 7)
- **The EU AI Act** text and the GPAI Code of Practice; the 2026 compliance guides (secureprivacy.ai, surecloud, groath) for deadlines and obligations.
- **GDPR** and the AI-Act/GDPR intersection analyses (recordinglaw, regolo.ai).
- AI Act **Article 10** (data governance/lineage requirements) and the GPAI training-data-transparency + copyright-policy obligations (live since Aug 2025).
- Data-lineage and model-documentation practice; **model cards** (arXiv:1810.03993) and **datasheets for datasets** (arXiv:1803.09010) — the documentation standards.

### Team, compute, improvement loop (Chapter 8)
- The **data flywheel** and eval-driven-development practice — synthesized across this knowledge base rather than one source; the post-training series' iteration-discipline through-line is the deepest treatment.
- **Inference series Chapter 12** for compute budgeting / FinOps; the **pretraining series** for the scaling-ladder / experiment-budget discipline.
- Epoch AI's compute-and-economics analyses for the training/inference compute split.

## Provenance notes

**Standards and regulations (authoritative, but moving):** OWASP Top 10, the EU AI Act, GDPR, and the license texts are the actual governing documents — cite and read them directly, not summaries, and check dates because the AI Act phases in through 2027 and guidance evolves. The 2026 compliance-guide blogs are useful orientation but are secondary; verify obligations and deadlines against primary regulatory text (and, for anything you'll act on, actual legal counsel — this series is a technical map, not legal advice).

**Research papers (2022–2026):** The SAE, Matryoshka, MTEB, model-card, and datasheet references are peer-reviewed or heavily-cited and settled. The 2026-dated arXiv preprints (interpretability, red-teaming, embeddings) postdate the January 2026 knowledge cutoff and were surfaced by mid-2026 web search — included because they represent current practice, but as recent preprints weight them below established work.

**2026 landscape write-ups (blogs, benchmarks):** The open-model comparisons, SLM guides, security attack-maps, and embedding-benchmark posts reflect the state at compilation and move fast (license terms, model rankings, attack techniques, and regulatory deadlines all change quarterly). Use them for orientation and re-verify specifics — model licenses at the source, security techniques against current OWASP, embedding choices on your own retrieval task.

**Cross-references:** Much of this series' depth lives in the companion series — safety training (post-training Ch 11), distillation (post-training Ch 10), serving economics and quantization (inference Ch 6, 12), RAG serving (inference Ch 3), and the scaling-ladder experiment discipline (pretraining). This series takes the cross-stage view; follow the pointers for the mechanics.

**A note on the whole knowledge base:** With this series, the five-part set spans the full LLM lifecycle — pretraining → midtraining → post-training → inference → and the cross-cutting concerns (multimodal, plus this series) that wrap them. The deepest recurring lesson, stated once more: **everything downstream of data is a proxy for what you actually want, every proxy can be gamed, and the discipline that separates teams that ship from teams that thrash is keeping the proxies honest** — decontaminated data, audited rewards, red-teamed guardrails, and held-out evals the training loop has never seen.
