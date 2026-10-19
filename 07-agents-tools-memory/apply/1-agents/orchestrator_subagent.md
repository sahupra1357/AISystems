# Agents → Orchestrator / subagent

## Video takeaway

"One agent reviewing forty contracts ends up with a window full of forty documents and can't reason about any of them. Split it: an orchestrator that holds the plan and the results, and forty subagents that each see one contract, one tool, and nothing else. Each returns a small structured result — not its transcript, which goes to the trace store. The orchestrator's window grows by a few hundred tokens per contract instead of ten thousand, and the work runs on a cheap model. Enforce the output schema with a length limit, or your subagents will hand back essays and you've rebuilt the problem."

## What it is

A single agent doing everything ends up with a window containing thirty tool results, three abandoned approaches, and the actual task buried at the top. The orchestrator/subagent split separates **coordination** from **execution**:

| | Orchestrator | Subagent |
|---|---|---|
| Holds | Goal, plan, subtask results (small structured summaries) | One subtask, its inputs, its observations |
| Window | Stays small — grows by result size, not work size | Fresh per subtask; discarded after |
| Tools | Dispatch, aggregate, maybe approve | A narrow allow-list for its job |
| Model | Reasoning tier (planning and synthesis are hard) | Fast tier (narrow tasks are easy) |
| Lifetime | The whole run | One subtask |

```
orchestrator: [goal][plan][result A 300tok][result B 250tok][result C 280tok] → synthesis
       │              │              │
   subagent A     subagent B     subagent C      each: own window, own tools, own budget,
   (20k tokens    (30k tokens    (15k tokens     transcript → trace store, not back to parent
    of work)       of work)       of work)
```

This is the architectural expression of context isolation ([context isolation](../../../05-prompt-and-context-engineering/apply/2-context-engineering/context_isolation.md)) plus task decomposition ([task decomposition](task_decomposition.md)). The economics are compelling: the orchestrator sees ~1k tokens of results instead of 65k tokens of work, and the work runs on a model costing a fraction as much.

## How to implement

1. **Contracts first** ([task decomposition](task_decomposition.md)): each subagent type has a typed input, typed output, success criteria, tool allow-list, model tier, and budget.
2. **Orchestrator plans**, validates the plan, and estimates cost before dispatching.
3. **Dispatch** each subtask with a *fresh* message list — the subagent never receives the orchestrator's history. Large inputs are passed by handle (doc_id), not by value.
4. **Subagent runs its own loop** ([single agent loop](single_agent_loop.md)) with its own budget and compaction, and returns a schema-validated result.
5. **Store the subagent transcript** in the trace store keyed by a child trace ID; return only the structured result to the orchestrator.
6. **Failures are structured results** (`status: failed, reason`), never exceptions propagating into the orchestrator's context.
7. **Orchestrator aggregates**: dedupe, resolve conflicts ([conflict handling](../../../06-rag/apply/8-generation-grounding/conflict_handling.md)), rank, and synthesise — accounting for every dispatched subtask.

## Small example

