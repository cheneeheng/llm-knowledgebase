# Chapter 7 — Audio, Speech, and Omni Models

## 7.1 Conceptual picture

Everything in Chapters 2–4 generalizes from vision to audio: an **audio encoder** turns sound into vectors, a **connector** maps them into the LLM, and the LLM reasons over them. The architecture is the same "encoder + connector + LLM" adapter pattern, and the training is the same "align then jointly tune" recipe. What's new is that audio has *two* distinct jobs — **understanding** sound (speech recognition, audio comprehension, music/environment) and **generating** speech (talking back) — and that speech generation, especially *streaming real-time* speech, forces architecture that vision doesn't. An **omni model** is one that does all modalities at once — text, image, audio, video in; text and speech out — and is the frontier's current shape (Qwen-Omni, Gemini, GPT-realtime).

## 7.2 Audio understanding

The input side, structurally identical to vision:

- **Audio encoder.** Convert the waveform (usually via a mel-spectrogram) into vectors with a pretrained audio encoder — **Whisper's encoder** is the common workhorse (pretrained on massive ASR data, so its features are speech-aware), with newer purpose-built audio encoders in frontier models. The encoder is your ear, as the ViT was your eye.
- **Connector + LLM.** Project audio features into the LLM space (MLP or resampler — audio sequences are long, so resampling/downsampling is more common than in vision) and train with the staged recipe: align on audio-text pairs (ASR transcripts, audio captions), then jointly tune on audio instructions.
- **Tasks:** ASR (transcription), speech translation, audio understanding (what's happening in this sound), speech Q&A, and — increasingly — *reasoning* over audio (answer a question that requires understanding spoken content). Qwen3-Omni reaches ASR and audio-understanding parity with frontier proprietary models.

The token-budget problem is worse for audio than vision: audio is long (a minute of speech is a lot of frames), so **downsampling/compression of audio tokens** is important — you can't afford one token per audio frame. Resampler connectors and audio-token compression (OmniZip-style dynamic compression) manage this.

## 7.3 Speech generation

The output side, which vision-understanding VLMs don't have. Producing speech means the model must generate *audio* tokens, which requires audio to be first-class in the output — the native/discrete-token territory of Chapter 5. Approaches:

- **Discrete audio tokens + vocoder.** Represent speech as discrete tokens (via a neural audio codec like a VQ-VAE), have the model predict them autoregressively, and decode to waveform with the codec's vocoder. Unified next-token prediction, as in discrete native models.
- **Thinker-talker architectures (Qwen-Omni).** Split the model: a **thinker** (the LLM) reasons and produces text/semantics; a **talker** module generates streaming speech tokens conditioned on the thinker's output. This decouples reasoning from speech synthesis and — critically — enables **streaming**: the talker can start speaking before the full response is reasoned, giving low-latency real-time voice.
- **Separate TTS (the non-native shortcut).** Just pipe the LLM's text output to a standalone text-to-speech system. Simple, modular, but not "native" — no prosody control from the reasoning, higher latency, and it can't do genuinely audio-native things (singing, tone, non-speech sound). Fine for many products; not an omni model.

## 7.4 Streaming and real-time

The hard constraint that shapes speech-generation architecture: **real-time voice interaction needs low latency and streaming in both directions.** The model must consume audio input as it arrives (not wait for the user to finish) and produce audio output as it reasons (not wait to finish thinking). This forces:

- **Streaming encoders** that process audio incrementally.
- **Streaming generation** (thinker-talker, where the talker emits speech chunks as the thinker produces content).
- **Latency budgets** measured in hundreds of milliseconds, like interactive serving (inference series Chapter 8) — real-time voice is a serving problem as much as a modeling one.

If your product is turn-based (send audio, get a response), you can avoid most of this. If it's real-time conversational voice, streaming is the central engineering challenge and the reason omni models are hard.

## 7.5 Omni models

An omni model unifies everything: text + image + audio + video in, text + speech out, in one model. The 2026 exemplar is Qwen3-Omni, and its recipe distills the whole series' lessons:

- **Native early fusion** (Chapter 5) — all modalities in one model, not separate adapters glued together.
- **Staged training with LLM frozen first** — "in the first stage, the LLM is locked and the vision and audio encoders are trained on audio-text and image-text pairs to enhance semantic understanding; in the second stage, all parameters are unfrozen and trained on a wider range of multimodal data." The same align-before-ability logic, across modalities.
- **Mix unimodal and cross-modal data early** (Chapter 5.4) — the key ingredient for parity without degradation.
- **Thinker-talker for streaming speech output** (7.3).
- **Result:** parity across all modalities (no modality sacrificed), enhanced cross-modal ability (e.g. video understanding benefits from joint audio-visual training), and real-time speech.

Building an omni model is a frontier undertaking — the compute and data of a native model (Chapter 5) times the number of modalities. Most teams building audio capability should start with **audio understanding via the adapter pattern** (7.2) and add TTS (7.3) if they need voice out, reserving true omni for when the product demands unified real-time multimodal interaction.

## 7.6 Decisions

1. **Add audio understanding via the adapter pattern** — audio encoder (Whisper-class) + connector + LLM, staged training — identical in shape to vision.
2. **Compress/downsample audio tokens aggressively** — audio sequences are long; one-token-per-frame is unaffordable.
3. **For voice output, choose by need:** standalone TTS for simple turn-based voice (modular, not native); thinker-talker / discrete audio tokens for native, streaming, low-latency speech.
4. **Treat real-time voice as a streaming (and serving) problem** — streaming encoders and generation, sub-second latency budgets; avoid it if turn-based suffices.
5. **Reserve full omni models for products needing unified real-time multimodal interaction** — start with adapter audio understanding + optional TTS; go omni (native, staged, unimodal+cross-modal data) only when the product demands it.

Next: video — where the token budget explodes and temporal modeling enters.
