# Build Guide — Tabular ML + LLM Report Hybrid

This guide builds a **hybrid system**: classical ML on a tabular dataset, then an LLM that writes a stakeholder-friendly report **strictly from computed metrics** (no invented numbers).

## Outcome

A project that:

1. Trains and evaluates a tabular model with a proper pipeline
2. Exports a machine-readable metrics bundle
3. Asks an LLM to draft an executive report using only that bundle
4. Validates the draft against the numbers (simple consistency checks)
5. Documents business caveats and next experiments

## Why this project impresses

It shows you can:

- Do real ML evaluation (not only chat demos)
- Use LLMs where they help (communication) without letting them invent metrics
- Think about reliability and HITL

## Suggested repo layout

```text
tabular_llm_report/
  README.md
  requirements.txt
  data/                    # or download script
  src/
    train_eval.py
    export_metrics.py
    generate_report.py
    validate_report.py
  artifacts/
    model.joblib
    metrics.json
  reports/
    executive_report.md
  eval/
    notes.md
```

## Step 0 — Choose a dataset

Pick a well-known open dataset you can download legally, for example:

- A scikit-learn toy/real dataset (`fetch_openml` where appropriate)
- A classic public CSV (housing, churn-style, credit—**be careful** with sensitive domains; prefer non-sensitive teaching datasets)

Document:

- Source
- License / terms
- Target definition
- Why the prediction matters (even if hypothetical product framing)

## Step 1 — Problem framing

Write in README:

- Prediction target
- Primary metric (and why)
- Baseline definition
- Ethical notes / limitations (especially if the dataset historically invites naive fairness claims—stay humble)

## Step 2 — Train/eval with scikit-learn discipline

Reuse Part 2 habits:

1. Split first
2. `ColumnTransformer` + `Pipeline`
3. Dummy baseline + ≥2 models
4. Cross-validation on train for selection
5. Final test score once for the winner
6. Save the full pipeline with joblib

```python
# Conceptual core
from sklearn.model_selection import train_test_split
from sklearn.pipeline import Pipeline
# ... build prep + model ...
pipe.fit(X_train, y_train)
# evaluate; compare to DummyClassifier/Regressor
```

Include a short **error analysis**: slice metrics by a couple of features if meaningful (for example, performance by category).

## Step 3 — Export `metrics.json`

The LLM must not compute metrics itself. Your code exports them.

Suggested schema:

```json
{
  "dataset": {
    "name": "...",
    "n_rows": 0,
    "n_features": 0,
    "target": "..."
  },
  "splits": {"train": 0, "val": 0, "test": 0},
  "baseline": {"model": "dummy_most_frequent", "metric_name": "f1", "metric_value": 0.0},
  "candidates": [
    {"model": "logistic", "val_metric": 0.0},
    {"model": "hist_gbm", "val_metric": 0.0}
  ],
  "selected": {
    "model": "hist_gbm",
    "hyperparams": {},
    "val_metric": 0.0,
    "test_metric": 0.0,
    "metric_name": "f1"
  },
  "feature_importance_top10": [
    {"feature": "age", "score": 0.12}
  ],
  "notes_for_writer": [
    "Class imbalance: positive rate 12% on train",
    "Removed customer_id to avoid leakage"
  ],
  "disallowed": [
    "Do not invent metrics not present in this JSON",
    "Do not claim causal effects"
  ]
}
```

Adapt fields for regression (MAE/RMSE, etc.).

## Step 4 — LLM report generation

Prompt pattern:

```text
System: You are a technical writer. Use ONLY the JSON metrics.
If something is missing, say it is not in the metrics pack.
Do not invent numbers. No causal claims.

User: Write a markdown executive report with sections:
1) Problem
2) Data snapshot
3) Model comparison
4) Selected model test performance
5) Top features (associative, not causal)
6) Risks & limitations
7) Recommended next experiments

METRICS_JSON:
{...}
```

Use low temperature. Save raw model output to `reports/executive_report.raw.md`.

## Step 5 — Validate the report

Write `validate_report.py` that:

1. Loads `metrics.json`
2. Checks that each numeric value appearing in the report that looks like a metric is consistent with JSON (string search for the printed values)
3. Fails if the report contains obviously forbidden phrases you choose (for example, “proves that”, “causes”)
4. Prints a PASS/FAIL checklist

This is not perfect NLP verification—it is an engineering seatbelt.

```python
def assert_number_present(report: str, value: float, tol_format="auto"):
    # Check that a reasonable string form of value appears in report
    ...
```

If validation fails, regenerate or edit manually—**HITL is expected**.

## Step 6 — Optional batch packaging

Stretch ideas:

- Script that retrains monthly and emails the report draft to a human
- Store metrics history as JSONL for trend plots (code-generated, not LLM-invented)

## Step 7 — README and demo script

Provide one command path:

```bash
python -m src.train_eval
python -m src.generate_report
python -m src.validate_report
```

Include expected artifacts and a sample report excerpt.

## Acceptance checklist

- [ ] Dummy baseline beaten (or honest discussion if not)
- [ ] Pipeline saved; no leakage in preprocessing
- [ ] `metrics.json` is the single source of truth for numbers
- [ ] Report validation step exists
- [ ] Limitations section is non-empty and honest
- [ ] Reproducible with pinned requirements and a seed

## Interview talking points

Practice answering:

1. Why not let the LLM calculate metrics?
2. What would you monitor after deploying the classifier?
3. How could feature importance mislead stakeholders?
4. How would you prevent leakage if a new column arrives from analytics?

## Common stuck points

| Problem | Try |
|---------|-----|
| LLM invents lift percentages | Strengthen system prompt; pass only JSON; validation fail + retry |
| Model cannot beat dummy | Fix leakage/features; simplify claim; still ship the honest report |
| Huge categorical cardinality | Hashing encoder / frequency cutoffs; document drop rules |

Continue with [04-portfolio-and-next-steps.md](04-portfolio-and-next-steps.md) to present the work.
