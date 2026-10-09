# Lesson 11.1 — AI Architecture Patterns for Production

## Why this lesson exists

Lessons 10.1–10.7 give you a **decision framework**, MLOps habits, serving shapes, monitors, safety, and reliability. This lesson goes one level deeper: **named production architectures**—recurring topologies teams actually ship—with enough teaching depth that you can sketch one on a whiteboard, defend the choice, and say where eval and humans sit in the path.

You will not memorize vendor product names. You will recognize **patterns**, their **tradeoffs**, **failure modes**, and **when** each is the best path for a small-to-mid team.

## Learning goals

- Name and sketch ~15 production AI patterns with components and data/control flow
- Compare options and tradeoffs without defaulting to “just call the LLM”
- Attach **eval gates** and **HITL hooks** to each pattern by risk tier
- Apply cross-cutting concerns: authZ, PII, observability, idempotency, versioning
- Use a decision table / tree to pick a starting pattern under constraints

## Teaching contract (same as Stage 10)

For each pattern: **diagram-in-text → components → flow → tradeoffs → failure modes → when it’s best → eval + HITL → anti-patterns.**

---

## Pattern catalog at a glance

| # | Pattern | Latency shape | Typical risk fit |
|---|---------|---------------|------------------|
| 1 | Batch scoring pipeline | Hours–day | Low–medium (campaigns, ranks) |
| 2 | Real-time sync inference API | ms–few s | Medium (interactive) |
| 3 | Async job / queue inference | Seconds–minutes | Medium–high (reviewable) |
| 4 | Feature store + online serving | ms–s | Medium (classical + hybrid) |
| 5 | Model gateway / multi-model router | ms–s | Medium (cost/quality) |
| 6 | RAG production architecture | 1–10s | Medium–high (knowledge) |
| 7 | RAG + tools/agent (constrained) | 2–30s+ | High (actions) |
| 8 | HITL review workflow | Human-paced | Medium–high |
| 9 | Shadow / canary / A/B topology | Same as target | Any (release control) |
| 10 | Event-driven AI | Sub-second–seconds | Medium–high (act loops) |
| 11 | Edge / on-device vs cloud hybrid | Local ms + cloud | Privacy / offline |
| 12 | LLM gateway (policy/cache/cost) | Adds overhead | Cross-cutting |
| 13 | Ensemble / cascade classifiers | ms–s | Medium (precision/cost) |
| 14 | Offline train → registry → online | Release cycle | All MLOps |
| 15 | Multi-tenant SaaS AI | Same as product | High (isolation) |

Patterns **compose**. A SaaS RAG product might be **15 + 6 + 12 + 9 + 8**.

---

## 1. Batch scoring pipeline

### Diagram

```text
[Warehouse / lake] → [Extract job] → [Feature join]
        → [Score workers] → [scores_staging]
        → [Eval/HITL sample] → [Promote pointer] → [scores_prod]
        → [Downstream: CRM, email, dashboard]
```

### Components

- Scheduled orchestrator (cron / Airflow / Dagster)
- Feature materialization from versioned tables
- Model artifact from registry (`model_id@version`)
- Staging table + publish pointer (or partition swap)
- Offline eval harness + optional human review cohort

### Data / control flow

1. Snapshot inputs at `as_of` time (avoid leakage from “future” rows).
2. Score offline; write immutable run id + model version on every row.
3. Run batch metrics vs holdout / previous run; raise if gate fails.
4. Sample for HITL if risk tier warrants; only then flip `prod` pointer.

### Tradeoffs

| Pros | Cons |
|------|------|
| Cheap GPUs/CPU amortized; simple retries | Stale scores between runs |
| Natural HITL before publish | Not for payment-time decisions |
| Easy offline eval | Pipeline failures become silent staleness |

### Failure modes

- Training-serving skew if online later uses different feature code
- Partial job success → mixed model versions in one “batch”
- Downstream consumes staging by mistake

### When it’s the best path

Nightly churn, lead scoring, catalog embeddings, FAQ regeneration, risk lists refreshed daily.

