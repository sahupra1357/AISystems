# Agents → Task decomposition ("define down")

## Video takeaway

"Before an orchestrator dispatches anything, it has to turn a vague goal into subtasks precise enough that a cheap model can execute them — each with an input schema, an output schema, and a success criterion you can check without redoing the work. And prefer writing that decomposition in code: per document, per section, per checklist item. LLM planning is the weak link. Validate the plan before you spend money on it, estimate its total cost before dispatch, and make the aggregation account for every single input — a summary of thirty-seven of forty contracts that doesn't mention the missing three is worse than no summary. A perfect subagent executing the wrong subtask is still wrong."

## What it is

"Define down" means turning a vague goal into a set of subtasks each of which is precisely specified enough that a subagent (or a function, or a person) could execute it without further clarification. The unit of decomposition is a **contract**, not a sentence:

```
Goal: "Review these 40 vendor contracts for auto-renewal risk"

Bad decomposition:  ["look at the contracts", "find risks", "write a summary"]

Good decomposition:
  subtask: review_contract
    input:   {doc_id, version, checklist_version}
    output:  {doc_id, has_auto_renewal: bool, notice_days: int|null,
              renewal_term: string|null, evidence: [{claim, chunk_ids}], status}
    success: every field either populated with a citation or explicitly null with a reason
    tools:   [search_doc(doc_id)]        budget: 25k tokens, 10 steps
    tier:    fast
  subtask: aggregate_risk
    input:   [review_contract outputs]
    output:  {ranked_risks: [...], summary: string, coverage: {reviewed, failed}}
    success: every input doc appears in either ranked_risks or coverage.failed
```

Three properties distinguish a good subtask contract: **independently verifiable** (you can check the output without redoing the work), **narrow enough for a small model**, and **composable** (the orchestrator can combine outputs mechanically).

The source file's warning is the operative point: decomposition errors are unrecoverable downstream. A subagent that perfectly executes the wrong subtask produces a confidently wrong result.

## How to implement

1. **Prefer static decomposition.** If the subtask structure is knowable at design time (per document, per section, per checklist item), write it in code. Reserve LLM-generated decomposition for genuinely variable goals.
2. **Define contracts as typed schemas** (Pydantic/JSON Schema) with descriptions — these serve as both validation and the subagent's instructions ([structured outputs & tool use](../../../05-prompt-and-context-engineering/apply/1-prompt-engineering/structured_outputs_tool_use.md)).
3. **Specify success criteria explicitly**, machine-checkable where possible: required fields, citation coverage, "every input accounted for".
4. **Attach resources to each subtask**: tool allow-list, model tier, token/step/time budget ([cost & step budgets](cost_step_budgets.md), [context isolation](../../../05-prompt-and-context-engineering/apply/2-context-engineering/context_isolation.md)).
5. **Validate the decomposition itself** before dispatch: does the union of subtask outputs actually answer the goal? Are there gaps or overlaps? A cheap LLM critique of the plan is far cheaper than executing a bad one.
6. **Cap the fan-out** and the depth (usually 1 level; 2 rarely; 3 almost never).
7. **Make the plan visible and revisable** — persist it ([state & durability](state_durability.md)) so a resumed or corrected run doesn't re-plan from scratch.

## Small example

```python
from pydantic import BaseModel, Field

class SubtaskContract(BaseModel):
    name: str
    input_schema: dict
    output_schema: dict
    success_criteria: list[str]
    tools: list[str]
    model_tier: str = "fast"
    max_steps: int = 10
    max_tokens: int = 25_000

class Plan(BaseModel):
    goal: str
    subtasks: list[dict]                 # [{contract_name, input}]
    aggregation: str
    covers_goal: bool = Field(description="Set false if the subtasks do not fully address the goal")
    gaps: list[str] = []

PLAN_SYS = """Decompose the goal into subtasks using ONLY the available contracts.
Rules:
- Every subtask must use one of the listed contracts and supply valid input for it.
- The union of subtask outputs must fully answer the goal; if it cannot, set covers_goal=false and list gaps.
- Prefer the fewest subtasks that cover the goal. Maximum 50.
Available contracts: {contracts}
Output JSON matching the Plan schema."""

def plan_and_validate(goal: str, contracts: dict[str, SubtaskContract], max_fanout=50) -> Plan:
    raw = llm("plan", PLAN_SYS.format(contracts=json.dumps({k: v.model_dump(exclude={"tools"}) for k, v in contracts.items()})),
              [{"role": "user", "content": goal}], max_tokens=1500)
    plan = Plan.model_validate_json(raw.text)

    # structural validation — cheap, catches most bad plans
    assert len(plan.subtasks) <= max_fanout, "fan-out too large"
    for st in plan.subtasks:
        c = contracts[st["contract_name"]]                                  # KeyError = invented contract
        jsonschema.validate(st["input"], c.input_schema)                    # invalid input = bad plan
    if not plan.covers_goal:
        raise PlanIncomplete(plan.gaps)

    # semantic validation — a second opinion before spending money on execution
    critique = json.loads(llm("plan_critique", CRITIQUE_SYS, [{"role": "user", "content":
                f"<goal>{goal}</goal><plan>{plan.model_dump_json()}</plan>"}]).text)
    if not critique["sufficient"]:
        log_plan_rejected(goal, critique["reason"])
        raise PlanInsufficient(critique["reason"])
    persist_plan(plan)                                                       # resumable (state & durability)
    return plan

# Static decomposition — preferred whenever the structure is knowable
def plan_contract_review(doc_ids, checklist_version) -> Plan:
    return Plan(goal="review contracts for auto-renewal risk",
                subtasks=[{"contract_name": "review_contract",
                           "input": {"doc_id": d, "version": current_version(d), "checklist_version": checklist_version}}
                          for d in doc_ids],
                aggregation="aggregate_risk", covers_goal=True)
```

