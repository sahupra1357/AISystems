# Query understanding → Decomposition (multi-hop into sub-questions)

## Video takeaway

"Ask 'how does our parental leave compare to the German minimum, and what changed in 2024' and a single top-k search answers maybe one third of it — the passages for the other parts aren't near that query. Split it into sub-questions, retrieve for each in parallel, label the evidence by part, and require the synthesis step to address every part and say so out loud when a part has no evidence. Gate it, because decomposing 'how many sick days' is pure waste. And if a sub-question needs the previous answer, bound the chain at two or three hops — that's where a query quietly becomes an expensive agent run."

## What it is

Top-k retrieval returns the k passages nearest one query vector. A compound question has multiple information needs, and the passages that satisfy need B may be nowhere near the query's centroid. Symptoms: the answer covers one clause of the question and silently ignores the rest.

Decomposition splits the question into independently retrievable sub-questions:

```
"Compare our parental leave with the German statutory minimum and note 2024 changes"
   ├── q1: "company parental leave policy entitlement"
   ├── q2: "German statutory minimum parental leave (Elternzeit) entitlement"
   └── q3: "changes to parental leave policy in 2024"
        each → retrieve (+rerank) → evidence set
             → synthesise one answer citing all three
```

Two flavours: **static decomposition** (one LLM call produces all sub-questions upfront; parallel retrieval — fast, works when the parts are independent) and **iterative/sequential** (each sub-question's answer feeds the next — needed for genuine multi-hop chains like "who is the vendor on the largest 2024 contract, and what is their SLA?"). Sequential is where this shades into agentic retrieval ([recursive / agentic retrieval](../6-retrieval-ranking/recursive_agentic_retrieval.md), [agentic RAG](../7-advanced-rag/agentic_rag.md)).

## How to implement

1. **Detect compound questions** cheaply: conjunctions ("and", "as well as", "compare", "versus"), multiple question marks, multiple named entities, or a classifier in the same call as rewriting ([query rewriting](query_rewriting.md)).
2. **Generate sub-questions** (fast tier, temperature 0): output JSON with `sub_questions` and a `dependency` flag (independent → parallel, dependent → sequential). Cap at 3–5.
3. **Retrieve per sub-question** in parallel with the same ACL filters ([filtered search](../4-indexing/filtered_search.md)); rerank each list independently ([reranking](../6-retrieval-ranking/reranking.md)) and keep the top 3–4 per sub-question.
4. **Label the evidence** by sub-question in the context block so the generator knows which passages answer which part (`<evidence for="q2">`).
5. **Synthesise** with a prompt that requires addressing every sub-question and citing per part, and that says explicitly what to do when a part has no evidence (say so — don't skip silently, [refusal / no-answer path](../8-generation-grounding/refusal_no_answer_path.md)).
6. **Sequential path**: run q1, extract the entity from its answer, template it into q2, retrieve; bound at 2–3 hops with a step budget ([cost & step budgets](../../../07-agents-tools-memory/apply/1-agents/cost_step_budgets.md)).
7. **Evaluate** on a multi-hop golden subset: per-part coverage, not just overall answer quality.

## Small example

```python
DECOMP_SYS = """Split the user's question into the minimum set of independently searchable sub-questions.
Rules:
- If the question has only one information need, return it unchanged as a single sub-question.
- Mark "dependent": true if a sub-question needs another's answer first.
- Maximum 4 sub-questions.
Output JSON: {"sub_questions": [{"id": "q1", "text": "...", "dependent_on": null}], "compound": bool}"""

def decompose(query: str) -> list[dict]:
    out = llm("decompose", DECOMP_SYS, [{"role": "user", "content": query}], max_tokens=250)
    try:
        j = json.loads(out.text)
    except Exception:
        return [{"id": "q1", "text": query, "dependent_on": None}]
    return j["sub_questions"][:4] if j.get("compound") else [{"id": "q1", "text": query, "dependent_on": None}]

async def retrieve_decomposed(query, caller, per_part=4):
    parts = decompose(query)
    independent = [p for p in parts if not p["dependent_on"]]
    results = await asyncio.gather(*[retrieve_and_rerank(p["text"], caller, k=per_part) for p in independent])
    evidence = {p["id"]: r for p, r in zip(independent, results)}

    for p in [p for p in parts if p["dependent_on"]]:                       # sequential hops, bounded
        prior = summarise_evidence(evidence[p["dependent_on"]])
        resolved = llm("resolve_hop", HOP_SYS, [{"role": "user",
                       "content": f"<template>{p['text']}</template><known>{prior}</known>"}]).text
        evidence[p["id"]] = await retrieve_and_rerank(resolved, caller, k=per_part)

    blocks = "\n".join(
        f'<evidence for="{pid}" question="{next(x["text"] for x in parts if x["id"] == pid)}">\n'
        + "\n".join(f'<passage id="{c["id"]}">{c["text"]}</passage>' for c in chunks) + "\n</evidence>"
        for pid, chunks in evidence.items())
    return parts, blocks

SYNTH_SYS = """Answer the question completely. It has multiple parts; each <evidence> block corresponds to one part.
Address every part. If a part has no supporting evidence, state that explicitly rather than omitting it.
Cite passage ids per claim."""
```

## Pros

- **Coverage**: every part of the question gets its own retrieval budget; no part is crowded out.
- **Diagnosable**: you can see which sub-question failed to retrieve anything.
- **Better citations**: evidence is attributable per part.
- **Enables comparisons and multi-hop** questions that flat top-k structurally cannot answer.

## Cons

- **Latency and cost**: n× retrieval, n× reranking, plus a decomposition call; sequential hops serialise.
- **Bad decomposition is unrecoverable** — sub-questions that miss the point produce confidently incomplete answers (the [agents](../../../AI_System.md#agents) warning about decomposition applies here too).
- **Over-decomposition** of simple questions wastes budget and can fragment a coherent need into unanswerable pieces.
- **Context bloat**: 4 parts × 4 passages = 16 passages competing for the window ([per-source truncation budgets](../../../05-prompt-and-context-engineering/apply/2-context-engineering/per_source_truncation_budgets.md)).
- Sequential hops compound errors: a wrong entity extracted at hop 1 poisons hop 2 ([failure modes: poisoning, distraction, confusion, clash](../../../05-prompt-and-context-engineering/apply/2-context-engineering/context_failure_modes.md)).

## When to use

- Comparison questions ("A versus B"), multi-entity questions, questions with explicit conjunctions.
- Analytical questions over a corpus ("what changed and why").
- Genuine multi-hop lookups where the second query needs the first's answer.

## When NOT to use

- Simple factoid questions — gate it; decomposing "how many sick days" into three parts is pure waste.
- When the corpus is small enough that a single k=20 retrieval covers everything (measure before adding).
- Latency-critical single-turn search.
- Aggregation questions ("total spend across all vendors") — that's a SQL/structured routing problem ([structured + unstructured hybrid](../7-advanced-rag/structured_unstructured_hybrid.md)), not decomposition; splitting it into sub-questions gives you 40 numbers and an LLM doing arithmetic.

## Production at scale

**Gate on a compound-detection signal**, and fold the detection into the rewrite call ([query rewriting](query_rewriting.md)) so you pay for one LLM call, not two. Track the decomposition rate; if it fires on most queries, the detector is too loose.

**Per-part budget enforcement.** Total context budget divided across parts ([per-source truncation budgets](../../../05-prompt-and-context-engineering/apply/2-context-engineering/per_source_truncation_budgets.md)), not per-part unlimited. Rerank within each part and take a small top-n; the synthesis step sees 3–4 passages per part, not 20.

**Explicit "no evidence for this part"** in the synthesis prompt is the difference between a complete answer and a silently partial one. Evaluate it: golden cases where one part is genuinely unanswerable, and the expected output says so ([generation evals](../../../09-evaluation/apply/generation_evals.md)).

**Bound sequential hops** with a hard max (2–3) and a step budget enforced by the runtime ([cost & step budgets](../../../07-agents-tools-memory/apply/1-agents/cost_step_budgets.md)). Unbounded hop chains are how a query turns into a 40-second, $2 agent run.

**Parallelism and rate limits.** 4 sub-questions × (dense + BM25) = 8 concurrent searches plus 4 reranker calls; combined with multi-query ([multi-query + Reciprocal Rank Fusion](multi_query_rrf.md)) this multiplies fast. Cap the product of expansions; shed load by disabling decomposition before disabling retrieval ([capacity & limits](../../../10-production/apply/1-serving-cost-latency/capacity_limits.md)).

**Failure attribution in evals.** For each multi-hop golden case, record per-part recall. A low end-to-end score with high per-part recall means the synthesis prompt is at fault; low per-part recall means decomposition or retrieval is. This is the component-vs-end-to-end discipline ([component vs end-to-end](../../../09-evaluation/apply/component_vs_end_to_end.md)).

**Caching.** Sub-questions repeat across users ("German statutory minimum parental leave"); cache retrieval results by normalised sub-question + ACL signature — note the ACL signature, or you cache a leak.

## Prod checklist

- [ ] Compound detection folded into the rewrite call; decomposition rate monitored
- [ ] Max 3–5 sub-questions; per-part retrieval + rerank with small top-n
- [ ] Evidence labelled per sub-question in the context block
- [ ] Synthesis prompt requires addressing every part and stating unsupported parts explicitly
- [ ] Sequential hops bounded (≤3) with runtime step budget
- [ ] Total context budget split across parts, not per-part unlimited
- [ ] Retrieval cache keyed by sub-question **and ACL signature**
- [ ] Per-part recall recorded in multi-hop evals

## Metrics to watch

| Metric | Why |
|---|---|
| Per-part recall on multi-hop golden set | Decomposition + retrieval quality |
| Part-coverage rate in answers (all parts addressed) | Synthesis quality |
| Decomposition firing rate | Gating discipline |
| p95 latency and retrieval QPS multiplier | Cost |

## Related

- [Query rewriting](query_rewriting.md) — share the LLM call
- [Multi-query](multi_query_rrf.md) — different axis of expansion; cap the product
- [Recursive/agentic retrieval](../6-retrieval-ranking/recursive_agentic_retrieval.md) — the unbounded, model-driven cousin
- [Structured hybrid](../7-advanced-rag/structured_unstructured_hybrid.md) — for aggregation questions decomposition can't fix
- [Component vs end-to-end](../../../09-evaluation/apply/component_vs_end_to_end.md) — per-part attribution
