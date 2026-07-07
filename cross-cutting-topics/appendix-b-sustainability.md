# Appendix B — Sustainability: Energy, Carbon, and Power

*A companion to Chapters 7 and 8. Energy was the one constraint family the series skipped; by 2026 it has joined the others — as a cost line that tracks the compute budget, an emerging reporting obligation, and (at frontier scale) the binding physical constraint on growth. This appendix is the practitioner's view: the numbers, the levers, and how much of this is actually your problem at each scale.*

## B.1 The scale of the thing

Orientation numbers, mid-2026: datacenters consumed roughly **500 TWh globally in 2025 (~2% of world electricity)**, growing 15–20% annually with AI as the primary driver. Within AI, the split mirrors the compute split this knowledge base has tracked all along: **inference dominates** — continuous serving load accounts for 80–90% of AI compute consumption, so the energy story, like the cost story (inference series Ch 12), is mostly a serving story. Training is spiky and visible (a frontier run is a large one-off draw — OLMo 3's 1,024 H100s for ~47 days is a few GWh); inference is the steady state that adds up. At frontier scale the constraint has become physical: **grid connection and power availability**, not GPUs, increasingly gate datacenter buildout — which is why power-grid co-design and energy-aware siting are 2026 research topics.

## B.2 Why it's on your desk (three forces)

1. **Cost.** Energy is a large share of GPU-hour pricing; everything in the FinOps loop (inference Ch 12.5) is also an energy lever. This alone justifies the efficiency work.
2. **Reporting.** The EU AI Act's GPAI documentation expects energy-consumption information for model training, and enterprise customers increasingly ask for per-workload carbon numbers; sustainability reporting frameworks (CSRD-class) pull AI workloads into scope. Track it before you're asked.
3. **Physics.** If you're building or reserving at scale, power procurement lead times now rival hardware lead times — a planning input to the cross-cutting Ch 8 compute-budgeting conversation.

## B.3 The metrics

Keep four numbers, mirroring the cost metrics you already track:

- **Energy per token (J/token)** for inference — the serving unit metric; falls with every optimization in the inference series.
- **Tokens/second/watt** for training throughput efficiency — MFU's energy twin.
- **PUE (Power Usage Effectiveness)** of the facility — global average ~1.55 in 2026 (down from 2.5 in 2007); hyperscale + liquid cooling reaches ~1.1–1.2. You mostly *choose* PUE via provider/region rather than engineer it.
- **Carbon intensity (gCO₂/kWh) of the grid feeding the facility** — varies ~50× between the cleanest and dirtiest regions, which makes *where and when* you compute the biggest carbon lever available (B.4).

Carbon per task = energy per task × PUE × grid intensity. Instrument the first factor; select the other two.

## B.4 The levers, in order of leverage

The encouraging fact: **almost every carbon lever is a cost lever you already have.** In rough order:

1. **Do less compute for the same output** — everything in this knowledge base: right-sized models and distillation (Ch 3), quantization (inference Ch 6), batching and caching (inference Ch 2–4, agentic Ch 2.4), thinking-budget caps (inference Ch 8), scaling-ladder discipline instead of failed big runs (pretraining). The efficiency chapters *are* the sustainability chapters.
2. **Site the compute on clean grids** — choosing a low-carbon region for training/serving cuts carbon multiples at often-zero cost difference; the ~50× regional spread dwarfs every other lever.
3. **Shift flexible work in time and space** — carbon-aware scheduling of batch/latency-tolerant workloads (the batch tier of inference Ch 12.4) to clean-grid hours/regions; elastic training that modulates batch size and active workers with grid carbon intensity showed **37–51% carbon savings** in 2026 studies. Your latency-tolerant tier is your carbon-flexible tier — the segmentation you built for cost doubles for carbon.
4. **Prefer efficient facilities** — low-PUE providers, liquid-cooled deployments (up to ~30% overhead reduction plus density gains). A procurement checkbox, not an engineering project.
5. **Hardware generation** — newer accelerators do more per watt; the B.3 tokens/s/W metric makes upgrade math explicit.

## B.5 How much is your problem, by scale

- **API consumer:** effectively none of it operationally — but per-token efficiency still argues for right-sized model choice, and your vendor's numbers feed your reporting.
- **Self-hosting a fleet:** track J/token alongside $/token (same dashboards), choose region/provider for grid + PUE, and batch-tier for carbon as you already do for cost. A day of setup, ongoing near-zero marginal effort.
- **Training at scale:** report training energy (regulatory expectation), site runs on clean grids, and consider carbon-aware elasticity for non-urgent experiments.
- **Building datacenters:** power procurement, grid co-design, and cooling are now first-order — beyond this knowledge base's scope, but the reason your GPU reservation quotes what it does.

## B.6 Decisions

1. **Track energy per token and per training run** alongside cost — same instrumentation, and the reporting ask is coming (EU AI Act GPAI documentation already expects training energy).
2. **Treat efficiency work as the primary lever** — the cost-optimization chapters across this knowledge base are the sustainability program; there is almost no trade-off to manage.
3. **Site on clean grids** — the ~50× regional carbon spread is the biggest single lever and usually free.
4. **Make the latency-tolerant tier carbon-aware** — time/space-shift batch work (37–51% demonstrated savings for elastic training) using the segmentation you built for cost.
5. **Scale concern to scale of operation** — an API consumer notes it; a fleet operator instruments it; a frontier trainer plans power like hardware.
