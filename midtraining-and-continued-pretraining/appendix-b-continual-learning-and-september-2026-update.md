# Appendix B — Continual Learning, Test-Time Training, and the September 2026 Update

*This series was compiled in early July 2026. It treats midtraining as a **one-shot** operation: take a base model, perturb it toward a target, and manage the forgetting. What it did not cover is the **repeated** version of that problem: a model that must keep absorbing new knowledge over its deployed life, or even while it runs. That is the continual-learning question. It had become a major research theme by mid-2026, and "why can't the model just keep learning?" is now a common product question. This appendix fills that gap (B.2–B.6), then records what changed in midtraining practice between July and September 2026 (B.7).*

## B.1 Four things that get called "continual learning"

The term is overloaded, and conflating its meanings leads to bad architecture decisions. Separate them:

| Mode | What updates | When | Where it's covered |
|---|---|---|---|
| **Continued pretraining / midtraining** | All weights (or adapters) | Offline, one campaign | This series, Chapters 3–8 |
| **Continual learning (lifecycle)** | Weights, repeatedly | Offline, on a recurring cadence (monthly/quarterly refreshes) | This appendix, B.2–B.3 |
| **Online / deployment-time learning** | Adapters or small parameter sets, from live signals | Near-continuously | This appendix, B.4 |
| **Test-time training / test-time memory** | Fast weights or a memory module, *inside a single inference* | During the forward pass | This appendix, B.5 |

Most of what ships under the label "the model learns" is none of these. It is **context and retrieval**: memory stores, RAG, and conversation summaries (application series Chapters 5 and 7). No weights change, and that is usually the right answer.

## B.2 Lifecycle continual learning: the refresh cadence

A deployed model's knowledge goes stale from the day of its data cutoff. The practical form of continual learning is a **recurring midtraining campaign** in which each refresh is a small CPT run. The following protocol reuses the series' existing tools:

1. **Version the data by time slice.** Each refresh trains on the new slice (for example, the last quarter's crawl, docs, and code) plus **replay** (Chapter 8.3). Replay now has two parts: general pretraining-distribution data and *earlier refresh slices*. Without the second part, refresh N quietly erodes what refresh N−1 installed.
2. **Keep the forgetting harness cumulative.** Add each refresh's target evals to the permanent suite. The Chapter 8 budget applies to *every* prior target, not only to general capability: ≤1–2 points of drop, and anything above 3 points is a bug.
3. **Decide whether each refresh starts from the last refresh or from the original base.** Chaining refreshes (N from N−1) is cheaper but compounds drift. Periodically re-running from the base with all slices plus replay resets the accumulated error. The common compromise is to chain a few refreshes, then re-base.
4. **Redo post-training on top, or merge.** A CPT'd base loses its post-trained behavior. Either re-run the post-training pipeline, which is expensive but clean, or merge a refreshed-base delta into the deployed model (post-training series Chapter 10). Merging is cheaper, but it needs the full eval harness to confirm the merge held.

**Anchor fact:** frontier labs still update weights *offline*. Every update needs evaluation, safety review, and a rollback path. A monthly or quarterly refresh cadence, with retrieval covering the gap between refreshes, is the realistic state of the art in 2026.

## B.3 Which update method forgets least

The midtraining series ranks anti-forgetting tools as replay first, then LR discipline, then LoRA and parameter expansion (Chapter 8.5). For *repeated* updates, one more result matters: **on-policy RL forgets markedly less than SFT at matched target gains.** The explanation offered is that RL updates stay close to the model's own distribution, since the model trains on its own samples, while SFT drags the model toward an external distribution. This is sometimes called "RL's razor" (Shenfeld et al., 2025). The practical implication: when the new capability can be expressed as a verifiable or rubric-scored task, **deliver it through RL rather than SFT to reduce cumulative drift**. Keep SFT and CPT for bulk knowledge, which RL cannot inject.

Parameter isolation scales better than any penalty method over many updates. Examples: a LoRA adapter per refresh or domain, or new experts added to an MoE. Frozen base weights cannot be forgotten. The costs move to serving (adapter routing, inference series Chapter 9) and to composition (adapters trained separately do not always stack cleanly).

## B.4 Online learning from deployment signals

In production, updating from live traffic looks like this:

- **Adapter-level updates.** Periodically train LoRA/IA³-class adapters on logged, filtered interactions. The base model stays fixed, rollback means swapping the adapter out, and the blast radius is bounded.
- **Preference updates from logs.** Convert explicit and implicit user signals into preference pairs, then run DPO-class updates (post-training series Chapter 5). The danger is well documented: **optimizing for engagement signals amplifies sycophancy.** Any preference loop fed by user approval needs a counter-signal, such as held-out quality evals and sycophancy probes, before any update ships.
- **The data flywheel.** Most "online learning" value comes from the loop itself (cross-cutting Chapter 8): logs → failure mining → curated data → the next offline update. Weight-updating on the fly adds little to this.

**Guardrails for any deployment-time update:** a canary rollout, automatic comparison against the frozen predecessor on the full harness, data-poisoning defenses on the ingestion path (any user can write to your training set now; see cross-cutting Chapter 6), and a one-command rollback.

## B.5 Test-time training and test-time memory

The research frontier makes *learning part of inference*. The model updates a set of fast weights or a memory module while it processes the current input, so a long document or session is compressed into parameters instead of a growing KV cache.

- **TTT layers** (Sun et al., 2024). The hidden state of a sequence layer is itself a small model, trained by a self-supervised gradient step on each incoming chunk. Linear-attention and delta-rule layers can be read as a special case. This is the conceptual link to the KDA and Gated DeltaNet hybrids in pretraining series Appendix B.
- **Titans-style neural memory** (Behrouz et al., 2025). A long-term memory module is updated at test time using a surprise signal: inputs the memory predicts poorly get written more strongly. **Nested-learning** proposals (Google, late 2025) extend this to multiple update frequencies within one model.
- **End-to-end test-time training for long context.** These methods treat language modeling over very long inputs as continual learning within the sequence, and 2025–2026 work reports competitive long-context quality at constant memory.

**Status in September 2026:** the idea is architecturally live, because the delta-rule hybrids now shipping at frontier scale are its descendants. As a mechanism that *durably* updates a deployed model's knowledge across sessions, it remains research-grade. Stability guarantees, safety review of self-modifying weights, and evaluation methodology are all unresolved. Surveys such as *Continual Learning in Transition* (arXiv:2608.06216, August 2026) are the entry point. **Don't design a product around test-time weight updates yet.**

## B.6 Decisions (continual learning)

1. **Name the mode.** Most "keep learning" requirements are met by retrieval and memory (no weights) plus a periodic refresh. Test-time weight updates are not a product dependency yet.
2. **Run refreshes as small CPT campaigns** with time-sliced data, replay that includes *previous refresh slices*, and a forgetting harness that grows cumulatively.
3. **Chain refreshes, then re-base periodically** to reset compounding drift.
4. **Prefer RL for repeatable behavioral updates.** It forgets less than SFT. Keep CPT/SFT for knowledge.
5. **Isolate parameters for high-frequency updates** (adapters per refresh or domain) so rollback is a swap.
6. **Gate every deployment-time update** with a canary, predecessor comparison, poisoning defenses, and a sycophancy counter-signal.

## B.7 September 2026 update: midtraining practice

What changed after this series was compiled, mapped to its chapters:

- **Long-context extension is dissolving into pretraining for efficient-attention models (Chapter 5).** DeepSeek-V4 ramped sequence length from 4K to 1M *during* 32T+ tokens of pretraining. That is affordable only because its compressed/sparse attention is sub-quadratic. Kimi K3 (hybrid linear attention) and Qwen3.8 ship at 1M-class context. For standard-attention models, Chapter 5's dedicated extension phase still applies. For hybrid or sparse models, plan context growth as a schedule across the whole run. A separate late phase is no longer the only design.
- **The two-phase "breadth then quality" pretraining split is now documented at 20T scale (Chapter 2).** Nemotron 3 Ultra ran 15T tokens of broad data, then 5T of quality-weighted data, under WSD. This is the Chapter 2 annealing pattern with the "decay" window stretched to a quarter of the run, and a useful data point for sizing your own quality phase.
- **Base reuse across generations is the norm (Chapters 3–4).** GLM-5.3 kept the GLM-5.2 base and got its gains from scaled post-training on agent environments. Qwen released the Qwen3.8-Flash-Next *base* as the shared foundation for its Omni variants (multimodal series Appendix A). The economic logic of this series is being applied at the frontier: re-train the base only when the architecture changes, and otherwise midtrain or post-train on the existing one.
- **Native multimodality is moving into the base (multimodal series Chapter 5).** DeepSeek-V4.1-Flash, GLM-5.3-Flash, and Kimi K3 all ship natively multimodal. When you continue-pretrain one of these, your replay mixture must include *multimodal* general data. Text-only replay protects text ability and silently erodes vision (multimodal series Chapter 13).

## B.8 Sources

- Shenfeld, Pari, Agrawal, *RL's Razor: Why Online Reinforcement Learning Forgets Less*, 2025 — arXiv:2509.04259
- Sun et al., *Learning to (Learn at Test Time): RNNs with Expressive Hidden States*, 2024 — arXiv:2407.04620
- Behrouz, Zhong, Mirrokni, *Titans: Learning to Memorize at Test Time*, 2025 — arXiv:2501.00663
- *Continual Learning in Transition*, 2026 — arXiv:2608.06216 (survey entry point)
- DeepSeek-AI, *DeepSeek-V4*, 2026 — arXiv:2606.19348
- NVIDIA, *Nemotron 3 Ultra* technical report, 2026 — arXiv:2606.15007
- Z.ai GLM-5.3 release materials (August 2026); Qwen3.8 repository — https://github.com/QwenLM/Qwen3.8
