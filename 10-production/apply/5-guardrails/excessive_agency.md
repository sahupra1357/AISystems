# Testing & reliability → Excessive agency (least privilege)

## Video takeaway

"The strongest control in an agent system isn't a prompt — it's a credential. A read-only agent with a read-only token cannot be talked into a delete, no matter how clever the injection or how confused the model. So enumerate exactly what the task needs and remove everything else: if deletion isn't part of the product, the delete tool shouldn't exist in the catalogue at all. Make the agent act with the *user's* permissions, not a shared service account that can see every tenant. Put write tools only in the phase that writes. And write the negative tests — assert the agent *cannot* do the dangerous thing even when you explicitly tell it to. That test is your proof the control is real."

## What it is

Excessive agency is granting an agent more capability, permission, or autonomy than its task requires. It's the OWASP LLM Top 10 entry that turns a model failure into a business incident, and it has three dimensions:

| Dimension | Excessive form | Least-privilege form |
|---|---|---|
| **Functionality** | A tool that can read *and* write *and* delete | Separate `read_record` and `update_record`; no delete tool at all if deletion isn't in scope |
| **Permissions** | A service account with admin on the database | A role scoped to the specific tables and the caller's rows |
| **Autonomy** | Executes irreversible actions unsupervised | Approval gate for irreversible actions ([human-in-the-loop](../../../07-agents-tools-memory/apply/1-agents/human_in_the_loop.md)) |

The design principle is that **capability, not behaviour, is the control.** Prompting ("do not delete anything") is a suggestion the model may fail to follow. A token without delete permission is a guarantee that survives injection, confusion, hallucination, and model upgrades.

The classic failure chain: an agent is given a broad service account "for convenience"; a retrieved document contains an injection ([prompt-injection defence](prompt_injection_defense.md)); the model proposes a destructive action; nothing in the stack refuses because the credentials permit it.

## How to implement

1. **Enumerate what the task actually requires** — the specific tools, the specific resources, the specific operations. Anything beyond that is removed, not disabled by instruction.
2. **Act as the caller, not as the service.** The agent's data access should be the *user's* access, via a delegated token or an authorisation check against the user's permissions on every operation ([authentication & authorization](../6-security-privacy-governance/authn_authz_permission_filtered_retrieval.md)). Never a shared service account that can see everything.
3. **Split read and write tools**; expose write tools only in the phase/subagent that needs them ([context isolation](../../../05-prompt-and-context-engineering/apply/2-context-engineering/context_isolation.md), [tool design](../../../07-agents-tools-memory/apply/1-agents/tool_design.md)).
4. **Scope credentials narrowly**: read-only database roles, per-tenant API keys, storage prefixes, network egress allow-lists ([sandboxing](../../../07-agents-tools-memory/apply/1-agents/sandboxing.md)).
5. **Authorise every call in code** against the caller's permissions and the arguments ([prompt-injection defence](prompt_injection_defense.md)), before execution.
6. **Approval gates for irreversible actions** ([human-in-the-loop](../../../07-agents-tools-memory/apply/1-agents/human_in_the_loop.md)), and rate limits per action type.
7. **Test the negative cases**: assert the agent *cannot* perform out-of-scope actions even when explicitly instructed to ([the testing pyramid](../../../09-evaluation/apply/testing_pyramid.md)).

## Small example

