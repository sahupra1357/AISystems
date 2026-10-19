# Advanced RAG → Agentic RAG (retrieval as a tool)

## Video takeaway

"Agentic RAG turns retrieval from a fixed step into a tool the model calls as often as it likes, with queries and filters it chooses. That's genuinely more capable for research questions where you can't know the path in advance. It's also how the same question costs two cents one time and two dollars the next. The rules that make it survivable: budgets enforced in code — tool calls, tokens, seconds, dollars — not asked for in the prompt; security filters applied inside the tool, never passed by the model; and a scripted pipeline still handling the ninety percent of questions that never needed an agent."

## What it is

Classic RAG is a pipeline: rewrite → retrieve → rerank → generate. The retrieval strategy is decided by your code. Agentic RAG inverts control: the model decides *whether* to retrieve, *what* to retrieve, *from where*, and *when it has enough*:

```
                      ┌──── search_documents(query, filters) ────┐
user question ──► agent ├──── search_tickets(query) ─────────────┤──► observations ──► agent ──► ... ──► answer
                      ├──── query_warehouse(sql) ───────────────┤
                      └──── read_document(doc_id, section) ─────┘
```

The spectrum runs from bounded recursion ([recursive / agentic retrieval](../6-retrieval-ranking/recursive_agentic_retrieval.md) — one tool, capped hops) to a full agent with several retrieval backends, filters it composes itself, and a free-form plan. The further right you go, the more capability and the more variance in cost, latency, and correctness.

The source file's own guidance from [agents](../../../AI_System.md#agents) applies directly: *most production agents are far more scripted than the literature suggests — prefer a fixed workflow with LLM steps inside it, and reach for a free-form agent only when the path genuinely can't be enumerated.*

## How to implement

1. **Start from the scripted pipeline** and identify the specific query classes it fails on. If you can't name them, you don't need agentic RAG.
2. **Design the tools** ([tool design](../../../07-agents-tools-memory/apply/1-agents/tool_design.md)): few, non-overlapping, well-described, with `when to use` and `when not to use` in the description. `search_documents(query, doc_type?, date_from?)` beats five near-identical search tools.
3. **Return structured observations**: passages with IDs, scores, and a truncation marker ([per-source truncation budgets](../../../05-prompt-and-context-engineering/apply/2-context-engineering/per_source_truncation_budgets.md)) — not raw dumps. Cap tokens per tool result.
4. **Enforce budgets in the runtime**: max tool calls, max tokens, wall-clock, dollars ([cost & step budgets](../../../07-agents-tools-memory/apply/1-agents/cost_step_budgets.md)). The prompt asks; the runtime enforces.
5. **Filters come from the caller, not the model** for anything security-related: the tool implementation applies tenant/ACL/current-version filters internally ([tenant / ACL capture at ingest](../1-ingestion-parsing/tenant_acl_capture_at_ingest.md), [authentication & authorization](../../../10-production/apply/6-security-privacy-governance/authn_authz_permission_filtered_retrieval.md)). The model may pass *intent* filters (date, doc type) only.
6. **Termination**: an explicit final-answer tool or a structured `{done: true}` decision, plus hard caps and a "no progress" detector.
7. **Trace everything**: every tool call, arguments, result IDs, scores ([observability & tracing](../../../10-production/apply/4-observability-reliability/observability_tracing.md)). This is the only way to debug why the agent looked in the wrong place.

## Small example

