# Agents → Multi-agent communication (MCP, A2A, shared blackboard)

## Video takeaway

"Three different problems get called multi-agent. Sharing tools: that's MCP — implement your search tool once as a server and every client and agent can discover and call it. Handing a task to another agent across a real boundary: that's an envelope with a goal, an output schema, a budget, and the original caller's identity, sent over a durable queue. And several agents co-authoring one artifact: that's a shared blackboard with versioned section writes. Two warnings. A third-party MCP server's tool descriptions are instructions your model will follow — review them. And most 'multi-agent systems' should just be an orchestrator calling subagents in one process."

## What it is

Three distinct problems, often conflated:

**1. Tool interoperability — MCP (Model Context Protocol).** An open protocol (JSON-RPC over stdio or HTTP/SSE) where a *server* exposes tools, resources, and prompts, and any MCP-capable *client* (Claude Desktop, Claude Code, IDEs, your own agent) can discover and call them. The value: your `search_documents` tool is implemented once and works in every agent and every client, instead of being re-wrapped per framework.

**2. Agent-to-agent handoff — A2A-style protocols.** Passing a task from one agent to another across a boundary (different team, different system, different vendor), with a task envelope: goal, inputs, constraints, expected output schema, budget, and a correlation ID. Google's A2A and similar efforts standardise this; internally, most teams use their own envelope over a queue.

**3. Shared state — the blackboard.** When multiple agents contribute to *one* artifact (a report, a plan, a code change), they read and write a shared structured state object rather than passing messages. Coordination happens through the artifact.

```
MCP:        agent ──JSON-RPC──► MCP server ──► tools/resources    (one implementation, many clients)
A2A:        agent A ──task envelope──► agent B ──result envelope──► agent A
Blackboard: agent A ─┐
            agent B ─┼──► shared state object (versioned, locked) ──► final artifact
            agent C ─┘
```

## How to implement

**MCP**
1. Implement tools once as an MCP server (Python/TypeScript SDKs); expose `tools/list` and `tools/call`, plus `resources` for read-only data.
2. Keep tool descriptions to the same standard as native tools ([tool design](tool_design.md)) — they're prompt real estate wherever the server is used.
3. Enforce authorisation *inside* the server from the caller's identity, not from arguments the model supplies ([excessive agency](../../../10-production/apply/5-guardrails/excessive_agency.md)).
4. Version the server; pin the version in clients; treat description changes as prompt changes ([prompt versioning & evaluation](../../../05-prompt-and-context-engineering/apply/1-prompt-engineering/prompt_versioning_evaluation.md)).

**A2A / handoff**
5. Define a task envelope: `{task_id, goal, inputs, constraints, output_schema, budget, deadline, caller_context, correlation_id}` and a result envelope with `status` and provenance.
6. Communicate over a durable queue with idempotency ([state & durability](state_durability.md)), not a synchronous call that can't survive a restart.
7. Propagate identity and permissions explicitly — the receiving agent must act with the *original* caller's authority, not its own.

**Blackboard**
8. Model the artifact as a versioned structured object with sections owned by roles; writes are section-scoped with optimistic concurrency (compare-and-swap on a version).
9. Define a coordinator that decides who writes next; avoid free-for-all concurrent writes.

## Small example

