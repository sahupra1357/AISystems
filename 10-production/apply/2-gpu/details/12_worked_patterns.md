# 12 — Worked patterns for a prod AI app

Back to [GPUResizing.md](../GPUResizing.md).

---

## A — RAG chat, self-hosted 8B + API reasoning (common hybrid)

Generation: 1× L4/A10G per replica, `max-model-len` 8–16k, HPA on queue. Embeddings: separate 1× L4 with large batch. Rerank: same or MIG slice.

Resize path: first cap context and prefix-cache (**S**), then add 8B replicas (**H**). Do not jump to H100.

## B — Self-hosted 70B for residency

Weights ~40 GB at 4-bit → 80 GB card for KV, or TP=2 on 40 GB. Vertical to 80 GB usually beats TP=2 on 40 GB for latency. Keep an 8B pool for rewrite/classify so the 70B fleet stays small.

## C — Ingest OCR spike

Do not add chat GPUs. Page queue → OCR pool with scale-from-zero or scheduled scale before known backfills. If GPU util is low, CPU raster is often the bottleneck — pre-stage images on CPU workers.

## D — Fine-tune weekly, serve always

Training node group (spot OK) distinct from serving (on-demand). Never “borrow” serving GPUs for an emergency train run without an explicit shed of non-interactive traffic.

## E — Over-provisioned overnight

Scheduled **H** scale-down to `minReplicas` that still hold SLO + N+1. Not scale-to-zero for interactive. Cron-based scale-up **30+ minutes** before known peaks (GPU provision time).

## At larger scale

- **Pools > SKUs > replicas.** Org chart of GPUs matches workload profiles, not a single `ml-gpu` node group.
- **N+1 across AZs**, not just replica count.
- **FinOps monthly:** `$/1M tokens` by pool, idle %, KV p95, reservation utilisation. Vertical overbuy shows up as idle VRAM, not idle SM.
