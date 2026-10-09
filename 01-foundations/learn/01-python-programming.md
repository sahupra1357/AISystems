# Lesson 1.1 — Python Programming for AI Engineers

This lesson starts from zero. Read each section, type the examples, and do the mini practices. You do not need to memorize every detail; you need to become comfortable reading and changing Python code.

## Why Python for AI

Python is the default language for most AI and machine learning work because:

- The ecosystem is rich: NumPy, pandas, scikit-learn, PyTorch, and many others
- Syntax is readable, so you spend more time on ideas and less on ceremony
- Notebooks and scripts both work well for exploration and for production glue code
- Almost every tutorial, course, and library example you will meet is in Python

Other languages matter later (for example C++ or Rust for performance, SQL for data warehouses). For foundations, Python is enough.

## Setup

### Install Python 3

Use **Python 3.10+** if you can. Check what you have:

```bash
python3 --version
```

If that fails, install Python from the official site at python.org, or use your system package manager. On some Windows installs the command is `python` instead of `python3`.

### Virtual environments

A **virtual environment** isolates project packages so one project’s libraries do not break another’s.

```bash
python3 -m venv .venv
```

Activate it:

```bash
# macOS / Linux
source .venv/bin/activate

# Windows (Command Prompt)
.venv\Scripts\activate.bat
```

When active, your prompt often shows `(.venv)`. Deactivate with `deactivate`.

