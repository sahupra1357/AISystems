# Stage 2 — Classical ML: Overview

Welcome back. Stage 1 gave you Python, math intuition, and data hygiene. Stage 2 turns that into **classical machine learning**: models that learn patterns from tabular features without neural networks.

You will train models with **scikit-learn**, evaluate them honestly, and learn which algorithm fits which problem. These skills stay useful forever—even when deep learning or LLMs enter the picture, classical ML is often the right first (or only) tool.

## Goals for Stage 2

By the end of Stage 2, you should be able to:

- Explain supervised vs unsupervised learning, features, labels, and the bias–variance tradeoff
- Describe overfitting and name practical ways to fight it (more data, regularization, simpler models, cross-validation)
- Train and interpret linear regression, logistic regression, decision trees, random forests, and gradient boosting at a conceptual level
- Use k-NN for simple prediction and k-means for clustering, with clear limitations in mind
- Build a scikit-learn **Pipeline** with preprocessing, fit only on training data, and evaluate on held-out sets
- Choose metrics and baselines appropriate to the task
- Complete a small end-to-end classical ML project and pass the Ready-for-Stage-3 checklist

## Prerequisites

Complete **Stage 1 — Foundations**, or be equally comfortable with:

- Python: functions, lists/dicts, virtual environments, reading CSVs
- Math intuition: vectors, mean/variance, train/val/test, bias vs variance in plain language
- Data literacy: cleaning, leakage, and basic metrics (accuracy, precision/recall/F1, RMSE)

Install for this part (in your course venv):

```bash
python -m pip install numpy pandas scikit-learn matplotlib
```

## Suggested time

About **2–3 weeks** part-time:

| Week | Focus |
|------|--------|
| 1 | ML concepts (lesson 2.1); start supervised algorithms (lesson 2.2) |
| 2 | Finish algorithms; scikit-learn workflow (lesson 2.3) |
| 3 | Exercises, checklist, Stage 2 capstone |

If scikit-learn is new, give yourself the full three weeks. If you have used it before, spend more time on evaluation discipline and algorithm intuition.

## Order to study

1. **[01-ml-concepts.md](01-ml-concepts.md)** — Vocabulary and mental models: supervised/unsupervised, features, overfitting, bias–variance.
2. **[02-supervised-algorithms.md](02-supervised-algorithms.md)** — Core algorithms you will actually use.
3. **[03-scikit-learn-workflow.md](03-scikit-learn-workflow.md)** — Pipelines, CV, metrics, baselines—the professional loop.

Then complete **[04-exercises-and-checklist.md](04-exercises-and-checklist.md)** before Stage 3.

## Success criteria before Stage 3

You are ready for Deep Learning (Stage 3) when you can honestly check most of these:

- Explain supervised vs unsupervised learning with one example each
- Describe overfitting and name two ways to reduce it
- Train a Pipeline that scales features and fits a classifier, without leaking validation data
- Compare at least two models (plus a dumb baseline) using a metric you can justify
- Sketch when you would prefer a linear model vs a tree ensemble
- Finish the Stage 2 capstone (or a close variant)

When those feel true, open Stage 3. Classical ML is the bridge: deep learning will reuse the same split–train–evaluate mindset with different model machinery.
