# Lesson 02 — Data Pipelines and MLOps

## Why this lesson exists

Production AI is downstream of **data you can trust**. If you cannot say which rows trained a model, which docs sit in the RAG index, or which experiment produced the champion metric, you cannot safely promote, debug, or roll back.

This lesson covers dataset versioning options, feature computation and leakage, experiment tracking, model registries, reproducibility, **eval gates in the pipeline**, and when **humans must review labels or corpora**.

## Learning goals

- Compare DVC vs lakehouse tables vs S3+manifest and choose for your context
- Separate online vs offline features; spot leakage and training/serving skew
- Choose experiment tracking (W&B / MLflow / lightweight logs) with tradeoffs
- Design registry stages: dev → staging → prod with promotion rules
- Put eval gates before promote; know when HITL on data is required
- Apply a reproducibility checklist

---

## 1. MLOps in one picture

```text
raw sources ──► validate ──► versioned dataset/corpus
                                │
                     features / chunks / embeds
                                │
                           train / adapt
                                │
                        evaluate (gates)
                                │
                     register candidate ──► staging ──► production
                                │
                           observe & feedback ──► new data versions
```

**MLOps** is DevOps plus data/model artifacts: versioning, lineage, promotion, and rollback for things that are not only code.

You do not need every tool on day one. You need the **habits**—and an honest choice among options.

---

## 2. Dataset versioning: options and tradeoffs

Treat training/eval data (and RAG corpora) like code: immutable snapshots, documented lineage, checksums.

### Option A — Simple object storage + manifest

```text
s3://bucket/datasets/churn/v2026-09-15/
  train.parquet
  val.parquet
  test.parquet
  manifest.json
```

Example `manifest.json`:

```json
{
  "name": "churn",
  "version": "v2026-09-15",
  "created_at": "2026-09-15T14:00:00-04:00",
  "git_commit_data_script": "a1b2c3d",
  "files": {
    "train.parquet": {"rows": 80000, "sha256": "..."},
    "val.parquet": {"rows": 10000, "sha256": "..."},
    "test.parquet": {"rows": 10000, "sha256": "..."}
  },
  "label_rate": 0.12,
  "notes": "Excluded test accounts; fixed timezone bug in last_login"
}
```

### Option B — DVC (Data Version Control)

DVC stores hashes in Git; large files live in remote storage. `dvc pull` / `dvc repro` ties pipelines to data versions.

### Option C — Lakehouse / warehouse time travel

Tables in systems with snapshots or time travel (e.g. delta/iceberg-style tables, warehouse snapshots): query `AS OF` a version or timestamp.

### Tradeoff table

| Dimension | S3 + manifest | DVC | Lakehouse snapshots |
|-----------|---------------|-----|---------------------|
| Setup cost | Lowest | Medium (learn DVC) | Higher (platform) |
| Git-friendliness | Manifest in Git | First-class | Often separate |
| Large binary UX | Manual discipline | Strong | Strong |
| Cross-team discoverability | Weak unless documented | Medium | Strong if catalogued |
| Ad-hoc SQL on history | Poor | Poor | Excellent |
| Small-team fit | **Excellent** | Good if already Git-centric | Best when lake exists |
| RAG corpus fit | Folder of docs + manifest | Same pattern | Doc metadata tables |

### How to choose

**Small-team default:** **S3/local + manifest** until pain appears (too many ad-hoc copies, lost hashes). Adopt **DVC** when multiple people reproduce training weekly. Use **lakehouse snapshots** when the company already runs that platform and analysts need SQL time travel.

### Eval + HITL on versioning

- Automated: schema checks, row counts, null rates, hash mismatch fails the job
- Humans: review **label definitions** and spot-check stratified samples when the labeling guide changes; for RAG, humans sample new doc batches for toxic/PII/injection before upsert (see §9)

---

## 3. RAG corpora as versioned datasets

LLM apps often forget that the **index is a model artifact**.

