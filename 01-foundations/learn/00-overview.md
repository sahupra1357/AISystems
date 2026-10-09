# Stage 1 — Foundations: Overview

Welcome. This part is the ground floor of becoming an AI engineer. You will not train neural nets yet. You will build the habits and language that make later work make sense.

## Goals for Stage 1

By the end of Stage 1, you should be able to:

- Write and run small Python programs using variables, control flow, functions, collections, files, and JSON
- Create a virtual environment, install packages with `pip`, and read a basic stack trace
- Use Git at a high level: clone, status, add, commit, and branch
- Explain vectors, matrices, and the idea of a gradient in plain language, and run tiny NumPy examples
- Reason about mean, variance, bias vs variance, and train/validation/test splits
- Clean a simple tabular dataset: missing values, duplicates, outliers
- Choose sensible evaluation metrics and avoid obvious data leakage
- Complete a small end-to-end practice: load a CSV, clean it, compute stats, save a JSON summary

## Prerequisites

- **Motivation** to learn by doing
- A **computer** where you can install Python 3 (Windows, macOS, or Linux)
- Optional but helpful: comfort with opening a terminal / command prompt

You do **not** need prior programming, calculus courses, or a math degree. We build intuition first and add only the math that AI engineers use daily.

## Suggested time

About **2–4 weeks** part-time:

| Week | Focus |
|------|--------|
| 1 | Python programming (lesson 1.1), lots of typing practice |
| 2 | Finish Python; start math for AI (lesson 1.2) |
| 3 | Finish math; data literacy (lesson 1.3) |
| 4 | Exercises, checklist, Stage 1 capstone |

If you already know some Python, spend more time on math intuition and data literacy. If Python is new, give yourself the full four weeks without rushing.

## Order to study

Study the three lessons **in this order**:

1. **[01-python-programming.md](01-python-programming.md)** — Your daily tool. Without Python, the rest stays abstract.
2. **[02-math-for-ai.md](02-math-for-ai.md)** — Enough linear algebra, probability, calculus, and statistics to follow model training and evaluation.
3. **[03-data-literacy.md](03-data-literacy.md)** — How real projects succeed or fail: data quality, splits, metrics, honest evaluation.

Then complete **[04-exercises-and-checklist.md](04-exercises-and-checklist.md)** before moving to Stage 2.

## Success criteria before Stage 2

You are ready for Classical ML (Stage 2) when you can honestly check most of these:

- Run a Python script from a virtual environment without copy-pasting commands you do not understand
- Write a function that reads a CSV (or JSON), transforms a list or dict, and writes a result file
- Explain in one or two sentences what a vector, a matrix, a derivative, and a train/val/test split are for
- Name when you would prefer accuracy vs F1 vs RMSE
- Spot a simple data-leakage mistake (for example, normalizing using the full dataset before splitting)
- Finish the Stage 1 capstone exercise (or a close variant)

When those feel true, open Stage 2 (Classical ML). If not, revisit the weak lesson and the exercises—foundations are not a race.
