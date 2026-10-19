# Evaluation → Generation evals (faithfulness, relevance, citation accuracy)

## What it is

Generation evaluation asks four separate questions, and conflating them hides which part is broken:

| Metric | Question | How |
|---|---|---|
| **Faithfulness / groundedness** | Is every claim supported by the retrieved context? | Decompose into claims, check entailment ([faithfulness check](../../06-rag/apply/8-generation-grounding/faithfulness_check.md)) |
| **Answer relevance** | Does the answer address the question asked? | Judge, or reverse-question similarity: generate questions from the answer and compare to the original |
| **Context precision** | Are the retrieved passages actually used and useful? Are relevant ones ranked high? | Per-passage relevance judgement, position-weighted |
| **Context recall** | Does the retrieved context contain everything the reference answer needs? | Decompose the reference answer into claims; check each is covered by the context |
| **Citation accuracy** | Do the cited passages exist and support the claims attributed to them? | Deterministic ID validation + entailment ([citations](../../06-rag/apply/8-generation-grounding/citations.md)) |
| **Behaviour correctness** | Did it answer when it should, abstain when it should, hedge when uncertain? | Compare to `expected_behaviour` ([refusal / no-answer path](../../06-rag/apply/8-generation-grounding/refusal_no_answer_path.md)) |

The relationship matters: **faithfulness ≠ correctness**. An answer can be perfectly faithful to retrieved context that is itself wrong or outdated. Faithfulness measures the generation step; correctness needs the reference answer.

## How to implement

1. **Deterministic checks first** (free): citation IDs exist in the retrieved set, must-include/must-not-include strings, schema validity, numbers present in context.
2. **Claim decomposition**: either the model emits claims natively ([grounded prompting](../../06-rag/apply/8-generation-grounding/grounded_prompting.md)) or split the answer into atomic sentences.
3. **Faithfulness**: for each claim, entailment against its *cited* passages (NLI or small LLM). Score = supported claims / factual claims ([faithfulness check](../../06-rag/apply/8-generation-grounding/faithfulness_check.md)).
4. **Context recall** (needs a reference answer): decompose the reference into claims, check each is present in the retrieved context. Low context recall with high retrieval recall@k means the *right document* was retrieved but the *right span* wasn't.
5. **Answer relevance**: judge against the question, or generate questions from the answer and measure similarity to the original.
6. **Behaviour**: compare the actual answer/abstain/clarify decision to the expected one — including the unanswerable cases.
7. **Report per query type**, with cost and latency ([golden dataset first](golden_dataset.md)).

## Small example

```python
def evaluate_generation(cases, system, nli, judge):
    rows = []
    for c in cases:
        caller = load_caller(c.caller)
        retrieved = system.retrieve(c.query, caller, k=6)
        out = system.answer(c.query, caller)
        text = out.get("answer") or ""
        by_id = {r["chunk_id"]: r for r in retrieved}

        # --- deterministic (free) ---
        cited = out.get("citations", [])
        citation_valid = all(cid in by_id for cid in cited)
        must_inc = all(s.lower() in text.lower() for s in c.must_include)
        must_not = not any(s.lower() in text.lower() for s in c.must_not_include)
        behaviour_ok = out.get("refused", False) == (c.expected_behaviour == "abstain")

        # --- faithfulness: claims vs their cited passages ---
        claims = out.get("claims") or [{"text": s, "citations": cited} for s in split_sentences(text) if is_factual(s)]
        supported = 0
        for cl in claims:
            ev = "\n".join(by_id[cid]["text"] for cid in cl["citations"] if cid in by_id)
            if ev and check_claim(cl["text"], ev)["verdict"] == "entailed":     # Faithfulness check
                supported += 1
        faithfulness = supported / len(claims) if claims else 1.0

        # --- context recall: does the context contain what the reference needs? ---
        ctx_recall = None
        if c.expected_answer:
            ref_claims = decompose_reference(c.expected_answer)
            ctx_text = "\n".join(r["text"] for r in retrieved)
            covered = sum(1 for rc in ref_claims
                          if nli(f"{ctx_text[:4000]} [SEP] {rc}", truncation=True)[0]["label"].lower().startswith("entail"))
            ctx_recall = covered / len(ref_claims) if ref_claims else None

        # --- context precision: are relevant passages ranked high? ---
        rel = [1 if r["chunk_id"] in c.relevant_chunk_ids else 0 for r in retrieved]
        ctx_precision = (sum(sum(rel[:i + 1]) / (i + 1) for i, v in enumerate(rel) if v) / max(sum(rel), 1)) if any(rel) else 0.0

        rows.append({
            "case_id": c.case_id, "query_type": c.query_type,
            "faithfulness": faithfulness, "context_recall": ctx_recall, "context_precision": ctx_precision,
            "citation_valid": citation_valid, "must_include": must_inc, "must_not_include": must_not,
            "behaviour_correct": behaviour_ok,
            "answer_relevance": judge_relevance(c.query, text, judge) if not out.get("refused") else None,
            "correctness": judge(c, out)["overall"] if c.expected_answer and not out.get("refused") else None,
            "cost_usd": out.get("cost_usd"), "latency_ms": out.get("latency_ms"),
        })
    return rows

def diagnose_generation(rows, retrieval_rows):
    """Attribute failures to the right stage."""
    f = np.mean([r["faithfulness"] for r in rows])
    cr = np.mean([r["context_recall"] for r in rows if r["context_recall"] is not None])
    rr = np.mean([r["recall@50"] for r in retrieval_rows if r["recall@50"] is not None])
    if rr < 0.85:                       return "Fix RETRIEVAL first (retrieval evals)"
    if cr < 0.8:                        return "Right documents, wrong spans — revisit CHUNKING (chunking)"
    if f < 0.9:                         return "Context is sufficient but the model is ungrounded — fix PROMPT/model (grounded prompting)"
    return "Grounded and complete — remaining errors are correctness of the sources themselves"
```

