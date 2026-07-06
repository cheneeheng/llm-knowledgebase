# Chapter 4 — Supervised Fine-Tuning

## 4.1 Conceptual picture

SFT is next-token prediction on demonstrations — mechanically identical to pretraining, just on (prompt, response) pairs with the loss restricted to the response. It is behavior cloning, with behavior cloning's classic properties: the model learns the *format and style* of responses fast, learns *shallow* skills well, and inherits the demonstrations' distribution — including errors, and including claims it has no internal knowledge to support. That last one matters: SFT data asserting facts the base model doesn't know teaches the *behavior of asserting things it doesn't know*, i.e., trains hallucination. Use SFT to shape behavior; use context to supply knowledge; use RL to push reliability beyond what you can demonstrate.

What SFT is for in the 2026 pipeline: (a) installing the chat/tool format and instruction-following; (b) the **cold start** for RL — giving the model the long-CoT or agentic behavior *pattern* so RL has something to amplify; (c) distillation's vehicle (Chapter 10). Nobody ships an SFT-only general model anymore, but every pipeline still begins here, and a bad SFT stage caps everything after it.

## 4.2 Mechanics: templates, masking, packing

- **Chat template.** The serialization of turns into tokens (special tokens for roles, tool calls, and — for thinking models — reasoning spans, e.g. `<think>...</think>`). Use the base family's template unless you have a reason; if training from a raw base, you're defining it — keep it minimal and unambiguous, and freeze it early: template changes invalidate downstream data and deployed prompts. **The template used in training must byte-for-byte match the one used in serving and evals.** Template mismatch is the most common "my fine-tune got worse" bug in existence.
- **Loss masking.** Compute loss only on tokens the model should produce: assistant turns (and reasoning spans), not system/user turns, not tool *outputs* (the environment wrote those — training on them teaches the model to hallucinate tool results; this becomes critical in agentic data, Chapter 9). In multi-turn examples, mask all non-final user/system content but train on every assistant turn.
- **Packing.** Concatenate short examples to fill the sequence length, with attention masking (or FlashAttention varlen/position-id resets) preventing cross-example attention. 2–5× throughput on chat-length data. **Verify your stack actually masks across boundaries** — silent cross-contamination is a classic.
- **The one non-negotiable check:** before any real run, decode 2–3 collated batches back to text and *look at them* — template, masks, packing boundaries, truncation. Five minutes; catches the majority of silent SFT failures.

## 4.3 Hyperparameters

Defaults that work (8B dense, full-parameter):

| Knob | Default | Notes |
|---|---|---|
| LR | 1–2e-5 (≤5e-6 at 70B+) | Cosine or linear decay to ~0, warmup 3% |
| Epochs | 2 (1–4) | More epochs = more style, more forgetting |
| Global batch | 128–512 sequences | Stability knob; small batches are noisy at low LR |
| Seq length | Cover your data; 4–16k chat, 32k+ for long-CoT | Truncating responses mid-thought is poison — filter or raise length |
| Optimizer | AdamW, wd 0.0–0.1, β2 0.95–0.999 | Nothing exotic needed |

Long-CoT SFT (reasoning cold start) differs: smaller datasets (10³–10⁵ traces) of *verified* long traces, 16–64k sequence length, low LR, and the data quality bar is everything — s1 showed 1k excellent traces suffice to unlock test-time scaling on a strong base. Guard against forgetting throughout: mix 10–30% general/previous-stage data into narrow SFT, keep LR low, prefer fewer epochs. Checkpoint every epoch (or more often) and eval each checkpoint — SFT overfits fast and the best checkpoint is often not the last.

## 4.4 Full fine-tune vs LoRA/QLoRA

LoRA trains low-rank adapters on frozen weights: ~1–2% of parameters, fits 8B training on one 24GB card (QLoRA: frozen weights in 4-bit, even smaller), merges into the base at deploy, and *forgets less* because the base is untouched. The 2025 Thinking Machines "LoRA without regret" result sharpened the folklore: **LoRA matches full fine-tuning whenever the dataset's information content fits the adapter capacity** — which covers most instruction-tuning-scale SFT and essentially all RL (RL's learning signal per episode is tiny) — *provided* you apply LoRA to **all layers (MLP/MoE, not just attention)** and use ~10× the full-FT learning rate. LoRA underperforms on large-corpus SFT (many millions of examples) where capacity binds.

Decision rule: LoRA by default for narrow/domain work, adapters-per-customer, RL experiments, and anything under ~1M examples; full FT for flagship general models, large mixtures, and when you're distilling a whole model family. Typical LoRA config: r=16–64 (r=256 for heavier SFT), α=2r, all linear layers, LR ~1e-4–2e-4.

## 4.5 Rejection-sampling fine-tuning

The bridge between SFT and RL, and a stage in most industrial recipes (Llama 2/3 ran iterative rounds of it): sample k responses per prompt *from your own model* (k=4–32, temperature ~0.7–1.0), keep the best by verifier or reward model, SFT on the keepers, repeat. Also the standard way to convert an RL-trained big model's wins back into clean SFT data (R1's stage 3 built ~600k reasoning + ~200k general samples this way). It is embarrassingly simple, needs no RL infra, and captures a surprising fraction of RL's gains on verifiable tasks — the correct first "RL-flavored" experiment for any team, and a sanity baseline any real RL run must beat.