```text
docs_raw/v42/ ──► clean ──► chunk ──► embed ──► index/v42
manifest: doc ids, ACL tags, embed model id, chunker version
```

Version **together**:

| Artifact | Why |
|----------|-----|
| Source doc snapshot | Reproducible retrieval |
| Chunker config | Boundaries change answers |
| Embedding model id | Vectors not comparable across models |
| Index build id | What production points at |

**Anti-pattern:** “just re-embed prod continuously” with no snapshot id—you cannot roll back a bad crawl.

---

## 4. Feature computation: offline vs online

### Offline features

Computed in batch from historical tables (aggregates, embeddings backfill). Written to a feature table keyed by entity + time.

### Online features

Computed at request time (current cart size, last 5 minutes of events) or fetched from a low-latency store.

### Tradeoffs

| Dimension | Offline | Online |
|-----------|---------|--------|
| Freshness | Snapshot lag | Request-time |
| Consistency with training | Easy if same job | Skew risk if different code |
| Latency | None at request (precomputed) | Adds to request path |
| Complexity | Batch jobs | Feature service / cache |
| Cost | Scheduled compute | Always-on + hot path |

### Training/serving skew

```text
Training:  days_since_signup = (train_date - signup).days
Serving:   days_since_signup = (request_time - signup).days  ✓ same definition

Bug:       training used a cached CSV with signup truncated to month
Serving:   used exact timestamp  → silent distribution shift
```

**Mitigations:** share feature code; generate feature definitions once; integration tests that compute a golden row both ways; log feature values for sampled traffic.

### Hybrid pattern (common)

```text
Batch: write entity features nightly → online store
Request: read precomputed + append truly real-time fields
```

### How to choose

- Campaign/email scores → mostly **offline**
- Fraud at payment → **online** (or hybrid with very fresh aggregates)
- RAG retrieval features (query embed) → **online**; doc embeds → **offline** batch

---

## 5. Leakage risks (classical and LLM-adjacent)

**Leakage** = information in training that will not be legitimately available at prediction time—metrics look great, production fails.

| Leak example | Why it hurts |
|--------------|--------------|
| Using `churned_flag` to build features | Direct label leak |
| Future aggregates in a window | Time-travel cheat |
| Same user in train and test without grouping | Over-optimistic generalization |
| RAG: gold answer text sitting in retrieved chunk from eval construction | Fake faithfulness |
| Prompt fine-tune data overlapping eval phrasing verbatim | Memorization theater |

### Checklist

- Split by time or by entity groups when life is temporal
- Document feature availability time
- For RAG eval, keep questions crafted so answers are not trivially copy-pasted from a single planted sentence unless that is the task
- Review new features with a second person when stakes are medium+

### HITL

Humans are good at catching “this feature feels like cheating” when reading a data dictionary. Budget 30–60 minutes of review when feature sets change for medium/high tier models.

---

## 6. Experiment tracking: options and tradeoffs

Log each run so you can answer: what data, code, params, and metrics produced this artifact?

### Minimum fields

| Field | Example |
|-------|---------|
| run_id | `uuid` |
| git_commit | `abc123` |
| data_version | `churn/v2026-09-15` |
| params | `lr=0.05, max_depth=6` |
| metrics | `val_pr_auc=0.81` |
| artifacts | model path, plots, confusion matrix |
| notes | “removed leaky feature X” |

For LLM systems also log: `prompt_version`, `index_version`, `model_name`, token costs, gold-set scores.

### Option A — Lightweight JSON/CSV logs

```python
import json, time, pathlib

def log_run(record: dict, path="runs.jsonl"):
    record = {**record, "ts": time.time()}
    pathlib.Path(path).open("a").write(json.dumps(record) + "\n")
```

### Option B — MLflow

Open tracking + model registry patterns; self-host or managed.

### Option C — Weights & Biases (W&B)

Hosted experiment UI, artifacts, collaboration features.

