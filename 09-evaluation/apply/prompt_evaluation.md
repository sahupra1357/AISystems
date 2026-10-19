# Evaluation → Prompt evaluation

## What it is

Prompt evaluation is A/B testing for prompts, run offline against fixed ground truth. Given a call site ([prompt versioning & evaluation](../../05-prompt-and-context-engineering/apply/1-prompt-engineering/prompt_versioning_evaluation.md)), you have several candidate prompts and need to know which is better on *your* cases — and, later, whether a change regressed anything.

The scoring method depends on the task:

| Task type | Metric | Notes |
|---|---|---|
| Classification / routing | Accuracy, per-class F1, confusion matrix | Deterministic, cheap, exact |
| Extraction | Field-level precision/recall/F1, exact match on normalised values | Per-field, not aggregate — one bad field hides in an average |
| Structured output | Schema-violation rate, required-field coverage | Free to compute ([structured outputs & tool use](../../05-prompt-and-context-engineering/apply/1-prompt-engineering/structured_outputs_tool_use.md)) |
| Grounded answering | must_include / must_not_include, citation validity, faithfulness ([faithfulness check](../../06-rag/apply/8-generation-grounding/faithfulness_check.md)) | Deterministic checks first |
| Open-ended text | Rubric score via LLM-as-judge ([LLM-as-judge](llm_as_judge.md)), pairwise preference | Calibrate against humans |
| Behaviour | Abstention correctness, refusal rate, format adherence | Often the thing that actually broke |

The discipline: **deterministic checks before model-based ones.** Schema validity, citation existence, required substrings, and forbidden substrings cost nothing and catch most regressions.

## How to implement

1. **Define the call site's eval subset**: which golden cases exercise this prompt ([golden dataset first](golden_dataset.md)). A change to the answer prompt runs generation cases, not parser cases.
2. **Pin everything else**: temperature 0, fixed model version, fixed retrieval results (cache them) so the only variable is the prompt.
3. **Run variants in parallel**, scoring each case with the deterministic checks, then the judge where needed.
4. **Report a diff**, not an absolute: per-metric change versus the current production version, per query type, with per-case regressions listed.
5. **Gate on regressions**, not just aggregate improvement: a variant that raises the mean while breaking 5 previously-passing cases needs a human decision.
6. **Store results** keyed by `(prompt_id, prompt_version, model_version, golden_version)` so any historical comparison is reproducible.
7. **Wire into CI** on changes under `prompts/` ([prompt versioning & evaluation](../../05-prompt-and-context-engineering/apply/1-prompt-engineering/prompt_versioning_evaluation.md), [the testing pyramid](testing_pyramid.md)).

## Small example

```python
from dataclasses import dataclass

@dataclass
class EvalResult:
    case_id: str; passed: bool; scores: dict; output: str; cost_usd: float; latency_ms: float

def score_case(case, output, retrieved) -> dict:
    text = (output.get("answer") or "")
    scores = {
        # --- deterministic, free ---
        "schema_valid":     output.get("schema_valid", True),
        "must_include":     all(s.lower() in text.lower() for s in case.must_include),
        "must_not_include": not any(s.lower() in text.lower() for s in case.must_not_include),
        "citation_valid":   set(output.get("citations", [])).issubset({r["chunk_id"] for r in retrieved}),
        "behaviour_correct": (output.get("refused", False) == (case.expected_behaviour == "abstain")),
    }
    # --- model-based, only where needed and only if deterministic checks passed ---
    if case.expected_answer and not output.get("refused"):
        scores["judge"] = judge_rubric(case.query, case.expected_answer, text)     # LLM-as-judge
    return scores

def eval_prompt_variant(variant_id: str, cases, fixed_retrieval: dict) -> list[EvalResult]:
    results = []
    for c in cases:
        retrieved = fixed_retrieval[c.case_id]                       # pinned: isolate the prompt as the variable
        t0 = time.time()
        out = generate_with_prompt(variant_id, c.query, retrieved, temperature=0)
        results.append(EvalResult(c.case_id, all(v is True for k, v in score_case(c, out, retrieved).items()
                                                 if isinstance(v, bool)),
                                  score_case(c, out, retrieved), out.get("answer", ""),
                                  out["cost_usd"], (time.time() - t0) * 1000))
    return results

def compare(baseline: list[EvalResult], candidate: list[EvalResult], cases) -> dict:
    by_id_b = {r.case_id: r for r in baseline}
    regressions = [r.case_id for r in candidate if by_id_b[r.case_id].passed and not r.passed]
    fixes       = [r.case_id for r in candidate if not by_id_b[r.case_id].passed and r.passed]
    metrics = {}
    for key in ("must_include", "citation_valid", "behaviour_correct", "schema_valid"):
        b = mean(bool(by_id_b[r.case_id].scores.get(key)) for r in candidate)
        c = mean(bool(r.scores.get(key)) for r in candidate)
        metrics[key] = {"baseline": round(b, 3), "candidate": round(c, 3), "delta": round(c - b, 3)}
    metrics["judge"] = {"baseline": mean_opt(r.scores.get("judge") for r in baseline),
                        "candidate": mean_opt(r.scores.get("judge") for r in candidate)}
    metrics["cost_per_query"] = {"baseline": mean(r.cost_usd for r in baseline),
                                 "candidate": mean(r.cost_usd for r in candidate)}
    by_type = per_query_type(candidate, baseline, cases)
    verdict = "BLOCK" if regressions else "PASS"
    return {"verdict": verdict, "regressions": regressions, "fixes": fixes,
            "metrics": metrics, "by_query_type": by_type}

# CI gate
def ci_gate(report, tolerance=0.0):
    if report["regressions"]:
        raise SystemExit(f"Blocked: {len(report['regressions'])} regressions: {report['regressions'][:5]}")
    if report["metrics"]["judge"]["candidate"] < report["metrics"]["judge"]["baseline"] - tolerance:
        raise SystemExit("Blocked: judge score regression")
```

