# 04 — Three resources and the sizing math

Back to [GPUResizing.md](../GPUResizing.md).

---

A production GPU replica is limited by three coupled resources. **Resize to the one that is actually binding.**

| Resource | Dominated by | Failure mode | User-visible effect |
|---|---|---|---|
| **VRAM** | Weights + KV + activations + fragmentation | OOM, KV preemption, refused seqs | 503, truncated context, recompute spikes |
| **Memory bandwidth** | Decode (token-by-token), KV read | Tokens/s ceiling | Slow streaming, high TPOT |
| **Compute (FLOPS)** | Prefill, embeddings, rerank, OCR, training | Queue of prefill-heavy jobs | High TTFT, ingest backlog |

Buying H100s to fix a KV problem caused by 32k RAG prompts is often worse than capping context, enabling prefix cache, or splitting long jobs to a dedicated pool. Buying more T4s to fix a 70B model that does not fit is wasted — you need V or W.

## VRAM equation

```
VRAM ≈ weights + KV cache + activations + overhead

weights   ≈ params × bytes_per_param
            70B @ FP16 = 140 GB; 70B @ AWQ-4bit ≈ 40 GB
            8B @ FP16 ≈ 16 GB; 8B @ 4-bit ≈ 4 GB

KV cache  ≈ 2 × layers × kv_heads × head_dim × bytes × context_len × concurrent_seqs
            (K and V; bytes = 2 for FP16 KV, 1 for FP8 KV)

activations ≈ ~1–2 GB per concurrent batch (model dependent)
overhead    = CUDA graphs, allocator fragmentation, engine workspace
headroom    = leave 10–20% (do not run gpu-memory-utilization at 0.98)
```

At production concurrency, **KV usually dominates**. It scales linearly with **both** context length and concurrent sequences — which is why long-context RAG is expensive to self-host.

**GQA / MLA:** KV size uses `kv_heads`, not query heads. A 70B with GQA is cheaper to serve than the parameter count suggests. Do not size KV from param count alone.

## Concurrency from leftover VRAM

```
usable = VRAM × headroom − weights − overhead
max_seqs ≈ usable / kv_per_sequence(context)
```

If `max_seqs` at your p95 context is below the SLO concurrency, you are VRAM/KV-bound: **S** (shorter ctx, quant, FP8 KV) or **V** (more VRAM), not more tiny replicas of the same SKU.

If `max_seqs` is fine but tokens/s per GPU cannot clear the queue: **H** or a **newer-gen V**.

## Throughput to replica count

```
replicas ≈ ceil(peak_output_tokens_per_s / measured_tokens_per_s_per_gpu) + N+1
```

Measure tokens/s on the **real prompt-length histogram**, not 128-token synthetic prompts. Prefill-heavy RAG traffic has a different knee than short chat.

Worked functions: [11_examples.md](11_examples.md).
