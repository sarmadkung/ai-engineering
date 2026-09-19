# Topic 8 — Tiny Language Model

**Why this topic:** assemble Phase 1's pieces into one real model class — in PyTorch this
time — with a proper forward pass and a loss. This is the model you train in Topic 9.

**Setup needed:** this is where you create a virtual environment and install PyTorch.
`python3 -m venv .venv`, activate it, `pip install torch`. Everything before this was
standard library; from here on it isn't.

---

# Part 1 — Theory

## 8.1 Why PyTorch now

You have written attention by hand, so you know what's inside. PyTorch adds two things you
cannot practically hand-roll:

- **Autograd** — it records every operation and computes all gradients automatically. You
  write the forward pass; the backward pass is derived for you.
- **Tensors on hardware** — batched operations on GPU/MPS, thousands of times faster than
  Python loops.

A `nn.Module` is just a class with parameters registered so PyTorch can find them. `nn.Linear`
is your matrix multiply plus a bias. Nothing conceptually new — only faster and
differentiable.

## 8.2 Shapes, which are most of the difficulty

Almost every bug in a model is a shape bug. Learn to say shapes out loud:

```
B = batch size          T = sequence length (block size)
C = model dimension     V = vocabulary size

input IDs        (B, T)          integers
after embedding  (B, T, C)       floats
after blocks     (B, T, C)
logits           (B, T, V)       one prediction per position
```

Note the last line: the model predicts at **every** position, not just the end. During
training all of them are used; during generation you keep only the last.

## 8.3 The embedding layer

Two lookups added together (Topic 3):

```
token embedding    (V, C)  -> row per token
position embedding (T, C)  -> row per position
x = tok_emb(ids) + pos_emb(arange(T))
```

Learned positions are fine for a tiny model. RoPE comes in Topic 12.

## 8.4 The transformer block

Same structure you built in Topic 4, now as a module:

```
x = x + attention(layer_norm(x))     # mixes across positions, causal mask
x = x + feed_forward(layer_norm(x))  # per position, expand 4x and back
```

Normalization **before** each sublayer ("pre-norm") is the modern default; it trains more
stably than the original post-norm. Stack `n_layer` of these.

The causal mask must be built into the block. If it's missing the model sees the future,
loss collapses toward zero, and generation is garbage — a failure that looks like success.

## 8.5 Output projection and weight tying

A final `nn.Linear(C, V)` turns each position's vector into logits.

**Weight tying** is a common trick: use the transpose of the token embedding matrix as this
output layer. Justification: the embedding maps token → vector, the output maps vector →
token, so they are two directions of the same relationship. It saves `V × C` parameters and
usually improves small models.

## 8.6 The loss function

Cross-entropy — the same quantity you measured in Topic 1, now differentiable:

```
loss = cross_entropy(logits.view(B*T, V), targets.view(B*T))
```

Flatten batch and time into one list of predictions, compare against the flat targets. Two
details that matter:

- PyTorch's `cross_entropy` expects **raw logits**, not softmax output. Applying softmax
  first is a real and common bug — it trains, badly.
