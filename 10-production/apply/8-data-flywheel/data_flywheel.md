# Data flywheel → The compounding loop

## Video takeaway

"The flywheel is what separates systems that improve from systems that plateau. Production traffic produces traces and user signals; you triage the failures weekly; each confirmed failure becomes a permanent golden case; you fix it at the right layer, gate it in CI, and canary it. Every turn makes your eval set more representative and the next fix cheaper. Two things people miss. The new test case must *fail* before the fix — assert it, or you've written a test that protects nothing. And the same triage produces three other free assets: hard negatives for your retriever, preference pairs from user edits, and training examples from accepted outputs. Most teams collect the test cases and throw the rest away."

## What it is

```
        ┌─────────────────────────────────────────────────────────────┐
        │                                                             │
        ▼                                                             │
  prod traffic ──► traces + user feedback ──► triage failures ────────┤
        ▲                                            │                │
        │                                            ▼                │
        │                                     new golden cases        │
        │                                            │                │
        │                                            ▼                │
   canary ◄──── eval ◄──── fix: prompt / retrieval / chunking / model │
        └─────────────────────────────────────────────────────────────┘
```

Every turn of the loop produces four durable assets:

| Asset | Where it comes from | What it improves |
|---|---|---|
| **Golden cases** | Confirmed production failures | Regression protection, forever ([golden dataset first](../../../09-evaluation/apply/golden_dataset.md)) |
| **Hard negatives** | Failed retrievals: what ranked high but was wrong | Embedding and reranker fine-tuning ([fine-tuning embeddings on your own logs](../../../06-rag/apply/3-embeddings/fine_tuning_embeddings.md)) |
| **Preference pairs** | User edits: draft vs sent | Preference tuning ([preference tuning](../../../08-choosing-the-lever/apply/fine-tuning/preference_tuning_dpo_rlhf.md)) |
| **Training examples** | Accepted outputs | SFT and distillation ([SFT](../../../08-choosing-the-lever/apply/fine-tuning/sft.md), [distillation](../../../08-choosing-the-lever/apply/fine-tuning/distillation.md)) |

The compounding property: each fixed failure becomes a permanent test, so the system cannot regress there again, and the accumulated data makes the *next* fix cheaper. A system without the loop fixes the same class of bug repeatedly and plateaus at whatever quality its initial build reached.

## How to implement

1. **Capture**: traces with the full version vector, retrieved chunk IDs and scores, and outcomes ([observability & tracing](../4-observability-reliability/observability_tracing.md)); explicit feedback and implicit signals — copy, edit distance, retry, escalation, task completion ([online eval](../../../09-evaluation/apply/online_eval.md)).
2. **Triage on a cadence** (weekly is the practical rhythm): bucket failures by stage using the diagnostic ladder ([component vs end-to-end](../../../09-evaluation/apply/component_vs_end_to_end.md), [feedback capture & triage](feedback_capture_triage.md)).
3. **Convert to assets**: each confirmed failure becomes a golden case with a labelled expectation; failed retrievals yield hard negatives; edits yield preference pairs.
4. **Fix at the right layer** — the bucket tells you where: parse, chunk, retrieve, rank, generate, or tooling.
5. **Evaluate**: the new case must fail before the fix and pass after; the full suite must not regress ([prompt evaluation](../../../09-evaluation/apply/prompt_evaluation.md)).
6. **Ship through the gate**: CI evals → canary → online comparison → promote ([the testing pyramid](../../../09-evaluation/apply/testing_pyramid.md), [versioning & rollout](../1-serving-cost-latency/versioning_rollout.md)).
7. **Measure the loop itself**: cases added per week, time from failure to fix, and the trend in the failure-bucket distribution.

## Small example

