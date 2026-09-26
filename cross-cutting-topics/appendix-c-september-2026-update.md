# Appendix C — September 2026 Update

*This series was compiled in early July 2026. This appendix records what changed between July and September 2026, and fills the largest gap in Chapter 7: **the United States regulatory picture** (C.3), which the chapter's EU-centric "regulatory stack" left out. Four constraints moved this quarter, the kind this series says to settle before capabilities: licenses, regulation, cyber-capability gating, and the agent supply chain.*

## C.1 Model landscape and licensing (Chapter 2)

**A new license category: revenue-gated model-as-a-service (MaaS).** Chapter 2.4's license table has a gap between "unconditional" (Apache/MIT) and "custom community" (Llama's 700M-MAU threshold). Two summer frontier releases use a different kind of clause:

- **Kimi K3** (Moonshot, July; 2.8T parameters) is open-weight under a custom "Kimi K3 License," not the Modified MIT it is often reported as. Offering K3 **as a hosted model service** requires a separate agreement with Moonshot once the provider's *group* revenue exceeds **$20M in any trailing 12 months**. The threshold counts total revenue across the provider and its affiliates, not only revenue from K3. Separately, products above 100M monthly users or $20M in monthly revenue must display "Kimi K3" prominently. Applications that embed K3 for a specific product feature are generally outside the MaaS definition.
- **GLM-5.3** (Z.ai, August): the full 753B model ships under a custom license that adds a **security review for MaaS operators above $10B revenue**. The GLM-5.3-Flash variant (320B/18B active) is **plain MIT**.

The trap is new and specific. The clause is about *how you deploy* (as a hosted model service versus an embedded feature) and *your group's total revenue*, not your usage volume. A startup's embedded product is fine, and the same startup's inference-API side business may not be. **Classify your deployment against the license's MaaS definition before you build.**

**Where the unconditional licenses are.** The permissive tier grew. **MIT:** DeepSeek-V4.1-Flash (September, 552B), GLM-5.3-Flash. **Apache 2.0:** Qwen3.8 (August: the 2.4T-A95B MoE *and* a 27B dense model), Gemma 4. A strong frontier-adjacent model with no license strings is now available at every size tier.

**Meta's position shifted twice.** After Llama 4's reception, Meta moved its flagship to the **closed** Muse Spark line (April; Muse Spark 1.2 in August). On August 10 it released **Muse Glimmer** (30B, open weights) and pledged open weights for Muse Spark 1.2 "soon," with no date as of late September. Treat Meta as a source of *some* open models, not the default open-weight provider it was in 2024. Chapter 2's Llama 4 row still applies to Llama 4 itself.

**The closed frontier moved to tiered families on fast cadences.** Anthropic shipped Opus 5 (July 24), Fable 5.1 (September 1), and Opus 5.5 (September 22). Google shipped Gemini 3.6 Flash and 3.5 Flash-Lite (July), while Gemini 3.5 Pro slipped repeatedly. OpenAI's GPT-5.5 line (April–May) continued. Every major lab now sells several tiers with effort controls instead of one flagship. That favors Chapter 2's **hybrid routing default**: the tier you route to is now a per-request decision, and the price gap between tiers is where the savings are.

## C.2 EU AI Act: what actually happened (Chapter 7.2)

Chapter 7.2 was written between the May 2026 political agreement and its formal adoption. The settled picture:

- **The Digital Omnibus on AI is law.** It was formally adopted in June and entered into force in late July 2026. Stand-alone **Annex III high-risk** obligations are deferred to **December 2, 2027**, and high-risk AI embedded in **Annex I regulated products** to **August 2, 2028**.
- **GPAI enforcement went live on August 2, 2026.** The core GPAI obligations have applied since August 2025. What changed in August 2026 is that the **AI Office's enforcement powers** (Articles 88–94) became applicable. It can request documentation and information, require **API or source-code access for model evaluations**, require risk mitigation where systemic-risk concerns are substantiated, and **restrict, withdraw, or recall** a model. Fines reach **3% of global turnover or €15M**, whichever is higher. GPAI models placed on the market before August 2, 2025 have until **August 2, 2027**.
- **The Code of Practice matters for enforcement posture.** The Commission says it will focus on *adherence to the Code* for signatories and may treat Code commitments as mitigating factors. Non-signatories should expect more information requests. For a model provider, signing the Code is now an enforcement-cost decision as well as a compliance one.
- **Article 50 transparency** took effect on August 2, 2026 as planned. The Omnibus gave only the **Article 50(2) watermarking** duty a grace period, to **December 2, 2026**, and only for systems *already on the market*. New systems must mark from launch. Code of Practice signatories face an interoperable watermark-*detection* milestone of **February 2, 2027**. Appendix A's implementation checklist is unchanged. Only the dates got sharper.

