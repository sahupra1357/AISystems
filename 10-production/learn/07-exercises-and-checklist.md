# Stage 10 — Exercises and “Ready for Stage 11” Checklist

These exercises push you to **decide with tradeoffs**, not only memorize terms. Build small prototypes where you can. The capstone is a production design doc a teammate could implement.

---

## Exercises — Lesson 10.1 (Mindset & decisions)

### Beginner

1. **Notebook gap.** List five notebook habits from a past project that would break in production; map each to a mitigation from Lesson 10.1.
2. **Risk tier.** Assign low/medium/high to: (a) emoji suggestion, (b) customer policy chatbot, (c) auto-refund agent. Justify in 3–4 sentences each.
3. **One-pager.** Fill the decision template (problem → … → rollback) for a feature you might build in Stage 12.
4. **Batch vs online.** For nightly churn email scores vs payment-time fraud vs docs chatbot, choose batch/sync/async/hybrid and cite the constraint that dominated.

### Stretch

5. **Three-option memo.** For “Q&A over 500 internal markdown docs,” compare managed LLM vs self-host vs hybrid routing; include a tradeoff table and small-team default.
6. **Promote plan.** Describe how golden-set + shadow + HITL sample combine before promote for a medium-tier RAG bot (include sample rates).

---

## Exercises — Lesson 10.2 (Data & MLOps)

### Beginner

1. **Manifest.** Write a `manifest.json` for any dataset you have used (row counts, hashes or hash plan, split sizes, notes).
2. **Experiment log.** Create a JSONL schema and log three fictional (or real) runs including data version and metrics.
3. **Skew story.** Invent a training/serving skew bug for `days_since_signup` or `last_30d_spend`; write a test that would catch it.
4. **Leak hunt.** Given features `{label, user_id, signup_date, future_30d_revenue, country}`, which columns are unsafe for training a churn model at signup time?

### Stretch

5. **RAG ingest DAG.** ASCII pipeline from “docs in Drive export” → “prod index pointer,” including PII scan, gold smoke, and notify-on-fail.
6. **Promotion gates.** Write pass/fail gates for (a) classical churn model and (b) support RAG bundle (prompt+index+model).
7. **HITL on labels.** Design a label audit plan when you add a new “cancel_reason” class: dual-label rate, adjudicator, exit criteria.

---

## Exercises — Lesson 10.3 (Serving)

### Beginner

1. **FastAPI stub.** Implement `/health` and a fake `/predict` returning a constant; run locally with uvicorn.
2. **Latency budget.** Allocate 5,000 ms across retrieval, LLM, filters for RAG; list two cuts if LLM alone is 4,500 ms.
3. **Cache key.** Write a safe response-cache key for a multi-tenant FAQ bot; explain one unsafe key.
4. **When queue.** Give three signals that would make you add a worker queue to a sync LLM API.

### Stretch

5. **Dockerize** your stub API (Dockerfile + run instructions); pin versions.
6. **Canary plan.** One-page rollout for prompt vN→vN+1: shadow → 1% → 5% → 25% → 100%, with rollback triggers and HITL rates at each stage.
7. **Routing.** Design cheap→expensive routing rules for support chat; define how you eval misrouting.

---

## Exercises — Lesson 10.4 (Monitoring, drift, cost)

### Beginner

1. **Log schema.** JSON log for an LLM turn with redaction notes (no raw secrets/PII).
2. **Dashboard.** Sketch five panels for a RAG bot (reliability, quality, cost).
3. **Judge caveat.** Write a paragraph on when LLM-as-judge is misleading for your Stage 12 idea.
4. **Cost alert.** Formula + thresholds at 80% and 100% of daily budget; what human does when it fires.

### Stretch

5. **Drift plan.** For RAG, list signals for index staleness and the HITL sample you pull when empty-retrieval doubles.
6. **Review queue UX.** Wireframes or bullet UI for a thumbs-down review tool (fields + actions + SLA).
7. **Capacity math.** Traffic 5,000/day, capacity 50 reviews/day, thumbs-down 1%: design a feasible sample/trigger policy.

---

## Exercises — Lesson 10.5 (Safety, privacy, HITL)

### Beginner

1. **PII redaction.** Extend email redaction to phones; test on ≥5 strings including a hard case.
2. **Injection examples.** Write three malicious retrieved-doc snippets; map each to a layered mitigation.
3. **Tool policy.** For `send_email(to, body)`, propose schema caps + HITL rules + server-side authz checks.
4. **HITL rate.** Using risk × volume × error cost, propose rates for: FAQ answer, “change password” link send, $500 refund.

### Stretch

5. **Threat model memo.** One page for an employee-handbook RAG bot with SSO and ACL.
6. **Red-team set.** Create ≥10 items (jailbreak, injection, PII probe, ACL cross-tenant, refusal).
7. **Kill switches.** List five switches for a support agent with refunds; note who can flip and how you test them.

---

## Exercises — Lesson 10.6 (Reliability & incidents)

### Beginner

1. **Timeout budget.** Set per-stage timeouts for retrieve + LLM + tool; align with a 15s gateway.
2. **Fallback copy.** User-facing fallback text with support reference id; tone check.
3. **Retry policy.** Pseudocode: what errors retry, backoff, when to stop.
4. **Rollback drill.** Write exact steps to revert `prompt@17/index@9` → `@16/@8` including who verifies gold.

### Stretch