### Tradeoffs

| Dimension | JSON/CSV | MLflow | W&B |
|-----------|----------|--------|-----|
| Time to start | Minutes | Hours | Minutes (account) |
| UI / compare runs | DIY | Good | Excellent |
| Cost | Free | Infra or cloud | Seat/usage |
| Privacy | Local | Your server or SaaS | SaaS considerations |
| Registry integration | Manual | Strong | Artifacts + links |
| Team skill | Any | Medium | Low–medium |
| LLM prompt diffs | Manual | Custom | Nice UIs / tables |

### How to choose

**Small-team default:** start with **JSONL + a spreadsheet** or a tiny MLflow local server. Move to **W&B or hosted MLflow** when more than two people need compare UIs weekly or auditors ask for lineage.

Do not block shipping on perfect tracking—but never promote without a logged run id.

### Eval + HITL

- Automated: CI attaches gold metrics to the run
- Humans: for subjective LLM tasks, attach **human grade summary** to the run before registry promote

---

## 7. Model registry and promotion stages

A **registry** stores approved artifacts with versions and stages:

```text
name: churn_rf
  v3  Production
  v4  Staging (candidate)
  v2  Archived

name: support_rag
  artifacts: prompt@17, index@9, llm=provider-model-x
  stage: Staging
```

### Stages

| Stage | Meaning | Who can promote |
|-------|---------|-----------------|
| Dev / None | Experiment only | Anyone |
| Staging | Candidate under test | Eng + eval gates |
| Production | Live traffic | Checklist + approval |
| Archived | Kept for rollback | — |

### Promotion checklist (classical)

- [ ] Data version immutable and recorded
- [ ] Metrics ≥ bars on holdout + key slices
- [ ] No known leakage
- [ ] Artifact hash stored
- [ ] Smoke predictions on fixed fixtures
- [ ] Rollback target identified (previous prod version)
- [ ] Owner + on-call noted

### Promotion checklist (LLM / RAG)

- [ ] Prompt + index + model ids recorded as a **bundle**
- [ ] Golden set gates pass (retrieval + answer + safety)
- [ ] Shadow/canary plan ready
- [ ] HITL sample graded vs control (tier-dependent)
- [ ] Cost estimate within budget
- [ ] Kill switch / flag to revert bundle

### Eval gates before promote (examples)

```text
Classical:
  val_pr_auc >= 0.78
  slice_min_recall(region=EU) >= 0.60
  prediction_fixture_diff == 0

RAG:
  recall_at_5 >= 0.85 on gold
  rubric_pass_rate >= 0.80
  refusal_precision on safety set >= 0.95
  p95_latency_ms on staging soak <= 5000
  human_severe_issues == 0 on pre-promote sample
```

**Fail closed** on safety gates; negotiate quality gates with product.

---

## 8. Pipeline orchestration

Schedulers: cron, cloud schedulers, Airflow, Prefect, Dagster, CI nightly jobs.

### Classical ML minimal DAG

```text
validate_raw → build_features → train → evaluate →
  if gates_pass: register(staging) →
  optional: human_approval → promote(prod) → publish_scores
```

### RAG ingest DAG

```text
fetch_docs → PII/toxin scan → chunk → embed → upsert_index_candidate →
  smoke_gold_queries → if pass: swap prod pointer → notify
```

### Tradeoffs: cron vs workflow engine

| | Cron + scripts | Workflow engine |
|--|----------------|-----------------|
| Complexity | Low | Higher |
| Retries/visibility | DIY | Built-in |
| Multi-step deps | Fragile | First-class |
| When enough | 1–3 jobs | Many deps / teams |

**Small-team default:** cron or CI workflows until dependency graph hurts.

---

## 9. Human review of data labels and RAG corpora

Automated checks catch schema and obvious drift. They **miss** subtle label errors and malicious or wrong documents.

### When humans must review data

