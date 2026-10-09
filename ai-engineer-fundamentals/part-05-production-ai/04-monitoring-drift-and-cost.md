# Lesson 04 — Monitoring, Drift, and Cost

## Why this lesson exists

Silent failure is the default mode of production AI: the HTTP 200 still returns, the sentence still sounds fluent, and the metric you watched in training no longer matches user harm. **Monitoring** is how you notice. **Drift** detection tells you the world moved. **Cost** controls keep the feature alive commercially. **Human review queues** catch what dashboards cannot.

This lesson covers logging (classical vs LLM), quality monitoring options, drift types (including RAG index staleness), SLOs and alert design, dashboards vs traces, and how HITL queues connect to low confidence and thumbs-down.

## Learning goals

- Design redacted, useful logs for classical and LLM/tool calls
- Compare offline gold regressions vs online feedback vs LLM-as-judge
- Detect data drift, concept drift, and embedding/index staleness
- Set latency/cost SLOs and alerts without fatigue
- Choose dashboards vs traces for different questions
- Operate human review queues tied to risk and signals

---

## 1. What “healthy” means in production

```text
Infra healthy  ≠  Model healthy  ≠  Product healthy

200 OK + low latency  +  fluent text  +  users still escalate angry
        ▲                     ▲                    ▲
   metrics/traces        may look fine         need quality + HITL
```

You need **three planes**:

| Plane | Examples |
|-------|----------|
| Reliability | error rate, timeouts, dependency health |
| Quality | gold regressions, thumbs-down, human grades |
| Efficiency | $/request, tokens, cache hit rate, p95 latency |

---

## 2. What to log

### Shared fields (always)

| Field | Why |
|-------|-----|
| `request_id` / `trace_id` | Join across services |
| `timestamp` | Order events |
| `caller_id` / `tenant` | Slice and abuse detection |
| `endpoint` / `feature_name` | Multi-feature services |
| `latency_ms` | SLO |
| `status` / `error_class` | Reliability |
| `version_bundle` | Reproduce (model/prompt/index/code) |

### Classical model extras

| Field | Why |
|-------|-----|
| Feature vector hash or sampled features | Skew debug (careful with PII) |
| `prediction` / `scores` | Audit |
| `model_version` | Registry link |

### LLM / RAG / tools extras

| Field | Why |
|-------|-----|
| `model_id`, `prompt_version`, `index_version` | Reproduce |
| Token counts in/out | Cost |
| `retrieval_ids` (+ scores) | RAG debug |
| Tool names + **redacted** args | Agent debug |
| Schema validation ok? | Quality proxy |
| Guardrail hit flags | Safety |
| `cached` boolean | Cost/quality interpretation |

### Redaction rules of thumb

```text
DO NOT log raw: passwords, API keys, full payment data, auth tokens
OFTEN REDACT: emails, phones, free-text that may contain PII
MAY LOG: document ids, non-sensitive categorical features, error codes
WHEN IN DOUBT: hash or drop; keep a secure break-glass store with short TTL if needed for incidents
```

```python
import re

EMAIL = re.compile(r"\b[\w.+-]+@[\w-]+\.[\w.-]+\b")
PHONE = re.compile(r"\b(?:\+?1[-.\s]?)?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}\b")

def redact(text: str) -> str:
    text = EMAIL.sub("[EMAIL]", text)
    text = PHONE.sub("[PHONE]", text)
    return text
```

Redaction is imperfect—combine with retention limits and access control (Lesson 05).

### Mini practice

Draft a JSON log record for a RAG chat turn including versions, retrieval ids, tokens, latency, and a redaction note. Omit raw user text or show it redacted.

---

## 3. Quality monitoring: three families

### Option A — Offline golden regressions

Nightly/CI job runs the gold set against the **production bundle** (or staging candidate).

| Pros | Cons |
|------|------|
| Stable, comparable | Misses live distribution |
| Catches prompt/index foot-guns | Costs tokens if large |
| Good ship gate | Goes stale if gold not curated |

