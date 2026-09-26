# Appendix A — September 2026 Update

*This series was compiled in early July 2026. This appendix records what changed between July and September 2026, plus one topic the main chapters under-covered: **reasoning effort as a trained, controllable behavior** (A.2). The main pipeline is unchanged: SFT → preference → RL, with verifiers and rubric rewards. What moved is how the frontier organizes the RL stage and what it now treats as a release gate.*

## A.1 The frontier recipe, as published: Kimi K3

The Kimi K3 technical report (July 2026) is the most detailed public description of a frontier agentic post-training pipeline so far. Read it alongside Chapters 7–10. The structure:

1. **SFT cold start** establishes baseline agent capability (Chapter 9's trajectory-SFT step).
2. **Specialist RL experts.** Nine expert policies are trained as a 3×3 grid: domain (general, general agents, coding agents) × reasoning effort (low, high, max). Each is trained in white-box agentic environments: live terminals, codebases, web browsers, and simulated workplace tools.
3. **Consolidation** of the specialists into the single shipped model (the report gives the exact mechanism). This is the Chapter 10 pattern of distilling or merging specialists into one generalist, now used as the primary path rather than a cleanup step.

The engineering details that make it work map onto chapters you have already read:

- **Partial rollouts with per-token staleness regularization.** Long agent trajectories are generated across several policy versions. Instead of discarding stale tokens or ignoring the staleness, the loss regularizes per token by how off-policy each one is. This is Chapter 7's staleness discussion turned into an explicit mechanism.
- **An agentic generative reward model under a mandatory rubric protocol** scores non-verifiable tasks. This is Chapter 6.3's rubric-reward approach, with the rubric step *required* rather than optional. The judge must write the rubric before it scores. That structure is the defense against judge drift and reward hacking.
- **Per-problem token budgets** teach each effort level its compute envelope (A.2).
- **Sandbox infrastructure as a first-class system.** The report describes a sandbox service (AgentEnv) with fast **snapshot, restore, and fork**, plus million-token agentic RL with *persistent* rollout and sandbox state. Forking a sandbox mid-trajectory lets you branch several continuations from one expensive prefix. Chapter 12's "environment fleet" is now as much of an engineering project as the trainer.

GLM-5.3 (Z.ai, August 2026) points the same way from a different lab. It kept the GLM-5.2 base and got its gains from **scaling post-training alone**: longer and richer agent environments, harder tasks, more complete trajectories, and stronger verification. On Z.ai's own evals, Terminal-Bench 3.0 went from 4.6% to 28.3%. The lesson for Chapter 2's scoping decision: at the frontier, a new generation often means a new post-training run on the same base, not a new base.

## A.2 Reasoning effort as a trained behavior (previously under-covered)

Chapter 8 covers thinking models and length dynamics. It did not treat **effort control** as a training target, and by mid-2026 it is one:

- **The product surface.** Every major line now exposes effort as a parameter. Anthropic's Opus 5 ships a five-level effort setting. Qwen3.8 exposes `reasoning_effort` and a `preserve_thinking` option that keeps thinking across turns. Kimi K3 trains low, high, and max effort levels explicitly.
- **How it is trained.** Condition on the effort level (a system-prompt tag or control token), and give each level a **token budget** during RL. Enforce the budget with length penalties or truncation beyond it, so the policy learns what quality it can reach within each envelope. K3 trains separate experts per effort level and then consolidates them. A single policy with budget-conditioned rewards is the cheaper alternative.
- **What to measure.** Per level, track quality versus tokens used. Healthy effort control gives a monotone curve: more effort gives more quality at more cost, with clearly separated levels. Failure modes to watch: (a) *level collapse*, where all levels converge to the same length; (b) *low-effort brittleness*, where the model truncates reasoning mid-thought instead of planning for a short budget; (c) *budget ignorance* on agentic tasks, where tool calls and observations are not counted toward the budget.
- **Why it matters downstream.** Effort control is how serving teams trade latency and cost against quality per request (inference series Chapter 8). A model without reliable effort levels forces the application to pick a single operating point.

**Practical call:** if you ship a reasoning model to others, train at least two effort levels with explicit budgets, and report the quality-per-token curve in your model card.

## A.3 Research that sharpens the chapters

- **The coverage principle, made concrete** (*Demystifying RL Post-Training of Language Models*, arXiv:2608.24949, August 2026). In controlled settings, how easily RL can learn a response depends strongly on how likely the **base model** already was to sample it. Behaviors with negligible initial probability are hard to discover under sparse rewards. The study also finds that the effect of "spurious rewards" depends on the prompt distribution. Implications for Chapters 2 and 8: measure the base model's pass@k on your target tasks *before* committing to RL. If pass@k is near zero, invest first in midtraining or SFT coverage, or add denser (rubric or process) rewards. Also be skeptical of RL gains from a narrow prompt set.
- **Teacher signal inside RL** (*Distilled Reinforcement Learning for LLM Post-training*, arXiv:2607.17247, July 2026) folds teacher supervision into the RL objective for fine-grained guidance. This continues the 2025–2026 convergence between on-policy distillation and RL (Chapter 10). The practical question is whether a frontier teacher is available for your domain. If one is, a hybrid objective beats pure sparse-reward RL on sample efficiency.
- **Rubric rewards consolidated.** *Rubrics as Rewards* (ICLR 2026), pairwise adaptive-rubric systems, alternating rubric-generator/judge training (ICML 2026), and rubric-curriculum RL (ICML 2026) together make Chapter 6.3's rubric approach the standard toolkit for non-verifiable RL. Frontier practice (K3's mandatory-rubric GenRM) now matches it.

