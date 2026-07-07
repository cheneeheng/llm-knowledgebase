# Chapter 8 — Forgetting and Stability

## 8.1 Conceptual picture

Every technique in this series perturbs a working model toward a new distribution, and every such perturbation induces **catastrophic forgetting** — the loss of general capability the model already had. This chapter is the shared tax collector for the whole series. Forgetting is not a bug you can eliminate; it is a conserved quantity you *budget*. The discipline is: know how much you are spending, spend it deliberately, and pay it down with the cheapest available instrument, which is almost always replay.

The mental model: the model's weights encode a distribution. Gradient descent on new data moves that distribution toward the new data and, because capacity is finite and shared, away from the old. High LR moves it faster (more learning, more forgetting); replay pulls it back toward the old distribution; the two are in tension and you are choosing the operating point.

## 8.2 Measuring forgetting — the non-negotiable harness

You cannot manage what you do not measure, and forgetting is *silent* — the training loss on your new data looks great while general capability quietly erodes. So the one rule that governs this entire series: **run a fixed general-eval suite on every checkpoint, alongside your target-domain eval, and watch both curves.**

A minimal forgetting harness:

- **General knowledge/reasoning**: MMLU (or MMLU-Pro), HellaSwag, ARC, a GSM8K-style math eval. These are your "did I break the base model" probes.
- **The target metric**: whatever you're specializing for (domain benchmark, long-context RULER, code eval).
- **A perplexity probe on held-out *general* data** (original pretraining distribution) — a fast, cheap early-warning signal that moves before the benchmark does.
- **Format/behavior spot-checks**: does it still follow instructions, still stop at EOS, still handle short context — the things a domain CPT quietly breaks.

Run it before you start (the baseline), and at every checkpoint. The concrete budget from Chapter 4, restated because it is the load-bearing number: **a well-tuned run holds general-eval drop to ≤1–2 points; >3 points is a bug** (replay too low or LR too high), not an acceptable cost of specialization.

## 8.3 Replay — the primary instrument

**Replay** = mixing a fraction of the *original general/pretraining distribution* back into your midtraining data. It is the most effective, cheapest, most universal forgetting mitigation, and under-replaying is the single most common midtraining mistake.

- **Ratio: 10–30%.** The consensus band. 20% is a good default. Empirically, 10–30% replay preserves source-language capability when adapting to a new language, and a 20%-replay CPT should show ≤1–2 point general-eval drop.
- **Source: the original pretraining mix, or a strong open proxy** (FineWeb-Edu, DCLM, RedPajama) if you don't have the original. Match the *distribution* you're trying to preserve, not just "some general text."
- **Front-load it.** Higher replay early (while the model re-stabilizes through the stability gap, Chapter 3.4), tapering later. This spends replay budget where forgetting risk is highest.
- **Replay can *help* the target, not just protect the general.** A 2026 result: replaying pretraining data during adaptation improves data efficiency on the target by up to ~2× — because it keeps the model in a well-conditioned region where it learns the new distribution more efficiently. Replay is not purely defensive; it is often Pareto-improving. This is why "add 20% replay" is close to a free lunch.

Turn replay *up* when: LR is high, the distribution shift is large, target data is scarce, or your general-eval drop exceeds budget. Turn it *down* (toward 10%) when: you need maximal target absorption, the shift is small, and general evals have margin.

## 8.4 The stability gap and LR discipline

Recapping and extending Chapter 3.4, because it is where instability lives:

- **The stability gap** — target performance dips before it climbs, and general performance drops, in the first fraction of CPT. Caused by the model destabilizing from its old optimum. Mitigate with gentle LR warmup, front-loaded replay, and (for small corpora) multiple short epochs over a subset rather than one long pass.
- **LR is the master forgetting knob.** Re-warming too high is the most common cause of both loss spikes and excess forgetting. Re-warm to a *fraction* of from-scratch peak (Chapter 3.3), decay to near-zero, and if general evals crater, halve the LR before you touch anything else.
- **Loss spikes on re-warmup** — if you see a spike when CPT starts, your warmup is too short or your LR too high. Lengthen warmup, lower peak. Persistent spikes mid-run are the same failure modes as pretraining (Chapter 12) — bad data batches, precision issues — diagnosed the same way.

## 8.5 Architectural and algorithmic anti-forgetting

Beyond replay and LR, in rough order of practical value:

- **LoRA / adapters** — the strongest structural anti-forgetting tool: the base weights are *frozen*, so general capability is protected by construction, and you can even keep multiple domain adapters and swap them. This is why LoRA is the default for domain *flavor* (Chapter 4.5). The cost is capacity: LoRA can't absorb a large-corpus capability. Use LoRA when retention matters more than absorption.
- **Parameter-expansion methods** (ADEPT-style: add new blocks/width for the new domain, keep old parameters mostly frozen) — install new capability in *new* capacity so old capacity is undisturbed. More machinery; used for serious continual-learning pipelines where forgetting is unacceptable.
- **Selective/decoupled tuning** — freeze the most general-knowledge-bearing layers, train the rest. Cheaper than full CPT, more retentive; a middle ground.
- **Context-aware / regularization methods** (EWC-style penalties, context-aware CPT) — penalize moving weights the base model relied on. Classic continual-learning tools; in LLM practice, replay usually dominates them on a cost/benefit basis, so reach for these only when replay alone can't hold the budget.

The 2026 consensus: **replay first, LoRA if retention is paramount, expansion/regularization only for hard continual-learning cases.** Most teams never need more than replay + LR discipline.

## 8.6 Decisions

1. **Run a fixed general-eval + target-eval harness on every checkpoint**, from a pre-run baseline. This is not optional.
2. **Hold general-eval drop to ≤2 points**; treat >3 as a bug in replay or LR.
3. **Replay 10–30% of the original distribution** (default 20%), front-loaded; it protects the general model *and* often improves target efficiency.
4. **Re-warm LR to a fraction of peak, decay to zero**; LR is your master forgetting knob — halve it before anything else when evals crater.
5. **Expect the stability gap**; don't judge early checkpoints, mitigate with gentle warmup and multiple short epochs on small corpora.
6. **Use LoRA when retention outweighs absorption**; reach for expansion/regularization only for hard continual-learning needs.

Next: frameworks and infrastructure — what carries over from the pretraining stack and what changes for these shorter, more numerous, adaptation-flavored runs.
