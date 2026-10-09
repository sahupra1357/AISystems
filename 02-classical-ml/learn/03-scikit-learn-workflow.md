# Lesson 03 — The scikit-learn Workflow

Concepts and algorithms matter, but **workflow** is what separates a toy demo from trustworthy work. This lesson walks a standard loop with scikit-learn: split → preprocess → model → evaluate → iterate.

Official API details live in the **scikit-learn documentation**. Prefer docs over random blog snippets when signatures change.

## The shape of a project

```text
raw CSV
  → load + light clean
  → train / validation / test split  (or train/test + CV on train)
  → Pipeline(preprocess, model)
  → fit on train
  → evaluate on validation (tune here)
  → refit final pipeline on train(+val) if that is your protocol
  → score once on test
  → save model + report
```

## Train / validation / test in code

```python
from sklearn.model_selection import train_test_split

X_trainval, X_test, y_trainval, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y  # stratify for classification
)

X_train, X_val, y_train, y_val = train_test_split(
    X_trainval, y_trainval, test_size=0.25, random_state=42, stratify=y_trainval
)
# 0.25 of 0.8 → 0.2 overall validation; 0.6 train; 0.2 test
```

For regression, drop `stratify` (or bin the target if you have a reason).

**Rule:** any statistic used for imputation, scaling, or target encoding must be computed on **train** (or inside a CV fold’s train portion), then applied to val/test.

## Pipelines

A `Pipeline` chains steps so transforms and the model fit together—and so CV applies them correctly without leakage.

```python
from sklearn.pipeline import Pipeline
from sklearn.impute import SimpleImputer
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import LogisticRegression

pipe = Pipeline([
    ("impute", SimpleImputer(strategy="median")),
    ("scale", StandardScaler()),
    ("clf", LogisticRegression(max_iter=1000)),
])

pipe.fit(X_train, y_train)
val_pred = pipe.predict(X_val)
```

### ColumnTransformer for mixed types

Real tables mix numeric and categorical columns.

```python
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import OneHotEncoder
from sklearn.ensemble import RandomForestClassifier

numeric_features = ["age", "income"]
categorical_features = ["city", "plan_type"]

preprocess = ColumnTransformer(
    transformers=[
        ("num", Pipeline([
            ("impute", SimpleImputer(strategy="median")),
            ("scale", StandardScaler()),
        ]), numeric_features),
        ("cat", Pipeline([
            ("impute", SimpleImputer(strategy="most_frequent")),
            ("onehot", OneHotEncoder(handle_unknown="ignore")),
        ]), categorical_features),
    ]
)

pipe = Pipeline([
    ("prep", preprocess),
    ("model", RandomForestClassifier(n_estimators=200, random_state=42)),
])
```

This pattern is worth memorizing: **ColumnTransformer + Pipeline**.

## Cross-validation

When data is limited, **k-fold cross-validation** on the training set gives a more stable estimate than a single validation split.

```python
from sklearn.model_selection import cross_val_score

scores = cross_val_score(pipe, X_trainval, y_trainval, cv=5, scoring="f1")
print(scores.mean(), scores.std())
```

Use **StratifiedKFold** for classification (scikit-learn often does this by default for classifiers in `cross_val_score`).

### Nested CV (awareness level)

If you tune hyperparameters with CV, the CV score is slightly optimistic. Nested CV estimates generalization more honestly but costs compute. For learning projects, a clear train/val/test protocol is enough; know nested CV exists for high-stakes claims.

## Hyperparameter search

```python
from sklearn.model_selection import GridSearchCV

param_grid = {
    "model__n_estimators": [100, 300],
    "model__min_samples_leaf": [1, 5],
}

search = GridSearchCV(pipe, param_grid, cv=5, scoring="f1", n_jobs=-1)
search.fit(X_trainval, y_trainval)
print(search.best_params_, search.best_score_)
best_model = search.best_estimator_
```

Notes:

- Parameter names use `step__param` syntax for pipelines
- Prefer a modest grid you understand over a huge random search you cannot interpret
- `RandomizedSearchCV` is often more efficient for larger spaces

## Metrics that match the job

### Classification

| Metric | Use when |
|--------|----------|
| Accuracy | Classes balanced; all errors similar cost |
| Precision | False positives are costly |
| Recall | False negatives are costly |
| F1 | Balance precision and recall |
| ROC AUC | Ranking quality across thresholds (careful with heavy imbalance) |
| PR AUC | Imbalanced positives; focus on precision–recall |

