# Lesson 02 — Supervised Algorithms (and a Cluster Friend)

This lesson surveys the algorithms you will use most often in classical ML. You do not need to derive every equation. You need intuition: **what the model assumes**, **what it is good at**, and **what breaks it**.

We will use scikit-learn-shaped APIs in sketches. Full pipelines come in lesson 03.

## Linear regression

**Job:** predict a continuous \(y\) as a weighted sum of features plus an intercept.

\[
\hat{y} = w_1 x_1 + w_2 x_2 + \cdots + w_d x_d + b
\]

### Intuition

Each feature gets a coefficient. Holding other features fixed, increasing \(x_j\) by 1 changes the prediction by \(w_j\) (for linear features).

### When it shines

- Relationships are roughly linear (or you engineered features to make them so)
- You want **interpretable** coefficients
- You need a strong, fast baseline

### When it struggles

- Strong non-linear interactions you did not encode
- Heavy outliers (ordinary least squares is sensitive)—consider robust variants or transforms
- Multicollinearity (correlated features) makes individual coefficients unstable—regularization helps

### Regularized cousins

- **Ridge** (L2): shrinks coefficients; good default when many correlated features
- **Lasso** (L1): can zero out coefficients
- **Elastic Net:** combine both

### Tiny sketch

```python
from sklearn.linear_model import LinearRegression, Ridge

model = Ridge(alpha=1.0)
model.fit(X_train, y_train)
preds = model.predict(X_val)
```

### Mini practice

On paper, for \(\hat{y} = 2x + 10\), what is \(\hat{y}\) when \(x = 3\)? What does the intercept mean when \(x = 0\)?

## Logistic regression

Despite the name, this is a **classifier** (usually binary; multinomial extension exists).

It models the **probability** of the positive class with a sigmoid of a linear score:

\[
P(y=1 \mid x) = \sigma(w \cdot x + b)
\]

where \(\sigma(z) = 1 / (1 + e^{-z})\).

You threshold the probability (often at 0.5, sometimes tuned) to get a class label.

### When it shines

- Linearly separable (or nearly) decision boundaries in feature space
- Need calibrated-ish probabilities and interpretable weights
- Strong baseline for binary problems

### Tips

- Scale features (standardization) before fitting—coefficients and optimization behave better
- Use `class_weight="balanced"` when classes are imbalanced
- Regularization (`C` in scikit-learn is inverse strength) is important

```python
from sklearn.linear_model import LogisticRegression

clf = LogisticRegression(max_iter=1000, class_weight="balanced")
clf.fit(X_train, y_train)
proba = clf.predict_proba(X_val)[:, 1]
```

### Mini practice

If the model outputs probability 0.91 for “fraud,” and your threshold is 0.5, what class do you predict? If false positives are very expensive, would you raise or lower the threshold?

## Decision trees

A **decision tree** asks a sequence of yes/no questions on features (“Is age ≤ 30?”) and ends in a leaf prediction (class votes or mean value).

### Intuition

Trees carve the feature space into rectangles (axis-aligned splits). They capture non-linearities and interactions automatically.

### Pros

- Interpretable (small trees)
- Handle mixed feature types with little preprocessing (still be careful with high-cardinality categories)
- No need to scale features for the split logic itself

### Cons

- Single trees **overfit** easily if grown deep
- Unstable: small data changes can reshape the tree
- Axis-aligned splits can be inefficient for diagonal boundaries

### Key hyperparameters

| Parameter | Effect |
|-----------|--------|
| `max_depth` | Limits complexity |
| `min_samples_leaf` | Requires more data per leaf → smoother |
| `max_features` | Restricts features considered at each split |

```python
from sklearn.tree import DecisionTreeClassifier

tree = DecisionTreeClassifier(max_depth=4, random_state=42)
tree.fit(X_train, y_train)
```

## Random forests

A **random forest** trains many trees on bootstrap samples of the data, with random feature subsets at splits, then **averages** (regression) or **votes** (classification).

### Why it works

Individual trees overfit in different ways; averaging reduces variance. Random feature subsets decorrelate trees.

### When it shines

- Tabular data with non-linearities
- Minimal tuning to get a strong baseline (still tune a bit)
- Feature importance heuristics (use carefully; not causal)

### Watch-outs

- Less interpretable than one small tree
- Can be large in memory
- Extrapolation beyond training ranges is still weak (like most tree methods)

```python
from sklearn.ensemble import RandomForestClassifier

rf = RandomForestClassifier(
    n_estimators=300,
    max_depth=None,
    min_samples_leaf=2,
    n_jobs=-1,
    random_state=42,
)
rf.fit(X_train, y_train)
```

