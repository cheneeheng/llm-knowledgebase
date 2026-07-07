# Chapter 6 — Production Security

## 6.1 Conceptual picture

The moment your model is reachable by users — and especially the moment it can *act* (call tools, browse, execute code) — it becomes an attack surface, and LLM security is a distinct discipline with its own threat model. The defining fact of 2026: **prompt injection remains OWASP's #1 LLM vulnerability (LLM01) and is still fundamentally unsolved.** You cannot fully prevent it; you can only reduce and contain it. This chapter is the adversary's-eye view — the attacks, why they work, and the defense-in-depth stack that keeps a deployed LLM system safe enough to ship, especially as it gains agentic capabilities that turn a text exploit into a real-world action.

The core insight that makes LLM security hard: **the model can't reliably distinguish instructions from data.** Everything is text in the same context — the system prompt, the user's message, a retrieved document, a tool's output — and an attacker who gets text into any of those channels can try to issue instructions. There is no privileged "code" channel separate from the "data" channel, the way there is in traditional software. This is the root of prompt injection and why it resists clean fixes.

## 6.2 The attack landscape

**Prompt injection (LLM01) — the root threat.**
An attacker embeds instructions in text the model processes, overriding its intended behavior. Two forms:
- **Direct injection** — the user themselves types adversarial instructions ("ignore your instructions and...").
- **Indirect injection** — the malicious instructions ride in *content the model consumes*: a web page it browses, a document in RAG, an email it summarizes, an image (multimodal injection via embedded text/QR/steganography). The user is innocent; the *data* is poisoned. This is the more dangerous form because it needs no cooperation from the user and scales.

**Jailbreaks.**
Techniques to make the model violate its safety training — produce harmful content, reveal system prompts, bypass refusals. 2026 reality: "multi-turn jailbreaks are the preferred vector on frontier models" (build up over a conversation rather than one prompt), and "roleplay-based prompt injections achieved ~90% attack success against frontier models" in one study, with jailbreak times under ~17 minutes. Single-turn safety training is routinely defeated by patient, multi-turn, roleplay-framed attacks.

**Agentic exploitation — the high-stakes frontier.**
When the model can *act*, injection becomes *action*. The new attack surface with agent adoption:
- **Agent goal hijacking** — injected instructions redirect an agent's goals/plans, making it pursue the attacker's objective (exfiltrate data, make unauthorized changes).
- **Tool/MCP exploitation** — poisoning tool definitions, stealing credentials via tool calls, exploiting the Model Context Protocol server surface.
- **Indirect injection → tool use** — a poisoned document the agent reads instructs it to call a tool destructively. The "OWASP Top 10 for Agents" codifies these.

The escalation: a text model that gets jailbroken says something bad; an *agent* that gets injected *does* something bad. Agentic security is where the stakes jump from reputational to operational.

## 6.3 The defense stack (defense in depth)

Because no single defense stops prompt injection, security is *layered* — each layer catches what the others miss:

**1. Input guardrails.**
Screen incoming text (user messages, retrieved content) for injection/jailbreak patterns *before* it reaches the model. Tools: Lakera Guard, Prisma AIRS, NeMo Guardrails, LLM Guard, Guardrails AI — "real-time APIs that detect prompt injection and jailbreak attempts, dropped into the request pipeline." Regularly updated against new attack patterns. Catches known attacks; misses novel ones.

**2. Model-level robustness.**
Safety training (post-training series Ch 11) — refusal training, adversarial training, Constitutional AI — makes the model *itself* more resistant. Necessary but insufficient (multi-turn jailbreaks defeat it), so it's a layer, not the answer.

**3. Output validation.**
Screen the model's output *before acting on it* — "every LLM output should pass through validators before acting on it, especially for agents that execute tool calls." Check outputs for policy violations, PII leakage, malformed/malicious tool calls. Interpretability probes (Chapter 5) can feed this — read "about to do something off-policy" off activations.

**4. Architectural containment (the most important for agents).**
Assume the model *will* be compromised and limit the blast radius:
- **Least privilege** — the agent's tools/permissions are the minimum needed; a hijacked agent can only do what its tools allow. Don't give an agent broad credentials.
- **Human-in-the-loop for consequential actions** — irreversible/high-stakes tool calls (send money, delete data, email externally) require confirmation.
- **Sandboxing** — code execution and tool use run in isolated, revocable environments (the post-training/agentic-infra sandboxing story, Ch 9).
- **Separation of trusted and untrusted content** — treat retrieved/browsed content as untrusted data, never as instructions; structure prompts so injected content can't easily escalate.

The principle: **you can't prevent injection, so contain it.** The defense that matters most for agents isn't a better filter — it's ensuring a compromised agent can't do much damage.

## 6.4 Red-teaming

You cannot secure what you haven't attacked. In 2026, "red-teaming is no longer optional — it's part of how production AI gets built." The practice:

- **Adversarial testing before and after launch** — systematically attack your own system with injection, jailbreak, and agentic-exploit techniques; measure attack success rates; fix and re-test.
- **Automated red-teaming tools** — DeepTeam, PI-Hunter (automated injection discovery/localization), IPI-proxy (indirect-injection testing for browsing agents), AgentRedBench (agentic red-teaming over integrations) — scale attack coverage beyond manual testing.
- **Continuous, not one-time** — new attacks emerge constantly; red-team on a schedule and against each new capability (especially each new tool you give an agent).
- **Cover the agentic surface** — for agents, red-team the *actions*, not just the outputs: can an injected document make the agent misuse a tool?

Treat red-teaming like the eval discipline from every series: an adversary-honesty check on your safety proxy (Chapter 1.4). A guardrail you haven't red-teamed is an untested assumption.

## 6.5 Decisions

1. **Assume prompt injection cannot be prevented, only contained** — it's OWASP #1 and unsolved; design for compromise, don't rely on stopping it.
2. **Layer defenses** — input guardrails + model robustness + output validation + architectural containment; no single layer suffices.
3. **For agents, prioritize containment** — least-privilege tools, human-in-the-loop for consequential actions, sandboxing, and treating retrieved/browsed content as untrusted data. A compromised agent's blast radius is the thing you actually control.
4. **Validate every output before acting on it** — especially tool calls; feed interpretability probes (Chapter 5) into the validator.
5. **Red-team continuously** — automated + manual, before and after launch, against each new capability and tool; treat it as the honesty-check on your security proxy.

Next: privacy, compliance, and governance — the legal constraints that reach all the way back to your training data.