```python
from dataclasses import dataclass
from datetime import datetime, timedelta

@dataclass
class FlywheelCandidate:
    trace_id: str; query: str; caller: dict
    signal: str                     # "thumbs_down" | "heavy_edit" | "retry" | "escalated" | "guardrail_flag" | "low_faithfulness"
    signal_strength: float
    retrieved: list[dict]           # chunk ids + scores (observability & tracing)
    answer: str | None
    version_vector: dict
    detected_at: datetime

def harvest_candidates(traces, since: datetime) -> list[FlywheelCandidate]:
    """Automated signals do the first pass — humans only review what's already suspicious."""
    out = []
    for t in traces_since(traces, since):
        signals = []
        if t.get("thumbs") == -1:                       signals.append(("thumbs_down", 1.0))
        if t.get("edit_distance", 0) > 0.3:             signals.append(("heavy_edit", t["edit_distance"]))
        if t.get("retried"):                            signals.append(("retry", 0.7))
        if t.get("escalated_to_human"):                 signals.append(("escalated", 1.0))
        if t.get("faithfulness", 1.0) < 0.85:           signals.append(("low_faithfulness", 1 - t["faithfulness"]))
        if not t.get("citation_valid", True):           signals.append(("guardrail_flag", 1.0))
        if t.get("abstained") and t.get("user_rephrased_and_succeeded"):
            signals.append(("false_refusal", 0.9))      # the invisible failure (score thresholding & abstention)
        if signals:
            name, strength = max(signals, key=lambda s: s[1])
            out.append(FlywheelCandidate(t["trace_id"], t["query"], t["caller"], name, strength,
                                         t["retrieved"], t.get("answer"), t["version_vector"], t["ts"]))
    return sorted(out, key=lambda c: -c.signal_strength)

def promote_to_golden(candidate: FlywheelCandidate, review: dict) -> dict | None:
    """A human confirms the failure and writes the expectation; that becomes a permanent test."""
    if not review["is_genuine_failure"]:
        log_triage(candidate.trace_id, "not_a_failure", review["note"]); return None
    case = {
        "case_id": f"GOLD-{next_case_id()}",
        "query": candidate.query,
        "query_type": review["query_type"],
        "caller": candidate.caller,
        "relevant_chunk_ids": review["should_have_retrieved"],
        "expected_answer": review["expected_answer"],
        "must_include": review["must_include"],
        "must_not_include": review.get("must_not_include", []),
        "expected_behaviour": review["expected_behaviour"],
        "source": f"production_failure:{candidate.signal}",
        "origin_trace": candidate.trace_id,
        "failure_stage": review["diagnosed_stage"],          # from the ladder (component vs end-to-end)
        "added": datetime.utcnow().date().isoformat(),
        "corpus_version": current_corpus_version(),
    }
    golden_set.add(case)
    assert not run_single_case(case)["passed"], "new case must FAIL before the fix"
    return case

def mine_hard_negatives(candidate: FlywheelCandidate, review: dict) -> list[dict]:
    """Chunks that ranked high but were wrong → training signal for embedder/reranker (fine-tuning embeddings on your own logs)."""
    correct = set(review["should_have_retrieved"])
    return [{"query": candidate.query, "positive_id": cid, "negative_ids":
             [c["chunk_id"] for c in candidate.retrieved[:10] if c["chunk_id"] not in correct][:5]}
            for cid in correct]

def mine_preference_pairs(traces) -> list[dict]:
    """User edits are free preference data (preference tuning)."""
    return [{"prompt": t["query"], "rejected": t["answer"], "chosen": t["final_text"],
             "distance": t["edit_distance"]}
            for t in traces if 0.05 < t.get("edit_distance", 0) < 0.8]

def flywheel_health(period_days=30) -> dict:
    return {
        "candidates_harvested": count_candidates(period_days),
        "cases_added": count_golden_added(period_days),
        "cases_from_production": pct_golden_from_production(),
        "median_days_failure_to_fix": median_time_to_fix(period_days),
        "bucket_distribution": failure_buckets(period_days),          # should shift over time (feedback capture & triage)
        "hard_negatives_mined": count_hard_negatives(period_days),
        "preference_pairs": count_preference_pairs(period_days),
        "regression_recurrence": recurrence_rate(),                    # fixed bugs coming back — should be ~0
    }
```

## Pros

- **Compounding quality**: each cycle makes the eval set more representative and the next fix cheaper.
- **Regressions become impossible** in areas already fixed — the permanent-test property.
- **Evals stay representative** as the corpus, users, and product change ([golden dataset first](../../../09-evaluation/apply/golden_dataset.md)).
- **Free training data**: hard negatives, preference pairs, and SFT examples accumulate as a by-product of use.
- The bucket distribution over time is the clearest signal of where the system actually is.

## Cons

- **Requires sustained human attention** — triage is unglamorous and the first thing dropped under pressure.
- **Signal ambiguity**: implicit signals are noisy; a thumbs-down may be a UI complaint, an edit may be personal style.
- **Bias toward complainers**: explicit feedback over-represents frustrated users; silent failures are invisible.
- **Golden-set drift and bloat** without pruning: stale cases referencing deleted documents.
- **Privacy**: harvesting production data into eval sets and training data has retention and consent implications ([data residency & retention](../6-security-privacy-governance/data_residency_retention.md), [PII](../6-security-privacy-governance/pii.md)).