```python
from pydantic import BaseModel

class SubagentSpec(BaseModel):
    name: str; system: str; tools: list[str]; tier: str
    output_model: str; max_steps: int = 10; max_tokens: int = 25_000; deadline_s: int = 45

SPECS = {
    "review_contract": SubagentSpec(
        name="review_contract", tier="fast", tools=["search_doc"], output_model="ContractReview",
        system=("You review ONE contract for auto-renewal terms. Use search_doc to find relevant clauses. "
                "Return the structured result. If a field is not present in the contract, set it null and say why. "
                "Do not speculate.")),
}

def dispatch_subagent(spec: SubagentSpec, task_input: dict, caller, parent_trace) -> dict:
    child = parent_trace.child(spec.name)
    budget = Budget(max_steps=spec.max_steps, max_tokens=spec.max_tokens, deadline_s=spec.deadline_s)
    messages = [{"role": "user", "content": f"<task>{json.dumps(task_input)}</task>"}]   # fresh window
    tools = [TOOL_DEFS[t] for t in spec.tools]                                            # narrow allow-list
    try:
        result = run_agent_loop(messages, spec.system, tools, TIER[spec.tier], caller, budget, child)
        out = MODELS[spec.output_model].model_validate(result["output"])
        store_transcript(child, result["messages"])                    # transcript stays OUT of the parent
        return {"status": "ok", "task": task_input, "result": out.model_dump(),
                "trace_id": child.id, "tokens": budget.tokens, "usd": round(budget.usd, 4)}
    except Exception as e:
        store_transcript(child, locals().get("result", {}).get("messages", []))
        return {"status": "failed", "task": task_input, "reason": str(e)[:300],
                "trace_id": child.id, "tokens": budget.tokens, "usd": round(budget.usd, 4)}

ORCH_SYS = """You coordinate a review. You receive structured results from subagents, one per contract.
Synthesise a ranked risk summary. Every contract must appear either in the ranked list or in the failures list.
Cite subagent trace ids for each finding. Do not invent findings not present in the results."""

def orchestrate(goal, doc_ids, caller, max_parallel=8, run_budget_usd=5.0):
    trace = new_trace("orchestrator", goal)
    plan = plan_contract_review(doc_ids, checklist_version="v3")                 # static plan (task decomposition)
    est = len(plan.subtasks) * estimate_cost(SPECS["review_contract"])
    if est > run_budget_usd:
        return request_approval(plan, est)                                       # Human-in-the-loop

    results = parallel_map(lambda st: dispatch_subagent(SPECS[st["contract_name"]], st["input"], caller, trace),
                           plan.subtasks, max_workers=max_parallel)
    ok = [r for r in results if r["status"] == "ok"]
    failed = [r for r in results if r["status"] != "ok"]
    trace.log("fanin", n_ok=len(ok), n_failed=len(failed), usd=sum(r["usd"] for r in results))

    compact = json.dumps({"results": [r["result"] | {"trace_id": r["trace_id"]} for r in ok],
                          "failures": [{"task": r["task"], "reason": r["reason"]} for r in failed]})
    return llm("orchestrator_synthesis", ORCH_SYS,
               [{"role": "user", "content": f"<goal>{goal}</goal><results>{compact}</results>"}],
               max_tokens=2500)
```

## Pros

- **Orchestrator reasoning stays sharp** because its window stays small — the main quality argument.
- **Cost**: narrow subtasks run on fast-tier models; typical savings of 60–90% versus one reasoning-tier agent doing everything.
- **Parallelism** is natural for independent subtasks ([fan-out / fan-in](fan_out_fan_in.md)).
- **Least privilege per subagent** — a document-reviewing subagent has no ability to send email ([excessive agency](../../../10-production/apply/5-guardrails/excessive_agency.md)).
- **Debuggable**: per-subagent traces with clean inputs and outputs.
- Prompt caching works well: each subagent type has a stable system prompt ([prompt caching of static prefixes](../../../05-prompt-and-context-engineering/apply/2-context-engineering/prompt_caching.md)).

## Cons

- **Information loss at the boundary**: the subagent discards nuance the orchestrator might have wanted; the orchestrator can't ask follow-ups without a new dispatch.
- **Coordination code** — dispatch, aggregation, partial failure, ordering — is real engineering.
- **Duplicate work**: N subagents independently re-fetch or re-derive the same context.
- **Aggregation is often the hard part**, especially conflict resolution across subagent findings.
- More moving parts to trace, budget, and evaluate.

## When to use

- Fan-out over collections (documents, sections, records) — the canonical case.
- Multi-phase workflows where phases need different tools or permissions.
- Long-running work where a single agent's context would exceed the window.
- When you want cheap models doing most of the work.

## When NOT to use

