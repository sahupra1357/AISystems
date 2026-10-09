# Lesson 1.2 — Math for AI (Practical Intuition)

This is not a mathematics textbook. The goal is **intuition you can use**: enough linear algebra, probability, calculus, and statistics to understand what models are doing when you train and evaluate them.

Keep a Python environment ready with NumPy:

```bash
python -m pip install numpy
```

## Why math matters for AI engineers

You can call libraries without math—for a while. Then you need to:

- Understand why a model underfits or overfits
- Read a loss curve and decide what to try next
- Choose metrics that match the product risk
- Spot when an evaluation is lying to you
- Talk clearly with researchers and with stakeholders

Math here is a **thinking tool**, not a ritual. Prefer a correct mental model over memorizing formulas.

## Linear algebra: vectors, matrices, and the dot product

### Vectors

A **vector** is an ordered list of numbers. In AI it often represents a data point, a set of weights, or an embedding.

```python
import numpy as np

# 3 features for one house: size (sqft/1000), bedrooms, age
x = np.array([1.5, 3.0, 10.0])
print(x.shape)  # (3,)
```

You can add vectors and scale them:

```python
a = np.array([1.0, 2.0])
b = np.array([3.0, 4.0])
print(a + b)      # [4. 6.]
print(2 * a)      # [2. 4.]
```

### Matrices

A **matrix** is a 2D grid of numbers. A dataset with `n` rows and `d` features is naturally an `n × d` matrix. A linear layer’s weights are also a matrix.

```python
# 3 houses, 2 features each
X = np.array([
    [1.5, 3.0],
    [2.0, 4.0],
    [1.0, 2.0],
])
print(X.shape)  # (3, 2)
```

### Dot product

The **dot product** of two vectors multiplies matching entries and sums them:

```python
w = np.array([0.5, 1.0, -0.1])  # weights
x = np.array([1.5, 3.0, 10.0])  # features
score = np.dot(w, x)
# 0.5*1.5 + 1.0*3.0 + (-0.1)*10.0 = 0.75 + 3.0 - 1.0 = 2.75
print(score)
```

A simple linear prediction is often just a weighted sum (dot product) plus a bias:

```python
bias = 0.2
prediction = np.dot(w, x) + bias
print(prediction)
```

Matrix–vector multiply applies that idea to many rows at once:

```python
W = np.array([0.5, 1.0])          # 2 weights
X = np.array([[1.5, 3.0],
              [2.0, 4.0],
              [1.0, 2.0]])
preds = X @ W + 0.2               # @ is matrix multiply
print(preds)
```

### Why this shows up everywhere

- Linear regression: `ŷ = Xw + b`
- Neural nets: repeated matrix multiplies and nonlinearities
- Similarity search: related to dot products / cosine similarity between embeddings

You do not need to derive eigenvalues today. You **do** need to be comfortable with shapes: if `X` is `(n, d)` and `w` is `(d,)`, then `X @ w` is `(n,)`.

### Mini practice — linear algebra

1. Create a vector `w` of length 2 and a matrix `X` with shape `(4, 2)`.
2. Compute `X @ w` and confirm the result has length 4.
3. Change one weight and observe how all predictions move.

## Probability: randomness, distributions, and Bayes (one plain example)

### Randomness and uncertainty

Models rarely output certainty. Classification scores, sampling in generative models, and noisy labels are all about **uncertainty**.

A **probability** is a number between 0 and 1 describing how likely an event is. A **distribution** describes probabilities over many possible outcomes (for example, heights of adults, or token choices in a language model).

Intuition, not formulas:

- A **uniform** distribution treats outcomes as equally likely (a fair die).
- A **normal** (bell curve) distribution clusters around a mean—measurement noise often looks roughly like this.
- In ML we often assume noise or priors so that math and algorithms stay tractable.

```python
import numpy as np

rng = np.random.default_rng(42)
samples = rng.normal(loc=0.0, scale=1.0, size=5)
print(samples)
```

### Bayes in one plain example

**Bayes’ idea:** start with a prior belief, observe evidence, update to a posterior belief.

Plain example: rare disease screening.

- 1% of people have the disease (prior).
- The test is 99% sensitive: if you have it, it detects it 99% of the time.
- The test has a 2% false positive rate: if you do not have it, it still says positive 2% of the time.

You test positive. Are you almost certainly sick? **No.**

Rough counting with 10,000 people:

- About 100 are sick; ~99 of them test positive.
- About 9,900 are healthy; ~198 of them test positive (2%).
- Total positives ≈ 99 + 198 = 297.
- Of those, only 99 are truly sick ≈ **33%**.

So a positive test moved you from 1% to about 33%—important, but not certainty. AI engineers meet the same pattern when class imbalance meets a “good” detector: **base rates matter**.

### Mini practice — probability

Explain in two sentences why a model with 99% accuracy can still be useless for a class that appears in 0.5% of rows.

## Calculus: derivatives, gradients, and training

### Derivative as slope

If `loss` depends on a parameter `w`, the **derivative** `d(loss)/dw` is the slope: how fast loss changes when you nudge `w`.

- Positive slope: increasing `w` increases loss → decrease `w` to improve.
- Negative slope: increasing `w` decreases loss → increase `w`.
- Near zero: you are near a flat spot / local optimum.

