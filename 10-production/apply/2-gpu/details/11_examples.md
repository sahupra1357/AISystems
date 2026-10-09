# 11 — Examples (sizing, axis choice, K8s)

Back to [GPUResizing.md](../GPUResizing.md). Playbook: [10_cutover_playbook.md](10_cutover_playbook.md).

---

```python
# --- 1) Binding resource: what would a resize even fix? ---
def binder(m: dict) -> str:
    """m = live metrics for ONE pool (generation XOR embed XOR ocr)."""
    if m["oom_events"] > 0 or m["kv_preempt_per_min"] > 1:
        return "vram_or_kv"          # S (ctx/seqs/quant) or V (more VRAM) or W (TP)
    if m["queue_depth"] > 8 and m["ttft_p95_ms"] > m["ttft_slo_ms"]:
        if m["tokens_per_s_per_gpu"] < m["expected_tokens_per_s"] * 0.5:
            return "slow_sku"        # V (newer gen / bandwidth) after confirming config
        return "need_replicas"       # H
    if m["gpu_util"] < 0.3 and m["kv_util"] < 0.3 and m["replicas"] > m["min_replicas"]:
        return "overprovisioned"     # scale H down
    return "hold"


# --- 2) VRAM → concurrency ---
def kv_bytes(layers, kv_heads, head_dim, ctx, seqs, kv_b=2) -> float:
    return 2 * layers * kv_heads * head_dim * kv_b * ctx * seqs  # K+V

def shape_ok(vram_gb, weights_gb, layers, kv_heads, head_dim, ctx, seqs, headroom=0.85, overhead_gb=2):
    kv_gb = kv_bytes(layers, kv_heads, head_dim, ctx, seqs) / 1e9
    used = weights_gb + kv_gb + overhead_gb
    return used <= vram_gb * headroom, {"used_gb": round(used, 1), "kv_gb": round(kv_gb, 1)}

# Llama-3.1-8B FP16 ~16 GB weights. Concurrent 64 @ 8k ctx on 24 GB vs 40 GB vs 80 GB:
# shape_ok(24, 16, 32, 8, 128, 8192, 64)  → False  → need quant, fewer seqs, or V
# shape_ok(40, 16, 32, 8, 128, 8192, 64)  → True   → H on this SKU
# 70B AWQ ~40 GB weights: single 40 GB card has almost no KV → V to 80 GB or W (TP=2)


# --- 3) Pick axis ---
def resize_plan(fit_current, fit_quantized, weights_gt_one_card, binder_id, peak_tps, tps_per_gpu):
    if binder_id == "overprovisioned":
        return {"axis": "H", "delta": "scale_down"}
    if not fit_current and fit_quantized:
        return {"axis": "S", "action": "quantise_or_cap_ctx_and_re_eval"}
    if weights_gt_one_card:
        return {"axis": "W", "action": "tensor_parallel", "tp": 2}  # or 4/8; prefer min TP that fits
    if binder_id == "need_replicas":
        return {"axis": "H", "replicas": -(-peak_tps // tps_per_gpu) + 1}  # ceil + N+1
    if binder_id in {"vram_or_kv", "slow_sku"}:
        return {"axis": "V", "action": "next_sku_same_or_less_TP"}
    return {"axis": "hold"}


# --- 4) Autoscale signal (thresholds are SKU-specific) ---
def hpa_signal(queue_depth, ttft_p95, slo_ms, q_target=4):
    if queue_depth > q_target * 2 or ttft_p95 > slo_ms:
        return "up"
    if queue_depth < 1:
        return "down_candidate"  # still require cooldown + minReplicas + not during peak
    return "hold"


# --- 5) Cutover gates ---
def canary_abort(canary, baseline, eval_delta, ttft_budget=1.15, err_budget=1.5):
    if canary["ttft_p95"] > baseline["ttft_p95"] * ttft_budget:
        return "abort_latency"
    if canary["error_rate"] > max(baseline["error_rate"] * err_budget, 0.01):
        return "abort_errors"
    if eval_delta["golden_drop"] > 0.02:
        return "abort_quality"
    return "continue"
```

Kubernetes sketch (generation pool exclusive GPU; resize = new node group + new deployment):

```yaml
# Do not patch nvidia.com/gpu from 1 → 2 on a live Deployment of a TP=1 model
# and expect a resize. That schedules a new pod; TP flags must match.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-gen-llama8b
spec:
  replicas: 3                          # H-axis; PDB minAvailable: 2
  template:
    spec:
      nodeSelector:
        pool: gpu-gen-l4               # SKU-pinned node group (V-axis = new pool)
      terminationGracePeriodSeconds: 180
      containers:
        - name: vllm
          resources:
            limits:
              nvidia.com/gpu: 1        # W-axis: 2 only if --tensor-parallel-size=2
          readinessProbe:
            httpGet: { path: /v1/models, port: 8000 }
            failureThreshold: 3
            periodSeconds: 10          # fail unready until weights are in VRAM
```
