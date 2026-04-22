# TinyTorch -- Deep Dive

> **Purpose:** Explain how TinyTorch is structured, what the student actually does, how the
> tooling works, what the tests cover, and what's tricky. Read REPO_OVERVIEW.md first.

---

## What Is TinyTorch?

TinyTorch is a course where students build a complete ML framework from scratch, using only
Python and NumPy -- no PyTorch, no TensorFlow. The goal is that by the end, the student has
written every layer of the stack: memory layout, automatic differentiation, optimizers,
convolutional networks, transformers, quantization, and performance benchmarking.

The tagline is "AI Bricks" -- you learn the stable engineering foundations that all AI
frameworks are built on, so you can understand, debug, and innovate at the framework level,
not just use the framework.

---

## The 20-Module Chain

Modules must be completed in order -- each one builds on the previous. The dependency chain
is strict: you cannot do backprop (Module 06) without Tensors (01) and Layers (03).

```
PART I: FOUNDATIONS (Modules 01-08)
  01  Tensor           N-dimensional array: shape, strides, indexing, broadcasting
  02  Activations      ReLU, Sigmoid, Tanh, Softmax (forward pass only here)
  03  Layers           Linear layer, Module base class, parameter management
  04  Losses           MSE, CrossEntropyLoss -- how to measure model error
  05  DataLoader       Batching, shuffling, iterating over datasets efficiently
  06  Autograd         Backward pass: compute graph, chain rule, gradient accumulation
  07  Optimizers       SGD with momentum, Adam, AdamW, learning rate schedulers
  08  Training         Full training loop: forward/loss/backward/step, evaluation, checkpoints

PART II: VISION (Module 09)
  09  Convolutions     Conv2d, MaxPool2d, building CNNs for image classification

PART III: LANGUAGE (Modules 10-13)
  10  Tokenization     BPE tokenizer, vocabulary, encoding/decoding text
  11  Embeddings       Token embeddings, positional embeddings (sinusoidal + learned)
  12  Attention        Scaled dot-product attention, multi-head attention
  13  Transformers     Full transformer block: attention + FFN + LayerNorm + residuals

PART IV: OPTIMIZATION (Modules 14-20)
  14  Profiling        FLOP counting, memory profiling, latency measurement
  15  Quantization     INT8/INT4 quantization: scale, zero_point, quantize/dequantize
  16  Compression      Pruning (weight zeroing) and knowledge distillation
  17  Acceleration     Tiling, vectorization, parallelism patterns
  18  Memoization      KV-cache for transformer inference
  19  Benchmarking     MLPerf-style measurement: throughput, latency percentiles
  20  Capstone         Full ML systems project pulling everything together
```

**Key insight:** Parts I-II build something that works. Parts III-IV teach you how to make
it work *well* -- the gap between a toy model and production systems.

---

## Historical Milestones

After completing certain modules, students can unlock "milestone scripts" that recreate
historically significant ML results using their own framework code:

```
Milestone 1  (1958)  Perceptron    -- after Module 08
  Rosenblatt's first trainable network. Binary classification, gradient descent.

Milestone 2  (1969)  XOR Crisis    -- after Module 08
  Minsky showed single-layer networks can't solve XOR. Multi-layer solution.

Milestone 3  (1986)  Backprop MLP  -- after Module 08
  Rumelhart's backpropagation. Train on MNIST digits.

Milestone 4  (1998)  CNN Revolution -- after Module 09
  LeCun's LeNet. Image classification with convolutions on CIFAR-10.

Milestone 5  (2017)  Transformer Era -- after Module 13
  "Attention is All You Need." Language generation with self-attention.

Milestone 6  (2018+) MLPerf        -- after Module 19
  Production optimization: measure, profile, optimize to MLPerf benchmarks.
```

These are real scripts in `tinytorch/milestones/` that run with no modification -- the
student's framework just needs to implement the required modules correctly.

---

## Directory Layout

