# Lesson 04 — Tools, Agents, and Structured Output

RAG retrieves documents. Many product questions need **live actions or state**: look up an order, create a ticket, calculate a number, call an internal API. **Tool / function calling** lets the model request those actions in a structured way. **Agents** loop that process. **Structured output** makes results machine-checkable.

This lesson teaches why tools beat hallucinated actions, how to design schemas and validation loops, when a single tool call is enough versus an agent loop, and how least privilege keeps systems safe enough to ship.

## Learning goals

- Explain why free-text “I refunded the order” is unacceptable without a tool side effect you control
- Design JSON schemas for tools and for final answers; validate in code
- Implement a single-tool and multi-tool orchestration pattern with caps
- Contrast agent loops vs DAGs vs one-shot tool calls
- Apply least privilege, timeouts, idempotency, and human approval gates
- Place tools in the production LLM app architecture next to retriever and LLM

---

## 1. Why tools beat hallucinated actions

Language models emit tokens. They do not magically mutate your database. If the user reads “I’ve issued a refund” and no refund API ran, you have a **trust failure**.

| Approach | What happens | Risk |
|----------|--------------|------|
| Model narrates actions | Fluent story, no side effect | User believes false state |
| Model outputs JSON your code ignores | Still no effect | Same |
| Model emits **tool call**; your code executes after validation | Real effect | Must constrain power |
| Deterministic button in UI | Real effect, no LLM | Often best for irreversible actions |

**Tools exist so the model can propose typed intents while your server remains the executor and security boundary.**

---

## 2. Structured output fundamentals

### Three layers people confuse

1. **Natural language answer** for humans
2. **Structured final output** (JSON the app parses as the product result)
3. **Tool call arguments** (JSON for side effects / fetches)

All three benefit from schemas. Lesson 02 covered schema-first prompting; here we operationalize it.

### Strategies (strongest first when available)

1. Provider **structured output / JSON schema** binding
2. JSON mode + your schema validation
3. Prompted JSON + strict parse + limited repair retry
4. Free text + brittle regex (avoid for production)

```python
import json
from typing import Any

def parse_strict(raw: str, required_keys: set[str]) -> dict[str, Any]:
    data = json.loads(raw)
    missing = required_keys - data.keys()
    if missing:
        raise ValueError(f"missing keys: {sorted(missing)}")
    return data
```

### Validation loops

```text
attempt = 0
while attempt < 2:
  raw = call_model(...)
  try:
    data = validate(raw)
    break
  except ValidationError as e:
    attempt += 1
    raw = call_model_repair(raw, error=str(e))  # limited, logged
else:
  fail_closed()  # do not pretend success
```

**Why limited retries:** unbounded repair burns money and can amplify injection. Fail closed for high-stakes paths.

### Pydantic-style mental model

Even if you use another library, think:

- Types (str, int, enums)
- Ranges (priority 1–5)
- Extra fields forbidden
- Defaults explicit vs required

---

## 3. Tool / function calling mechanics

Conceptual flow:

```text
1. You declare tools: name, description, JSON Schema for arguments
2. Model returns either a normal message OR a tool call {name, args}
3. Your orchestrator validates args, executes, returns tool result message
4. Model continues with observation → final answer or another tool call
```

```python
TOOLS = [
    {
        "name": "get_order",
        "description": "Fetch an order by id for the authenticated customer.",
        "parameters": {
            "type": "object",
            "properties": {
                "order_id": {"type": "string", "pattern": "^[A-Z0-9]{8,12}$"}
            },
            "required": ["order_id"],
            "additionalProperties": False,
        },
    }
]

def run_tool(name: str, args: dict, user_ctx: dict) -> dict:
    if name != "get_order":
        raise PermissionError("unknown tool")
    order_id = validate_order_id(args["order_id"])
    order = order_db.get(order_id)
    if order is None or order["customer_id"] != user_ctx["customer_id"]:
        # Do not leak whether it exists in another tenant
        return {"error": "not_found"}
    return {"order_id": order_id, "status": order["status"], "total": order["total"]}
```

### Design rules

1. **Least privilege** — each tool can do the minimum
2. **Validate arguments server-side** — never trust model JSON shape alone
3. **Authz using server session**, not args like `customer_id` supplied by the model unless verified
4. **Timeouts, rate limits, size limits** on results fed back into context
5. **Idempotency** for writes (`Idempotency-Key`) when possible
6. **Human approval** for irreversible or expensive actions
7. **Never** treat the model as the security boundary

---

## 4. What makes a good tool description

The model chooses tools partly from descriptions. Write them like API docs for a careful junior engineer:

- When to use / when not to use
- Argument meanings and formats
- Side effects (read-only vs write)
- Error semantics

```text
name: create_support_ticket
description: >
  Create a support ticket for the authenticated user after they confirm.
  Do NOT use for password resets (use reset_password_flow).
  Do NOT invent severity; ask the user if unclear.
```

