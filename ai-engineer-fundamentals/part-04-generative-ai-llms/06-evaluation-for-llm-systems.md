# Lesson 06 — Evaluation for LLM Systems

If you cannot measure quality, every prompt edit is a coin flip. LLM systems fail differently from classical classifiers: fluent wrong answers, partial citations, brittle formats, retrieval misses. Evaluation must mix **automatic checks**, **rubric grading**, **human review**, and **online signals**—with regression gates that block silent damage.

This lesson builds that discipline so Part 5 can plug it into CI, monitoring, and incidents.

## Learning goals

- Build offline golden sets for prompts, RAG, and tools
- Choose metrics per task family (exact, schema, retrieval, faithfulness, preference)
- Use pairwise compares and rubric scoring without worshipping LLM-as-judge
- Design regression gates with a CI mindset
- Wire online feedback (thumbs, edits, escalations) back to the gold set
- Avoid common eval anti-patterns

---

## 1. Why LLM eval is different

| Classical ML eval | LLM system eval |
|-------------------|-----------------|
| Clear labels often | Many tasks are open-ended |
| Single model score | Pipeline: retrieve → generate → validate → tools |
| Accuracy/F1 often enough | Need faithfulness, format, safety, UX |
| Static features | Prompts and corpora change weekly |

You evaluate **systems**, not only models.

---

## 2. Offline golden sets

A **golden set** is a curated list of inputs with expected behaviors:

- Exact expected strings or field values (extraction)
- Acceptable answer rubrics (open generation)
- Required citations / doc ids (RAG)
- Tool call expectations (name, args constraints)
- Hard **must refuse / abstain** cases

### Sizing

Start with **30–100** examples; grow as production failures appear. Ten is better than zero; 500 with no maintenance becomes stale theater.

### Coverage dimensions

- Happy paths
- Ambiguous asks
- Adversarial / injection-ish inputs (lightweight here; deeper in Part 5)
- Empty retrieval
- Multi-turn coreference
- Each product locale / tier if relevant
- Safety refusals

### Example schema for a gold item

```json
{
  "id": "refund_digital_14",
  "input": {"question": "How long for digital refunds?"},
  "expect_retrieval_ids_any": ["c_digital_refund"],
  "rubric": [
    "Mentions 14 days",
    "Does not invent restocking fee",
    "Cites a digital-refund chunk id"
  ],
  "must_abstain": false,
  "tags": ["policy", "refunds"]
}
```

Store gold in version control. Review changes like code.

---

## 3. Metrics and checks by layer

### Prompt / generation layer

| Check | Use |
|-------|-----|
| Schema validation rate | Structured outputs |
| Exact match / field F1 | Extraction |
| Regex / allowlist labels | Classification |
| Length bounds | UX constraints |
| Forbidden string checks | “as an AI language model…”, leaked system text |

### Retrieval layer

| Check | Use |
|-------|-----|
| recall@k | Needed chunk present |
| MRR / nDCG | Ranking quality |
| ACL negative tests | Forbidden docs never appear |

### Faithfulness / groundedness

- Human or rubric: is each claim supported by context?
- Automatic span checks when quotes required
- Citation precision: cited ids exist in provided context

### Tool layer

- Correct tool selected
- Args valid and authorized
- No write tool on ineligible cases
- Step count within cap

### Operational

- Latency p50/p95
- Cost per successful answer
- Error / repair rate

---

## 4. Rubrics and human review

For open answers, define **dimensions** with scales:

```text
Correctness 1-5
Faithfulness 1-5
Helpfulness 1-5
Tone 1-5
Safety pass/fail
```

Train reviewers with 10+ anchor examples. Measure inter-rater agreement occasionally.

**Sampling:** review all gold offline; online review a stratified sample (failures, low confidence, high-cost, VIP users).

---

## 5. Pairwise comparison

When absolute scores are noisy, ask: “Which answer is better, A or B, given the rubric?”

Uses:

- Prompt v3 vs v4
- Model upgrade shadowing
- With/without reranker

Control for position bias (randomize order). Record win/lose/tie rates with confidence intervals if you can.

---

## 6. LLM-as-judge (use carefully)

An LLM can apply a rubric at scale **if**:

