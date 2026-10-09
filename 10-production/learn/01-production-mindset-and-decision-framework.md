# Lesson 10.1 — Production Mindset and Decision Framework

## Why this lesson exists

Most AI production failures are not “wrong algorithm” failures. They are **decision** failures: shipping a notebook path that cannot roll back; choosing an online LLM for a nightly report; skipping human review on a high-stakes action; trusting a single offline metric that does not match user harm.

This lesson gives you a **repeatable decision framework**, worked examples, and risk-tiered verification habits. Later lessons deepen pipelines, serving, monitoring, safety, and reliability—but they all hang on the mindset here.

## Learning goals

- Name the notebook → production gap in concrete failure modes
- Apply: problem → constraints → options → tradeoffs → eval → HITL → ship → rollback
- Choose batch vs online, managed LLM API vs self-host, build vs buy vector DB with eyes open
- Combine golden-set eval, shadow traffic, and human review before promote
- Assign risk tiers and required verification levels

---

## 1. The notebook vs production gap

### What notebooks optimize for

Notebooks optimize for **exploration speed**: plot quickly, mutate data in place, hard-code paths, leave cells out of order, keep secrets in env cells. That is fine for learning and research spikes.

### What production optimizes for

Production optimizes for **operability**:

| Concern | Notebook habit | Production need |
|---------|----------------|-----------------|
| Reproducibility | “It worked on my laptop” | Same git SHA + data version → same metric |
| Latency | Ignore wall clock | p95/p99 SLOs |
| Failure | Rerun the cell | Timeouts, retries, fallbacks, user-safe errors |
| Change | Overwrite the CSV | Versioned artifacts + rollback |
| Privacy | Print the dataframe | Redaction, retention, access control |
| Quality | One accuracy number | Golden regressions + online feedback + HITL |
| Cost | Free credits vibe | Budgets and alerts |
| Ownership | Nobody | On-call and runbooks |

### Classic failure story

```text
1. Great notebook F1 on a local CSV
2. Unknown which CSV export (marketing renamed columns last Tuesday)
3. Hyperparameters only in a chat thread
4. Model pickled from a different sklearn minor version
5. Deployed; silent drop in precision on a new region
6. No way to roll back to “the good week”
```

MLOps and production design exist to interrupt that story.

### Mini practice

List three things in your last AI notebook that would break if a teammate ran it next month on a clean machine.

---

## 2. Decision framework (use this every time)

Print this on a sticky note. Every major Stage 10 decision should leave a short memo following it.

```text
1. Problem      — Who is harmed/helped? What job is the model doing?
2. Constraints   — Latency, cost, privacy, compliance, team skill, deadline
3. Options       — At least 2–3 real alternatives (not strawmen)
4. Tradeoffs     — Cost · latency · quality · complexity · risk · skill fit
5. Eval plan     — Offline golden set / metrics / pass bars
6. HITL plan     — Who reviews what, sample rate, escalation
7. Ship criteria — What must be true to promote (gates)
8. Rollback      — Exact previous artifact versions + kill switch
```

### Why each step matters

- Skipping **options** locks you into the first idea you saw on a blog.
- Skipping **tradeoffs** means you cannot defend the choice when cost or latency bites.
- Skipping **eval** means you promote vibes.
- Skipping **HITL** means automated metrics become the sole judge of human-facing harm.
- Skipping **rollback** means every bad release is an incident without an exit.

### Template (fill in one page)

```text
Feature:
Problem / user:
Risk tier (low/med/high):
Constraints:
  latency:
  cost ceiling:
  privacy:
  team:
Options considered:
  A)
  B)
  C)
Chosen: __ because __
Eval plan:
  golden set size / metrics / bars:
  shadow / canary:
HITL plan:
  pre-action / post-hoc / sample rate:
Ship criteria:
Rollback:
```

---

## 3. Risk tiers and verification level

Not every feature needs the same ceremony. **Match verification to stakes.**

| Tier | Examples | Typical harm if wrong | Required verification (baseline) |
|------|----------|------------------------|----------------------------------|
| **Low** | Internal FAQ draft; autocomplete suggestions users can edit | Mild annoyance | Golden set smoke; optional 1–2% human sample |
| **Medium** | Customer support answers shown to users; ranking in product | Trust loss, support load, mild financial | Golden regressions; shadow or canary; weekly audit sample; clear fallback |
| **High** | Refunds/charges; medical/legal/employment/credit-adjacent; irreversible tool actions | Legal, safety, large $$, discrimination risk | Strict golden + red-team; pre-action HITL or strong dual control; high sample audit; documented rollback; often compliance review |