### Eval + HITL

- **Eval:** PSI/drift vs last week; AUC/PR on labeled slice; row-count + null-rate gates
- **HITL:** 1–5% stratified sample of high-impact segments before publish; dual review on policy-sensitive cohorts

### Anti-patterns

- Writing straight to the table users read with no staging pointer
- Re-scoring only “changed” rows without versioning the rest (incomparable cohorts)

---

## 2. Real-time sync inference API

### Diagram

```text
Client → LB → [API pods]
              ├─ authn/z, validate, rate limit
              ├─ feature fetch (optional)
              ├─ model / LLM call
              ├─ post-filter / schema validate
              └─ respond + log (request_id, versions)
```

### Components

- Stateless API (e.g. FastAPI) + health/readiness
- Model loaded at startup or sidecar; or remote model server
- Timeouts, circuit breaker to dependencies
- Structured logging / traces with `request_id`

### Data / control flow

Request → authorize → validate schema → infer → optional guardrails → response. Side effects (DB writes) should be **idempotent** keyed by `request_id` or business key.

### Tradeoffs

| Pros | Cons |
|------|------|
| Simple UX; easy to reason | Tail latency = user pain |
| Clear SLOs | Hard to insert blocking HITL |
| Horizontal scale | Cost scales with QPS |

### Failure modes

- Cold start / model load spike
- Cascading timeout when retriever + LLM both slow
- Logging PII in plaintext prompts

### When it’s the best path

Autocomplete, fraud-at-checkout (classical), RAG chat with tight UX budget, ranking for search page.

### Eval + HITL

- **Eval:** golden set in CI; shadow traffic; online metrics (accept rate, thumbs-down)
- **HITL:** async review of low-confidence / thumbs-down (cannot block every sync call)

### Anti-patterns

- Holding the HTTP connection open for multi-minute agent loops
- No `model_version` / `prompt_version` on responses or logs

---

## 3. Async job / queue-based inference

### Diagram

```text
Client → API (enqueue) → [Queue]
                           ↓
                     [Workers]
                      ├─ infer
                      ├─ optional HITL state
                      └─ callback / store result
Client ← poll / webhook / email
```

### Components

- Ingress API that only validates + enqueues
- Durable queue (at-least-once)
- Workers with concurrency limits + DLQ
- Result store with TTL and status machine: `queued|running|needs_review|done|failed`

### Data / control flow

Enqueue returns `job_id`. Workers claim jobs; on success write result; on poison → DLQ. Idempotency: `(tenant_id, business_key)` unique so retries do not double-act.

### Tradeoffs

| Pros | Cons |
|------|------|
| Backpressure; natural HITL pause | More moving parts |
| Hides long LLM/tool latency | UX must tolerate delay |
| Retries without user spinning | Exactly-once is hard—design for at-least-once |

### Failure modes

- Duplicate side effects (double email, double refund)
- Queue backlog → SLA breach without user-visible progress
- Review queue starvation (jobs stuck in `needs_review`)

### When it’s the best path

Document summarization, video/audio jobs, “draft then send” emails, any **tool that mutates money/state**.

### Eval + HITL

- **Eval:** job success rate, p95 queue wait, quality on completed outputs
- **HITL:** first-class `needs_review` before external side effect for medium/high tier

### Anti-patterns

- Fire-and-forget with no DLQ or no job status API
- Side effects inside the worker with no idempotency key

---

## 4. Feature store + online model serving

### Diagram

```text
Offline: events → feature pipelines → offline store (training)
Online:  request → entity keys → online store (low-latency)
                              → model server → score
Training uses same feature definitions as serving (shared code/contract).
```

### Components

- Feature definitions (code or declarative) with owners
- Offline store (warehouse) + online store (Redis/KV)
- Point-in-time correct training set builder
- Model server reading **only** features available online at serve time

### Data / control flow

At train: point-in-time join avoiding leakage. At serve: lookup features by entity id; fail closed or use defaults with monitoring if missing.

### Tradeoffs

