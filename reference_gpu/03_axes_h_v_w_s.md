# 03 — Four axes: S, H, V, W

Back to [GPUResizing.md](../GPUResizing.md).

---

**Default order: S → H → V → W.**

## S — Software packing

Change how work uses the *same* metal.

| Lever | Effect | Risk |
|---|---|---|
| Quantisation (AWQ/GPTQ/FP8/INT8) | 2–4× less weight VRAM; more room for KV | Quality; must eval |
| FP8 / lower-precision KV | Large concurrency win | Quality on long context |
| Lower `max-model-len` | Linear drop in KV per seq | Product: refuse or route long jobs |
| Lower `max-num-seqs` | Stops preemption; can *improve* TTFT | Throughput cap |
| Prefix / automatic prefix caching | Prefill and KV reuse | Hit rate drops if prompts are unique |
| Split pools (embed/rerank/OCR off gen) | Stops SLO collisions | More node groups |
| Multi-LoRA vs many full models | One base + adapters | Active adapter count still costs VRAM |
| MIG slices for small models | Density on A100/H100 leftovers | Not for bandwidth-hungry decode |

Exhaust S before a purchase. Flags: [08_serving_knobs.md](08_serving_knobs.md).

## H — Horizontal (more identical replicas)

Add (or remove) copies of the same GPU SKU and the same serving unit (usually TP=1).

- **Wins:** throughput roughly linear; blast radius small; rolling replace is ordinary.
- **Loses:** idle dollars; prefix-cache hit rate can fall (each replica has its own cache); does not help if one replica cannot fit the model.
- **Autoscale:** queue depth and TTFT, plus a warm `minReplicas`. Never scale interactive generation to zero.

N+1 (or N+2 across AZs) is part of H, not optional spare.

## V — Vertical (different SKU)

Change GPU generation and/or VRAM: T4 → L4/A10G → A100 40/80 → H100/H200/B200.

Two different vertical buys:

| Buy | What you get | When |
|---|---|---|
| **More VRAM** (40 → 80 GB, same-ish gen) | More KV and/or a larger model at TP=1 | OOM, preemption, concurrency shortfall |
| **Newer gen, similar VRAM** (A10 → L4, A100 → H100) | Bandwidth + FLOPS → tokens/s, TTFT | Decode/prefill slow at the concurrency knee |

V implies a **new node group**, new drivers possibly, new flags, overlap cost, and often reservation churn. Treat as a production deploy.

## W — Width (more GPUs per replica)

Tensor / pipeline / expert parallelism so one *request* spans N GPUs.

| Mode | Role in serving | Tax |
|---|---|---|
| **Tensor parallel (TP)** | Split layers across GPUs when weights do not fit | NCCL on every layer; needs NVLink for large TP |
| **Pipeline parallel (PP)** | Stage layers; rare for latency-sensitive serving | Pipeline bubbles |
| **Expert parallel (EP)** | MoE expert placement | Routing imbalance |
| **Data parallel** | This is H, not W | None in the request path |

**Prefer 8× one-GPU replicas over 1× eight-GPU replica** unless the model cannot fit. TP a model that already fits “for speed” is often slower and always more expensive.

TP size must match `nvidia.com/gpu` on the *pod*, divide model heads correctly, and live on a SKU family with the interconnect you assumed.

Trade-off tables: [09_trade_offs.md](09_trade_offs.md).
