# Context engineering → Compaction / summarisation of long history

## What it is

Turn 40 of a support chat, or step 60 of an agent run, has 30k tokens of history. Sending all of it every call is expensive, slow, and degrades attention (["Lost in the middle"](lost_in_the_middle.md)). Truncating the oldest turns loses the constraint the user stated at turn 2.

Compaction replaces old turns with a **structured summary** that preserves what matters — decisions, constraints, entities, open questions, user preferences, tool results that are still relevant — and keeps the most recent N turns verbatim for local coherence:

```
[system]
[summary of turns 1–34: goals, decisions, constraints, entities, unresolved items]
[turn 35 verbatim] … [turn 40 verbatim]
[user turn 41]
```

The summary is regenerated (or incrementally updated) when history exceeds a threshold. For agents, the same applies to tool-call/observation history: compact old observations to their conclusions.

## How to implement

1. **Trigger**: when history tokens > `H_max` (e.g. 6k for chat, larger for agents) or every K turns.
2. **Structured summary prompt** (fast tier, temperature 0): produce fixed sections — `goal`, `decisions_made`, `constraints_and_preferences`, `entities_and_ids`, `facts_established (with source refs)`, `open_questions`, `last_state`. Structured beats prose: it resists drift and is diffable.
3. **Incremental update**: feed `previous_summary + turns since` → new summary; never re-summarise from raw history each time (cost and drift).
4. **Keep N recent turns verbatim** (N=4–8 for chat; for agents, the last 2–3 tool cycles).
5. **Pin critical items** outside the summary: the original user request, explicit constraints, and any safety-relevant statements are stored verbatim in a `pinned` block that never gets summarised away.
6. **Store the summary** with the session (durable), with version and the turn range it covers; log each compaction.
7. **Evaluate on multi-turn golden conversations**: does the answer at turn 30 still respect the constraint from turn 2? ([generation evals](../../../09-evaluation/apply/generation_evals.md), [agent evals](../../../09-evaluation/apply/agent_evals.md))

## Small example

```python
SUMMARY_SYS = """Update the conversation summary. Output JSON with keys:
goal, decisions_made (list), constraints_and_preferences (list), entities_and_ids (list),
facts_established (list of {fact, source}), open_questions (list), last_state (string).
Keep every constraint, id, number, and decision. Remove chit-chat. Max 350 words total."""

class Session:
    def __init__(self): self.turns, self.summary, self.pinned, self.covered = [], None, [], 0

    def add(self, role, text, pin=False):
        self.turns.append({"role": role, "text": text})
        if pin: self.pinned.append(text)

    def compact_if_needed(self, llm, h_max=6000, keep=6):
        old = self.turns[self.covered:-keep]
        if sum(T(t["text"]) for t in self.turns[self.covered:]) < h_max or not old:
            return
        prev = json.dumps(self.summary) if self.summary else "{}"
        transcript = "\n".join(f"{t['role']}: {t['text']}" for t in old)
        out = llm("history_compact", SUMMARY_SYS, [{"role": "user", "content": f"<previous_summary>{prev}</previous_summary>\n<new_turns>{transcript}</new_turns>"}])
        self.summary = json.loads(out.text); self.covered = len(self.turns) - keep
        log_compaction(len(old), T(transcript), T(out.text))

    def context(self):
        return {"pinned": self.pinned, "summary": self.summary,
                "recent": self.turns[self.covered:] if self.summary else self.turns}
```

## Pros

- **Constant-ish cost per turn** regardless of conversation length.
- **Better attention**: less noise, key facts surfaced in a compact block.
- **Durable state**: the summary is a checkpoint you can persist and resume from ([state & durability](../../../07-agents-tools-memory/apply/1-agents/state_durability.md)).
- Structured summaries are auditable and testable.

## Cons

- **Lossy**: the summariser drops something eventually — a number, a negation, a "not before Friday". Pinning mitigates; evals catch.
- **Drift**: repeated incremental summarisation can amplify an early error (context poisoning, [failure modes: poisoning, distraction, confusion, clash](context_failure_modes.md)).
- **Extra LLM calls** (cheap, but nonzero) and latency at compaction time.
- Summaries can leak into answers verbatim ("As summarised, …") — instruct against.