**Heuristic:** if a wrong answer can move money, change access, or give regulated advice, treat as **high** until counsel/product says otherwise. “The LLM sounded confident” is not a tier.

### How eval and HITL scale with tier

```text
Low:    offline smoke ──────────────────────────► ship
Medium: offline gates → shadow/canary → sample HITL → ship
High:   offline gates → red-team → shadow → HITL on actions → slow canary → ship
```

---

## 4. Worked example A — Classical scoring API (churn)

### Problem

Marketing wants a **churn propensity score** per account to prioritize outreach. Wrong high scores waste sales time; wrong low scores miss saves. Not a regulated credit decision in this fictional product—but still customer-facing strategy.

### Constraints (example)

- Latency: scores can be up to 24h stale for email campaigns
- Cost: small team, no GPU budget
- Privacy: account features only; no raw email bodies in model input
- Skill: strong pandas/sklearn; limited SRE time

### Options

| Option | Idea |
|--------|------|
| A | Nightly **batch** sklearn model; write scores to warehouse; CRM reads table |
| B | **Online** FastAPI scoring at send-time |
| C | LLM rates “likelihood to churn” from tickets |

### Tradeoffs

| Dimension | A Batch sklearn | B Online API | C LLM judge |
|-----------|-----------------|--------------|-------------|
| Latency to user | Hours (OK for email) | ms–s | s + $ |
| Cost | Low compute | Always-on service | High per call |
| Quality | Strong if features solid | Same model, fresher | Unstable, hard to calibrate |
| Complexity | Scheduler + table | Serving + on-call | Prompt + eval + spend |
| Risk | Stale scores | Serving incidents | Hallucinated rationale |
| Skill fit | High for this team | Medium | Overkill |

### How to choose

**Default for small team:** **A**. The constraint “24h OK” removes the need for online serving. Prefer the simplest path that meets freshness.

Choose **B** if another product surface needs sub-second scores (e.g. in-app banner). Choose **C** only as a research spike—not as the score of record—unless you have a calibration story and budget.

### Eval plan

- Offline: holdout AUC/PR, calibration plots, slice by region/plan
- Gate: no promote if PR-AUC drops > X vs production champion on same data version
- Shadow (if later moving to online): log online scores vs batch for two weeks

### HITL plan

- Medium-low stakes: weekly sample of 50 high-score accounts reviewed by CS for “does this feel churny?”
- Escalation: if CS disagrees > threshold, pause campaign use and investigate feature drift

### Ship criteria & rollback

- Registry: `churn_model` vN passes gates → `Staging` → `Production`
- Rollback: point CRM job at previous table partition / previous model version

---

## 5. Worked example B — RAG support bot

### Problem

Deflect tier-1 support with answers grounded in a help center. Wrong answers create angry customers and more tickets.

### Constraints

- Latency: p95 < 5s acceptable for chat widget
- Cost: must beat human deflection ROI
- Privacy: no training on customer PII by provider if policy forbids; redact logs
- Skill: one backend engineer + one support lead for review

### Options (architecture slice)

| Option | Serving shape |
|--------|---------------|
| A | Sync API: retrieve → LLM → respond |
| B | Async: queue job, poll for answer (too slow for chat UX) |
| C | Batch FAQ generation only (no interactive RAG) |

Plus retrieval store choices (see §8).

### Tradeoffs (A vs C abbreviated)

| | Interactive RAG (A) | Batch FAQ only (C) |
|--|---------------------|--------------------|
| UX | Conversational | Static articles |
| Quality risk | Hallucination / bad retrieval | Stale FAQ |
| Cost | Per-message LLM | Periodic generation |
| HITL fit | Sample live answers | Edit FAQ before publish |

### How to choose

If users already chat and deflection ROI is the goal → **A** with citations and abstain. If volume is low and docs change slowly → **C** can be safer and cheaper.

**Default small team path for interactive:** managed LLM API + managed or simple vector store + sync FastAPI orchestrator + golden set of 50–100 support questions + support-lead weekly review queue.

### Eval plan

- Offline golden: retrieval hit rate, answer rubrics, refusal on out-of-scope, citation presence
- Shadow: new prompt/index versions log side-by-side for 5–10% traffic
- Online: thumbs-down, “escalate to human” rate, CSQA spot checks

### HITL plan