## When to use

Every production system, from the first week. The loop is cheap to start (harvest + weekly triage) and expensive to retrofit once months of traces have expired.

## When NOT to use (heavy machinery)

- Pre-launch: use expert-written cases and a manual review of every failure — you don't have traffic yet.
- Don't build automated mining pipelines before you have the manual triage habit; the habit is the hard part.

## Production at scale

**Automated harvesting, human confirmation.** Signals shortlist candidates; a person confirms the failure and writes the expected answer. Skipping the human step fills the golden set with noise; skipping the automation makes triage unaffordable.

**Weekly cadence with an owner.** A named person, a fixed slot, a bounded batch (20–40 candidates). Without an owner it lapses within a month, and the loop's value is entirely in its continuity.

**Every case records its diagnosed failure stage** ([component vs end-to-end](../../../09-evaluation/apply/component_vs_end_to_end.md)). The bucket histogram over time is your roadmap and your progress report: if parsing failures fall from 20% to 5% after a parser upgrade, that's the flywheel working.

**The new case must fail before the fix.** Assert it. A "regression test" that passed before the change protects nothing.

**Harvest inside your retention window.** If traces expire in 30 days ([data residency & retention](../6-security-privacy-governance/data_residency_retention.md)), the flywheel must sample into a longer-lived, anonymised store within that window, or short retention silently starves your evals.

**Mine all four asset types.** Most teams collect golden cases and stop, leaving hard negatives (which directly improve retrieval, [fine-tuning embeddings on your own logs](../../../06-rag/apply/3-embeddings/fine_tuning_embeddings.md)) and preference pairs ([preference tuning](../../../08-choosing-the-lever/apply/fine-tuning/preference_tuning_dpo_rlhf.md)) on the floor — both are free by-products of the same triage.

**Look for silent failures too**: answers users accepted that were subtly wrong. Sampled LLM-judge scoring of *passing* traffic ([LLM-as-judge](../../../09-evaluation/apply/llm_as_judge.md), [online eval](../../../09-evaluation/apply/online_eval.md)) catches what no user signal reports.

**Track recurrence**: a previously-fixed failure reappearing means the case wasn't added, the gate didn't run, or the fix was superficial. Recurrence should be near zero.

**Prune deliberately**: retire cases whose documents no longer exist; keep the set representative rather than merely large.

## Prod checklist

- [ ] Traces capture version vector, retrieved chunk IDs + scores, and outcomes
- [ ] Implicit and explicit signals instrumented (edit distance, retry, escalation, thumbs, guardrail flags)
- [ ] Automated candidate harvesting ranked by signal strength
- [ ] Weekly triage with a named owner and a bounded batch
- [ ] Human confirmation before a case enters the golden set; expected answer written
- [ ] New cases asserted to fail before the fix
- [ ] Failure stage recorded per case; bucket histogram tracked over time
- [ ] Hard negatives, preference pairs, and SFT examples mined from the same triage
- [ ] Sampled judging of passing traffic to catch silent failures
- [ ] Harvesting operates inside the trace retention window; anonymised eval store
- [ ] Recurrence rate tracked; stale cases pruned

## Metrics to watch

| Metric | Why |
|---|---|
| Golden cases added per week; % sourced from production | Loop is turning |
| Median days from failure detection to fix shipped | Loop speed |
| Failure-bucket distribution over time | Where the system is improving |
| Regression recurrence rate | Whether fixes are durable |
| Hard negatives / preference pairs accumulated | Training-data flywheel |

## Related

- [Golden dataset](../../../09-evaluation/apply/golden_dataset.md) — the primary output
- [Online eval](../../../09-evaluation/apply/online_eval.md) — the input signals
- [Feedback capture & triage](feedback_capture_triage.md) — the mechanics
- [Embedding fine-tuning](../../../06-rag/apply/3-embeddings/fine_tuning_embeddings.md), [Preference tuning](../../../08-choosing-the-lever/apply/fine-tuning/preference_tuning_dpo_rlhf.md), [Distillation](../../../08-choosing-the-lever/apply/fine-tuning/distillation.md) — data consumers
- [Observability](../4-observability-reliability/observability_tracing.md), [Rollout](../1-serving-cost-latency/versioning_rollout.md) — capture and delivery
