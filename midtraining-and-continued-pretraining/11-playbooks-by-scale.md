# Chapter 11 — Playbooks by Scale

Concrete end-to-end recipes assembling the previous chapters. Each is a starting point to adapt, not a law. All assume you have read Chapters 2–10 for the *why*.

---

## 11.1 Single GPU — domain LoRA CPT

**Goal:** give a 7–8B base a domain flavor (your docs, a vertical's terminology) on one 24–48GB card.

- **Framework:** Unsloth (fastest QLoRA) or Axolotl.
- **Data:** 1–5B domain tokens. Convert ~30–50% to reading-comprehension format (Chapter 4.4). Mix in **20% general replay** (FineWeb-Edu).
- **Method:** QLoRA, r=32–64, α=2r, **all linear layers**, LR ~1–2e-4 (10× a full-FT LR, per LoRA-without-regret).
- **Schedule:** short warmup (~3%), cosine or WSD decay to ~0, 1–3 epochs.
- **Eval:** MMLU + HellaSwag (retention) + your domain task, before and after. Hold general drop ≤2 points.
- **Cost/time:** hours to ~1 day; tens of dollars of GPU.
- **Ship:** merge the adapter into the base for deployment, or keep it as a swappable adapter.

**When this is the whole job:** narrow domain, knowledge/flavor goal, retention matters. If your domain eval barely moves, the corpus is too small or too generic — augment or reconsider CPT vs RAG.

---

## 11.2 One node (8×H100) — full domain CPT

**Goal:** real domain competence on a 7–13B base (the SaulLM shape).

- **Framework:** Axolotl (config-driven) or torchtitan; DeepSpeed ZeRO if memory-tight.
- **Data:** 20–50B domain tokens + **20% replay**, front-loaded (higher early, tapering). Include reading-comprehension conversion for any small/precious sub-corpus.
- **Method:** full-parameter CPT. **Re-warm LR to ~2–3e-4**, warmup a few hundred steps, decay to near-zero (WSD). AdamW β=(0.9, 0.95), wd 0.01.
- **Stages:** CPT → (optional) domain-SFT on instructions for a shippable assistant.
- **Guard the stability gap:** don't judge early checkpoints; eval at several points; use 2–4 short epochs if the corpus is small.
- **Eval:** four-axis harness every epoch; select on the paired trade; decontaminate the corpus against your domain benchmark first.
- **Cost/time:** ~cluster-week; low tens of thousands of dollars.

**Failure signal:** general eval down >3 points → replay too low or LR too high. Fix before proceeding.

---

## 11.3 Few nodes — long-context extension

**Goal:** take an 8–70B base from 8K to 128K.

- **Framework:** Megatron/NeMo (large) or torchtitan (mid). Confirm YaRN/RoPE-scaling and multi-stage schedule support.
- **Method:** progressive, two stages — **8K → 32K → 128K**. At each stage raise RoPE base / set YaRN factor for the new target (e.g. base ~1M for 32K, higher for 128K).
- **Data per stage:** **mix lengths** (~75% at the stage's target band, ~25% shorter). Natively long, coherent documents (books, whole repos, long docs) + a few % long-range-dependency tasks (multi-doc QA, repo completion, synthetic needles). Keep replay of general short data.
- **Budget:** tens of billions of tokens total across stages (DeepSeek did ~2×1000 steps; Llama 3's six-stage was the ~800B high end).
- **Eval:** RULER (report **effective** length) + multi-needle NIAH + a realistic long suite; **always** re-check short-context (MMLU) for regression.
- **Cost/time:** days of cluster; cheap in tokens.

**Failure signal:** RULER effective length far below nominal → extend more gradually, add long-range-dependency data. Short-context regression → too much long-only data; raise the short fraction.

---

## 11.4 Frontier — a full midtraining stage

**Goal:** you're training a flagship base and want the midtraining stage that turns it into a strong, RL-ready, long-context base (the Qwen3/OLMo 3/DeepSeek shape).

Assemble the whole series in sequence:

1. **Tokenizer** (Chapter 6): settled *before* pretraining; touch here only for a planned multilingual/domain expansion, with mean-of-constituents init + warm-start.
2. **Stable-phase pretraining** ends; branch a frozen trunk checkpoint (WSD).
3. **Annealing / decay stage** (Chapter 2): 10–20% of tokens, LR→0, mixture swapped ~30% toward quality — upweight math/code/reasoning (10–15%), a few % instruction/CoT-formatted (unmasked, Chapter 7), a few % synthetic. **Rank the mixture with microanneals first.**
4. **Reasoning injection** folded into the decay mixture (Chapter 7) — front-load long-CoT so post-training RL starts warm; verify traces.
5. **Long-context extension** (Chapter 5) last in the self-supervised span: progressive 32K → 128K(+), length-mixed data.
6. **Four-axis eval** throughout (Chapter 10); decontaminate; hold a private eval; select the trunk-to-ship checkpoint on the paired trade.

- **Framework:** Megatron-Core / NeMo, full cluster.
- **Scale/cost:** hundreds of billions of midtraining tokens; the midtraining+long-context portion is days-to-weeks of full cluster. Public anchor: OLMo 3's entire 32B pipeline (incl. base) was ~47 days / 1024 H100 / ~$2.75M — midtraining alone is a fraction.
- **Hand-off:** the output is an RL-ready base. Post-training (the companion series) takes it from here; if you front-loaded reasoning well, you may lighten or skip cold-start SFT.

---

## 11.5 The cross-cutting checklist

Regardless of scale, before any midtraining run:

- [ ] Baseline evals measured on *your* harness (not published numbers).
- [ ] Replay source and ratio chosen (default 20%, front-loaded).
- [ ] Mixture ratios verified by decoding real batches.
- [ ] Corpus decontaminated against every benchmark you'll report.
- [ ] LR schedule confirmed (re-warmup fraction, WSD decay, multi-stage if long-context).
- [ ] Checkpoint format converted and *verified with eval prompts* before GPU spend.
- [ ] Checkpointing frequent enough for selection.
- [ ] A private held-out eval reserved.

Next: failure modes — the specific ways these runs go wrong, as symptom → cause → fix.
