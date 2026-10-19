# Memory → Types (short-term, working, episodic, semantic, procedural)

## Video takeaway

"Agent memory isn't one thing, it's five. Short-term is just the context window. Working memory is the scratchpad for the current run. Episodic is what happened before — past questions and how they went. Semantic is durable facts about the user: their role, their region, their default entity. Procedural is learned how-tos. The key production split: exact facts go in a relational store where you can read, update, and delete them precisely; fuzzy recall goes in a vector store. Put the user's timezone only in a vector index and you're *hoping* to retrieve it. And build working memory first, episodic last — that's where the privacy and quality problems live."

## What it is

| Type | What it holds | Lifetime | Store | Retrieval |
|---|---|---|---|---|
| **Short-term** | The context window itself — current turns, current tool results | One request/run | None (it's the prompt) | N/A — managed by context engineering ([context engineering](../../../05-prompt-and-context-engineering/apply/2-context-engineering/)) |
| **Working** | An explicit scratchpad/state object the agent reads and writes across steps | One run | Run state ([state & durability](../1-agents/state_durability.md)) | Direct read; always in context |
| **Episodic** | Past interactions and their outcomes: "last time the user asked X, we did Y and they accepted it" | Weeks–months | Vector store + metadata | Similarity + recency |
| **Semantic** | Durable facts about the user/domain: role, team, preferences, entities they care about | Indefinite, superseded | KV/relational (exact) + vector (fuzzy) | Key lookup for known fields; similarity for open-ended |
| **Procedural** | Learned how-tos, skills, reusable prompts, successful workflows | Indefinite, versioned | File/config store; sometimes an example store ([dynamic few-shot selection](../../../05-prompt-and-context-engineering/apply/1-prompt-engineering/dynamic_few_shot_selection.md)) | By task type |

The two-store pattern matters: **exact facts belong in a relational/KV store** (`user.timezone = "Europe/Berlin"`, `user.default_entity = "Acme GmbH"`) where they can be read deterministically, updated, and deleted. **Fuzzy recall belongs in a vector store** ("things this user asked about that resemble this question"). Putting exact facts only in a vector store means you *hope* to retrieve the user's timezone; putting fuzzy recall in a relational store means you can't find anything you didn't anticipate.

## How to implement

1. **Classify before building.** For each thing you want to remember, ask: is it a fact (semantic), an event (episodic), a skill (procedural), or run state (working)? Different answers, different stores.
2. **Working memory = run state** ([state & durability](../1-agents/state_durability.md)): a typed object, checkpointed, always in context, discarded or archived at run end.
3. **Semantic memory as a typed profile**: `{user_id, field, value, source, valid_from, confidence}` in a relational store, with an update-not-append rule ([memory conflict resolution](conflict_resolution.md)). Inject only the fields relevant to the current task.
4. **Episodic memory as records**: `{interaction_id, user_id, query, outcome, artifacts, feedback, timestamp}` embedded for similarity search, filtered by user/tenant and recency ([when to retrieve memory](when_to_retrieve.md)).
5. **Procedural memory as versioned assets**: successful workflows, prompts, and query templates stored like code ([prompt versioning & evaluation](../../../05-prompt-and-context-engineering/apply/1-prompt-engineering/prompt_versioning_evaluation.md)), retrieved by task type — often the same machinery as dynamic few-shot ([dynamic few-shot selection](../../../05-prompt-and-context-engineering/apply/1-prompt-engineering/dynamic_few_shot_selection.md)).
6. **One injection point**: a memory assembler that, given the current task, returns a bounded set of memories with provenance, feeding the context assembler ([context engineering](../../../05-prompt-and-context-engineering/apply/2-context-engineering/what_is_context_engineering.md)).
7. **Scope everything by tenant and user** ([tenant / ACL capture at ingest](../../../06-rag/apply/1-ingestion-parsing/tenant_acl_capture_at_ingest.md)) and treat all of it as data, never instructions ([memory security](memory_security.md)).

## Small example

```python
from pydantic import BaseModel
from datetime import datetime

# --- Semantic: exact facts, relational, superseded not appended ---
class SemanticFact(BaseModel):
    user_id: str; tenant_id: str
    field: str                      # "timezone", "role", "default_entity", "reporting_currency"
    value: str
    source: str                     # "user_stated" | "profile_sync" | "inferred"
    confidence: float
    valid_from: datetime
    superseded_by: str | None = None

def get_semantic(conn, user_id, fields: list[str]) -> dict:
    rows = conn.execute("""SELECT field, value, source, confidence FROM semantic_facts
                           WHERE user_id=%s AND superseded_by IS NULL AND field = ANY(%s)""",
                        (user_id, fields)).fetchall()
    return {f: {"value": v, "source": s, "confidence": c} for f, v, s, c in rows}

# --- Episodic: events, vector + metadata, similarity + recency ---
class Episode(BaseModel):
    interaction_id: str; user_id: str; tenant_id: str
    summary: str                    # "asked about EMEA renewal terms; used the 2024 MSA; accepted answer"
    outcome: str                    # "accepted" | "edited" | "rejected" | "escalated"
    artifacts: list[str] = []       # chunk ids, doc ids, trace id
    created_at: datetime

def recall_episodes(vec, user_id, tenant_id, query, k=3, half_life_days=30):
    hits = vec.search(embed_queries([query])[0], limit=k * 4,
                      filter={"user_id": user_id, "tenant_id": tenant_id})
    now = datetime.utcnow()
    for h in hits:                                                   # relevance × recency decay (when to retrieve memory)
        age = (now - h.payload["created_at"]).days
        h.combined = h.score * (0.5 ** (age / half_life_days))
    return sorted(hits, key=lambda h: -h.combined)[:k]

# --- Procedural: versioned skills, retrieved by task type ---
def load_procedure(task_type: str, tenant_id: str) -> dict | None:
    return procedure_store.get(task_type, tenant=tenant_id) or procedure_store.get(task_type, tenant="global")

# --- One assembler, bounded, with provenance ---
def assemble_memory(user, task_type, query, token_budget=600) -> list[dict]:
    blocks = []
    facts = get_semantic(conn, user.id, RELEVANT_FIELDS[task_type])
    if facts:
        blocks.append({"kind": "semantic", "text": render_facts(facts), "provenance": "profile"})
    for ep in recall_episodes(vec, user.id, user.tenant_id, query):
        blocks.append({"kind": "episodic", "text": ep.payload["summary"],
                       "provenance": ep.payload["interaction_id"]})
    proc = load_procedure(task_type, user.tenant_id)
    if proc:
        blocks.append({"kind": "procedural", "text": proc["steps"], "provenance": proc["version"]})
    return fit_to_budget(blocks, token_budget)                        # Per-source truncation budgets
```

## Pros

- **Continuity**: the assistant remembers the user's role, entity, and preferences instead of asking every session.
- **Fewer clarifying questions** and better first-attempt answers.
- **Episodic recall improves over time** — a genuine flywheel ([data flywheel](../../../10-production/apply/8-data-flywheel/data_flywheel.md)) with no retraining.
- **Procedural memory captures institutional know-how** as reusable assets.
- Separating stores makes each part testable and deletable.

## Cons

- **Memory is a hallucination amplifier if wrong**: a stale or misremembered fact injected confidently into every future prompt is context poisoning with persistence ([failure modes: poisoning, distraction, confusion, clash](../../../05-prompt-and-context-engineering/apply/2-context-engineering/context_failure_modes.md)).
- **Writing everything makes recall useless** ([what to write to memory](what_to_write.md)) — the most common failure.
- **Privacy and compliance surface**: memory is personal data with retention, access, and erasure obligations ([how to forget](how_to_forget.md), [data residency & retention](../../../10-production/apply/6-security-privacy-governance/data_residency_retention.md)).
- **Injection vector**: memory written from one user's content and read as instructions later ([memory security](memory_security.md)).
- Complexity: three stores, an assembler, write policies, and conflict resolution.

## When to use

- Multi-session assistants where users return: internal copilots, support agents, personal assistants.
- Long-running agents where working memory across steps is necessary.
- Domains with stable user attributes (role, region, entity, preferences) that change answers.

## When NOT to use

- Single-shot Q&A systems — session context is enough; adding memory adds risk for no gain.
- Anonymous or high-privacy contexts where storing user facts isn't acceptable.
- Before the base system is good: memory amplifies both good and bad behaviour, and debugging a memory-laden system is much harder.

## Production at scale

**Start with working memory only, then semantic, then episodic.** Working memory (run state) is required for agents anyway. Semantic facts are high-value and low-risk if kept to a small, typed set of fields. Episodic memory is where quality problems and privacy exposure concentrate — add it last, with tight caps.

**Typed semantic fields beat free-form notes.** A fixed schema (`timezone`, `role`, `default_entity`, `communication_style`) is auditable, updatable, deletable, and reviewable by the user. A pile of free-text "memories about the user" is none of those things and drifts into speculation.

**Bound the injection.** Memory competes with retrieved context for the window ([per-source truncation budgets](../../../05-prompt-and-context-engineering/apply/2-context-engineering/per_source_truncation_budgets.md)). Cap it hard (a few hundred tokens), inject only fields relevant to the task type, and log what was injected — every memory in the prompt must be traceable to a record.

**Provenance and confidence on every memory**, surfaced to the model: "user stated" vs "inferred" changes how it should be used, and a low-confidence inference should not be asserted back to the user as fact.

**Per-tenant and per-user isolation** is absolute ([tenant / ACL capture at ingest](../../../06-rag/apply/1-ingestion-parsing/tenant_acl_capture_at_ingest.md)). Memory leakage across users is one of the most damaging failures possible in an assistant product.

**User-visible controls** are a requirement, not a nicety: view, correct, and delete memories ([how to forget](how_to_forget.md), [data residency & retention](../../../10-production/apply/6-security-privacy-governance/data_residency_retention.md)). Building this after launch is far harder than designing for it.

**Evaluate memory's effect** on the golden set: run multi-session scenarios with and without memory, measuring both improvement (fewer clarifications, better answers) and harm (stale facts, wrong assumptions). Memory that isn't measured will silently degrade.

## Prod checklist

- [ ] Each memory item classified as working / semantic / episodic / procedural
- [ ] Exact facts in a relational/KV store with a typed schema; fuzzy recall in a vector store
- [ ] Working memory = checkpointed run state ([state & durability](../1-agents/state_durability.md))
- [ ] Provenance, source, confidence, and `valid_from` on every stored memory
- [ ] Single memory assembler with a hard token budget; injected memories logged
- [ ] Tenant and user scoping enforced in every query
- [ ] User-visible view/correct/delete controls
- [ ] All memory treated as data, never instructions ([memory security](memory_security.md))
- [ ] Multi-session evals measuring both benefit and harm

## Metrics to watch

| Metric | Why |
|---|---|
| Memory injection rate and tokens per request | Budget discipline |
| Answer quality with vs without memory (eval) | Is it earning its risk |
| Stale/incorrect memory rate (user corrections) | Quality of writes |
| Cross-user leakage incidents (must be 0) | The critical safety property |
| Memory store growth per user | Write-policy discipline ([what to write to memory](what_to_write.md)) |

## Related

- [What to write](what_to_write.md), [When to retrieve](when_to_retrieve.md), [How to forget](how_to_forget.md), [Conflicts](conflict_resolution.md), [Security](memory_security.md)
- [Context assembly](../../../05-prompt-and-context-engineering/apply/2-context-engineering/what_is_context_engineering.md), [Budgets](../../../05-prompt-and-context-engineering/apply/2-context-engineering/per_source_truncation_budgets.md) — memory competes for the window
- [State & durability](../1-agents/state_durability.md) — working memory
- [Dynamic few-shot](../../../05-prompt-and-context-engineering/apply/1-prompt-engineering/dynamic_few_shot_selection.md) — procedural memory's cousin
- [Data residency & retention](../../../10-production/apply/6-security-privacy-governance/data_residency_retention.md) / [PII](../../../10-production/apply/6-security-privacy-governance/pii.md) — retention and PII
