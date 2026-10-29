# Agents → Fan-out / fan-in

![Fan-Out Fan-In for AI Agents Explained: Why Fan-In Breaks](demo/fan-out-fan-in-demo.gif)

*([full-resolution MP4 with audio](demo/fan-out-fan-in-demo.mp4) — 40s · [watch on YouTube](https://youtu.be/8xyodd8Gt14))*

## What it is

```
                  ┌─► subagent 1 ─┐  ok
                  ├─► subagent 2 ─┤  ok           ┌── dedupe
plan (N subtasks) ├─► subagent 3 ─┤  TIMEOUT ────►├── resolve conflicts ──► ranked, complete result
                  ├─► subagent 4 ─┤  ok           ├── rank                  + explicit failure list
                  └─► subagent N ─┘  ok           └── account for every subtask
   fan-out (bounded concurrency)                     fan-in
```

**Fan-out** is the parallel dispatch: per-document review, per-section extraction, per-source research, per-candidate scoring. Concurrency is bounded by rate limits, cost, and downstream capacity — not by how many subtasks exist.

**Fan-in** is where the engineering is. It must:
- **Account for every subtask** — succeeded, failed, or skipped, with reasons.
- **Deduplicate** overlapping findings (two documents state the same fact).
- **Resolve conflicts** between subagent outputs ([conflict handling](../../../06-rag/apply/8-generation-grounding/conflict_handling.md)).
- **Rank** by whatever the goal cares about.
- **Produce deterministic output** — the same inputs must yield the same ordering, or every regeneration looks like a different answer and evals become unreproducible.

## How to implement

1. **Bounded concurrency**: a semaphore sized to your rate limits and cost tolerance (8–32 typical), not unbounded `gather`.
2. **Per-subtask timeout and budget** ([cost & step budgets](cost_step_budgets.md)); a slow subtask is cancelled, not allowed to hold the run.
3. **Never let one failure abort the fan-out**: collect results with exceptions captured as structured failures.
4. **Retry policy per subtask**: transient errors (429, 5xx, timeout) retried with backoff and jitter, bounded; deterministic failures (schema, permission) not retried ([failure handling](../../../10-production/apply/4-observability-reliability/failure_handling.md)).
5. **Sort results deterministically** before aggregation — by subtask index or a stable key, never by completion order.
6. **Fan-in step**: dedupe (content hash or embedding similarity), conflict-resolve, rank, then synthesise with an explicit failure list.
7. **Completeness assertion**: `len(ok) + len(failed) + len(skipped) == len(subtasks)`. Fail loudly if not.

## Small example

```python
import asyncio, random

async def run_with_retry(spec, task_input, caller, trace, sem, attempts=3):
    async with sem:                                                  # bounded concurrency
        for attempt in range(attempts):
            try:
                return await asyncio.wait_for(
                    dispatch_subagent_async(spec, task_input, caller, trace), timeout=spec.deadline_s)
            except asyncio.TimeoutError:
                if attempt == attempts - 1:
                    return {"status": "failed", "task": task_input, "reason": "timeout", "retriable": True}
            except RateLimited as e:
                await asyncio.sleep(min(2 ** attempt + random.random(), 20))
            except (SchemaError, PermissionDenied) as e:             # deterministic → do not retry
                return {"status": "failed", "task": task_input, "reason": type(e).__name__, "retriable": False}
            except Exception as e:
                if attempt == attempts - 1:
                    return {"status": "failed", "task": task_input, "reason": str(e)[:200], "retriable": True}
                await asyncio.sleep(2 ** attempt + random.random())

async def fan_out(plan, caller, trace, max_parallel=12):
    sem = asyncio.Semaphore(max_parallel)
    tasks = [run_with_retry(SPECS[st["contract_name"]], st["input"], caller, trace, sem) for st in plan.subtasks]
    results = await asyncio.gather(*tasks, return_exceptions=False)
    ordered = sorted(zip(plan.subtasks, results), key=lambda x: stable_key(x[0]))     # DETERMINISTIC order
    return [r for _, r in ordered]

def fan_in(goal, subtasks, results):
    assert len(results) == len(subtasks), "lost a subtask"
    ok = [r for r in results if r["status"] == "ok"]
    failed = [r for r in results if r["status"] != "ok"]

    findings = [f | {"from": r["task"], "trace_id": r["trace_id"]} for r in ok for f in r["result"].get("findings", [])]
    findings = dedupe_findings(findings)                                              # content hash + embedding sim
    conflicts = detect_finding_conflicts(findings)                                    # Conflict handling
    findings.sort(key=lambda f: (-f["severity"], f["from"]["doc_id"]))                # deterministic tie-break

    coverage = {"total": len(subtasks), "succeeded": len(ok), "failed": len(failed),
                "failures": [{"task": r["task"], "reason": r["reason"], "retriable": r.get("retriable")} for r in failed]}
    log_fanin(goal, coverage, n_findings=len(findings), n_conflicts=len(conflicts))

    return llm("fanin_synthesis", FANIN_SYS, [{"role": "user", "content":
        f"<goal>{goal}</goal><findings>{json.dumps(findings[:100])}</findings>"
        f"<conflicts>{json.dumps(conflicts)}</conflicts><coverage>{json.dumps(coverage)}</coverage>"}])

def dedupe_findings(findings, threshold=0.93):
    kept, vecs = [], []
    for f in findings:
        v = embed_documents([f["claim"]])[0]
        if any(float(v @ u) > threshold for u in vecs):
            kept[-1].setdefault("also_from", []).append(f["from"]); continue          # merge provenance
        kept.append(f); vecs.append(v)
    return kept
```

## Pros

- **Wall-clock time collapses** from N × per-task to roughly per-task (bounded by concurrency).
- **Cheap models × many subtasks** is far cheaper than one expensive model over everything.
- **Natural fault isolation** — one document's failure doesn't affect the other 39.
- Deduped, conflict-resolved output is materially better than concatenated subagent results.

## Cons

- **Rate limits bite immediately**: 40 parallel subagents × several LLM calls each will hit provider limits without a semaphore and backoff.
- **Partial failure handling is easy to get wrong** — and the failure mode (silently reporting on 37 of 40) is dangerous.
- **Non-determinism** from completion-order aggregation makes runs unreproducible and evals unstable.
- **Cost spikes**: fan-out multiplies spend; a 500-document fan-out can be a very expensive accident.
- **Fan-in synthesis has its own context limit** — 200 findings won't fit; you need ranking and capping before synthesis.

## When to use

- Independent subtasks over a known collection: per-document, per-section, per-record, per-candidate.
- Multi-source research where sources can be queried in parallel.
- Batch processing with a latency requirement.

## When NOT to use

- Sequential dependencies (subtask B needs A's output) — that's a chain, not a fan-out.
- Very large collections without a sampling or paging strategy — 10,000 subagents is a batch job, not an interactive request.
- When rate limits or cost caps make the parallelism illusory.

## Production at scale

**Concurrency sized to the real constraint.** Usually the LLM provider's rate limit, sometimes the vector store or the source API. Compute it: `max_parallel × calls_per_subtask × calls_per_minute` must sit under the quota with headroom. Add adaptive backpressure — reduce concurrency when 429s appear ([capacity & limits](../../../10-production/apply/1-serving-cost-latency/capacity_limits.md)).

**Partial failure must reach the user.** The synthesis prompt receives the coverage object and is instructed to state failures explicitly. The UI shows "37 of 40 reviewed; 3 failed (timeout)" with a retry action. Silent partial results are the most dangerous output of any fan-out system.

**Determinism has three requirements**: stable subtask ordering, stable tie-breaks in ranking, and temperature 0 in the synthesis call. Without all three, regenerating a report produces a visibly different document and users lose confidence.

**Retry the retriables, once, after the first pass.** A second pass over timed-out subtasks (with a longer deadline or a smaller scope) recovers most transient failures. Deterministic failures — schema, permission, missing document — must not be retried.

**Cap findings before synthesis**: rank and take the top N (or summarise per group) so the fan-in call fits its context budget ([per-source truncation budgets](../../../05-prompt-and-context-engineering/apply/2-context-engineering/per_source_truncation_budgets.md)). Log how many findings were dropped.

**Run-level cost ceiling** checked before fan-out and monitored during: cancel remaining subtasks if the run exceeds budget, and report what was completed ([cost & step budgets](cost_step_budgets.md)).

**Idempotency across the fan-out**: each subtask carries an idempotency key so a resumed run skips completed subtasks rather than redoing them ([state & durability](state_durability.md)).

**Fan-in is where conflicts surface.** Two subagents reading two documents will disagree; apply the same conflict rules as generation ([conflict handling](../../../06-rag/apply/8-generation-grounding/conflict_handling.md)) and route unresolved ones to a data-quality queue.

**Streaming progress** for long fan-outs: "reviewed 22 of 40" keeps users engaged and makes a 3-minute run acceptable.

## Prod checklist

- [ ] Bounded concurrency sized to rate limits, with adaptive backpressure on 429s
- [ ] Per-subtask timeout, budget, and bounded retry; retriable vs deterministic failures distinguished
- [ ] No exception aborts the fan-out; failures collected as structured results
- [ ] Deterministic ordering: stable subtask sort, stable tie-breaks, temperature 0 synthesis
- [ ] Completeness assertion: every subtask accounted for
- [ ] Coverage object passed to synthesis and surfaced in the UI with a retry action
- [ ] Findings deduped with merged provenance; conflicts detected and surfaced
- [ ] Findings capped before synthesis; drops logged
- [ ] Run-level cost ceiling with cancellation and partial reporting
- [ ] Idempotency keys so resumed runs skip completed subtasks
- [ ] Progress streaming for long runs

## Metrics to watch

| Metric | Why |
|---|---|
| Subtask failure rate by reason; retry recovery rate | Reliability |
| Coverage (completed / dispatched) per run | Silent partial results |
| Wall-clock vs sum of subtask times | Parallelism effectiveness |
| 429 rate and concurrency throttling events | Rate-limit health |
| Run cost distribution (p95/p99) | Cost blowups |
| Output determinism (same input → same output hash) | Reproducibility |

## Related

- [Task decomposition](task_decomposition.md), [Orchestrator/subagent](orchestrator_subagent.md)
- [State & durability](state_durability.md) — idempotency and resume
- [Budgets](cost_step_budgets.md) — run-level ceiling
- [Failure handling](../../../10-production/apply/4-observability-reliability/failure_handling.md) — backoff, breakers
- [Conflict handling](../../../06-rag/apply/8-generation-grounding/conflict_handling.md) — in fan-in
