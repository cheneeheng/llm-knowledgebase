# Chapter 12 — Failure Modes

Midtraining fails in a characteristic register: not the loud divergences of pretraining nor the silent reward-hacking of post-training, but **quiet trades gone bad** — you gained the thing you measured and lost something you didn't. Each entry below is symptom → cause → fix. Most trace back to two root causes: under-replaying, and shocking a model that wanted a nudge.

---

## 12.1 Silent forgetting

**Symptom:** target metric up nicely; weeks later, users report the model is worse at general tasks / lost a skill you never re-measured.
**Cause:** you didn't run a general-eval harness, or you ran too narrow a one. The training loss looked great the whole time — forgetting is invisible in the loss on your new data.
**Fix:** the non-negotiable four-axis harness (Chapter 10) on every checkpoint, from a self-measured baseline. Prevention only; there is no cure after you've shipped except re-running with replay. This is the single most common midtraining failure.

---

## 12.2 Loss spike on re-warmup

**Symptom:** loss jumps sharply in the first hundreds of steps of CPT, sometimes never fully recovers.
**Cause:** LR re-warmed too high or too fast for a model that finished pretraining at LR≈0; the sudden large steps blow it off its optimum.
**Fix:** re-warm to a *fraction* of from-scratch peak (~1/3–1/10), lengthen the warmup, decay to near-zero (Chapter 3.3). If it persists, the cause is the pretraining-class ones (bad data batch, precision) — bisect the data and check for NaNs/Infs.

---

## 12.3 The stability gap mistaken for failure

**Symptom:** target metric *drops* in early CPT checkpoints; you conclude the run is broken and kill it.
**Cause:** the normal stability gap (Chapter 3.4) — the model destabilizes from its old optimum before re-converging on the new distribution. It's expected, not failure.
**Fix:** don't judge early checkpoints; eval at several points and expect a dip-then-climb. Mitigate the gap's depth with gentle warmup, front-loaded replay, and (small corpora) multiple short epochs. Only kill a run if the metric hasn't recovered past baseline by mid-run.

---

## 12.4 Under-replay collapse

**Symptom:** general eval drops 4–8 points; model becomes narrow, repeats domain phrasings out of context, "forgot how to be a general model."
**Cause:** replay ratio too low (or zero) for the LR / distribution shift. The single most common *tuning* mistake.
**Fix:** raise replay to 20–30%, front-load it, and/or lower LR. Re-run. Budget is ≤2 points; >3 means fix this before anything else (Chapters 4, 8).

---

## 12.5 Long-context degradation (short-context regression)

**Symptom:** model now handles 128K, but its 4K quality (MMLU, everyday chat) got noticeably worse.
**Cause:** trained the extension on *only* long sequences, over-fitting the model to needing long range; and/or jumped context length too aggressively in one step.
**Fix:** mix lengths (~25% shorter data throughout, Chapter 5.4), extend progressively in stages, keep general short replay. Always run the short-context regression check (Chapter 10) — this trade is usually bad and you should catch it.

---

## 12.6 Nominal-vs-effective context gap

**Symptom:** you advertise 128K; RULER shows reliable performance only to ~32K; users hit a wall retrieving from long inputs.
**Cause:** you extended the *positional range* (RoPE surgery) but under-trained the *use* of it, or evaluated with perplexity/NIAH-single only, which pass while real long-range reasoning fails.
**Fix:** add long-range-dependency data (multi-doc QA, repo completion, multi-needle), train longer at the target length, and evaluate/report **effective** length via RULER, not nominal (Chapters 5.5, 10.3).

---

## 12.7 Decay-phase overfit

**Symptom:** annealing on a small, highly-curated (often synthetic) mixture; benchmarks that resemble the mixture soar, everything else stalls or drops, outputs take on a synthetic "textbook" voice.
**Cause:** the decay phase, at low LR, settles hard into whatever it sees — a too-narrow or contaminated decay mixture locks the model onto it. Style lock-in from heavy CoT/synthetic (Chapter 7.5) is a variant.
**Fix:** keep the decay mixture *diverse* (upweight, don't dominate — 10–15% per capability), cap synthetic at a few percent, decontaminate it hardest of all (synthetic regurgitates benchmarks), and hold the ~70/30 continuity/novel split (Chapter 2.3). Rank with microanneals to catch a bad mixture before the full decay.

---

## 12.8 Tokenizer drift / silent corruption

**Symptom:** after vocabulary surgery, quality tanks in ways that don't correlate with anything; occasional garbled output; special-token behavior broken.
**Cause:** random-initialized new embeddings, an untrained LM head for new tokens, a new-token collision with a reserved/special token, or a broken merge that corrupts encoding round-trips.
**Fix:** never random-init (mean-of-constituents minimum, init the LM head too), run the verification ritual (round-trip, fertility, special-token audit, decode batches — Chapter 6.6), and freeze the tokenizer permanently. Prevention only — a tokenizer bug discovered post-training means re-doing everything downstream.

---

## 12.9 Checkpoint-conversion corruption

**Symptom:** loaded base model performs far below its published numbers before you've trained anything; or trains but is subtly broken.
**Cause:** format conversion (HF ↔ Megatron ↔ NeMo) flipped a dtype, mishandled tied/untied embeddings, or dropped the RoPE-scaling config.
**Fix:** eval a few prompts on the *converted* checkpoint before spending GPU-hours (Chapter 9.3). Diff the config (RoPE base, vocab size, tie_word_embeddings, dtype) against the source. Treat conversion as a verified step, not a copy.

---

## 12.10 Contaminated midtraining data

**Symptom:** reported benchmarks jump; private held-out eval and real-world use don't improve or get worse.
**Cause:** synthetic, reading-comprehension-converted, or curated midtraining data overlapped your benchmarks (synthetic data especially regurgitates memorized benchmark items).
**Fix:** decontaminate (n-gram/embedding overlap) against every eval before training, filter synthetic as hard as scraped, and always keep a private eval the loop never saw. When public and private evals disagree, trust the private one (Chapter 10.4).

---

## 12.11 The meta-lesson

Nine of these ten failures are prevented by two habits: **replay generously, and measure both axes against a real baseline on every checkpoint.** The tenth (tokenizer/conversion corruption) is prevented by *verifying artifacts before spending compute on them.* Midtraining has no exotic failure modes — it has the predictable failures of editing a working model without watching what you're breaking. Watch, and the stage is mechanical.

Next: essential reading.
