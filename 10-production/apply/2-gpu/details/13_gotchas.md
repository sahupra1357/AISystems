# 13 — Finer details that bite in production

Back to [GPUResizing.md](../GPUResizing.md). Related knobs: [08_serving_knobs.md](08_serving_knobs.md).

---

**CUDA graphs and `gpu-memory-utilization`.** Graphs capture extra memory. After a V-axis upgrade teams raise utilisation to “use the card” and OOM on the first traffic spike.

**GQA / MLA vs MHA.** KV size is `kv_heads`, not `query_heads`. Do not size KV from parameter count alone.

**Long-context is a pool, not a flag.** Raising `max-model-len` on every replica to 128k can cut concurrency by 8–16×.

**Prefix cache vs replica count.** More replicas reduce local prefix-cache hit rate. Measure after scale-out.

**Disaggregated prefill/decode** specialises GPUs. Large-scale only; not a first-year move.

**Multi-LoRA.** Active adapter count can OOM a SKU that fit a single adapter.

**Training jobs on serving node groups.** A `nvidia.com/gpu: 8` training pod will evict or starve serving if they share a queue. Separate taints.

**Clock / power / thermal.** Some clouds run GPUs at reduced clocks under dense packing. Tokens/s then miss the datasheet. Benchmark *on the actual instance shape*.

**NUMA and GPU–CPU pinning.** High prefill + tokenisation can become CPU-bound after a GPU upgrade. Watch vCPU steal and tokeniser time; you may need more CPUs on the *same* GPU node — a different resize.

**Network for TP.** TP=8 needs NVLink / NVSwitch. TP=8 across PCIe-only cards is a latency regression. Width axis implies SKU family, not just count.

**Spot interruption.** GPU spot reclaim is abrupt. Serving: on-demand. Training: checkpoint often.

**Driver mismatch on mixed node groups.** Cluster with both 535 and 550 drivers will place pods on nodes that cannot run the image.

**Readiness vs liveness.** Liveness that hits a heavy `/health` during load will kill slow pods. Liveness cheap; readiness = weights loaded.

**PDB + HPA fighting a resize.** During rolling SKU replace, HPA may scale the old Deployment. Freeze HPA or flip at the gateway to a new Deployment name.

**Regional inventory.** The correct SKU in one region may be unavailable. Plan a second AZ/region or API fallback.

**CPU raster in OCR.** Underutilised OCR GPUs usually mean rasterisation is the bottleneck — move that to a CPU pre-stage.
