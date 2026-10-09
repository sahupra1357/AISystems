# 14 — Prod checklist and metrics

Back to [GPUResizing.md](../GPUResizing.md).

---

## When to resize vs when not to

**Resize when**

- The model **does not fit** at required context × concurrency after quantisation and ctx caps.
- **Queue + TTFT** grow while tokens/s per GPU is already at the measured knee — need H.
- Decode **TPOT** misses SLO and the card is a generation behind on bandwidth — need V.
- Weights exceed one card — need W (or a smaller model / API).
- **Cost:** KV util p95 < ~30% and queue empty — H down or SKU down.

**Do not resize**

- To fix **quality** — that is a model/eval problem.
- Because **GPU utilisation is high** — that is normal for vLLM.
- Because **one tenant** ran 100k-token jobs — cap them.
- To absorb **ingest/eval/train** — separate pools.
- Before **caching, tiering, and context trim** if the goal is cost.
- In place on Friday afternoon without canary evals and overflow.

---

## Prod checklist

- [ ] Binder identified per pool (VRAM/KV vs queue vs TPOT vs overprovision) with traces, not anecdotes
- [ ] Software levers tried first: quant (eval-gated), `max-model-len`, `max-num-seqs`, prefix cache, tenant caps, split pools
- [ ] Target SLO written: concurrency × context × TTFT/TPOT
- [ ] VRAM math: weights + KV + headroom; TP degree is the minimum that fits
- [ ] Axis chosen: S / H / V / W — and written down
- [ ] New SKU/flags load-tested on **production length histogram**
- [ ] Golden-set quality compared at new quant/TP/engine/CUDA
- [ ] Node group per SKU per pool; exclusive GPU for interactive generation
- [ ] Readiness after model load; grace period ≥ max decode; PDB for N+1
- [ ] HPA/KEDA on queue depth + TTFT; minReplicas never zero for interactive
- [ ] Blue/green or weighted canary; both fleets funded for overlap
- [ ] API/old-SKU overflow until first real peak
- [ ] Drivers/CUDA/engine/weights pinned; mixed-driver groups forbidden
- [ ] Training/OCR/embed cannot schedule onto generation taints
- [ ] Cost: $/1M tokens and reservation impact reviewed after cutover

---

## Metrics to watch

| Metric | Why |
|---|---|
| Queue depth (per pool) | Primary scale-up signal |
| TTFT p95, TPOT p95 | User SLO; distinguishes prefill vs decode binder |
| Tokens/s per GPU | SKU efficiency; input to H-axis math |
| KV cache utilisation + preemption/recompute | VRAM/concurrency ceiling |
| GPU mem used vs `gpu-memory-utilization` | Headroom / OOM risk |
| OOM / CUDA alloc failures | Immediate V or S trigger |
| Prefix-cache hit rate | Can drop after H scale-out |
| GPU SM util | Secondary; high is normal; low + high queue = CPU/IO binder |
| Time-to-ready (provision + load) | Autoscale feasibility |
| Idle GPU-hours, $/1M tokens by pool | Whether the last resize was right |
| Golden-set delta after SKU/quant/TP change | Hidden quality trade |
