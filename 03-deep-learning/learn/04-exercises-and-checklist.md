# Stage 3 — Exercises and “Ready for Stage 4” Checklist

Use these after lessons 3.1–3.3. Beginner first; Stretch when ready. Prefer finishing imperfect projects over endless tutorial-hopping.

---

## Exercises tied to Lesson 3.1 (Neural net basics)

### Beginner

1. **Layer count.** Draw an MLP with input 10, two hidden layers of 32 ReLU units, and 3 output logits. Count Linear layers.
2. **Activation choice.** Why do we rarely use sigmoid in every hidden layer of a deep net today?
3. **Loss match.** Pick a loss for: (a) house price regression; (b) 10-class digit classification.
4. **Epoch vs batch.** If a dataset has 6,400 examples and batch size 64, how many updates per epoch?

### Stretch

5. **Underfit/overfit plan.** List concrete knobs you would change for each regime.
6. **Softmax intuition.** For logits `[2.0, 1.0, 0.1]`, which class wins? Does softmax change the argmax versus raw logits?

---

## Exercises tied to Lesson 3.2 (PyTorch loop)

### Beginner — MNIST MLP

1. Install PyTorch / torchvision in your venv.
2. Train the lesson’s MLP (or your variant) on MNIST for several epochs.
3. Plot or print train loss and accuracy each epoch.
4. Save `state_dict` and reload; verify evaluation accuracy matches.

### Stretch

5. Create a **true validation split** from MNIST training data; tune only on val; report test once.
6. Add dropout and weight decay; compare val curves.
7. Swap Adam for SGD+momentum; rewrite what you observed in five sentences.

---

## Exercises tied to Lesson 3.3 (CNNs and Transformers)

### Beginner — Small CNN

1. Implement `SmallCNN` (or similar) for MNIST/Fashion-MNIST.
2. Compare MLP vs CNN val accuracy under a similar training budget (epochs × time).
3. Write a short paragraph: did the CNN win? Was it worth the complexity?

### Beginner — Fine-tune path (optional if compute allows)

4. Using torchvision, load a pretrained small model, adapt the head for a simple dataset (for example a subset of CIFAR-10 or a tiny custom image folder). Freeze backbone first; train the head.

### Stretch — Tiny Transformer tutorial path

5. Follow a reputable PyTorch “tiny Transformer” or character-level language model tutorial (official docs / well-known courses). Goal: run something that trains and predicts text **on a small file**.
6. Print shapes inside attention or blocks once; confirm batch and sequence dimensions.
7. Journal: what is shared between this training loop and your MNIST loop? What is different (tokenization, sequence dim, causal masking if any)?

---

## Stage 3 mini-capstone

Ship a small vision project **or** a tiny sequence model—your choice.

### Option A — Vision

- Dataset: MNIST, Fashion-MNIST, or CIFAR-10
- Models: MLP baseline + CNN (required)
- Report: curves, final metrics, 5 bullets on errors you inspected
- Code: clear train/eval scripts and README

### Option B — Tiny language model

- Dataset: a small public text file (public-domain or your own notes)
- Model: tiny Transformer or even a small LSTM if Transformers feel blocked—document why
- Report: train loss curve; a few qualitative samples; limitations

### Acceptance bar

A teammate can run your README steps and get similar metrics. You can explain every major line of the training loop.

---

## Checklist: Ready for Stage 4

### Concepts

- [ ] I can explain forward pass, loss, backward pass, and parameter update
- [ ] I know what batches and epochs are
- [ ] I can describe ReLU and why nonlinearities matter
- [ ] I can explain overfitting remedies used in deep nets

### PyTorch

- [ ] I can write or modify a training loop with `train()` / `eval()`
- [ ] I can use `Dataset` / `DataLoader`
- [ ] I can save and load a `state_dict`
- [ ] I know how to select `cpu` vs `cuda` devices

### Architectures

- [ ] I can explain why CNNs help with images
- [ ] I can explain attention/Transformers at a high level for sequences
- [ ] I know when to fine-tune a pretrained model instead of training from scratch

### Practice

- [ ] I completed MNIST MLP training
- [ ] I completed a CNN or fine-tune experiment
- [ ] I at least started a tiny Transformer / sequence tutorial path
- [ ] I finished the Stage 3 mini-capstone (or equivalent)

---

## Suggested next step

Proceed to **Stages 4–9 — Generative AI / LLMs**: foundation models, prompting, RAG, tools, and evaluation—using APIs and retrieval while keeping the same honest evaluation mindset.