| Pros | Cons |
|------|------|
| Attacks training-serving skew | Platform cost / complexity |
| Reuse features across models | Stale online features if TTL wrong |
| Clear ownership | Overkill for one model + three columns |

### Failure modes

- Leakage via “as-of” bugs
- Online store outage → default features silently degrade quality
- Feature renamed in offline but not online

### When it’s the best path

Multiple classical models sharing entities (user, merchant); real-time fraud/credit; personalization at scale.

### Eval + HITL

- **Eval:** skew tests (offline vs online feature parity on replay); canary on feature freshness
- **HITL:** label audits on disputed fraud/credit decisions feeding feature bugs

### Anti-patterns

- “Feature store” that is only a Redis dump with no training point-in-time API
- Different SQL in notebook vs service

---

## 5. Model gateway / multi-model router (cheap → expensive)

### Diagram

```text
Request → Gateway
           ├─ classify intent / complexity / risk
           ├─ route: small model | mid | frontier | classical
           ├─ optional escalate on low confidence
           └─ unify response schema + cost ledger
```

### Components

- Router policy (rules, classifier, or LLM-as-router carefully)
- Adapters per provider/model with timeouts
- Cost and latency budgets per tenant/route
- Fallback chain (primary → secondary → template)

### Data / control flow

Route decision logged with reason. Escalation path: if small model confidence < τ or guardrail flags, call larger model. Never silently drop required citations/tools.

### Tradeoffs

| Pros | Cons |
|------|------|
| Large cost savings | Router errors send hard cases to weak models |
| Latency wins on easy traffic | Eval must be **per route**, not blended only |
| Vendor flexibility | Complexity of many SLOs |

### Failure modes

- Optimizer Goodhart: router learns to send everything “easy”
- Cost blowup if escalation rate spikes
- Inconsistent tone/quality across routes (UX trust)

### When it’s the best path

High-QPS assistants, support deflection, any product where 80% of queries are simple.

### Eval + HITL

- **Eval:** stratified golden by route; measure escalation rate, cost/query, quality delta vs always-frontier
- **HITL:** review disagreements between cheap vs expensive on shadow pairs weekly

### Anti-patterns

- Blended dashboard only (hides that “simple” route is failing)
- Router that can call tools the policy forbids for that tier

---

## 6. RAG production architecture (ingest → index → retrieve → generate → cite)

### Diagram

```text
Ingest:  sources → clean/PII scan → chunk → embed → index build
Publish: index_staging → gold smoke → index_prod pointer

Query:   user → ACL filter → retrieve (hybrid) → rerank
         → prompt+context → LLM → cite spans → guardrails → answer
```

### Components

- Corpus versioning + ACL metadata on chunks
- Hybrid retrieval (BM25 + dense) + optional reranker
- Prompt/index/model as a **versioned bundle**
- Citation post-processor (force quotes / reject ungrounded claims)

### Data / control flow

Ingest is a batch (or incremental) pipeline with eval gates. Query path must enforce **document ACLs before** context assembly. Answers carry `index_version`, `prompt_version`, `model_version`.

### Tradeoffs

| Pros | Cons |
|------|------|
| Grounding + updatable knowledge | Retrieval failures → confident nonsense |
| Citations aid trust & HITL | Freshness vs rebuild cost |
| Separates knowledge from weights | ACL bugs are severe |

### Failure modes

- Prompt injection via retrieved docs
- Citation theater (numbers that don’t match spans)
- Index/query embedding model mismatch after “upgrade”

### When it’s the best path

Internal knowledge Q&A, policy assistants, product docs help—when corpus is the source of truth.

### Eval + HITL

- **Eval:** retrieval recall@k, citation entailment, groundedness, refusal quality on out-of-corpus
- **HITL:** sample answers with citation click-through; escalate high-severity topics (legal/HR/medical policy)

### Anti-patterns

- One giant undifferentiated chunk dump with no ACL
- Shipping index rebuilds without golden smoke queries

---

## 7. RAG + tools / agent orchestrator (constrained)

### Diagram

