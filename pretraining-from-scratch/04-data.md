# Chapter 4 — Data: The Real Product

## 4.1 Conceptual picture

If you internalize one thing from this report: **at fixed compute, data quality and composition are the highest-leverage variables you control** — above architecture, above optimizer. Architecture differences between sane choices are worth a few percent; data pipeline differences are worth entire model generations. The best open pipelines (FineWeb-Edu, DCLM, Nemotron-CC) each demonstrated multi-point MMLU swings from filtering decisions alone — Nemotron-CC's high-quality subset improved MMLU by +5.6 over DCLM at identical model size and token count. Two models with identical architectures trained on different data are barely comparable. Frontier labs put more engineers on data than on modeling; you should allocate your attention the same way.

A pretraining dataset in 2026 is a *portfolio*: filtered web text as the base; code and math as the reasoning backbone; curated sources (books, papers, encyclopedias, forums) for density; multilingual data for coverage; and a growing synthetic fraction. The craft is in the filtering and the mixture.

## 4.2 Where tokens come from

**Web crawl** is the foundation, and in practice that means **Common Crawl (CC)** — monthly snapshots of the web, petabytes of raw HTML, ~100 snapshots deep. Nobody trains on raw CC; you train on the output of a refinement pipeline (4.3), and the open ecosystem has done spectacular work you can build on:

| Dataset | Size | Character |
|---|---|---|
| **FineWeb / FineWeb-2** | 15T+ tokens (En); multilingual in v2 | The reference open CC refinement; excellent docs and a reproducible pipeline (datatrove) |
| **FineWeb-Edu** | ~1.3T | Aggressively filtered for educational value via classifier (~90% of raw discarded); very high quality per token, but small |
| **DCLM-Baseline** | ~3.8T | DataComp-LM's classifier-filtered CC; strong benchmark performer; ~1T unique after global dedup |
| **Nemotron-CC (+HQ)** | 6.3T (4.4T unique real + 1.9T synthetic; 1.1T HQ subset) | Classifier *ensembling* + synthetic rephrasing; ~4× more unique tokens than FineWeb-Edu/DCLM; built for long-horizon (15T+) training. An 8B trained on 15T tokens (7.2T from Nemotron-CC) beat Llama-3.1-8B by ~5 MMLU. The reference for any 10T+ run; the HQ subset is the current quality leader per token |
| **Dolma, RedPajama-v2, TxT360, C4, The Pile** | various | Earlier/broader corpora; still useful as components and for their documented methods |
| **Code:** The Stack v2 / StarCoder2 data | ~900B+ | License-filtered GitHub; the standard open code layer |
| **Math:** Nemotron-CC-Math (133B/52B), FineMath, OpenWebMath, MegaMath, proof-pile | 10s–100s B | Math-dense web + papers. Nemotron-CC-Math's layout-aware extraction (preserving LaTeX) is current SOTA: +4.8–12.6 MATH over prior corpora |
| **Papers/books:** peS2o, arXiv, public-domain books | 100s B | Density and genuinely long documents |
| **Multilingual:** FineWeb-2 and language-specific CC derivatives | varies | Non-English quality is often the differentiator for non-US labs |

Independent replications (e.g., OpenEuroLLM reference models) confirm the ordering Nemotron-CC-HQ > DCLM > FineWeb-Edu as pretraining data at matched budgets.

A realistic 2026 open recipe: Nemotron-CC or FineWeb-family as the web base (~60–75% of the mix), 10–20% code, 5–10% math/science, 5–15% multilingual, plus curated and synthetic components. Llama-3's disclosed mix was roughly 50% general knowledge, 25% math/reasoning, 17% code, 8% multilingual — note how large the reasoning-relevant share already was in 2024; it has only grown since.

## 4.3 The refinement pipeline, stage by stage

If you build or extend your own pipeline — using **datatrove** (Hugging Face), **NeMo Curator** (NVIDIA, GPU-accelerated; worth it at trillion-token scale), or the **Dolma toolkit** (AI2) — the canonical stages are:

**1. Text extraction.** HTML → text with trafilatura or resiliparse. Extraction quality matters far more than it sounds: naive extraction destroys tables, equations, and code blocks. Nemotron-CC-Math exists precisely because standard extractors mangle MathJax/KaTeX; its lynx-based layout-aware extraction preserved equations and measurably improved downstream math and code (malformed equations teach malformed reasoning).

**2. Language identification.** fastText-based LID; route documents to language-specific pipelines.

**3. Heuristic quality filtering.** Gopher/C4-style rules: document-length bounds, symbol-to-word ratios, bullet/ellipsis line fractions, repetition ratios, word blocklists, boilerplate detection. Cheap; removes the worst garbage. Trend note: Nemotron-CC showed *over*-reliance on heuristics throws away good tokens — they relaxed heuristics in favor of learned classifiers to preserve unique-token supply.

**4. Deduplication — the single most important stage.** Two levels: **exact** (hash-based, often at line/paragraph granularity, killing boilerplate) and **fuzzy** (MinHash-LSH document-level near-dedup), often plus exact substring dedup. Global fuzzy dedup across all of CC is expensive but valuable: FineWeb-Edu and DCLM only dedup within shards and consequently remain ~80% fuzzy duplicates globally — which matters a lot once you train multiple epochs. Duplicated data wastes compute and increases memorization. Note the direction of craft: dedup the corpus fully, then *deliberately* reintroduce repetition of very high-quality sources via mixture weights — chosen repetition, not accidental.

**5. Model-based quality filtering.** Train a small classifier (fastText or a small LM head) on seed sets of "good" text (educational, textbook-like, instruction-dense) vs. random CC; score every document; keep the top fraction. This is the core DCLM/FineWeb-Edu trick. The 2026 refinement is **classifier ensembling** — multiple quality notions, documents bucketed by score — rather than one binary gate, preserving diversity while still ranking. Perplexity filtering (score with a small LM, drop the tails) and embedding-cluster filters are also standard. One 2026 caveat: classifiers trained on natural web text **mis-score synthetic and format-transformed data** (an educational-quality scorer can penalize exactly the table/FAQ/math reformulations that help) — don't blindly apply web classifiers to synthetic data.

**6. PII and safety scrubbing.** Regex + NER-based removal/masking of emails, phone numbers, government IDs; CSAM/NSFW/toxicity filtering at URL and content level. An ethical necessity and a legal one.

**7. Decontamination.** Remove your evaluation sets from training data: n-gram overlap (e.g., 13-gram) against every benchmark you will report, plus fuzzy matching. Contaminated evals inflate your numbers and — worse — mislead your own decisions during the run. Re-run *every time the data changes* and keep the decontamination report.

**8. Synthetic augmentation.** Use a strong existing LLM to rephrase high-quality documents (multiple styles: Wikipedia-like, QA pairs, textbook exercises), generate textbook-style content in underrepresented domains, and expand math/code with worked solutions. Nemotron-CC derived 1.9T synthetic tokens this way, restoring much of what filtering removed; DatologyAI's BeyondWeb and Hugging Face's synthetic-data experiments chart the design space. Two cautions: **diversity collapse** (synthetic data is self-similar — cap the fraction, vary generation prompts and seed documents) and the **license terms of the generator model**.

## 4.4 Tokenization

You need a tokenizer before you can count a single token, and it is a **permanent, irreversible decision** — you cannot change it after pretraining without retraining. The universal choice is **byte-level BPE** (train with Hugging Face `tokenizers` or SentencePiece in BPE mode; byte-level means any input is representable, no UNK tokens). Decisions:

- **Vocabulary size.** 32K (Llama-2 era) is now considered small; the 2026 norm is **~64K–256K** (Llama 3: 128K; many multilingual models 150K+). Larger vocab = better compression — fewer tokens per text, so effectively cheaper training and inference and better multilingual fertility — at the cost of a bigger embedding/unembedding matrix. That is why small models keep smaller vocabs (embeddings would dominate parameters) and often **tie** input/output embeddings, while large models untie. Practical rule: embedding parameters should stay well under ~15–20% of total.
- **Training corpus for the tokenizer.** Sample it to match your intended data mixture — include code and multilingual, or you will over-fragment those domains and quietly tax them forever.
- **Digits.** Split into individual digits or consistent 1–3-digit groups (helps arithmetic).
- **Whitespace.** Include whitespace-run tokens (critical for code indentation).
- **Special tokens.** Reserve document separators plus a generous block of unused special tokens — future you, doing post-training, will thank you.