- You calibrate against humans on a panel
- The judge prompt is versioned and simple
- You do not treat scores as absolute truth
- You avoid judges grading their own unconstrained generations without anchors

Failure modes: verbosity bias, position bias, shared blind spots with the candidate model.

**Practice:** on 10 examples, compare your scores to a judge; note disagreements—that gap is your calibration debt.

---

## 7. Regression gates (CI mindset)

Treat prompts, chunkers, and tool schemas as code:

```text
on change to prompts/** or rag/**:
  run offline gold suite
  if critical tags fail → block merge / block deploy
  publish score diff artifact
```

### Critical vs non-critical

| Critical (block) | Non-critical (warn) |
|------------------|---------------------|
| Safety refusals | Mild tone variance |
| ACL tests | Small latency blip within SLO |
| Schema_pass on payments path | Optional emoji style |

```python
CRITICAL_TAGS = {"safety", "acl", "payments"}

def gate(results):
    for r in results:
        if r["tag"] in CRITICAL_TAGS and not r["pass"]:
            raise SystemExit(f"critical fail: {r['id']}")
```

---

## 8. Online signals

Offline gold never covers everything. Wire:

| Signal | Meaning |
|--------|---------|
| Thumbs down / up | Coarse UX |
| User edits of drafts | What “good” looked like |
| Escalation to human | Failure or high stakes |
| Repeat query rate | Confusion |
| Citation click-through | Trust / usefulness |
| Tool error rates | Integration health |

**Closed loop:** weekly triage → add 5 new gold items from real failures → fix → re-run suite.

Privacy: strip PII before promoting logs into gold.

---

## 9. Experiment tracking (lightweight → serious)

Log per experiment:

- Model name + version
- Prompt template hash/version
- Retriever config (k, hybrid, rerank)
- Tool schema version
- Gold score summary by tag
- Cost/latency sample
- Qualitative failure notes
- Owner + date

A markdown table or spreadsheet works at first; Part 5 introduces fuller MLOps tooling. The habit matters more than the brand of tracker.

```text
| date       | exp_id | prompt | recall@5 | rubric_pass | notes            |
|------------|--------|--------|----------|-------------|------------------|
| 2026-09-23 | e14    | v7     | 0.82     | 0.86        | hybrid on        |
```

---

## 10. Eval for each Part 4 subsystem

### Prompting

- Schema fixtures
- Style checklist
- Injection-ish delimited cases (expect ignore)

### RAG

- Retrieval recall
- Faithfulness
- Abstain correctness
- Freshness spot checks after reindex

### Tools / agents

- Expected tool traces
- Cap enforcement tests
- Authz negatives
- HITL required on write paths

### Fine-tunes

- Beat baselines
- Refusal suite
- Drift vs previous adapter

---

## 11. Worked example: standing up eval in a week

**Day 1–2:** Collect 40 questions from stakeholders; label expected chunk ids; write rubrics.

**Day 3:** Automate schema + recall@k scripts.

**Day 4:** Human grade baseline answers (prompt+RAG).

**Day 5:** Add CI job on prompt changes; document how to add a gold case.

**Day 6–7:** Sample production failures; add 10 cases; fix top retrieval issue; confirm scores rise.

---

## 12. Statistical humility

- Small gold sets → noisy win rates; look for large, stable lifts
- Multiple changes at once → confounded results
- Model provider backend changes → pin versions when possible and rebaseline

Do not claim “+2%” as gospel on 25 examples without checking variance.

---

## 13. Anti-patterns

| Anti-pattern | Prefer |
|--------------|--------|
| Vibes-only demos | Gold + rubrics |
| One overall accuracy number | Layered metrics |
| Judging only final prose | Include retrieval/tool traces |
| Stale gold forever | Closed loop from prod |
| Blocking on vanity metrics | Critical tags only |
| LLM-judge without human calibration | Calibrate or keep humans |
| Hiding eval failures to ship dates | Transparent debt list |

---

## 14. Debugging with eval

When scores drop after a “harmless” change:

1. Diff which **ids** flipped fail←pass
2. Separate retrieval fails vs generation fails
3. Check token truncation metrics
4. Re-run on pinned model version
5. Roll back template if critical tags fail

Eval is a flashlight, not a punishment.

---

## 15. Practice