```python
SEARCH_TOOL = {
    "name": "search_documents",
    "description": ("Search the company's internal document corpus (policies, contracts, wikis) by meaning and keywords. "
                    "Use when the question needs narrative text from internal documents. "
                    "Do NOT use for counts/aggregations (use query_warehouse) or for live record status (use get_record). "
                    "Returns up to 6 passages with ids and relevance scores."),
    "input_schema": {"type": "object", "properties": {
        "query": {"type": "string", "description": "Focused search query, not the whole user question"},
        "doc_type": {"type": "string", "enum": ["contract", "policy", "report", "any"], "default": "any"},
        "date_from": {"type": "string", "description": "ISO date; only documents effective on/after"},
    }, "required": ["query"]},
}

def search_documents_impl(args, caller, budget):
    intent_filters, dropped = validate_filters({k: {"eq": v} for k, v in args.items() if k in ("doc_type",)})
    if args.get("date_from"): intent_filters["effective_date"] = {"gte": args["date_from"]}
    flt = merge_filters(build_acl_filter(caller), intent_filters)     # security filters injected here, never from the model
    hits = rerank_hosted(args["query"], hybrid_retrieve_with_filter(args["query"], flt), top_n=6)
    budget.charge_tool_result(hits)
    return {"passages": [{"id": h["chunk_id"], "score": round(h["rerank_score"], 3),
                          "text": truncate(h["text"], 900), "source": h["title"], "page": h.get("page")} for h in hits],
            "n_found": len(hits), "filters_applied": intent_filters, "filters_dropped": dropped}

def agentic_rag(question, caller, max_tool_calls=6, deadline_s=20, max_cost_usd=0.25):
    budget = Budget(max_tool_calls, deadline_s, max_cost_usd)
    messages = [{"role": "user", "content": question}]
    while budget.ok():
        r = client.messages.create(model=TIER["balanced"], system=AGENT_SYS, tools=TOOLS,
                                   messages=messages, max_tokens=1500)
        budget.charge_llm(r.usage)
        if r.stop_reason != "tool_use":
            return finalize(r, messages, budget)
        for block in [b for b in r.content if b.type == "tool_use"]:
            result = TOOL_IMPLS[block.name](block.input, caller, budget)
            trace_tool_call(block.name, block.input, result)
            messages += [{"role": "assistant", "content": r.content},
                         {"role": "user", "content": [{"type": "tool_result", "tool_use_id": block.id,
                                                       "content": json.dumps(result)}]}]
    return forced_answer_or_abstain(messages, budget, reason=budget.exhausted_reason())
```

## Pros

- **Handles genuinely open-ended research** — unknown vocabulary, cross-references, multiple backends, questions whose shape isn't known in advance.
- **Backend selection per need** without a hand-written router for every combination.
- **Self-correcting**: a fruitless search can be followed by a different angle.
- The tool-call trail is a legible research narrative for the user.

## Cons

- **Cost and latency variance is the defining problem** — the same question can cost $0.02 or $2.00 depending on the path taken. Budgeting is mandatory, not optional.
- **Harder to evaluate**: trajectory quality matters, not just the final answer ([agent evals](../../../09-evaluation/apply/agent_evals.md)).
- **More failure modes**: wrong tool ([failure modes: poisoning, distraction, confusion, clash](../../../05-prompt-and-context-engineering/apply/2-context-engineering/context_failure_modes.md) confusion), looping, drifting off-topic, over-searching a question that needed one lookup.
- **Injection surface widens**: retrieved content influences the *next tool call*, not just the answer ([prompt-injection defence](../../../10-production/apply/5-guardrails/prompt_injection_defense.md)).
- **Regression risk**: prompt or model changes shift trajectories in ways unit tests don't catch.

## When to use

- Research and investigation workflows where the path genuinely can't be enumerated: due diligence, incident investigation, competitive analysis, complex compliance questions spanning several systems.
- Multi-backend questions where routing rules would combinatorially explode.
- Async/deep-research product modes with a latency budget measured in tens of seconds.

## When NOT to use

- **Standard document Q&A.** A scripted hybrid + rerank pipeline is faster, cheaper, more predictable, and usually just as accurate. This is the default and most systems should stay here.
- Interactive chat with tight latency budgets.
- Before you have retrieval evals — you won't be able to tell whether the agent helped.
- When the failure you're trying to fix is actually parsing, chunking, or reranking ([common failure modes](../../../AI_System.md#common-failure-modes-and-the-fix) triage first).

## Production at scale

