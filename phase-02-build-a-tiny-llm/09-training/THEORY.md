# Topic 9 — Training

**Why this topic:** this is where your model stops being random. You write the loop that
turns the loss from Topic 8 into learned weights — and learn to read a loss curve, which is
the core diagnostic skill of all model work.

---

# Part 1 — Theory

## 9.1 The loop

Five steps, repeated:

```
1. get a batch of (inputs, targets)
2. forward pass   -> logits -> loss          "how wrong am I?"
3. backward pass  -> gradients               "which way is each weight wrong?"
4. optimizer step -> update weights          "move a little that way"
5. zero the gradients                        "forget it, ready for next batch"
```

In PyTorch: `loss.backward()`, `optimizer.step()`, `optimizer.zero_grad()`. Forgetting step
5 makes gradients accumulate across batches, so updates are based on stale information and
training quietly degrades.

## 9.2 Forward and backward

**Forward** runs your model and records every operation in a graph. **Backward** walks the
graph in reverse applying the chain rule, producing for every parameter a gradient: the
direction in which increasing that parameter increases the loss.

You move **against** the gradient, because you want loss to go down. That is gradient
descent. Backward costs roughly twice a forward pass, which is why training a model is
~3× the cost of running it.

## 9.3 Learning rate — the setting that matters most

The gradient says which direction; the learning rate says how far.

- **Too high** — loss jumps around, spikes, or becomes `NaN`. The step overshoots the valley.
- **Too low** — loss drops painfully slowly, or stalls.
- **Right** — smooth, steady decline.

For a small transformer, start near `3e-4`. If loss is `NaN` within a few steps, your
learning rate is almost certainly too high.

**Schedules** — the rate should change over training:

- **Warmup** — start near zero and ramp up over the first few hundred steps. Early gradients
  are large and unreliable; a big early step can wreck the run permanently.
- **Cosine decay** — after warmup, decay smoothly toward near zero. Big steps early to find
  the right region, small steps late to settle in it.

## 9.4 Optimizers

**SGD** subtracts `learning_rate × gradient`. Simple, but sensitive to how differently
scaled the parameters are.

**Adam / AdamW** keeps a running average of each parameter's gradient (momentum) and of its
squared magnitude, then scales each parameter's step accordingly — so rarely-updated
parameters still move usefully. It is the default for language models, at the cost of
storing two extra numbers per parameter.

**Weight decay** pulls weights gently toward zero, discouraging over-reliance on any one.
**AdamW** is the version that applies decay correctly, which is why it's what people use.

## 9.5 Reading the two curves

Track **training loss** and **validation loss** together. That pair is the whole diagnosis:

| Pattern | Meaning | Action |
|---|---|---|
| both falling | learning normally | continue |
| train falls, validation rises | **overfitting** — memorizing | stop, or add data/regularization |
| both flat and high | not learning | check learning rate, shapes, data pipeline |
| loss = `NaN` | numerical blow-up | lower learning rate, add gradient clipping |
| loss spikes then recovers | bad batch or rate slightly high | watch; clip gradients |
| validation lower than train | leakage or a bug | investigate immediately |

The point where validation loss turns upward is the moment to stop. That's **early stopping**,
and it's why you keep the checkpoint with the best *validation* loss rather than the last one.

## 9.6 Gradient clipping and accumulation

**Clipping** — rescale the gradient if its total magnitude exceeds a threshold (typically 1.0).
Prevents one unusual batch from producing a huge, destructive step. Cheap insurance; standard
practice.

**Accumulation** — if the batch size you want doesn't fit in memory, run several small
batches, add up their gradients, and step once. This simulates a large batch on small
hardware. Just remember to divide the loss so the average is right.

## 9.7 Checkpoints

Save the model state, optimizer state, step number, and config — not just the weights. Without
optimizer state, resuming restarts momentum and the loss visibly jumps.

Save periodically (crashes happen) and keep the best-validation checkpoint separately from
the latest one. Config must be saved with the weights, because loading needs to rebuild
exactly the same architecture.