**Brief note on `uv`:** [uv](https://github.com/astral-sh/uv) is a fast modern tool for creating environments and installing packages. If your team uses it, follow their docs. For this course, classic `venv` + `pip` is fine.

### pip and installing packages

With the environment active:

```bash
python -m pip install --upgrade pip
python -m pip install numpy pandas
```

`python -m pip` uses the pip tied to the current Python, which avoids installing into the wrong interpreter.

### Running scripts

Create a file `hello.py`:

```python
print("Hello, AI engineer")
```

Run it:

```bash
python hello.py
```

You can also start an interactive shell with `python` and type expressions line by line. Use scripts for anything you want to keep.

### Mini practice — setup

1. Create a folder for this course and a virtual environment inside it.
2. Activate the environment and install `numpy`.
3. Write and run a script that prints your name and today’s goal in one sentence.

## Variables, types, and operators

A **variable** names a value in memory.

```python
name = "Ada"
age = 28
height_m = 1.72
is_student = True

print(name, age, height_m, is_student)
print(type(name), type(age), type(height_m), type(is_student))
```

Common built-in types:

| Type | Example | Use |
|------|---------|-----|
| `str` | `"hello"` | Text |
| `int` | `42` | Whole numbers |
| `float` | `3.14` | Decimals |
| `bool` | `True` / `False` | Yes/no flags |
| `NoneType` | `None` | Missing / empty value |

### Operators

```python
a = 10
b = 3

print(a + b)   # 13
print(a - b)   # 7
print(a * b)   # 30
print(a / b)   # 3.333... (float division)
print(a // b)  # 3 (integer division)
print(a % b)   # 1 (remainder)
print(a ** b)  # 1000 (power)

print(a > b)   # True
print(a == b)  # False
print(a != b)  # True

ready = True
print(ready and age > 18)
print(not ready)
```

Strings concatenate with `+` or f-strings:

```python
model = "baseline"
score = 0.91
print(f"Model {model} scored {score:.2f}")
```

### Mini practice — variables

Create variables for a tiny experiment: model name, training rows, and accuracy. Print a one-line summary with an f-string.

## Control flow

### if / elif / else

```python
accuracy = 0.82

if accuracy >= 0.9:
    print("Excellent")
elif accuracy >= 0.75:
    print("Good enough to ship with monitoring")
else:
    print("Needs more work")
```

Indentation (usually 4 spaces) defines blocks. Be consistent.

### for loops

```python
metrics = ["accuracy", "precision", "recall"]
for m in metrics:
    print(m)

for i in range(3):
    print(i)  # 0, 1, 2
```

### while loops

```python
n = 3
while n > 0:
    print(n)
    n -= 1
```

Prefer `for` when you know the sequence. Use `while` when you wait for a condition.

### Mini practice — control flow

Loop over a list of accuracies `[0.5, 0.8, 0.95]`. For each value, print `"fail"`, `"ok"`, or `"great"` using thresholds you choose.

## Functions and simple classes

### Functions

Functions package reusable behavior.

```python
def accuracy(correct, total):
    """Return accuracy as a float between 0 and 1."""
    if total == 0:
        return 0.0
    return correct / total


print(accuracy(90, 100))
```

Arguments can have defaults:

```python
def greet(name, greeting="Hello"):
    return f"{greeting}, {name}"


print(greet("Pradeep"))
print(greet("Pradeep", greeting="Welcome"))
```

### Simple classes

A **class** groups data and behavior. You will meet classes constantly in ML libraries.

```python
class Experiment:
    def __init__(self, name, score):
        self.name = name
        self.score = score

    def summary(self):
        return f"{self.name}: {self.score:.3f}"


exp = Experiment("logistic-regression", 0.874)
print(exp.summary())
```

`__init__` runs when you create an instance. `self` refers to that instance.

### Mini practice — functions

Write `mean(numbers)` that returns the average of a list of floats. Call it on `[1.0, 2.0, 3.0, 4.0]`.

## Lists, dicts, sets, tuples, and comprehensions

### Lists

Ordered, mutable sequences.

```python
losses = [0.9, 0.7, 0.5]
losses.append(0.4)
print(losses[0], losses[-1])
print(len(losses))
```

### Tuples

Ordered, **immutable** sequences. Good for fixed pairs.

```python
point = (3, 4)
# point[0] = 5  # would raise TypeError
```

### Dicts

Key → value maps. Extremely common for configs and JSON-like data.

```python
row = {"age": 29, "city": "Austin", "label": 1}
print(row["age"])
row["city"] = "Boston"
print(row.get("missing", "default"))
```

### Sets

Unordered unique items. Useful for membership checks and deduplication.

```python
labels = {0, 1, 1, 0, 1}
print(labels)  # {0, 1}
print(1 in labels)
```

### Comprehensions

Compact loops that build collections.

```python
nums = [1, 2, 3, 4, 5]
squares = [n * n for n in nums]
evens = [n for n in nums if n % 2 == 0]
lookup = {n: n * n for n in nums}
print(squares, evens, lookup)
```

### Mini practice — collections

Given `preds = [0, 1, 1, 0, 1]` and `truth = [0, 1, 0, 0, 1]`, count how many predictions match the truth (without libraries).

## Files and JSON

### Text files

```python
path = "notes.txt"

with open(path, "w", encoding="utf-8") as f:
    f.write("First line\n")
    f.write("Second line\n")

with open(path, "r", encoding="utf-8") as f:
    content = f.read()
print(content)
```

Always prefer `with open(...)` so the file closes cleanly.

### JSON

JSON is the lingua franca of configs, APIs, and experiment logs.

```python
import json

payload = {
    "model": "baseline",
    "metrics": {"accuracy": 0.88, "f1": 0.84},
    "seed": 42,
}

with open("run.json", "w", encoding="utf-8") as f:
    json.dump(payload, f, indent=2)

with open("run.json", "r", encoding="utf-8") as f:
    loaded = json.load(f)

print(loaded["metrics"]["accuracy"])
```

`json.dumps` / `json.loads` work on strings; `json.dump` / `json.load` work with files.

### Mini practice — files

Write a dict of three hyperparameters to `hparams.json`, read it back, and print one value.

## Errors and stack traces

Errors are normal. Learning to **read** them is a core skill.

```python
def divide(a, b):
    return a / b


# divide(1, 0)  # ZeroDivisionError
```

A stack trace shows:

1. The exception type and message (`ZeroDivisionError: division by zero`)
2. The call chain from top (outer) to bottom (where it blew up)
3. File names and line numbers

Catch only what you can handle:

```python
def safe_divide(a, b):
    try:
        return a / b
    except ZeroDivisionError:
        print("Cannot divide by zero; returning None")
        return None


print(safe_divide(10, 2))
print(safe_divide(10, 0))
```

Do not blanket-catch all exceptions while learning—let crashes teach you. When something fails, read the **last** few lines of the traceback first, then work upward.

### Mini practice — errors

Intentionally open a file that does not exist. Read the `FileNotFoundError` traceback and note the line number Python reports.

## Git basics (high level)

Git tracks versions of your code. You need only a small daily vocabulary at first.

| Command | Purpose |
|---------|---------|
| `git clone <url>` | Copy a remote repository to your machine |
| `git status` | See what changed |
| `git add <file>` | Stage a file for commit |
| `git commit -m "message"` | Save a snapshot with a message |
| `git branch` | List branches |
| `git checkout -b name` or `git switch -c name` | Create and switch to a new branch |

Typical loop:

```bash
git status
git add hello.py
git commit -m "Add hello script"
```

Use clear commit messages. Branch when you try a risky change so `main` stays calm. Hosting (GitHub, GitLab, etc.) comes later; local commits already help you undo mistakes.

### Mini practice — git

In your course folder, run `git init` (if it is not already a repo), add one file, and make a commit with a short message.

## Libraries preview: NumPy and pandas

You will use these heavily in Stage 2 and beyond. For now, **install and import** them so setup is not a blocker later.

```bash
python -m pip install numpy pandas
```

```python
import numpy as np
import pandas as pd

arr = np.array([1.0, 2.0, 3.0])
print(arr.mean())

df = pd.DataFrame({"x": [1, 2, 3], "y": [2, 4, 6]})
print(df.describe())
```

Deep NumPy and pandas usage appears in later lessons. If imports fail, fix the virtual environment before continuing.

## Putting it together

Here is a small script that ties several ideas:

```python
import json
from pathlib import Path


def summarize_scores(scores):
    if not scores:
        return {"count": 0, "mean": None}
    return {
        "count": len(scores),
        "mean": sum(scores) / len(scores),
        "min": min(scores),
        "max": max(scores),
    }


def main():
    scores = [0.71, 0.76, 0.80, 0.79]
    summary = summarize_scores(scores)
    out = Path("score_summary.json")
    out.write_text(json.dumps(summary, indent=2), encoding="utf-8")
    print(f"Wrote {out.resolve()}")


if __name__ == "__main__":
    main()
```

`if __name__ == "__main__":` means “run `main()` only when this file is executed directly,” which is a common script pattern.

## Section recap

You covered:

- Why Python dominates AI workflows
- Environments, pip, and running scripts
- Core syntax: types, control flow, functions, classes
- Collections and comprehensions
- Files, JSON, errors, and Git basics
- A first glimpse of NumPy and pandas

Next: [02-math-for-ai.md](02-math-for-ai.md) for the math intuition behind learning algorithms. Come back to this lesson whenever syntax feels rusty.
