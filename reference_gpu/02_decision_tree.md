# 02 — Decision tree: whether to resize, and which axis

Back to [GPUResizing.md](../GPUResizing.md).

---

Ask in this order. Stop at the first “yes.” Identify the binder *per pool* (generation vs embed vs OCR vs train) — see [05_workload_pools.md](05_workload_pools.md).

```
1. Is the model / quantisation / max context the real constraint?
   YES → change S (quantise, cap context, split workloads). Do not buy a bigger GPU yet.

2. Is latency SLO breached while GPU util is high AND queue depth is growing?
   YES → add H (same SKU replicas) if the model still fits and tokens/s per GPU is healthy.

3. Are you OOM, KV-preempting, or unable to reach target concurrency on this SKU?
   YES → V (more VRAM / bandwidth) or S (smaller KV: shorter ctx, fewer seqs, GQA/MLA, FP8 KV).

4. Do weights + KV at target concurrency exceed the largest single card you can get?
   YES → W (tensor parallel), or shrink the model / context, or keep that model on an API.

5. Is utilisation low and cost high?
   YES → scale H down, consolidate pools, or stop self-hosting that workload.
```

## Signals that look like “need GPUs” but are not

| Symptom | Often actually | First move |
|---|---|---|
| GPU util ~95% | Healthy vLLM | Ignore util; look at queue + TTFT + KV |
| TTFT up, KV full, preemption | Context/concurrency too high for VRAM | Cap `max-model-len` / `max-num-seqs`, or V |
| TTFT up, KV empty, queue deep | Not enough replicas *or* prefill-bound SKU | H if tokens/s/GPU at knee; else V |
| OOM on one tenant | No per-tenant cap | Tenant concurrency / context cap |
| Chat p95 dies during ingest | Shared pool | Split OCR/embed off generation |
| High $ , empty queue at night | Over-provisioned H | Scheduled scale-down to minReplicas |
| Quality complaints after “upgrade” | Quant/TP/engine change | Golden set; rollback SKU/flags |

## Target you must write down before buying

Example: “p95 TTFT ≤ 800 ms at 40 concurrent 8k-ctx generations, with 20% KV headroom, N+1 replicas.”

Without concurrency × context × SLO, every SKU is defensible and none is.

Binder identification in code: [11_examples.md](11_examples.md). Sizing math: [04_resources_and_sizing.md](04_resources_and_sizing.md).
