# Chapter 3 — Data

## 3.1 Conceptual picture

Pretraining data is measured in trillions of tokens and refined in bulk; post-training data is measured in *examples* — thousands to a few million — and curated nearly one class at a time. Five distinct kinds of data feed the pipeline, and confusing them causes real design errors:

1. **Prompts** — the questions/tasks. The most underrated asset: prompt distribution *is* your product definition, and in RL, prompts are the entire dataset (the model writes the responses itself).
2. **Demonstrations** — prompt + gold response (possibly multi-turn, tool calls, long CoT). Feeds SFT.
3. **Preferences** — prompt + chosen/rejected pair (or scalar ratings). Feeds DPO and reward-model training.
4. **Verifiable tasks** — prompt + machine-checkable success criterion: math answer, unit tests, constraint checker, environment goal. Feeds RLVR/agentic RL.
5. **Trajectories** — full agent rollouts (states, tool calls, observations). Feeds agentic SFT and RL.

## 3.2 Where prompts come from

- **Open mixes** — start here: Tulu 3 (~939k SFT prompts, the reference open general mix), OLMo 3's **Dolci** suite (per-stage mixes for SFT/DPO/RLVR, tuned for reasoning + tool use), SmolTalk, WildChat/LMSYS-Chat (real user prompts), OpenAssistant. For verifiable tasks: NuminaMath, OpenR1/DeepScaleR math sets, code-contest sets with tests (TACO, LiveCodeBench-style), IFEval-style constraint prompts.
- **Your product logs** — the best prompt distribution for your use case is the real one; dedupe, filter PII, and sample by user intent clusters.
- **Synthetic prompt generation** — the lineage: Self-Instruct (seed tasks → LLM expands) → Evol-Instruct (mutate prompts toward complexity) → **persona-driven** (PersonaHub: condition generation on millions of synthetic personas for diversity — used heavily in Tulu 3) → **Magpie** (prompt an aligned model with only its chat template preamble; it *emits* a plausible user query — startlingly effective and free). Diversity is the whole game; dedupe aggressively (exact + fuzzy + embedding), and cap per-source/per-template counts.

## 3.3 Demonstrations: synthetic first, humans for what's left

2026 reality: **most SFT tokens are model-generated.** The standard pattern is *teacher distillation*: a stronger model answers your prompts, optionally with rejection sampling — sample k, keep the ones passing a verifier/judge — so quality is bounded by your filter, not the teacher's average. (Mind the ToS chain from Chapter 2 if the teacher is a commercial API; open teachers like DeepSeek/Qwen avoid the issue.) On-policy variants (sample from *your own* model, keep verified successes — RFT/STaR-style) sidestep imitation-of-alien-style problems and feed the loop that eventually becomes RL.

Humans remain irreplaceable for: taste-defining "golden" sets (LIMA's lesson: 1k exceptional demonstrations beat 50k mediocre ones for style), domains without strong teachers, preference labels where AI judges are biased, and red-teaming. Human data ops is its own discipline: written rubrics with worked examples, calibration rounds, inter-annotator agreement tracking, gold-question audits, and vendor management (typical 2026 costs: $1–10 per preference pair, $10–100+ per expert demonstration). Budget the *ops*, not just the labels.

Quality control for any demonstration set, human or synthetic: verify what's verifiable (run the code, check the math), judge-score the rest against a rubric, filter refusals/apologies/self-reference ("As an AI..."), enforce length and format sanity, and check the *mixture* — capability balance matters more than any single source (Tulu 3's central finding: mixture composition, tested by ablation, drives results).

## 3.4 Preference and verifiable data

- **Preference pairs.** Best practice is **on-policy + AI judge**: sample 2–8 responses from *your current model* per prompt, have a strong judge (or ensemble) rank them, take chosen/rejected from that ranking (UltraFeedback pattern, now standard). Off-policy pairs (two different models' outputs) train worse and inject style artifacts. Human preferences are reserved for the axes where judges are unreliable: subtle helpfulness, tone, safety judgment. Beware inherited judge biases — length, sycophancy, self-preference (Chapter 14) — they become your model's biases.
- **Verifiable tasks.** The scarce ingredients are *correct answers* and *strict checkers*. Math: canonical answers + robust equivalence checking (math-verify; exact-match graders create false negatives that poison RL). Code: real test suites, mutation-tested for coverage; beware tasks solvable by printing the expected output. Constraints: programmatic validators (word counts, JSON schemas, keyword rules). **Difficulty filter against your own model**: estimate pass rate by sampling; keep prompts with pass rate strictly between 0 and 1 (0 gives no gradient signal, 1 wastes compute — Chapter 8). Refresh this filter as the model improves.

## 3.5 Decontamination and governance

Decontamination is more urgent than in pretraining because post-training sets are small and targeted — one leaked AIME problem visibly inflates your headline number. N-gram (8–13 gram) plus embedding-similarity matching of all training data against every eval you report; do it for prompts *and* synthetic completions (teachers happily regurgitate benchmark items); log and publish match rates. Governance: PII scrubbing on user logs, license tracking per source, teacher-ToS tracking per synthetic set, and a datasheet per mixture — future-you, three mixtures later, will not remember what's in `sft_mix_v7`.

## 3.6 Decisions

Start from an open mix (Tulu 3 / Dolci) rather than from scratch; add your domain via teacher-distilled demonstrations over your real prompt distribution with verification filtering; generate preferences on-policy with an AI judge; build verifiable sets with strict checkers and pass-rate difficulty filtering; decontaminate against every eval you report; spend human budget on golden sets, judgment-heavy preferences, and red-teaming — not bulk demonstrations.