5. **Circuit breaker.** Pseudocode open/half-open/closed around a provider SDK; define fallback.
6. **Runbook.** Quality incident: “citation-less answers spike after deploy.” Include eval-gap questions.
7. **Postmortem.** Fictional bad promote; write blameless postmortem with ≥2 gold/HITL action items.

---

## Stage 10 Capstone — Production design doc

Pick **one** system: **RAG support/docs bot** *or* **tabular classical scoring (+ optional LLM report)**.

### Deliverables

Produce a doc (roughly 4–8 pages) with these sections:

1. **Problem & users** — job to be done; out of scope  
2. **Risk tier** — justification  
3. **Options considered** — at least **three** real alternatives for the hardest architectural choice (serving shape *or* model hosting *or* vector store *or* HITL pattern)  
4. **Tradeoff table** — cost, latency, quality, complexity, risk, team skill  
5. **Chosen path & why** — small-team heuristics welcome  
6. **Architecture diagram** — ASCII OK: data → adapt → serve → observe → improve  
7. **Artifact inventory** — data/corpus versions, model/prompt/index versions, config  
8. **Eval plan** — golden set outline, gates, shadow/canary  
9. **HITL plan** — patterns, sample rates, triggers, reviewer roles, capacity check  
10. **Monitoring & SLOs** — reliability, quality, cost; alert list  
11. **Safety & privacy** — PII, injection, tools, kill switches  
12. **Reliability** — timeouts, retries, fallbacks, rollback  
13. **Rollout plan** — stages and abort criteria  
14. **Open questions** — honest unknowns  

### Acceptance bar

A teammate could implement an MVP from your doc without inventing major missing pieces. Options and tradeoffs must be real (not strawmen). Eval and HITL cannot be “TBD after launch.”

### Optional build

Implement a thin slice: FastAPI `/health` + stub orchestrator + manifest + fake gold runner. Not required to “pass” Stage 10, but excellent portfolio evidence.

---

## Checklist: Ready for Stage 11

### Decision mindset

- [ ] I can walk a feature through options → tradeoffs → eval → HITL → ship → rollback
- [ ] I can assign a risk tier and match verification level

### Data & MLOps

- [ ] I can explain at least two dataset versioning options and when I’d pick each
- [ ] I understand training/serving skew and leakage examples
- [ ] I know what a model/prompt/index registry promote requires

### Serving

- [ ] I can choose batch vs sync vs async vs streaming with a tradeoff table
- [ ] I can sketch FastAPI (+ Docker ideas) and when to add a queue
- [ ] I can describe shadow/canary and safe caching keys

### Monitoring & cost

- [ ] I know what to log (and redact) for classical vs LLM calls
- [ ] I can compare gold vs online feedback vs LLM-as-judge
- [ ] I can outline drift signals for my system type and a review queue trigger

### Safety & HITL

- [ ] I can explain prompt injection + a layered mitigation (including authz for tools)
- [ ] I have a HITL stance with **rates** for at least two action classes
- [ ] I can list kill switches and abuse reporting

### Reliability

- [ ] I can name timeouts, retries, idempotency, circuit breakers, fallbacks in a design
- [ ] I can describe rollback of a version **bundle**
- [ ] I know how a quality incident runbook differs from an infra one

### Capstone

- [ ] I completed the Stage 10 production design-doc capstone (or equivalent at work)

---



---



---

## How to pace the exercises (4–6 week map)

| Week | Lesson read | Exercises to complete |
|------|-------------|------------------------|
| 1 | 00–01 | Lesson 10.1 beginner 1–4; start one-pager for capstone topic |
| 2 | 02 | Lesson 10.2 beginner + stretch 5 or 6 |
| 3 | 03 | FastAPI stub + canary plan stretch |
| 4 | 04 | Log schema + capacity math |
| 5 | 05 | Threat model + HITL rates + red-team set |
| 6 | 06–07 | Runbook + full design-doc capstone; checklist |

If short on time: prioritize **Lesson 10.1 one-pager**, **promotion gates**, **HITL rate design**, and the **full capstone doc**—those transfer directly into Stage 12.

---

## Peer review questions (swap docs with a friend)

1. Did they present strawman options or real alternatives?
2. Is the HITL plan staffed (capacity) or aspirational?
3. Can you find the rollback bundle ids?
4. Would you feel safe as on-call using only their runbook section?
5. What did eval miss that a hostile user would try first?

## Capstone self-grade rubric

Score yourself 0–2 on each (target ≥ 12/16 before Stage 12):

| Criterion | 0 | 1 | 2 |
|-----------|---|---|---|
| Options real (≥3) | One path only | Two shallow | Three real with costs |
| Tradeoff table | Missing | Partial | Full dimensions used |
| Eval plan | “TBD” | Gold mentioned | Gates + shadow/canary |
| HITL plan | “We’ll review” | Pattern named | Rates + capacity + triggers |
| Rollback | Vague | Previous version named | Bundle + flag steps |
| Safety | Ignored | Generic | Threats tied to design |
| Monitoring | None | Infra only | Infra + quality + cost |
| Honesty | Overclaim | Some unknowns | Clear open questions |

If you score under 12, revise the weak sections before starting Stage 12 builds—the design doc is the rehearsal for production judgment.

## Suggested next step

Proceed to **[Stage 12 — Capstones](../../12-capstones-and-interviews/learn/00-overview.md)**: pick a portfolio project, follow a build guide, and package metrics, limitations, and production habits for hiring conversations or real stakeholders.

Carry your Stage 10 design doc forward—Stage 12 is where you implement a slice of it end-to-end.