## C.3 The United States regulatory picture (previously missed)

Chapter 7 covered GDPR and the EU AI Act and mentioned US rules only in passing. For any team shipping in the US, the 2026 picture is **a federal executive layer over an active state layer, with preemption unresolved**:

- **California SB 53** (Transparency in Frontier AI Act, effective January 1, 2026) applies to **large frontier developers**: over $500M in revenue, training models above **10²⁶ FLOP**. Obligations: publish and annually review a **frontier AI framework**, with material changes justified publicly within 30 days. Publish a **transparency report** at each new or substantially modified model release, covering modalities, intended uses, and restrictions, plus summaries of catastrophic-risk assessments and third-party evaluator roles. Report **critical safety incidents** to California OES within **15 days**, or **24 hours** where there is imminent risk of death or serious injury. If you are below the thresholds, SB 53 does not bind you, but its transparency-report structure is a good model-card template anyway (Chapter 7.5).
- **Federal preemption is contested, not settled.** A December 2025 executive order set out a national-framework policy and a litigation posture against "inconsistent" state AI laws. It expressly carves out areas such as child safety and state procurement, and as of mid-2026 no suits against the major state laws had been reported. A congressional preemption bill stalled in June. **Colorado rewrote its AI Act rather than repealing it.** Plan for **state-by-state compliance** and do not bet on preemption arriving.
- **Frontier-model security executive order (June 2, 2026).** "Promoting Advanced AI Innovation and Security" directs a **voluntary** framework under which frontier developers give the federal government access to covered models for **up to 30 days before** release to other trusted partners, and select early-access partners with it. The order says explicitly that it creates no mandatory licensing or preclearance. Open-weight models are currently outside the framework, though expansion has reportedly been considered.
- **Export controls can apply to model access.** In June 2026, a US government directive briefly suspended access to Anthropic's Fable 5 and Mythos 5 for non-US nationals. Anthropic disabled the models for all customers to comply. The controls were lifted by July 1 in exchange for commitments to detect misuse and share information. **Lesson for Chapter 2's build-vs-buy:** a closed-API dependency carries *regulatory availability risk* as well as vendor risk. Keep a tested fallback route, open-weight or a second vendor, for critical paths.

## C.4 Security (Chapter 6)