```python
# --- MCP server: one implementation, reusable across clients ---
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("acme-knowledge")

@mcp.tool()
def search_documents(query: str, doc_type: str = "any", effective_after: str | None = None) -> dict:
    """Search internal documents by meaning and keywords.
    USE FOR: narrative questions about policies, contracts, handbooks.
    DO NOT USE FOR: counts or totals (use query_warehouse).
    RETURNS: {status, passages: [{id, score, text, source, page}], n_found}."""
    caller = current_caller()                       # from the MCP session's auth context, NOT from arguments
    flt = merge_filters(build_acl_filter(caller), validate_filters({"doc_type": {"eq": doc_type}})[0])
    hits = rerank_hosted(query, hybrid_retrieve_with_filter(query, flt), top_n=6)
    return {"status": "ok", "n_found": len(hits),
            "passages": [{"id": h["chunk_id"], "score": round(h["rerank_score"], 3),
                          "text": truncate(h["text"], 900), "source": h["title"], "page": h.get("page")} for h in hits]}

# --- A2A-style handoff envelope over a durable queue ---
from pydantic import BaseModel
from datetime import datetime

class TaskEnvelope(BaseModel):
    task_id: str; correlation_id: str; goal: str
    inputs: dict; constraints: list[str] = []
    output_schema: dict; budget: dict                       # {max_usd, max_steps, deadline}
    caller: dict                                            # identity + permissions, propagated
    idempotency_key: str; created_at: datetime

class ResultEnvelope(BaseModel):
    task_id: str; correlation_id: str
    status: str                                             # ok | failed | partial | rejected
    output: dict | None = None; reason: str | None = None
    provenance: list[dict] = []; cost_usd: float = 0.0; trace_id: str

def handoff(agent_queue: str, env: TaskEnvelope):
    queue.publish(agent_queue, env.model_dump_json(), idempotency_key=env.idempotency_key)   # durable (state & durability)
    audit_log(action="agent_handoff", to=agent_queue, task_id=env.task_id, caller=env.caller["id"])

# --- Blackboard with optimistic concurrency ---
def write_section(store, artifact_id: str, section: str, content: dict, agent: str, expected_version: int):
    with store.transaction():
        cur = store.get(artifact_id)
        if cur["version"] != expected_version:
            raise ConcurrentModification(cur["version"])            # caller re-reads and retries
        cur["sections"][section] = {"content": content, "by": agent, "at": now(), "v": cur["version"] + 1}
        cur["version"] += 1
        store.put(artifact_id, cur)
    return cur["version"]
```

## Pros

- **MCP**: write once, use in every client; a growing ecosystem of ready-made servers; clean separation between tool implementation and agent framework; standard discovery.
- **A2A/handoff**: crosses organisational and system boundaries; durable and auditable; lets specialised agents be owned by different teams.
- **Blackboard**: natural for collaborative artifacts; the state *is* the coordination, avoiding message-passing complexity.

## Cons

- **MCP servers are a supply-chain and injection surface**: a third-party server's tool descriptions are text your model follows, and its results enter your context ([model & supply-chain hygiene](../../../10-production/apply/6-security-privacy-governance/supply_chain_hygiene.md), [prompt-injection defence](../../../10-production/apply/5-guardrails/prompt_injection_defense.md)). Vet before enabling.
- **Latency and failure modes multiply** across agent boundaries; a handoff chain is a distributed system.
- **Identity propagation is the hard part** — an agent acting with its own service credentials instead of the caller's is a privilege-escalation bug ([excessive agency](../../../10-production/apply/5-guardrails/excessive_agency.md)).
- **Blackboards invite conflicts**: concurrent writes, contradictory sections, and unclear ownership.
- Multi-agent architectures are frequently adopted before they're needed; most tasks are better served by orchestrator/subagent within one process ([orchestrator / subagent](orchestrator_subagent.md)).

## When to use

- **MCP**: whenever the same tools are used by more than one agent, client, or team — and to consume existing servers rather than writing integrations.
- **A2A/handoff**: genuine boundaries — different teams, different security domains, different vendors, or asynchronous long-running work.
- **Blackboard**: several agents co-authoring one artifact (a report with sections, a plan with parts, a codebase change).

## When NOT to use

- **Don't build a multi-agent protocol inside one service** — orchestrator/subagent with function calls is simpler, faster, and easier to debug ([orchestrator / subagent](orchestrator_subagent.md)).
- Don't enable third-party MCP servers without review; a tool description is executable instruction text.
- Don't use a blackboard when a sequential pipeline works — shared mutable state is the hardest thing to reason about.

