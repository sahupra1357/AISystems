# GPU resizing for production AI applications

A standalone playbook: how to approach GPU capacity changes, what to consider, and the trade-offs. Start here. Finer details live in [`details/`](details/).

This playbook is separate from [`AI_System.md`](../../../AI_System.md) and covers only GPU capacity.

---

## 1. The point

On CPUs you often change vCPU/RAM on a running node. On GPUs in production, **the card is the unit of capacity**, and most clouds **cannot hot-swap GPU type or GPU count on a live instance**. A “resize” is almost always:

```
decide new shape → provision new capacity → load model(s) → shift traffic → drain old → terminate
```

That sequence is minutes to tens of minutes, not seconds. Model weights must land in VRAM, CUDA contexts initialise, and KV-cache capacity only exists *after* the replica is warm. Treat every resize as a **controlled fleet replacement**, not a live edit.

Four axes people lump together as “resize.” They are not interchangeable:

| Axis | What you change | Typical reason | Detail |
|---|---|---|---|
| **S — Software packing** | Quantisation, max context, max seqs, batch, MIG | Fit more work on the same metal | [03_axes](details/03_axes_h_v_w_s.md) |
| **H — Horizontal** | Replica / node count of the *same* GPU SKU | Traffic volume, availability | [03_axes](details/03_axes_h_v_w_s.md) |
| **V — Vertical (SKU)** | GPU generation / VRAM | Model no longer fits; decode too slow; KV too small | [03_axes](details/03_axes_h_v_w_s.md) |
| **W — Width (parallelism)** | GPUs *per replica* (tensor / pipeline / expert parallel) | Weights + KV exceed one card | [03_axes](details/03_axes_h_v_w_s.md) |

**Default order in production: exhaust S, then H, then V, then W.** Bigger/newer GPUs and multi-GPU replicas are expensive and sticky. Software packing and more identical replicas are reversible.

---

## 2. How to approach it

Ask in this order. Stop at the first “yes.” Full tree: [02_decision_tree](details/02_decision_tree.md).

```
1. Is the model / quantisation / max context the real constraint?
   YES → change S. Do not buy a bigger GPU yet.
2. Is latency SLO breached while queue depth is growing, and tokens/s per GPU is healthy?
   YES → add H (same SKU) if the model still fits.
3. Are you OOM, KV-preempting, or unable to reach target concurrency on this SKU?
   YES → V (more VRAM / bandwidth) or S (smaller KV).
4. Do weights + KV at target concurrency exceed the largest single card you can get?
   YES → W (tensor parallel), shrink the model / context, or keep that model on an API.
5. Is utilisation low and cost high?
   YES → scale H down, consolidate pools, or stop self-hosting that workload.
```

**Do not resize because GPU utilisation is high.** vLLM sitting at 95% SM util can be healthy. A GPU at 60% util with KV full and TTFT climbing is already failing.

A production GPU replica is limited by **three coupled resources** — VRAM, memory bandwidth, and FLOPS. Resize to the one that is actually binding. See [01_mental_model](details/01_mental_model.md) and [04_resources_and_sizing](details/04_resources_and_sizing.md).

Rule of thumb:

```
VRAM ≈ weights + KV(context × concurrent seqs) + activations + 10–20% headroom
```

At production concurrency, **KV usually dominates**. Vertical resize (more VRAM) buys concurrency and/or context. A newer generation at similar VRAM buys tokens/s and TTFT. They are different purchases.

---

## 3. Considerations

Never resize “the GPU fleet” as one thing. A prod AI app is several **pools** with different shapes — generation, embeddings, rerank, OCR/ingest, training. Co-locating them looks efficient and destroys SLOs. Detail: [05_workload_pools](details/05_workload_pools.md).

Before you change a SKU, walk this list (full notes: [06_considerations](details/06_considerations.md)):

| Area | What to decide |
|---|---|
| **SLO** | Which number is failing — TTFT, TPOT, E2E p95, ingest lag, train wall time? Each maps to a different axis. |
| **Context mix** | Size for p95 context, not p99 on every replica. Long context is a *pool*, not a flag. |
| **Quality** | SKU / quant / TP / engine changes behaviour. Gate with a golden set, not only a load test. |
| **Serving flags** | `max-num-seqs`, `max-model-len`, `gpu-memory-utilization`, TP size, KV dtype must be *re-derived* for the new card. |
| **Topology** | Prefer many one-GPU replicas over one wide TP replica unless the model cannot fit. |
| **Time constants** | VM provision 2–15+ min; model load 30 s–10 min. Autoscale from queue + TTFT, never GPU util. Interactive generation: never scale to zero. |
| **Cost** | Compare `$/1M tokens` after packing, not `$/GPU-hour`. Reservations lock SKU. Overlap during cutover means you pay for two fleets. |
| **Isolation** | Staging SKU must predict prod VRAM. Per-tenant concurrency caps often beat a bigger GPU. |

Cloud/K8s mechanics (no in-place GPU type change, MIG vs exclusive, drivers): [07_cloud_kubernetes](details/07_cloud_kubernetes.md). Serving knobs: [08_serving_knobs](details/08_serving_knobs.md).