## A.4 Safety: cyber capability is now a release gate

Chapter 11.4 lists "dual-use capability evals (bio/cyber uplift)" as one item on a release checklist. In summer 2026, cyber capability became the item that actually *changed releases*:

- **Z.ai withheld GLM-5.3's weights at launch** (August 14) and linked the delay partly to the model's strong vulnerability-finding performance. It was the first time the GLM family had done so. The weights followed about two weeks later. The Flash variant came under MIT, and the full model came under a custom license with a security-review clause for large model-as-a-service operators.
- **Anthropic restricted its Mythos-class models** to vetted defensive-security partners (Project Glasswing, from April 2026). It shipped a safeguarded general-availability variant (Fable 5, June 2026) alongside restricted-access versions with reduced cyber safeguards for verified defenders. A June 2026 US government directive briefly suspended access for non-US nationals. It was lifted by July 1 on commitments to detect and report misuse.
- **OpenAI launched a "Trusted Access for Cyber" program** for vetted defensive users. Tiered access by verified identity is becoming a cross-lab pattern.

**What this means for a post-training team:**

1. **Run cyber-uplift evals on every candidate that scores well on agentic coding.** Strong SWE and terminal agents *are* strong vulnerability finders, because the capabilities are the same. If you RL on coding agents (Chapter 9), you are training this capability whether you target it or not.
2. **Plan a tiered release.** A safeguarded general model plus a verified-access variant is now an established pattern. Post-training produces *both*: the safeguard training (Chapter 11.3) is what separates them.
3. **For open weights, the gate comes before publication.** Safeguards trained into open weights can be fine-tuned away. The GLM-5.3 delay shows the evaluation happening before release, which is where it belongs.

## A.5 Tooling and benchmark updates

- **RL-serving integration kept tightening (Chapters 12–13).** vLLM added a peer-to-peer sharded weight-sync backend for RL trainers, per-request speculative-decoding metrics, and Mamba prefix caching for hybrid models (versions 0.28–0.30, August–September). SGLang added sampling masks for RL rollouts (v0.5.20). If you run hybrid-architecture policies, check that your rollout engine's prefix caching supports them. Without it, rollout throughput on long agentic prefixes collapses.
- **Terminal-Bench 3.0** (August 24, 2026) replaces Terminal-Bench 2.x as the agentic yardstick of Chapter 9 and Chapter 14. It has 74 tasks across 7 domains, GPU-enabled nodes, multi-container topologies, live microservices, and strict if-and-only-if grading. It is versioned like software, with CI and result migrations. Top scores at launch were in the 30–40% range, so it has headroom that SWE-bench Verified no longer has. Cross-lab comparisons need the same benchmark *version*.

## A.6 Decisions

1. **Treat agentic post-training as specialists plus consolidation** when you have the compute, and invest in sandbox snapshot/fork infrastructure early.
2. **Make rubrics mandatory in your generative RM**: the judge writes the rubric, then scores against it.
3. **Train effort levels explicitly**, with token budgets, and publish the quality-per-token curve.
4. **Check the base model's pass@k before RL.** Near-zero coverage calls for midtraining or SFT first, or denser rewards.
5. **Gate agentic-coding models on cyber-uplift evals**, plan tiered access if they are strong, and finish evaluation *before* publishing open weights.
6. **Move agentic evaluation to Terminal-Bench 3.0**, and pin the version.

## A.7 Sources

- Kimi Team, *Kimi K3: Open Frontier Intelligence*, 2026 — arXiv:2607.24653
- *Demystifying Reinforcement Learning Post-Training of Language Models*, 2026 — arXiv:2608.24949
- *Distilled Reinforcement Learning for LLM Post-training*, 2026 — arXiv:2607.17247
- *Rubrics as Rewards: Reinforcement Learning Beyond Verifiable Domains* (ICLR 2026) — arXiv:2507.17746; *Open Rubric System* — arXiv:2602.14069; *Alternating RL for Rubric-Based Reward Modeling* (ICML 2026) — arXiv:2602.01511
- Z.ai, GLM-5.3 release (August 14, 2026) and open-weights release (August 26–28, 2026), as reported by release coverage
- Anthropic, *Claude Mythos*, https://www.anthropic.com/claude/mythos; *Redeploying Claude Fable 5*, https://www.anthropic.com/news/redeploying-fable-5
- OpenAI, *Introducing Trusted Access for Cyber* — https://openai.com/index/trusted-access-for-cyber/
- Terminal-Bench 3.0 announcement — https://www.tbench.ai/news/terminal-bench-3-0
- vLLM releases — https://github.com/vllm-project/vllm/releases; SGLang releases — https://github.com/sgl-project/sglang/releases
