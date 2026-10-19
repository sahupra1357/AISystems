# Common failure modes (and the fix)

## What it is

| Symptom | Usual cause | Fix | Files |
|---|---|---|---|
| **Right doc exists, never retrieved** | Chunk lacks context/keywords; dense-only search | Hybrid + contextual retrieval + query rewriting | [Hybrid search](../../../06-rag/apply/6-retrieval-ranking/hybrid_search.md), [contextual retrieval](../../../06-rag/apply/2-chunking/contextual_retrieval.md), [query rewriting](../../../06-rag/apply/5-query-understanding/query_rewriting.md) |
| **Answer cites the wrong section** | Chunks too large, no reranking | Parent–child + cross-encoder rerank | [Parent–child](../../../06-rag/apply/2-chunking/parent_child_small_to_big.md), [reranking](../../../06-rag/apply/6-retrieval-ranking/reranking.md) |
| **Confidently wrong on a fresh question** | No abstention path | Score threshold + faithfulness check | [Score thresholding & abstention](../../../06-rag/apply/6-retrieval-ranking/score_thresholding_abstention.md), [faithfulness check](../../../06-rag/apply/8-generation-grounding/faithfulness_check.md) |
| **Correct at turn 1, wrong at turn 3** | No query rewriting over history | Coreference-resolving rewrite step | [Query rewriting](../../../06-rag/apply/5-query-understanding/query_rewriting.md), [compaction / summarisation of long history](../../../05-prompt-and-context-engineering/apply/2-context-engineering/compaction_summarization.md) |
| **Costs 10× the estimate** | No caching, top-k too large, agent loops | Prompt cache, rerank-then-trim, step budgets | [Prompt caching of static prefixes](../../../05-prompt-and-context-engineering/apply/2-context-engineering/prompt_caching.md), [reranking](../../../06-rag/apply/6-retrieval-ranking/reranking.md), [cost & step budgets](../../../07-agents-tools-memory/apply/1-agents/cost_step_budgets.md) |
| **Works in dev, fails in prod** | Eval set isn't representative | Sample real traffic into the golden set | [Golden dataset first](../../../09-evaluation/apply/golden_dataset.md), [online eval](../../../09-evaluation/apply/online_eval.md), [data flywheel](../8-data-flywheel/data_flywheel.md) |
| **Table questions always wrong** | Table flattened at parse time | Structure-preserving parser, table-aware chunks | [Tables](../../../06-rag/apply/1-ingestion-parsing/tables.md), [complex layout](../../../06-rag/apply/1-ingestion-parsing/complex_layout_ocr.md), [layout / structure-aware](../../../06-rag/apply/2-chunking/layout_structure_aware.md) |
| **Agent takes a destructive action** | Excessive agency + injection | Least-privilege tokens, HITL gate, data≠instructions | [Excessive agency](../5-guardrails/excessive_agency.md), [human-in-the-loop](../../../07-agents-tools-memory/apply/1-agents/human_in_the_loop.md), [prompt-injection defence](../5-guardrails/prompt_injection_defense.md) |

Four more that recur often enough to add:

| Symptom | Usual cause | Fix | Files |
|---|---|---|---|
| **Answers a superseded policy** | Old versions still retrievable | Version filter + `is_current` on every query | [Document versioning](../../../06-rag/apply/1-ingestion-parsing/document_versioning.md), [conflict handling](../../../06-rag/apply/8-generation-grounding/conflict_handling.md) |
| **Top-5 results are five copies of one paragraph** | No dedupe; chunk overlap | Exact/near dedupe at ingest + MMR at query time | [Deduplication](../../../06-rag/apply/1-ingestion-parsing/deduplication.md), [MMR / diversity](../../../06-rag/apply/6-retrieval-ranking/mmr_diversity.md) |
| **Refuses things it clearly knows** | Threshold too strict; over-grounded prompt | Recalibrate τ; measure false-refusal rate | [Score thresholding & abstention](../../../06-rag/apply/6-retrieval-ranking/score_thresholding_abstention.md), [grounded prompting](../../../06-rag/apply/8-generation-grounding/grounded_prompting.md) |
| **Quality dropped and nobody knows why** | No version vector in traces; unpinned model | Version-stamped traces; pin models | [Observability & tracing](../4-observability-reliability/observability_tracing.md), [versioning & rollout](../1-serving-cost-latency/versioning_rollout.md) |

## How to implement (the diagnosis workflow)

