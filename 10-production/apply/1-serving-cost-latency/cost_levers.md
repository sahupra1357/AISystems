# Production → Cost levers

## Video takeaway

"Cost is tokens times price, and the levers are ranked. Prompt caching first — it's the biggest single win, it's quality-neutral, and it can cut input costs by ninety percent, but one variable token in your prefix silently kills it, so monitor the hit rate like an error rate. Then tiering: classification and query rewriting don't need your best model. Then rerank fifty candidates and send six, not twenty. Batch everything offline at half price. Together that's routinely a ten to twenty-five times reduction. And track cost per *successful task*, not per request — a cheap system that fails half the time isn't cheap."

## What it is

LLM cost is `tokens × price`, and every lever moves one of those two. In rough order of impact for a typical chat/RAG product:

| Lever | Typical saving | Effort | Where |
|---|---|---|---|
| **Prompt caching** of static prefixes | 50–90% of input cost | Low | [Prompt caching of static prefixes](../../../05-prompt-and-context-engineering/apply/2-context-engineering/prompt_caching.md) |
| **Model routing / tiering** | 40–80% of blended cost | Medium | [Model selection & tradeoffs](../../../04-llm-foundations/apply/model_selection.md), [routing & escalation](../../../04-llm-foundations/apply/routing_escalation.md) |
| **Trim top-k after reranking** | 30–60% of context tokens | Low | [Reranking](../../../06-rag/apply/6-retrieval-ranking/reranking.md) |
| **Batch API for offline work** | ~50% on batched jobs | Low | ingestion, backfills |
| **Semantic caching** of near-identical queries | 10–40% of requests | Medium | [Latency levers](latency_levers.md) |
| **Shorter system prompts / fewer examples** | 10–30% of input tokens | Low | [Prompt structure](../../../05-prompt-and-context-engineering/apply/1-prompt-engineering/prompt_structure.md), [zero-shot and few-shot prompting](../../../05-prompt-and-context-engineering/apply/1-prompt-engineering/zero_few_shot.md) |
| **Token budgets per request** | Caps the tail | Low | [Per-source truncation budgets](../../../05-prompt-and-context-engineering/apply/2-context-engineering/per_source_truncation_budgets.md), [cost & step budgets](../../../07-agents-tools-memory/apply/1-agents/cost_step_budgets.md) |
| **Cheaper embeddings / smaller dimensions** | Ingest + index cost | Medium | [Model choice](../../../06-rag/apply/3-embeddings/embedding_model_choice.md) |
| **Change detection / dedupe** at ingest | 20–60% of ingest cost | Medium | [Change detection](../../../06-rag/apply/1-ingestion-parsing/change_detection.md), [deduplication](../../../06-rag/apply/1-ingestion-parsing/deduplication.md) |
| **Self-hosting** at high steady volume | Variable | High | [Self-hosted serving](self_hosted_serving.md) |

The two numbers that matter operationally: **$/task** (cost per completed unit of user value, not per API call) and **$/user/month**. Both need cost attribution per call site, model, tenant, and feature ([observability & tracing](../4-observability-reliability/observability_tracing.md)).

## How to implement

1. **Instrument first.** Every LLM call records model, call site, tokens in/out/cached, and computed cost, tagged with tenant and feature ([observability & tracing](../4-observability-reliability/observability_tracing.md)). You cannot optimise what you cannot attribute.
2. **Build the cost model**: `$/request = Σ(call sites) calls × (in_tokens × price_in + cached × price_cache + out × price_out)`. Roll up to $/task and $/user/month.
3. **Apply caching first** — the largest, cheapest win. Restructure prompts stable-first and verify cache hit rate ([prompt caching of static prefixes](../../../05-prompt-and-context-engineering/apply/2-context-engineering/prompt_caching.md)).
4. **Tier and route**: cheap models for classification/extraction/rewriting; reasoning models only for final answers and planning ([model selection & tradeoffs](../../../04-llm-foundations/apply/model_selection.md), [routing & escalation](../../../04-llm-foundations/apply/routing_escalation.md)).
5. **Cut context**: rerank then trim to the smallest k that holds quality on the golden set; enforce per-source budgets ([per-source truncation budgets](../../../05-prompt-and-context-engineering/apply/2-context-engineering/per_source_truncation_budgets.md), [reranking](../../../06-rag/apply/6-retrieval-ranking/reranking.md)).
6. **Batch everything offline**: contextual retrieval generation, embeddings, backfills, evals — provider batch endpoints are typically half price.
7. **Set budgets and alerts** per request, per user, per tenant, per day ([cost & step budgets](../../../07-agents-tools-memory/apply/1-agents/cost_step_budgets.md), [capacity & limits](capacity_limits.md)), and review the cost dashboard weekly.

## Small example

