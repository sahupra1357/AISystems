# Query understanding → Query rewriting (coreference over chat history)

## Video takeaway

"'What about its termination notice?' — the retriever has no idea what 'its' is. Before you search, rewrite the turn into a standalone query using the conversation: 'termination notice period of the Acme MSA'. One cheap model call, skippable when the query is already self-contained. Guard against the model inventing entities: if the rewrite is way longer than the original or mentions something never discussed, fall back. Keep the user's original words for the answer prompt — the rewrite is for search only. This is the fix for RAG that works on turn one and falls apart on turn three."

## What it is

Users talk in context: "and the notice period?", "same for the UK entity", "what did you say the limit was?", "compare it with the 2023 version". The retriever sees only the current turn. Query rewriting (a.k.a. contextual query reformulation, conversational query rewriting) uses a cheap LLM call to produce a **self-contained search query** that carries the entities and intent from prior turns:

```
history: U: What is the renewal term of the Acme MSA?  A: … 12 months … [c14]
turn:    U: what about its termination notice?
rewrite: "termination notice period of the Acme MSA"
```

It also normalises: expands abbreviations the corpus uses, fixes typos, strips politeness, and can emit multiple queries ([multi-query + Reciprocal Rank Fusion](multi_query_rrf.md)) or filters ([self-query](self_query_metadata_filters.md)). It is *not* the answer generation; it is a retrieval preprocessing step, and it should be evaluated as one.

## How to implement

1. **Trigger**: run on every turn after the first (or when the turn has pronouns/ellipsis/short length — a cheap heuristic can skip obviously standalone queries to save latency).
2. **Inputs**: compacted history ([compaction / summarisation of long history](../../../05-prompt-and-context-engineering/apply/2-context-engineering/compaction_summarization.md)) — last N turns + summary + pinned entities — and the current turn. Keep it short; the rewriter doesn't need the full transcript.
3. **Prompt** (fast tier, temperature 0): "Rewrite the last user message as a standalone search query that includes all entities and constraints implied by the conversation. Do not answer. Output the query only." Provide 2–3 examples ([zero-shot and few-shot prompting](../../../05-prompt-and-context-engineering/apply/1-prompt-engineering/zero_few_shot.md)) including a "no change needed" case.
4. **Guardrails**: if the rewrite is much longer than the original or introduces entities absent from history, fall back to the original (hallucinated rewrites are a real failure). Never let the rewrite drop explicit constraints the user stated.
5. **Retrieve with the rewrite**; keep the original for the generation prompt (the model should see what the user actually said) and log both.
6. **Optionally emit structure**: `{"query": ..., "filters": {...}, "sub_queries": [...]}` — combining rewriting with self-query ([self-query](self_query_metadata_filters.md)) and decomposition ([decomposition](decomposition.md)) in one call saves latency.
7. **Evaluate** on multi-turn golden conversations: retrieval recall at turn ≥ 2 with vs without rewriting ([retrieval evals](../../../09-evaluation/apply/retrieval_evals.md)).

## Small example

```python
REWRITE_SYS = """You rewrite the latest user message into a standalone search query for a document retrieval system.
Rules:
- Include every entity, document name, date, and constraint the message refers to, resolved from the conversation.
- Do not answer the question. Do not add information not implied by the conversation.
- If the message is already standalone, return it unchanged.
- Output JSON: {"query": string, "changed": boolean}

<example>
<history>U: What is the renewal term of the Acme MSA? A: The Acme MSA renews for 12 months [c14].</history>
<message>what about its termination notice?</message>
<output>{"query": "termination notice period of the Acme MSA", "changed": true}</output>
</example>
<example>
<history>U: hello</history>
<message>What is our parental leave policy in Germany?</message>
<output>{"query": "What is our parental leave policy in Germany?", "changed": false}</output>
</example>"""

PRONOUN = re.compile(r"\b(it|its|that|this|they|them|same|there|those|the (former|latter))\b", re.I)

def rewrite(history_block: str, message: str) -> str:
    if len(message.split()) > 12 and not PRONOUN.search(message):
        return message                                                     # cheap skip for standalone queries
    out = llm("query_rewrite", REWRITE_SYS, [{"role": "user", "content": f"<history>{history_block}</history>\n<message>{message}</message>"}], max_tokens=120)
    try:
        j = json.loads(out.text); q = j["query"].strip()
    except Exception:
        return message
    if len(q) > 3 * len(message) + 40 or not q:                              # runaway / hallucinated rewrite guard
        return message
    log_rewrite(message, q, changed=j.get("changed"))
    return q
```