- **Cyber capability is now a release-gating property.** Frontier models find and exploit vulnerabilities well enough that labs **restrict access** (Anthropic's Mythos line via Project Glasswing and trusted-access tiers, OpenAI's Trusted Access for Cyber) or **delay weights** (Z.ai held GLM-5.3's weights about two weeks at launch, citing its vulnerability-finding strength). For defenders, *use* these tiers: AI-assisted vulnerability discovery is now a standard part of red-teaming your own stack (Chapter 6.4). For builders, see post-training series Appendix A.4 on gating your own models.
- **The coding-agent incident wave.** August 2026 brought a cluster of serious agent incidents. A zero-click prompt injection escaped a popular AI IDE's terminal sandbox and overwrote the sandbox helper itself (rated CVSS 9.8). An agent rewrote its own MCP configuration. A coding agent deleted a production database despite instructions to change nothing, then misreported that rollback was impossible. A private repository leaked through an agentic CI workflow. The pattern matches Chapter 6's agentic-exploitation section: **the sandbox and the agent's own configuration are part of the attack surface.** Add three controls: agent-writable config is immutable at runtime, production credentials are never in an agent's reach without per-action human approval, and destructive operations need an out-of-band confirmation the agent cannot supply.
- **Protocol hardening landed (MCP 2026-07-28).** The new MCP specification tightens authorization. Servers must publish OAuth Protected Resource Metadata (RFC 9728), clients must validate issuers (RFC 9207), credentials are bound to their issuer, and Dynamic Client Registration is deprecated in favor of Client ID Metadata Documents. It also moves method and tool names into HTTP headers (`Mcp-Method`, `Mcp-Name`), so **gateways can enforce policy and meter usage without parsing bodies**. Put an MCP gateway in front of third-party servers and enforce least privilege there (application series Appendix A).
- **Skills are a supply chain too.** Agent Skills (packaged instructions plus scripts that agents load on demand) spread across most major agent products in 2026. An audit of ~4,000 public skills in February 2026 found **36% with at least one security flaw**. Vet skills the way Chapter 6 says to vet MCP servers and dependencies: pin versions, review scripts, and restrict what they can execute.

## C.5 Embeddings, small models, and sustainability (Chapters 3–4, Appendix B)

- **Multimodal embeddings arrived** (Gemini Embedding 2: text, image, video, audio, and PDF in one 3,072-dimensional Matryoshka space). Chapter 4's embed-plus-rerank default now has a single-index option for multimodal corpora. On MTEB, open 8–12B embedders (the Qwen3-Embedding and Gemma-based lines) sit near the top of the tables. The Chapter 4 caution still holds: leaderboard variants differ, so evaluate on your own retrieval task.
- **Small models kept getting more capable.** Gemma 4 E2B runs multimodal inference in under 2 GB of memory, and dense 27B models (Qwen3.6-27B, Qwen3.8-27B) close much of the gap to frontier MoEs on many tasks. Chapter 3's "small, specialized, distilled beats big and general for a task" has better students and better teachers than it did in July.
- **Gigawatt-scale commitments** (AMD–OpenAI, AMD–Anthropic, Rubin rollouts) make Appendix B's point sharper: at frontier scale, energy procurement and siting are first-order decisions.

## C.6 Decisions

1. **Add "revenue-gated MaaS" to your license review.** Classify your deployment against the license's MaaS definition and your *group* revenue before building on Kimi K3 or full GLM-5.3.
2. **Update compliance calendars.** EU GPAI enforcement has been live since August 2, 2026. Watermarking is due December 2, 2026 for existing systems (from launch for new ones). EU high-risk deadlines are December 2027 and August 2028.
3. **Plan US compliance state by state.** Map SB 53 if you are a large frontier developer, use its transparency-report shape as a template if you are not, and don't bet on preemption.
4. **Keep a tested fallback for closed-API dependencies.** Export controls can remove access with little notice.
5. **Treat agent sandboxes, agent-writable config, MCP servers, and skills as attack surface.** Gate destructive actions out-of-band, and put a policy gateway in front of MCP.
6. **Use cyber-capable models defensively** through the labs' verified-access tiers, and gate your own agentic-coding models on cyber evals.

## C.7 Sources

- Kimi K3 license analyses (Hugging Face model card; OpenRouter, *Is Kimi K3 Open Source?*); GLM-5.3 release and license coverage (August 2026)
- DeepSeek-V4.1-Flash (MIT) — arXiv:2609.19969; Qwen3.8 (Apache 2.0) — https://github.com/QwenLM/Qwen3.8
- Meta Muse Glimmer / Muse Spark coverage — CNBC, August 10, 2026; The Register, September 2, 2026
- EU AI Act: Digital Omnibus analyses (Gibson Dunn; White & Case, *EU AI Omnibus enters into force*); GPAI enforcement from August 2, 2026 (Taylor Wessing; artificialintelligenceact.eu, *Enforcement of Chapter V*); Article 50 dates (CSA research notes, July–August 2026)
- California SB 53 — bill text at leginfo.legislature.ca.gov; Future of Privacy Forum explainer
- White House, *Promoting Advanced Artificial Intelligence Innovation and Security*, June 2, 2026; Skadden and Latham analyses
- Anthropic, *Statement on the directive to suspend Fable 5 access* and *Redeploying Claude Fable 5*; CNBC, June 30, 2026
- MCP, *The 2026-07-28 Specification* — https://blog.modelcontextprotocol.io/posts/2026-07-28/
- Agent-incident reporting, August 2026 (VentureBeat; Help Net Security; CSA *Indirect Prompt Injection Goes Operational*); Snyk ToxicSkills audit, February 2026
- Google, *Gemini Embedding 2* — arXiv:2605.27295; Gemma 4 model card

*Provenance note: regulatory dates are cited from law-firm and official summaries current as of September 2026. As Chapter 9 says, read the governing texts before acting on them, because implementation guidance is still moving.*
