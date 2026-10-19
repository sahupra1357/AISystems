# Production → Latency levers

## What it is

Users judge **time to first token** and whether the interface feels alive, far more than total completion time. A 6-second answer that starts streaming in 400 ms feels faster than a 3-second answer that appears all at once.

The latency budget of a typical RAG request:

```
guardrails (in)      20–80 ms    ├─ can run parallel with query embedding
query rewrite       150–400 ms   ├─ skippable via heuristic
embed query          10–40 ms    ├─ parallel with rewrite (speculative)
retrieval (hybrid)   20–100 ms   ├─ dense and sparse in parallel
rerank               50–300 ms   ├─ hosted low-latency endpoint or self-hosted GPU
context assembly      5–20 ms
generation TTFT     300–900 ms   ← what the user actually feels
generation stream   500–3000 ms  ← perceived as progress, not waiting
guardrails (out)     10–200 ms   ← conflicts with streaming (output guardrails)
```

| Lever | Effect | Cost |
|---|---|---|
| **Streaming** | TTFT becomes the perceived latency | Complicates output guardrails |
| **Semantic cache** | Near-zero latency on cache hits (10–40% of traffic in some products) | Staleness and false-hit risk |
| **Parallelism** | Removes serial hops from the critical path | Concurrency and rate-limit pressure |
| **Speculative prefetch** while typing | Hides retrieval latency entirely | Wasted work on abandoned queries |
| **Smaller model on the critical path** | Large TTFT and throughput gains | Quality trade-off, needs evals |
| **Prompt caching** | Cuts prefill time substantially | Prefix discipline ([prompt caching of static prefixes](../../../05-prompt-and-context-engineering/apply/2-context-engineering/prompt_caching.md)) |
| **Hosted/low-latency reranker** | 50–150 ms instead of 300+ | Cost or GPU ops |

## How to implement

1. **Measure the breakdown first** ([observability & tracing](../4-observability-reliability/observability_tracing.md)): p50/p95 per stage. Optimise the stage that dominates p95, not the one that feels slow.
2. **Stream by default**, with a visible "searching…" state during retrieval so the pre-TTFT period isn't blank.
3. **Parallelise everything independent**: guardrails ∥ query embedding ∥ ACL resolution; dense ∥ sparse retrieval; multi-query variants ([multi-query + Reciprocal Rank Fusion](../../../06-rag/apply/5-query-understanding/multi_query_rrf.md)); independent tool calls.
4. **Skip optional stages by heuristic**: query rewriting only when the turn has pronouns ([query rewriting](../../../06-rag/apply/5-query-understanding/query_rewriting.md)); multi-query only for short queries ([multi-query + Reciprocal Rank Fusion](../../../06-rag/apply/5-query-understanding/multi_query_rrf.md)); HyDE only on low top-score ([hyDE](../../../06-rag/apply/5-query-understanding/hyde.md)).
5. **Semantic cache** with an embedding-similarity threshold, scoped per tenant and per ACL signature, with a short TTL.
6. **Speculative prefetch**: begin retrieval on a debounced partial query while the user is still typing; discard if the final query diverges.
7. **Prompt caching** for prefill reduction ([prompt caching of static prefixes](../../../05-prompt-and-context-engineering/apply/2-context-engineering/prompt_caching.md)), and a smaller model on latency-critical paths validated by evals ([model selection & tradeoffs](../../../04-llm-foundations/apply/model_selection.md)).

## Small example

