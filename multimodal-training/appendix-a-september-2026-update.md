# Appendix A — September 2026 Update

*This series was compiled in early July 2026. This appendix records what changed between July and September 2026, plus two things the main chapters missed: **disaggregated serving for multimodal models** (A.4) and a pointer to **embodied/action models** as the next adjacent frontier (A.6). The headline change: multimodality stopped being a variant and became the default for frontier open LLMs.*

## A.1 Native multimodality became the default for frontier open LLMs

Chapter 1 framed the choice as adapter VLM versus native multimodal, with native as the frontier direction. By September 2026 the frontier open releases had largely made that choice:

- **Kimi K3** (July, 2.8T) ships with native vision.
- **DeepSeek-V4.1-Flash** (September, 552B, MIT) "natively processes images and text." It is the first DeepSeek flagship-line model where vision is part of the base rather than a separate VL model.
- **GLM-5.3-Flash** (August, 320B/18B active, MIT) ships natively multimodal.
- **Qwen3.8** released the **Qwen3.8-Flash-Next base** as the shared foundation for its Omni line (A.2).

The counterexample is also informative. **Meta's Muse Glimmer** (August 10, 30B, open weights) combines an LLM with a **dedicated perception encoder**, which is the adapter-style design of Chapter 2. For mid-size open releases, encoder + connector + LLM remains a competitive and cheaper recipe.

**What this changes for you:**

1. **"Start from a text LLM and add vision" is no longer the only entry point.** For many projects, the right base is now an *already-multimodal* open model. Your job becomes multimodal post-training (Chapter 9) or domain adaptation, not connector training (Chapters 2–4).
2. **Replay must be multimodal.** If you continue-pretrain or fine-tune a natively multimodal base on text, text-only replay protects text ability and silently erodes vision. This is catastrophic forgetting in reverse (Chapter 13; midtraining series Appendix B.7). Keep image-text data in your replay mixture, and keep multimodal evals in your forgetting harness.
3. **Licenses now govern vision capabilities too.** The multimodal bases above carry MIT, Apache, or custom licenses (cross-cutting Appendix C). Check before building on them.

## A.2 Omni models went agentic

Chapter 7.5 described omni models, which take any modality in and produce text or speech out, with Qwen3-Omni and Qwen3.5-Omni as references. Two developments:

- **Qwen3.8-Omni** (September 2026, *Towards Native Omni-Modal Agents*, arXiv:2609.25611) builds the omni model on the sparse-MoE Qwen3.8-Flash-Next base, extends context to **1M tokens**, and is built around **agentic audio-video understanding and tool use**: long-horizon planning over long audiovisual inputs, not just answering a question about a clip. A realtime variant keeps the Thinker–Talker split described in Chapter 7: a Thinker generates text, and a Talker streams speech tokens.
- **NVIDIA Nemotron 3 Nano Omni** (April 2026, missed by the main chapters) is an *open* omni model on a **hybrid Mamba-Transformer MoE backbone** (Nemotron 3 Nano 30B-A3B). The hybrid backbone is the point. Long audio and video sequences are exactly where linear-time sequence mixers pay off (pretraining series Appendix B.2). It shows that the hybrid-attention trend and the omni trend are converging.

**Implication for Chapter 7's decisions:** if you are building a voice or video *agent*, the omni base you start from should have been trained with tool use over long multimodal context. Bolting tool use onto an omni model trained only for QA repeats the Chapter 9 lesson that agentic behavior has to be trained, not prompted.

## A.3 Small multimodal models (Chapter 12)

**Gemma 4** (April 2026, Apache 2.0) is the reference for the small end. All sizes take images at variable aspect ratio and resolution. The E2B, E4B, and 12B variants take **native audio**. The smallest (E2B) targets devices with under 2 GB of available memory, and it ships with configurable thinking modes. The small-scale playbook in Chapter 12 ("start from a capable small VLM, adapt, and distill") now has an Apache-licensed omni-capable starting point that runs on phones.

## A.4 Serving multimodal models: encoder disaggregation (previously missed)

This series stops at training, and the inference series is text-centric. Neither covered the serving pattern that emerged in 2026 for VLMs and omni models:

- **The problem.** A multimodal request has three phases with different resource profiles: **encode** (the vision or audio encoder runs over pixels or waveforms, compute-heavy and embarrassingly parallel), **prefill** (the LLM runs over text plus visual tokens), and **decode**. Running all three on the same GPUs means an image-heavy burst stalls decode for every request (inference series Chapter 4's interference problem, now with a third phase).
- **The pattern.** **E/P/D disaggregation** separates the encoder into its own elastically scaled tier, ahead of the prefill/decode split. SGLang shipped it in January 2026 (the LMSYS "EPD Disaggregation" post). vLLM's Model Runner V2 added E/P/D disaggregation in August 2026 (v0.28).
- **When to use it.** Use it for high-volume VLM serving with a variable image or video load per request, such as document pipelines, video agents, and screenshot-driven computer-use agents. For low-volume or text-dominant traffic, the extra tier costs more than it saves.
- **Related:** serving engines now host diffusion image and video generators as well. SGLang added Cosmos3, LTX-2.5, and LongCat-Image in August. Teams running any-to-any systems (Chapter 5.5) can consolidate on one serving stack.

## A.5 Multimodal retrieval (cross-cutting Chapter 4)

**Gemini Embedding 2** (preview March 2026, GA late April; missed by the main chapters) embeds **text, images, video, audio, and PDFs into a single 3,072-dimensional space**, with Matryoshka truncation. Per-call limits: 8,192 text tokens, 6 images, 120 s of video, 180 s of audio, 6 PDF pages. It makes *multimodal RAG* practical without per-modality indexes. For agentic multimodal systems, pair it with a native-multimodal generator so that retrieval and generation share an understanding of the same modalities.

## A.6 The adjacent frontier: embodied reasoning

Scope note: this series ends at models that see, hear, and speak. Models that *act* in the physical world are the next step on the same machinery. **Gemini Robotics ER 2** (July 30, 2026) is an embodied-reasoning VLM with real-time video understanding, task-progress tracking, low-latency orchestration, tool integration, and multi-robot collaboration. Its data (video + grounding + temporal progress), post-training (RL on task completion), and serving (streaming, low latency) are recognizably Chapters 6, 8, 9, and 7.4 of this series. If your roadmap goes there, this series is the prerequisite, and vision-language-action (VLA) models are the specialized literature beyond it.

## A.7 Decisions

1. **Consider an already-multimodal open base** before training a connector. Frontier open models now ship with native vision.
2. **Keep multimodal data in replay and multimodal evals in the harness** whenever you adapt a multimodal base on text.
3. **For voice or video agents, start from an omni base trained for tool use over long context**, such as Qwen3.8-Omni-class models.
4. **Use Gemma 4-class models as the small or on-device multimodal starting point** (Apache 2.0, native audio at the small sizes).
5. **Adopt E/P/D disaggregation for high-volume, image-heavy serving.** Skip it for text-dominant traffic.
6. **Use a unified multimodal embedder** for multimodal RAG instead of separate per-modality indexes.

## A.8 Sources

- Qwen Team, *Qwen3.8-Omni: Towards Native Omni-Modal Agents*, 2026 — arXiv:2609.25611
- NVIDIA, *Nemotron 3 Nano Omni: Efficient and Open Multimodal Intelligence*, April 2026 — https://research.nvidia.com/labs/nemotron/files/NVIDIA-Nemotron-3-Omni-report.pdf
- DeepSeek-AI, *DeepSeek-V4.1-Flash*, 2026 — arXiv:2609.19969; Kimi Team, *Kimi K3*, 2026 — arXiv:2607.24653
- Google, Gemma 4 model card — https://ai.google.dev/gemma/docs/core/model_card_4
- Google DeepMind, *Gemini Embedding 2: A Native Multimodal Embedding Model from Gemini*, 2026 — arXiv:2605.27295
- LMSYS, *EPD Disaggregation: Elastic Encoder Scaling for Vision-Language Models in SGLang*, January 2026 — https://www.lmsys.org/blog/2026-01-12-epd/
- vLLM v0.28 release notes (E/P/D disaggregation) — https://github.com/vllm-project/vllm/releases; SGLang v0.5.18 release notes — https://github.com/sgl-project/sglang/releases
- Meta Muse Glimmer open-weights release (August 10, 2026), CNBC and Latent Space coverage
- Google DeepMind, Gemini Robotics ER 2 announcement (July 30, 2026)