```python
# Toy loss: L(w) = (w - 3)^2  → minimum at w = 3
def loss(w):
    return (w - 3) ** 2


def dloss_dw(w):
    return 2 * (w - 3)


w = 0.0
for step in range(8):
    w = w - 0.1 * dloss_dw(w)  # gradient descent step
    print(step, w, loss(w))
```

That loop is the spirit of **gradient descent**: move parameters opposite the slope to reduce loss.

### Gradient as direction of steepest change

With many parameters, the **gradient** is the vector of partial derivatives. It points toward the steepest increase of the loss. Training steps move **against** the gradient.

In deep learning frameworks, you almost never write derivatives by hand—**autograd** computes them. Your job is to understand:

- Loss measures “how wrong”
- Gradients tell each weight how to change
- Learning rate scales the step
- Bad rates or bad data → unstable or useless training

### Link to training

A typical supervised loop:

1. Predict with current weights
2. Compute loss vs labels
3. Backpropagate gradients
4. Update weights a little
5. Repeat

You will implement this mentally long before you implement it in PyTorch. The calculus you need first is “slope” and “step opposite the slope.”

### Mini practice — calculus intuition

For `L(w) = (w - 3)^2`, without running code: if `w = 5`, is the derivative positive or negative? Should the next update increase or decrease `w`?

## Statistics: mean, variance, bias vs variance, splits

### Mean and variance

- **Mean**: typical central value.
- **Variance** (and standard deviation): how spread out values are.

```python
import numpy as np

xs = np.array([2.0, 4.0, 4.0, 6.0])
mean = xs.mean()
# population variance for intuition; sample variance uses ddof=1
variance = xs.var()
print(mean, variance)
```

From scratch:

```python
def mean(xs):
    return sum(xs) / len(xs)


def variance(xs):
    m = mean(xs)
    return sum((x - m) ** 2 for x in xs) / len(xs)
```

### Bias vs variance (model error intuition)

- **Bias**: error from a model that is too simple to capture the pattern (underfitting). A straight line fitting a curve has high bias.
- **Variance**: error from a model that is too sensitive to the training sample (overfitting). A wiggly curve that memorizes noise has high variance.

You want a balance: flexible enough to learn signal, constrained enough to ignore noise. Regularization, more data, and simpler models are tools for that balance—you will use them explicitly in Stage 2.

### Train / validation / test intuition

| Split | Purpose |
|-------|---------|
| **Train** | Fit parameters |
| **Validation** | Tune choices (model type, hyperparameters) |
| **Test** | Final honest estimate of performance |

If you peek at the test set to make decisions, it stops being a test set. Treat it like a sealed exam.

### Mini practice — statistics

Compute mean and variance of `[10, 12, 9, 11, 18]` by hand or with your functions. Which point pulls the mean the most?

## What you can skip for now

You can postpone (until a later course or need arises):

- Formal proofs and epsilon-delta definitions
- Heavy measure-theoretic probability
- Manual multivariable derivative drills beyond gradient intuition
- Advanced linear algebra (SVD proofs, etc.)—know that SVD/PCA exist; details can wait
- Information theory beyond “entropy measures uncertainty” (optional later for LLMs)

Curiosity is good; blocking yourself on textbook completeness is not.

## Mini practice — mean/variance and tiny linear regression outline

### A. Implement mean and variance

Write `mean` and `variance` without NumPy. Compare to `np.mean` / `np.var` on the same list.

### B. Tiny linear regression from scratch (outline)

Goal: fit `y ≈ w * x + b` on a few points.

1. Make toy data: `x = [1, 2, 3, 4]`, `y = [3, 5, 7, 9]` (true line roughly `y = 2x + 1`).
2. Initialize `w` and `b` to small values (for example 0).
3. For many steps:
   - Predict `y_hat = w * x_i + b` for each i
   - Compute mean squared error loss
   - Compute gradients of loss w.r.t. `w` and `b` (or use finite differences if stuck)
   - Update `w` and `b` with a small learning rate
4. Print final `w`, `b`, and loss

Skeleton:

```python
xs = [1.0, 2.0, 3.0, 4.0]
ys = [3.0, 5.0, 7.0, 9.0]
w, b = 0.0, 0.0
lr = 0.01

for step in range(200):
    # predictions
    y_hats = [w * x + b for x in xs]
    # gradients for MSE: dL/dw, dL/db
    errors = [y_hat - y for y_hat, y in zip(y_hats, ys)]
    dL_dw = sum(2 * e * x for e, x in zip(errors, xs)) / len(xs)
    dL_db = sum(2 * e for e in errors) / len(xs)
    w -= lr * dL_dw
    b -= lr * dL_db

print(w, b)
```

You should land near `w ≈ 2`, `b ≈ 1`. This is gradient descent on a tiny model—the same idea scales to neural nets.

## Section recap

- Vectors and matrices are the containers of modern ML; the dot product is the core linear operation
- Probability keeps you honest about uncertainty and base rates
- Derivatives and gradients explain how training updates weights
- Mean, variance, bias/variance, and splits explain evaluation and generalization

Next: [03-data-literacy.md](03-data-literacy.md). Strong math intuition paired with weak data habits still produces failing systems—data literacy is the other half of foundations.
