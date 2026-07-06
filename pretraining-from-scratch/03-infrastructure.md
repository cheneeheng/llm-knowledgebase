# Chapter 3 — Infrastructure: The Cluster

## 3.1 Conceptual picture

Pretraining is a distributed-systems problem wearing a machine-learning hat. A training cluster is thousands of accelerators that must behave like one giant computer: every optimizer step, gradients (or gradient shards) must be synchronized across all data-parallel replicas, which means the network is as much a part of your "computer" as the GPUs. The three resources you are buying are **FLOPs** (the GPUs), **bandwidth** (NVLink within a node, InfiniBand/RoCE between nodes), and **memory** (HBM on-device, plus host RAM and storage). Pretraining is unusual in that it stresses all three simultaneously and continuously for weeks — which is why hardware failures, rare per-device, become a daily fact of life at cluster scale.

## 3.2 Choosing accelerators (as of mid-2026)

**NVIDIA Hopper (H100/H200) — the default.** ~990 TFLOPs dense BF16, ~2 PFLOPs FP8. H100: 80GB HBM3 at ~3.35 TB/s. H200: same compute, 141GB HBM3e at ~4.8 TB/s — the extra memory meaningfully eases large-model and long-context training; a low-risk upgrade. Mature software, enormous availability; this is what most published recipes assume. Typical cloud pricing in 2026: roughly $2–3.50/GPU-hour depending on provider tier and commitment length; prices have softened as Blackwell ramps.

**NVIDIA Blackwell (B200/B300/GB200 NVL72) — the frontier tier.** B200: ~2.2 PFLOPs dense BF16, ~4.5 PFLOPs FP8, native FP4, 192GB HBM3e at ~8 TB/s — roughly 2× H100 on large-model training in MLPerf. B300 ("Blackwell Ultra"): 288GB, ~15,000 FP4 TFLOPs dense, 1,400W. **GB200 NVL72** is a liquid-cooled rack fusing 72 GPUs into a single NVLink domain (~130 TB/s aggregate) that behaves like one enormous accelerator — transformative for expert-parallel all-to-all (Chapter 7), and it qualitatively changes parallelism strategy (very wide tensor/expert parallelism without touching the inter-node network). More expensive per hour, usually cheaper per FLOP; software now mature. Notes: top Blackwell SKUs dropped PCIe — B200 lives in HGX/NVL systems; through mid-2026 supply remains constrained and largely reserved, with on-demand seen around ~$3–6/GPU-hr at specialist clouds.

