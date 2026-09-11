# 01 — Mental model: what GPU resize actually is

Back to [GPUResizing.md](../GPUResizing.md).

---

On CPUs you often change vCPU/RAM on a running node. On GPUs in production, **the card is the unit of capacity**, and most clouds **cannot hot-swap GPU type or GPU count on a live instance**. A “resize” is almost always:

```
decide new shape → provision new capacity → load model(s) → shift traffic → drain old → terminate
```

That sequence is minutes to tens of minutes, not seconds. Model weights must land in VRAM, CUDA contexts initialise, and KV-cache capacity only exists *after* the replica is warm. Treat every resize as a **controlled fleet replacement**, not a live edit.

## Why GPUs are not CPUs

| CPU service | GPU serving |
|---|---|
| Capacity ≈ cores × clock | Capacity ≈ VRAM pages + memory bandwidth + FLOPS, coupled |
| Utilisation high → add replicas | Utilisation high is *normal* for continuous batching |
| Scale in seconds | Warm replica takes minutes (provision + weight load) |
| Vertical resize often in place | SKU change = new VM / node / endpoint variant |
| Memory is shared RAM | Weights + KV must fit in *this card's* VRAM or you OOM |

## What people call “resize”

Four different operations. Using the wrong word buys the wrong hardware.

| People say | They usually mean | Axis |
|---|---|---|
| “We need more GPUs” | More copies of the same serving unit | **H** |
| “We need bigger GPUs” | More VRAM or a newer generation | **V** |
| “The model doesn’t fit” | More GPUs *inside one replica* (TP) or a larger SKU | **W** or **V** |
| “We’re wasting the cards” | Quantise, cap context, split pools, scale down | **S** or **H down** |

See [03_axes_h_v_w_s.md](03_axes_h_v_w_s.md).

## Implications that follow from the mental model

1. **Readiness is “weights in VRAM,” not “process started.”** Load balancers must wait.
2. **Drain time is max decode time**, not HTTP idle timeout.
3. **You pay for overlap** during any SKU change — old fleet + new fleet.
4. **Autoscaling is predictive or heavily buffered.** GPU VMs are not Lambda.
5. **A SKU/quant/TP change is a behaviour change.** Treat it like a model deploy.

Next: [02_decision_tree.md](02_decision_tree.md).