Bad descriptions (“powerful general database tool”) invite disaster.

---

## 5. Single tool call vs agent loops vs DAGs

### Single tool call (start here)

```text
User → model decides get_order → tool result → model final answer
```

Cap steps at 1–2. Easy to test and log.

### Directed workflow (DAG) you own

```text
retrieve policy (RAG) → maybe get_order → generate answer
```

Your code decides the graph; the model fills slots. Often **more reliable** than freeform agents.

### Agent loop

```text
loop until done or caps hit:
  model → (tool | final)
  if tool: execute → append observation
```

Powerful; easy to overengineer. Require:

- Max steps (for example 4)
- Max wall clock
- Max tokens / cost budget
- Allowlisted tools only
- Circuit breaker on repeated identical calls

```python
def agent_loop(client, messages, tools, user_ctx, max_steps=4):
    for step in range(max_steps):
        resp = client.chat(..., tools=tools, messages=messages)
        if resp.tool_calls:
            for call in resp.tool_calls:
                result = run_tool(call.name, call.args, user_ctx)
                messages.append({"role": "tool", "name": call.name, "content": json.dumps(result)})
            continue
        return resp.content  # final
    return "I could not finish within step limits; please refine the request."
```

### When agents are worth it

| Worth considering | Usually not |
|-------------------|-------------|
| Multi-source research with known APIs | Pure formatting / classify |
| Guided troubleshooting with 2–3 lookups | “Autonomous employee” with email+spend |
| Internal copilot with strong allowlists | Unbounded web + code exec |

---

## 6. Combining RAG + tools (support bot shape)

Example customer support flow:

1. Authenticate user (your app)
2. Retrieve policy chunks (RAG) filtered by locale/product
3. If order id present and owned by user → `get_order`
4. Generate answer citing policies + including order facts from tool JSON
5. Log retrieval ids, tool args/results hashes, model output, cost

```text
Do not invent order status.
If get_order returns not_found, say you could not find an order for this account.
Policy questions must follow Context; order facts must follow tool JSON.
```

---

## 7. Least privilege patterns

| Pattern | Example |
|---------|---------|
| Read-only tools by default | `get_order`, `search_docs` |
| Separate write tools | `request_refund` behind human confirm |
| Scoped credentials | Tool token can only hit `/orders/{id}` GET |
| Argument allowlists | status enums; max page size 50 |
| Result redaction | Strip SSN before returning to model context |
| Rate limits per user | Stop runaway loops |

**Human-in-the-loop (HITL):** for refunds, emails, production changes—show draft + args; require click.

---

## 8. Structured final answers for product UIs

Even without tools, return structured objects:

```json
{
  "answer_markdown": "...",
  "citations": ["c12", "c14"],
  "confidence": "low",
  "suggested_actions": [{"type": "open_url", "id": "refund_form"}]
}
```

Your UI renders markdown, resolves citations via metadata, and only enables actions that your backend authorizes again.

---

## 9. Failure modes

| Failure | Symptom | Mitigation |
|---------|---------|------------|
| Hallucinated tool success | “Refunded!” with no call | UI only confirms after server success |
| Invalid args | Tool errors loop | Tighter schema; enum; repair once |
| Over-looping | Cost spike | max_steps, budgets |
| Confused tool choice | Wrong API | Better descriptions; fewer tools |
| Sensitive data in context | PII in logs/prompts | Redact; retention policy |
| Confused authority | Model picks `customer_id` | Bind identity from session |
| Schema drift | New field breaks clients | Version schemas; reject unknowns |

---

## 10. Debugging playbook

1. Log the raw tool call proposals
2. Validate offline against schema fixtures
3. Replay with frozen tool results (deterministic stubs)
4. Ensure step cap triggers in tests
5. Compare DAG orchestration vs free agent on the same gold tasks—pick the simpler winner

```python
def test_get_order_rejects_other_tenant():
    user = {"customer_id": "A"}
    out = run_tool("get_order", {"order_id": "ORDERFROMB"}, user)
    assert out["error"] == "not_found"
```

---

## 11. Anti-patterns

| Anti-pattern | Prefer |
|--------------|--------|
| One `run_sql(query)` tool | Narrow parameterized tools |
| Agent with 25 tools day one | 2–3 tools; grow with eval |
| Trusting model narrated side effects | Server truth |
| Infinite repair loops | Fail closed |
| Putting API keys in tool descriptions | Secret manager |
| Parsing freeform ReAct text when native tools exist | Native function calling |

---

## 12. Production architecture (tools edition)

```text
User → API gateway → Orchestrator
                      ├─ Retriever (RAG)
                      ├─ LLM (prompt + tool schemas)
                      ├─ Tool executors (scoped creds)
                      ├─ Validator / schema registry
                      └─ Logs, traces, cost meters, eval hooks
```