```python
import asyncio, hashlib

# --- 1. Parallelise the independent prep work ---
async def prepare(query, caller, history):
    guard_t   = asyncio.create_task(input_guardrails_async(query, caller))
    embed_t   = asyncio.create_task(embed_query_async(query))          # speculative: original query
    acl_t     = asyncio.create_task(resolve_groups_async(caller))
    rewrite_t = asyncio.create_task(rewrite_async(history, query)) if needs_rewrite(query) else None

    guard, qvec, groups = await asyncio.gather(guard_t, embed_t, acl_t)
    rewritten = await rewrite_t if rewrite_t else query
    if rewritten != query:
        qvec = await embed_query_async(rewritten)                       # cheap re-embed only if changed
    return guard, qvec, groups, rewritten

# --- 2. Semantic cache: tenant- and ACL-scoped, never global ---
class SemanticCache:
    def __init__(self, vec, threshold=0.95, ttl_s=900):
        self.vec, self.threshold, self.ttl_s = vec, threshold, ttl_s

    def _scope(self, caller) -> str:
        return hashlib.sha256(f"{caller.tenant_id}|{sorted(caller.groups)}".encode()).hexdigest()[:16]

    def get(self, query, caller):
        hits = self.vec.search(embed_queries([query])[0], limit=1,
                               filter={"scope": self._scope(caller), "fresh_after": time.time() - self.ttl_s})
        if hits and hits[0].score >= self.threshold:
            log_cache_hit(query, hits[0].payload["query"], hits[0].score)
            return hits[0].payload["answer"]                             # ACL-identical callers only
        return None

    def put(self, query, caller, answer, abstained: bool):
        if abstained or answer is None:
            return                                                       # never cache refusals or errors
        self.vec.upsert(id=uuid.uuid4().hex, vector=embed_queries([query])[0],
                        payload={"query": query, "answer": answer, "scope": self._scope(caller),
                                 "fresh_after": time.time()})

# --- 3. Streaming with a visible pre-TTFT state ---
async def stream_answer(query, caller, history, send):
    await send({"type": "status", "text": "Searching documents…"})       # the blank period is now informative
    guard, qvec, groups, rewritten = await prepare(query, caller, history)
    if (cached := semantic_cache.get(rewritten, caller)):
        await send({"type": "cached_answer", "text": cached}); return

    dense_t  = asyncio.create_task(vector_search_async(qvec, groups))
    sparse_t = asyncio.create_task(bm25_search_async(rewritten, groups))
    pool = rrf({"dense": await dense_t, "sparse": await sparse_t})
    chunks = await rerank_async(rewritten, pool)                          # low-latency endpoint
    await send({"type": "sources", "sources": summarise_sources(chunks)}) # show sources BEFORE the answer

    buffer = ""
    async for delta in generate_stream(rewritten, chunks):
        buffer += delta
        if deterministic_output_checks_ok(buffer):                        # cheap checks stream-safe (output guardrails)
            await send({"type": "delta", "text": delta})
        else:
            await send({"type": "error", "text": "Response withheld by policy."}); return
    verdict = output_guardrails(parse(buffer), chunks, caller, policy_for(caller), risk="high")
    if not verdict.deliver:
        await send({"type": "correction", "text": verdict.text})
    semantic_cache.put(rewritten, caller, buffer, abstained=False)

# --- 4. Speculative prefetch while typing ---
async def on_typing_pause(partial_query, caller):
    if len(partial_query.split()) >= 4:
        asyncio.create_task(warm_retrieval_cache(partial_query, caller))  # discard if the query changes
```

## Pros

- **Streaming transforms perceived latency** at almost no engineering cost — the highest-leverage change available.
- **Parallelism removes hundreds of milliseconds** from the critical path with no quality cost.
- **Semantic caching gives near-zero latency** on repeated questions, common in support and internal search.
- **Showing sources before the answer** gives users something to read during generation.
- Prompt caching cuts prefill time as well as cost ([prompt caching of static prefixes](../../../05-prompt-and-context-engineering/apply/2-context-engineering/prompt_caching.md)).

## Cons

- **Streaming conflicts with output guardrails** ([output guardrails](../5-guardrails/output_guardrails.md)): you either buffer (losing the benefit) or stream and risk retracting.
- **Semantic caching risks wrong answers**: a near-match that isn't equivalent, or a stale answer after the corpus changed. Requires ACL scoping, tight thresholds, and short TTLs.
- **Parallelism raises concurrency** and rate-limit pressure ([capacity & limits](capacity_limits.md)).
- **Speculative prefetch wastes work** on abandoned queries — bounded, but real cost.
- Smaller models trade quality; must be eval-validated ([model selection & tradeoffs](../../../04-llm-foundations/apply/model_selection.md)).

## When to use

- **Streaming**: every interactive interface.
- **Parallelism**: everywhere independent work exists.
- **Semantic cache**: high query-repetition products (support, internal search, FAQ-like traffic).
- **Prefetch**: search-box interfaces with typing pauses.

## When NOT to use

