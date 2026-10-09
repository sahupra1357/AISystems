# Lesson 10.6 — Reliability and Incident Response

## Why this lesson exists

AI features inherit every failure mode of distributed systems—plus fuzzy quality failures that look like success in HTTP terms. Reliability engineering (timeouts, retries, circuit breakers, fallbacks) keeps the product up. **Incident response** and **rollback of prompts/indexes/models** keep quality incidents from becoming week-long mysteries. Postmortems should ask: *what eval or HITL gap let this through?*

## Learning goals

- Apply timeouts, retries, idempotency, circuit breakers, and fallbacks with tradeoffs
- Design graceful degradation paths
- Separate quality incidents from infra incidents in on-call/runbooks
- Version and roll back prompts, indexes, and models
- Write postmortems that improve eval and HITL—not only restart servers

---

## 1. Failure modes unique to AI services

| Failure | Looks like | User impact |
|---------|------------|-------------|
| Provider timeout | Slow spinner / 503 | Abandonment |
| Provider 429 rate limit | Intermittent errors | Flaky UX |
| Invalid JSON / schema | 200 with broken client | Silent product bugs |
| Hallucinated answer | 200 fluent text | Trust damage |
| Empty / wrong retrieval | Confident wrong FAQ | Wrong actions |
| Cost runaway | Bill alert | Feature turned off hastily |
| Bad prompt promote | Sudden quality cliff | Support surge |

Reliability patterns below address the left column; eval/HITL (Lessons 10.1, 10.4 and 10.5) address the middle quality rows.

---

## 2. Timeouts

**Every** outbound call needs a timeout: LLM provider, vector DB, tools, feature store.

### Options

| Approach | Description |
|----------|-------------|
| One global timeout | Simple; crude |
| Budgeted per stage | retrieve 300ms, LLM 8s, tools 2s |
| Deadline propagation | Remaining time passed downstream |

### Tradeoffs

| | Short timeouts | Long timeouts |
|--|----------------|---------------|
| UX | Fail fast | Wait longer |
| False failures | More | Fewer |
| Resource holds | Fewer | Threads/workers stuck |

**Heuristic:** set LLM timeouts from p99 of healthy traffic + margin, not from wishful p50. Align API gateway timeout **above** internal budgets or you will 504 while the worker still runs.

```python
# Pseudocode
try:
    return call_provider(prompt, timeout=10.0)
except TimeoutError:
    metrics.increment("llm_timeout")
    return fallback_response(request_id)
```

---

## 3. Retries

### When to retry

| Retry | Do not blindly retry |
|-------|----------------------|
| 429 / 503 / connection reset | 400 validation errors |
| Idempotent GETs | Non-idempotent charges without key |
| Transient timeouts | Persistent 401 (fix auth) |

### Options

1. No retries  
2. Fixed 2–3 retries  
3. Exponential backoff + jitter  
4. Retry on another model/provider (failover)

### Tradeoffs

| | No retry | Aggressive retry |
|--|----------|------------------|
| User success on blips | Worse | Better |
| Avalanche on outage | Safer | Can worsen outages |
| Cost | Lower | Duplicate spend |
| Duplicate side effects | — | Dangerous without idempotency |

**Small-team default:** 2–3 retries with exponential backoff + jitter on idempotent LLM reads; **no** automatic retry on tool writes unless idempotency keys exist.

---

## 4. Idempotency

For creates/charges/emails:

```text
Client sends Idempotency-Key: uuid
Server stores key → result
Replay returns same result without re-doing side effect
```

AI agents that “try again” after timeout are a duplicate-refund machine without this.

### HITL note

If a human approves a refund and the worker times out, the retry must not double-pay—idempotency is a safety control, not only an infra nicety.

---

## 5. Circuit breakers

When a dependency is failing, **stop calling it** for a cool-down; fail fast to fallback.

```text
Closed (normal) → open after N failures → half-open probe → closed if healthy
```

### Tradeoffs

| Without breaker | With breaker |
|-----------------|--------------|
| Pile threads on dead provider | Fast fail |
| Cascading latency | Localized pain |
| — | Need good fallback UX |

**Eval:** measure fallback quality—not only “breaker opened.” A breaker that serves nonsense still pages humans via quality monitors.

---

## 6. Fallbacks and graceful degradation

### Fallback options (LLM feature)