Part 5 adds service hardening; your job now is to **own** this shape on a whiteboard and in a prototype.

---

## 13. Worked example: refund assistant (constrained)

**Allowed tools:** `get_order` (read), `request_refund` (write, requires `user_confirmed=true` and server-side eligibility checks).

**Policy:** RAG on refund policy; model cannot invent eligibility.

**Flow:**

1. Classify intent (no tools)
2. If refund help → retrieve policy
3. If order id → get_order
4. Model proposes `request_refund` only if policy + order support it
5. Orchestrator checks eligibility again; if OK, send HITL card; on confirm execute

**Gold tests:** ineligible order never calls write tool; cross-tenant order id returns not_found; policy abstain when chunks missing.

---

## 14. Practice

1. Write a JSON schema for `search_kb(query, product_area)` with max query length.
2. List three tools that must never be exposed without HITL.
3. Convert a freeform ReAct prompt into a native tool list + orchestrator caps.
4. Design structured output for a “meeting notes” feature with action items array.
5. Draw sequence diagram for RAG + one read tool + final JSON answer.

---



## 15. Argument validation catalog

Beyond JSON Schema types, enforce:

| Check | Example |
|-------|---------|
| Regex / format | order ids, ISO dates |
| Enum membership | `currency in {USD,EUR}` |
| Bounds | `page_size <= 50` |
| Referential integrity | ticket exists |
| Authz binding | resource owned by session user |
| Time windows | refund only if purchased_at within policy |
| Quotas | max N writes/day/user |

```python
def validate_order_id(value: str) -> str:
    if not isinstance(value, str) or not 8 <= len(value) <= 12 or not value.isalnum():
        raise ValueError("invalid order_id")
    return value.upper()
```

Schema catches shape; **business rules** catch meaning.

## 16. Tool result hygiene

Whatever you return to the model becomes future context:

- Prefer small, relevant JSON over huge blobs
- Redact secrets and unnecessary PII
- Include error codes the model can handle (`not_found`, `rate_limited`)
- Truncate long text fields with a clear `truncated: true` flag

```python
def slim_order(order: dict) -> dict:
    return {
        "order_id": order["order_id"],
        "status": order["status"],
        "total": order["total"],
        "currency": order["currency"],
        "purchased_at": order["purchased_at"],
    }
```

## 17. Parallel tool calls

Some models propose multiple tools at once. Only run in parallel if:

- Tools are independent (no ordering need)
- Your authz checks still apply per call
- You have a fan-out budget

Otherwise serialize. Parallelism is an optimization, not a reliability strategy.

## 18. Deterministic stubs for tests

```python
class FakeOrders:
    def get(self, order_id):
        return {"order_id": order_id, "customer_id": "A", "status": "delivered", "total": 42}

# In tests, inject FakeOrders into run_tool
```

Gold conversations should run without network I/O so CI is stable (Lesson 06).

## 19. Cost model for tool-using chats

Cost ≈ sum over steps of (prompt tokens + completion tokens) + tool infra cost.

Agents with verbose thoughts multiply tokens. Prefer:

- Short system prompts
- Slim observations
- Small models for routing; larger for final synthesis when needed
- Hard step caps

## 20. Anti-patterns deep dive: the “supertool”

`exec_python(code: str)` or `run_sql(sql: str)` looks flexible and fails spectacularly:

- Prompt injection → data exfiltration
- Accidental `DROP` / expensive scans
- Non-reproducible debugging

Replace with purpose-built tools: `get_order`, `list_invoices(start, end)`, `sum_column(table, column, filters)`.



## 21. Orchestrator checklist before enabling tools in prod

- [ ] Tool allowlist per feature flag / environment
- [ ] Schema registry versioned alongside prompts
- [ ] Authz tests for cross-tenant ids
- [ ] max_steps / max_cost enforced
- [ ] Timeouts on every executor
- [ ] Structured logs: tool name, latency, error class, arg hash (not raw secrets)
- [ ] HITL path for writes
- [ ] Gold traces covering happy path + denial path

## 22. Structured output vs tools: what goes where

| Need | Mechanism |
|------|-----------|
| UI needs fields from one shot | Structured final output |
| Must read/write external system | Tool call |
| Both | Tool(s) then structured final answer summarizing server truth |

Do not encode side effects only in structured output fields like `"refunded": true` without an executor—that field would be fiction.


## Key vocabulary

function / tool calling; JSON Schema; structured output; validation loop; fail closed; agent loop; DAG orchestration; least privilege; idempotency; HITL; allowlist; observation / tool result message

## What is next

**[05-fine-tuning-vs-rag-vs-prompt.md](05-fine-tuning-vs-rag-vs-prompt.md)** — decide when to change weights versus context versus prompts, with scenarios, LoRA intuition, data and cost realities.