**Budgets are the product decision.** Set max tool calls, tokens, wall-clock, and dollars per run, enforced by the runtime and surfaced in the trace. Publish per-run cost distributions (p50/p95/p99) — the p99 is what determines whether this feature is viable.

**Tool-result token caps** are the biggest cost lever: an agent that gets 6 passages of 900 tokens per call, six times, has 32k tokens of context by the end. Cap per result, compact older observations ([compaction / summarisation of long history](../../../05-prompt-and-context-engineering/apply/2-context-engineering/compaction_summarization.md)), and store full results in the trace store with handles the agent can re-read.

**Security filters live in the tool implementation.** The model never supplies `tenant_id` or ACL parameters. Passing the caller through to every tool implementation and enforcing there is the only safe design ([authentication & authorization](../../../10-production/apply/6-security-privacy-governance/authn_authz_permission_filtered_retrieval.md), [excessive agency](../../../10-production/apply/5-guardrails/excessive_agency.md)).

**Prompt caching matters enormously here** — the system prompt and tool definitions are re-sent on every step. Keep them byte-identical across the run ([prompt caching of static prefixes](../../../05-prompt-and-context-engineering/apply/2-context-engineering/prompt_caching.md)), and never mutate the tool list mid-run.

**Trajectory evals** ([agent evals](../../../09-evaluation/apply/agent_evals.md)): did it pick the right tools, in a sensible order, at reasonable cost? Report cost and step count alongside accuracy — a 2% accuracy gain for 4× the tool calls is a loss.

**Fallback to scripted.** On budget exhaustion or repeated tool failures, fall back to the deterministic pipeline and answer from what was gathered, or abstain. Never return an empty result because the agent ran out of steps.

**Progressive disclosure in UX.** Stream the tool calls as human-readable steps. Users tolerate 20 seconds when they can see "searching contracts for termination clauses…"; they abandon a blank spinner at 8.

**Start scripted, graduate selectively.** A production pattern that works: scripted pipeline handles 90% of traffic; a complexity classifier ([routing](../5-query-understanding/query_routing.md)) routes the remaining 10% to the agentic path. You get agent capability at scripted-pipeline average cost.

## Prod checklist

- [ ] Scripted pipeline exists and its specific failure classes are named
- [ ] Few, non-overlapping tools with "when not to use" in descriptions
- [ ] Security filters injected inside tool implementations from the caller identity
- [ ] Runtime-enforced budgets: tool calls, tokens, wall-clock, dollars; exhaustion is a logged outcome
- [ ] Tool results structured, token-capped, with full payloads in the trace store
- [ ] Tools and system prompt byte-identical across steps for prompt caching
- [ ] Explicit termination + no-progress detector + fallback to scripted answer or abstention
- [ ] Full trajectory tracing; trajectory evals with cost and step count
- [ ] Complexity routing so only hard queries take the agentic path

## Metrics to watch

| Metric | Why |
|---|---|
| Cost and steps per run (p50/p95/p99) | Viability |
| Task success vs scripted baseline | Is it earning its cost |
| Tool-selection accuracy | Confusion ([failure modes: poisoning, distraction, confusion, clash](../../../05-prompt-and-context-engineering/apply/2-context-engineering/context_failure_modes.md)) |
| Budget-exhaustion rate and reason | Bound tuning |
| Agentic-path routing rate | Cost containment |

## Related

- [Recursive retrieval](../6-retrieval-ranking/recursive_agentic_retrieval.md) — the bounded ancestor
- [Agent loop](../../../07-agents-tools-memory/apply/1-agents/single_agent_loop.md), [Tool design](../../../07-agents-tools-memory/apply/1-agents/tool_design.md), [Budgets](../../../07-agents-tools-memory/apply/1-agents/cost_step_budgets.md)
- [Routing](../5-query-understanding/query_routing.md) — decides who takes this path
- [Agent evals](../../../09-evaluation/apply/agent_evals.md), [Injection](../../../10-production/apply/5-guardrails/prompt_injection_defense.md)