| Situation | Why automation insufficient | Pattern |
|-----------|----------------------------|---------|
| New labeling guideline | Ambiguous edge cases | Dual annotation on 100 items; measure agreement |
| High-tier classifier | Cost of wrong labels high | Ongoing audit sample |
| RAG corpus from web/user uploads | Injection, PII, wrong policy | Review queue before index; ACL checks |
| Synthetic data generation | Model copies biases | Human grade stratified sample |
| Customer thumbs-down clusters | Unknown failure mode | Relabel / add to gold |

### Sample rate heuristics (labels)

```text
agreement_check_rate = f(risk, labeler_experience, change_size)
```

Starting points:

- New annotators: 100% dual-label until agreement stabilizes
- Steady state low tier: 5–10% audit
- Medium: 10–20% + all disagreements adjudicated
- High: dual-label critical classes; adjudicator is domain expert

### RAG corpus HITL

```text
New doc batch
  → automated PII/malware/size checks
  → sample N docs for human: “Would we be OK retrieving this for customers?”
  → quarantine fails
  → embed & index candidate
  → gold query smoke
```

### Mini practice

Write a 5-question review rubric for docs entering an employee-handbook bot (PII, outdated policy, injection-like text, ACL sensitivity, readability).

---

## 10. Reproducibility checklist

Before you call a result “real”:

- [ ] Git commit clean or noted dirty flag
- [ ] Data/corpus version + hashes
- [ ] Library versions pinned (`requirements.txt` / lockfile)
- [ ] Random seeds set where stochastic
- [ ] Hardware/notes if nondeterministic GPU ops matter
- [ ] Train/val/test split rule documented (time/entity)
- [ ] Metric code versioned (same script as CI)
- [ ] For LLMs: temperature/decoding params recorded; provider model id frozen
- [ ] Someone else can re-run from the README and land within tolerance

**Tolerance:** exact bitwise match is hard with GPUs and vendor APIs. Aim for metric bands and identical inputs/outputs on deterministic fixtures.

---

## 11. CI ideas for AI repos

- Unit tests for feature transforms and prompt builders
- Schema validation on sample payloads
- Golden-set eval on pull requests (subset) + full set nightly
- Block merge/promote when critical gates fail
- Secret scanning; no API keys in prompts committed
- For RAG: chunker tests + retrieval tests on fixture index

```text
PR → unit + tiny gold (fast)
main → nightly full gold + cost report
promote job → full gates + optional HITL attestation flag
```

---

## 12. Documentation that saves incidents

Keep a living **model/system card lite**:

- Owner and backup
- Intended use / out-of-scope
- Data sources and versions
- Metrics that mean “healthy”
- Known failure modes
- How to roll back
- PII / retention notes
- Link to runbook (Lesson 06)

### Mini practice

Write a half-page card for either: (a) churn batch model or (b) support RAG bot.

---

## 13. Anti-patterns

| Anti-pattern | Fix |
|--------------|-----|
| Overwriting `latest.csv` | Immutable `vDATE` paths + manifest |
| Metrics only in chat | Experiment log with run ids |
| Promote on training accuracy | Holdout + slices + gates |
| Index without version | Bundle prompt+index+model |
| Ignoring label audits | Risk-tiered HITL on data |
| Different feature code paths | Shared library + skew tests |

---

## 14. Putting eval gates + HITL in one promote flow

```text
pipeline produces candidate artifact
        │
        ▼
automated eval job (gold + fixtures + safety set)
        │
   pass? ──no──► fail build / alert
        │yes
        ▼
registry ← Staging
        │
        ▼
HITL attestation (checkbox + link to reviewed sample) if tier ≥ medium
        │
        ▼
canary / shadow (Lesson 03–04)
        │
        ▼
Production pointer update + announce
```

---

## What is next

**[03-serving-architectures.md](03-serving-architectures.md)** — batch vs sync vs async vs streaming, FastAPI sketches, caching, multi-model routing, and rollout strategies with eval/HITL.