```python
PRICES = {  # $/million tokens — keep in config, refresh on provider changes
    "claude-haiku-4-5-20251001": {"in": 1.00, "out": 5.00,  "cache_write": 1.25, "cache_read": 0.10},
    "claude-sonnet-5":           {"in": 3.00, "out": 15.00, "cache_write": 3.75, "cache_read": 0.30},
    "claude-opus-5":             {"in": 15.00,"out": 75.00, "cache_write": 18.75,"cache_read": 1.50},
}

def cost_of(model, usage) -> float:
    p = PRICES[model]
    return (usage.input_tokens * p["in"]
            + getattr(usage, "cache_creation_input_tokens", 0) * p["cache_write"]
            + getattr(usage, "cache_read_input_tokens", 0) * p["cache_read"]
            + usage.output_tokens * p["out"]) / 1_000_000

def model_request_cost(profile: dict) -> dict:
    """profile: {call_site: {model, calls, in, cached_in, out}} for one user request."""
    total, by_site = 0.0, {}
    for site, c in profile.items():
        p = PRICES[c["model"]]
        cost = c["calls"] * (c["in"] * p["in"] + c["cached_in"] * p["cache_read"] + c["out"] * p["out"]) / 1e6
        by_site[site] = round(cost, 5); total += cost
    return {"total_usd": round(total, 5), "by_call_site": by_site}

# Before / after the standard levers
BEFORE = {                                   # everything on the reasoning tier, no caching, k=20
    "answer": {"model": "claude-opus-5", "calls": 1, "in": 26_000, "cached_in": 0, "out": 700},
    "rewrite": {"model": "claude-opus-5", "calls": 1, "in": 1_200, "cached_in": 0, "out": 80},
}
AFTER = {                                    # tiered + cached prefix + rerank-and-trim to k=6
    "answer": {"model": "claude-sonnet-5", "calls": 1, "in": 2_400, "cached_in": 5_600, "out": 700},
    "rewrite": {"model": "claude-haiku-4-5-20251001", "calls": 1, "in": 300, "cached_in": 900, "out": 80},
}
# model_request_cost(BEFORE)["total_usd"] ≈ 0.46   →  AFTER ≈ 0.018   (~25x)

def cost_dashboard(traces, period="7d"):
    import pandas as pd
    df = pd.DataFrame(traces)
    return {
        "per_request": {"p50": df.cost_usd.median(), "p95": df.cost_usd.quantile(.95),
                        "p99": df.cost_usd.quantile(.99)},
        "per_successful_task": df.cost_usd.sum() / max(df.task_completed.sum(), 1),   # the real number
        "by_call_site": df.groupby("call_site").cost_usd.sum().sort_values(ascending=False).to_dict(),
        "by_tenant":    df.groupby("tenant").cost_usd.sum().nlargest(10).to_dict(),
        "cache_hit_rate": df.cached_tokens.sum() / max(df.input_tokens.sum() + df.cached_tokens.sum(), 1),
        "monthly_per_user": df.cost_usd.sum() / max(df.user_id.nunique(), 1) * (30 / 7),
    }

def cost_alerts(current, baseline):
    alerts = []
    if current["per_request"]["p95"] > baseline["per_request"]["p95"] * 1.3:
        alerts.append("p95 cost per request up >30% — check routing/escalation and context growth")
    if current["cache_hit_rate"] < baseline["cache_hit_rate"] - 0.15:
        alerts.append("cache hit rate dropped — something variable entered the prompt prefix (prompt caching of static prefixes)")
    return alerts
```

## Pros

- **Order-of-magnitude savings are routine** — caching plus tiering plus context trimming often reduces cost 10–25× versus a naive implementation.
- **Most levers are low-effort and quality-neutral**: caching, batching, and trimming to the eval-verified k cost nothing in quality.
- **Cost attribution reveals surprises**: a rewrite step on the reasoning tier, an agent path taken 5% of the time consuming 40% of spend.
- Budgets make the tail predictable ([cost & step budgets](../../../07-agents-tools-memory/apply/1-agents/cost_step_budgets.md)).

## Cons

- **Some levers trade quality**: smaller models, fewer chunks, shorter prompts — each needs eval validation before shipping ([prompt evaluation](../../../09-evaluation/apply/prompt_evaluation.md)).
- **Caching is fragile** to prompt variability, and failures are silent until the invoice arrives ([prompt caching of static prefixes](../../../05-prompt-and-context-engineering/apply/2-context-engineering/prompt_caching.md)).
- **Aggressive routing adds complexity** and can degrade quality invisibly for the routed-down cases ([routing & escalation](../../../04-llm-foundations/apply/routing_escalation.md)).
- Provider price changes and model deprecations invalidate your cost model periodically.

## When to use

From the first production deployment, in this order: instrument → cache → tier → trim → batch → budget. Do not optimise before instrumenting, and do not trade quality before exhausting the free levers.

