# Chapter 5 — Long-Context Extension

## 5.1 Conceptual picture

Almost every base model is pretrained at a short sequence length — 4K or 8K — because attention is quadratic in sequence length and training long is expensive per token. Long-context extension is the midtraining stage that stretches that window to 32K, 128K, 1M+ *after* the bulk of pretraining, cheaply, by training on a small budget of long documents while surgically adjusting the model's position encoding.

Why it is a separate stage with its own machinery: the bottleneck is not knowledge, it is **positional generalization**. A model trained only on positions 0–4095 has never seen position 100,000 and its rotary embeddings produce garbage there. The fix is partly an *architecture edit* (rescale the position encoding so far-apart tokens land in a range the model has seen) and partly *training* (a short CPT-style run on long data so the model learns to actually use the extended range). That architecture-edit component is what makes long-context extension unlike every other technique in this series.

## 5.2 RoPE and why extension works

Modern models use **RoPE** (Rotary Position Embedding): position is encoded by rotating query/key vectors by an angle proportional to position, at a spectrum of frequencies (a base frequency, often 10,000, sets the spectrum). High-frequency dimensions rotate fast (encode local order); low-frequency dimensions rotate slowly (encode long-range position). The model learns to read these rotations — but only over the *range of angles it saw in training*. Extend naively and the low-frequency dimensions rotate into unseen territory; the model breaks.

The families of fixes, all reshaping how position maps to angle:

- **Position Interpolation (PI)** — linearly squeeze positions so the new max length maps to the old trained range. Simple, but compresses local resolution; needs fine-tuning.
- **NTK-aware / Adjusted Base Frequency (ABF)** — instead of squeezing positions, *increase the RoPE base frequency* so the angle spectrum stretches. This preserves high-frequency (local) resolution while extending low-frequency (long-range) reach. The dominant approach. Published base-frequency choices: Phi-4 uses 250K for 16K; Qwen3 uses 1M for 32K; Hunyuan uses 1B for 256K; Pangu-Ultra 25.6M for 128K. **Rule of thumb: the longer the target, the higher the base you set** (roughly, base scales super-linearly with target length).
- **YaRN** — the refinement most 2026 models use (DeepSeek-V3, Qwen3). It splits RoPE dimensions into three frequency bands and treats each differently: extrapolate the high-frequency (local) dims untouched, interpolate the low-frequency (long-range) dims, and blend in between — plus a small attention-temperature tweak. Reaches 128K+ with only **400–600 fine-tuning steps**, a 2.5–10× reduction in extension tokens vs naive PI.
- **LongRoPE** — searches per-dimension rescaling factors (non-uniform interpolation) and extends to **2M+** tokens, with a progressive schedule. The go-to when you need extreme context.

You do not need to implement these — every framework ships them as a config flag (`rope_scaling: {type: yarn, factor: ..., original_max_position: ...}`). You need to *choose* the method and target and then *train briefly* to realize it.

## 5.3 Progressive, multi-stage extension

The single most important operational fact: **extend in stages, not in one jump.** Going 4K → 128K directly wastes tokens and degrades short-context performance. The published recipes are almost all two-to-six stage ladders:

| Model | Ladder |
|---|---|
| DeepSeek-V3 | 4K → 32K → 128K, two YaRN phases of ~1000 steps each |
| Qwen3 | pretrain 4K → long-context stage at 32K |
| Llama 3 | 8K → 128K in **six** stages, ~800B tokens total in the extension |
| Most 2025 models | base → 32K → 128K (two-stage is the mode) |

At each stage you raise the RoPE base / scaling factor to the new target, then train a short CPT-style run on data at that length until short- and long-context evals both stabilize, then step up. The staging keeps distributional continuity (Chapter 1) — each step is a nudge the model can absorb without shocking its short-context ability.

## 5.4 Long-context data construction

The data is as important as the RoPE surgery, and the classic mistake is training *only* on long documents.

- **Mix lengths.** Training solely on 128K sequences degrades performance at *all* lengths — the model over-fits to needing long range. Use a mixture: Qwen3's long stage is ~75% sequences in the 16–32K band and ~25% shorter (4–16K). Keep a meaningful fraction of short data throughout.
- **Genuinely long, coherent documents.** Books, long code repositories (whole-repo packing), legal/scientific documents, long conversations. Concatenating unrelated short docs to fill 128K teaches nothing about long-range dependency — the model learns the tokens are unrelated. Prefer natively long documents; when packing, keep related content together.
- **Long-range-dependency data.** The highest-value long-context data forces the model to *use* distant tokens: multi-document QA, repository-level code completion, summarization of long inputs, "needle" style synthetic tasks where the answer depends on a fact planted far back. A few percent of this dramatically improves retrieval-over-context.
- **Budget.** Long-context extension is cheap in tokens: **tens to low-hundreds of billions** (DeepSeek: ~2×1000 steps; Llama 3: ~800B across six stages is the high end). You are teaching a skill, not new knowledge.

## 5.5 Evaluation: the part everyone gets wrong

Perplexity on long documents is a *terrible* long-context metric — it drops smoothly while the model remains unable to actually retrieve from context. Use task-based long-context evals:

- **Needle-in-a-Haystack (NIAH)** — plant a fact at varying depths in a long context, ask for it. Table stakes; a model that fails NIAH is not really long-context regardless of its trained length. Multi-needle variants are harder and more honest.
- **RULER** — synthetic tasks (multi-hop, aggregation, variable tracking) across lengths; reveals the *effective* context length, which is usually far below the *trained* length. A model "trained to 128K" is often only reliable to ~32–64K on RULER. Report effective, not nominal, length.
- **LongBench / ∞Bench / HELMET** — realistic long-context tasks (QA, summarization, code). Use for the final quality bar.
- **Short-context regression check** — always re-run your standard short evals (MMLU, etc.). Extension that buys 128K at the cost of 4K quality is usually a bad trade; the staging and length-mixing in 5.3–5.4 exist to prevent it.

## 5.6 Decisions

1. **Use YaRN (or ABF/NTK) as the default extension method**; LongRoPE only for extreme (500K+) targets. Set the RoPE base per your target length.
2. **Extend progressively** — two stages minimum (base → 32K → 128K), more for bigger jumps.
3. **Mix sequence lengths**; never train only on max-length data. Keep ~25% shorter.
4. **Use natively long, coherent documents** plus a few percent of long-range-dependency tasks; don't just pack short docs.
5. **Budget tens of billions of tokens** — it is a skill, not knowledge.
6. **Evaluate with RULER/NIAH and report effective context length**, not the nominal trained length; always check short-context regression.

Next: tokenizer surgery — the one intervention that must precede everything else because it changes the token stream itself.
