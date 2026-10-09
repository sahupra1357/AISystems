# Lesson 03 — Serving Architectures

## Why this lesson exists

How you **expose** a model shapes cost, latency, failure modes, and what you can evaluate in production. The same RAG pipeline can be a sync chat API, an async “I’ll email you” job, or a nightly FAQ builder—and those are different products.

This lesson compares serving shapes, sketches FastAPI + Docker, ties the LLM gateway/orchestrator to Part 4, covers caching and multi-model routing, and walks canary / blue-green / shadow rollouts with **eval + HITL** during promotion.

## Learning goals

- Choose among batch, sync API, async queue, and streaming with tradeoffs
- Sketch a FastAPI service and know when to add workers
- Apply orchestrator patterns (retriever, model, tools)
- Cache safely (and know when caching is dangerous)
- Route cheap → expensive models deliberately
- Roll out with canary/blue-green/shadow; wire eval and human review
- Decide self-host vs API provider for serving (decision table)

---

## 1. Serving shapes: options, tradeoffs, heuristics

### The four shapes

| Shape | What the user experiences | Typical tech |
|-------|---------------------------|--------------|
| **Batch** | Results appear in a table/dashboard later | Cron, Spark/pandas jobs, warehouse |
| **Sync API** | Request waits for response | FastAPI/uvicorn, load balancer |
| **Async queue** | Accepted now; result later (poll/webhook/email) | API + Redis/SQS/Rabbit + workers |
| **Streaming** | Tokens/events arrive incrementally | HTTP chunked / SSE / WebSockets |

### Tradeoff table

| Dimension | Batch | Sync | Async queue | Streaming |
|-----------|-------|------|-------------|-----------|
| UX latency | High | Low–medium | Medium (perceived) | Low first-byte |
| Implementation complexity | Low–medium | Medium | Higher | Higher |
| Backpressure handling | Natural (next run) | Need limits | Excellent | Need care |
| Cost predictability | Good | Traffic-coupled | Traffic-coupled | Similar to sync |
| Failure UX | Stale data | Error/fallback now | Retry invisibly | Partial output risk |
| Eval ease | Snapshot offline | Need request logs | Job logs + results | Log full stream |
| HITL fit | Review cohort before use | Inline escalate hard | Easy to insert review step | Hard mid-stream |

### How to choose

```text
Interactive chat / autocomplete?     → sync or streaming
Report / campaign / embeddings?      → batch
LLM call often > UX budget (15s+)?   → async or streaming
Need human approve before send?      → async + HITL queue (natural fit)
Unsure and UX allows delay?          → batch first
```

**Small-team default:** batch for scores; **sync FastAPI** for RAG chat; add **streaming** when users complain about waiting for full answers; add **queues** when timeouts pile up or work fans out to many tools.

### Eval + HITL by shape

- **Batch:** score → hold in `scores_staging` → human sample → publish pointer
- **Sync:** golden in CI; canary %; review thumbs-down asynchronously (cannot block every request)
- **Async:** optional `needs_review` state before delivering to user (great for medium/high tier)

---

## 2. Sync API sketch: FastAPI + model

```python
# Conceptual sketch — verify current FastAPI / Pydantic docs when implementing
from fastapi import FastAPI, Header, HTTPException
from pydantic import BaseModel, Field
import joblib
import time
import uuid

app = FastAPI()
model = joblib.load("model.joblib")  # load once at startup


class PredictRequest(BaseModel):
    age: float = Field(..., ge=0)
    income: float = Field(..., ge=0)


class PredictResponse(BaseModel):
    probability: float
    request_id: str
    model_version: str


@app.get("/health")
def health():
    return {"ok": True, "model_version": "churn_rf/v3"}


@app.post("/predict", response_model=PredictResponse)
def predict(req: PredictRequest, x_api_key: str | None = Header(default=None)):
    # Production: real authn/z, rate limits, metrics
    if not x_api_key:
        raise HTTPException(status_code=401, detail="missing api key")
    request_id = str(uuid.uuid4())
    t0 = time.time()
    X = [[req.age, req.income]]
    proba = float(model.predict_proba(X)[0][1])
    latency_ms = (time.time() - t0) * 1000
    # log structured fields (redacted) — see Lesson 04
    _ = latency_ms
    return PredictResponse(
        probability=proba,
        request_id=request_id,
        model_version="churn_rf/v3",
    )
```

