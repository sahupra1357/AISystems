# 07 — Cloud and Kubernetes mechanics

Back to [GPUResizing.md](../GPUResizing.md).

---

## There is no in-place GPU type change

AWS `modify-instance-attribute` (and equivalents) does not change GPU type. You replace the node / instance / Vertex or SageMaker endpoint variant.

| Strategy | Traffic shift | Risk | Use |
|---|---|---|---|
| **Rolling node replace** (same SKU, count change) | Gradual | Low if readiness gates | H-axis |
| **Blue/green new node group** (SKU or TP change) | Weighted, then flip | Medium; pay for both fleets | V/W-axis |
| **Canary 5–10%** | Shadow or real traffic | Required when quant/TP/engine changes | Any behaviour change |
| **Big-bang stop-the-world** | Fast | Outage | Only if two SKUs cannot run (quota) |

## Scheduling

- NVIDIA device plugin (or cloud equivalent): request `nvidia.com/gpu` as **integers** unless using MIG. Fractional GPU requests without MIG are a lie — the process still sees the whole card unless you isolate.
- **Node groups per SKU and per pool.** Mixing L4 and A100 in one autoscaling group makes the scheduler and the cost model both lie.
- **Taints/tolerations** so training and OCR cannot land on generation nodes.
- **Topology spread** across AZs. GPU capacity often fails in a single AZ.

## MIG vs exclusive vs time-slice

| Mode | Isolation | Latency | Density | Prod interactive LLM |
|---|---|---|---|---|
| **Exclusive** (1 process, 1 GPU) | Strong | Best | Lowest | **Default** |
| **MIG** (A100/H100 partitions) | Hardware; less bandwidth per slice | Good if slice sized for KV | High for small models | OK for embed/rerank/classifier |
| **Time-slice / MPS** | Weak | Tail latency explodes | Highest on paper | **No** |

## Drivers, CUDA, images

- A new SKU (Hopper vs Ampere vs Ada) can require a newer driver. Pin AMI + driver + CUDA + engine + weights hash.
- Mixed node groups with different driver majors will schedule a pod onto a node that cannot run the image. Pin nodeSelector.
- Pre-pull images onto GPU nodes; cold pull dominates time-to-ready.

## Probes, PDB, HPA

- **Readiness:** “weights loaded + CUDA context” (e.g. `/v1/models` after load). Unready until then.
- **Liveness:** a cheap endpoint. A liveness probe that hits a loaded inference path will kill slow-but-healthy pods.
- **`terminationGracePeriodSeconds`** ≥ max allowed decode + drain.
- **PDB** so rolling updates keep N+1.
- **HPA/KEDA on queue depth + TTFT**, never GPU util. Freeze HPA on the *old* Deployment during a SKU flip, or HPA will fight the drain.
- Interactive generation: **`minReplicas` never zero**.

## Quota and inventory

GPU capacity is a **quota and a regional inventory** problem. A resize plan without a fallback region or API overflow is an incident plan. `desired: 12` of a scarce SKU is a wish, not a plan.

YAML sketch: [11_examples.md](11_examples.md).