## Pros

- **Separates the generation failure modes** — ungrounded claims, incomplete answers, wrong behaviour, bad citations — each with a different fix.
- **Faithfulness and citation validity are cheap** and catch the most damaging failures.
- **Context recall bridges retrieval and generation**: it detects "right document, wrong chunk", which pure retrieval recall misses.
- **Behaviour correctness measures abstention**, the most-skipped metric ([metrics cheat sheet](../../AI_System.md#metrics-cheat-sheet)).
- Directly gates prompt and model changes ([prompt evaluation](prompt_evaluation.md)).

## Cons

- **Claim decomposition is imperfect** — sentences with multiple claims, or claims spanning sentences.
- **Faithfulness ≠ correctness**: a faithful answer from a stale document scores well and is wrong. Needs reference-based correctness too.
- **Judge-based metrics carry judge bias** ([LLM-as-judge](llm_as_judge.md)) and need calibration.
- **Expensive** if every metric runs on every case with an LLM — layer the cheap checks first.
- Reference answers are costly to write and must be maintained as the corpus changes.

## When to use

After retrieval evals show the right passages are being retrieved. Running generation evals on top of broken retrieval measures noise ([evaluation](../../AI_System.md#evaluation)).

## When NOT to use

- As the first diagnostic — always check retrieval first ([retrieval evals](retrieval_evals.md), [component vs end-to-end](component_vs_end_to_end.md)).
- Faithfulness alone as a quality claim: pair it with correctness against references, or you're measuring "consistent with whatever we retrieved".

## Production at scale

**Run the cheap layer on everything, the expensive layer on a sample.** Deterministic checks (citations, must-include, behaviour) on all cases every run; NLI faithfulness on all (it's fast); LLM-judge correctness and relevance on a subset or on flagged cases only.

**Context recall is the most under-used metric here.** High retrieval recall with low context recall points squarely at chunking: the answer's document was found but the span containing the fact wasn't in a retrieved chunk. That's a parent–child or chunk-size fix ([parent–child](../../06-rag/apply/2-chunking/parent_child_small_to_big.md)), and no amount of prompt work will help.

**Faithfulness against *cited* passages, not the whole context** — checking against everything retrieved inflates the score ([faithfulness check](../../06-rag/apply/8-generation-grounding/faithfulness_check.md)).

**Behaviour metrics need the unanswerable cases** from the golden set. Report correct-abstention and false-refusal rates as first-class dashboard numbers ([score thresholding & abstention](../../06-rag/apply/6-retrieval-ranking/score_thresholding_abstention.md)).

**Track by prompt and model version** ([prompt versioning & evaluation](../../05-prompt-and-context-engineering/apply/1-prompt-engineering/prompt_versioning_evaluation.md)). Faithfulness dropping after a model upgrade is one of the most common silent regressions ([versioning & rollout](../../10-production/apply/1-serving-cost-latency/versioning_rollout.md)).

**Production sampling** with the same metrics ([online eval](online_eval.md)): a 1–5% sample of live traffic scored for faithfulness and citation validity gives you an online quality signal that catches corpus drift and prompt regressions between releases.

**Cost and latency belong in the report.** A prompt change that raises faithfulness by 2 points and triples cost is a trade-off decision ([cost levers](../../10-production/apply/1-serving-cost-latency/cost_levers.md)).

**Reference maintenance**: when documents change, reference answers can become wrong. Tie references to `corpus_version` and review flagged cases when the corpus updates ([golden dataset first](golden_dataset.md)).

## Prod checklist

- [ ] Deterministic checks (citation IDs, must-include/not-include, schema, behaviour) on every case
- [ ] Faithfulness via NLI against each claim's cited passages
- [ ] Context recall computed against reference-answer claims
- [ ] Context precision (position-weighted relevance) reported
- [ ] Answer relevance and correctness via calibrated judge on a subset
- [ ] Correct-abstention and false-refusal rates from unanswerable cases
- [ ] Results broken down by query type, with cost and latency
- [ ] Diagnosis line attributing failures to retrieval / chunking / generation
- [ ] Same metrics sampled on production traffic
- [ ] References tied to corpus version and reviewed on corpus change

## Metrics to watch

| Metric | Why |
|---|---|
| Faithfulness / groundedness | Hallucination rate |
| Context recall | "Right document, wrong chunk" detection |
| Citation accuracy (valid IDs + supported claims) | Verifiability |
| Correct-abstention and false-refusal rates | The most-skipped metrics |
| Correctness vs reference (judge) | Faithful-but-wrong detection |

## Related

- [Retrieval evals](retrieval_evals.md) — run first
- [LLM-as-judge](llm_as_judge.md) — for relevance and correctness
- [Generation & grounding](../../06-rag/apply/8-generation-grounding/) — the generation controls being measured
- [Component vs end-to-end](component_vs_end_to_end.md) — the diagnostic ladder
- [Online eval](online_eval.md) — the same metrics on live traffic

## Video takeaway (30 seconds)

"Once retrieval is doing its job, measure what the model does with the passages. Four separate things, because they have different fixes. Faithfulness: is every claim supported by the passage it cites — check that against the cited passage only, not the whole context, or you'll inflate it. Context recall: does the retrieved text actually contain what the reference answer needs — if that's low while retrieval recall is high, you have a chunking problem, not a prompt problem. Citation accuracy: do the IDs exist and support the claims. And behaviour: did it abstain when it should have. That last one needs unanswerable questions in your golden set, and almost nobody has them."
