# Part 2 — Exercises and “Ready for Part 3” Checklist

Use these after lessons 01–03. Attempt **Beginner** items first. **Stretch** items deepen skill; return to them if you skip.

Work in your course virtual environment with `numpy`, `pandas`, and `scikit-learn` installed.

---

## Exercises tied to Lesson 01 (ML concepts)

### Beginner

1. **Task labeling.** For each scenario, say supervised classification, supervised regression, or unsupervised: (a) predict house price; (b) group shoppers into segments with no labels; (c) detect spam email.
2. **Leakage story.** Invent a feature that would leak for “predict hospital readmission within 30 days.” Explain why.
3. **Overfit sketch.** In 4–6 sentences, describe how train vs validation loss curves look for underfit, good fit, and overfit.
4. **Bias–variance.** Give one high-bias and one high-variance model idea for predicting exam scores from hours studied.

### Stretch

5. **Metric memo.** Write a half-page memo to a product manager explaining when you would optimize recall vs precision for a fraud detector.
6. **Regularization journal.** Explain L1 vs L2 regularization as if to a teammate who knows linear regression but not penalties.

---

## Exercises tied to Lesson 02 (Algorithms)

### Beginner

1. **Linear vs tree.** For predicting electricity usage from temperature and hour-of-day, which model family would you try first and why?
2. **Logistic threshold.** Given probabilities `[0.1, 0.4, 0.6, 0.9]` and labels `[0, 0, 1, 1]`, compute accuracy at thresholds 0.5 and 0.7 by hand.
3. **Forest intuition.** Why might averaging many deep trees generalize better than one deep tree?
4. **k-NN scaling.** Why does k-NN need feature scaling while a decision tree often does not?
5. **k-Means choice.** List two practical ways to choose \(k\), and one reason domain knowledge still matters.

### Stretch

6. **Boosting vs bagging.** In your own words, contrast random forests (parallel bagging-style) with gradient boosting (sequential).
7. **Tiny bake-off.** On a built-in dataset (`sklearn.datasets.load_breast_cancer` or `load_diabetes`), compare logistic/linear regression vs random forest with a shared split. Report one metric and a short recommendation.

---

## Exercises tied to Lesson 03 (scikit-learn workflow)

### Beginner

1. **Pipeline first.** Build a Pipeline with `SimpleImputer` + `StandardScaler` + `LogisticRegression` on a numeric-only slice of a dataset. Fit on train; score on val.
2. **Dummy baseline.** Compare your pipeline’s metric to `DummyClassifier(strategy="most_frequent")` or `DummyRegressor`.
3. **Confusion matrix.** Print a confusion matrix and explain each cell in one sentence.
4. **CV score.** Run `cross_val_score` with `cv=5` on the training portion; report mean and std.
5. **Save/load.** `joblib.dump` your pipeline and reload it; assert predictions match on a small batch.

### Stretch

6. **ColumnTransformer.** Add a categorical column path with `OneHotEncoder(handle_unknown="ignore")`.
7. **Grid search.** Tune one or two hyperparameters with `GridSearchCV` inside a pipeline; document best params and val score.
8. **Leakage fix review.** Take a buggy script that scales before splitting; rewrite it correctly and show both val scores (expect the buggy one to look unrealistically strong sometimes).

---

## Part 2 capstone

Build a small **tabular classification or regression** project that proves you can run a professional classical ML loop.

### Requirements

1. **Data.** Use a public tabular dataset you download yourself (for example from scikit-learn’s loaders, or a well-known open CSV such as Titanic / housing—document the source). At least a few hundred rows preferred.
2. **Problem statement.** In `README.md`, write: prediction target, business-ish motivation, metric choice, and leakage risks.
3. **Splits.** Train/val/test or train/test + CV on train. Fix a random seed.
4. **Baseline.** Include a dummy model and at least **two** real models (for example logistic/linear and a forest or histogram gradient boosting).
5. **Pipeline.** Preprocessing must live in a `Pipeline` / `ColumnTransformer` fitted without leakage.
6. **Report.** Save `report.json` (or markdown) with:
   - dataset name and row counts per split
   - metric for dummy + each model on validation (and final test for the chosen model)
   - selected hyperparameters
   - 3–5 bullet “what I would try next”
7. **Reproducibility.** A teammate can create a venv, install from `requirements.txt`, and run one command to reproduce the report.

### Suggested layout

```text
part2_capstone/
  data/                 # or scripts/download_data.py
  train_eval.py
  requirements.txt
  report.json
  README.md
```

### Capstone acceptance bar

Someone else can reproduce your numbers within normal random variation and understand why you picked the winning model.

---

## Checklist: Ready for Part 3

Mark these when true (`[ ]` → `[x]`):

### Concepts

- [ ] I can explain supervised vs unsupervised learning with examples
- [ ] I can describe overfitting and name at least two remedies
- [ ] I can explain bias vs variance in plain language
- [ ] I can identify a simple leakage mistake in a pipeline sketch

### Algorithms

- [ ] I know when to reach for linear/logistic models vs tree ensembles
- [ ] I can describe random forests and gradient boosting at a high level
- [ ] I know why k-NN and k-Means need scaling

### Workflow

- [ ] I can build a scikit-learn Pipeline with preprocessing + model
- [ ] I evaluate with a metric I can justify and compare to a dummy baseline
- [ ] I use cross-validation or a validation set for model selection
- [ ] I keep the test set honest

### Capstone

- [ ] I finished the Part 2 capstone (or equivalent)
- [ ] My README and report would make sense to a teammate

If most boxes are checked, you are ready for neural networks.

---

## Suggested next step

Proceed to **Part 3 — Deep Learning**: neural net basics, a PyTorch training loop, and an introduction to CNNs and Transformers.

You will reuse everything here—splits, metrics, baselines, and overfitting—on tensors instead of `DataFrame` columns.
