# Agents → Single agent loop

## Video takeaway

"The agent loop itself is twenty lines: ask the model, run the tool it picked, hand back the result, repeat. Everything that makes it production-ready sits around it. Decide explicitly how it ends — a final-answer tool with a schema, not just 'the model stopped calling things'. Cap steps, tokens, seconds, and dollars in code, because the prompt can only ask. Put a timeout on every call. And when a tool fails, don't throw — send the error text back to the model. 'File not found, check the path' is something it recovers from; a stack trace in your logs is something it never sees."

## What it is

The agent loop is twenty lines of code. Everything that makes it production-worthy is around it:

```
                 ┌──────────────────────────────────────────┐
messages ──► LLM ─┤ stop_reason == "tool_use"? ──────────────┤
   ▲             └─ no → final answer → validate → return   │
   │                                                         │
   │             yes ──► validate args ──► authorise ──► execute (timeout, idempotent)
   │                                              │
   └──────────── append tool_result (or error text) ◄────────┘
                       │
                 budget check: steps, tokens, wall-clock, dollars → exhausted? force finalize
```

Four design points, from the source file:

1. **Termination condition** — how does the agent know it's done? An explicit final-answer tool, a structured `{done: true}`, or a stop reason. Never "the model stopped calling tools" as the only signal.
2. **Max iterations** — a hard cap enforced by the runtime.
3. **Per-step timeout** — every tool call and every LLM call has one.
4. **Error handling that feeds the error back** — a tool that throws should produce a `tool_result` containing the error message, not an exception that kills the run. Models recover from "file not found: check the path" remarkably well; they cannot recover from a stack trace in your logs.

## How to implement

1. **Represent state explicitly**: `messages`, `budget`, `trace_id`, `caller`, `step`. Persist it if runs can outlive a request ([state & durability](state_durability.md)).
2. **Loop with a hard cap** and check the budget *before* each LLM call and each tool execution ([cost & step budgets](cost_step_budgets.md)).
3. **Validate tool arguments** against the schema before executing ([structured outputs & tool use](../../../05-prompt-and-context-engineering/apply/1-prompt-engineering/structured_outputs_tool_use.md)); a validation failure is a `tool_result` with the error, not a crash.
4. **Authorise separately** — the policy check uses the caller's permissions, never the model's assertion ([excessive agency](../../../10-production/apply/5-guardrails/excessive_agency.md)).
5. **Execute with a timeout** and an idempotency key ([state & durability](state_durability.md)); catch everything and convert to a structured error result.
6. **Cap tool-result size** ([per-source truncation budgets](../../../05-prompt-and-context-engineering/apply/2-context-engineering/per_source_truncation_budgets.md)) and store the full payload in the trace with a handle the agent can re-read.
7. **Force finalisation** on budget exhaustion: one last LLM call instructed to answer from what it has, or an explicit failure response. Never return nothing.
8. **Trace every step**: LLM call, tool call, arguments, result IDs, timings, tokens ([observability & tracing](../../../10-production/apply/4-observability-reliability/observability_tracing.md)).

## Small example