### Option B — Online user feedback

Thumbs, edits, “contact human,” CS tags, retry rates.

| Pros | Cons |
|------|------|
| Real user signal | Sparse / biased (angry users over-index) |
| Cheap to collect | Needs volume |
| Direct product link | Not a root-cause |

### Option C — LLM-as-judge (automatic)

A model scores outputs with a rubric.

| Pros | Cons |
|------|------|
| Scales grading | **Misleading** when judge shares biases with generator |
| Fast iteration | Gaming / verbosity bias |
| Useful as smoke | Weak on factuality without evidence |

### Tradeoffs and when to trust what

| Need | Prefer | HITL role |
|------|--------|-----------|
| Block bad promote | Offline gold (+ safety set) | Grade ambiguous failures |
| Detect live UX pain | Online feedback | Review thumbs-down 100% at low volume |
| Cheap continuous score | LLM-judge **with** caveats | Calibrate judge vs humans monthly |
| Regulated / high tier | Human grades primary | Judge only as triage |

**Heuristic:** LLM-as-judge is a **smoke alarm**, not a court of law. If judge and humans disagree on a stratified sample, trust humans and fix the rubric/judge—or stop using the judge for gates.

### Combined quality loop

```text
gold regressions (CI/nightly)
        +
online feedback + escalation rate
        +
optional judge on sample
        +
HITL queue on triggers
        →
incident or gold-set update
```

---

## 4. Drift

### Data drift

Input distribution changes (new region, new device, seasonal behavior). Model may still be “valid” but features shift.

**Signals:** PSI/KL on feature histograms; embedding population stats; retrieval query length changes.

### Concept drift

Relationship between inputs and target changes (fraudsters adapt; policy meaning shifts).

**Signals:** online labels (when available) degrade; human auditors disagree with model more; campaign lift falls.

### Embedding / index staleness (RAG)

Docs change but index does not; or embedding model changes mid-flight; or crawl misses new policies.

**Signals:** rising empty-retrieval rate; falling citation click satisfaction; gold retrieval recall drop; doc `updated_at` vs index `built_at` lag.

### Tradeoffs among drift detectors

| Method | Cost | Sensitivity | False alarms |
|--------|------|-------------|--------------|
| Simple traffic/feature means | Low | Low–medium | Medium |
| PSI on top features | Low | Medium | Needs thresholds |
| Classifier “train vs live” | Medium | Medium | Needs care |
| Gold retrieval recall monitor | Medium | High for RAG | Gold must stay fresh |
| Human audit trend | High | High on meaning | Slow |

### How to choose

**Small-team default:** monitor a handful of input stats + gold suite + empty-retrieval % + thumbs-down. Add PSI when you have stable feature schemas. Do not build a drift platform before you have gold + logging.

### Eval + HITL on drift

- Automatic alert → open review task with 20–50 live samples
- Humans label “still correct?” to distinguish data drift (recalibrate/features) vs concept drift (retrain/relabel) vs product change
- For RAG: human compares answer to **current** source doc, not only index chunk

---

## 5. Cost and latency SLOs

### Latency

Define SLOs per endpoint, e.g.:

```text
Classical /predict:  p95 < 200 ms
RAG /chat:           p95 < 5 s; time-to-first-token p95 < 1.5 s
Batch job:           finish before 06:00 America/New_York
```

Track **p50/p95/p99**, not only averages. Averages hide tail pain.

### Cost

```text
daily_cost ≈ Σ (tokens_in * price_in + tokens_out * price_out)
           + retrieval infra + host costs
```

Budgets per environment (dev/staging/prod) and per tenant if multi-tenant.

### Alerting without fatigue

| Bad alert | Better |
|-----------|--------|
| Page on every 5xx blip | Page on error rate > 2% for 5 minutes |
| Alert on mean latency | Alert on p95 SLO burn |
| Daily cost +1% noise | Alert at 80% and 100% of budget; anomaly vs 7-day baseline |
| Judge score −0.01 | Alert on severe human findings or gold gate fail |

