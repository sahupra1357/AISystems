# Reference architecture (end to end)

## What it is

```
                          ┌──────────────── Offline / batch ─────────────────┐
 sources ──► ingest ──► parse ──► chunk ──► embed ──► index (vector + BM25 + graph)
 (S3, SP,     dedupe,   route     parent/   version    metadata + ACL + tenant
  Confluence, hash,     per page  child,    pinned
  DB, web)    ACLs      validate  contextual

                          ┌──────────────── Online / request ────────────────┐
 user ──► authn/z ──► guardrails(in) ──► query understanding ──► route
                                          (rewrite, multi-query,   │
                                           decompose, filters)     │
                                                                   ▼
                                       hybrid retrieve (BM25 + vector, ACL-filtered)
                                                    │
                                                    ▼
                                          rerank (cross-encoder) ──► top-k
                                                    │
                                                    ▼
                              context assembly (budget, order, dedupe, cite)
                                                    │
                                                    ▼
                                    LLM / agent loop (tools, HITL gates)
                                                    │
                                                    ▼
                             grounding check ──► guardrails(out) ──► stream to user
                                                    │
                                                    ▼
                              traces, cost, feedback ──► eval set ──► flywheel
```

Every box maps to a file:

| Stage | Files |
|---|---|
| Ingest | [Connectors](../../06-rag/apply/1-ingestion-parsing/connectors.md) connectors · [change detection](../../06-rag/apply/1-ingestion-parsing/change_detection.md) change detection · [deduplication](../../06-rag/apply/1-ingestion-parsing/deduplication.md) dedupe · [document versioning](../../06-rag/apply/1-ingestion-parsing/document_versioning.md) versioning · [ACL](../../06-rag/apply/1-ingestion-parsing/tenant_acl_capture_at_ingest.md) |
| Parse | [Per-page classification and routing](../../06-rag/apply/1-ingestion-parsing/per_page_routing.md) routing · [OCR / digitization](../../AI_System.md#ocr--digitization) parsers · [structure extraction](../../AI_System.md#structure-extraction-what-the-parser-must-preserve) structure · [validation](../../AI_System.md#validation) validation · [bounded escalation with visible failure](../../06-rag/apply/1-ingestion-parsing/retry_ladder.md) retry ladder |
| Chunk | [Chunking](../../06-rag/apply/2-chunking/) |
| Embed | [Embeddings](../../06-rag/apply/3-embeddings/) |
| Index | [Indexing & vector stores](../../06-rag/apply/4-indexing/) |
| AuthN/Z + guardrails in | [Authentication & authorization](../../10-production/apply/6-security-privacy-governance/authn_authz_permission_filtered_retrieval.md) · [input guardrails](../../10-production/apply/5-guardrails/input_guardrails.md) |
| Query understanding + route | [Query understanding](../../06-rag/apply/5-query-understanding/) |
| Retrieve + rerank | [Retrieval & ranking](../../06-rag/apply/6-retrieval-ranking/) |
| Context assembly | [Context engineering](../../05-prompt-and-context-engineering/apply/2-context-engineering/) |
| Generate / agent | [Prompt engineering](../../05-prompt-and-context-engineering/apply/1-prompt-engineering/) · [generation & grounding](../../06-rag/apply/8-generation-grounding/) · [agents](../../AI_System.md#agents) · [memory](../../07-agents-tools-memory/apply/2-memory/) |
| Grounding + guardrails out | [Faithfulness check](../../06-rag/apply/8-generation-grounding/faithfulness_check.md) · [output guardrails](../../10-production/apply/5-guardrails/output_guardrails.md) |
| Observe + evaluate | [Observability & tracing](../../10-production/apply/4-observability-reliability/observability_tracing.md) · [evaluation](../../AI_System.md#evaluation) |
| Operate | [Production, cost & latency](../../10-production/apply/1-serving-cost-latency/) · [failure handling](../../10-production/apply/4-observability-reliability/failure_handling.md) · [security, privacy & governance](../../AI_System.md#security-privacy--governance) |
| Improve | [Data flywheel](../../10-production/apply/8-data-flywheel/) · [fine-tuning](../../08-choosing-the-lever/apply/fine-tuning/) |

## How to implement (build order)

1. **Golden set first** ([golden dataset first](../../09-evaluation/apply/golden_dataset.md)) — before any pipeline code. It defines what "correct" means and gates everything after.
2. **Thin vertical slice**: one connector → one parser → recursive chunking → embeddings → a vector index → simple retrieval → a grounded prompt with citations. Measure on the golden set.
3. **Security from day one**: tenant and ACL capture at ingest ([tenant / ACL capture at ingest](../../06-rag/apply/1-ingestion-parsing/tenant_acl_capture_at_ingest.md)), permission-filtered retrieval ([authentication & authorization](../../10-production/apply/6-security-privacy-governance/authn_authz_permission_filtered_retrieval.md)). Retrofitting is a re-ingest.
4. **Add the high-ROI quality layers in order**: hybrid search ([hybrid search](../../06-rag/apply/6-retrieval-ranking/hybrid_search.md)) → reranking ([reranking](../../06-rag/apply/6-retrieval-ranking/reranking.md)) → parent–child chunking ([parent–child](../../06-rag/apply/2-chunking/parent_child_small_to_big.md)) → query rewriting ([query rewriting](../../06-rag/apply/5-query-understanding/query_rewriting.md)) → abstention ([score thresholding & abstention](../../06-rag/apply/6-retrieval-ranking/score_thresholding_abstention.md)).
5. **Add the production layers**: observability ([observability & tracing](../../10-production/apply/4-observability-reliability/observability_tracing.md)), budgets ([per-source truncation budgets](../../05-prompt-and-context-engineering/apply/2-context-engineering/per_source_truncation_budgets.md), [cost & step budgets](../../07-agents-tools-memory/apply/1-agents/cost_step_budgets.md)), guardrails ([input guardrails](../../10-production/apply/5-guardrails/input_guardrails.md), [output guardrails](../../10-production/apply/5-guardrails/output_guardrails.md)), failure handling ([failure handling](../../10-production/apply/4-observability-reliability/failure_handling.md)), caching ([prompt caching of static prefixes](../../05-prompt-and-context-engineering/apply/2-context-engineering/prompt_caching.md)).
6. **Close the loop**: online signals ([online eval](../../09-evaluation/apply/online_eval.md)) → triage ([feedback capture & triage](../../10-production/apply/8-data-flywheel/feedback_capture_triage.md)) → golden cases ([data flywheel](../../10-production/apply/8-data-flywheel/data_flywheel.md)) → CI gates ([the testing pyramid](../../09-evaluation/apply/testing_pyramid.md)) → canary ([versioning & rollout](../../10-production/apply/1-serving-cost-latency/versioning_rollout.md)).
7. **Only then** consider agents ([agents](../../07-agents-tools-memory/apply/1-agents/)), advanced RAG ([advanced RAG patterns](../../06-rag/apply/7-advanced-rag/)), and fine-tuning ([fine-tuning](../../08-choosing-the-lever/apply/fine-tuning/)) — each justified by a measured gap.

## Small example — the online path, assembled

```python
def handle_request(raw_query: str, request, cfg) -> dict:
    trace_id = new_trace_id()
    with span("app.request", trace_id=trace_id, **version_vector(cfg)):          # Observability & tracing

        caller = authenticate(request)                                            # Authentication & authorization
        guard = input_guardrails(raw_query, caller, policy_for(caller))           # Input guardrails
        if not guard.allow:
            return refusal(guard.reason, trace_id)

        understanding = understand_query(guard.masked_text, caller, history())    # Query rewriting/.4/.5/.6
        if understanding.route != "vector":
            return HANDLERS[understanding.route](understanding, caller, trace_id) # Routing, structured + unstructured hybrid

        budget = Budget(max_tokens=cfg["budgets"]["tokens"], max_usd=cfg["budgets"]["usd"],
                        max_seconds=cfg["budgets"]["seconds"])                    # Cost & step budgets

        pool = hybrid_retrieve(understanding.query, caller,                        # Hybrid search (ACL-filtered, authentication & authorization)
                               k=cfg["retrieval"]["pool_k"], filters=understanding.filters)
        ranked = rerank(understanding.query, pool, top_n=cfg["retrieval"]["final_k"])  # Reranking
        ranked = mmr_select(ranked, understanding.query_vec, k=cfg["retrieval"]["final_k"])  # MMR / diversity

        conf = retrieval_confidence(ranked)                                        # Score thresholding & abstention
        if conf["top"] < cfg["thresholds"]["tau_low"]:
            return refuse(RefusalReason.NO_RELEVANT, {...}, trace_id)              # Refusal / no-answer path

        parents = collapse_to_parents(ranked)                                      # Parent–child
        memory = retrieve_memory(caller, understanding.task_type, understanding.query)  # When to retrieve memory
        context, meta = assemble(understanding.query, parents, memory, history(),  # Context engineering
                                 budget=cfg["budgets"]["context_tokens"])

        out = generate_grounded(understanding.query, context, budget=budget)       # Prompt structure, grounded prompting
        out = validate_citations(out, parents)                                     # Citations
        out = faithfulness_gate(out, parents)                                      # Faithfulness check
        verdict = output_guardrails(out, parents, caller, policy_for(caller))      # Output guardrails

        record_outcome(trace_id, out, ranked, meta, budget)                        # Observability & tracing → online eval → data flywheel
        audit.record("ai.query_answered", caller, understanding, out, ranked)      # Auditability
        return {"answer": verdict.text, "citations": render_citations(out, parents),
                "trace_id": trace_id, "degraded": out.get("degraded", False)}
```

## Pros

- **Every stage has a named owner, a metric, and a failure mode** — the architecture is also the operations model.
- **Each layer is independently evaluable** ([component vs end-to-end](../../09-evaluation/apply/component_vs_end_to_end.md)), so problems localise.
- **Offline/online separation** lets ingestion scale independently of serving.
- **The flywheel is part of the architecture**, not an afterthought, which is what makes the system improve.

## Cons

- **It's a lot of machinery** — a naive prototype is five components, this is thirty. Build incrementally against measured gaps.
- **Every stage is a failure point**; without observability the composite is hard to debug.
- **Cost and latency accumulate** across stages; each needs a budget.
- Over-engineering is the common failure: teams build advanced RAG patterns before measuring whether hybrid + rerank suffices.

## When to use

As the map for any document-grounded AI system. Use it to locate what you're building, and what you're missing.

## When NOT to use (all of it)

- A prototype needs the thin slice, not the full diagram.
- Not every system needs agents, graphs, multimodal retrieval, or fine-tuning — each is justified by a measured gap, not by the diagram's completeness.

## Production at scale

**The order of investment is not the order of the diagram.** After a working thin slice, the highest-return additions are consistently: reranking ([reranking](../../06-rag/apply/6-retrieval-ranking/reranking.md)), hybrid search ([hybrid search](../../06-rag/apply/6-retrieval-ranking/hybrid_search.md)), parent–child chunking ([parent–child](../../06-rag/apply/2-chunking/parent_child_small_to_big.md)), and abstention ([score thresholding & abstention](../../06-rag/apply/6-retrieval-ranking/score_thresholding_abstention.md)). Advanced RAG patterns ([advanced RAG patterns](../../06-rag/apply/7-advanced-rag/)), agents ([agents](../../07-agents-tools-memory/apply/1-agents/)), and fine-tuning ([fine-tuning](../../08-choosing-the-lever/apply/fine-tuning/)) come far later, and only when evals show the specific gap they close.

**Security and versioning cannot be retrofitted cheaply.** ACL capture at ingest ([tenant / ACL capture at ingest](../../06-rag/apply/1-ingestion-parsing/tenant_acl_capture_at_ingest.md)) and document versioning ([document versioning](../../06-rag/apply/1-ingestion-parsing/document_versioning.md)) touch every chunk; adding them later means re-ingesting the corpus. Build them into the thin slice.

**Observability is a prerequisite, not a phase.** Without retrieved chunk IDs and scores in the trace ([observability & tracing](../../10-production/apply/4-observability-reliability/observability_tracing.md)), every debugging session is guesswork and the flywheel cannot start.

**The offline path usually costs more than the online path** in document-heavy systems: parsing, embedding, and contextual enrichment dominate. Change detection ([change detection](../../06-rag/apply/1-ingestion-parsing/change_detection.md)) and dedupe ([deduplication](../../06-rag/apply/1-ingestion-parsing/deduplication.md)) are the largest cost levers most teams haven't pulled ([cost levers](../../10-production/apply/1-serving-cost-latency/cost_levers.md)).

**Budgets at every stage**: context tokens ([per-source truncation budgets](../../05-prompt-and-context-engineering/apply/2-context-engineering/per_source_truncation_budgets.md)), agent steps and dollars ([cost & step budgets](../../07-agents-tools-memory/apply/1-agents/cost_step_budgets.md)), per-tenant quotas ([capacity & limits](../../10-production/apply/1-serving-cost-latency/capacity_limits.md)). Unbudgeted stages are where cost incidents originate.

**Each stage needs a degraded mode** ([failure handling](../../10-production/apply/4-observability-reliability/failure_handling.md)): reranker down → fusion order; dense down → BM25; generation down → return sources. The composite availability of ten dependencies without degradation is poor.

**Keep the version vector everywhere** ([versioning & rollout](../../10-production/apply/1-serving-cost-latency/versioning_rollout.md), [observability & tracing](../../10-production/apply/4-observability-reliability/observability_tracing.md)) — it's what makes a thirty-component system debuggable at all.

## Prod checklist

- [ ] Golden set exists before the pipeline
- [ ] Thin vertical slice working end to end, measured
- [ ] ACL capture and document versioning built in from the start
- [ ] Hybrid + rerank + parent–child + abstention added and measured in that order
- [ ] Observability with chunk IDs, scores, and the version vector on every request
- [ ] Budgets at every stage; per-tenant quotas
- [ ] Degraded mode defined for each stage
- [ ] CI eval gates and canary rollout wired
- [ ] Flywheel running: signals → triage → golden cases → fixes
- [ ] Advanced components (agents, graph, multimodal, fine-tuning) justified by measured gaps

## Metrics to watch

| Metric | Why |
|---|---|
| Per-stage latency and cost | Where the budget goes |
| Failure-bucket histogram ([feedback capture & triage](../../10-production/apply/8-data-flywheel/feedback_capture_triage.md)) | Which stage to invest in |
| End-to-end quality on the golden set | The headline |
| Degraded-response rate by stage | Composite reliability |
| Ingest cost per 1k documents | The usually-dominant line item |

## Related

Every file in this reference maps to a stage in the diagram above; see the table in *What it is*.

## Video takeaway (30 seconds)

"Two pipelines and a loop. Offline: ingest with permissions and versions attached, parse per page, chunk with structure, embed, index into vector and keyword stores. Online: authenticate, guard the input, understand and route the query, retrieve hybrid with the caller's permission filter, rerank, assemble a budgeted context, generate with grounding, check citations and faithfulness, guard the output, stream. Then traces and feedback flow into the flywheel. Build the thin slice first and measure it — but put permission capture and document versioning in from day one, because those two are the ones you cannot retrofit without re-ingesting everything you own."