| Fallback | Quality | Cost | When |
|----------|---------|------|------|
| Cached prior answer | Medium if fresh | Free | Repeat questions |
| Smaller / cheaper model | Lower | Lower | Provider down or budget |
| Retrieval-only snippets | Medium for FAQ | Low | Generator down |
| Template apology + ticket id | Low | Free | Last resort |
| Keyword search FAQ | Medium | Low | RAG stack failing |
| Human handoff queue | High | High | Medium+ tier |

### Tradeoff table

| Path | User trust | Eng complexity | Risk |
|------|------------|----------------|------|
| Smaller model | Usually OK | Low | Silent quality drop—monitor |
| Retrieval only | OK if cited | Low | No synthesis |
| Static apology | Honest | Tiny | High abandon |
| Auto human handoff | Best | Needs staffing | HITL capacity |

### How to choose

```text
Prefer degradation that preserves truthfulness over fluent guessing.
Announce reduced mode when possible ("showing search results only").
Never degrade by disabling authz or ACL.
```

**Small-team default ladder:** retry → smaller model → retrieval-only with citations → apology + ticket id → disable feature flag.

### Eval + HITL

- Gold set should include “provider down” simulation expectations
- Sample fallback responses in HITL—they often sound robotic or overshare ticket internals
- Alert when fallback rate > X% (hidden outage)

---

## 7. Rate limiting and load shedding

Protect yourself and upstream:

- Per-tenant and per-key limits
- Global shed when queue depth or CPU exceeds threshold
- Prefer shedding **new** low-tier traffic before dropping high-tier authenticated UX

Tradeoff: harsh limits anger users; no limits take down everyone.

---

## 8. On-call and runbooks for AI features

### Two incident classes

| Class | Examples | First checks |
|-------|----------|--------------|
| **Infra** | 5xx, timeouts, dependency down | Dashboards, provider status, deploy diff |
| **Quality** | Thumbs-down spike, hallucination reports, weird tool calls | Version bundle, gold job, HITL queue, recent prompt/index change |

Page on-call for both—but playbooks differ. A quality incident rarely needs “restart the pod” as step one.

### Runbook skeleton

```text
Title: Support RAG — elevated thumbs-down
Owner: ...
Severity guide: ...
Symptoms: ...
Immediate actions:
  1. Confirm metric vs seasonal baseline
  2. Check last prompt/index/model change timestamp
  3. Sample 10 failing traces
  4. If clear regression → flip flag to previous bundle
  5. If injection/abuse → enable kill / tighten guardrail
  6. Notify support lead; elevate HITL sample to 100% temporarily
Comms template: ...
Rollback steps: ...
Follow-ups: add gold items; postmortem
```

### Mini practice

Write a half-page runbook for “daily LLM cost > 150% budget.”

---

## 9. Rollback: version everything

### What to version as a bundle

```text
code_git_sha
model_id or weights_hash
prompt_version
index_version / corpus_snapshot
tool_schema_version
guardrail_config_version
```

Coupled pieces roll back **together**. Rolling back the prompt but keeping a bad index often leaves the fire burning.

### Rollback options

| Mechanism | Speed | Risk |
|-----------|-------|------|
| Feature flag to previous bundle | Fastest | Need flags wired |
| Redeploy previous artifact | Fast | CI/CD dependency |
| Blue-green flip | Fast | Dual infra |
| Re-build index from snapshot | Slow | Needed for corpus poison |

### Tradeoffs

Flags are the small-team friend. Practice rollback in staging monthly.

### Eval after rollback

Confirm gold job green on restored bundle; keep elevated HITL until stable.

---

## 10. Postmortems focused on eval gaps

Classic SRE postmortems ask about monitoring and deploys. AI postmortems add:

```text
What failed in offline gold that should have caught this?
Was the failure mode absent from the gold set?
Did LLM-as-judge disagree with humans?
Was HITL sample rate too low for this risk?
Did we promote without shadow/canary?
Was a tool side effect possible in shadow? (should never be)
What new red-team or gold items did we add?
```

### Blameless + concrete

Focus on system gaps. Action items should change gates, not only “be more careful.”

| Weak action | Stronger action |
|-------------|-----------------|
| “Review prompts better” | “Add 15 gold cases from this incident; CI gate on rubric ≥ 0.8” |
| “Tell people not to jailbreak” | “Server-side cap + HITL on refund tool” |

---

## 11. Worked example: provider outage