## 9.8 Why compute matters (scaling laws, informally)

Loss falls predictably with more parameters, more data, and more compute — smoothly, over
orders of magnitude. That predictability is why labs can justify huge training runs before
seeing results.

The practical corollary for you: your tiny model on a small corpus has a floor it cannot go
below, and no amount of tuning will break it. Recognizing "this is a capacity limit, not a
bug" is the lesson of Question 10.

---

# Part 2 — Questions to implement

Build `train.py` in this folder, importing your Topic 7 dataset and Topic 8 model.

### Q1. The minimal loop
**Build:** the five steps of §9.1 over batches, printing loss every N steps.
**Check:** loss starts near `ln(V)` and falls.
**Explain:** what does each of the five steps do? What happens if you omit `zero_grad()`?
(Try it and report.)

### Q2. Learning rate sweep
**Build:** run 200 steps at learning rates `1e-2, 1e-3, 3e-4, 1e-5`, recording the curve for
each.
**Check:** the highest is unstable or `NaN`; the lowest barely moves.
**Explain:** which did you choose and on what evidence?

### Q3. Validation loss
**Build:** every N steps, evaluate on the validation split with gradients disabled and
`model.eval()`, then switch back to `model.train()`.
**Check:** evaluation does not change the weights. Both losses are logged together.
**Explain:** why must gradients be off and eval mode on for this?

### Q4. Make overfitting happen on purpose
**Build:** train a deliberately oversized model on a deliberately small slice until
validation loss turns upward. Plot or print both curves.
**Check:** the divergence is clearly visible.
**Explain:** identify the step where you should have stopped, and explain what the model was
doing after that point.

### Q5. Warmup and cosine decay
**Build:** a scheduler: linear warmup for the first ~100 steps, then cosine decay.
**Check:** print the rate at steps 0, 50, 100, 500, 1000 and confirm the shape.
**Explain:** re-run Q2's unstable learning rate *with* warmup. Did it survive? Why?

### Q6. Optimizer comparison
**Build:** the same run with SGD and with AdamW.
**Check:** AdamW reaches a lower loss in the same number of steps.
**Explain:** what is AdamW doing per parameter that SGD is not?

### Q7. Gradient clipping
**Build:** add clipping at 1.0. Log the gradient norm each step.
**Check:** norms are large early and settle down; clipping engages mostly at the start.
**Explain:** what does one un-clipped enormous gradient do to a model?

### Q8. Checkpoints and resume
**Build:** save model + optimizer + step + config periodically, keep the best-validation
copy, and support resuming.
**Check:** kill the run mid-training, resume, and confirm the loss continues smoothly rather
than jumping.
**Explain:** resume **without** optimizer state and describe what you see. Why?

### Q9. Gradient accumulation
**Build:** accumulate over 4 micro-batches and step once.
**Check:** the loss curve resembles a real batch 4× larger, and memory use stays near the
micro-batch.
**Explain:** where did you divide the loss, and what happens if you don't?

### Q10. Find your model's floor
**Build:** train to convergence. Then double the model size and train again. Then double the
data and train again.
**Check:** record best validation loss for all three.
**Explain:** which change helped more? Is your bottleneck capacity or data? How do you know?

### Q11. A trained model you can keep
**Build:** train your best configuration to completion, saving the best checkpoint.
**Check:** report initial loss, best validation loss, and perplexity (`exp(loss)`).
**Explain:** compare this perplexity to your Topic 1 n-gram result, and state clearly why the
two numbers are only comparable if the tokenizer is the same.

---

# Done when you can answer

1. What are the five steps of the training loop, and what breaks without each?
2. What does the learning rate control, and what do too-high and too-low look like?
3. Why warmup? Why decay?
4. What does the train/validation gap tell you, and when do you stop?
5. Why clip gradients?
6. What must a checkpoint contain besides weights?
7. How do you tell a capacity limit from a bug?

Write answers in `notes.md`.