```text
User → Orchestrator
         ├─ plan (bounded steps)
         ├─ retrieve (RAG)
         ├─ tools (allowlist, args schema, authZ)
         ├─ validate observations
         └─ final answer / proposed action
                    ↓
              [policy] → execute or HITL
```

### Components

- Hard **step budget**, tool allowlist, argument schemas
- Separate “propose” vs “execute” for mutating tools
- Sandbox / least-privilege credentials per tool
- Transcript logging for audit

### Data / control flow

Each tool call: authorize → validate args → execute with timeout → sanitize observation → feed back. Loop ends on final answer, budget exhaust, or escalation.

### Tradeoffs

| Pros | Cons |
|------|------|
| Real work beyond chat | Unbounded agents = outages & spend |
| Composable skills | Harder to eval (trajectory quality) |
| HITL fits propose/execute | New injection surfaces via tool outputs |

### Failure modes

- Confused deputy (agent uses user auth to over-fetch)
- Infinite tool loops / cost spirals
- Acting on injected instructions in retrieved text

### When it’s the best path

Ticket triage that **drafts** replies, research assistants with search tools, ops copilots—with **constraints**.

### Eval + HITL

- **Eval:** task success on scripted environments; tool-policy violation rate; step-budget adherence
- **HITL:** **pre-action** approval for money/state changes; post-hoc audit sampling for reads

### Anti-patterns

- “Autonomous” agent with shell + prod credentials
- No separate propose/execute for irreversible tools

---

## 8. Human-in-the-loop review workflow (pre-action / post-hoc)

### Diagram

```text
Pre-action:  model draft → review queue → approve/edit/reject → act
Post-hoc:    model acts (low risk) → sample → audit → labels/feedback
Hybrid:      auto if conf≥τ else pre-action; always post-hoc sample
```

### Components

- Review UI with context, citations, diffs
- SLAs for review age; prioritization by risk/impact
- Label schema feeding eval sets
- Escape hatch: force-human / force-auto feature flags

### Data / control flow

Items enter queue with reason codes (`low_conf`, `policy_keyword`, `random_sample`, `user_escalate`). Decisions write audit events. Rejects become hard negatives.

### Tradeoffs

| Pros | Cons |
|------|------|
| Real risk reduction | Cost & latency; reviewer fatigue |
| Creates gold labels | Inconsistent human quality |
| Stakeholder trust | Queue backlog = product failure |

### Failure modes

- Rubber-stamping under time pressure
- Biased sampling (only easy cases reviewed)
- No closed loop into training/eval

### When it’s the best path

Any medium/high tier: refunds, medical/legal-ish content, external customer email, compliance claims.

### Eval + HITL

- Meta-eval: inter-rater agreement; reviewer accuracy vs adjudicator
- Measure auto vs human disagreement rate as a release signal

### Anti-patterns

- “HITL” with no staffing model or SLA
- Only reviewing random 0.1% of high-severity actions

---

## 9. Shadow / canary / A/B model deployment topology

### Diagram

```text
Traffic ──► Router
             ├─ 100% primary (serve user)
             ├─ shadow: candidate scored, not shown
             ├─ canary: 1→5→25% users see candidate
             └─ A/B: sticky assignment, measure outcomes
Rollback = flip weight to 0 + previous bundle pointer
```

### Components

- Traffic splitter with sticky sessions / user hashing
- Metrics by `variant` + guardrail auto-stop
- Bundle registry (model+prompt+index+config)
- Experiment assignment log for audit

### Data / control flow

Shadow never affects user-visible output but writes paired logs. Canary affects a slice; auto-rollback if error/latency/quality gates trip. A/B needs pre-registered metrics to avoid p-hacking.

### Tradeoffs

| Mode | Strength | Weakness |
|------|----------|----------|
| Shadow | Safe quality compare | Misses UX effects; cost 2× infer |
| Canary | Real UX signal | Residual risk on canary users |
| A/B | Causal outcomes | Needs volume & discipline |

### Failure modes

- Leakage of canary into caches keying wrong variant
- Shadow overload doubles spend / rate limits
- Stopping early on noisy metrics