## Production at scale

**MCP server security is the headline risk.** Treat every server as untrusted until reviewed: read the tool descriptions (they instruct your model), check what data they access, pin the version, and re-review on upgrade. Prefer servers you or your vendors control for anything touching sensitive data. Run third-party servers with least privilege and, where possible, network-isolated ([sandboxing](sandboxing.md)).

**Authorisation belongs in the server**, derived from the session's authenticated identity. Never accept `tenant_id` or `user_id` as a tool argument — the model can supply anything ([excessive agency](../../../10-production/apply/5-guardrails/excessive_agency.md), [authentication & authorization](../../../10-production/apply/6-security-privacy-governance/authn_authz_permission_filtered_retrieval.md)).

**Identity propagation across handoffs**: pass a signed token or an internal assertion carrying the original caller's identity and scopes; the receiving agent enforces them. Log the full chain ([auditability](../../../10-production/apply/6-security-privacy-governance/auditability.md)). Without this, agent B does things agent A's user was never allowed to do.

**Durable, idempotent handoffs**: queue-based with idempotency keys and correlation IDs, so a retried delivery doesn't duplicate work ([state & durability](state_durability.md)). Set deadlines and define what happens on timeout.

**Budget propagation**: the envelope carries the remaining budget; the receiving agent enforces it and reports consumption back. Without this, a chain of agents can each spend "their" budget and the total is unbounded ([cost & step budgets](cost_step_budgets.md)).

**Tracing across boundaries**: propagate a correlation ID into every trace so a multi-agent run is one searchable timeline ([observability & tracing](../../../10-production/apply/4-observability-reliability/observability_tracing.md)). This is the single most valuable investment for debugging multi-agent systems.

**Blackboard discipline**: section ownership, optimistic concurrency with versions, an explicit coordinator deciding write order, and a final consistency pass that resolves contradictions between sections ([conflict handling](../../../06-rag/apply/8-generation-grounding/conflict_handling.md)).

**Start with one process.** The overwhelming majority of production "multi-agent systems" are an orchestrator calling subagents in-process ([orchestrator / subagent](orchestrator_subagent.md)). Cross-process protocols are for real boundaries, and they cost real reliability.

## Prod checklist

- [ ] Tools implemented once (MCP server or shared library) rather than per framework
- [ ] Third-party MCP servers reviewed (descriptions, data access), version-pinned, re-reviewed on upgrade
- [ ] Authorisation inside the server from the session identity; never from model-supplied arguments
- [ ] Handoffs over a durable queue with idempotency keys and correlation IDs
- [ ] Caller identity and scopes propagated and enforced by the receiving agent
- [ ] Remaining budget and deadline carried in the envelope and enforced downstream
- [ ] Correlation ID propagated into all traces; cross-agent timeline searchable
- [ ] Blackboard: section ownership, versioned compare-and-swap writes, coordinator, final consistency pass
- [ ] Justification documented for any cross-process agent boundary

## Metrics to watch

| Metric | Why |
|---|---|
| Per-server tool call volume, error rate, p95 latency | Health of shared tools |
| Handoff success / timeout / retry rates | Distributed reliability |
| End-to-end cost per multi-agent run | Budget propagation working |
| Cross-boundary permission denials | Identity propagation correctness |
| Blackboard write conflicts | Coordination design |

## Related

- [Orchestrator/subagent](orchestrator_subagent.md) — the in-process alternative, usually better
- [Tool design](tool_design.md) — MCP tool descriptions are prompts
- [State & durability](state_durability.md) — durable handoffs
- [Excessive agency](../../../10-production/apply/5-guardrails/excessive_agency.md), [AuthZ](../../../10-production/apply/6-security-privacy-governance/authn_authz_permission_filtered_retrieval.md) — identity propagation
- [Supply-chain hygiene](../../../10-production/apply/6-security-privacy-governance/supply_chain_hygiene.md) — third-party servers
