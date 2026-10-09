# 09 — Trade-off matrices

Back to [GPUResizing.md](../GPUResizing.md).

---

## Horizontal vs vertical vs width vs software

| Move | Wins | Loses | Best when |
|---|---|---|---|
| **S — Pack better** (quant, ctx cap, prefix cache, FP8 KV, split pools) | $ down, often latency down | Quality risk; eng time; eval cost | Always first |
| **H — More same GPUs** | Linear-ish throughput; simple; good HA | Idle $; still cannot fit a too-big model | SLO miss, model fits, tokens/s/GPU OK |
| **V — Bigger/newer SKU** | Fit, concurrency, TTFT, future headroom | $; reservation lock-in; driver/CUDA churn; capacity scarcity | OOM/KV-bound or decode-bound on current gen |
| **W — TP > 1** | Serve models that do not fit | NCCL tax; worse blast radius; slower rolling updates | Weights + KV > one card after quant |
| **API overflow / stay on API** | Elasticity, no GPU ops | Unit $ at high volume; residency | Burst, frontier quality, small team |

## One large GPU vs many small

| | One 80 GB (A100/H100) | Several 24 GB (L4/A10G) |
|---|---|---|
| **Fits** | 70B 4-bit + real concurrency; long context | 7–13B (or 70B only with TP or heavy quant) |
| **Packing** | Can MIG-split leftover for embed/rerank | Simple 1:1 model↔card |
| **Failure** | One card down is a large capacity hole | Many small holes; easier N+1 |
| **Cost curve** | High idle cost | Better for many small models |
| **Bandwidth** | High; good decode | Ada L4 is strong $ / inference; T4 is usually a trap for LLM decode |

**Practical default:** L4/A10G for embed + rerank + 8B fast tier; A100/H100 only for the generation model that needs it. Do not put embeddings on H100s.

## MIG vs exclusive vs time-slice

See [07_cloud_kubernetes.md](07_cloud_kubernetes.md). Short: exclusive for interactive LLM; MIG for small models; time-slice not for chat.

## Warm capacity vs autoscale-from-zero

| | Keep warm minReplicas | Scale to zero |
|---|---|---|
| **TTFT after idle** | SLO-safe | Minutes of ice-cold load |
| **Cost** | Pay for idle | Attractive for nights/weekends |
| **Use** | User-facing generation | Batch pools, internal tools, staging |

## Headroom

- **VRAM 10–20%** after weights + target KV. `gpu-memory-utilization` 0.98 turns fragmentation into random OOM.
- **Replica N+1** (or N+2 across AZs) so one death or one rolling update does not breach SLO.
- **Regional GPU quota headroom.** The limiter is often not your Helm chart.

## Paying for two fleets

During V/W you **pay for both fleets**. Budget the overlap window. Drain with connection draining longer than max decode. Do not cancel old reservations until the new fleet survives a real peak.