## Pros

- **Multi-turn retrieval works** — the biggest single fix for "correct at turn 1, wrong at turn 3" ([common failure modes](../../../AI_System.md#common-failure-modes-and-the-fix)).
- **Cheap**: one fast-tier call, ~100 tokens out; skippable heuristically.
- **Normalisation** (abbreviations, typos) improves BM25 as well as dense retrieval.
- Can carry filters and sub-queries in the same call.

## Cons

- **Latency** on the critical path (50–300 ms); mitigated by the skip heuristic and parallelising with other prep.
- **Hallucinated rewrites**: the model invents an entity or "helpfully" changes the question; guardrails and evals needed.
- **Over-rewriting** simple queries into verbose ones can hurt BM25 (extra terms).
- Depends on history quality ([compaction / summarisation of long history](../../../05-prompt-and-context-engineering/apply/2-context-engineering/compaction_summarization.md)): if the summary lost the entity, the rewrite can't recover it.

## When to use

- Every conversational RAG system, from turn 2 onward.
- Single-turn systems with heavy abbreviation/typo traffic (rewrite as normalisation).

## When NOT to use

- Single-turn search boxes with clean queries — no history to resolve; skip.
- Ultra-low-latency autocomplete-style retrieval.
- Don't use the rewrite as the text shown to the user or as the generation question — keep the original for those.

## Production at scale

**Latency budget**: run the rewrite in parallel with query embedding of the *original* (speculative) and with ACL resolution; if the rewrite comes back changed, embed and search again (fast) — or just accept the small serial cost with the skip heuristic covering most turns.

**Prompt caching** ([prompt caching of static prefixes](../../../05-prompt-and-context-engineering/apply/2-context-engineering/prompt_caching.md)): the rewrite system prompt with examples is a stable prefix.

**Guardrails as metrics**: rewrite-changed rate, fallback rate (guard triggered), rewrite length ratio. A spike in fallbacks after a model change is a regression.

**Golden multi-turn conversations** with expected rewrites and expected relevant chunks; measure both rewrite exact/semantic match and downstream recall ([retrieval evals](../../../09-evaluation/apply/retrieval_evals.md)). Include adversarial turns ("actually, forget Acme — the Globex one").

**Entity pinning**: keep key entities (document names, IDs, tenant-specific terms) in the pinned block ([compaction / summarisation of long history](../../../05-prompt-and-context-engineering/apply/2-context-engineering/compaction_summarization.md)) so rewrites can always reference them.

**Domain vocabulary**: give the rewriter a short glossary of corpus abbreviations (per tenant if needed) — "MSA", "PTO", product codes — so queries normalise to the corpus's terms; a tenant overlay block ([prompt structure](../../../05-prompt-and-context-engineering/apply/1-prompt-engineering/prompt_structure.md)).

**Logging**: original + rewrite + changed flag on every retrieval trace ([observability & tracing](../../../10-production/apply/4-observability-reliability/observability_tracing.md)); this is essential for debugging "why did it retrieve that".

**Privacy**: rewrites contain history content; same retention rules ([data residency & retention](../../../10-production/apply/6-security-privacy-governance/data_residency_retention.md)).

## Prod checklist

- [ ] Rewrite step on turns ≥ 2 with a cheap skip heuristic
- [ ] Fast tier, temperature 0, cached system prompt with examples incl. "unchanged"
- [ ] Guardrails: length ratio, empty, parse failure → fallback to original
- [ ] Original kept for generation/UI; rewrite used for retrieval; both logged
- [ ] Pinned entities and tenant glossary available to the rewriter
- [ ] Multi-turn golden conversations; recall at turn ≥ 2 tracked with/without rewrite
- [ ] Changed/fallback rates monitored per model version

## Metrics to watch

| Metric | Why |
|---|---|
| Retrieval recall at turn ≥ 2 (with vs without rewrite) | The value |
| Rewrite changed rate; fallback rate | Health/regression |
| Rewrite latency p95 | Critical path |
| Hallucinated-entity rate (eval) | Guardrail effectiveness |

## Related

- [Compaction](../../../05-prompt-and-context-engineering/apply/2-context-engineering/compaction_summarization.md) — history the rewriter reads
- [Multi-query](multi_query_rrf.md), [Decomposition](decomposition.md), [Self-query](self_query_metadata_filters.md) — can share the call
- [Retrieval evals](../../../09-evaluation/apply/retrieval_evals.md)
- 15 Common failure modes — "correct at turn 1, wrong at turn 3"
