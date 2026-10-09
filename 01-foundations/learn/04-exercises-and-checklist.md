# Stage 1 — Exercises and “Ready for Stage 2” Checklist

Use these exercises after lessons 1.1–1.3. Attempt **Beginner** items first. **Stretch** items deepen understanding; skip them only if you are blocked, then return later.

Work in your virtual environment. Prefer scripts or a notebook—whichever you will actually finish.

---

## Exercises tied to Lesson 1.1 (Python)

### Beginner

1. **Environment check.** Create a venv, activate it, install `numpy`, and run a script that prints `numpy.__version__`.
2. **Fizz-style warm-up.** For numbers 1–30, print the number; if divisible by 3 print `ai`, if by 5 print `ml`, if by both print `aiml`.
3. **Word stats.** Write a function that takes a string and returns a dict with `n_chars`, `n_words`, and `n_unique_words` (case-insensitive).
4. **JSON round-trip.** Save a dict of hyperparameters to `hparams.json`, load it, change one value, save again.
5. **Safe parse.** Write `parse_int(text)` that returns an `int` or `None` if parsing fails (use `try` / `except ValueError`).

### Stretch

6. **Mini CLI.** Use `argparse` (stdlib) so `python summarize.py --path notes.txt` prints line count and word count.
7. **Class refactor.** Turn your word-stats function into a small `TextReport` class with a `summary()` method that returns a dict.
8. **Git branch practice.** Create a branch `practice/python`, commit your scripts, list branches, then switch back to your main branch.

---

## Exercises tied to Lesson 1.2 (Math for AI)

### Beginner

1. **Shapes.** Create `X` with shape `(5, 3)` of random numbers and `w` with shape `(3,)`. Compute `y = X @ w` and assert `y.shape == (5,)`.
2. **Dot product by hand.** For `a = [1, 2, 3]` and `b = [4, 5, 6]`, compute the dot product without NumPy, then verify with `np.dot`.
3. **Mean and variance.** Implement both from scratch; compare to NumPy on `[2, 4, 4, 6]`.
4. **Gradient step.** For `L(w) = (w - 3)**2`, start at `w = 0` and take 10 gradient steps with `lr = 0.1`. Plot or print `w` and `L` each step.

### Stretch

5. **Tiny regression.** Fit `y ≈ w x + b` on `x = [1,2,3,4]`, `y = [3,5,7,9]` with the MSE gradient descent sketch from lesson 1.2. Report final `w`, `b`, and RMSE.
6. **Bayes recount.** Change the disease prior from 1% to 10% in the lesson’s counting argument. What roughly happens to the posterior after a positive test?
7. **Bias vs variance journal.** In five sentences, describe a high-bias model and a high-variance model for predicting house prices from square footage alone.

---

## Exercises tied to Lesson 1.3 (Data literacy)

### Beginner

1. **Format ID.** For each of CSV, JSON, PNG, and free-text email body, say structured / semi-structured / unstructured and one typical AI task.
2. **Cleaning plan.** Given columns `age` (some nulls), `fare` (one value 99999), and duplicate passenger IDs, write a short cleaning plan before coding.
3. **Leakage hunt.** Explain why fitting a scaler on train+test together leaks information.
4. **Metric choice.** For cancer screening (rare positive, missing a case is terrible), which would you optimize first: precision or recall? Why?
5. **Baseline.** On an imbalanced binary dataset that is 90% class 0, what accuracy does “always predict 0” get? Why is that a bad north star?

### Stretch

6. **Manual metrics.** Implement accuracy, precision, recall, and F1 for lists `y_true` and `y_pred` without scikit-learn. Test on a tiny handmade example.
7. **Split discipline.** Load any small CSV; split indices into train/val/test; compute the median of a numeric column **on train only**; add a filled column for all splits using that median.
8. **Dataset brief.** For Titanic or a housing CSV, write a one-page brief: target, features, missingness, metric, baseline, and leakage risks.

---

## Stage 1 capstone

Build a small end-to-end script (one file is fine) that proves you can move data through a careful pipeline.

### Requirements

1. **Load** a CSV from disk (use a public dataset you download yourself, or a tiny CSV you create with at least 30 rows and a few columns).
2. **Clean**
   - Drop or impute missing values with a documented rule
   - Remove exact duplicate rows
   - Optionally clip or flag one outlier column
3. **Compute stats** on the cleaned training portion after a train/val/test or train/test split (split first!).
4. **Save** a `summary.json` that includes at least:
   - row counts before/after cleaning
   - missing-value counts per column (before cleaning)
   - mean/variance (or std) for one numeric column on the **train** split
   - the split sizes
5. **Print** a short human-readable report to the terminal.

### Optional stretch (if you install scikit-learn)

```bash
python -m pip install scikit-learn
```

- Train a tiny baseline (`LogisticRegression` or `LinearRegression` depending on the target)
- Fit preprocessing **only on train**
- Report one metric on the validation or test split
- Append that metric into `summary.json`

Official docs live at the scikit-learn documentation site if you need API details—prefer reading docs over random snippets.

### Capstone acceptance bar

You are done when another engineer can run your script in a fresh venv and get a `summary.json` plus clear console output without guessing your steps.

Suggested project layout:

```text
part1_capstone/
  data/           # your csv (or instructions to download)
  clean_and_summarize.py
  summary.json    # generated
  README.md       # how to run, cleaning rules, metric choice
```

---

## Checklist: Ready for Stage 2

Mark these when they are true for you (change `[ ]` to `[x]`):

### Python

- [ ] I can create a venv, install a package with pip, and run a script
- [ ] I can write functions using lists and dicts comfortably
- [ ] I can read/write JSON and text files
- [ ] I can read a Python traceback well enough to find the failing line
- [ ] I can make a git commit on a branch with a clear message

### Math intuition

- [ ] I can explain vectors, matrices, and the dot product with a tiny example
- [ ] I know what a gradient is for in training (direction to increase loss; we step opposite)
- [ ] I can compute mean and variance and explain bias vs variance in plain language
- [ ] I understand why train / validation / test splits exist

### Data literacy

- [ ] I can tell structured from unstructured data and name common formats
- [ ] I have a plan for missing values, duplicates, and outliers
- [ ] I can explain data leakage with at least one concrete example
- [ ] I can choose among accuracy, precision, recall, F1, and RMSE for a simple scenario
- [ ] I compare models to a dumb baseline before celebrating scores

### Capstone

- [ ] I completed the Stage 1 capstone (or an equivalent personal project)
- [ ] My `summary.json` and cleaning rules would make sense to a teammate

If most boxes are checked and the rest have a plan, you are ready.

---

## Suggested next step

Proceed to **Stage 2 — Classical ML**: supervised learning with scikit-learn-style workflows (train/val/test for real, baselines, trees and linear models, feature preprocessing, and proper evaluation).

Bring your Stage 1 habits with you. Classical ML will feel much easier if your Python, math intuition, and data hygiene are already in place.

When Stage 2 materials are available in this repo, start with that part’s overview file and work in order—just as you did here.