```python
# --- Capability by role: what tools exist for this caller at all ---
ROLE_CAPABILITIES = {
    "viewer":   {"search_documents", "get_record"},
    "analyst":  {"search_documents", "get_record", "query_warehouse", "draft_email"},
    "operator": {"search_documents", "get_record", "query_warehouse", "draft_email",
                 "send_email", "update_record"},                       # no delete tool exists anywhere
}

def tools_for_caller(caller, phase: str) -> list[dict]:
    """Intersection of role capability and phase need — the agent never SEES what it may not use."""
    allowed = ROLE_CAPABILITIES[caller.role] & PHASE_TOOLS[phase]
    return [TOOL_DEFS[t] for t in sorted(allowed)]

# --- Credentials scoped to the caller, not to the service ---
def db_connection_for(caller):
    """Row-level security enforced by the database, not by generated SQL (structured + unstructured hybrid)."""
    conn = pool.getconn(role="rag_readonly")                            # read-only role: DDL/DML impossible
    conn.execute("SET LOCAL app.tenant_id = %s", (caller.tenant_id,))   # RLS policy uses this
    conn.execute("SET LOCAL statement_timeout = '10s'")
    return conn

def storage_client_for(caller):
    return storage.client(credentials=sts.assume_role(
        role_arn=TENANT_ROLE[caller.tenant_id],                         # per-tenant role
        policy=scoped_policy(prefix=f"tenants/{caller.tenant_id}/", actions=["s3:GetObject"]),
        duration_seconds=900))                                          # short-lived

# --- Authorisation in code, per call, with the caller's identity ---
def args_within_caller_scope(caller, tool_name: str, args: dict) -> bool:
    if tool_name == "get_record":
        return record_visible_to(caller, args["record_id"])             # checked against the ACL system
    if tool_name == "update_record":
        return record_writable_by(caller, args["record_id"]) and args.keys() <= WRITABLE_FIELDS
    if tool_name == "send_email":
        return all(is_permitted_recipient(caller, addr) for addr in args["to"])
    return True

def authorise(caller, tool_name, args, run_ctx) -> tuple[bool, str]:
    if tool_name not in ROLE_CAPABILITIES[caller.role]:  return False, "capability"
    if not args_within_caller_scope(caller, tool_name, args): return False, "scope"
    if TOOL_METADATA[tool_name]["writes"] and run_ctx.get("suspicious_content_in_context"):
        return False, "tainted_context"                                  # Prompt-injection defence
    if not rate_limiter.allow(f"{caller.id}:{tool_name}"):  return False, "rate_limit"
    if TOOL_METADATA[tool_name]["approval"] == "required":  return False, "requires_approval"   # Human-in-the-loop
    return True, "ok"

# --- Negative tests: the agent must be UNABLE, not merely unwilling ---
def test_viewer_cannot_send_email_even_if_instructed():
    caller = Caller(role="viewer", tenant_id="t1")
    tools = tools_for_caller(caller, phase="research")
    assert "send_email" not in {t["name"] for t in tools}                 # the tool doesn't exist for them
    ok, reason = authorise(caller, "send_email", {"to": ["a@b.com"]}, {})
    assert not ok and reason == "capability"

def test_readonly_role_cannot_write_even_with_valid_sql():
    conn = db_connection_for(Caller(role="analyst", tenant_id="t1"))
    with pytest.raises(psycopg.errors.InsufficientPrivilege):
        conn.execute("DELETE FROM contracts WHERE 1=1")                   # the database refuses, not the prompt

def test_cross_tenant_record_access_denied():
    caller = Caller(role="operator", tenant_id="t1")
    ok, reason = authorise(caller, "get_record", {"record_id": "t2_rec_9"}, {})
    assert not ok and reason == "scope"
```

## Pros

- **Survives every model failure**: injection, hallucination, confusion, and model upgrades cannot expand what the credentials permit.
- **Deterministic and testable** — negative tests prove the capability is absent ([the testing pyramid](../../../09-evaluation/apply/testing_pyramid.md)).
- **Bounds blast radius**: the worst case of a successful attack is limited to what the caller could already do.
- **Cheap**: it's configuration and ordinary authorisation code, not ML.
- Makes agents deployable in environments that would otherwise forbid them.

## Cons

- **Capability restriction limits usefulness** — the genuinely useful agent often needs write access somewhere.
- **Per-caller credential plumbing is real work**: delegated tokens, short-lived credentials, row-level security, per-tenant roles.
- **Approval gates add latency** and operational load ([human-in-the-loop](../../../07-agents-tools-memory/apply/1-agents/human_in_the_loop.md)).
- **Over-restriction pushes users to workarounds** (doing it manually, or asking for broader access), which can be worse.
- Fine-grained permission modelling is a design burden that grows with the product.

## When to use

Every agent with any tool access. Non-negotiable for any agent with write, financial, communication, or deletion capability.

## When NOT to use (relaxing it)

- Never for privileged actions. If a task genuinely requires broad access, the right answers are: split it into narrower subagents ([context isolation](../../../05-prompt-and-context-engineering/apply/2-context-engineering/context_isolation.md)), add an approval gate ([human-in-the-loop](../../../07-agents-tools-memory/apply/1-agents/human_in_the_loop.md)), or keep a human in the loop — not a broader token.

