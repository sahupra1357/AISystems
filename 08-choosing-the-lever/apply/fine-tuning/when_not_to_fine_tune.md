# Fine-tuning → When *not* to fine-tune

## What it is

Fine-tuning is the most-reached-for and least-often-correct lever. The decision gate:

| Situation | Why fine-tuning is wrong | What to do instead |
|---|---|---|
| **The model doesn't know your facts** | Weights go stale the day training ends; no citations; can't delete a fact for GDPR; doesn't stop hallucination | RAG ([retrieval / RAG pipeline](../../../AI_System.md#retrieval--rag-pipeline)) |
| **The task is still changing** | Every requirement change means a new dataset, training run, and eval cycle | Prompting until the task stabilises ([prompt engineering](../../../AI_System.md#prompt-engineering)) |
| **A better prompt fixes it** | Weeks of work to replicate what a 20-minute prompt change achieves, with worse reversibility | Prompt engineering, measured ([prompt evaluation](../../../09-evaluation/apply/prompt_evaluation.md)) |
| **You can't build an eval** | You will not be able to tell whether it helped, or detect regressions | Build the golden set first ([golden dataset first](../../../09-evaluation/apply/golden_dataset.md)) |
| **Fewer than ~200 quality examples** | Insufficient signal; the model learns your labelling noise | Dynamic few-shot ([dynamic few-shot selection](../../../05-prompt-and-context-engineering/apply/1-prompt-engineering/dynamic_few_shot_selection.md)) |
| **Multi-tenant with per-tenant behaviour** | Adapter per tenant multiplies training, storage, and eval surface | Dynamic few-shot per tenant, or a fine-tuned reranker ([fine-tuning embeddings on your own logs](../../../06-rag/apply/3-embeddings/fine_tuning_embeddings.md)) |
| **Retrieval is the actual failure** | A fine-tuned model still can't answer from context it never received | Fix retrieval ([retrieval evals](../../../09-evaluation/apply/retrieval_evals.md), [component vs end-to-end](../../../09-evaluation/apply/component_vs_end_to_end.md)) |
| **You want it to hallucinate less** | Fine-tuning on correct answers does not teach uncertainty; it often increases confident errors | Grounding, citations, faithfulness, abstention ([generation & grounding](../../../06-rag/apply/8-generation-grounding/)) |

The two failures that recur most: **fine-tuning to inject knowledge** and **fine-tuning to fix a retrieval problem**. Both produce a model that is confidently wrong in a new way, having consumed weeks.

## How to implement (the gate)

1. **Diagnose first with the ladder** ([component vs end-to-end](../../../09-evaluation/apply/component_vs_end_to_end.md)): is the failure in parsing, chunking, retrieval, ranking, assembly, or generation? Only generation failures are fine-tuning candidates, and most of those are prompt failures.
2. **Classify the failure**: missing knowledge → RAG; wrong behaviour → prompting; inconsistent style at scale → fine-tuning; too slow/expensive → distillation ([prompting vs RAG vs fine-tuning](../prompting_vs_rag_vs_finetuning.md)).
3. **Prove the prompting ceiling** with data: the best prompt still shows a material failure rate on the golden set, and adding examples no longer helps.
4. **Check stability**: has the task specification been unchanged for a meaningful period? If it changed last month, wait.
5. **Check eval readiness**: golden set, held-out split, safety cases. No eval, no fine-tune.
6. **Check the data**: several hundred verified, consistent, diverse examples available or obtainable.
7. **Only then** proceed to [SFT](sft.md).

## Small example

```python
from dataclasses import dataclass

@dataclass
class FineTuneDecision:
    proceed: bool
    reason: str
    alternative: str | None = None

def should_fine_tune(diagnosis: dict, task: dict, data: dict, evals: dict) -> FineTuneDecision:
    # 1. Is the failure even in generation?
    if diagnosis["primary_failure_stage"] != "GENERATE":
        return FineTuneDecision(False, f"failures are at {diagnosis['primary_failure_stage']}",
                                alternative=STAGE_FIX[diagnosis["primary_failure_stage"]])
    # 2. Knowledge belongs in retrieval, never in weights
    if diagnosis["failure_type"] == "missing_knowledge":
        return FineTuneDecision(False, "knowledge failures need retrieval, not weights",
                                alternative="improve RAG coverage (§4); fine-tuned facts go stale and can't be cited or deleted")
    # 3. Has prompting actually hit a ceiling?
    if evals["best_prompt_score"] >= evals["target_score"]:
        return FineTuneDecision(False, "prompting already meets the target",
                                alternative="ship the prompt; revisit if cost/latency becomes the issue")
    if not evals["prompt_variants_exhausted"]:
        return FineTuneDecision(False, "prompt variants not exhausted",
                                alternative="structured output, dynamic few-shot (dynamic few-shot selection), decomposition (decomposition, negative examples, and prefilling)")
    # 4. Task stability
    if task["days_since_spec_change"] < 60:
        return FineTuneDecision(False, f"spec changed {task['days_since_spec_change']} days ago",
                                alternative="stabilise the task; you will retrain on every change")
    # 5. Eval readiness — no eval, no fine-tune
    if not (evals["golden_set_size"] >= 100 and evals["has_holdout"] and evals["has_safety_cases"]):
        return FineTuneDecision(False, "insufficient evaluation infrastructure",
                                alternative="build the golden set first (golden dataset first) — you cannot detect regressions otherwise")
    # 6. Data sufficiency and consistency
    if data["verified_examples"] < 200:
        return FineTuneDecision(False, f"only {data['verified_examples']} verified examples",
                                alternative="dynamic few-shot retrieval of similar solved cases (dynamic few-shot selection)")
    if data["label_agreement"] < 0.85:
        return FineTuneDecision(False, f"labeller agreement {data['label_agreement']:.2f} too low",
                                alternative="fix the labelling guidelines — the model will learn the inconsistency")
    # 7. Multi-tenant divergence
    if task["per_tenant_behaviour_required"] and task["n_tenants"] > 5:
        return FineTuneDecision(False, "per-tenant fine-tunes multiply ops surface",
                                alternative="tenant-scoped dynamic few-shot, or fine-tune the reranker (fine-tuning embeddings on your own logs)")
    return FineTuneDecision(True, "stable task, prompting ceiling proven, evals and data ready")

STAGE_FIX = {
    "PARSE":    "fix parsing/OCR routing (OCR / digitization, validation)",
    "CHUNK":    "parent-child or structure-aware chunking (layout / structure-aware, parent–child)",
    "RETRIEVE": "hybrid search, query understanding, embedding fine-tuning (hybrid search, query understanding, fine-tuning embeddings on your own logs)",
    "RERANK":   "cross-encoder reranking or reranker fine-tuning (reranking)",
    "ASSEMBLE": "context budgets and ordering (context engineering, "Lost in the middle", per-source truncation budgets)",
}

# The check that catches the most common mistake
def is_knowledge_failure(cases) -> bool:
    """If the answer text was never in the retrieved context, no amount of fine-tuning helps."""
    return sum(1 for c in cases if not answer_present_in_context(c)) / len(cases) > 0.3
```

## Pros (of *not* fine-tuning)

- **Weeks saved** and redirected to retrieval, chunking, or prompting work with higher expected value.
- **Reversibility**: prompts roll back in seconds; a fine-tuned model is a migration ([versioning & rollout](../../../10-production/apply/1-serving-cost-latency/versioning_rollout.md)).
- **Knowledge stays fresh and deletable** — a document update propagates immediately; a fact in weights does not.
- **Citations remain possible** — fine-tuned knowledge cannot be attributed to a source ([citations](../../../06-rag/apply/8-generation-grounding/citations.md)).
- **Simpler evaluation**: one fewer version dimension interacting with prompts and retrieval.

## Cons (of never fine-tuning)

- **Long prompts persist**: format rules stay in every request, costing tokens and latency at scale ([cost levers](../../../10-production/apply/1-serving-cost-latency/cost_levers.md)).
- **Format violations plateau** at a few percent that prompting cannot close.
- **The fast tier stays weak** without distillation, forcing expensive models on high-volume steps ([distillation](distillation.md)).
- Domain register and conventions that only weights can capture remain out of reach.

## When to apply this gate

Every time someone proposes fine-tuning — including yourself. It takes fifteen minutes and frequently redirects a multi-week project.

## When NOT to apply it (skip to fine-tuning)

- The case is already established: stable narrow task, proven prompting ceiling, evals in place, data curated, and the goal is format consistency or distillation for cost. Then the gate simply passes.

## Production at scale

**"Fine-tune it" is the most common wrong answer** to an AI quality problem, and the diagnostic ladder ([component vs end-to-end](../../../09-evaluation/apply/component_vs_end_to_end.md)) is the antidote: run it, and most failures resolve to retrieval or prompting. Publishing the failure-stage histogram to the team pre-empts the conversation entirely.

**Knowledge in weights is a compliance problem, not just a freshness problem.** You cannot delete a fact from weights on a GDPR request, you cannot cite it, and you cannot show a user where it came from. In regulated contexts RAG is the only defensible design ([data residency & retention](../../../10-production/apply/6-security-privacy-governance/data_residency_retention.md), [citations](../../../06-rag/apply/8-generation-grounding/citations.md)).

**Fine-tuning makes evaluation harder.** You add a model-version dimension that interacts with prompt versions and retrieval configs, and regressions become harder to attribute ([observability & tracing](../../../10-production/apply/4-observability-reliability/observability_tracing.md)). Teams without disciplined evals get worse at detecting problems after fine-tuning, not better.

**Dynamic few-shot is the underrated alternative** ([dynamic few-shot selection](../../../05-prompt-and-context-engineering/apply/1-prompt-engineering/dynamic_few_shot_selection.md)): retrieving the k most similar solved cases often reaches most of a fine-tune's benefit, with no training, instant updates, per-tenant scoping, and a natural improvement loop as the example store grows ([data flywheel](../../../10-production/apply/8-data-flywheel/data_flywheel.md)).

**Fine-tune the reranker before the generator** ([fine-tuning embeddings on your own logs](../../../06-rag/apply/3-embeddings/fine_tuning_embeddings.md)): retrieval quality is more often the bottleneck, cross-encoder fine-tuning is cheaper, needs no re-index, and can be swapped per tenant.

**Task stability is measurable** — track days since the last specification change. A task whose definition changes monthly will consume more effort in retraining than it saves.

**Revisit periodically.** A "no" today can become a "yes" once the task stabilises, volume grows, or evals mature. Record the decision and the trigger conditions that would change it.

## Prod checklist

- [ ] Diagnostic ladder run; primary failure stage identified ([component vs end-to-end](../../../09-evaluation/apply/component_vs_end_to_end.md))
- [ ] Failure classified: knowledge / behaviour / style / cost ([prompting vs RAG vs fine-tuning](../prompting_vs_rag_vs_finetuning.md))
- [ ] Prompting ceiling demonstrated with eval data, not asserted
- [ ] Task specification stable for a meaningful period
- [ ] Golden set, held-out split, and safety cases in place before any training
- [ ] ≥200 verified examples with acceptable labeller agreement
- [ ] Dynamic few-shot and reranker fine-tuning evaluated as alternatives
- [ ] Compliance implications of knowledge-in-weights considered
- [ ] Decision recorded with the conditions that would change it

## Metrics to watch

| Metric | Why |
|---|---|
| Failure-stage histogram | Whether generation is even the problem |
| Best-prompt score vs target | The ceiling claim |
| Days since task-spec change | Stability |
| Labeller agreement on candidate data | Whether the data can teach anything |
| Prompt tokens per request | The cost argument for fine-tuning |

## Related

- [Prompting vs RAG vs fine-tuning](../prompting_vs_rag_vs_finetuning.md) — the framing
- [Component vs end-to-end](../../../09-evaluation/apply/component_vs_end_to_end.md) — the diagnostic ladder
- [Dynamic few-shot](../../../05-prompt-and-context-engineering/apply/1-prompt-engineering/dynamic_few_shot_selection.md) — the main alternative
- [Embedding/reranker fine-tuning](../../../06-rag/apply/3-embeddings/fine_tuning_embeddings.md) — often the better target
- [SFT](sft.md) — what to do once the gate passes

## Video takeaway (30 seconds)

"'Let's fine-tune it' is the most common wrong answer to an AI quality problem. Three cases where it's definitely wrong. The model doesn't know your facts — that's retrieval; facts in weights go stale, can't be cited, and can't be deleted when someone asks. The task is still changing — you'll retrain every sprint. And a better prompt would fix it, which you can only know by measuring. Before anyone proposes training, run the diagnostic ladder: is the answer even in the retrieved context? If not, no fine-tune helps. And check the cheaper alternative first — retrieving the most similar solved cases as few-shot examples often gets most of the benefit with no training at all."