### LLM / RAG handler shape (conceptual)

```text
POST /chat
  validate + auth
  → orchestrator:
       retrieve(k, filters=ACL)
       build_prompt(prompt_version)
       call_llm(model_id, timeout)
       validate_output / filter
  → return answer + citation ids + request_id
```

### Docker sketch

```text
Dockerfile
  FROM python:3.12-slim
  WORKDIR /app
  COPY requirements.txt .
  RUN pip install --no-cache-dir -r requirements.txt
  COPY app/ ./app/
  # large artifacts: COPY or download at boot from object storage
  CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Pin versions. Same image in staging and production. Config via env (model path, provider keys)—never bake secrets into the image.

### When this is enough

Single region, moderate QPS, sync work < few seconds, one process can hold the model. Many internal tools stop here happily.

---

## 3. When to add a worker queue

Add async workers when:

| Signal | Why queue helps |
|--------|----------------|
| p95 handler time > UX or gateway timeout | Work continues after accept |
| Spiky LLM latency | Buffer absorbs spikes |
| Fan-out tools / multi-doc summarize | Parallel workers |
| HITL approval before deliver | Natural state machine |
| Rate limits on provider | Controlled egress |

### Minimal architecture

```text
Client → API: enqueue job → 202 + job_id
              │
              ▼
           Queue
              │
              ▼
           Worker: retrieve → LLM → store result
              │
Client poll GET /jobs/{id}  or  webhook / email
```

### Tradeoffs vs sync

| | Sync | Async |
|--|------|-------|
| Client complexity | Low | Higher (poll/webhook) |
| Timeout pain | High | Lower |
| Ops | Fewer moving parts | Broker + workers + idempotency |
| HITL insert | Awkward | Natural |

**Heuristic:** if you need more than one retry layer and work > 10s, design async early rather than bolting it on during an incident.

---

## 4. LLM gateway / orchestrator pattern (ties to Part 4)

Part 4’s mental model becomes a service boundary:

```text
                 ┌─────────────────────────┐
  Client ──────► │ API Gateway / BFF       │
                 └───────────┬─────────────┘
                             ▼
                 ┌─────────────────────────┐
                 │ Orchestrator service    │
                 │  - prompt registry      │
                 │  - policy / guardrails  │
                 │  - timeouts & fallbacks │
                 └─┬─────────┬─────────┬───┘
                   ▼         ▼         ▼
              Retriever   LLM API    Tools
              (index v)   (model)   (allowlist)
                   │         │         │
                   └────┬────┴────┬────┘
                        ▼         ▼
                   Logs/traces   Eval hooks