```
tinytorch/
|
|-- src/                   Where the reference implementations live.
|   |-- 01_tensor/
|   |   |-- 01_tensor.py   The actual Python source code (version controlled).
|   |   |-- ABOUT.md       Conceptual overview and learning objectives for module.
|   |-- 02_activations/
|   |   |-- 02_activations.py
|   |   |-- ABOUT.md
|   ... (one folder per module, same pattern x20)
|
|-- modules/               Where students do their work.
|   |-- 01_tensor/
|   |   |-- tensor.ipynb   Jupyter notebook version (auto-generated from src).
|   |   |-- tensor.py      Student's implementation file (they edit this).
|   ... (generated, not hand-edited by contributors)
|
|-- tinytorch/             The importable Python package.
|   |-- core/              Assembled from whatever modules student has completed.
|                          When tests import `from tinytorch import Tensor`, it
|                          comes from here.
|
|-- tests/                 600+ tests.
|   |-- 01_tensor/         Tests for each module.
|   |-- 02_activations/
|   ... (mirrors src/ structure)
|   |-- e2e/               End-to-end tests (train a real model).
|   |-- integration/       Cross-module interaction tests.
|   |-- milestones/        Tests that milestone scripts actually run.
|   |-- regression/        Historical regression tests (bugs we've fixed stay fixed).
|
|-- milestones/            The historical milestone runner scripts.
|   |-- 01_1958_perceptron/
|   |-- 02_1969_xor/
|   ... (6 total)
|
|-- tito/                  The tito CLI tool source.
|   |-- main.py            Entry point.
|   |-- commands/          One file per command group (module, milestone, bench, etc).
|   |-- core/              Config, console, virtual env manager, exceptions, theme.
|
|-- site-quarto/           Documentation site source (Quarto).
|-- datasets/              Small bundled datasets for exercises.
```

**The key workflow arrow:** `src/*.py` -> `modules/*.ipynb` (auto-generated) -> student edits
`modules/*.py` -> `tinytorch/` package assembles from student's code -> tests run against it.

---

## How a Student Actually Works (tito CLI)

Students never directly run pytest or navigate file paths. They use `tito`, the course CLI:

```bash
# First-time setup
tito setup                    # Creates virtual env, installs deps

# Starting a module
tito module start 01_tensor   # Copies template to modules/01_tensor/tensor.py
                              # Opens the notebook in browser

# While working
tito module test 01_tensor    # Runs tests for just this module
tito module check 01_tensor   # Checks completeness without running all tests

# When done with a module
tito module submit 01_tensor  # Validates, packages, marks complete

# Unlocking a milestone (once you've completed prerequisites)
tito milestone run 01_perceptron

# Benchmarking
tito benchmark run            # Runs performance benchmarks
tito olympics                 # Community leaderboard submission (when enabled)
```

The tito CLI is a thin wrapper over pytest and file manipulation. It handles path setup,
virtual env activation, and output formatting so students see clean progress bars instead
of raw pytest output.

---

## The src/ vs modules/ Distinction

This is the most confusing part for contributors. There are TWO copies of each module's code:

**`src/01_tensor/01_tensor.py`** -- the canonical reference implementation  
- This is what contributors edit when fixing bugs or adding features  
- It's version controlled and reviewed like any production code  
- Students never see this file directly  