**Heuristic:** pages should be rare and actionable. Everything else is ticket/dashboard.

### Levers when over SLO / budget

- Route easy traffic cheaper (Lesson 03)
- Cap `max_tokens`; shrink retrieval `k`
- Cache safe layers
- Batch offline work
- Degrade gracefully (Lesson 06): smaller model, retrieval-only, canned apology + ticket id

---

## 6. Dashboards vs traces

### Dashboards (aggregates)

Good for: “Is error rate up?” “Did cost double?” “Empty retrieval % this week?”

### Traces (single request path)

Good for: “Why did request `abc` retrieve these chunks and call that tool?”

| | Dashboards | Traces |
|--|------------|--------|
| Question type | Trends, SLOs | Root cause one case |
| Cost to store | Low–medium | Higher (sample or short TTL) |
| Privacy risk | Aggregates safer | Full payloads risky |
| HITL link | Queue volume widgets | Deep links into reviewed turns |

**Small-team default:** one dashboard with reliability + cost + quality proxies; sample traces (e.g. 1–5% or all errors) with redaction.

### Mini practice

Name five panels for a RAG bot dashboard and one trace field list you would need to debug a wrong citation.

---

## 7. Human review queues

Monitoring without a place for humans to **act** becomes wallpaper.

### Trigger sources

| Trigger | Typical action |
|---------|----------------|
| User thumbs-down | Review + gold candidate |
| Low model/retriever confidence | Optional delay + review (async UX) or escalate |
| Guardrail soft-flag | Human decide allow/refuse |
| Judge low score | Triage before trusting auto metric |
| Random sample by tier | Continuous audit |
| Drift alert | Stratified live sample |
| New canary traffic | Higher sample rate |

### Queue UX essentials

```text
show: user ask (redacted if needed), retrieval snippets, model output, versions
actions: approve, edit & approve, reject, escalate, add_to_gold
sla: medium tier ≤ 1 business day; high tier irreversible ≤ minutes with on-call
```

### Deciding sample rates (connects to Lesson 01 / 05)

```text
rate ≈ clip(
  base_tier_rate
  + boost_if_canary
  + boost_if_drift_alert
  + 100%_if_thumbs_down_or_high_risk_action,
  max_affordable_reviews_per_day / traffic
)
```

If review capacity is fixed at 30/day and traffic is 3,000/day, you cannot do 5% blindly—use **stratified + triggered** sampling: all thumbs-down, all empty-retrieval, all canary, plus random fill.

### When automated metrics are enough to skip review

- Exact-match extraction with schema ok and no safety surface
- Classical scores consumed only by internal batch with separate business QA

### When humans must review

- Customer-visible generative answers (ongoing sample)
- Any disagreement between judge and cheap heuristics
- First N days after prompt/model/index change (elevated rate)
- High-tier actions (often pre-action, not only post-hoc)

---

## 8. Online evaluation design

### Shadow scoring

Log candidate answers; score offline later with rubrics/humans.

### Interleaving / A/B

Careful with user experience ethics; prefer for low tier. Measure deflection, CSAT, escalation—not only token metrics.

### Feedback → gold

```text
thumbs-down cluster → weekly triage → add 5–10 gold items → prevent recurrence
```

Without this loop, gold sets rot and monitors lie.

---

## 9. Worked example: monitors for support RAG

| Monitor | Type | Alert |
|---------|------|-------|
| 5xx / provider errors | Reliability | Page > 2% / 5 min |
| p95 latency | SLO | Ticket > 5s for 15 min |
| Daily token spend | Cost | Page at 100% budget |
| Empty retrieval % | Quality proxy | Ticket if +50% vs baseline |
| Thumbs-down rate | Quality | Ticket if doubles week over week |
| Nightly gold pass | Regression | Block + page on fail |
| HITL severe count | Safety/quality | Page on any severe |