```

**Why a gateway helps:** one place for auth, cost meters, model routing, redaction, and version stamps (`prompt_version`, `index_version`). Product surfaces stay thin.

**When not to:** a single script serving 3 internal users—avoid premature platform. Extract when a second surface needs the same RAG stack.

### Eval hooks at the gateway

- Attach `gold_run_id` in staging
- Emit events for online eval jobs
- Flag `shadow=true` responses discarded from UX but scored

---

## 5. Caching: prompt/prefix, retrieval, response

Caching saves money and latency—and can serve **wrong stale truth**.

### Cache layers

| Layer | Key idea | Safe when | Dangerous when |
|-------|----------|-----------|----------------|
| **HTTP/CDN** | Cache GET by URL | Static docs | Personalized answers |
| **Response cache** | hash(user_tier + question + versions) → answer | FAQ-like, low personalization | User-specific ACL data |
| **Retrieval cache** | hash(query) → chunk ids | Repeated queries | Freshness-critical corpus |
| **Embedding cache** | text → vector | Immutable text | Rarely dangerous |
| **Provider prompt prefix** | Reuse large system prompt | Supported providers | Still pay for unique suffixes |

### Tradeoffs

| Dimension | Aggressive cache | No cache |
|-----------|------------------|----------|
| Cost/latency | Better | Worse |
| Freshness | Risk of stale | Always live |
| Privacy | Risk of cross-user leakage if key weak | Safer |
| Eval | May hide live regressions | Sees true distribution |

### Safe key design

```text
cache_key = hash(
  prompt_version,
  index_version,
  model_id,
  acl_principal,   # critical
  normalized_question
)
TTL = short for high-churn corpora; longer for stable policies
```

**Never** share response cache entries across users unless content is explicitly public and identical.

### Eval + HITL

- Bust caches on promote of prompt/index
- Sample cached vs recomputed answers weekly for medium tier
- Thumbs-down should invalidate that key

---

## 6. Multi-model routing (cheap → expensive)

Not every request needs the strongest model.

```text
request → classifier / rules
            │
            ├─ easy FAQ ──► small/cheap model or canned retrieval
            ├─ normal ────► mid model + RAG
            └─ hard/high risk ─► strong model (+ HITL?)