```python
import anthropic, json, time

class BudgetExceeded(Exception): ...

class Budget:
    def __init__(self, max_steps=10, max_tokens=120_000, deadline_s=60, max_usd=0.50):
        self.max_steps, self.max_tokens, self.deadline, self.max_usd = max_steps, max_tokens, time.time() + deadline_s, max_usd
        self.steps = self.tokens = 0; self.usd = 0.0; self.reason = None
    def check(self):
        if self.steps >= self.max_steps:      self.reason = "max_steps"
        elif self.tokens >= self.max_tokens:  self.reason = "max_tokens"
        elif time.time() > self.deadline:     self.reason = "deadline"
        elif self.usd >= self.max_usd:        self.reason = "max_cost"
        if self.reason: raise BudgetExceeded(self.reason)
    def charge(self, usage, model):
        self.tokens += usage.input_tokens + usage.output_tokens
        self.usd += price(model, usage)

def run_agent(question, caller, tools, tool_impls, system, model=TIER["balanced"], budget=None):
    budget = budget or Budget()
    trace = new_trace("agent", question)
    messages = [{"role": "user", "content": question}]

    while True:
        try:
            budget.check()
        except BudgetExceeded as e:
            trace.log("budget_exhausted", str(e))
            return finalize_forced(messages, system, model, reason=str(e), trace=trace)

        budget.steps += 1
        r = client.messages.create(model=model, system=system, tools=tools, messages=messages,
                                   max_tokens=2000, timeout=30.0)
        budget.charge(r.usage, model)
        trace.log_llm(budget.steps, r.usage, r.stop_reason)

        if r.stop_reason != "tool_use":
            return finalize(r, messages, trace, budget)              # explicit termination

        messages.append({"role": "assistant", "content": r.content})
        results = []
        for block in [b for b in r.content if b.type == "tool_use"]:
            results.append({"type": "tool_result", "tool_use_id": block.id,
                            "content": execute_tool(block, caller, tool_impls, trace, budget)})
        messages.append({"role": "user", "content": results})

def execute_tool(block, caller, tool_impls, trace, budget, timeout_s=20):
    name, args = block.name, block.input
    try:
        validated = SCHEMAS[name].model_validate(args)               # Structured outputs & tool use
    except Exception as e:
        trace.log_tool(name, args, error=f"schema: {e}")
        return json.dumps({"status": "error", "error": f"Invalid arguments: {e}. Fix and call again."})
    if not authorise(caller, name, validated):                        # Excessive agency — policy, not the model
        trace.log_tool(name, args, error="unauthorised")
        return json.dumps({"status": "error", "error": f"Not permitted to call {name} with these arguments."})
    key = idempotency_key(block.id, name, validated)                  # State & durability
    try:
        out = with_timeout(lambda: tool_impls[name](validated, caller, key), timeout_s)
        payload = truncate_result(out, max_tokens=3000, trace=trace)  # Per-source truncation budgets; full result in the trace store
        trace.log_tool(name, args, ok=True)
        return json.dumps({"status": "ok", **payload})
    except TimeoutError:
        trace.log_tool(name, args, error="timeout")
        return json.dumps({"status": "error", "error": f"{name} timed out after {timeout_s}s. Try a narrower query."})
    except Exception as e:                                            # error text goes BACK to the model
        trace.log_tool(name, args, error=str(e))
        return json.dumps({"status": "error", "error": f"{name} failed: {str(e)[:300]}"})

def finalize_forced(messages, system, model, reason, trace):
    messages.append({"role": "user", "content":
        f"You have reached the {reason} limit. Give your best answer using only what you have gathered, "
        "and state clearly what you could not complete."})
    r = client.messages.create(model=model, system=system, messages=messages, max_tokens=1200)
    return {"answer": text_of(r), "complete": False, "stopped_by": reason, "trace_id": trace.id}
```

## Pros

- **Simple and general** — one loop serves research, extraction, workflow automation.
- **Error feedback makes agents self-correcting**: wrong path, bad filter, malformed query — the model reads the error and fixes it, which is the single biggest reliability gain over "throw on failure".
- **Budgets make cost bounded** and the worst case knowable.
- **Traceable**: a linear sequence of steps is easy to replay and debug.

## Cons

- **Cost and latency variance** is inherent — the same task can take 2 or 12 steps.
- **Context grows monotonically**; without compaction ([compaction / summarisation of long history](../../../05-prompt-and-context-engineering/apply/2-context-engineering/compaction_summarization.md)) and result caps ([per-source truncation budgets](../../../05-prompt-and-context-engineering/apply/2-context-engineering/per_source_truncation_budgets.md)), later steps are slow, expensive, and distracted.
- **Loops and thrashing**: the model retries the same failing call with the same arguments. Needs explicit detection.
- **Silent partial success**: an agent that "finished" without completing the task, because nothing checked.
- Error text fed back can itself be an injection vector if it contains untrusted content ([prompt-injection defence](../../../10-production/apply/5-guardrails/prompt_injection_defense.md)).

## When to use

- Tasks requiring tool use where the sequence isn't fixed: investigation, multi-system lookups, iterative data gathering.
- As the substrate for orchestrator/subagent designs ([orchestrator / subagent](orchestrator_subagent.md)).

## When NOT to use

- Anything you can express as a fixed workflow with LLM steps inside it — which is most production tasks ([planning styles](planning_styles.md)). A scripted pipeline is cheaper, faster, and testable.
- Single-tool, single-call operations — just call the tool.
- Latency-critical paths.

## Production at scale