**HITL plan:** 100% thumbs-down; 2% random; 10% of canary; weekly 30-min review with support lead.

---

## 10. Classical scoring example: monitors

| Monitor | Why |
|---------|-----|
| Score distribution shift | Data drift proxy |
| Null feature rate | Pipeline break |
| Batch job duration | Freshness SLO |
| Downstream lift / conversion | Concept drift proxy |
| Slice metrics monthly | Fairness / regional issues |

HITL: analysts sample high-score accounts monthly; compare to CRM outcomes when labels arrive.

---

## 11. Anti-patterns

| Anti-pattern | Fix |
|--------------|-----|
| Logging full prompts forever | Redact + retention TTL |
| Only watching GPU/CPU | Add quality + cost planes |
| Trusting LLM-judge gates alone | Calibrate with humans; gold primary |
| Alerts for every blip | SLO burn + multi-window |
| No owner on dashboard | Name on-call in panel footer |
| Gold set never updated | Feedback → gold ritual |

---

## 12. Putting it together: weekly operating rhythm

```text
Daily:  check burn alerts; drain HITL high-pri queue
Weekly: gold trend; cost vs budget; thumbs-down themes; add gold items
Monthly: drift review; judge-vs-human calibration; slice metrics
Per release: elevated sample + shadow/canary report before full ramp
```

---



---

## 13. Worked example: choosing quality monitors for a tabular + LLM report

Imagine a weekly job that scores churn risk (classical) and drafts an account summary (LLM) for CSMs.

### Options for monitoring the LLM summary

| Option | What you measure | Tradeoff |
|--------|------------------|----------|
| A. None beyond job success | Exit code | Blind to fluent nonsense |
| B. Schema + length checks | JSON fields present | Misses factual errors |
| C. Gold accounts weekly | Rubric on 30 fixed accounts | Strong regression signal; misses new accounts |
| D. LLM-as-judge on all | Scalar score | Cheap but misleading without calibration |
| E. HITL on random 10% + all low-confidence | Human grades | Best truth; capacity-bound |

**Small-team default:** B + C + E (10% or max 25 reviews/week) ; use D only to *prioritize* the HITL queue after calibrating on 50 human-graded examples.

### How eval + HITL inform the classical score side

- Automatic: PSI on top features; batch lift vs holdout when labels arrive lagged
- Humans: CSM “does this prioritization feel right?” monthly panel of 20 accounts
- If humans and model disagree systematically on a segment → concept drift investigation, not only threshold tweak

---

## 14. Alert taxonomy (so pages mean something)

| Severity | Example | Channel | Response expectation |
|----------|---------|---------|----------------------|
| P1 | Error rate > 5% for 5 min; safety severe ≥ 1 | Page on-call | Immediate |
| P2 | Gold nightly fail; budget 100%; p95 SLO burn | Page or urgent ticket | Same day |
| P3 | Thumbs-down +30% WoW; empty retrieval +50% | Ticket | This week |
| P4 | Cache hit rate drift; judge score soft dip | Dashboard only | Backlog |

Tie each P1/P2 to a runbook link (Lesson 06). If an alert has no runbook, it will be ignored or cause thrash—either write the runbook or demote the alert.

---

## 15. Privacy-preserving analytics

Aggregates still leak if slices are tiny.

| Practice | Why |
|----------|-----|
| Minimum cohort sizes for dashboards | Avoid re-identifying one user |
| Hash subject ids in analytics export | Limit join keys |
| Separate “break glass” raw store | Incident debug without daily access |
| Retention: traces 7–30 days typical starting point | Align with policy |

HITL tools often need more raw text than dashboards—apply stricter ACLs and shorter retention on review UIs.

## What is next

**[05-safety-privacy-and-hitl.md](05-safety-privacy-and-hitl.md)** — PII, prompt injection, guardrails, and deep HITL patterns: rates, escalation, and safety eval.
