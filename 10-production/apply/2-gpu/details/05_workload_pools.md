# 05 — Workload pools: never resize one fleet

Back to [GPUResizing.md](../GPUResizing.md).

---

A prod AI application is several GPU *shapes*, not one. Resize **per pool**.

| Pool | Profile | Binding resource | Resize instinct |
|---|---|---|---|
| **Generation (LLM)** | Small batch, long KV, latency SLO | VRAM (KV) + bandwidth | H on queue/TTFT; V if KV-bound; W if model > 1 GPU |
| **Embeddings** | Huge batch, short seq, throughput | FLOPS + host I/O | H on queue depth; smaller/cheaper GPUs often win |
| **Rerankers / NLI** | Medium batch, short seq | FLOPS | Same as embeddings; do not co-locate with generation |
| **OCR / layout / VLM ingest** | Page-parallel, bursty, minutes-scale | FLOPS + CPU raster bottleneck | Separate queue; autoscale on page queue, not chat QPS |
| **Training / LoRA jobs** | Occupies a card for hours | All of VRAM, often multi-GPU | Isolated node group; never share with interactive serving |

## Why co-location fails

A generation replica with a full KV cache cannot absorb a 50k-chunk embed job. An OCR backfill will raise TTFT for interactive chat even if “average GPU util” looks fine. Training `nvidia.com/gpu: 8` on the serving node group evicts or starves chat.

**Shared node groups with time-slicing are for dev, not interactive prod.**

## Practical default for a RAG app

- **L4 / A10G class:** embeddings, rerank, 8B fast tier.
- **A100 / H100 class:** only the generation model that actually needs the VRAM/bandwidth.
- **Spot / scale-toward-zero:** OCR, embed backfill, training — not user-facing decode.

See patterns: [12_worked_patterns.md](12_worked_patterns.md).