---

## 4. Trade-offs

Full matrices: [09_trade_offs](details/09_trade_offs.md).

| Move | Wins | Loses | Best when |
|---|---|---|---|
| **S — Pack better** | $ down, often latency down | Quality risk; eval cost | Always first |
| **H — More same GPUs** | Linear-ish throughput; simple HA | Idle $; cannot fit a too-big model | SLO miss, model fits, tokens/s/GPU OK |
| **V — Bigger/newer SKU** | Fit, concurrency, TTFT | $, reservation lock-in, driver churn, scarcity | OOM/KV-bound or decode-bound on current gen |
| **W — TP > 1** | Serve models that do not fit | NCCL tax; worse blast radius | Weights + KV > one card after quant |
| **API overflow** | Elasticity, no GPU ops | Unit $ at high volume; residency | Burst, frontier quality, small team |

Other forks you will hit:

- **One 80 GB card vs several 24 GB cards** — large cards for 70B + KV; small cards for 8B, embed, rerank. Do not put embeddings on H100s.
- **Exclusive vs MIG vs time-slice** — exclusive for interactive generation; MIG for small models on leftover slices; time-slice is for batch/dev, not chat.
- **Warm minReplicas vs scale-to-zero** — chat stays warm; OCR/embed may scale down.
- **Rolling vs blue/green** — H-axis can roll; V/W is a new node group with overlap cost.

---

## 5. Cutover (short)

Detail and code: [10_cutover_playbook](details/10_cutover_playbook.md), [11_examples](details/11_examples.md).

1. **Diagnose** the binder *per pool* (OOM/KV vs queue vs TPOT vs idle).
2. **Size on paper** — VRAM math, axis, replica count + N+1, cost including overlap.
3. **Prove off to the side** — canary node group, golden set, load test on the *production prompt-length histogram*.
4. **Shift traffic** 5% → 25% → 50% → 100% with abort on TTFT, errors, eval drop. Keep API/old-SKU overflow through the first peak.
5. **Drain** the old fleet; only then drop reservations. Recalibrate HPA on queue depth. Recheck `$/1M tokens`.

Worked patterns (hybrid 8B, 70B residency, OCR spike, training isolation, overnight scale-down): [12_worked_patterns](details/12_worked_patterns.md).

---

## 6. When to resize vs not

Resize when the model **does not fit** at required context × concurrency after packing; when **queue + TTFT** grow at the measured tokens/s knee; when **TPOT** misses SLO on an old generation; when weights exceed one card; or when KV util is low and you are overpaying.

Do **not** resize to fix quality, because GPU util is high, because one tenant ran 100k-token jobs (cap them), to absorb ingest/eval/train on the chat pool, or before caching/tiering/context trim if the goal is cost.

---

## 7. Finer details index

| File | What is in it |
|---|---|
| [01_mental_model.md](details/01_mental_model.md) | What a resize is; why GPUs are not CPUs |
| [02_decision_tree.md](details/02_decision_tree.md) | Step-by-step “whether / which axis” |
| [03_axes_h_v_w_s.md](details/03_axes_h_v_w_s.md) | Horizontal, vertical, width, software packing |
| [04_resources_and_sizing.md](details/04_resources_and_sizing.md) | VRAM / bandwidth / FLOPS; KV math |
| [05_workload_pools.md](details/05_workload_pools.md) | Gen vs embed vs rerank vs OCR vs train |
| [06_considerations.md](details/06_considerations.md) | SLO, flags, topology, time, cost, tenancy |
| [07_cloud_kubernetes.md](details/07_cloud_kubernetes.md) | Node groups, MIG, drivers, probes, HPA |
| [08_serving_knobs.md](details/08_serving_knobs.md) | vLLM flags, prefix cache, GQA, multi-LoRA |
| [09_trade_offs.md](details/09_trade_offs.md) | Decision matrices |
| [10_cutover_playbook.md](details/10_cutover_playbook.md) | Diagnose → canary → flip → FinOps |
| [11_examples.md](details/11_examples.md) | Python sizing + Kubernetes sketch |
| [12_worked_patterns.md](details/12_worked_patterns.md) | Common prod AI shapes |
| [13_gotchas.md](details/13_gotchas.md) | Details that cause outages |
| [14_checklist_metrics.md](details/14_checklist_metrics.md) | Prod checklist and metrics |

---

## Video takeaway (30 seconds)

GPU resize is not an instance-type dropdown. You replace a fleet: new nodes, load weights, shift traffic, drain. First figure out what is binding — KV and VRAM, decode bandwidth, or just not enough replicas — because those three buy three different SKUs. Exhaust packing: quantise, cap context, split embeddings and OCR off the chat GPUs, and cap noisy tenants. Then add identical replicas. Only then buy more VRAM or tensor-parallel. Never autoscale on GPU utilisation; autoscale on queue and TTFT, keep a warm minimum, and treat a SKU change like a model deploy: golden set, canary, overflow, rollback.