**The termination condition deserves real design.** Prefer an explicit `submit_answer` tool with a schema over "the model stopped calling tools" — it gives you validated output, a clear stop, and a place to enforce completeness. Log runs that ended without it as anomalies.

**Loop detection**: hash `(tool_name, normalised_args)`; if the same call repeats 3 times, inject a message naming the repetition and forbidding it, and count it as a step. Repeated identical failing calls are the most common runaway pattern.

**No-progress detection**: track whether each step added new information (new chunk IDs, new record IDs). N steps without new information → force finalisation.

**Budgets belong in the runtime.** The prompt can *request* brevity; only code can *enforce* it. Steps, tokens, wall-clock, and dollars, all four, checked before each step ([cost & step budgets](cost_step_budgets.md)).

**Prompt caching is essential here** — the system prompt and tool definitions are re-sent every step. Keep them byte-identical for the whole run and never mutate the tool list mid-run ([prompt caching of static prefixes](../../../05-prompt-and-context-engineering/apply/2-context-engineering/prompt_caching.md)).

**Context management inside the loop**: cap each tool result, compact older observations after N steps ([compaction / summarisation of long history](../../../05-prompt-and-context-engineering/apply/2-context-engineering/compaction_summarization.md)), and keep the current instruction and the latest result near the end (["Lost in the middle"](../../../05-prompt-and-context-engineering/apply/2-context-engineering/lost_in_the_middle.md)).

**Errors returned to the model are untrusted text.** An error message containing content from an external system can carry an injection. Wrap error text in a delimited block and never let it reach a privileged action path unchecked ([delimiters: instructions vs data](../../../05-prompt-and-context-engineering/apply/1-prompt-engineering/delimiters_data_vs_instructions.md), [prompt-injection defence](../../../10-production/apply/5-guardrails/prompt_injection_defense.md)).

**Idempotency for every side-effecting tool** ([state & durability](state_durability.md)): a retried step must not double-send, double-charge, or double-create.

**Trace-first debugging.** The trace (step, tool, args, result IDs, tokens, timing) is the only way to answer "why did it do that". Store it for every run, with full tool payloads by handle, and make it searchable ([observability & tracing](../../../10-production/apply/4-observability-reliability/observability_tracing.md)).

**Evaluate the trajectory, not just the answer** ([agent evals](../../../09-evaluation/apply/agent_evals.md)): right tools, sensible order, reasonable cost. Report steps and cost alongside accuracy.

## Prod checklist

- [ ] Explicit termination via a schema-validated final-answer tool; runs ending otherwise are flagged
- [ ] Hard caps on steps, tokens, wall-clock, dollars, checked before each step
- [ ] Per-tool-call and per-LLM-call timeouts
- [ ] Tool errors converted to structured `tool_result` text fed back to the model; never exceptions
- [ ] Argument validation and separate authorisation before every execution
- [ ] Loop detection (repeated identical calls) and no-progress detection
- [ ] Tool results token-capped with full payloads in the trace store
- [ ] Tools and system prompt byte-identical across steps for caching
- [ ] Forced finalisation on budget exhaustion, never an empty return
- [ ] Full per-step tracing; trajectory evals with cost and step count

## Metrics to watch

| Metric | Why |
|---|---|
| Steps per run (p50/p95/p99) | Cost and loop detection |
| Budget-exhaustion rate by reason | Bound tuning |
| Tool error rate by tool; recovery rate after an error | Error-feedback effectiveness |
| Runs ending without explicit termination | Silent partial success |
| $ per run distribution | Unit economics |

## Related

- [Tool design](tool_design.md), [State & durability](state_durability.md), [Budgets](cost_step_budgets.md)
- [Compaction](../../../05-prompt-and-context-engineering/apply/2-context-engineering/compaction_summarization.md), [Result caps](../../../05-prompt-and-context-engineering/apply/2-context-engineering/per_source_truncation_budgets.md), [Caching](../../../05-prompt-and-context-engineering/apply/2-context-engineering/prompt_caching.md) — inside the loop
- [Structured outputs](../../../05-prompt-and-context-engineering/apply/1-prompt-engineering/structured_outputs_tool_use.md) — argument validation
- [Excessive agency](../../../10-production/apply/5-guardrails/excessive_agency.md) — authorisation
- [Agent evals](../../../09-evaluation/apply/agent_evals.md)