### When it’s the best path

Any non-trivial promote of models/prompts/indexes. Default release path for serious teams.

### Eval + HITL

- Pre-register primary metric + guardrails; HITL samples **both** variants blindly when subjective quality matters

### Anti-patterns

- “Canary” that is only infra health checks, no quality metric
- A/B without assignment logging (cannot debug or audit)

---

## 10. Event-driven AI (stream → score → act)

### Diagram

```text
Event bus → filter/enrich → score (model)
              → policy → act (notify, block, enqueue ticket)
              → audit log
Dead-letter for poison events; idempotent consumers
```

### Components

- Stream processor (consumer group)
- Feature enrichment with clocks aligned to event time
- Action side effects with idempotency keys
- Lag / watermark monitoring

### Data / control flow

Events processed at-least-once. Actions keyed by `event_id`. Late events: define policy (recompute vs ignore).

### Tradeoffs

| Pros | Cons |
|------|------|
| Fast reaction (fraud, abuse) | Harder debugging than batch |
| Scales with partitions | Ordering & exactly-once myths |
| Composable with queues | Poison events stall partitions |

### Failure modes

- Acting twice on retries
- Feature joins using processing time instead of event time
- Silent consumer lag → “real-time” is hours behind

### When it’s the best path

Fraud/abuse signals, IoT anomaly, personalization triggers, content moderation pipelines.

### Eval + HITL

- Replay eval from recorded topics; HITL on high-impact actions (ban, block payment)

### Anti-patterns

- Blocking human review inside the hot consumer without a side queue
- No lag alerts

---

## 11. Edge / on-device vs cloud hybrid

### Diagram

```text
Device: sensors → tiny model (wakeword / redact / rank)
         → optional cloud call (heavy LLM / big model)
Cloud:  policy, training, aggregated learning
Privacy: prefer on-device PII; send embeddings/redacted text
```

### Components

- On-device runtime (quantized model)
- Sync protocol for model updates with rollback
- Cloud fallback when device confidence low or offline rules
- Bandwidth / battery budgets

### Data / control flow

Default local inference; escalate to cloud with user consent / policy. Model packs versioned; failed update reverts.

### Tradeoffs

| Pros | Cons |
|------|------|
| Privacy, offline, low latency | Quality / size limits |
| Lower cloud cost at scale | Fragmented device fleet |
| Regulatory advantages | Update & eval complexity |

### Failure modes

- Fleet stuck on old vulnerable models
- Hybrid path leaking raw PII “just this once”
- Divergent behavior device vs cloud confusing users

### When it’s the best path

Mobile keyboards, cameras, industrial offline, strong privacy products.

### Eval + HITL

- Device farm golden tests per chip class; HITL on cloud-escalated traces

### Anti-patterns

- Training only on cloud logs while most traffic never leaves device (skew)

---

## 12. LLM gateway with policy, caching, cost controls

### Diagram

```text
Services → LLM Gateway
            ├─ authn/z, tenant quotas, PII redaction
            ├─ prompt template resolve (versioned)
            ├─ semantic/exact cache
            ├─ provider route + retries
            ├─ output filter / schema validate
            └─ cost & trace ledger
```

### Components

- Central policy engine (who can call which model)
- Cache keyed by tenant + template + normalized input (careful with personalization)
- Budget breaker (daily caps)
- Unified hang/timeout behavior

### Data / control flow

All product code talks to gateway, not raw providers. Versions of prompts resolved server-side so clients cannot silently drift.

### Tradeoffs

| Pros | Cons |
|------|------|
| One place for safety/cost | Single point of failure—needs HA |
| Caching saves spend | Bad cache keys → wrong-tenant leaks |
| Easier audits | Latency hop |

### Failure modes

- Cross-tenant cache collision
- Quotas that fail open
- Gateway outage blocks all AI features (need degrade modes)

### When it’s the best path

Any org with >1 AI feature or >1 provider; almost always worth it by medium scale.

### Eval + HITL

- Gateway-level golden probes; cache hit-rate vs quality audits; HITL on policy blocks (false positives)

