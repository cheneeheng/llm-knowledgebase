# Chapter 8 — Video

## 8.1 Conceptual picture

Video is images plus time, and both parts make it the hardest common modality. A video is a sequence of frames — each frame is an image costing hundreds-to-thousands of visual tokens (Chapter 2.5) — so a video costs `frames × tokens_per_frame`, which explodes fast: a few minutes of video at a reasonable frame rate and resolution can blow past any context window. On top of that, video has **temporal structure** — the model must understand not just each frame but how they relate over time (motion, events, causality, before/after). Video is where the visual-token budget becomes an existential constraint and where "just add frames" stops working. It's also, per Chapter 1.3, a *current frontier axis* (Gemini leads it) — so it's where differentiation lives.

## 8.2 The token explosion

The core problem, quantified. Say each frame costs ~256 visual tokens (already compressed). A 10-minute video at even 1 frame/second is 600 frames × 256 = ~150K tokens — an entire long-context budget for one video, before the question. At higher frame rates or resolutions it's hopeless. So video modeling is *dominated* by the question: **how do you represent enough of the video to answer questions about it, within a feasible token budget?** Every video technique is an answer to that.

## 8.3 Frame sampling

The first and bluntest lever: don't use every frame.

- **Uniform sampling** — take N evenly-spaced frames (e.g. 8, 16, 32, 64). Simple, standard, but misses events between sampled frames and wastes tokens on static stretches.
- **Adaptive / keyframe sampling** — sample more densely where the video changes (scene cuts, motion) and sparsely where it's static. Better information-per-token; more complex.
- **Frame rate vs coverage trade** — for a short clip you can afford dense sampling; for a long video you must sample sparsely and accept temporal blindness between frames, or use compression (8.4). The choice depends on whether your tasks need fine temporal detail (action recognition) or gist (what is this video about).

Sampling is where you spend or save the token budget; tie it to the task and the context window.

## 8.4 Token compression

Beyond sampling frames, compress the tokens *within and across* frames:

- **Per-frame compression** — pool/merge patch tokens within each frame (a frame doesn't need full image resolution if you have many frames), or use a resampler to fixed few-tokens-per-frame.
- **Temporal compression / merging** — merge redundant tokens *across* adjacent frames (consecutive frames are highly similar, so much is redundant). Techniques merge similar tokens over time, keeping only what changes.
- **Audio-guided / content-aware compression** — use one modality to guide compression of another (OmniZip uses audio to guide dynamic visual-token compression in omni models) — spend visual tokens where the audio suggests something is happening.

The goal is always the same: **maximize represented information per token** so more video fits the budget. Compression is what makes long-video understanding tractable; without it you're capped at short clips.

## 8.5 Temporal modeling

Representing frames is necessary but not sufficient — the model must understand *time*:

- **Temporal position encoding** — the model needs to know frame *order* and ideally *timing* (this frame is at 3:20). Frame position/timestamp embeddings give temporal grounding, letting the model answer "what happened after X."
- **Video-native architectures** — some models add explicit temporal attention or 3D patchifying (patches spanning space *and* time); others rely on the LLM's sequence modeling to pick up temporal relations from ordered frame tokens. The adapter approach (ordered frame tokens + temporal position info) works well and is simpler.
- **The data teaches temporality** — temporal understanding comes largely from *data* that requires it: "what happened before/after," action recognition, event ordering, temporal grounding ("when does X occur"). Without such data, a model with all the temporal-encoding machinery still won't reason about time (the data-beats-architecture through-line again).

## 8.6 Video data and evaluation

- **Data types:** video captioning (describe the video), video QA (answer questions requiring temporal understanding), temporal grounding (locate when an event happens), action recognition, and long-video understanding (summarize/answer over minutes). As with images, *coverage of temporal skills requires temporal data.*
- **Sourcing:** video-caption pairs (often synthetic — a VLM captions sampled frames, an LLM composes a temporal description), instructional videos with transcripts, and existing video datasets converted to instruction format. Audio tracks are a free supervision signal (transcribe speech, align to frames) — omni training (Chapter 7) makes video understanding *better* by using the audio.
- **Evaluation (Chapter 10):** video benchmarks target the temporal axis specifically — TempCompass (temporal perception), PerceptionTest (video/audio/text perception and reasoning), and long-video QA. Report these, not just frame-level accuracy, because a model can answer many "video" questions from a single frame — a benchmark that doesn't *require* temporal reasoning doesn't measure it.

## 8.7 Decisions

1. **Treat the token budget as the binding constraint** — video cost is `frames × tokens/frame`; design everything around fitting enough video in the window.
2. **Sample frames deliberately** — uniform for short clips, adaptive/keyframe for long or event-heavy video; match sampling density to whether tasks need fine temporal detail or gist.
3. **Compress tokens within and across frames** (per-frame pooling, temporal merging, content-guided compression) to make long-video tractable.
4. **Give the model temporal grounding** (frame timestamps/position) *and* train on data that requires temporal reasoning — machinery without temporal data won't learn time.
5. **Use the audio track as free supervision** (omni-style), and **evaluate on temporal benchmarks** that genuinely require multi-frame reasoning, not single-frame-answerable QA.

Next: multimodal post-training — instruction tuning, RLHF/RLVR over images, and the hallucination problem.
