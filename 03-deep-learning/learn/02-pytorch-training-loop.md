# Lesson 3.2 — PyTorch Training Loop

PyTorch is a leading deep learning framework. This lesson focuses on the **everyday loop**: tensors → dataset → model → loss → backward → optimizer step → evaluate.

Prefer the official **PyTorch documentation** and tutorials when APIs evolve.

## Tensors

A **tensor** is a multi-dimensional array (like a NumPy ndarray) with optional GPU support and automatic differentiation tracking.

```python
import torch

x = torch.tensor([[1.0, 2.0], [3.0, 4.0]])
print(x.shape, x.dtype)

y = torch.randn(4, 3)  # random normal
z = y @ torch.randn(3, 2)  # matmul
print(z.shape)
```

Useful creations: `torch.zeros`, `torch.ones`, `torch.arange`, `torch.from_numpy`.

### Autograd basics

```python
w = torch.tensor([1.0, 2.0], requires_grad=True)
loss = (w ** 2).sum()
loss.backward()
print(w.grad)  # dl/dw
```

Inside `nn.Module`, parameters already have `requires_grad=True`.

## Device: CPU and GPU

```python
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
print(device)

model = model.to(device)
batch_x = batch_x.to(device)
batch_y = batch_y.to(device)
```

**Notes:**

- Not having a GPU is fine for MNIST-scale learning
- Move **model and batch tensors** to the same device
- `.cpu()` before converting to NumPy for plotting

## `nn.Module`

Models subclass `torch.nn.Module`, define layers in `__init__`, and compute the forward pass in `forward`.

```python
import torch.nn as nn

class MLP(nn.Module):
    def __init__(self, in_dim=784, hidden=128, n_classes=10):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(in_dim, hidden),
            nn.ReLU(),
            nn.Linear(hidden, n_classes),
        )

    def forward(self, x):
        x = x.view(x.size(0), -1)  # flatten NCHW or N,28,28 → N,784
        return self.net(x)
```

Calling `model(x)` invokes `forward`.

## Dataset and DataLoader

`Dataset` provides `__len__` and `__getitem__`. `DataLoader` batches, shuffles, and parallel-loads.

```python
from torch.utils.data import DataLoader
from torchvision import datasets, transforms

transform = transforms.Compose([
    transforms.ToTensor(),  # scales images to [0, 1]
])

train_ds = datasets.MNIST(root="data", train=True, download=True, transform=transform)
test_ds = datasets.MNIST(root="data", train=False, download=True, transform=transform)

train_loader = DataLoader(train_ds, batch_size=64, shuffle=True)
test_loader = DataLoader(test_ds, batch_size=256, shuffle=False)
```

Custom datasets follow the same pattern when you load your own files.

## The training loop (memorize this shape)

```python
import torch.nn.functional as F

model = MLP().to(device)
opt = torch.optim.Adam(model.parameters(), lr=1e-3)

def train_one_epoch(model, loader, opt, device):
    model.train()
    total_loss = 0.0
    for xb, yb in loader:
        xb, yb = xb.to(device), yb.to(device)
        opt.zero_grad(set_to_none=True)
        logits = model(xb)
        loss = F.cross_entropy(logits, yb)
        loss.backward()
        opt.step()
        total_loss += loss.item() * xb.size(0)
    return total_loss / len(loader.dataset)

@torch.no_grad()
def evaluate(model, loader, device):
    model.eval()
    correct = 0
    total = 0
    total_loss = 0.0
    for xb, yb in loader:
        xb, yb = xb.to(device), yb.to(device)
        logits = model(xb)
        loss = F.cross_entropy(logits, yb)
        total_loss += loss.item() * xb.size(0)
        pred = logits.argmax(dim=1)
        correct += (pred == yb).sum().item()
        total += yb.size(0)
    return total_loss / total, correct / total

for epoch in range(5):
    tr_loss = train_one_epoch(model, train_loader, opt, device)
    va_loss, va_acc = evaluate(model, test_loader, device)  # toy: using test as monitor—prefer a true val split in real work
    print(f"epoch {epoch}: train_loss={tr_loss:.4f} val_loss={va_loss:.4f} val_acc={va_acc:.3f}")
```

### Loop checklist

1. `model.train()` for training; `model.eval()` for evaluation
2. `optimizer.zero_grad()` before `backward`
3. `loss.backward()` then `optimizer.step()`
4. Use `torch.no_grad()` (or `@torch.no_grad()`) when evaluating to save memory and avoid tracking
5. Log train **and** val metrics every epoch

In serious projects, carve a validation set out of training data instead of tuning on the official test set. MNIST tutorials often blur this; you should not.

## Saving and loading

```python
torch.save(model.state_dict(), "mlp_mnist.pt")

model2 = MLP().to(device)
model2.load_state_dict(torch.load("mlp_mnist.pt", map_location=device))
model2.eval()
```

Save `state_dict` (recommended) plus a note of the architecture and hyperparameters.

## Debugging tips

| Symptom | Try |
|---------|-----|
| Loss is `nan` | Lower lr; check labels; check input scaling |
| Accuracy stuck at chance | Verify shapes; confirm labels; print a batch |
| Train good, val bad | Regularize; reduce model; augment; early stop |
| Extremely slow | Smaller model; fewer printouts; ensure GPU in use if expected |

```python
xb, yb = next(iter(train_loader))
print(xb.shape, yb.shape, yb[:8])
print(model(xb.to(device)).shape)
```

## Learning rate and simple schedules (awareness)

Start with Adam at `1e-3` for many MLPs. If loss plateaus, try `1e-4` or a scheduler (step decay, cosine). Do not change five things at once.

## Mini practice

1. Change hidden size from 128 to 32 and to 512; compare val accuracy after a few epochs.
2. Replace Adam with `torch.optim.SGD(..., lr=0.1, momentum=0.9)` and observe behavior.
3. Add `nn.Dropout(0.2)` before the last layer; remember `train` vs `eval`.

## What is next

**[03-cnns-and-transformers.md](03-cnns-and-transformers.md)** — why we do not always flatten images into MLPs, and how attention powers modern sequence models.
