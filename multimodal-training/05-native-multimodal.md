# Chapter 5 — Native Multimodal Models

## 5.1 Conceptual picture

Native (early-fusion) multimodal models abandon the "bolt an encoder onto a text LLM" composition and instead train a single model over all modalities from the start, giving vision, audio, and text "equal architectural standing" in one shared representation space. Where the adapter VLM is text-centric — the LLM is the brain, other modalities are translated into its language — a native model has no privileged modality; all are first-class citizens fused early. This is more expensive and less modular, but it unlocks two things adapters struggle with: **genuine cross-modal reasoning** (the model reasons *in* a joint space rather than translating everything to text) and **any-to-any generation** (text in → image or speech out, from one model). The frontier is moving here; most teams still shouldn't start here.

## 5.2 Why go native

The motivation is diminishing returns on adapters (the roadmap literature is explicit):

- **Adapter bottlenecks limit cross-modal reasoning** — the connector and frozen encoder are a narrow pipe; fine-grained visual/audio detail gets compressed before the LLM sees it, and the LLM can only reason about what survived the pipe.
- **Language-centric architectures suppress non-text understanding** — treating vision as "text to be translated" caps how deeply the model integrates it.
- **Any-to-any generation needs it** — an adapter VLM outputs text; generating images or speech requires the model to *produce* non-text tokens, which means non-text must be first-class in the output space, not just the input. Native models (Emu, Janus, Qwen-Omni's talker) do this.
- **At scale, native wins and doesn't cost per-modality quality** — Qwen3-Omni's headline result: joint multimodal training achieves *parity across all modalities with no per-modality degradation* while improving cross-modal capability. The feared "jack of all trades, master of none" didn't materialize when done well.

The cost, restated: full joint optimization (expensive), less modularity (can't swap an encoder), and harder training dynamics (balancing modalities). You go native when you need generation or frontier cross-modal reasoning and have the compute; otherwise adapter (Chapters 2–4) is the right, cheaper choice.

## 5.3 Tokenizing non-text modalities

The central design question native models must answer that adapters dodge: how do non-text modalities enter (and leave) the shared token stream? Two competing approaches:

- **Continuous tokenization.** Keep non-text as continuous embeddings (like the adapter connector's output) fed into the transformer. Advantages: preserves rich signal, minimal information loss. Trade-off: the model is hybrid — continuous for perception, discrete for text — and *generating* a continuous modality needs a separate decoder (e.g. a diffusion head for images). Most **understanding-focused** native models use this.
- **Discrete tokenization.** Convert images/audio to *discrete tokens* via a learned quantizer (VQ-VAE and successors), so non-text becomes a sequence of vocabulary IDs just like text. Advantages: **unified sequence modeling** — one transformer, one next-token-prediction objective, over a combined vocabulary of text + image + audio tokens; generation is just "predict the next token" in any modality. Trade-off: quantization loses information (a discrete image token is lossier than continuous features), so understanding can lag. This is the path to clean **any-to-any generation** (Chameleon, Emu3, "lexicalizing modalities as discrete tokens").

The 2026 landscape: **understanding-first models lean continuous; generation/any-to-any models lean discrete**, and hybrid designs (continuous in, discrete out, or separate understanding and generation paths like Janus's decoupled encoders) are common. Choosing the tokenization *is* choosing what the model can do — decide generation-vs-understanding first, then pick.

## 5.4 Architecture and training principles

The roadmap's recommendations for native design:

- **Early fusion at the token level** — interleave multimodal tokens early in the stack, not at a late fusion layer, so every layer reasons jointly.
- **Shared representation space** — unified vocabulary (discrete) or shared embedding space (continuous) to minimize modality-switching cost.
- **Modality-aware normalization** — modality-specific layer norms / adaptive normalization to stop one modality (usually text, with the most data) from dominating training dynamics. This is a real and subtle problem: without it, the abundant modality's gradients swamp the scarce ones.
- **Symmetric encoder-decoder paths** — for generation, avoid an LLM-centric bottleneck; design input and output paths for each modality symmetrically.
- **Joint objectives** — combine generative (next-token), contrastive (alignment), and reconstruction losses so modalities align *and* generate.

The staged training recipe mirrors the adapter one at a higher level: **foundation stage** (joint pretraining on large multimodal corpora) → **alignment stage** (align modality representations, instruction tuning) → **specialization** (task/capability tuning) → **RL stage** (preference/RLHF for generation quality). The critical native-specific ingredient (Qwen-Omni): **mix unimodal and cross-modal data in the early pretraining** — pure-text, pure-image, pure-audio *and* paired data together — so the model builds both strong per-modality representations and cross-modal links, avoiding the degradation that pure-cross-modal training causes.

## 5.5 Any-to-any generation

The capability that most justifies going native: one model that takes any modality in and produces any modality out. Mechanics:

- **Discrete-token models** generate non-text by predicting image/audio tokens autoregressively, then decoding them back to pixels/waveform with the quantizer's decoder — unified next-token prediction across modalities (Emu3, Chameleon).
- **Hybrid models** pair an understanding transformer with a generation head — e.g. a diffusion decoder for images conditioned on the transformer's output, or a separate "talker" module for streaming speech (Qwen-Omni's thinker-talker split, where a text-reasoning "thinker" drives a speech-generating "talker" for real-time voice).
- **Decoupled designs** (Janus) use *separate* visual encoders for understanding vs generation, because the features good for understanding differ from those good for generating — a key insight that resolved the understanding-generation tension.

For most teams this is aspirational; if you need image or speech *output*, native (likely discrete or hybrid) is the path, and it's a serious undertaking. If you need only understanding, stay adapter.

## 5.6 Decisions

1. **Choose native only for any-to-any generation or frontier cross-modal reasoning** with the compute to match; otherwise use adapter VLMs (Chapters 2–4).
2. **Decide generation-vs-understanding first**, then pick tokenization: continuous for understanding-first, discrete for generation/any-to-any, hybrid/decoupled for both.
3. **Fuse early, share the representation space, and use modality-aware normalization** to stop the data-rich modality from dominating.
4. **Mix unimodal and cross-modal data in early pretraining** (the Qwen-Omni ingredient) to get per-modality strength *and* cross-modal links without degradation.
5. **For speech/image output**, adopt a hybrid or decoupled design (thinker-talker, separate understanding/generation encoders) rather than forcing one path to do both.

Next: multimodal data — the through-line lever, where curation beats architecture.