### Anti-patterns

- Per-service bespoke provider SDKs with no shared logging
- Caching personalized medical/financial answers globally

---

## 13. Ensemble / cascade classifiers

### Diagram

```text
Cascade:  cheap model → if uncertain → mid → if uncertain → expensive / human
Ensemble: models vote / stack; meta-learner combines
```

### Components

- Calibration of probabilities (critical for thresholds)
- Cost-aware threshold policy
- Optional human as final stage
- Per-stage metrics

### Data / control flow

Early exit on high-confidence easy cases. Log stage depth for cost analytics.

### Tradeoffs

| Pros | Cons |
|------|------|
| Precision/cost control | Threshold tuning debt |
| Graceful degrade | Correlated model failures |
| Natural HITL final stage | Latency variance |

### Failure modes

- Poor calibration → wrong early exits
- All models share same biased features
- Cascade depth explosion under drift

### When it’s the best path

Moderation, fraud, intent routing, medical triage **assist** (with human final).

### Eval + HITL

- Cost-quality curves; human stage agreement; slice metrics for rare classes

### Anti-patterns

- Averaging uncalibrated probabilities as if comparable
- No monitoring of stage distribution shift

---

## 14. Offline training → registry → online serving loop (MLOps reference)

### Diagram

```text
Data version → Train/adapt → Eval suite → Registry (Staging)
     → Shadow/Canary → Registry (Prod pointer) → Serve
     → Monitor/drift/feedback → new data version
Rollback: move prod pointer to previous good bundle
```

### Components

- Immutable data/model/prompt/index artifacts
- Registry with stages + approvals
- Automated eval gates in CI/CD
- Monitoring closed loop into backlog

### Data / control flow

Nothing reaches prod without: artifact digest, eval report, approver (human or policy), rollout plan. Lineage links pred → model → data.

### Tradeoffs

| Pros | Cons |
|------|------|
| Reproducibility & rollback | Process overhead |
| Shared language across teams | Can become bureaucracy theater |
| Auditability | Needs cultural buy-in |

### Failure modes

- Registry that only stores files, no eval metadata
- “Prod” mutable bucket overwritten in place
- Feedback not labeled → loop never improves

### When it’s the best path

Default spine for **every** serious classical or LLM product (LLM “train” may mean prompt/index adapt).

### Eval + HITL

- Gates are the product: offline metrics + shadow + human sample rates by tier

### Anti-patterns

- Notebook → scp → prod
- Prompt edits in the provider UI with no version pin

---

## 15. Multi-tenant SaaS AI (isolation, quotas, per-tenant indexes)

### Diagram

```text
Tenant A ─┐
Tenant B ─┼→ Gateway (tenant context) → isolate:
Tenant C ─┘     data, indexes, caches, quotas, keys
                shared compute with hard tenancy walls
```

### Components

- Tenant context on every request (authenticated)
- Per-tenant encryption keys / index namespaces / rate quotas
- Noisy-neighbor controls (fair queues)
- Per-tenant cost attribution & data residency options

### Data / control flow

Retrieval and cache keys **must** include `tenant_id`. Deletes (GDPR) are per-tenant pipelines with verification. Admin break-glass is audited.

### Tradeoffs

| Pros | Cons |
|------|------|
| Leverage shared infra | Isolation bugs are catastrophic |
| Economies of scale | Noisy neighbor & support complexity |
| Productizable AI | Per-tenant eval harder |

### Failure modes

- Vector DB filter forgotten → cross-tenant doc leak
- Cache key without tenant → data bleed
- One tenant’s huge ingest starves others

### When it’s the best path

B2B AI products, per-customer knowledge bases, white-label assistants.

### Eval + HITL

- Continuous cross-tenant isolation tests (red-team); per-tenant quality SLOs; HITL for enterprise-tier content

### Anti-patterns

- “We’ll add tenancy later” on a shared index
- Single global prompt cache across tenants for CRM snippets

---

## Cross-cutting concerns (every pattern)

### Authorization (authZ)