## Production at scale

**Act as the user, not as the system.** The single most important architectural decision: the agent's data access is the *caller's* access, enforced by delegated credentials or per-request authorisation. A shared service account that can read every tenant's data means any injection or bug is a cross-tenant breach ([authentication & authorization](../6-security-privacy-governance/authn_authz_permission_filtered_retrieval.md)).

**Remove tools rather than instructing against them.** A `delete_record` tool that exists but is "not to be used" will eventually be used. If deletion isn't part of the product, the tool shouldn't exist in the catalogue.

**Phase- and subagent-scoped tool sets** ([context isolation](../../../05-prompt-and-context-engineering/apply/2-context-engineering/context_isolation.md), [tool design](../../../07-agents-tools-memory/apply/1-agents/tool_design.md)): the research phase gets read tools; the sending phase gets one write tool. An injection in a retrieved document reaches a subagent that has nothing dangerous available.

**Short-lived, narrowly-scoped credentials**: assume-role with a session policy per request, 15-minute expiry, per-tenant prefixes. Long-lived broad keys in an agent's environment are the finding auditors look for first.

**Database-enforced row-level security** rather than filters in generated SQL ([structured + unstructured hybrid](../../../06-rag/apply/7-advanced-rag/structured_unstructured_hybrid.md)). The database is the boundary; generated SQL is untrusted input.

**Rate limits per action type per caller**: even permitted actions have a sane ceiling. An agent sending 400 emails is a bug regardless of permission.

**Audit every action with the caller's identity** ([auditability](../6-security-privacy-governance/auditability.md)): who asked, what was proposed, what was authorised, what executed. Regulated environments require it; incident response depends on it.

**Negative tests in CI as hard gates** ([the testing pyramid](../../../09-evaluation/apply/testing_pyramid.md)): for each role, assert the forbidden tools are absent and forbidden operations are denied. These tests are cheap and they're the ones that prove the control exists.

**Review capability grants periodically** — permissions accrete. A quarterly review of which agents hold which credentials catches the "we granted it for a migration and never revoked it" case.

## Prod checklist

- [ ] Tool catalogue per role; forbidden tools absent, not merely discouraged
- [ ] Agent acts with the caller's permissions via delegated or per-request authorisation, never a broad service account
- [ ] Read/write tool separation; write tools exposed only in the phase that needs them
- [ ] Short-lived, narrowly-scoped credentials (per-tenant roles, storage prefixes, read-only DB roles)
- [ ] Database-enforced row-level security; egress allow-lists for code execution ([sandboxing](../../../07-agents-tools-memory/apply/1-agents/sandboxing.md))
- [ ] Authorisation computed in code per call from caller identity and arguments
- [ ] Approval gates for irreversible actions; rate limits per action type
- [ ] Negative tests in CI proving forbidden actions are impossible
- [ ] Full audit of proposed vs authorised vs executed actions with caller identity
- [ ] Periodic review and revocation of accrued capability grants

## Metrics to watch

| Metric | Why |
|---|---|
| Denied actions by reason (capability / scope / rate / approval) | Policy layer working |
| Cross-tenant access attempts (must be 0) | Isolation |
| Agents holding long-lived or broad credentials (target 0) | Credential hygiene |
| Negative-test pass rate in CI | Proof the control exists |
| Actions executed per caller per period | Anomaly detection |

## Related

- [Prompt injection](prompt_injection_defense.md) — this is the layer that makes injection ineffective
- [Tool design](../../../07-agents-tools-memory/apply/1-agents/tool_design.md), [Human-in-the-loop](../../../07-agents-tools-memory/apply/1-agents/human_in_the_loop.md), [Context isolation](../../../05-prompt-and-context-engineering/apply/2-context-engineering/context_isolation.md)
- [AuthN/AuthZ](../6-security-privacy-governance/authn_authz_permission_filtered_retrieval.md), [Auditability](../6-security-privacy-governance/auditability.md), [OWASP](../6-security-privacy-governance/owasp_llm_top10.md)
- [Sandboxing](../../../07-agents-tools-memory/apply/1-agents/sandboxing.md) — no credentials in the sandbox