- `ignore_index` makes it skip padded positions (Topic 7's mask).

**Your baseline before any training:** a random model spreads probability evenly over V
tokens, so loss should be about `ln(V)`. If your untrained loss is far from that, something
is wrong before training even starts. This is the single most useful sanity check in this
topic.

## 8.7 Counting parameters, and what they're for

Know where the weights live:

```
token embedding    V * C
position embedding T * C
per block:  attention  4 * C * C      (Q, K, V, output)
            feed-forward 8 * C * C    (expand 4x, contract)
total per block ≈ 12 * C * C
output head        C * V              (0 if tied)
```

So parameters ≈ `12 * n_layer * C²` plus embeddings. Two consequences: most parameters sit
in the feed-forward layers, and width (C) costs quadratically while depth costs linearly.

## 8.8 Initialization and dropout

- **Initialization** — start weights small and random (normal with std ~0.02). Too large and
  activations explode through the stack; all zeros and every neuron learns identically.
- **Dropout** — randomly zero a fraction of activations during training only, so the model
  can't lean on any single path. Must be disabled at inference (`model.eval()`), or your
  generations become randomly damaged. Forgetting this is a classic bug.

---

# Part 2 — Questions to implement

Build `model.py` in this folder, using PyTorch. Keep the model configurable and small:
`n_layer=4, n_head=4, C=128, block_size=128` trains on a laptop.

### Q1. Environment
**Build:** create and activate a virtual environment, install torch, and print the version
and the available device (`cuda`, `mps`, or `cpu`).
**Check:** create a tensor, move it to the device, do one multiply.
**Explain:** which device are you on, and what does that imply about the model size you
should attempt?

### Q2. Shape discipline
**Build:** a script that makes a random `(B, T)` integer tensor and pushes it through an
`nn.Embedding`, printing the shape at each step.
**Check:** shapes match §8.2.
**Explain:** write out the meaning of every dimension in `(B, T, C)`.

### Q3. Embedding layer module
**Build:** a module combining token and position embeddings.
**Check:** the same token at two positions gives different output vectors. Input `(B, T)` →
output `(B, T, C)`.
**Explain:** why must position information be added rather than concatenated?

### Q4. Causal self-attention, in PyTorch
**Build:** one multi-head causal attention module. Register the mask as a buffer (not a
parameter — it isn't learned).
**Check:** compare its output on a small input against your hand-written Topic 4
implementation and confirm they agree to a few decimal places. Then verify causality
directly: change the **last** input token and confirm earlier positions' outputs do not move.
**Explain:** why is that second test the definitive proof the mask works?

### Q5. Block
**Build:** pre-norm block: attention with residual, feed-forward with residual.
**Check:** input shape equals output shape, so blocks can stack. Ten stacked blocks don't
blow up in magnitude.
**Explain:** what would happen to the output scale without the residual connections?

### Q6. The full model
**Build:** embeddings → N blocks → final layer norm → output projection to V. Add an optional
`targets` argument that also returns the loss.
**Check:** `(B, T)` in → logits `(B, T, V)` out.
**Explain:** why does the model produce a prediction at every position rather than only the
last?

### Q7. The loss sanity check
**Build:** compute loss on random targets with a freshly initialized model.
**Check:** it is close to `ln(V)`. Compute `ln(V)` yourself and compare.
**Explain:** why exactly `ln(V)`? What does a much *lower* initial loss suggest about your
masking?

### Q8. Parameter count
**Build:** count parameters, grouped by component (embeddings, attention, feed-forward, head).
**Check:** your count roughly matches the `12 * n_layer * C²` estimate.
**Explain:** double C and report the change; double n_layer and report the change. Which is
more expensive, and why?

### Q9. Weight tying
**Build:** make tying optional. Count parameters both ways.
**Check:** tied mode saves exactly `V × C`.
**Explain:** why is tying a reasonable thing to do at all?

### Q10. Dropout and modes
**Build:** add dropout to attention and feed-forward. Run the same input twice in
`model.train()` and twice in `model.eval()`.
**Check:** train mode gives different outputs each time; eval mode is identical.
**Explain:** what would a user see if you forgot `model.eval()` before generating?

### Q11. Overfit one batch — the real readiness test
**Build:** take a single batch and train on only that batch for a few hundred steps (a
minimal loop; Topic 9 does this properly).
**Check:** loss goes to near zero.
**Explain:** if loss *cannot* reach near zero on one batch, the bug is in the model, not the
data or the learning rate. Why is that a reliable inference?

---

# Done when you can answer

1. What do B, T, C and V mean, and what shape flows where?
2. What does autograd remove the need to write?
3. Why is pre-norm plus residuals the standard block layout?
4. Why should untrained loss be about `ln(V)`?
5. Where do most parameters live, and how do width and depth differ in cost?
6. What is weight tying, and why is it justified?
7. Why does dropout have to be off at inference?
8. Why is "overfit one batch" the right first test?

Write answers in `notes.md`.
