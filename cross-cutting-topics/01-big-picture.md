# Chapter 1 — The Big Picture

## 1.1 What "cross-cutting" means

The other series in this knowledge base are organized by *stage* — pretraining, midtraining, post-training, inference, multimodal. Each is a phase of work with a start and an end. This series is organized by *concern* — a property or decision that isn't a phase, that instead runs *through* multiple phases and can't be fully resolved inside any one of them. A concern is cross-cutting when getting it wrong in one stage is caused by a decision in another, or when it constrains several stages at once.

Five examples, each a chapter:

- **Model selection and licensing** is decided before you train anything, but it governs what you can deploy, distill, and monetize. A license clause discovered at launch invalidates months of training.
- **Small/on-device models** are a *deployment target* that reaches back and rewrites your *training* plan (you distill instead of train-from-scratch, you quantize aggressively, you specialize hard).
- **Embeddings and retrieval** are models you train *alongside* your LLM, so RAG works — a parallel training track most LLM guides ignore.
- **Interpretability** touches training (what did it learn), safety (is it behaving), and serving (steer it live).
- **Security and compliance** are properties of the *deployed system* that constrain the *data you collect* and the *data you train on*, upstream.

None of these fits in a stage box. All of them sink projects when handled late.

## 1.2 The constraints that span stages

The deepest cross-cutting truth is that the hardest LLM decisions are about **constraints**, not capabilities, and constraints span stages by nature. Four families:

- **Legal constraints** — licenses (of the base model, of the training data), copyright, and regulation (GDPR, the EU AI Act). These say what you *may* do, independent of what you *can* do. They're decided by lawyers and regulators, not loss curves, and they invalidate technical work retroactively (Chapters 2, 7).
- **Economic constraints** — the compute budget, the cost per token, the build-vs-buy math. These say what you can *afford*, and they cut across training (millions for a run), serving (two-thirds of AI compute), and the org (headcount) (Chapters 2, 8, and the inference series Ch 12).
- **Security constraints** — what an adversary can make your deployed system do, which constrains what data you let it touch and what actions you let it take, reaching back into training (safety) and forward into serving (guardrails) (Chapter 6).
- **Organizational constraints** — who does the work, how the team is structured, how compute is allocated, how the improvement loop runs. The human system that produces the technical system (Chapter 8).

The pattern: **capabilities are produced within stages; constraints are enforced across them.** A team that masters every stage but ignores the constraints ships a model it can't legally deploy, can't afford to serve, can't secure, and can't maintain.

## 1.3 Constraints before capabilities

The governing principle of this series, stated as a rule: **establish the constraints first, then build the best model within them.** The natural instinct — build the best model, handle the rest before launch — is backwards, because each constraint discovered late invalidates completed work:

- Discover the base model's license forbids your commercial use *after* you fine-tuned it → the fine-tune is worthless (Chapter 2).
- Discover your training data violates a copyright/GDPR rule *after* training → you may have to retrain or can't deploy (Chapter 7).
- Discover a prompt-injection hole *after* wiring the model to tools → you redesign the agent (Chapter 6).
- Discover the serving cost exceeds revenue *after* building the product → the product is unviable (inference series Ch 12).

Each is a real, common failure, and each was preventable by asking the constraint question *first*: What can we legally use? What must we protect? What can we afford? What can an attacker do? Who maintains this? The constraints are cheap to establish early and catastrophic to discover late — the same shape as the pretraining series' "cheap decisions with catastrophic late costs," generalized to the whole lifecycle.

## 1.4 Core mental models

Four ideas recur across the series.

- **The hybrid default.** For almost every "which model / build or buy" question, the 2026 answer is *not one thing* but a *mix routed by request*: a small self-hosted model for private/high-volume work, an API for the hardest queries, a specialized model for the vertical. Purity (all-open or all-API) is usually wrong; the winning architecture is a router over a portfolio (Chapters 2, 3).

- **Small, specialized, and distilled beats big and general — for a task.** The 2026 SLM story: a well-distilled 2–4B model can beat a 671B general model *on a specific task*. The frontier general model is the *teacher* and the *fallback*, not necessarily the *deployment*. Right-sizing the model to the task is the biggest cost and latency lever there is (Chapter 3).

- **Everything is a proxy, and every proxy is attacked.** The post-training series' lesson generalizes: your eval is a proxy for quality (and gets gamed), your guardrail is a proxy for safety (and gets bypassed), your compliance checklist is a proxy for legality (and misses edge cases). Cross-cutting work is largely about keeping proxies honest — red-teaming the guardrail, auditing the eval, stress-testing the compliance story (Chapters 5, 6, 8).

- **The loop matters more than the launch.** A deployed LLM system is never done; it's a flywheel — collect production data, evaluate, improve, redeploy. The teams that win optimize the *loop* (how fast they can measure a problem and ship a fix), not just the initial model. This is the organizational through-line (Chapter 8).

## 1.5 How to use this series

Unlike the stage series, you needn't read this one front-to-back. But:

- **Read Chapters 1–2 first, always.** The cross-cutting mental models and the selection/licensing decision frame everything and are the constraints most often discovered too late.
- **Then read by need:** building for the edge → Chapter 3; doing RAG → Chapter 4; debugging model behavior → Chapter 5; going to production → Chapters 6–7; scaling a team → Chapter 8.
- **Cross-reference the stage series** where a topic overlaps — this series takes the between-stage view and points to the deep treatment (safety → post-training Ch 11; serving cost → inference Ch 12; distillation mechanics → post-training Ch 10).

Start with Chapter 2: whether you build or buy, and under what license, is the constraint that shapes every other decision.
