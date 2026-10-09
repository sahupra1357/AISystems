# Lesson 1.3 — Data Literacy for AI Engineers

Models get the headlines; **data** decides whether those models work. This lesson teaches how to look at data critically: formats, cleaning, splits, leakage, metrics, and honest evaluation.

## Structured vs unstructured data

| Kind | What it looks like | Examples |
|------|--------------------|----------|
| **Structured** | Rows and columns with clear fields | CSV tables, SQL tables, parquet files |
| **Semi-structured** | Nested fields, flexible schema | JSON logs, JSONL event streams |
| **Unstructured** | No fixed columns; meaning is in the content | Free text, images, audio, video, PDFs |

Classical ML shines on structured tables. Deep learning and generative AI often start from unstructured inputs—but even then you usually create structured labels, metadata, and evaluation tables around them.

As an AI engineer you must be fluent in **both**: tabular pipelines and messy real-world media/text.

## Common formats

### CSV

Comma-separated values: the workhorse of small and medium tabular datasets.

```text
age,fare,survived
22,7.25,0
38,71.28,1
```

```python
import csv

with open("toy.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["age", "fare", "survived"])
    writer.writeheader()
    writer.writerow({"age": 22, "fare": 7.25, "survived": 0})

with open("toy.csv", newline="", encoding="utf-8") as f:
    for row in csv.DictReader(f):
        print(row)
```

With pandas (once installed):

```python
import pandas as pd

df = pd.read_csv("toy.csv")
print(df.head())
```

### JSON / JSONL

JSON objects for nested records; JSONL (one JSON object per line) for logs and large streams.

```python
import json

record = {"id": 1, "tokens": ["hello", "world"], "label": "greeting"}
print(json.dumps(record))
```

### Images

Stored as files (PNG, JPEG, etc.) plus a table of paths and labels. Pixel tensors come later with libraries like Pillow and PyTorch. For now, remember: **the label file is as important as the pixels**.

### Text

Plain `.txt`, CSV columns of strings, or JSON documents. Text needs decisions about encoding (`utf-8`), language, and cleaning (lowercasing, stripping control characters)—but do not over-clean before you understand the task.

## Cleaning: missing values, duplicates, outliers

Raw data is messy. Cleaning is not glamorous, but it is most of the job early on.

### Missing values

Causes: sensors fail, users skip fields, joins do not match.

Options (task-dependent):

- Drop rows (if few and random)
- Impute with mean/median/mode (simple baseline)
- Impute with models (more advanced)
- Keep an explicit “missing” indicator feature

```python
import pandas as pd

df = pd.DataFrame({"age": [22, None, 30], "fare": [7.25, 8.0, None]})
print(df.isna().sum())

# Simple numeric fill for a baseline
df["age"] = df["age"].fillna(df["age"].median())
df["fare"] = df["fare"].fillna(df["fare"].median())
```

Never casually fill **after** peeking at the test set with train+test statistics—that is a leak (see below).

### Duplicates

Duplicate rows can inflate scores and leak identity across splits.

```python
df = pd.DataFrame({"id": [1, 1, 2], "value": [10, 10, 20]})
print(df.duplicated())
df = df.drop_duplicates()
```

Ask: are duplicates true repeated events, or copy-paste errors? The answer changes whether you drop them.

### Outliers

Extreme values may be errors (age 999) or rare but real events (fraud amounts). Plot or summarize before deleting.

```python
fares = pd.Series([7.0, 8.0, 7.5, 500.0])
print(fares.describe())
# Investigate the 500 before dropping
```

Rule of thumb: **investigate** outliers; do not auto-delete them because a tutorial said so.

### Mini practice — cleaning

Create a tiny DataFrame with one missing value, one duplicate row, and one absurd outlier. Write down what you would do with each and why.

## Data leakage (plain explanation)

**Data leakage** means information from outside the training-time world sneaks into training or preprocessing, so validation scores look great and production fails.

Plain examples:

1. **Target leakage:** including a feature that is only known after the outcome happens (for example, “days in ICU” to predict “will be admitted to ICU”).
2. **Split leakage:** normalizing features using the **whole** dataset mean before splitting train/test—test information influences training.
3. **Duplicate leakage:** the same person or near-identical row appears in train and test.
4. **Time leakage:** using future rows to predict the past in a forecasting problem.

If your offline score is magical and production is sad, suspect leakage first.

Safe habit:

```text
Split first → fit preprocessing on train only → apply to val/test
```

## Splits: train / validation / test

| Split | You use it to… | You must not… |
|-------|----------------|---------------|
| Train | Learn parameters | Report it as final generalization |
| Validation | Choose models / hyperparameters | Pretend it is untouched forever if you peeked 100 times |
| Test | Report final performance once | Tune on it |

For tiny datasets, people sometimes use cross-validation on train+val and keep a final test holdout. For time series, split by time, not by random shuffle.

