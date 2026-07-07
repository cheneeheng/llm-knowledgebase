# Chapter 2 — The Adapter VLM

## 2.1 Conceptual picture

The adapter (compositional) VLM is three parts in a row: a **vision encoder** turns an image into a set of vectors; a **connector** (projector) maps those vectors into the LLM's embedding space; the **LLM** consumes them alongside text tokens and reasons. That's it. LLaVA established the template in 2023 and it has barely changed because it *works*: reuse two strong pretrained models, train a small bridge between them, and you have a capable VLM for a tiny fraction of the cost of training multimodal from scratch. Understanding this three-part composition is understanding most of practical multimodal training.

The elegance is that the LLM doesn't need to know an image is an image. The connector produces vectors that live in the same space as text-token embeddings, so from the transformer's perspective the image is just some extra tokens in the sequence — "soft tokens" it attends over like any others. The whole trick is making those soft tokens *meaningful* to the LLM, which is the connector's job and the alignment stage's purpose (Chapter 4).

## 2.2 The three components

**Vision encoder** (Chapter 3 in depth). A pretrained image model — almost always a ViT trained with a vision-language objective like CLIP or SigLIP2 — that turns an image into a grid of patch embeddings (e.g. a 384×384 image → a 27×27 grid → ~729 patch vectors). It already encodes rich visual semantics because it was pretrained to align images with text. You inherit its quality; you mostly don't train it.

**Connector / projector.** The learned bridge, and the only component that's always trained. It maps the encoder's output vectors (in the encoder's dimension and semantics) to the LLM's input space (the LLM's hidden dimension and its learned notion of what an embedding means). This is where the design choices live (2.3).

**LLM backbone.** Your text model (any from the earlier series — an 8B, a 70B, a reasoning model). It receives the projected visual tokens interleaved with text tokens and does what it always does: attend and predict. Whether you train it, LoRA it, or freeze it is a staging decision (Chapter 4).

## 2.3 Connector types

The connector is a small architecture with an outsized reputation; the 2026 consensus is that **simple wins**.

- **MLP projector (the default).** A 2-layer MLP mapping each patch embedding to an LLM token embedding. LLaVA-1.5 switched from a linear layer to an MLP and it's been the standard since. It preserves *all* the visual tokens (one LLM token per patch), is trivial to train, and — the key 2026 finding — "MLP projectors are highly effective and more lightweight compared with prior connectors such as Resamplers and Q-Formers." Start here; you'll rarely need more.
- **Q-Former (BLIP-2).** A small transformer with a fixed set of learned query tokens that *cross-attend* to the visual features, compressing them into a fixed, small number of output tokens (e.g. 32) regardless of image size. Reduces the visual-token budget dramatically, but is harder to train and empirically underperforms the MLP on understanding tasks. Largely fell out of favor.
- **Perceiver Resampler (Flamingo).** Similar idea — a fixed set of latents resamples variable-length visual features to a fixed length via cross-attention. Used in Flamingo/IDEFICS for interleaved data; also mostly superseded by MLP + tiling for pure understanding.

The tension the connector resolves is **token count**: an MLP keeps every patch (many tokens, high fidelity, expensive); a resampler compresses to few tokens (cheap, some loss). The 2026 answer for most work is "keep the tokens (MLP) and manage the budget with resolution/tiling choices (Chapter 3)," because visual fidelity matters for OCR, charts, and grounding — the tasks that still discriminate (Chapter 1.3).

## 2.4 Early vs late fusion

*Where* in the LLM the visual tokens enter matters:

- **Early fusion (the LLaVA line, dominant).** Project visual tokens into the LLM's input embedding space and prepend/interleave them with text tokens at the *input* — the LLM's very first layer sees them and every layer attends over them. Simple, powerful, and the default. The visual tokens are full participants in the residual stream.
- **Cross-attention fusion (the Flamingo line).** Keep the LLM's text stream separate and inject visual information via *added cross-attention layers* at intervals in the LLM. Keeps the text pathway more intact (less text forgetting) and handles many interleaved images gracefully, but adds parameters and complexity. Used by Flamingo/IDEFICS and some large models; less common than early fusion for standard VLMs.

Note the terminology overlap with native models (Chapter 5): there, "early fusion" means training *all modalities jointly from scratch* in one model. Here, within adapter VLMs, "early fusion" means injecting visual tokens at the LLM input. Both point the same direction — fuse sooner, in a shared space — but at different scopes.

## 2.5 The visual-token budget

The single most important quantitative concept in adapter VLMs, analogous to serving's KV budget. Every image costs some number of tokens in the LLM's sequence, and that number is the master trade:

- **More visual tokens** (higher resolution, more tiles, MLP not resampler) → better fine detail, OCR, small-object grounding → **but** longer sequences, more compute, more memory, and visual tokens crowding out text context.
- **Fewer visual tokens** (lower resolution, tiling caps, resampler) → cheaper, longer effective text context → **but** worse on detail-heavy tasks.

Concrete: a single 384×384 tile is ~729 tokens (SigLIP-style); dynamic tiling up to 12 tiles + a thumbnail can push one high-res image to **several thousand** visual tokens — often more than the text prompt. Video multiplies this by frame count and becomes the dominant cost (Chapter 8). Every resolution and tiling decision in Chapter 3, and every compression technique, is managing this budget. Reason about it explicitly for your tasks: an OCR/document product spends tokens on resolution; a general chat VLM economizes.

## 2.6 Why this architecture dominates

It's worth being explicit about why the simple three-part adapter is so entrenched, because it justifies starting there:

1. **Compute efficiency** — you train a connector (millions of params) and maybe LoRA the LLM, not a full multimodal model. An 8B VLM can be built for hundreds of dollars.
2. **Modularity** — a better vision encoder or a better LLM drops in with a re-align; you're not locked to a monolith.
3. **Leverages the best of both worlds** — a vision encoder pretrained on billions of image-text pairs and an LLM pretrained on trillions of text tokens, combined without redoing either.
4. **It's enough** — for understanding tasks (the vast majority of use cases), adapter VLMs are competitive with native models; native's edge is in generation and the deepest cross-modal reasoning, which most products don't need.

## 2.7 Decisions

1. **Build an adapter VLM** (encoder + connector + LLM) unless you specifically need any-to-any generation or frontier cross-modal reasoning (then Chapter 5).
2. **Use an MLP projector** as the connector — simple, effective, and the 2026 default; reach for a resampler only under a hard token-budget constraint.
3. **Use early (input-space) fusion** for standard VLMs; consider cross-attention fusion only for heavy interleaved-image workloads or when text-forgetting is severe.
4. **Reason about the visual-token budget explicitly** — it's the master trade between visual fidelity and cost/context; spend tokens where your tasks need detail (OCR, charts, grounding), economize elsewhere.
5. **Keep the visual tokens (MLP), manage the budget with resolution/tiling** (Chapter 3) rather than compressing them away with a resampler.

Next: vision encoders — the eye you're inheriting, and the resolution and freezing decisions that most affect what your VLM can see.