- Short, tightly coupled tasks where each step depends on the full prior context — the handoff costs more than it saves.
- When the subtask needs most of the orchestrator's context anyway (you'd copy the window; no isolation gained).
- Latency-critical single-hop interactions.
- Before you have contracts — without typed outputs, subagents return prose and you've rebuilt the polluted window.

## Production at scale

**Result size is the whole benefit.** If subagents return 5k-token narratives, the orchestrator's window fills anyway. Enforce output schemas with length limits (a `summary` field capped at ~120 words plus structured fields), and validate on return.

**Shared read-through caches** for retrieval, parsed documents, and tool results keyed by content hash prevent N subagents paying N times for the same fetch. Measure the duplicate-fetch rate.

**Deterministic ordering** of aggregated results (sort by doc_id or subtask index) so runs are reproducible and diffable — essential for evals and for user trust when a report is regenerated.

**Partial failure is normal, not exceptional.** Design the aggregation to produce a useful answer from 37 of 40 successes *and* to state the 3 failures explicitly ([fan-out / fan-in](fan_out_fan_in.md)). Never let one failed subagent fail the run.

**Budgets at two levels**: per subagent (enforced in its loop) and per run (checked before dispatch and monitored during). A run-level budget prevents 40 well-behaved subagents from collectively costing $50.

**Least privilege per subagent type**: a scoped token and a tool allow-list per spec. This is where the agent security story is actually implemented — an injected instruction inside a document reaches a subagent that has only `search_doc` ([prompt-injection defence](../../../10-production/apply/5-guardrails/prompt_injection_defense.md), [excessive agency](../../../10-production/apply/5-guardrails/excessive_agency.md)).

**Nesting depth**: one level of subagents is the norm; two is occasionally justified; three means you've built a distributed system without meaning to. Cap it in code.

**Tracing hierarchy**: parent trace with child spans, cost rolled up per subagent type. "Which subagent type burns the most tokens" is the question that drives optimisation.

**Evaluate independently**: each subagent type gets its own eval set (its contract makes this easy), plus an end-to-end orchestration eval ([agent evals](../../../09-evaluation/apply/agent_evals.md), [component vs end-to-end](../../../09-evaluation/apply/component_vs_end_to_end.md)). Most quality problems localise to one subagent type.

## Prod checklist

- [ ] Typed contracts per subagent type with output size limits; validated on return
- [ ] Fresh window per subagent; inputs by handle; orchestrator history never copied
- [ ] Narrow tool allow-list and scoped credentials per subagent type
- [ ] Transcripts to the trace store; only structured results to the orchestrator
- [ ] Failures returned as structured results; run continues on partial failure
- [ ] Budgets per subagent and per run; run cost estimated before dispatch
- [ ] Shared caches for repeated fetches; duplicate-fetch rate monitored
- [ ] Deterministic result ordering; aggregation accounts for every subtask
- [ ] Nesting depth capped; parent/child tracing with cost roll-up
- [ ] Per-subagent-type evals plus end-to-end orchestration eval

## Metrics to watch

| Metric | Why |
|---|---|
| Orchestrator window tokens per step | Isolation working |
| Cost and tokens by subagent type | Where to optimise |
| Subagent failure rate by type and reason | Contract/tool problems |
| Duplicate-fetch rate | Cache effectiveness |
| Run cost: estimated vs actual | Budget accuracy |

## Related

- [Task decomposition](task_decomposition.md) — the contracts
- [Fan-out/fan-in](fan_out_fan_in.md) — parallel dispatch and aggregation
- [Context isolation](../../../05-prompt-and-context-engineering/apply/2-context-engineering/context_isolation.md) — the rationale
- [Budgets](cost_step_budgets.md), [Excessive agency](../../../10-production/apply/5-guardrails/excessive_agency.md)
- [Agent evals](../../../09-evaluation/apply/agent_evals.md)
