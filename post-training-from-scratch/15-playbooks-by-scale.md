# Chapter 15 — Playbooks by Scale

Concrete end-to-end recipes. Each assumes the eval-first discipline of Chapter 2.4: target eval written, base and frontier models scored, kill criteria set. Costs are mid-2026 cloud prices, final-run only (multiply 3–10× for iteration).

## 15.1 Playbook A — one consumer GPU: the specialist (~$0–50, a weekend)

**Goal:** a 4–8B model doing one narrow job (format transform, domain Q&A style, classification-with-rationale) at API-quality for that job.
**Recipe:** take the best *instruct* model that fits (Qwen3-4B/8B class) — you're inheriting its post-training, not redoing it. Build 1–10k demonstrations: distill from a frontier teacher over your real prompts, verify/filter (Chapter 3), hold out 10%. QLoRA SFT via Unsloth or LLaMA-Factory: r=16–32, α=2r, all linear layers, LR 1e-4–2e-4, 2–3 epochs, decode-and-read a batch before training. Optional: 1 round of judge-labeled on-policy DPO pairs if style needs polish (LR 5e-6 with LoRA, β=0.1).
**Reality checks:** if RAG or prompting solves it, stop (Chapter 2.1); most failures here are chat-template mismatch or training on knowledge instead of behavior.

## 15.2 Playbook B — one 8×H100 node: the full-pipeline assistant ($1–6k, 1–2 weeks)

**Goal:** a general instruction-following 7–9B model with your domain strengths — Tulu-3-8B-class, the best learning project in post-training.
**Recipe (follow Tulu 3 / OLMo 3's published playbooks — they are the documentation):**
1. **SFT** from the *base* model on an open mix (Tulu 3 SFT mix or Dolci, ~1M samples) blended with your domain demonstrations. Full-param, LR 5e-6–2e-5, 2 epochs, packing on, ~6–12h.
2. **DPO** on ~100–300k on-policy-ish pairs (open preference sets + pairs sampled from *your* SFT model, judge-labeled). LR 5e-7, β=0.1, 1 epoch, ~2–4h. Watch length inflation and chosen-logprob (Chapter 5).
3. **RLVR taste:** TRL GRPOTrainer or verl on one node — GSM8K/MATH-class prompts with a math-verify grader, G=8–16, LR 1e-6, a few hundred steps. Expect modest but real gains and your first entropy/length curves (Chapter 8).
4. **Panel + regression gate** per Chapter 14; compare against the *official* Tulu/OLMo checkpoints as your sanity baseline.

## 15.3 Playbook C — 16–64 H100s, small lab: the reasoning/domain-RL model ($30–300k, 4–10 weeks)

**Goal:** a 14–32B thinking model that beats every open model on *your* verifiable domain (your test suites, your math/logic tasks, your pipelines).
**Recipe:**
1. **Base:** strongest open base/instruct in budget with heavy math/code pretraining (Qwen3-14B/32B class). Sub-7B target? Distill instead (Chapter 8.5/10).
2. **Cold-start SFT:** 5–50k verified long-CoT traces (distilled from an open thinking model over your domain + open reasoning sets), 32k context.
3. **Verifiable dataset:** 10⁴–10⁵ prompts with hardened graders; pass-rate filter to 0<p<1 against the cold-start model; decontaminate.
4. **RL:** verl, GRPO+DAPO tricks (clip-higher 0.28, dynamic sampling, no KL, token-level loss, overlong penalty), G=16, rollout batch ~512 prompts, LR 1e-6, temp 1.0, staged max-length 8k→24k, TIS on, colocated to start. Refresh the pass-rate filter as it climbs.
5. **Generality restore:** rejection-sample the RL'd model into SFT data, blend with general data, re-SFT or light general RL (the R1 stage-3/4 move); OPD from the pre-RL checkpoint if chat quality dipped.
6. **Operate it like Chapter 12/14 say:** logprob-gap dashboard, holdout grader, transcripts daily, pass@k probe set, release gate.
**Kill criteria matter here:** if step-4 reward stalls for a week after data refresh and hyperparameter sanity passes, the base model is your ceiling — move up a size or revisit the task, don't grind.

## 15.4 Playbook D — the frontier-style pipeline (hundreds–thousands of GPUs, months, $10M+)

What the full apparatus looks like when the model family *is* the product — a map, not a recipe:
- **Parallel workstreams, one release train:** data ops (human + synthetic pipelines, Chapter 3), RM/judge team (Chapter 6), reasoning RL, agentic RL with an environments team (Chapter 9), safety (Chapter 11), evals (Chapter 14) — each owning stages of a multi-stage recipe: SFT → RM training → reasoning RLVR → general RLHF → agentic RL → safety integration → distillation of the family (OPD, Chapter 10) → merge/regression/release gate.
- **Infra is a platform team:** disaggregated async RL (Chapter 12) on a fork of verl/Megatron or in-house, environment fleets, judge fleets, experiment tracking at hundreds-of-runs concurrency.
- **The scaling behavior to internalize:** RL compute at this tier now rivals pretraining; capability comes increasingly from environments + verifiers + RL scale, and the family's smaller models are made by distillation, not by rerunning the pipeline.
- **What transfers down from this tier to yours:** the *shape* — eval-first, staged pipeline, proxies audited, generality restored after every narrowing stage, distill downward — is identical in Playbook B. Only the zeroes differ.