1. Write 10 gold items for a handbook bot including 3 abstains; specify retrieval ids.
2. Define a 4-dimension rubric with anchor descriptions for scores 1, 3, and 5 on Correctness.
3. Sketch a CI gate policy for your team (what blocks vs warns).
4. Design an online feedback event schema (JSON fields).
5. Compare pairwise vs absolute scoring for a tone-only change—when is each better?

---

## 16. Bridge to production architecture

```text
Orchestrator emits: trace_id, template_version, chunk_ids, tool_trace, cost
        ↓
Eval harness consumes traces + gold
        ↓
Dashboards: pass rates, cost, latency, thumbs
        ↓
Alerts → on-call → gold additions → fix
```

Part 5 operationalizes serving and safety; your eval hooks must already exist.

---


## 17. Concrete metric formulas (intuition)

**recall@k** for one question: 1 if any relevant chunk id is in the top-k retrieved list, else 0. Average over questions.

**schema_pass_rate**: fraction of outputs that parse and satisfy the schema.

**rubric_pass**: fraction of items where all critical rubric bullets pass (define “critical” explicitly).

**tool_trace_accuracy**: fraction of items where the expected tool sequence matches (exact or “allowed set” policy you document).

```python
def recall_at_k(retrieved_ids, relevant_ids, k):
    top = set(retrieved_ids[:k])
    return 1.0 if top.intersection(relevant_ids) else 0.0
```

## 18. Slice-based reporting

Overall averages hide pain. Report by slice:

- tag (`refunds`, `benefits`, `acl`)
- must_abstain true/false
- long vs short questions
- with/without conversation history

A system can be “87% overall” while failing 40% of abstain cases—unacceptable for policy bots.

## 19. Human review session recipe

1. Sample 20 items (10 fail, 5 pass, 5 random)
2. Hide model / prompt version labels when possible (blind)
3. Grade with rubric independently
4. Discuss disagreements 10 minutes
5. Update anchors if the rubric was ambiguous
6. File gold fixes separately from model fixes



## 20. End-to-end eval harness sketch

```python
def evaluate_system(cases, retrieve_fn, generate_fn, k=5):
    rows = []
    for case in cases:
        chunks = retrieve_fn(case["input"])
        retrieved_ids = [c["chunk_id"] for c in chunks[:k]]
        recall = float(bool(set(retrieved_ids) & set(case.get("expect_retrieval_ids_any", [])))) if case.get("expect_retrieval_ids_any") else None
        raw = generate_fn(case["input"], chunks)
        schema_ok = schema_validate(raw, case.get("schema")) if case.get("schema") else None
        rubric_ok = human_or_checklist(raw, case.get("rubric", []))
        abstain_ok = check_abstain(raw, case.get("must_abstain", False))
        rows.append({
            "id": case["id"],
            "tags": case.get("tags", []),
            "recall": recall,
            "schema_ok": schema_ok,
            "rubric_ok": rubric_ok,
            "abstain_ok": abstain_ok,
        })
    return rows
```

Wire this to CI with deterministic stubs for retrieve/generate when you need hermetic tests; run live model eval nightly or pre-release.

## 21. Scoring abstain correctly

Abstain cases are easy to get wrong in metrics:

- If `must_abstain=true` and model answers inventively → **fail**
- If `must_abstain=true` and model abstains → **pass** (even if phrasing varies)
- If `must_abstain=false` and model abstains because retrieval missed → often a **retrieval fail**, not a “good careful model” win—track separately

## 22. Release checklist tying eval to Part 5

Before calling a change “production ready”:

- [ ] Offline gold gate green on critical tags
- [ ] Shadow or canary plan with online metrics
- [ ] Rollback switches for template / index / adapter / model id
- [ ] Logging fields present for postmortems
- [ ] Cost alert thresholds updated if context size grew
- [ ] Safety review triggered if tools or prompts changed privilege


## Key vocabulary

golden set; rubric; faithfulness; recall@k; pairwise comparison; LLM-as-judge; regression gate; critical tag; online signal; closed loop; baseline; shadow eval

## What is next

**[07-exercises-and-checklist.md](07-exercises-and-checklist.md)** — deliberate practice across all lessons, a prompt library + eval set capstone, and the Ready-for-Part-5 checklist.