Alternatively, adopt a proven open tokenizer (Llama-3's 128K, or GPT-4-style cl100k/o200k-class vocabularies) — completely defensible for a first run, and it makes your token counts comparable to published recipes.

## 4.5 Mixtures, curricula, and the annealing phase

**Mixture weights** — what fraction of each source per batch — are a first-class hyperparameter, tuned on the scaling ladder: train small models on candidate mixtures, evaluate on a broad suite, extrapolate. Automated methods exist (DoReMi, data-mixing laws), but honest grid search over a few candidate mixtures at ~1B scale is what most teams actually do. Robust findings:

- **Code and math help general reasoning** well beyond their own domains — this is why modern mixes carry 10–25% code.
- **Diversity protects you** against over-optimizing any single benchmark.
- **Upsample high-value, scarce data** (math, curated sources) relative to natural frequency, within repetition limits (Chapter 2.5).
- **Mixture effects can shift with scale** — a mix optimal at 1B may not be optimal at 70B. Validate direction on the ladder; hold a mid-scale check if budget allows.

**Multi-stage curriculum is now standard.** Bulk phase on the broad mixture; then, during LR decay, shift hard toward the high-quality/high-density mix — curated educational web, textbooks, dense math, high-quality code, synthetic reasoning data, benchmark-adjacent-but-decontaminated material. Because the LR decay and the quality shift coincide, the model "sets" on the good distribution; ablations consistently show several benchmark points won in this annealing phase at fixed total compute. This is why the decay-phase recipe is the part labs guard most closely. Engineering requirement: design the pipeline so mixture weights can change at scheduled step boundaries without regenerating the dataset. (Within-bulk-phase curricula — easy-to-hard ordering — have more mixed evidence; the annealing shift is the near-universal part.)

**Long-context data** for the extension phase: you need genuinely long documents — books, papers, repository-wise concatenated code, long transcripts — not short documents packed together. Upsample by length; consider structured concatenation (ordering files within a repo) so long-range dependencies are real.

## 4.6 Serving tokens to the trainer

Pre-tokenize offline and store fixed-length shards in an efficient binary format — Megatron's indexed `.bin`/`.idx`, MosaicML StreamingDataset, or WebDataset over object storage. Requirements that bite people:

- **Exact resumability.** Dataloader state (position, shuffle seed, epoch) must checkpoint with the model.
- **Two-level shuffling.** Shard order plus within-shard (shuffle buffer).
- **Sequence packing.** Documents vary in length; padding wastes massive compute. Concatenate documents separated by an EOD token to fill every 4K–8K window — and use **intra-document (block-diagonal) attention masking** with reset position IDs so tokens don't attend across unrelated documents. FlashAttention's varlen kernels support this efficiently; it is standard now and slightly improves quality. Best-fit packing beats naive greedy concatenation on truncation waste. Packing typically recovers 20%+ of the throughput padding would burn.
- **Throughput headroom.** Dataloading should use <5% of step time; pre-tokenized binary shards on local NVMe make this easy.

## 4.7 Legal and governance (brief but non-optional)

- **Provenance and licensing.** Track provenance for every source; honor code licenses (The Stack's license-filtering approach is the template). Books and paywalled content are legally fraught; the landscape is actively litigated in multiple jurisdictions in 2026 — if the model is commercial, involve counsel early.
- **Crawling conduct.** Respect robots.txt, site ToS, and emerging opt-out/content-provenance signals in your own crawling.
- **PII.** Scrub personal data; comply with GDPR/CCPA-class obligations, including deletion workflows.
- **Documentation.** Keep a datasheet: sources, filtering, dedup, decontamination, dates, known limitations. Essential for reproducibility, debugging, and defensibility — and decontamination is a credibility issue as much as a quality one.

Treat data governance as a real workstream, not paperwork.
