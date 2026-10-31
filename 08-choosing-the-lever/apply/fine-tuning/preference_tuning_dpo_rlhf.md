# Fine-tuning → Preference tuning (DPO / RLHF)

![DPO Preference Tuning Explained: What SFT Can't Teach](demo/preference-tuning-dpo-rlhf-demo.gif)

*([full-resolution MP4 with audio](demo/preference-tuning-dpo-rlhf-demo.mp4) — 51s · [watch on YouTube](https://youtu.be/DHXTkArvxDc))*

## What it is

SFT teaches the model to imitate good outputs. It cannot teach *relative* quality: that answer A is better than answer B when both are plausible. Many qualities are only expressible as comparisons — appropriately concise, correctly hedged, right level of detail, house tone.

Preference tuning learns from pairs:

| Method | Mechanism | Infrastructure |
|---|---|---|
| **RLHF** (PPO) | Train a reward model on preferences, then optimise the policy against it with RL | Reward model + RL loop + KL control; complex, unstable |
| **DPO** (Direct Preference Optimization) | Optimise the policy directly on preference pairs; the reward model is implicit | A single training run, like SFT |
| **ORPO / KTO / SimPO** | Variants: ORPO merges SFT and preference in one stage; KTO needs only binary good/bad, not pairs | Simpler data requirements |

DPO is the practical default: same tooling as SFT, one loss function, no separate reward model, no RL instability. Its objective increases the likelihood of chosen responses relative to rejected ones while a KL term keeps the model near a reference policy (usually the SFT checkpoint) to prevent drift.

**Order matters**: SFT first (teach the format and task), then DPO (shape the preference). Running DPO on a base model that hasn't learned the task produces a model that is preferred but wrong.

## How to implement

1. **Do SFT first** ([SFT](sft.md)) and use that checkpoint as both the starting point and the reference model.
2. **Collect preference pairs** from: human comparisons of two model outputs, production edits (original = rejected, edited = chosen — the highest-value source), thumbs with an alternative, and LLM-judge pairwise comparisons for scale ([LLM-as-judge](../../../09-evaluation/apply/llm_as_judge.md), with calibration).
3. **Curate**: pairs must differ on the dimension you care about, not on correctness (correctness belongs in SFT). Remove pairs where the "rejected" is factually wrong — otherwise you teach preference where you meant to teach accuracy.
4. **Train with DPO**: beta 0.1–0.5 (higher = stay closer to the reference), low learning rate (5e-6 to 5e-5), 1–2 epochs. Preference tuning overfits faster than SFT.
5. **Monitor reward margin and KL divergence**: a growing margin with controlled KL is healthy; exploding KL means the model is drifting into degenerate outputs.
6. **Evaluate with pairwise comparison** against the SFT checkpoint ([LLM-as-judge](../../../09-evaluation/apply/llm_as_judge.md)) plus the full quality and safety suite.
7. **Watch for the classic failures**: verbosity inflation, sycophancy, and eroded refusal behaviour.

## Small example

```python
from trl import DPOTrainer, DPOConfig
from peft import LoraConfig

# ---- 1. Preference pairs: production edits are the best source ----
def pairs_from_edits(interactions) -> list[dict]:
    """User edited the draft before sending → original is 'rejected', final is 'chosen'."""
    pairs = []
    for i in interactions:
        if not i.get("edited_before_use") or i["edit_distance"] < 0.05:
            continue                                        # unchanged: no preference signal
        if i["edit_distance"] > 0.8:
            continue                                        # rewritten from scratch: likely a correctness issue (→ SFT)
        pairs.append({"prompt": i["prompt"], "chosen": i["final_text"], "rejected": i["model_output"],
                      "meta": {"source": "user_edit", "distance": i["edit_distance"]}})
    return pairs

def curate_pairs(pairs) -> list[dict]:
    kept, dropped = [], Counter()
    for p in pairs:
        if factually_wrong(p["rejected"]):                  # correctness belongs in SFT, not preference
            dropped["rejected_is_incorrect"] += 1; continue
        if similarity(p["chosen"], p["rejected"]) > 0.97:
            dropped["indistinguishable"] += 1; continue
        if len(p["chosen"]) > 2.5 * len(p["rejected"]):
            dropped["length_confound"] += 1; continue       # otherwise you teach "longer is better"
        kept.append(p)
    return {"pairs": kept, "dropped": dict(dropped)}

# ---- 2. DPO training, starting from the SFT checkpoint ----
"""
trainer = DPOTrainer(
    model=sft_model,                      # policy: the SFT checkpoint
    ref_model=None,                       # with PEFT, the reference is the adapter-disabled base
    args=DPOConfig(output_dir="out/dpo-tone-v1",
                   beta=0.1,              # KL strength: higher keeps closer to the reference
                   learning_rate=5e-6, num_train_epochs=1,
                   per_device_train_batch_size=2, gradient_accumulation_steps=8,
                   max_length=2048, max_prompt_length=1024,
                   logging_steps=10, eval_strategy="steps", eval_steps=100),
    train_dataset=pref_train, eval_dataset=pref_eval,
    peft_config=LoraConfig(r=16, lora_alpha=32, task_type="CAUSAL_LM",
                           target_modules=["q_proj","k_proj","v_proj","o_proj"]),
)
trainer.train()
"""

# ---- 3. Health checks during and after training ----
def training_health(logs):
    margins = [l["rewards/margins"] for l in logs if "rewards/margins" in l]
    kls = [l["rewards/chosen"] - l["rewards/rejected"] for l in logs if "rewards/chosen" in l]
    acc = [l["rewards/accuracies"] for l in logs if "rewards/accuracies" in l]
    warnings = []
    if margins and margins[-1] < 0.05:   warnings.append("margin not growing — pairs may be indistinguishable")
    if acc and acc[-1] > 0.95:           warnings.append("accuracy saturated — likely overfitting; stop early")
    return {"final_margin": margins[-1] if margins else None,
            "preference_accuracy": acc[-1] if acc else None, "warnings": warnings}

def evaluate_preference_tuning(sft_model, dpo_model, cases, safety_cases):
    wins = sum(judge_pairwise(c.query, dpo_model(c), sft_model(c), c.reference) == "A" for c in cases)
    lens = {"sft": mean(len(sft_model(c).split()) for c in cases),
            "dpo": mean(len(dpo_model(c).split()) for c in cases)}
    safety = {"sft": behaviour_score(sft_model, safety_cases), "dpo": behaviour_score(dpo_model, safety_cases)}
    return {
        "win_rate_vs_sft": wins / len(cases),                          # target > 0.6 to justify shipping
        "verbosity_inflation": lens["dpo"] / lens["sft"],              # > 1.3 → you taught "longer = better"
        "safety_delta": safety["dpo"] - safety["sft"],                 # must be ~0
        "sycophancy_check": sycophancy_probe(dpo_model),               # agrees with a wrong user premise?
    }
```

## Pros

- **Captures qualities SFT cannot express**: tone, appropriate length, hedging, level of detail.
- **DPO is operationally simple** — one training run with the same tooling as SFT, no reward model, no RL.
- **Production edits are free preference data** and are the most authentic signal you can get ([online eval](../../../09-evaluation/apply/online_eval.md)).
- **Measurable via pairwise win rate**, which is more reliable than absolute scoring ([LLM-as-judge](../../../09-evaluation/apply/llm_as_judge.md)).
- Works with LoRA adapters, so it's cheap ([PEFT / LoRA / QLoRA](peft_lora_qlora.md)).

## Cons

- **Verbosity inflation is the classic failure**: preference data subtly favours longer answers and the model learns "longer = better". Must be measured and controlled.
- **Sycophancy**: the model learns to agree with users rather than be correct.
- **Safety erosion**: refusal behaviour can weaken; explicit evaluation required.
- **Preference data is expensive** if human-collected, and LLM-judge pairs inherit the judge's biases ([LLM-as-judge](../../../09-evaluation/apply/llm_as_judge.md)).
- **Overfits fast** — one epoch is often enough; two can degrade.
- **Confounded pairs teach the wrong thing** (length, formatting, or correctness rather than the intended dimension).

## When to use

- Subjective quality matters and SFT has plateaued: tone, conciseness, helpfulness, appropriate hedging.
- You have a natural preference signal — especially user edits in a drafting product.
- After SFT, never instead of it.

## When NOT to use

- To fix correctness — that's SFT, retrieval, or grounding ([generation & grounding](../../../06-rag/apply/8-generation-grounding/)), not preference.
- Without pairwise evaluation infrastructure — you won't be able to tell whether it helped.
- On small or noisy preference sets: a few hundred confounded pairs will teach length and formatting artefacts.
- When the base model's tone is already acceptable; the gain may not justify the risk of safety erosion.

## Production at scale

**Production edits are the highest-value preference source.** In a drafting product, the model's draft and the user's sent version form a natural pair, with edit distance as an intensity signal ([online eval](../../../09-evaluation/apply/online_eval.md)). Filter out heavy rewrites — those signal correctness problems that belong in SFT.

**Control for length explicitly.** Drop pairs where the chosen response is much longer, and measure verbosity inflation after training. If mean output length rises more than ~20–30%, you taught length, not quality.

**Run a sycophancy probe**: cases where the user asserts something false and the correct behaviour is polite disagreement. Preference tuning is known to erode this.

**Safety and refusal evaluation is mandatory** after preference tuning, gated in CI ([the testing pyramid](../../../09-evaluation/apply/testing_pyramid.md)). This is the failure that reaches users as a security issue rather than a quality issue.

**Beta controls drift.** Low beta (0.05–0.1) allows more change and more risk; higher beta (0.3–0.5) keeps the model near the SFT reference. Start high and lower it only if the win rate is insufficient.

**One epoch, usually.** Monitor preference accuracy: once it saturates above ~0.9, further training degrades general capability.

**Pairwise win rate against the SFT checkpoint** is the ship criterion — target a clear majority (>60%) with no safety regression and controlled verbosity ([LLM-as-judge](../../../09-evaluation/apply/llm_as_judge.md), [statistical hygiene](../../../09-evaluation/apply/statistical_hygiene.md)).

**Consider simpler alternatives first**: a better system prompt often achieves tone goals with no training, no risk, and instant rollback ([prompting vs RAG vs fine-tuning](../prompting_vs_rag_vs_finetuning.md)).

## Prod checklist

- [ ] SFT completed first; its checkpoint used as start point and reference
- [ ] Preference pairs curated: differ on the target dimension, not on correctness; length confounds removed
- [ ] Production edits mined as pairs with distance filters
- [ ] DPO with beta tuned; 1 epoch default; low learning rate
- [ ] Training health monitored (reward margin, preference accuracy saturation)
- [ ] Pairwise win rate vs SFT measured on held-out cases
- [ ] Verbosity inflation measured and bounded
- [ ] Sycophancy probe and safety/refusal suite run and gated
- [ ] Prompt-only alternative evaluated before committing to training

## Metrics to watch

| Metric | Why |
|---|---|
| Pairwise win rate vs SFT checkpoint | The ship criterion |
| Mean output length ratio (DPO/SFT) | Verbosity inflation |
| Safety and abstention behaviour delta | Erosion check |
| Sycophancy probe score | Agreement bias |
| Preference accuracy during training | Overfitting signal |

## Related

- [SFT](sft.md) — must come first
- [LoRA](peft_lora_qlora.md) — the training mechanism
- [LLM-as-judge](../../../09-evaluation/apply/llm_as_judge.md) — pairwise evaluation
- [Online eval](../../../09-evaluation/apply/online_eval.md) — edit distance as preference signal
- [When not to fine-tune](when_not_to_fine_tune.md)