- Authenticate identity; authorize **actions and document access** separately
- For RAG/tools: filter retrieval by ACL **before** LLM sees text
- Confused-deputy checks: tools run as least privilege, not as “the app admin”

### PII & privacy

- Minimize collection; redact at ingest and at log boundaries
- Separate raw stores from training corpora when needed
- Deletion workflows: primary DB + indexes + caches + logs policy
- Regional residency constraints may force pattern 11 or region-pinned 15

### Observability

- Trace id / request id through gateway → retriever → model → tools
- Log **versions** (model, prompt, index, router policy) on every decision
- Metrics: quality, latency, cost, drift, queue depth, review age—not only CPU

### Idempotency

- At-least-once queues and streams presume duplicates
- Side effects keyed by business idempotency key
- LLM “generate” may not be idempotent—store results, don’t blindly re-call on retry if action already taken

### Versioning (prompts / models / indexes)

- Treat `(model, prompt, index, tools_config, router_policy)` as a **bundle**
- Immutable artifacts + mutable pointers (`staging`, `prod`)
- Rollback = pointer move + traffic weight; never “edit prod in place”

### Anti-pattern summary (cross-cutting)

| Smell | Fix |
|-------|-----|
| No request_id | Add everywhere |
| Prompt in source without pin | Registry + gateway resolve |
| Shared cache across tenants | Tenant in key + tests |
| Eval only offline accuracy | Add shadow + cost + safety |
| HITL unstaffed | Capacity plan or lower automation |

---

## How to pick a pattern: decision tree

```text
Must act within <300ms on-device / offline?
  YES → Edge/hybrid (11), maybe tiny cascade (13)
  NO ↓

Knowledge must stay fresh from docs/corpus?
  YES → RAG (6); need tools/actions too? → constrained agent (7)
  NO ↓

Decision mutates money/state or high severity?
  YES → Async (3) + HITL pre-action (8); gateway policies (12)
  NO ↓

User waiting interactively?
  YES → Sync API (2) or streaming; add router (5) if cost hurts
  NO → Batch (1)

Many classical features / entities shared?
  YES → Feature store (4)

Multiple products/providers?
  YES → LLM gateway (12) + MLOps loop (14)

B2B per-customer data?
  YES → Multi-tenant (15) composed with above

Always: release via shadow/canary (9); close the loop (14).
Event-time reactions? → Event-driven (10).
```

### Decision table (constraints → starting pattern)

| Dominant constraint | Start with | Usually compose with |
|---------------------|------------|----------------------|
| Cost at high QPS | Router (5) + cache gateway (12) | Sync (2), canary (9) |
| Human must approve sends | Async (3) + HITL (8) | RAG (6) |
| Strict doc ACL | RAG (6) + tenant (15) | Gateway (12) |
| Fraud milliseconds | Sync (2) or stream (10) + features (4) | Cascade (13) |
| Regulated deletion | Bundle versioning (14) + tenant delete (15) | Gateway logs policy |
| GPU scarce | Batch (1) / async (3) / cascade (13) | Router (5) |
| Stakeholder “ship Friday” | Shadow (9) thin canary—not full rewrite | Eval gates |

**Small-team defaults**

1. Classical scores → **Batch (1)** or **Sync (2)** + simple registry (14)
2. Docs Q&A → **RAG (6)** + **Gateway (12)** + **Canary (9)**
3. Anything that sends email/refunds → **Async (3)** + **HITL (8)**
4. Second AI feature in the company → introduce **Gateway (12)** before chaos

---

## Mapping patterns → risk tier → required eval + human verification

Risk tiers (from Lesson 10.1): **Low** (cosmetic), **Medium** (user-facing advice), **High** (money, safety, rights, irreversible).