**AMD MI300X/MI325X/MI350-series.** Genuinely credible now: huge HBM (192–288GB), competitive FLOPs, MLPerf-validated pretraining via ROCm with Megatron-LM and torchtitan backends (AMD's Primus framework unifies these); MI355X supports MXFP4 training. Trade-offs: smaller talent pool, more rough edges, more integration work. Viable if the price is right and your team can absorb some systems work.

**Google TPU (v5p/v6/Trillium).** A genuine alternative used to train frontier models (Gemini). Superb interconnect (ICI/OCS pod topology) and reliability; the path is JAX + XLA (MaxText, Chapter 6). The constraint is ecosystem lock-in and GCP-only availability.

**Recommendation for a first-timer with a cloud cluster:** H100/H200 with InfiniBand, mainstream PyTorch stack. Every tutorial, recipe, and Stack Overflow answer assumes this configuration. Graduate to Blackwell/AMD/TPU when you have a run under your belt or when FP8/FP4 throughput at scale justifies the risk.

## 3.3 Interconnect — the thing that actually determines whether you scale

Compute is easy to buy; **moving gradients and activations between GPUs is what limits you.** Two tiers:

- **Scale-up (within a node / NVLink domain).** NVLink/NVSwitch connects GPUs inside a box (or, for NVL72, a rack): ~900 GB/s per GPU on Hopper, ~1.8 TB/s bidirectional on Blackwell (NVLink 5). This is where the *chattiest* parallelism lives: tensor parallelism and expert-parallel all-to-all, which exchange data every layer.
- **Scale-out (between nodes).** InfiniBand (NDR 400 Gb/s, XDR 800 Gb/s per link) or RoCE v2 Ethernet, in a **rail-optimized, non-blocking (or near-non-blocking) fat-tree** topology. NVIDIA's Quantum-X800 InfiniBand and Spectrum-X Ethernet are the 800 Gb/s options. This tier carries the less chatty traffic: data-parallel/FSDP gradient collectives and pipeline point-to-point.

**The golden rule of placement: map communication frequency to bandwidth tier.** Tensor parallelism inside the NVLink domain, never across node boundaries; data parallelism can tolerate the network. Getting this backwards can cut throughput by more than half. This single principle drives most of Chapter 7.

Practical requirements: **GPUDirect RDMA** (NICs write GPU memory directly, bypassing the CPU), SHARP (in-network reduction on InfiniBand), and one NIC per GPU ("one rail per GPU" — 8 rails per 8-GPU node) for full bandwidth on large collectives. Oversubscribed networks silently destroy MFU: when evaluating a provider, run `nccl-tests` (all-reduce, all-gather, reduce-scatter) across your full allocation *before signing*; you want bus bandwidth near line rate at large message sizes.

## 3.4 Cluster topology and node shape

A standard NVIDIA training node is **8 GPUs** (HGX/DGX), fully NVLinked internally, with 8–9 NICs (one per GPU plus storage/management). Clusters are built as rail-optimized fat-trees: every GPU's NIC connects to a leaf switch on its "rail," leaves to spines, so same-rank GPUs across nodes get non-blocking bandwidth for collectives.

Sizes in practice: a serious 8B run uses 256–1,024 GPUs; a 70B run 1,000–4,000; a frontier run 10,000–100,000+. Engineering difficulty grows *super*-linearly with GPU count: failure rate scales with component count (3.7), and communication efficiency degrades as collectives span more switches.

**CPU/host side:** plenty of host RAM (1–2TB/node) and CPU cores for dataloader workers. Data loading should never be the bottleneck; you verify that by watching for GPU-utilization dips at step boundaries.

## 3.5 Storage and the data path

Your GPUs are a multi-million-dollar engine; **data starvation is the cardinal sin.** Three tiers:

1. **Hot path (active shards + checkpoints):** a parallel filesystem (Lustre, GPFS, WEKA, or vendor equivalent) or high-throughput object store, sized for aggregate bandwidth. Checkpoint writes are bursty and huge (3.6) and must not stall training.
2. **Tokenized training data:** pre-tokenize offline and store as memory-mapped binary shards (Megatron `.bin`/`.idx`, MosaicML Streaming, WebDataset tar shards). You do **not** tokenize on the fly at scale. Loaders pull shards from object storage to node-local NVMe, shuffle at shard and sample level, and keep a prefetch buffer. Rule of thumb: the loader must sustain (global batch tokens ÷ step time) with comfortable margin; pre-tokenized data keeps this modest (single-digit GB/s cluster-wide).
3. **Cold object storage (S3/GCS):** the raw data lake and checkpoint archive.

## 3.6 Checkpointing

Checkpoints are your insurance policy against everything in 3.7. Key facts:

- A checkpoint stores model weights **plus optimizer state** (Adam keeps two FP32 moments, plus FP32 master weights in mixed precision → optimizer state is ~3× the BF16 parameter bytes). A 70B model's full training state is **~1TB+**; an 8B is ~130GB.
- Write **asynchronously** (PyTorch Distributed Checkpoint (DCP), Megatron/NeMo async save, torchtitan distributed checkpointing): each rank writes its shard while training continues, so a checkpoint costs seconds of GPU stall, not minutes. Stage to node-local NVMe, upload to object storage in the background.
- **Cadence:** every 30–60 minutes is typical — tune so (expected lost work × failure rate) stays small relative to checkpoint cost.
- **Retention:** keep a rolling window of the last N checkpoints (the latest may be the corrupt one) plus periodic permanent snapshots (e.g., every 1,000 steps forever) — you will want them for eval curves, for rewinding past data-induced loss spikes, and for post-hoc research.
- Checkpoints must be **resharding-aware**: you may resume on a different GPU count or parallelism layout than you saved from. Use a format that supports this (DCP, Megatron distributed checkpoint).
- Also checkpoint the **dataloader state** (position, shuffle seed, epoch) and RNG states — exact resumability is non-negotiable (Chapter 4.6).

## 3.7 Fault tolerance — the defining operational problem

At scale, failure is continuous, not exceptional. Meta reported roughly one interruption every ~3 hours during Llama 3 405B training on 16K H100s — over 400 GPU-related failures in 54 days, largely HBM errors, GPUs falling off the bus, and network flaps. Scale that intuition: at 1,024 GPUs expect a hardware interruption every day or two. Your defenses:

**Automatic detection and restart.** The job should detect failure, exclude the bad node, pull a healthy spare from a warm pool (keep **2–5% spare capacity**), and resume from the last checkpoint with zero human intervention — mature teams get median time-to-resume under 10 minutes. Slurm requeue hooks or K8s controllers plus a watchdog monitoring training-log heartbeats is the standard pattern; frameworks increasingly ship pieces of this (torchft, NVIDIA resiliency features).

**Health screening / burn-in.** Before every large job and after every restart: `nccl-tests` across all nodes, per-GPU GEMM benchmarks (to catch throttling or "silently slow" GPUs), memory tests, disk/network checks. One degraded GPU among 1,024 sets the pace for all 1,024. Cloud providers ship marginal nodes more often than you'd hope; automated screening that drains bad nodes is table stakes.

**Straggler and silent-failure detection.** The nastiest failures don't crash. Watch per-rank step times to catch stragglers, and monitor loss/grad-norm for **silent data corruption (SDC)** — GPUs that compute wrong answers without erroring. Frontier labs run periodic online numeric self-checks; at smaller scale, your main defense is determinism (bitwise-reproducible restarts turn "mystery divergence" into "diffable bug") plus anomaly eyeballing.

**Elastic training.** Some stacks let the job continue on fewer nodes rather than halting, then re-expand — trading a little throughput for a lot of uptime.

The metric that captures all of this is **goodput**: the fraction of wall-clock time spent making useful forward progress (not restarting, recomputing lost work, or idling). A run at 70% goodput costs ~40% more wall-clock and dollars than one at 95%.

## 3.8 Orchestration and scheduling

The classic stack is **Slurm** — batch scheduling, gang-scheduled multi-node jobs, `sbatch` scripts launching `srun`/`torchrun`. It remains the default at most GPU clouds (CoreWeave, Lambda, Crusoe, Nebius, etc. offer managed Slurm). **Kubernetes** with Kueue/Volcano (or provider-managed equivalents) is the alternative — better if your org is already K8s-native and wants one substrate for training, data processing, and evals. Either works; what matters is **gang scheduling** (all N nodes or none), **topology-aware placement** (your job's nodes network-adjacent), and **automatic node health screening** wired into the scheduler.

## 3.9 Monitoring and observability

Instrument three layers and put them on one dashboard (Prometheus/Grafana + W&B or MLflow is the common combo):

- **Hardware:** per-GPU utilization, HBM usage, temperature, power, ECC error counts, NVLink/IB throughput, node up/down.
- **Training systems:** step time and its variance, MFU, tokens/sec, dataloader wait time, checkpoint duration, goodput.
- **Learning:** loss, per-domain loss, gradient norm, parameter/update norms, learning rate, periodic eval metrics (Chapter 9).

Alert on: loss spikes or NaN, grad-norm explosions, step-time regression >5%, ECC error bursts, heartbeat loss.

## 3.10 Procurement — how to actually get the machines

The market:

- **Hyperscalers (AWS, GCP, Azure, OCI):** deepest capacity, managed networking, strongest enterprise integration, highest sticker price. B200/GB200 instances rolled out through 2026, often reserved for large customers first.
- **Neoclouds (CoreWeave, Lambda, Crusoe, Nebius, Together, and many more):** GPU-specialist clouds, typically 30–50% cheaper for the same silicon, InfiniBand clusters purpose-built for training. This is where most independent pretraining actually happens. Budget rule of thumb mid-2026: ~$2–2.5/H100-hour at multi-hundred-GPU commitment.
- **Reserved vs. on-demand vs. spot:** long runs want **reserved/committed** capacity (1–12 months) — dramatically cheaper and you cannot have a month-long run evicted. Spot/preemptible is for cheap ablations and ladder runs that checkpoint aggressively, not the main event.

**What to verify and negotiate before signing:**

- Interconnect: InfiniBand NDR/XDR (or equivalent RoCE) — non-negotiable for multi-node; validated topology (non-blocking/rail-optimized), confirmed with your own `nccl-tests` during an **acceptance-test window before the billing clock starts**.
- Node-replacement SLAs in hours, not days; a spare-node pool.
- Local NVMe per node; storage aggregate bandwidth; real availability dates (Blackwell backlogs are long).
