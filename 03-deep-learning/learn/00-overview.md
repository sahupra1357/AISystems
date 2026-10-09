# Stage 3 — Deep Learning: Overview

Classical ML shines on many tabular problems. **Deep learning** shines when you have large datasets and rich structure—images, audio, text, and other high-dimensional signals—where useful features are hard to hand-engineer.

Stage 3 builds intuition for neural networks and a practical **PyTorch** training loop. You will not need to reinvent backpropagation algebra from scratch; you will need to understand what training *is doing* and how to run experiments without fooling yourself.

## Goals for Stage 3

By the end of Stage 3, you should be able to:

- Explain neurons, layers, activations, loss functions, and the role of backpropagation in plain language
- Describe optimizers (SGD, Adam), minibatches, and epochs
- Use PyTorch tensors, `Dataset` / `DataLoader`, and a standard training + evaluation loop
- Know when a GPU helps and how to move tensors/models to a device
- Describe why CNNs help with images and why Transformers / attention help with sequences
- Complete beginner exercises (MNIST MLP, small CNN or fine-tune path, tiny Transformer tutorial path) and pass the Ready-for-Stage-4 checklist

## Prerequisites

Complete **Stage 2 — Classical ML**, or be equally comfortable with:

- Train/val/test discipline, overfitting, metrics, and baselines
- Python classes and NumPy-style array thinking
- Installing packages into a virtual environment

Install for this part:

```bash
python -m pip install torch torchvision matplotlib
```

CPU-only PyTorch is fine for learning. Follow the official **PyTorch** install guidance if you have an NVIDIA GPU and want CUDA builds.

## Suggested time

About **2–4 weeks** part-time:

| Week | Focus |
|------|--------|
| 1 | Neural net basics (lesson 3.1); start PyTorch loop (lesson 3.2) |
| 2 | Solidify training loop; debugging under/overfitting |
| 3 | CNNs and Transformers (lesson 3.3) |
| 4 | Exercises, checklist, Stage 3 mini-capstone |

## Order to study

1. **[01-neural-net-basics.md](01-neural-net-basics.md)** — Building blocks and training intuition.
2. **[02-pytorch-training-loop.md](02-pytorch-training-loop.md)** — Tensors, data loading, the loop you will reuse forever.
3. **[03-cnns-and-transformers.md](03-cnns-and-transformers.md)** — Two dominant architectures and when to use which.

Then complete **[04-exercises-and-checklist.md](04-exercises-and-checklist.md)**.

## Success criteria before Stages 4–9

You are ready for Generative AI / LLMs (Stages 4–9) when you can:

- Draw a small MLP and name loss, forward pass, and parameter update
- Write or clearly modify a PyTorch training loop with train and eval modes
- Explain minibatch SGD vs Adam at a high level
- Say why CNNs exploit locality/spatial structure and why attention helps sequences
- Finish the Stage 3 exercises (at least the beginner track) without copying blindly

LLMs are deep models too—Stages 4–9 focuses on using and adapting foundation models, building on the training intuition you form here.