| Pattern | Low tier | Medium tier | High tier |
|---------|----------|-------------|-----------|
| Batch (1) | Auto-publish after metric gates | Staging + 1% HITL sample | Staging + stratified HITL + dual review on sensitive slices |
| Sync API (2) | Golden CI + error budget | + shadow + thumbs-down review | + confidence gating; block/defer high-severity intents to HITL |
| Async (3) | Auto-complete | Optional review on flags | **Pre-action HITL default** for side effects |
| Feature store (4) | Skew tests | + slice metrics | + human appeal path feeding labels |
| Router (5) | Cost dashboards | Per-route golden | Human review of escalation disagreements |
| RAG (6) | Smoke queries | Groundedness + citation eval | ACL red-team; topic holdouts; expert review |
| Agent (7) | Sandbox only | Propose-only in prod | Pre-action approve; tool allowlist; step budget |
| HITL (8) | Light audit | Staffed queue + SLA | Dual control; adjudicator; meta-metrics |
| Shadow/canary (9) | Infra canary | Quality gates | Blind human preference on paired outputs |
| Event-driven (10) | Lag monitors | Action rate limits | HITL / manual ack for bans & blocks |
| Edge hybrid (11) | Device tests | Fleet canary packs | Privacy review; cloud escalate audited |
| LLM gateway (12) | Quotas | Policy probes | DLP audits; break-glass reviews |
| Cascade (13) | Threshold dashboards | Calibration checks | Human final stage for positive class |
| MLOps loop (14) | Artifact pins | Eval report required | Human approve promote; signed lineage |
| Multi-tenant (15) | Quota tests | Isolation integration tests | Continuous cross-tenant red-team + contracts |

### Minimum bar before “we’re in production”

1. Named pattern(s) and why not the alternatives  
2. Bundle versions + rollback pointer test  
3. Eval gates matching risk tier  
4. HITL plan with **capacity numbers**, not vibes  
5. Observability: versions on every decision, cost, quality  
6. Idempotency story for any side effect  

---

## Worked mini-examples (pattern composition)

### A. Nightly churn emails

**Patterns:** Batch (1) + MLOps (14) + light HITL (8)  
**Why not sync?** No user waiting; email is naturally delayed.  
**Eval:** AUC on labeled churn; stability vs last week; HITL on top-decile “sure churn” messages tone.

### B. B2B policy assistant

**Patterns:** Multi-tenant RAG (15+6) + Gateway (12) + Canary (9) + post-hoc HITL  
**Why not free agent?** High ACL risk; tools limited to “cite-only” until later.  
**Eval:** per-tenant smoke; groundedness; isolation tests; expert review on HR/legal tags.

### C. Support agent that can refund

**Patterns:** Constrained agent (7) + Async (3) + pre-action HITL (8) + Gateway (12)  
**Why not sync auto-refund?** Irreversible money movement.  
**Eval:** tool-policy violations in staging; refund accuracy golden; reviewer SLA.

### D. Cost crisis on chatbot

**Patterns:** Router (5) + Gateway cache (12) + Cascade intent (13)  
**Why not only “buy more credits”?** Need architectural rate of spend control.  
**Eval:** quality parity on hard slice; escalation rate; weekly human pairs cheap vs expensive.

---

## Mini practices

1. Pick a Stage 12 project idea. Name **two** patterns you will compose and **one** you explicitly reject; write the rejecting reason as a tradeoff row.  
2. For pattern 6 (RAG), list three failure modes and the monitor or HITL hook that catches each.  
3. Draw pattern 9 for a prompt-only change (no model weight change)—what still belongs in the bundle?  
4. Tenant T asks for GDPR delete. Which components in patterns 6, 12, and 15 must prove deletion?  
5. Stakeholder wants “fully autonomous ops agent” Friday. Which patterns and gates do you insist on before any prod credential?

## Checkpoint

You should now be able to whiteboard any of the 15 patterns, compose 2–3 for a product, attach risk-tiered eval + HITL, and explain cross-cutting authZ/PII/idempotency/versioning without hiding behind a vendor logo.

**Next:** [05-scenario-based-prod-ai-questions.md](../../12-capstones-and-interviews/learn/05-scenario-based-prod-ai-questions.md) — grind high-difficulty scenarios. Then return to [07-exercises-and-checklist.md](../../10-production/learn/07-exercises-and-checklist.md) for the design-doc capstone with richer pattern vocabulary.