- **Semantic caching on personalised or permission-sensitive answers** without exact ACL scoping — a cache hit across permission boundaries is a data leak.
- Caching abstentions or errors — they'd persist past the fix.
- Streaming on outputs that must be validated atomically (financial figures, medical advice) — buffer and check.

## Production at scale

**Optimise p95, not p50.** The median request is usually fine; the tail is what users complain about. Instrument per stage and attack the dominant p95 contributor ([observability & tracing](../4-observability-reliability/observability_tracing.md)).

**Semantic cache scoping is a security control, not a performance detail.** Key by tenant *and* the caller's ACL signature; two users with different permissions must never share a cache entry ([tenant / ACL capture at ingest](../../../06-rag/apply/1-ingestion-parsing/tenant_acl_capture_at_ingest.md), [authentication & authorization](../6-security-privacy-governance/authn_authz_permission_filtered_retrieval.md)). Invalidate on corpus updates for affected documents, and keep TTLs short.

**Cache the retrieval, not just the answer.** Retrieval results keyed by `(normalised_query, acl_signature)` hit far more often than full answers and carry less staleness risk.

**Show progress, not spinners.** "Searching documents…", then the source list, then streaming text. Users tolerate multi-second waits when they can see work happening; a blank spinner at 3 seconds reads as broken.

**Skip stages adaptively under load** ([capacity & limits](capacity_limits.md)): drop multi-query, then HyDE, then reranking depth, before dropping retrieval or degrading the model. Define the shed order in advance.

**TTFT is the SLO.** Set and alert on it separately from total latency. Prompt caching, smaller prefill, and a faster model tier are the levers that move it.

**Agent latency is a different problem** — it's dominated by step count, not per-call latency ([cost & step budgets](../../../07-agents-tools-memory/apply/1-agents/cost_step_budgets.md)). Stream tool-call narration so multi-second agent runs feel like progress ([agentic RAG](../../../06-rag/apply/7-advanced-rag/agentic_rag.md)).

**Measure the abandoned-query rate** at each latency band; it's the business consequence of latency and the justification for the work.

## Prod checklist

- [ ] Per-stage latency instrumented; p95 breakdown reviewed
- [ ] Streaming enabled with pre-TTFT status and source list shown before the answer
- [ ] Independent work parallelised (guardrails ∥ embedding ∥ ACL; dense ∥ sparse; multi-query variants)
- [ ] Optional stages gated by heuristics (rewrite, multi-query, HyDE)
- [ ] Semantic cache scoped by tenant + ACL signature, short TTL, invalidated on corpus change; refusals never cached
- [ ] Retrieval-level caching keyed by query + ACL signature
- [ ] Prompt caching for prefill reduction; TTFT SLO with alerts
- [ ] Load-shedding order defined for latency pressure
- [ ] Streaming vs output-guardrail strategy decided and documented
- [ ] Abandonment rate tracked by latency band

## Metrics to watch

| Metric | Why |
|---|---|
| TTFT p50/p95 | What users actually perceive |
| Per-stage latency breakdown (p95) | Where to optimise |
| Semantic cache hit rate and false-hit rate | Value vs risk |
| Abandonment rate by latency band | Business impact |
| Streaming interruption rate (guardrail retractions) | UX cost of safety checks |

## Related

- [Prompt caching](../../../05-prompt-and-context-engineering/apply/2-context-engineering/prompt_caching.md) — cost and prefill latency
- [Query understanding](../../../06-rag/apply/5-query-understanding/) Query understanding — the skippable stages
- [Output guardrails](../5-guardrails/output_guardrails.md) — the streaming conflict
- [Capacity & limits](capacity_limits.md) — shedding under load
- [Cost levers](cost_levers.md) — semantic caching overlaps

## Video takeaway (30 seconds)

"Users judge time to first token, not total time. A six-second answer that starts in four hundred milliseconds feels faster than a three-second one that appears all at once. So stream, and fill the pre-token gap with real information: 'searching documents', then the list of sources you found. Parallelise everything independent — guardrails, embedding, permission resolution, dense and keyword search. Skip optional stages when a cheap heuristic says they won't help. And if you add a semantic cache, scope it by tenant *and* the caller's permissions — a cache hit across a permission boundary isn't a latency win, it's a data leak."
