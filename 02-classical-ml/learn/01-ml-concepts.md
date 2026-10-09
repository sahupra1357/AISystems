# Lesson 01 — Machine Learning Concepts

This lesson builds the vocabulary you will use for the rest of the course. Read slowly, draw diagrams if that helps, and do the mini practices. Algorithms come in the next lesson; here we focus on *ideas*.

## What is machine learning?

**Machine learning (ML)** is a family of techniques where a program improves at a task by learning from **data**, instead of relying only on hand-written rules.

Classic software:

```text
rules + input → output
```

Machine learning:

```text
data + learning algorithm → model
model + new input → prediction
```

You still write code—but much of the “logic” for mapping inputs to outputs is **estimated from examples**.

### Supervised learning

You have **features** \(X\) and a **label** \(y\) for each example. The goal is to learn a function \(\hat{y} = f(X)\) that predicts labels for new examples.

| Task type | Label | Examples |
|-----------|--------|----------|
| Classification | Discrete class | Spam vs not spam; disease yes/no; digit 0–9 |
| Regression | Continuous number | House price; temperature; delivery time |

### Unsupervised learning

You have features but **no labels**. Goals include finding structure: clusters, unusual points, or compressed representations.

| Task type | Goal | Examples |
|-----------|------|----------|
| Clustering | Group similar rows | Customer segments; topic groups |
| Dimensionality reduction | Fewer features that keep structure | Visualization; compression before another model |
| Anomaly detection (often semi-supervised) | Flag weird points | Fraud-like transactions |

### Semi-supervised and reinforcement (brief)

- **Semi-supervised:** many unlabeled examples plus a few labeled ones.
- **Reinforcement learning:** an agent learns by taking actions and receiving rewards over time (games, robotics, some recommendation systems).

This part focuses on **supervised** classical ML and a touch of **clustering**. You will meet deep unsupervised and generative ideas later.

## Features, labels, and examples

An **example** (row, instance, sample) is one observation.

A **feature** is a measurable property used as input (columns of \(X\)).

A **label** (target) is what you want to predict (\(y\)).

Example: predicting whether a loan defaults.

| Feature ideas | Label |
|---------------|--------|
| Income, credit score, loan amount, employment years | `defaulted` (0/1) |

### Feature engineering (high level)

Models only see numbers (or encoded categories). Feature engineering turns raw fields into useful inputs:

- Scaling numeric features (standardize or min–max)
- One-hot or ordinal encoding for categories
- Combining fields (`debt_to_income = debt / income`)
- Handling dates (day of week, month, days since event)

In classical ML, **good features often matter more than fancy models**. Deep learning can learn features from raw pixels or text; tabular work still rewards thoughtful columns.

### Mini practice — features

Pick a real-world prediction you care about (for example, “will this support ticket escalate?”). Write:

1. The label definition
2. Five candidate features
3. One feature that would likely **leak** the label (and must be excluded)

## Training, validation, and test (again, with teeth)

You met splits in Part 1. Here is how they drive classical ML decisions:

| Split | Used for | Rule |
|-------|----------|------|
| **Train** | Fit the model (and fit preprocessors) | Model can see these labels |
| **Validation** | Choose models, hyperparameters, features | Do **not** fit the final preprocessor using this data’s statistics mixed with train carelessly—fit on train, transform val |
| **Test** | Final honest estimate | Touch once (or rarely) at the end |

If you only have enough data for two splits, use **train + test** and prefer **cross-validation** on train for model selection (next lesson and lesson 03).

### Data leakage (classical ML edition)

Leakage means information from outside the training-time world sneaks into features or preprocessing.

Common classical leaks:

- Including a column that is a proxy for the label (future payment status when predicting default)
- Fitting a scaler or imputer on the **full** dataset before splitting
- Using test-set accuracy repeatedly to tune until it looks great (the test set becomes a second validation set)

**Fix:** split first; fit transforms on train only; keep a true holdout.

## Overfitting and underfitting

**Underfitting (high bias):** the model is too simple to capture the pattern. Train and val errors are both high.

**Overfitting (high variance):** the model memorizes training quirks and fails to generalize. Train error is low; val/test error is much higher.

```text
Too simple  →  underfit
Just right  →  good generalization
Too complex / too little data  →  overfit
```

### Ways to reduce overfitting

- More diverse training data
- Simpler models or fewer features
- **Regularization** (penalties on large weights; depth limits on trees)
- Dropout and early stopping (deep learning; Part 3)
- Cross-validation for honest model selection
- Ensembles that average many weak learners (often helps variance)

### Ways to reduce underfitting

- Richer features
- More flexible model family
- Longer training / better hyperparameters
- Fixing bugs in labels or feature pipelines

## Bias–variance tradeoff

In plain language:

- **Bias:** error from wrong assumptions (a linear model for a strongly curved relationship).
- **Variance:** error from sensitivity to the particular training sample (a deep tree that changes drastically if you remove a few rows).

Total expected error roughly decomposes into bias + variance + irreducible noise. You rarely eliminate all three; you **balance**.

Practical takeaway:

- Start with a **simple baseline**
- Increase complexity only while validation performance improves
- Prefer the simpler model when scores are tied

### Mini practice — bias vs variance

For house prices from square footage alone:

1. Describe a high-bias approach in one sentence.
2. Describe a high-variance approach in one sentence.
3. Name one change that would likely reduce variance.

## Regularization (intuition)

**Regularization** means adding a preference for simpler solutions while fitting.

For linear models, common penalties are:

- **L2 (Ridge):** shrink weights toward zero smoothly; keeps all features but smaller coefficients.
- **L1 (Lasso):** can drive some weights to exactly zero → feature selection effect.
- **Elastic Net:** mix of L1 and L2.

For trees:

- Limit `max_depth`, `min_samples_leaf`
- Use fewer features per split (random forests)
- Shrink learning rate (boosting)

You will tune regularization strength with validation or cross-validation—not by gut feel alone.

## The modeling loop

Every classical ML project repeats a loop:

1. **Define** the prediction task and success metric
2. **Collect / clean** data; document assumptions
3. **Split** honestly
4. **Baseline** (predict mean; predict majority class; simple linear model)
5. **Iterate** features and models
6. **Evaluate** on validation; adjust
7. **Lock** a candidate and score once on test
8. **Ship or reject** based on business criteria, not only leaderboard vanity

Lesson 03 turns this loop into scikit-learn code.

## Common failure modes

| Failure | Symptom | First checks |
|---------|---------|--------------|
| Leakage | Unrealistically high scores | Feature audit; preprocessing order |
| Wrong metric | Model “wins” but product fails | Align metric with cost of errors |
| Distribution shift | Val good, production bad | Compare train vs live feature stats |
| Tiny data | Huge score swings | Simpler models; stronger regularization; more data |
| Imbalance ignored | High accuracy, useless recall | Class weights; PR curves; resampling carefully |

## Key vocabulary checklist

Before lesson 02, make sure these words feel usable in a sentence:

- supervised / unsupervised
- classification / regression
- feature / label / example
- train / validation / test
- overfitting / underfitting
- bias / variance
- regularization
- baseline
- leakage

### Mini practice — teach it back

Explain to an imaginary teammate (or rubber duck) in under two minutes:

1. What supervised learning is
2. Why we hold out a test set
3. What overfitting looks like on a train vs val plot

## What is next

**[02-supervised-algorithms.md](02-supervised-algorithms.md)** introduces the workhorse models: linear and logistic regression, trees and forests, gradient boosting, k-NN, and k-means—with intuition first and light math second.
