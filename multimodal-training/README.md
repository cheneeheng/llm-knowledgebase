# Training a Multimodal Model: The Complete Playbook

*From a text LLM to a model that sees, hears, and speaks — vision encoders, connectors, native fusion, image/audio/video data, omni models, multimodal post-training, and evaluation.*

*Compiled July 2026. A companion to the [pretraining](../pretraining-from-scratch/README.md), [midtraining](../midtraining-and-continued-pretraining/README.md), [post-training](../post-training-from-scratch/README.md), and [inference](../inference-and-serving/README.md) series. Assumes you know how a text LLM is trained (those series) and what a transformer and attention are; assumes nothing about vision or audio modeling. Covers all scales, from bolting a vision encoder onto an 8B in an afternoon to training a native omni model.*

---

## Why this series exists

By 2026 "an LLM" almost always means a *multimodal* model — it reads images, documents, charts, and increasingly audio and video. Every frontier model (GPT, Gemini, Claude, Qwen-Omni) is multimodal, and the frontier benchmark for pure image-QA (MMMU-Pro) is *saturated* — every frontier model clears ~80%, so the action has moved to video, audio, OCR-heavy documents, and chart reasoning. Multimodality is no longer a research curiosity bolted onto text models; it is a core competency, and training it is its own discipline layered on everything the text series taught.

That discipline has two big shapes, and choosing between them is the spine of this series. The **adapter (compositional) approach** connects a pretrained vision (or audio) encoder to a pretrained text LLM through a small learned connector — cheap, modular, and how most teams (and most 2024–2026 open VLMs) do it. The **native (early-fusion) approach** trains a single model over all modalities from the start, treating vision, audio, and text as first-class citizens in one representation space — more expensive, less modular, and where the frontier is heading for genuine cross-modal reasoning and any-to-any generation. This series covers both, in depth, and tells you which to reach for when.

## How to read this report

Every chapter is **layered**, like the companion series: it opens with the *conceptual picture* — what the thing is and why it exists — then descends into the *practitioner layer*: concrete recipes, token budgets, which encoder, which connector, which training stage freezes what. If you want the map, read the opening of each chapter and the "Decisions" summaries. If you're about to train one, read everything.

A recurring theme: **multimodal training is mostly a data and alignment problem, not an architecture problem.** The architecture (encoder + connector + LLM) is nearly settled and simple; what separates a good VLM from a bad one is the data mixture, the staging of what you train when, and ruthless evaluation of hallucination. The strongest recent result is that *data curation alone* — same architecture — moves VLM quality more than architecture tweaks. Get the data and the staging right and the rest is mechanical.

## Chapters

| # | File | Contents |
|---|---|---|
| 1 | [The Big Picture](01-big-picture.md) | What multimodal training is, adapter vs native, the moved frontier, the core mental models |
| 2 | [The Adapter VLM](02-adapter-vlm.md) | The encoder+connector+LLM composition, connector types (MLP/Q-Former/Perceiver), early vs late fusion, the token-budget problem |
| 3 | [Vision Encoders](03-vision-encoders.md) | CLIP/SigLIP2, native resolution and dynamic tiling, frozen vs unfrozen, resolution/token trade, choosing an encoder |
| 4 | [The VLM Training Recipe](04-vlm-training-recipe.md) | The multi-stage recipe: connector alignment → joint tuning → instruction tuning; what to freeze when; hyperparameters |
| 5 | [Native Multimodal Models](05-native-multimodal.md) | Early fusion, unified tokenization (discrete vs continuous), any-to-any generation, when native beats adapter |
| 6 | [Multimodal Data](06-data.md) | Image-text pairs, interleaved data, captioning, OCR, grounding, synthetic data, curation, the mixture |
| 7 | [Audio, Speech, and Omni Models](07-audio-speech-and-omni.md) | Audio encoders, ASR/speech, omni (any-modality) models, streaming, real-time speech generation |
| 8 | [Video](08-video.md) | Temporal modeling, frame sampling, video token explosion and compression, video data and evaluation |
| 9 | [Multimodal Post-Training](09-multimodal-post-training.md) | Visual instruction tuning, multimodal RLHF/RLVR, hallucination reduction, reasoning over images |
| 10 | [Evaluation](10-evaluation.md) | MMMU and saturation, hallucination (POPE), OCR/document, chart, video, the moved eval axes, contamination |
| 11 | [Infrastructure and Frameworks](11-infrastructure-and-frameworks.md) | What changes from text training, the token-budget/batching deltas, frameworks, data pipelines, cost |
| 12 | [Playbooks by Scale](12-playbooks-by-scale.md) | Recipes: LoRA a VLM on one GPU, an 8B VLM, a frontier VLM, a native omni model |
| 13 | [Failure Modes](13-failure-modes.md) | Hallucination, modality imbalance, resolution starvation, connector collapse, catastrophic text forgetting |
| 14 | [Essential Reading and References](14-essential-reading-and-references.md) | The must-read sources, ordered, plus the full source list |

## Scope

This series covers turning a text model into a model that *understands* (and, for native/omni models, *generates*) other modalities — primarily vision, with audio, speech, and video treated in their own chapters. It leans on the text series for everything shared (the LLM backbone's pretraining, the post-training algorithms, the serving) and covers only the multimodal deltas. Pure image/video *generation* models (diffusion, Stable-Diffusion-lineage) are out of scope except where native models unify understanding and generation (Chapter 5); embedding/retrieval models get a nod in the cross-cutting series.

## The through-line

Multimodal training rewards **balanced curation over architectural cleverness**. The architecture is a solved, simple recipe; the hard part is assembling a data mixture that covers captioning, interleaved text, OCR, grounding, charts, and reasoning without letting any one modality or task dominate, and staging the training so the model aligns the modalities before it specializes — all while never letting the new modality quietly degrade the text capability you started with. The teams that win measure hallucination as obsessively as accuracy, keep the text evals running alongside the vision ones, and reach for more and better data before they reach for a fancier connector. Align first, curate relentlessly, and watch what the new eye does to the old tongue.
