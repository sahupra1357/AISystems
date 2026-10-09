# Lesson 03 — CNNs and Transformers

MLPs treat inputs as flat vectors. Images and sequences have **structure**. Two architectures dominate modern deep learning for those domains: **Convolutional Neural Networks (CNNs)** and **Transformers**.

## Why structure matters

A 28×28 MNIST image has 784 pixels. Nearby pixels are related; an MLP must *relearn* locality from scratch at every position. CNNs bake locality and translation-friendly filters into the architecture.

Language (and many sequences) has long-range dependencies: the meaning of a word can depend on words far away. Transformers use **attention** to let positions share information flexibly.

## CNNs for images

### Convolution intuition

A **convolution** slides small learnable filters (kernels) over the image, detecting local patterns (edges, textures, later parts and objects).

Stacking convolutional layers builds a hierarchy:

```text
edges → textures → parts → objects
```

### Typical CNN block

```text
Conv2d → Activation (ReLU) → (optional BatchNorm) → (optional Pooling)
```

- **Conv2d:** learnable filters
- **Pooling** (MaxPool): downsample spatial size, reduce compute, increase receptive field
- **Fully connected / global pooling head:** map features to class logits

### Tiny CNN sketch (PyTorch)

```python
import torch.nn as nn

class SmallCNN(nn.Module):
    def __init__(self, n_classes=10):
        super().__init__()
        self.features = nn.Sequential(
            nn.Conv2d(1, 32, kernel_size=3, padding=1),
            nn.ReLU(),
            nn.MaxPool2d(2),
            nn.Conv2d(32, 64, kernel_size=3, padding=1),
            nn.ReLU(),
            nn.MaxPool2d(2),
        )
        self.head = nn.Sequential(
            nn.Flatten(),
            nn.Linear(64 * 7 * 7, 128),  # for 28x28 input with two 2x pools
            nn.ReLU(),
            nn.Linear(128, n_classes),
        )

    def forward(self, x):
        return self.head(self.features(x))
```

### When CNNs shine

- Images, video frames, some spectrograms
- Problems where local spatial patterns matter
- Still strong baselines; modern vision also uses Transformers (ViT) and hybrids

### Transfer learning / fine-tuning (practical path)

For real photos (not MNIST), start from a pretrained model (for example ResNet variants in `torchvision.models`), replace the classification head, and fine-tune:

1. Freeze early layers; train the head
2. Optionally unfreeze more layers with a small learning rate

This is often far better than training from scratch on a small dataset. Follow current **torchvision** docs for model weights APIs.

### Mini practice

Explain in two sentences why a CNN can share parameters across positions and why that helps.

## Transformers and attention for sequences

### Attention in one paragraph

**Attention** lets each position build a weighted mixture of other positions’ representations. Weights come from similarity between **queries** and **keys**; mixed values are **values**. Multi-head attention repeats this in parallel subspaces.

Intuition: when predicting or encoding a token, the model can look at the most relevant other tokens—not only neighbors.

### Transformer block (encoder-style sketch)

```text
x → LayerNorm → Multi-Head Self-Attention → residual add
  → LayerNorm → Feed-Forward MLP → residual add
```

Stack many blocks. Add **positional information** because plain attention is permutation-flexible and needs position cues.

### Where Transformers dominate

- Language models and machine translation
- Speech and multimodal models
- Increasingly vision (ViT) and other modalities

LLMs (Part 4) are Transformers (or Transformer variants) trained at huge scale on text (and more).

### Tiny learning path (tutorial-scale)

You do not need to train GPT from scratch. A sensible learning path:

1. Read a clear illustrated explanation of attention (classic blog explanations and the original “Attention Is All You Need” paper are widely known; use reputable sources).
2. Run a **tiny** Transformer tutorial in PyTorch—character-level or tiny vocab language modeling on a small text file.
3. Inspect shapes: batch, sequence length, embedding dim, number of heads.

A minimal conceptual module shape:

```python
# Pseudocode shapes — not a full implementation
# x: (batch, seq, d_model)
# attn_out = MultiHeadAttention(x, x, x)
# x = x + attn_out
# x = x + FeedForward(x)
```

### Efficiency note

Self-attention scales roughly with sequence length squared in the naive form. Long context needs clever engineering (efficient attention variants, chunking, etc.). As a practitioner consuming APIs, know that **context length is a real constraint and cost driver**.

## CNN vs Transformer — chooser

| Prefer CNN (or CNN backbone) | Prefer Transformer |
|------------------------------|--------------------|
| Small image datasets from scratch | Language / long-range sequence tasks |
| Strong inductive bias for local pixels | Large-scale pretrained models available |
| Lower compute budgets for small vision tasks | Multimodal foundation models |

| Often best in practice |
|------------------------|
| **Pretrained** vision model (CNN or ViT) fine-tuned for your labels |
| **Pretrained** language model adapted via prompts / adapters / fine-tunes (Part 4) |

## Connecting back to Part 2 habits

Architecture choice does not replace evaluation discipline:

- Hold out validation data
- Track train vs val loss
- Use a baseline (logistic regression on flattened pixels is a humbling MNIST baseline; majority class for imbalanced tasks)
- Prefer simpler models when scores tie

## Common beginner pitfalls

1. Flattening images into a huge MLP and wondering why it underperforms a tiny CNN
2. Forgetting `model.eval()` and dropout left on during testing
3. Fine-tuning a huge model on 200 images with a large learning rate
4. Confusing “Transformer architecture” with “chatbot product”

## What is next

**[04-exercises-and-checklist.md](04-exercises-and-checklist.md)** — MNIST MLP, CNN/fine-tune practice, a tiny Transformer path, and the Ready-for-Part-4 checklist.