```python
from sklearn.metrics import classification_report, confusion_matrix, f1_score

print(confusion_matrix(y_val, val_pred))
print(classification_report(y_val, val_pred))
print("F1:", f1_score(y_val, val_pred))
```

### Regression

| Metric | Notes |
|--------|-------|
| MAE | Average absolute error; robust-ish, interpretable units |
| RMSE / MSE | Penalizes large errors more |
| \(R^2\) | Variance explained; can mislead—pair with MAE/RMSE |

```python
from sklearn.metrics import mean_absolute_error, mean_squared_error

mae = mean_absolute_error(y_val, val_pred)
rmse = mean_squared_error(y_val, val_pred) ** 0.5
```

**Always** state the metric before heavy tuning.

## Baselines you should beat

A model that cannot beat a dumb baseline is not ready to ship.

| Task | Simple baselines |
|------|------------------|
| Classification | Predict majority class; predict from one obvious feature rule |
| Regression | Predict training mean or median |
| Ranking-ish | Sort by a single strong feature |

```python
from sklearn.dummy import DummyClassifier

dummy = DummyClassifier(strategy="most_frequent")
dummy.fit(X_train, y_train)
print("Dummy F1:", f1_score(y_val, dummy.predict(X_val)))
```

Report **model vs dummy** in every experiment log.

## End-to-end sketch

```python
import pandas as pd
from sklearn.model_selection import train_test_split
from sklearn.compose import ColumnTransformer
from sklearn.pipeline import Pipeline
from sklearn.impute import SimpleImputer
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import f1_score
from sklearn.dummy import DummyClassifier

df = pd.read_csv("data.csv")
y = df["label"]
X = df.drop(columns=["label"])

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y
)

num_cols = X.select_dtypes(include="number").columns.tolist()
cat_cols = X.select_dtypes(exclude="number").columns.tolist()

prep = ColumnTransformer([
    ("num", Pipeline([
        ("imp", SimpleImputer(strategy="median")),
        ("sc", StandardScaler()),
    ]), num_cols),
    ("cat", Pipeline([
        ("imp", SimpleImputer(strategy="most_frequent")),
        ("oh", OneHotEncoder(handle_unknown="ignore")),
    ]), cat_cols),
])

pipe = Pipeline([
    ("prep", prep),
    ("clf", LogisticRegression(max_iter=2000, class_weight="balanced")),
])

pipe.fit(X_train, y_train)
pred = pipe.predict(X_test)

dummy = DummyClassifier(strategy="most_frequent")
dummy.fit(X_train, y_train)

print("Model F1", f1_score(y_test, pred))
print("Dummy F1", f1_score(y_test, dummy.predict(X_test)))
```

Treat the test print as **final**—do not go back and re-tune after peeking unless you re-split with a new holdout discipline.

## Saving models

```python
import joblib

joblib.dump(pipe, "model.joblib")
pipe2 = joblib.load("model.joblib")
```

Save the **whole pipeline**, not only the final estimator—production needs the same preprocessing.

## Experiment hygiene (lightweight)

Even before Part 5 (MLOps), keep a simple log:

| Field | Example |
|-------|---------|
| Date | 2026-09-23 |
| Data version | `data_v3.csv` hash or path |
| Split seed | 42 |
| Pipeline | logistic + median impute + scale + one-hot |
| Metric | F1 on val = 0.71; dummy = 0.40 |
| Notes | Removed `customer_id`; fixed leakage on `label_proxy` |

A markdown or CSV experiment log beats memory.

## Common workflow bugs

1. **Fitting the imputer on all data before split**
2. **One-hot encoding with categories that only appear in test**—use `handle_unknown="ignore"` and fit on train
3. **Tuning on test**
4. **Ignoring class imbalance** while celebrating accuracy
5. **Comparing models trained with different preprocessing bugs**

### Mini practice

List three steps in the sketch above that specifically prevent leakage. Check your answers against the “fit on train, transform elsewhere” rule.

## What is next

**[04-exercises-and-checklist.md](04-exercises-and-checklist.md)** — practice problems, a Part 2 capstone, and the Ready-for-Part-3 checklist.