```

### Options

1. Heuristic rules (keyword, length, intent regex)  
2. Tiny classifier model  
3. LLM router (ironic cost—use carefully)  
4. Confidence-based escalation (if small model abstains)

### Tradeoffs

| Approach | Pros | Cons |
|----------|------|------|
| Rules | Cheap, clear | Brittle |
| Classifier | Learns intents | Needs labels |
| LLM router | Flexible | Cost + latency |
| Escalation on abstain | Aligns with quality | Needs good abstain behavior |

### How to choose

**Small-team default:** rules + retrieval hit confidence → mid model; escalate to strong model when retrieval weak or user repeats question. Add a trained classifier once you have labeled intents from logs.

### Eval + HITL

- Gold set tagged by intended route; measure misroute rate
- Humans review a sample of “cheap route” answers (silent quality loss common here)
- Track cost per successful resolution, not only per call

---

## 7. Rollouts: shadow, canary, blue-green

### Shadow

Candidate runs in parallel; users see **control** only. Log candidate outputs for eval.

| Pros | Cons |
|------|------|
| No user harm from bad candidate | Double cost; need comparable inputs |
| Great for prompts/models | Hard for irreversible tools (do not shadow-charge cards) |

### Canary

Send a small % of live traffic to candidate; ramp if healthy.

| Pros | Cons |
|------|------|
| True UX signal | Some users see regressions |
| Cost-efficient vs full shadow | Needs fast kill switch |

### Blue-green

Two full environments; flip the router when ready.

| Pros | Cons |
|------|------|
| Instant rollback flip | Infra cost of two stacks |
| Clean for big bang | Less gradual learning than canary |

### Tradeoff summary

| | Shadow | Canary | Blue-green |
|--|--------|--------|------------|
| User risk | Lowest | Low–medium | Medium at flip |
| Infra cost | Compute double | Slightly higher | Near double env |
| Speed of learning | Offline compare | Live metrics | Coarse |
| Best for | Prompt/model quality | Gradual API changes | Infra/version flips |

**Small-team default:** **shadow** new prompts on logged traffic (or offline replay) → **canary 1→5→25→100%** with auto-rollback on error/latency/quality guards → keep previous bundle for instant flag flip (poor-man’s blue-green).

### How eval + HITL apply during rollout

```text
1. Offline gold pass
2. Shadow: automatic rubric/LLM-judge optional + human grade N paired outputs
3. Canary 1%: watch error, latency, thumbs-down; HITL reviews all thumbs-down + 10% canary sample
4. Ramp only if guards green and no severe human findings
5. Prod: ongoing sample per risk tier (Lesson 01 / 05)
```

**Tools/actions:** do not canary irreversible tools without HITL. Shadow must be side-effect free.

---

## 8. Self-host vs API provider (serving decision)

Extends Lesson 01 with serving-specific rows:

| Dimension | API provider | Self-host weights |
|-----------|--------------|-------------------|
| Elasticity | Vendor scales | You scale GPUs |
| Cold start | Rare | Real for large models |
| Regional failovers | Vendor story | Your SRE story |
| Max concurrency | Rate limits | Hardware bound |
| Binary safety / airgap | Usually no | Possible |
| Streaming UX | SDK support | You implement |
| Version pinning | Model id strings | Exact weights on disk |

**Default:** API until privacy, cost at scale, or custom weights demand GPUs. If self-hosting, put a **gateway** in front so apps do not care which backend serves.

### Eval

Blind A/B on gold + human preference on your domain before cutting over serving backend—tokenization and finetune differences shift outputs.

---

## 9. Latency budget example (RAG chat)

Target p95 ≤ 5000 ms:

```text
auth + validation          50 ms
retrieval (+ rerank)      300 ms
prompt build               20 ms
LLM time-to-first-token  800 ms
LLM completion           2500 ms
output filters             50 ms
network / misc            280 ms
────────────────────────────────
buffer                     ~1000 ms
```

Tactics if over budget: stream; cache retrieval; smaller model on easy route; reduce `k`; shrink system prompt; precompute frequent FAQ.

### Mini practice

Allocate a 800 ms budget for a classical `/predict` with a feature-store fetch. Where do you cut first if feature fetch is 500 ms?

---

## 10. Auth, validation, and multi-tenant basics

Production serving is not only ML:

- Authenticate callers (API keys, JWT, SSO)
- Authorize **data** (RAG ACL filters per principal)
- Validate input size and schema (DoS and injection surface)
- Rate limit per key/tenant
- Separate staging keys from prod

**Eval note:** include ACL tests in gold—“user A must not retrieve user B docs.”

---

## 11. Anti-patterns

| Anti-pattern | Why | Do instead |
|--------------|-----|------------|
| One giant synchronous agent with 12 tools | Timeouts & chaos | Cap tools; async; HITL on writes |
| Cache by raw question only | Cross-user leaks / stale ACL | Key includes principal + versions |
| Canary irreversible refunds | Real money damage | Pre-action HITL; shadow without side effects |
| Deploy prompt without index pin | Irreproducible answers | Bundle versions |
| No `/health` or version stamp | Blind incidents | Expose version bundle in health/debug |

---

## 12. Worked mini-design: support bot serving

**Chosen path (small team):** Sync FastAPI orchestrator + provider LLM + pgvector; streaming responses; response cache off for personalized tickets; canary prompts at 5%.

**Why not async?** Chat UX expects turn-taking under ~5s; streaming covers perceived wait.

**Why not self-host?** Volume low; privacy OK under vendor DPA.

**HITL:** no pre-action (read-only); 5% post-hoc + 100% thumbs-down; refund tool stays disabled until Lesson 05 controls exist.

**Rollback:** feature flag `prompt_bundle=v17` → `v16` in one config change.

---



---

## 13. Streaming specifics

### Options

| Transport | Pros | Cons |
|-----------|------|------|
| Server-Sent Events (SSE) | Simple one-way; good for tokens | One-directional |
| WebSockets | Bidirectional | More ops/complexity |
| Chunked HTTP | Universal | Client support varies |

### Tradeoffs vs full-buffer sync

| | Streaming | Wait-for-full |
|--|-----------|---------------|
| Time-to-first-token UX | Better | Worse |
| Cancellation | User can stop early | Harder |
| Output filtering | Must filter incrementally or delay flush | Easier full-text filters |
| HITL pre-action | Poor fit mid-stream | Better with async |

**Safety note:** if output filters need the full answer (PII spanning tokens), buffer then release—or stream only after a classifier on completed sentences. Blind token streaming can leak a phone number token-by-token before a regex runs.

### Eval

- Measure TTFT and abandonment
- Humans review completed streams; optionally review early-cancel cases (were we slow or wrong?)

## What is next

**[04-monitoring-drift-and-cost.md](04-monitoring-drift-and-cost.md)** — what to log, quality monitors, drift, SLOs, dashboards vs traces, and human review queues.