1. **Reproduce with the trace** ([observability & tracing](../4-observability-reliability/observability_tracing.md)): the query, the caller, the retrieved chunk IDs and scores, the version vector.
2. **Run the diagnostic ladder** ([component vs end-to-end](../../../09-evaluation/apply/component_vs_end_to_end.md)) to find the first failing stage — this maps directly to a row above.
3. **Confirm with a second case** of the same shape; a single instance can be an outlier.
4. **Check the version vector**: did anything change (prompt, model, index, parser) around when it started?
5. **Apply the fix at the identified stage**, not at the stage where the symptom appeared.
6. **Add the case to the golden set** and confirm it fails before the fix and passes after ([data flywheel](../8-data-flywheel/data_flywheel.md)).
7. **Check whether the same cause explains other open failures** — most of these are systemic, not isolated.

## Small example

```python
SYMPTOM_PLAYBOOK = {
    "right_doc_never_retrieved": {
        "check": lambda t: t["expected_chunk_ids"] and not set(t["expected_chunk_ids"]) & set(t["pool_ids"]),
        "diagnostics": ["is the query identifier-heavy? → is BM25 in the mix (hybrid search)",
                        "do the chunks carry breadcrumbs/context headers? (layout / structure-aware, contextual retrieval)",
                        "was the query a pronoun-laden follow-up? (query rewriting)",
                        "did an ACL or date filter exclude it? (filtered search, self-query)"],
        "fix_order": ["hybrid search", "contextual retrieval", "query rewriting", "embedding fine-tune"],
    },
    "cites_wrong_section": {
        "check": lambda t: t["expected_chunk_ids"] and set(t["expected_chunk_ids"]) & set(t["pool_ids"])
                           and not set(t["expected_chunk_ids"]) & set(t["final_ids"]),
        "diagnostics": ["is a cross-encoder reranker in place? (reranking)",
                        "are chunks too large to be precise? (parent–child)"],
        "fix_order": ["add reranking", "parent-child chunking", "reduce child chunk size"],
    },
    "confidently_wrong_fresh_question": {
        "check": lambda t: t.get("retrieval_top_score", 1) < 0.35 and not t.get("abstained"),
        "diagnostics": ["is there a score threshold? (score thresholding & abstention)",
                        "is there a faithfulness gate? (faithfulness check)",
                        "does the golden set contain unanswerable questions? (golden dataset first)"],
        "fix_order": ["score threshold + abstention", "faithfulness gate", "grounded prompt rules"],
    },
    "cost_blowup": {
        "check": lambda t: t["cost_usd"] > 10 * MEDIAN_COST,
        "diagnostics": ["cache hit rate on this call site? (prompt caching of static prefixes)",
                        "how many chunks entered the context? (reranking, per-source truncation budgets)",
                        "agent step count and budget? (cost & step budgets)",
                        "escalation rate to the reasoning tier? (routing & escalation)"],
        "fix_order": ["prompt caching", "rerank-then-trim", "step/token budgets", "routing thresholds"],
    },
    "destructive_agent_action": {
        "check": lambda t: any(a["writes"] and not a.get("approved") for a in t.get("actions", [])),
        "diagnostics": ["does the caller's role even permit this tool? (excessive agency)",
                        "was flagged content in context? (prompt-injection defence)",
                        "is there an approval gate for this tool? (human-in-the-loop)"],
        "fix_order": ["remove the tool from the role", "tainted-context suppression", "approval gate"],
        "severity": "critical",
    },
}

def diagnose(trace) -> list[dict]:
    """Match a failing trace against the playbook; ordered by confidence."""
    matches = []
    for name, spec in SYMPTOM_PLAYBOOK.items():
        try:
            if spec["check"](trace):
                matches.append({"symptom": name, "diagnostics": spec["diagnostics"],
                                "fix_order": spec["fix_order"],
                                "severity": spec.get("severity", "normal")})
        except (KeyError, TypeError):
            continue
    return sorted(matches, key=lambda m: 0 if m["severity"] == "critical" else 1)

def systemic_check(symptom: str, window_days=7) -> dict:
    """Is this one bug or a class? Almost always a class."""
    traces = failing_traces(window_days)
    matching = [t for t in traces if SYMPTOM_PLAYBOOK[symptom]["check"](t)]
    return {"count": len(matching), "share": len(matching) / max(len(traces), 1),
            "concentrated_in": Counter(t["source"] for t in matching).most_common(3),
            "verdict": "systemic" if len(matching) / max(len(traces), 1) > 0.15 else "isolated"}
```

## Pros

- **Fast diagnosis**: a symptom lookup turns hours of investigation into minutes.
- **Prevents fixing the wrong layer** — the most common waste in AI projects is prompt-tuning a retrieval bug.
- **Shared vocabulary** across a team, and a natural onboarding document.
- Each row links to a tested remedy rather than a general principle.

## Cons

- **Symptoms overlap**: "wrong answer" can match several rows; the ladder ([component vs end-to-end](../../../09-evaluation/apply/component_vs_end_to_end.md)) disambiguates, the table alone does not.
- **Playbook thinking can stop investigation early** on a genuinely novel failure.
- **Fixes have costs**: reranking adds latency, parent–child adds complexity — the table names the remedy, not the trade-off.
- Needs trace data to apply; without chunk IDs and scores, most checks can't run ([observability & tracing](../4-observability-reliability/observability_tracing.md)).