## Pros

- **Contracts make subagents reliable**: narrow scope, typed output, checkable success — a fast model can do the work.
- **Parallelism becomes trivial** when subtasks are independent ([fan-out / fan-in](fan_out_fan_in.md)).
- **Failures are localised and attributable** to a named subtask.
- **Cost is predictable**: budget per subtask × count, known before execution.
- The plan is a reviewable artifact — a human can approve it before anything expensive happens ([human-in-the-loop](human_in_the_loop.md)).

## Cons

- **LLM-generated plans are the weakest link**: plausible-looking decompositions that miss a requirement or double-count.
- **Upfront design cost** — writing contracts is real work.
- **Rigid contracts can't adapt** when execution reveals the plan was wrong (needs replanning, [loop-back / reflection](loop_back_reflection.md)).
- **Over-decomposition** adds coordination overhead for tasks a single call handles.
- Aggregation logic is often harder than the subtasks.

## When to use

- Fan-out work over a known collection (documents, sections, records, checklist items) — the highest-value case.
- Multi-stage workflows with distinct phases.
- Any task where you want to run cheap models on narrow pieces instead of an expensive model on everything.

## When NOT to use

- Simple tasks a single call handles well — decomposition is overhead.
- Genuinely exploratory work where subtasks can't be known until execution starts (use bounded recursion, [recursive / agentic retrieval](../../../06-rag/apply/6-retrieval-ranking/recursive_agentic_retrieval.md), or an agentic loop, [agentic RAG](../../../06-rag/apply/7-advanced-rag/agentic_rag.md)).
- When you can't define success criteria for the subtasks — that's a sign the goal itself is underspecified.

## Production at scale

**Static beats dynamic, overwhelmingly.** In production, most decompositions are structural: per document, per section, per row, per checklist item. Write those in code with typed contracts. Reserve LLM planning for the minority of goals whose shape genuinely varies — and even then, constrain it to selecting from a fixed contract catalogue rather than inventing subtasks.

**Validate plans before executing them.** Structural validation (contracts exist, inputs match schemas, fan-out within limits) is free and catches most bad plans. A cheap LLM critique catches more. Both are far cheaper than executing 40 subtasks based on a flawed plan.

**Coverage is a first-class output.** The aggregation step must account for *every* input: reviewed, failed, or skipped, with reasons. A summary of 37 of 40 contracts that doesn't mention the missing 3 is a dangerous artifact.

**Contract catalogue as a versioned asset** ([prompt versioning & evaluation](../../../05-prompt-and-context-engineering/apply/1-prompt-engineering/prompt_versioning_evaluation.md)): contracts have versions, evals, and owners. Changing an output schema is a breaking change for the aggregator.

**Budget the whole plan.** `subtasks × per-subtask budget` must fit an overall run budget, checked *before* dispatch. A plan that would cost $40 should be rejected or escalated for approval ([human-in-the-loop](human_in_the_loop.md), [cost & step budgets](cost_step_budgets.md)), not discovered mid-execution.

**Persist the plan** so a crashed run resumes from completed subtasks rather than re-planning and re-executing ([state & durability](state_durability.md)). Plan + per-subtask status is the checkpoint.

**Human approval for expensive or irreversible plans**: show the plan, the fan-out, and the estimated cost before executing. This is where an approval gate is cheapest and most useful.

**Evaluate decomposition separately** from execution ([component vs end-to-end](../../../09-evaluation/apply/component_vs_end_to_end.md)): given a goal, does the planner produce a plan a human would accept? Low end-to-end quality with high subtask quality means the decomposition is at fault.

## Prod checklist

- [ ] Static decomposition in code wherever the structure is knowable
- [ ] Typed contracts: input schema, output schema, success criteria, tools, tier, budget
- [ ] LLM planning constrained to a fixed contract catalogue; no invented subtasks
- [ ] Structural + semantic plan validation before dispatch
- [ ] Whole-plan cost estimated and budget-checked before execution; approval gate above a threshold
- [ ] Fan-out and depth caps
- [ ] Aggregation accounts for every input (reviewed / failed / skipped with reasons)
- [ ] Plan persisted for resumability; contracts versioned with evals
- [ ] Decomposition quality evaluated independently of execution

## Metrics to watch

| Metric | Why |
|---|---|
| Plan rejection rate (structural / semantic) | Planner quality |
| Coverage: inputs accounted for in aggregation | Silent omissions |
| Subtask success rate by contract | Where execution fails |
| Estimated vs actual plan cost | Budget accuracy |
| End-to-end quality vs subtask quality | Attribution to decomposition |

## Related

- [Orchestrator/subagent](orchestrator_subagent.md) — who executes the contracts
- [Fan-out/fan-in](fan_out_fan_in.md) — parallel execution and aggregation
- [Context isolation](../../../05-prompt-and-context-engineering/apply/2-context-engineering/context_isolation.md) — contracts define the subagent's window
- [Budgets](cost_step_budgets.md) — per subtask and per plan
- [Human-in-the-loop](human_in_the_loop.md) — plan approval