## Gradient boosting

**Gradient boosting** builds trees **sequentially**. Each new tree fits the remaining errors (gradients of the loss) of the current ensemble. Final prediction is a weighted sum of trees.

Popular implementations you will meet in the wild: scikit-learn’s `HistGradientBoosting*`, plus libraries such as XGBoost, LightGBM, and CatBoost. For this course, scikit-learn’s histogram gradient boosting is enough to learn the ideas.

### When it shines

- Often **state of the art** on medium-sized tabular problems
- Handles non-linearities and feature interactions well

### When to be careful

- Easier to overfit than random forests if you boost too long with a high learning rate
- Needs more careful validation
- Training can be slower to tune

### Knobs that matter

| Knob | Role |
|------|------|
| Learning rate | Smaller → need more trees, usually better generalization |
| Number of trees / iterations | Capacity; use early stopping on validation |
| Max depth / leaf constraints | Tree complexity |
| Subsampling | Rows/features randomness → regularization |

```python
from sklearn.ensemble import HistGradientBoostingClassifier

gb = HistGradientBoostingClassifier(max_depth=6, learning_rate=0.05)
gb.fit(X_train, y_train)
```

### Trees vs linear models — quick chooser

| Prefer linear / logistic | Prefer trees / boosting |
|--------------------------|-------------------------|
| Need coefficients for stakeholders | Complex non-linear tabular patterns |
| Very high-dimensional sparse text (with right featurization) | Mixed numeric + categorical tabular |
| Extrapolation along a known trend | Sharp interactions and thresholds |

Often you **try both** with the same pipeline discipline.

## k-Nearest Neighbors (k-NN)

**k-NN** predicts by looking at the \(k\) closest training examples in feature space (majority vote or average).

### Intuition

“Who looks like me in the training set?”

### Pros

- Simple mental model; non-parametric
- Can work well in low dimensions with good distance metrics

### Cons

- Needs **feature scaling**
- Slow and memory-heavy on large datasets (stores training data)
- Suffers in high dimensions (distances concentrate)—the “curse of dimensionality”
- Sensitive to irrelevant features

```python
from sklearn.neighbors import KNeighborsClassifier
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline

knn = Pipeline([
    ("scale", StandardScaler()),
    ("knn", KNeighborsClassifier(n_neighbors=5)),
])
knn.fit(X_train, y_train)
```

### Mini practice

If \(k=1\), what overfitting risk do you expect compared to a large \(k\)?

## k-Means clustering (unsupervised)

**k-Means** partitions examples into \(k\) clusters by alternating:

1. Assign each point to the nearest centroid
2. Recompute centroids as the mean of assigned points

Until (roughly) stable.

### Uses

- Customer segmentation exploration
- Pseudolabels for later supervised steps (carefully)
- Vector quantization / compression ideas

### Limitations

- You must choose \(k\)
- Assumes spherical-ish clusters of similar scale—**scale features**
- Sensitive to initialization (run multiple times; scikit-learn does this)
- Not a density model; odd shapes confuse it

```python
from sklearn.cluster import KMeans
from sklearn.preprocessing import StandardScaler

X_scaled = StandardScaler().fit_transform(X)
km = KMeans(n_clusters=4, n_init=10, random_state=42)
labels = km.fit_predict(X_scaled)
```

Choosing \(k\): try an elbow plot of inertia, silhouette scores, or—best—domain judgment validated by downstream usefulness.

## Algorithm cheat sheet

| Algorithm | Supervised? | Scales features? | Typical use |
|-----------|-------------|------------------|-------------|
| Linear regression | Yes (reg) | Recommended | Interpretable numeric baseline |
| Logistic regression | Yes (clf) | Recommended | Binary/multiclass baseline |
| Decision tree | Yes | Optional | Interpretable non-linear |
| Random forest | Yes | Optional | Strong tabular default |
| Gradient boosting | Yes | Often helpful | Competitive tabular accuracy |
| k-NN | Yes | **Yes** | Small data, local patterns |
| k-Means | No | **Yes** | Segmentation |

## What you should *not* memorize yet

- Full boosting residual math
- Every hyperparameter of every library
- Research leaderboards

Focus on **matching problem → model family → evaluation**. Tuning skill grows with projects.

## Mini project prompt (warm-up for exercises)

Using any small binary classification CSV:

1. Train logistic regression and a random forest with a shared train/val split
2. Report the same metric for both
3. Write three sentences on which you would ship and why (include a baseline comparison)

## What is next

**[03-scikit-learn-workflow.md](03-scikit-learn-workflow.md)** turns these models into a professional workflow: `Pipeline`, cross-validation, metrics, and honest baselines.