## When NOT to use

- Don't optimise cost during the quality-establishing phase — find the quality ceiling with strong models first, then step down where evals allow ([model selection & tradeoffs](../../../04-llm-foundations/apply/model_selection.md)).
- Don't trade quality on high-stakes paths for marginal savings; route those to the reasoning tier unconditionally.

## Production at scale

**Prompt caching is the single biggest lever** for prompt-heavy products and it's quality-neutral. Structure stable-first, place breakpoints for global/tenant/conversation, monitor hit rate as an SLI, and alert when it drops — a deploy that adds a timestamp to the system prompt can double your bill overnight ([prompt caching of static prefixes](../../../05-prompt-and-context-engineering/apply/2-context-engineering/prompt_caching.md)).

**Cost per *successful task*, not per request.** A cheap system that fails half the time is expensive. This is the number to report to leadership alongside deflection and time saved ([metrics cheat sheet](../../../AI_System.md#metrics-cheat-sheet)).

**Attribution by call site is where the surprises live.** Common findings: query rewriting running on an expensive tier, an agentic path taken by 5% of traffic consuming 40% of spend, one tenant driving most cost, contextual-retrieval regeneration triggered by an unplanned re-ingest.

**Rerank-then-trim is underrated**: retrieving 50 and sending 6 costs a fraction of sending 20, usually with equal or better quality ([reranking](../../../06-rag/apply/6-retrieval-ranking/reranking.md), ["Lost in the middle"](../../../05-prompt-and-context-engineering/apply/2-context-engineering/lost_in_the_middle.md)). Sweep k on the golden set and take the plateau.

**Batch APIs for everything offline**: embeddings, contextual retrieval headers ([contextual retrieval](../../../06-rag/apply/2-chunking/contextual_retrieval.md)), VLM captions ([charts, images, diagrams](../../../06-rag/apply/1-ingestion-parsing/charts_images_vlm_captioning.md)), evals, backfills. Roughly half price and no latency requirement.

**Ingest-side levers are often the largest single line item** in document-heavy systems: change detection ([change detection](../../../06-rag/apply/1-ingestion-parsing/change_detection.md)), dedupe ([deduplication](../../../06-rag/apply/1-ingestion-parsing/deduplication.md)), and per-page routing ([per-page classification and routing](../../../06-rag/apply/1-ingestion-parsing/per_page_routing.md)) can cut ingestion cost by more than half, and they're quality-positive.

**Per-tenant cost visibility** enables pricing, quotas, and conversations with the tenant driving 60% of spend ([capacity & limits](capacity_limits.md)).

**Re-validate on model changes**: prices, cache behaviour, and token efficiency all shift. Rebuild the cost model on every provider or model change ([versioning & rollout](versioning_rollout.md)).

## Prod checklist

- [ ] Cost computed and logged per LLM call with model, call site, tenant, feature, cached tokens
- [ ] $/request (p50/p95/p99), $/successful task, $/user/month on a dashboard
- [ ] Prompt caching implemented; hit rate as an SLI with alerts
- [ ] Model tiering per call site, validated on the golden set
- [ ] Rerank-then-trim with k chosen at the quality plateau
- [ ] Batch endpoints for all offline work
- [ ] Per-request, per-user, per-tenant budgets and quotas
- [ ] Ingest levers in place: change detection, dedupe, per-page routing
- [ ] Cost model rebuilt on provider/model changes; alerts on p95 cost and cache-rate shifts

## Metrics to watch

| Metric | Why |
|---|---|
| $/request p50/p95/p99 | Tail costs are where incidents live |
| $/successful task | The honest efficiency number |
| Cache hit rate | Biggest lever's health |
| Cost by call site and by tenant | Where the money actually goes |
| Ingest cost per 1k documents | Often the largest line item |

## Related

- [Prompt caching](../../../05-prompt-and-context-engineering/apply/2-context-engineering/prompt_caching.md), [model selection & tradeoffs](../../../04-llm-foundations/apply/model_selection.md)/01.2 Model selection and routing
- [Reranking](../../../06-rag/apply/6-retrieval-ranking/reranking.md), [Budgets](../../../05-prompt-and-context-engineering/apply/2-context-engineering/per_source_truncation_budgets.md), [Agent budgets](../../../07-agents-tools-memory/apply/1-agents/cost_step_budgets.md)
- [Latency levers](latency_levers.md) (semantic cache overlaps), [Self-hosting](self_hosted_serving.md)
- [Change detection](../../../06-rag/apply/1-ingestion-parsing/change_detection.md) / [deduplication](../../../06-rag/apply/1-ingestion-parsing/deduplication.md) / [per-page classification and routing](../../../06-rag/apply/1-ingestion-parsing/per_page_routing.md) — ingest cost
- [Observability](../4-observability-reliability/observability_tracing.md) — attribution