- **Pre-action:** none for read-only answers; **required** if bot gains refund tool (then high tier)
- **Post-hoc:** 2–5% of conversations or 100% of thumbs-down into review UI
- Escalation: auto-handoff when retrieval empty or policy classifier flags

### Ship criteria & rollback

- Promote only if golden faithfulness/rubric ≥ bar and no severe safety fails
- Rollback: previous `prompt_version` + `index_version` pair (version **together**)

---

## 6. Worked example C — Agent with tools

### Problem

Internal ops agent that can `lookup_order`, and later `issue_refund`.

### Constraints

- Refunds are irreversible money movement → **high tier** for refund tool
- Latency: ops can wait 10–30s
- Team: early LLM experience; strong backend

### Options

| Option | Design |
|--------|--------|
| A | Single-shot tool calling, no loop |
| B | Constrained agent loop (max 3 steps) with allowlisted tools |
| C | Free-form “autonomous” agent with many tools |

### Tradeoffs

| | A | B | C |
|--|---|---|---|
| Quality | Good for simple flows | Handles multi-step | Unpredictable |
| Complexity | Low | Medium | High |
| Risk | Lower | Medium | High (tool sprawl) |
| Eval | Easy expected tool traces | Need multi-step gold | Hard |

### How to choose

**Default:** start with **A**, grow to **B** when metrics show need. Avoid **C** until eval, sandboxing, and HITL are mature.

### Eval + HITL

- Golden traces: expected tool name + arg constraints
- **HITL:** `issue_refund` requires human approve in UI (pre-action). `lookup_order` may be auto.
- Sample 100% of refunds for first N weeks; then risk-based sampling
- Red-team: prompt injection via order notes fields

### Ship criteria

- Refund tool disabled in prod until dual control + audit log + kill switch proven in staging

---

## 7. Major decision: batch vs online

### Options

1. **Batch** — scheduled jobs score many rows; results land in tables/files  
2. **Online (sync)** — score per request in an API  
3. **Async online** — accept request, queue work, notify when done  
4. **Hybrid** — batch precompute + online read; or online LLM + batch eval

### Tradeoff table

| Dimension | Batch | Sync online | Async queue |
|-----------|-------|-------------|-------------|
| Latency to consumer | Minutes–hours | ms–seconds | seconds–minutes |
| Cost shape | Smooth scheduled | Spiky with traffic | Spiky but buffered |
| Complexity | Jobs + storage | Always-on service | API + workers + broker |
| Failure mode | Missed SLA window | User-facing errors | Backlog / lag |
| Feature freshness | Snapshot at job time | Request-time | Request-time (delayed result) |
| Eval cadence | Easy offline on snapshots | Need production logging | Same + queue metrics |

### How to choose (heuristics)

```text
Need sub-second / interactive UX?     → sync online (or hybrid read of batch)
User can wait minutes (report/email)? → prefer batch
Work takes > few seconds (big LLM)?   → async queue or streaming
Unsure?                               → batch prototype first if UX allows
```

**Small-team default:** batch whenever freshness allows; sync FastAPI for chat/RAG; add queues when p95 LLM time or fan-out tools exceed UX budget.

### Eval + HITL

- Batch: evaluate on the same snapshot you score; humans sample scored cohorts before campaigns
- Online: golden set in CI; canary + sample HITL on live answers

---

## 8. Major decision: managed LLM API vs self-host

### Options

1. **Managed API** (vendor hosted)  
2. **Self-host open weights** (your GPUs/CPU)  
3. **Hybrid** — API for hard cases; small local/classifier for routing

### Tradeoffs

| Dimension | Managed API | Self-host | Hybrid |
|-----------|-------------|-----------|--------|
| Time to first value | Fast | Slow (infra) | Medium |
| $/quality at low volume | Often better | CapEx + idle GPUs | Tunable |
| $/quality at huge volume | Can get expensive | May win | Often best |
| Latency control | Vendor + network | You own | Mixed |
| Privacy | DPA / region / retention settings | Max control | Split carefully |
| Ops burden | Low | High (GPUs, scaling) | Medium |
| Quality ceiling | Top models available | Depends on weights + tuning | Route wisely |
| Team skill | App eng enough | ML platform + GPU ops | Both |

### How to choose

**Small-team default:** **managed API** until monthly spend or privacy policy forces otherwise. Add a **cheap router** (hybrid) before you buy GPUs.

Self-host when: hard data residency, stable high volume with favorable unit economics, or need custom weights you cannot get via API.

### Eval + HITL