## When to use

- Any multi-turn assistant beyond ~10 turns.
- Long-running agents with many tool calls.
- Sessions that must be resumable.

## When NOT to use

- Short, bounded interactions (≤ 5 turns) — send verbatim.
- Legal/medical transcripts where every word matters — keep verbatim history in storage and summarise only for the model, with citations back to turn numbers.

## Production at scale

**Compaction off the critical path.** Trigger compaction asynchronously after a reply, so the next turn doesn't pay the latency. Fall back to a coarse truncation if the async job hasn't finished.

**Pinned block discipline.** Original request, hard constraints ("budget $5k", "no emails to the client"), identity/permission context, and anything safety-relevant. Never summarised. Bound its size; if users keep adding constraints, the pinned block is the source of truth.

**Facts with sources.** Summaries of RAG conversations should carry chunk IDs for facts so later citations remain valid and the faithfulness checker ([faithfulness check](../../../06-rag/apply/8-generation-grounding/faithfulness_check.md)) can verify.

**Agents**: compact tool observations to conclusions + IDs ("searched 3 queries; best result chunk c81; file X has 2 matches"); keep the raw observations in the trace store, not the window ([observability & tracing](../../../10-production/apply/4-observability-reliability/observability_tracing.md)).

**Poison control**: if a fact in the summary is later contradicted by a tool result or the user, update — supersede, don't append ([memory conflict resolution](../../../07-agents-tools-memory/apply/2-memory/conflict_resolution.md)). Include `facts_established[].source` so the model can weigh them.

**Cost**: compaction is a fast-tier call every K turns; cache the summary prompt prefix. Track tokens saved vs spent.

**Multi-turn evals**: golden conversations with constraints introduced early and tested late; metric: constraint adherence at turn ≥ 20, entity recall, and "summary contains all pinned facts" check ([generation evals](../../../09-evaluation/apply/generation_evals.md)).

**Privacy**: summaries are derived data with the same retention/PII rules as transcripts ([data residency & retention](../../../10-production/apply/6-security-privacy-governance/data_residency_retention.md), [PII](../../../10-production/apply/6-security-privacy-governance/pii.md)); erasure must cover them.

## Prod checklist

- [ ] Trigger by tokens/turns; incremental structured summary; last N verbatim
- [ ] Pinned block for request, constraints, permissions; never summarised
- [ ] Facts carry source/chunk IDs
- [ ] Async compaction with truncation fallback
- [ ] Summary persisted with turn range and version; compaction logged
- [ ] Supersede-on-contradiction rule
- [ ] Multi-turn golden conversations in the eval suite
- [ ] Summaries covered by retention/erasure policy

## Metrics to watch

| Metric | Why |
|---|---|
| History tokens per turn (before/after) | Cost control |
| Constraint adherence at late turns (eval) | Loss detection |
| Compaction latency and failure rate | UX |
| Pinned-block size | Discipline |

## Related

- [Context engineering overview](what_is_context_engineering.md), [Lost in the middle](lost_in_the_middle.md)
- [Context failure modes](context_failure_modes.md) — poisoning via summaries
- [State & durability](../../../07-agents-tools-memory/apply/1-agents/state_durability.md) — summary as checkpoint
- [Memory conflict resolution](../../../07-agents-tools-memory/apply/2-memory/conflict_resolution.md)
- [Query rewriting](../../../06-rag/apply/5-query-understanding/query_rewriting.md) — uses history to resolve references

## Video takeaway (30 seconds)

"By turn forty, the chat history is thirty thousand tokens. Sending it all is slow, expensive, and the model stops paying attention to the middle anyway. Keep a structured rolling summary — goal, decisions, constraints, IDs, open questions — plus the last handful of turns verbatim. Two rules keep it safe: pin the original request and hard constraints outside the summary so they can never be summarised away, and update the summary incrementally, never from scratch. Then test it: does the answer at turn thirty still respect the rule the user stated at turn two?"