```python
import numpy as np

n = 100
idx = np.arange(n)
rng = np.random.default_rng(42)
rng.shuffle(idx)

train_idx = idx[:60]
val_idx = idx[60:80]
test_idx = idx[80:]
print(len(train_idx), len(val_idx), len(test_idx))
```

Common starting ratios (not laws): 60/20/20 or 80/10/10 depending on data size.

## Metrics: what to use when

Metrics translate predictions into a number you can optimize and discuss. The wrong metric optimizes the wrong thing.

### Classification

Assume a binary problem with positive and negative labels.

| Metric | Idea | Fits when… |
|--------|------|------------|
| **Accuracy** | Fraction correct | Classes are balanced; all errors cost similarly |
| **Precision** | Of predicted positives, how many are right | False positives are costly (spam → inbox, alerts → fatigue) |
| **Recall** | Of actual positives, how many you caught | False negatives are costly (disease, fraud) |
| **F1** | Harmonic mean of precision and recall | You need a balance; classes may be imbalanced |

```python
def precision_recall_f1(y_true, y_pred, positive=1):
    tp = sum(1 for t, p in zip(y_true, y_pred) if t == positive and p == positive)
    fp = sum(1 for t, p in zip(y_true, y_pred) if t != positive and p == positive)
    fn = sum(1 for t, p in zip(y_true, y_pred) if t == positive and p != positive)
    precision = tp / (tp + fp) if (tp + fp) else 0.0
    recall = tp / (tp + fn) if (tp + fn) else 0.0
    f1 = (2 * precision * recall / (precision + recall)) if (precision + recall) else 0.0
    return precision, recall, f1


y_true = [1, 1, 0, 0, 1]
y_pred = [1, 0, 0, 1, 1]
print(precision_recall_f1(y_true, y_pred))
```

### Regression

| Metric | Idea | Notes |
|--------|------|-------|
| **RMSE** | Square root of mean squared error | Penalizes large errors; same units as target after sqrt |
| **MAE** | Mean absolute error | More robust to outliers than squared error |

```python
import math

def rmse(y_true, y_pred):
    return math.sqrt(sum((t - p) ** 2 for t, p in zip(y_true, y_pred)) / len(y_true))


print(rmse([3.0, 5.0], [2.5, 5.5]))
```

### When each fits (short guide)

- Balanced “overall correctness” → accuracy may be fine
- Rare positives, costly misses → prioritize recall (and look at precision too)
- Rare positives, costly false alarms → prioritize precision
- Need one summary for imbalanced classification → F1 (or better: precision-recall curves later)
- Continuous targets → RMSE or MAE; pick based on how much large errors hurt

Always ask: **what mistake hurts the user more?**

## Honest evaluation habits

1. **Split before fitting** anything that learns from data (scalers, imputers, models).
2. **Write the metric down** before you train, tied to the product goal.
3. **Keep a real holdout** you almost never touch.
4. **Slice metrics** by important subgroups (region, device, demographic if appropriate and legal) so average scores do not hide failures.
5. **Compare to a dumb baseline** (predict majority class; predict mean target). If you cannot beat it, you do not have a result yet.
6. **Log seeds, data versions, and code versions** so results are reproducible.
7. **Distrust miracles.** If the score looks too good, search for leakage.

## Mini practice — public dataset idea

Pick one well-known public tabular dataset and practice the full hygiene loop. Two classic choices:

- **Titanic** survival classification (passenger features → survived or not)
- **California / Boston-style housing** regression (house features → price) — prefer maintained public versions from common ML libraries or educational mirrors

Do **not** need a fancy model yet. Practice:

1. Load the CSV into pandas
2. Inspect `head()`, `info()`, and basic describes
3. Note missing values and obvious oddities
4. Create train/val/test splits (or train/test for a first pass)
5. Choose a metric (accuracy/F1 for Titanic; RMSE for housing)
6. Write a short markdown note: what could leak? what is the majority-class baseline?

You will train stronger baselines in Stage 2 with scikit-learn. The win here is **seeing the data clearly**.

Example skeleton for a local CSV named `data.csv`:

```python
import json
import pandas as pd
from pathlib import Path

df = pd.read_csv("data.csv")
summary = {
    "rows": int(df.shape[0]),
    "cols": int(df.shape[1]),
    "columns": list(df.columns),
    "missing": {c: int(df[c].isna().sum()) for c in df.columns},
}
Path("data_summary.json").write_text(json.dumps(summary, indent=2), encoding="utf-8")
print(summary)
```

## Section recap

- Know your data’s shape: structured vs unstructured, and the file formats involved
- Clean deliberately: missingness, duplicates, outliers
- Prevent leakage with split-first discipline
- Match metrics to costs of errors
- Evaluate honestly; beat a simple baseline

Next: [04-exercises-and-checklist.md](04-exercises-and-checklist.md) to practice and confirm you are ready for Classical ML (Stage 2).
