# Foundations → Routing & escalation (cheap model first)

## What it is

Static tiering ([model selection & tradeoffs](model_selection.md)) assigns a model per call site. Routing/escalation makes it dynamic per *request*: most questions are easy; some are hard; you can't tell from the call site alone. So:

```
request ──► [complexity classifier / heuristics] ──► easy ──► fast model ──► confidence check ──► answer
                                                 └──► hard ──────────────────────────────────► reasoning model ──► answer
                                                                          low confidence ──────┘ (escalate)
```

Two escalation triggers:
- **Pre-routing**: a cheap classifier (or rules: query length, presence of "compare/analyse/why", multiple sub-questions, tool requirements) marks the request complex before any generation.
- **Post-check**: the fast model answers; a confidence signal (self-reported confidence, a verifier model, schema/validation failure, retrieval score floor, faithfulness check) decides whether to re-run on the reasoning model.

The economics: if 70% of requests are handled by the fast model at 1/10th the cost, the blended cost is ~0.37× — *if* the router itself is cheap and accurate. If escalation hits 40%+, you pay for the router, for the fast attempt, *and* for the reasoning model on nearly half of traffic; you'd have been better off with a static reasoning model.

## How to implement

1. **Define "hard"** for your domain from logs: multi-part questions, reasoning verbs, long inputs, ambiguous intent, agent tasks needing >2 tool calls, low retrieval scores.
2. **Pre-router**: start with rules; add a small classifier (fast-tier LLM with a fixed label set, or a fine-tuned small encoder) trained on labelled "was the fast model's answer acceptable?" outcomes.
3. **Fast attempt** with the fast-tier model.
4. **Confidence check**: cheapest reliable signal first — schema validation failure, empty/abstention answer, retrieval max score below floor, self-reported confidence < τ (calibrate!), or a fast verifier ("is this answer complete and supported? yes/no").
5. **Escalate** to the reasoning tier with the same context (and optionally the fast model's draft as a hint).
6. **Log** `route, escalated, reason, cost, latency, outcome` for every request; compute escalation rate and *escalation precision* (how often the escalated answer was actually better — sampled via LLM-judge or human review, [LLM-as-judge](../../09-evaluation/apply/llm_as_judge.md)).
7. **Tune τ and rules** to hit a target escalation rate (e.g. 15–25%) with acceptable quality on the golden set.

## Small example

```python
import re, json

HARD_PATTERNS = re.compile(r"\b(compare|contrast|why|analy[sz]e|trade-?offs?|step[- ]by[- ]step|explain how|versus|vs\.?)\b", re.I)

def pre_route(query: str, retrieval_max_score: float, n_subquestions: int) -> str:
    if n_subquestions > 1 or len(query.split()) > 60 or HARD_PATTERNS.search(query):
        return "reasoning"
    if retrieval_max_score < 0.35:
        return "reasoning"                        # weak context → needs a model that abstains well
    return "fast"

VERIFY_PROMPT = """Question: {q}
Answer: {a}
Context provided to the answerer: {ctx}
Is the answer complete, specific, and fully supported by the context? Reply with JSON: {{"ok": true|false, "reason": "..."}}"""

def answer_with_escalation(q, ctx, llm, tau_ok=True):
    route = pre_route(q, ctx["max_score"], ctx["n_sub"])
    if route == "fast":
        draft = llm("answer_generate_fast", SYSTEM, [{"role": "user", "content": build_prompt(q, ctx)}])
        v = json.loads(llm("answer_verify_fast", "", [{"role": "user", "content": VERIFY_PROMPT.format(q=q, a=draft.text, ctx=ctx['text'][:4000])}]).text)
        if v["ok"]:
            log_route(q, "fast", escalated=False); return draft.text
        reason = v["reason"]
    else:
        reason = "pre_route"
    final = llm("answer_generate_reasoning", SYSTEM, [{"role": "user", "content": build_prompt(q, ctx)}])
    log_route(q, "reasoning", escalated=(route == "fast"), reason=reason)
    return final.text
```

## Pros

- **Blended cost and latency drop** substantially when most traffic is easy.
- **Quality preserved** on hard requests.
- **Confidence signals double as quality monitoring**: the escalation reason distribution tells you what the fast model can't do.
- Works for agents too: cheap model for routine steps, escalate on tool failure or planning.

## Cons

- **Router errors** are silent quality losses (hard request handled by the fast model, no escalation) — the worst case is invisible.
- **Self-reported confidence is poorly calibrated**; naive thresholds either over- or under-escalate.
- **Double cost on escalated requests** (fast attempt + reasoning) and added latency.
- **Complexity**: another classifier, another prompt, more telemetry, more things to tune.
- Two models means two behaviours; users notice inconsistency in tone/format unless prompts are aligned.

## When to use

- High-volume traffic with a real mix of easy and hard requests (support, search Q&A, internal assistants).
- When $/request matters and evals show the fast model handles most of the golden set.

## When NOT to use

- **Low volume**: the savings don't cover the engineering and eval cost. Use a static tier.
- **Uniformly hard tasks** (legal analysis, code generation): escalation rate will be high; use the reasoning model directly.
- **Latency-critical paths** where the escalation round-trip blows the budget — use pre-routing only, no post-check.
- Before you have a golden set to measure router accuracy; otherwise you're guessing.

## Production at scale

**The 40% rule.** Escalation rate > ~40% means the router costs more than it saves. Compute blended cost explicitly: `p_fast × c_fast + p_esc × (c_fast + c_verify + c_reason) + c_router`. Compare to `c_reason`. If it's not a clear win on both cost and latency, remove the router.

**Escalation precision.** Sample escalated requests; judge whether the reasoning answer was better than the fast draft ([LLM-as-judge](../../09-evaluation/apply/llm_as_judge.md)). Low precision = you're escalating for nothing; tighten the trigger. Also sample *non-escalated* requests for quality — the invisible failure.

**Calibrate confidence.** If using self-reported confidence, fit a threshold on the golden set to maximise F1 of "should escalate". Prefer *structural* signals (schema failure, abstention, retrieval floor, verifier verdict) over the model's own number.

**Consistency across tiers.** Same system prompt, same output schema, same citation format for both models; run the format eval on both. Users shouldn't be able to tell which tier answered.

**Feedback loop.** Thumbs-down on a fast-tier answer is a labelled example for the router. Retrain/re-tune periodically ([data flywheel](../../10-production/apply/8-data-flywheel/data_flywheel.md)).

**Per-tenant and per-feature policies.** Enterprise tier always gets the reasoning model; free tier gets fast with escalation; high-stakes features (compliance answers) skip the fast path. Encode as config.

**Cost guardrails.** Daily reasoning-tier budget with alerting; if escalation spikes (a prompt regression made the fast model fail verification), you find out from the budget alert, not the invoice.

**Agents.** Route per step: routine tool selection on the fast model; planning/replanning and final synthesis on the reasoning model; escalate a step when tool errors repeat ([loop-back / reflection](../../07-agents-tools-memory/apply/1-agents/loop_back_reflection.md)).

## Prod checklist

- [ ] "Hard" definition from logs; rule-based pre-router first, classifier later
- [ ] Structural confidence signals preferred; thresholds calibrated on golden set
- [ ] Route, escalation, reason, cost, latency logged per request
- [ ] Escalation rate dashboard; blended-cost computation vs static reasoning
- [ ] Escalation precision and non-escalated quality sampled via judge/human
- [ ] Prompts/schemas aligned across tiers; format eval on both
- [ ] Per-tenant/feature routing policy
- [ ] Reasoning-tier daily budget alert

## Metrics to watch

| Metric | Why |
|---|---|
| Escalation rate (target band, e.g. 15–25%) | Router economics |
| Blended $/request vs static baseline | Is it worth it |
| Escalation precision (escalated answer better) | Trigger quality |
| Non-escalated answer quality (sample) | Invisible failures |
| Added latency on escalated requests | UX |

## Related

- [Model selection](model_selection.md) — static tiers
- [Score thresholding](../../06-rag/apply/6-retrieval-ranking/score_thresholding_abstention.md) — retrieval floor as a trigger
- [Faithfulness check](../../06-rag/apply/8-generation-grounding/faithfulness_check.md) — verifier as a trigger
- [LLM-as-judge](../../09-evaluation/apply/llm_as_judge.md) — measuring escalation precision
- [Cost levers](../../10-production/apply/1-serving-cost-latency/cost_levers.md)

## Video takeaway (30 seconds)

"Most questions are easy. Send them to the cheap model first, check the answer with a cheap verifier, and only escalate the hard ones to the expensive model. Done well, you cut cost by half or more. Done badly, you pay for the router, the cheap attempt, and the expensive model — on nearly every request. The rule of thumb: if you're escalating more than about forty percent, the router isn't earning its keep. And sample the answers that didn't escalate — that's where the silent failures hide."