**`modules/01_tensor/tensor.py`** -- the student's working copy  
- Auto-generated from src/ via a tito command  
- Has the same structure as the reference but with implementation bodies replaced by `...` or `raise NotImplementedError`  
- Students fill in the blanks  
- Not version controlled (it's in .gitignore because each student has their own)  

**When you fix a bug in TinyTorch:**
- Edit `src/XX_name/XX_name.py`
- The fix propagates to students when they regenerate or restart the module

---

## The Autograd Engine (Module 06) -- The Heart of the System

Understanding autograd is key to understanding why modules 01-05 have to be built first.

Every `Tensor` object has these fields:
```
Tensor
  .data         -- the numpy array with the actual numbers
  .grad         -- gradient (same shape as .data), None until backward is called
  .requires_grad -- whether this tensor participates in gradient computation
  ._grad_fn     -- the backward function that produced this tensor (None for leaf nodes)
  ._inputs      -- the tensors that fed into this tensor's operation
```

During a forward pass, every operation (add, matmul, relu, etc.) creates a new Tensor and
attaches a `_grad_fn` to it that knows how to compute gradients with respect to its inputs.
This forms a directed acyclic graph (the compute graph).

During `loss.backward()`, the system walks this graph in reverse topological order, calling
each `_grad_fn` with the gradient flowing from above, and accumulating into `.grad`.

This is why:
- Module 01 (Tensor) must define the data structure first
- Modules 02-05 define forward operations but don't implement backward yet
- Module 06 adds the backward pass machinery -- then all previous modules get gradients
- Module 07 (Optimizers) can then do `param.data -= lr * param.grad`

---

## The Training Loop (Module 08) -- What "Training" Actually Means

```
For each epoch:
  For each batch from DataLoader:
    1. optimizer.zero_grad()          -- clear gradients from last step
    2. outputs = model(inputs)        -- forward pass through all layers
    3. loss = criterion(outputs, targets)  -- compute scalar loss
    4. loss.backward()                -- backward pass, populate .grad everywhere
    5. optimizer.step()               -- update params: param -= lr * param.grad

After each epoch:
  6. model.eval()                     -- switch off dropout/batchnorm training behavior
  7. run evaluate() on validation set -- compute accuracy/loss without gradients
  8. model.train()                    -- switch back for next epoch
```

The Trainer class in Module 08 wraps this loop. It handles:
- Regression vs classification detection (shape-based)
- Accuracy computation (argmax for multi-class, threshold for binary, N/A for regression)
- Checkpoint saving/loading
- Progress logging

---

## Test Structure -- What 600+ Tests Cover

```
tests/
|-- 01_tensor/       Shape checks, broadcasting, slicing, data types
|-- 02_activations/  Output range checks, gradient flow through ReLU/Sigmoid
|-- 03_layers/       Linear layer output shape, weight initialization, parameter count
|-- 04_losses/       Loss value correctness vs manual calculation
|-- 05_dataloader/   Batch size, shuffle randomness, no data leakage across epochs
|-- 06_autograd/     Gradient correctness (compared to finite differences)
|-- 07_optimizers/   Parameter updates: SGD momentum, Adam bias correction
|-- 08_training/     Trainer trains (loss goes down), evaluate() accuracy range
|-- 09_convolutions/ Conv output shape formula: (W - K + 2P) / S + 1
|-- ...
|-- e2e/             Actually train a small CNN on CIFAR-10, assert >50% accuracy
|-- regression/      Bugs that were fixed and must never regress
```

**How gradient tests work:** To check that backprop is correct without running PyTorch,
tests use numerical differentiation -- perturb an input by epsilon, measure output change,
compare to what the computed gradient predicts. If they match to ~4 decimal places, the
backward pass is correct.

---

## Common Bugs and Gotchas

**1. Shape confusion in evaluate()**
The Trainer checks `outputs.data.shape[-1] > 1` to decide classification vs regression.
A 1D output (shape `[batch]`) means regression. A 2D output (shape `[batch, n_classes]`)
means classification. Getting this wrong silently returns accuracy=0.0 for regression tasks.

**2. Constant tensor quantization**
If all values in a tensor are equal, `max_val - min_val = 0`. The quantize function sets
`scale=1.0` but must derive `zero_point` from the actual value so dequantization recovers
the original. Early version hardcoded `zero_point=0`, silently zeroing all constant tensors.

**3. The stale gradient problem**
Calling `loss.backward()` twice without `optimizer.zero_grad()` between steps accumulates
gradients. This is intentional PyTorch behavior but trips up students who loop incorrectly.

**4. Module eval/train mode**
`model.eval()` affects Dropout (turns off) and BatchNorm (uses running stats). Forgetting to
call it during evaluation causes non-deterministic test accuracy.

---

## What's NOT in TinyTorch

These are deliberately excluded to keep things learnable:
- **GPU support** -- everything runs on CPU, numpy only. GPU abstraction would add a whole
  new layer of complexity before the fundamentals are understood.
- **Automatic broadcasting in backward** -- students implement the forward broadcast rules
  but the backward pass handles only specific common cases.
- **Dynamic shapes** -- tensor shapes are fixed at creation; no `torch.Tensor.reshape()` in
  the middle of autograd (except where explicitly implemented).
- **Distributed training** -- single machine, single process only.