## When to use

Whenever something is wrong — as the first lookup after reproducing with a trace. Also as a design checklist: a system with none of these mitigations will exhibit most of these symptoms.

## When NOT to use

- As a substitute for the diagnostic ladder when the symptom is ambiguous.
- For novel or safety-critical failures, where a full investigation is warranted regardless of pattern match.

## Production at scale

**Most of these are systemic, not isolated.** When one appears, run the systemic check: how many of the last week's failures share it, and are they concentrated in one source, tenant, or document type? Fixing the class is usually one change; fixing instances is endless ([feedback capture & triage](../8-data-flywheel/feedback_capture_triage.md)).

**Two of these are prevention, not diagnosis.** "Agent takes a destructive action" and "answers a superseded policy" should be structurally impossible — least-privilege credentials ([excessive agency](../5-guardrails/excessive_agency.md)) and version filtering ([document versioning](../../../06-rag/apply/1-ingestion-parsing/document_versioning.md)) — rather than symptoms you fix after they occur.

**"Works in dev, fails in prod" is an eval-set problem**, and the fix is the flywheel ([data flywheel](../8-data-flywheel/data_flywheel.md)): continuous sampling of real traffic into the golden set. If this symptom recurs, the loop isn't running.

**"Quality dropped and nobody knows why" is an observability problem.** The version vector on every trace ([observability & tracing](../4-observability-reliability/observability_tracing.md)) and pinned models ([versioning & rollout](../1-serving-cost-latency/versioning_rollout.md)) turn it into a five-minute bisection.

**Cost blowups have four common causes** in this order: cache hit rate collapse (a variable in the prompt prefix, [prompt caching of static prefixes](../../../05-prompt-and-context-engineering/apply/2-context-engineering/prompt_caching.md)), top-k too large ([reranking](../../../06-rag/apply/6-retrieval-ranking/reranking.md)), unbounded agent loops ([cost & step budgets](../../../07-agents-tools-memory/apply/1-agents/cost_step_budgets.md)), and over-escalation to the reasoning tier ([routing & escalation](../../../04-llm-foundations/apply/routing_escalation.md)). Check them in that order.

**Track symptom frequency over time** as a companion to the failure-bucket histogram ([feedback capture & triage](../8-data-flywheel/feedback_capture_triage.md)) — it shows which mitigations have landed.

**Add your own rows.** Every domain develops its own recurring failures; the table is a starting set, not a closed list, and extending it is how team knowledge accumulates.

## Prod checklist

- [ ] Trace-based reproduction available for any reported failure (chunk IDs, scores, version vector)
- [ ] Diagnostic ladder run before applying a fix
- [ ] Systemic check performed: is this a class or an instance?
- [ ] Fix applied at the diagnosed stage, not the symptom stage
- [ ] Case added to the golden set; fails before, passes after
- [ ] Structural preventions in place for the two critical rows (least privilege, version filtering)
- [ ] Symptom frequency tracked over time
- [ ] Playbook extended with domain-specific failures as they're found

## Metrics to watch

| Metric | Why |
|---|---|
| Symptom frequency by type over time | Which mitigations have landed |
| Systemic vs isolated ratio | Whether you're fixing classes or instances |
| Median time from report to diagnosis | Observability effectiveness |
| Recurrence of previously-fixed symptoms | Durability of fixes |

## Related

- [Component vs end-to-end](../../../09-evaluation/apply/component_vs_end_to_end.md) — the disambiguating ladder
- [Triage](../8-data-flywheel/feedback_capture_triage.md) — the bucket histogram
- [Observability](../4-observability-reliability/observability_tracing.md) — the trace data these checks need
- [Retrieval & ranking](../../../06-rag/apply/6-retrieval-ranking/), [chunking](../../../06-rag/apply/2-chunking/), [agents](../../../07-agents-tools-memory/apply/1-agents/), [testing & reliability](../../../AI_System.md#testing--reliability) — the individual fixes
- [Data flywheel](../8-data-flywheel/data_flywheel.md) — turning each failure into a permanent test

## Video takeaway (30 seconds)

"Eight symptoms cover most production AI failures, and each has a known fix. The right document exists but is never retrieved — that's dense-only search; add keyword search and contextual headers. It cites the wrong section — chunks too big, no reranker. Confidently wrong on a question your corpus can't answer — no abstention threshold. Correct on turn one, wrong on turn three — no query rewriting over history. Costs ten times the estimate — check your cache hit rate first. Table questions always wrong — the table got flattened at parse time. And when the agent does something destructive, the fix isn't a better prompt, it's a credential that couldn't do it. Look up the symptom, but confirm with the diagnostic ladder before you fix anything."