- Compare candidates on **your** golden set (quality, latency, cost)—never only vendor leaderboards
- Humans grade a blind sample when switching model families (style shifts fool automatic judges)

---

## 9. Major decision: build vs buy vector DB

### Options

1. **Managed vector DB** (vendor)  
2. **Open source you operate** (e.g. in-cluster engine)  
3. **Lightweight** — embeddings in Postgres/`pgvector`, or files + FAISS for small corpora  
4. **Keyword-only** (BM25) until RAG proves value

### Tradeoffs

| Dimension | Managed vector | Self-op vector | pgvector / FAISS | BM25 only |
|-----------|----------------|----------------|------------------|-----------|
| Ops | Low | High | Low–medium | Lowest |
| Scale | High | High if skilled | Good until large | Large text OK |
| Features | Filters, hybrid, ACL add-ons | Flexible | Enough for many apps | No semantic nn |
| Cost | Subscription | Eng time + infra | Cheap | Cheap |
| Lock-in | Medium | Lower | Lower | Lowest |
| Quality | Depends on embedding+tuning | Same | Same | Strong for exact terms |

### How to choose

**Small-team default:** start **BM25 or pgvector** for <100k chunks; add managed vector when hybrid search + filters + ops time justify it. Do not buy a vector DB before you have a golden retrieval set.

### Eval + HITL

- Retrieval metrics on gold queries (recall@k, MRR)
- Humans judge end answers—not only vector scores (high similarity ≠ correct doc)

---

## 10. Combining golden-set eval + shadow + human review

Promotion should not be a single metric green check.

```text
                 ┌─────────────┐
  Candidate ──►  │ Offline gold │── fail ──► stop
                 └──────┬──────┘
                        │ pass
                 ┌──────▼──────┐
                 │ Shadow /    │── regress ──► stop
                 │ canary      │
                 └──────┬──────┘
                        │ OK
                 ┌──────▼──────┐
                 │ HITL sample │── severe ──► stop / fix
                 └──────┬──────┘
                        │ OK
                   Promote + monitor
```

### Roles of each layer

| Layer | Catches | Misses |
|-------|---------|--------|
| Offline gold | Regressions on known cases; format; retrieval | Novel live phrasings; UX tone drift |
| Shadow/canary | Live distribution shift; latency/cost | Harm that needs human judgment |
| HITL sample | Tone, policy edge cases, “technically correct but wrong” | Low sample → rare events |

### Practical sample rates (starting points)

| Risk tier | Pre-promote human review | Ongoing production sample |
|-----------|--------------------------|---------------------------|
| Low | Optional 20–50 examples | 0.5–1% or weekly batch of 25 |
| Medium | 50–100 graded vs control | 1–5% + 100% of thumbs-down |
| High | Dual review on critical gold; red-team | Pre-action approval on tools; 5–20% audits or 100% of irreversible actions |

Adjust with volume: at 10 requests/day, review more percentage; at 1M/day, prioritize stratified + triggered review (low confidence, empty retrieval, user complaint).

### When automated metrics are enough

- Extraction with exact fields + schema validation
- Classical classifiers with clear labels and stable slices
- Format/JSON validity and allowlisted tool names

### When humans must review

- Open-ended answers to customers
- Safety/policy edge cases
- Model family or prompt rewrites that change tone
- Any high-tier action path

---

## 11. Anti-patterns

| Anti-pattern | Why it hurts | Do instead |
|--------------|--------------|------------|
| “We’ll add eval after launch” | You cannot detect silent quality drops | Minimal gold before first external user |
| One option considered | Blind to cheaper/safer paths | Force 2–3 options in the design doc |
| Copy big-tech architecture | Ops load kills small teams | Small-team defaults above |
| Same HITL for all tiers | Either burnout or negligence | Risk-tier the review |
| Vendor scoreboards as ship bar | Their eval ≠ your users | Your golden set + human sample |
| Rollback = “redeploy somehow” | Incident lasts hours longer | Named previous versions + one command/flag |

---

## 12. Mini practices

1. Fill the one-page template (§2) for a feature you want to build in Stage 12.  
2. Assign a risk tier and justify the HITL rate.  
3. For “Docs Q&A over 200 markdown files,” pick batch vs online and managed API vs self-host with a 5-row tradeoff table.  
4. Write ship criteria that mention **both** an automatic gate and a human gate.

---

## What is next

**[02-data-pipelines-and-mlops.md](02-data-pipelines-and-mlops.md)** — versioning data and corpora, feature skew, experiment tracking, registries, and eval gates inside the pipeline.