```text
15:00  p95 latency climbs; timeouts rise
15:05  circuit breaker opens on provider A
15:05  fallback: smaller model B + retrieval citations
15:10  fallback rate 40% — page quality + infra
15:20  status page note; HITL samples fallback answers
16:00  provider A healthy; breaker half-open → closed
16:30  gold job on A vs B compare; keep B as hot standby
Next day postmortem: add multi-provider failover runbook; budget for dual-model shadow monthly
```

### Tradeoffs in that response

| Choice | Why |
|--------|-----|
| Smaller model not static apology | Preserve partial UX |
| Elevate HITL | Fallback quality unknown |
| Not disabling ACL under pressure | Safety > uptime theater |

---

## 12. Worked example: bad prompt promote

```text
Tuesday 10:00  prompt v18 canary 5%
Tuesday 10:30  thumbs-down 3× on canary cohort; gold still green (gap!)
Tuesday 10:40  halt ramp; flag back to v17
Tuesday 11:00  HITL finds v18 over-confident without citations
Tuesday 14:00  add gold items requiring citations; fix prompt; re-shadow
```

**Lesson:** gold green + canary red means **gold coverage gap**. Fix gold before re-promoting.

---

## 13. Reliability checklist

- [ ] Timeouts on all dependencies; gateway aligned
- [ ] Retries with jitter; no unsafe write retries
- [ ] Idempotency keys for side-effecting tools
- [ ] Circuit breaker + tested fallback ladder
- [ ] Rate limits / load shedding
- [ ] Versioned bundles + one-step rollback flag
- [ ] Runbooks for infra **and** quality
- [ ] On-call knows kill switches (Lesson 10.5)
- [ ] Postmortem template includes eval/HITL questions
- [ ] Game-day practiced at least once pre-launch for medium+

---

## 14. Anti-patterns

| Anti-pattern | Fix |
|--------------|-----|
| Infinite retries on LLM writes | Idempotency + capped retries |
| Fallback = invent answer without retrieval | Prefer abstain / retrieval-only |
| Rollback = “find the old prompt in Slack” | Immutable version registry |
| Only infra on-call | Quality runbooks + owners |
| Postmortem without new gold | Mandatory gold/red-team additions |

---



---

## 15. Dependency map (know what you call)

Before incidents, draw:

```text
API ─┬─ Auth service
     ├─ Feature store / DB
     ├─ Vector index
     ├─ LLM provider A (primary)
     ├─ LLM provider B (optional failover)
     └─ Tool: billing API
```

For each edge: timeout, retry policy, breaker, fallback, owner, status page.

### Tradeoffs: single provider vs multi-provider

| | Single | Dual |
|--|--------|------|
| Complexity | Lower | Higher (output drift) |
| Outage resilience | Weaker | Stronger |
| Eval burden | One gold baseline | Must compare A vs B on gold + humans |
| Cost | Simpler contracts | Two bills; idle capacity |

**Small-team default:** single provider + strong fallback ladder (smaller model / retrieval-only). Add failover when downtime cost exceeds eng cost—and budget blind human compares when switching.

---

## 16. Idempotency + HITL interaction (detailed)

```text
t0  model proposes refund; HITL approves
t1  worker calls billing; timeout (unknown success)
t2  naive retry → double charge 💥
```

**Correct patterns (options):**

1. Idempotency key = `approval_id` shared across retries  
2. Poll billing “refund status by key” before re-POST  
3. HITL approval creates a durable `RefundIntent` row; worker only transitions `PENDING→SUCCEEDED|FAILED`

| Pattern | Complexity | Safety |
|---------|------------|--------|
| 1 | Low | High if billing supports keys |
| 2 | Medium | High |
| 3 | Higher | Highest for audit |

Choose 3 for high-tier money movement when you can afford the schema.

---

## 17. Quality vs infra: decision tree for first 15 minutes

```text
Alert fires
  │
  ├─ Error/timeout/dependency? ──► infra runbook (status, deploy, breaker)
  │
  └─ Mostly 200s but quality signal?
           │
           ├─ Recent bundle change? ──► rollback flag first, ask questions later
           ├─ Single tenant / abuse? ──► quarantine tenant; keep global up
           └─ Unknown ──► sample 10 traces + HITL severe check; then decide
```

**Heuristic:** if you are unsure whether it is quality or infra, **preserve rollback ability** (do not keep pushing new prompts while debugging).

## What is next

**[07-exercises-and-checklist.md](07-exercises-and-checklist.md)** — exercises per lesson, production design-doc capstone, and Ready-for-Stage-11 checklist.
