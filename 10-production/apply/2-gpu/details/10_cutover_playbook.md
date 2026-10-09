# 10 — Production cutover playbook

Back to [GPUResizing.md](../GPUResizing.md). Code: [11_examples.md](11_examples.md).

---

## Phase A — Diagnose (hours, not a ticket titled “need A100s”)

1. Split metrics **per pool** (generation / embed / rerank / OCR / train).
2. Identify the binder: VRAM OOM, KV utilisation + preemption, queue depth, TTFT, TPOT, SM util, CPU raster, or host disk.
3. Confirm it is not **software**: context bloat, missing prefix cache, wrong `max-num-seqs`, embed job on gen GPUs, tenant without concurrency cap.
4. Write the target: e.g. “p95 TTFT ≤ 800 ms at 40 concurrent 8k-ctx generations, with 20% KV headroom.”

## Phase B — Size the new shape on paper

5. Recompute VRAM (weights + KV at *p95* context × *target* concurrency + headroom).
6. Pick axis: S / H / V / W ([02_decision_tree.md](02_decision_tree.md)).
7. Estimate **tokens/s per GPU** from a similar SKU or a 15-minute benchmark, then `replicas = ceil(peak_tokens_per_s / tokens_per_s_per_gpu)` plus N+1.
8. Cost: `$/hour × 24 × 30` vs `$/1M tokens` vs API overflow. Include overlap during cutover.

## Phase C — Prove it off to the side

9. Stand up a **canary node group** with the new SKU/flags. Do not mutate prod.
10. **Correctness:** golden set at the new quant/TP/engine/CUDA.
11. **Load test** with *production prompt-length histogram*, not 128-token synthetic. Find the concurrency knee (TTFT/TPOT vs `max-num-seqs`).
12. Watch KV preemption, GPU memory, TTFT, tokens/s, error rate, and quality.

## Phase D — Cut over

13. Pin image, engine, CUDA, weights hash, flags.
14. Shift traffic 5% → 25% → 50% → 100% with automatic rollback on TTFT, error rate, or eval drift.
15. Keep API or old-SKU overflow until the new fleet survives a peak.
16. Drain old nodes; only then cancel old reservations / scale old ASG to 0.
17. Recalibrate HPA/KEDA: queue depth, not GPU util. Update runbooks.

## Phase E — After

18. Recompute `$/1M tokens` and `$/task`. If VRAM is half empty at peak, you over-bought — pack or downsize.
19. Record SKU, TP, flags, and the reason in traces / change log.

## Change management

A GPU SKU change is a production deploy: golden set, canary, rollback. Reservations should follow a *stable* SKU — do not convert to 1-year reserved H100s until model size and TP topology have been stable through two releases.

**Overflow is part of resize.** API or a second region is the elastic tier; self-hosted GPUs are the baseline. Circuit-break to overflow when queue exceeds a hard limit.

Checklist: [14_checklist_metrics.md](14_checklist_metrics.md).
