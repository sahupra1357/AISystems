# 06 — Considerations behind every resize

Back to [GPUResizing.md](../GPUResizing.md).

---

## Product / SLO

- **Which SLO is failing?** TTFT, TPOT (inter-token), E2E p95, ingest lag, train job wall time — each maps to a different axis ([02_decision_tree.md](02_decision_tree.md)).
- **Traffic class:** interactive vs batch vs overnight eval. Batch should never steal interactive GPUs.
- **Context distribution:** p50 vs p95 prompt length. Resizing for p99 context on every replica is how fleets 3× in cost. Prefer a **long-context pool** plus a default pool with a hard `max-model-len`.
- **Quality coupling:** changing SKU often changes quantisation, kernel, or TP degree — that is a **model-behaviour change**. Gate with a golden set, not just a load test.

## Fit and serving config (must change with the SKU)

Copying old flags onto a new card is a common outage. See [08_serving_knobs.md](08_serving_knobs.md).

## Parallelism topology

- Prefer **data-parallel replicas** (H) when the model fits one GPU.
- **TP** only when weights + KV at target concurrency do not fit.
- TP=8 implies NVLink/NVSwitch class hardware, not eight PCIe cards in a pod.

Detail: [03_axes_h_v_w_s.md](03_axes_h_v_w_s.md).

## Time constants

| Event | Typical time | Implication |
|---|---|---|
| Cloud GPU VM provision | 2–15+ min (or *no capacity*) | Predictive / scheduled scale; large warm buffer |
| Image pull + driver | 1–5 min | Pre-bake AMIs / cache images on GPU nodes |
| Model load to VRAM | 30 s–10 min | Do not count a replica in LB until `/ready` after load |
| KV / prefix warm | seconds–minutes of traffic | Fresh replicas have worse TTFT until cache warms |
| Drain in-flight generations | up to max decode time | `terminationGracePeriod` ≥ longest allowed output |

**Autoscale on queue depth and TTFT, never on GPU utilisation.** Scale-down only when queue is empty *and* utilisation is low *and* you stay above `minReplicas` for SLO + N+1.

## Cost and procurement

- GPUs bill **whether you have traffic or not** (hourly on-demand or committed).
- **Committed use / reservations lock SKU.** A vertical resize can strand them. Model the remaining term.
- **Spot / preemptible:** embed backfill, OCR, training. Not interactive generation unless instant on-demand overflow exists.
- Compare **$/1M tokens** (or $/1k pages) *after* the new packing, not $/GPU-hour. A 2× more expensive GPU that does 3× tokens/s is a win; a 2× GPU that only buys unused VRAM is not.
- Include **idle, on-call, and cutover overlap** in TCO.

## Safety, tenancy, environment

- **Staging GPU SKUs should match serving prod** or you will not predict OOM. Cheaper SKU in staging is OK for functional tests, not capacity sign-off.
- **Multi-tenant isolation:** one noisy tenant with long contexts fills KV. Per-tenant concurrency is often a better fix than a bigger GPU.
- **Weights are sensitive.** New node groups need the same egress, IAM, and disk encryption as the old.

Cloud/K8s specifics: [07_cloud_kubernetes.md](07_cloud_kubernetes.md).