## Pros

- **Turns prompt work from guesswork into measurement** — the highest-leverage discipline in the whole reference.
- **Catches regressions before users do**, which is the entire point of the golden set.
- **Per-case diffs** show exactly what a change broke, not just that something moved.
- **Cheap**: deterministic checks are free; judge calls only where needed; prompt caching makes runs affordable.
- Reproducible: pinned model, temperature 0, cached retrieval.

## Cons

- **Eval cost and time** on every change — mitigated by scoping to the affected call site and caching.
- **Golden-set overfitting**: repeatedly tuning against the same cases produces prompts that ace the eval and disappoint in production (hold out a split, [golden dataset first](golden_dataset.md)).
- **Judge noise** on open-ended tasks makes small deltas unreliable ([LLM-as-judge](llm_as_judge.md), [statistical hygiene](statistical_hygiene.md)).
- **Pinned retrieval hides interactions**: a prompt tuned on cached retrieval may behave differently with live retrieval variance — run an end-to-end pass before release ([component vs end-to-end](component_vs_end_to_end.md)).
- Small sets give wide confidence intervals; a 3-point move often means nothing.

## When to use

Every prompt change, every model upgrade, every schema change. Non-negotiable for anything reaching production.

## When NOT to use

- Exploratory prototyping — iterate freely, then evaluate before it becomes real.
- Don't build heavyweight eval infrastructure before you have a golden set; the set is the prerequisite.

## Production at scale

**Scope runs to the change.** Map call sites to golden subsets so a rewrite-prompt change runs the query-understanding cases in seconds, not the whole suite. Full-suite runs happen nightly and before releases.

**Regressions block; improvements don't auto-merge.** The gate should be "no case that passed now fails". Aggregate improvements with individual regressions require a human looking at the specific cases — that's usually where a subtle behaviour change is hiding.

**Report per query type, always.** A prompt change that improves factoid answers and destroys abstention shows as flat in the mean. The category breakdown is where the truth is ([golden dataset first](golden_dataset.md)).

**Cost and latency in the report.** A prompt that adds 400 tokens of examples for 1 point of accuracy is a decision with a price tag ([cost levers](../../10-production/apply/1-serving-cost-latency/cost_levers.md)).

**Deterministic checks first, judge second, humans third.** Only escalate to expensive scoring where cheap scoring can't decide. Use the judge on a subset if cost matters, and report confidence intervals ([statistical hygiene](statistical_hygiene.md)).

**Version the whole tuple.** Behaviour depends on prompt + model + retrieval config + examples + schema. Log all of them with each result so a historical comparison is meaningful ([prompt versioning & evaluation](../../05-prompt-and-context-engineering/apply/1-prompt-engineering/prompt_versioning_evaluation.md)).

**Prompt-eval tooling** (Langfuse, LangSmith, Braintrust, promptfoo, Weights & Biases) gives you run tracking, diffs, and dashboards. Keep prompts and golden data in git as the source of truth.

**Watch for eval-set contamination**: if your dynamic few-shot store ([dynamic few-shot selection](../../05-prompt-and-context-engineering/apply/1-prompt-engineering/dynamic_few_shot_selection.md)) or memory contains golden cases, results are inflated. Exclude explicitly.

## Prod checklist

- [ ] Golden subsets mapped per call site; scoped runs on change, full suite nightly
- [ ] Temperature 0, pinned model, cached retrieval for prompt-only comparisons
- [ ] Deterministic checks before judge calls; judge only where needed
- [ ] Report shows per-case regressions and fixes, per-query-type breakdown, cost and latency
- [ ] CI blocks on any previously-passing case that now fails
- [ ] Results keyed by (prompt version, model version, golden version, retrieval config)
- [ ] Held-out test split run before release
- [ ] Eval cases excluded from example stores and memory to prevent contamination
- [ ] End-to-end pass with live retrieval before shipping

## Metrics to watch

| Metric | Why |
|---|---|
| Regression count per candidate | The gate |
| Per-query-type deltas | Hidden trade-offs |
| Judge score with confidence interval | Whether the move is real |
| Cost and latency delta per change | Price of the improvement |
| Dev vs held-out divergence | Overfitting to the eval set |

## Related

- [Golden dataset](golden_dataset.md) — the prerequisite
- [LLM-as-judge](llm_as_judge.md) — scoring open-ended output
- [Statistical hygiene](statistical_hygiene.md) — reading the numbers
- [Prompt versioning](../../05-prompt-and-context-engineering/apply/1-prompt-engineering/prompt_versioning_evaluation.md) — what's being evaluated
- [Testing pyramid](testing_pyramid.md) — where this runs

## Video takeaway (30 seconds)

"A prompt change without an eval run is a deploy without a test. Pin everything except the prompt — temperature zero, fixed model, cached retrieval — run your golden cases against both versions, and report the *diff*: which cases newly pass, and which ones broke. That second list is the gate. Aggregate improvement with three individual regressions is not a win; it's a change someone needs to look at. Score with free deterministic checks first — schema valid, required phrase present, citations real — and only pay for a judge on the open-ended parts. And always break the results down by question type, because a change that helps factoids and breaks abstention looks flat in the average."
