# Lesson 01 — Neural Network Basics

This lesson is conceptual with light math. The goal is fluency: when someone says “we increased depth and switched the loss,” you should know what that implies.

## From linear models to neurons

A linear classifier computes \(z = w \cdot x + b\). A **neuron** often does the same, then applies a non-linear **activation** \(a = \sigma(z)\).

Stacking many neurons in **layers** lets the network represent complex functions. Without non-linear activations, stacked linear layers collapse into one linear map—so activations are not decoration; they are essential.

### A minimal multilayer perceptron (MLP)

```text
input x
  → Dense layer → activation
  → Dense layer → activation
  → Dense layer → output
```

- **Dense / fully connected / Linear layer:** each output unit connects to all inputs of that layer.
- **Hidden layers:** everything between input and output.
- **Depth:** number of layers; **width:** units per layer.

## Activations

| Activation | Shape intuition | Common use |
|------------|-----------------|------------|
| ReLU | \(\max(0, z)\) | Default hidden activation for many nets |
| GELU / SiLU | Smooth ReLU-like | Transformers and modern nets |
| Sigmoid | (0, 1) | Older nets; binary output probabilities |
| Softmax | Vector of positive numbers summing to 1 | Multiclass classification outputs |
| Tanh | (-1, 1) | Some recurrent / older architectures |

**ReLU** is a good mental default for hidden layers: simple, sparse gradients for negative inputs, works well in practice.

### Mini practice

If a hidden ReLU unit receives \(z = -2.5\), what is its activation? What happens to the gradient through that unit for that example (qualitatively)?

## Loss functions

The **loss** measures how wrong predictions are. Training minimizes average loss over the dataset (plus regularization terms sometimes).

| Problem | Typical loss |
|---------|--------------|
| Regression | MSE or MAE |
| Binary classification | Binary cross-entropy (with logits) |
| Multiclass | Cross-entropy (with logits + softmax conceptually) |

**Logits** means raw scores before softmax/sigmoid. Frameworks often combine softmax and cross-entropy in one numerically stable function.

```python
# Conceptual: not a full training loop yet
# loss = criterion(model(x), y)
```

## Forward pass and parameters

**Parameters** (weights and biases) are the knobs the model learns. A **forward pass** computes outputs (and intermediate activations) given inputs and current parameters.

At initialization, parameters are random-ish (with carefully chosen schemes). Training moves them toward values that reduce loss.

## Backpropagation intuition

You do not need to expand every chain-rule term by hand, but you should know:

1. Loss depends on parameters through the chain of layer computations.
2. **Backpropagation** applies the chain rule efficiently to compute \(\partial L / \partial w\) for every parameter.
3. Frameworks like PyTorch build a **computation graph** and call `.backward()` to populate gradients.

Gradient descent update (simplified):

\[
w \leftarrow w - \eta \cdot \frac{\partial L}{\partial w}
\]

where \(\eta\) is the **learning rate**.

### Mental picture

```text
Forward:  data → predictions → loss
Backward: loss → gradients for each parameter
Update:   parameters -= lr * gradient
```

## Optimizers

**SGD:** follow the gradient of a minibatch. Variants add **momentum** (remember recent gradients) to damp oscillations.

**Adam:** adapts learning rates per-parameter using estimates of first and second moments of gradients. A strong default for many deep learning tasks.

You still choose a base learning rate. Too high → loss explodes or oscillates. Too low → learning crawls.

### Mini practice

If training loss barely moves for many epochs, list three plausible causes (learning rate, bugs, data, model capacity, etc.).

## Batches and epochs

- **Batch / minibatch:** a small set of examples processed together. Gradients are averaged over the batch.
- **Epoch:** one full pass over the training set.
- **Iteration / step:** one batch update.

Why batches?

- Full-dataset gradients are expensive
- Minibatches add noise that can help generalization
- GPUs like medium-sized batched tensor ops

Typical pattern: shuffle training data each epoch; do **not** shuffle validation/test when you need deterministic evaluation (or shuffle only if your metric needs it—usually you do not).

## Generalization in deep nets

Deep nets are high-capacity. They can memorize. You fight overfitting with:

- More data / augmentation
- Weight decay (L2-style regularization)
- Dropout (randomly zero activations during training)
- Early stopping on validation loss
- Simpler architectures when data is small

**Train mode vs eval mode** matters: dropout and batch-norm behave differently at train and test time. In PyTorch you call `model.train()` and `model.eval()`.

## Capacity, data, and compute

Rough intuition:

| Situation | Tendency |
|-----------|----------|
| Tiny data, huge net | Overfit; prefer classical ML or strong pretraining + care |
| Huge data, expressive net | Deep learning often wins |
| Tabular medium data | Tree ensembles frequently competitive |

Pretrained models (Part 3 CNN fine-tuning; Part 4 LLMs) change the game: you often start from features learned on large datasets.

## What “learning” looks like on a plot

Healthy-ish run:

- Train loss trends down
- Val loss trends down, then may flatten
- Gap between train and val stays moderate

Unhealthy:

- Train ↓↓, val ↑ → overfitting
- Both stuck high → underfit, bad lr, buggy labels, or insufficient capacity
- Wild spikes → lr too high, bad batches, or numerical issues

## Key vocabulary

- neuron, layer, MLP
- activation, ReLU, softmax
- loss, logits, cross-entropy
- forward / backward
- gradient, learning rate, optimizer
- batch, epoch
- train vs eval mode
- overfitting in deep nets

## What is next

**[02-pytorch-training-loop.md](02-pytorch-training-loop.md)** implements these ideas in PyTorch with tensors, datasets, and a loop you can reuse.
