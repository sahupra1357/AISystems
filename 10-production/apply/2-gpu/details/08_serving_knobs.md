# 08 — Serving knobs that must change with the GPU

Back to [GPUResizing.md](../GPUResizing.md).

---

When GPU memory, count, or generation changes, **re-derive** these. Copying flags from the old SKU is a usual outage.

| Knob | Why it must change |
|---|---|
| `--gpu-memory-utilization` | 0.90 on 40 GB ≠ 0.90 on 80 GB; leave room for CUDA graphs / fragmentation |
| `--max-num-seqs` / max batch | Ceiling is VRAM − weights, not “what we used on A10G” |
| `--max-model-len` | KV grows linearly; a 128k window on a small card starves concurrency |
| `--tensor-parallel-size` | Must divide hidden size / heads; must match GPU count on the *pod* |
| Quantisation / dtype | FP16 → FP8/AWQ changes quality and KV bytes |
| `--kv-cache-dtype` | FP8 KV is a large concurrency win with a quality check |
| Prefix / automatic prefix caching | Hit rate changes with batch mix after you add replicas |
| Speculative decoding draft model | Extra VRAM; may no longer fit after shrinking SKU |

## Prefix cache vs replica count

More replicas (**H**) *reduce* prefix-cache hit rate because each cache is local unless you run a disaggregated KV store. Scale-out can *worsen* TTFT for shared-system-prompt workloads. Measure cache hit rate after H changes.

## Long-context is a pool, not a flag

Raising `max-model-len` on every replica to 128k can cut concurrency by 8–16×. Route long jobs to a dedicated high-VRAM pool.

## Multi-LoRA

Adapters are small; *active* adapter count and extra KV still matter. A SKU that fit one adapter may OOM at ten concurrent adapters.

## Disaggregated prefill / decode

Prefill pool + decode pool is a *topology* resize: GPUs specialise. Worth it at large scale when prefill and decode fight for SM/bandwidth. Not a first-year move.

## CUDA graphs

Graphs capture extra memory. After a V-axis upgrade, teams raise utilisation to “use the card” and OOM on the first spike. Keep 10–20% headroom.

Sizing math: [04_resources_and_sizing.md](04_resources_and_sizing.md). Gotchas: [13_gotchas.md](13_gotchas.md).
