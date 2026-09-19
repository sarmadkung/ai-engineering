# Topic 52 — Training Systems

**Why this topic:** Phase 3 explained distributed training conceptually. This topic is the engineering:
what each parallelism strategy costs, how memory is actually consumed, and how to run a multi-GPU job
that survives. Even if you never train a frontier model, this determines what you can fine-tune and
why a job won't fit.

---

# Part 1 — Theory

## 52.1 The memory problem

Training memory, per parameter, with AdamW in mixed precision:

```
weights (bf16)            2 bytes
gradients (bf16)          2 bytes
optimizer momentum (fp32) 4 bytes
optimizer variance (fp32) 4 bytes
fp32 master weights       4 bytes
                        ≈ 16 bytes per parameter
```

Plus **activations**, which scale with batch size × sequence length × layers and often dominate.

So a 7B model needs ~112 GB before activations — more than a single 80 GB GPU. **Optimizer state is
usually bigger than the model**, which is the fact that motivates everything below, and the reason
LoRA (Topic 14) makes single-GPU fine-tuning possible at all: it eliminates optimizer state for 99%+
of parameters.

## 52.2 Data parallelism

Every GPU holds the full model, processes a different slice of the batch, and gradients are averaged
with an **all-reduce** each step.

Simple and near-linear in throughput, but requires the model plus optimizer state to fit on one GPU,
and communicates the full gradient every step. Effective batch size = per-GPU batch × GPU count ×
accumulation steps — which matters, because a much larger batch usually needs a learning-rate
adjustment.

## 52.3 Sharded data parallelism (ZeRO / FSDP)

The technique that makes large-scale training practical: shard the optimizer state, gradients, and
parameters **across** GPUs instead of replicating them.

```
ZeRO-1: shard optimizer state         ~4x memory reduction on state
ZeRO-2: + shard gradients
ZeRO-3 / FSDP: + shard parameters     near-linear memory reduction
```

With ZeRO-3, each GPU holds a slice of the weights and gathers what it needs for each layer during
the forward pass, then frees it. Memory falls dramatically; communication rises. Add **CPU offload**
to trade speed for the ability to fit at all.

This is the first thing to reach for when a job doesn't fit, and it's what Hugging Face
`accelerate`/DeepSpeed configure for you.

## 52.4 Tensor (model) parallelism

Split individual layers' matrices across GPUs; each computes part of the operation and they combine
results **within every forward and backward pass**.

Necessary when a single layer doesn't fit. Extremely communication-heavy, so it stays **inside one
node** with fast interconnect (NVLink). Typical degree: 2–8.

## 52.5 Pipeline parallelism

Different **layers** on different GPUs; micro-batches flow through the stages.

Communication is small (activations at stage boundaries only), so it works across nodes. The cost is
the **bubble** — idle time while stages wait for work to arrive. More micro-batches shrink the bubble;
interleaved schedules (1F1B) shrink it further.

## 52.6 Combining them (3D parallelism)

Large runs combine all three, plus sharding:

```
tensor parallel    — within a node (fast interconnect)
pipeline parallel  — across nodes
data parallel      — across pipeline replicas
```

Choosing the split is an optimization problem over memory, interconnect bandwidth, and bubble size.
The rule of thumb: **use the cheapest form that makes it fit** — data parallel first, then sharding,
then tensor parallel within a node, then pipeline across nodes.

## 52.7 Mixed precision

Compute in a 16-bit format, keep master weights in fp32.

- **fp16** — narrow range, so it needs **loss scaling** to stop small gradients underflowing to zero.
- **bf16** — same exponent range as fp32, less mantissa. No loss scaling needed. The default on modern
  hardware, and the right choice when available.
- **fp8** — newest hardware; more care required.

Benefits: roughly 2× throughput and half the memory for activations and gradients. The usual failure
mode is fp16 instability — `NaN` losses that bf16 would not have produced.

## 52.8 Other essential techniques

- **Gradient checkpointing** — don't store all activations; recompute them in the backward pass.
  Typically ~30% slower for a large memory saving. The standard trade when activations are the problem.
- **Gradient accumulation** — simulate a big batch on small hardware (Topic 9).
- **Flash attention** — faster attention with O(n) rather than O(n²) memory.
- **Sequence/context parallelism** — split a long sequence across GPUs when even one sample's
  activations don't fit.
- **Efficient data loading** — GPUs idle waiting for data is a common and embarrassing bottleneck;
  prefetch, use multiple workers, and stream from pre-tokenized shards.

## 52.9 Running a real job

- **Checkpointing** must include model, optimizer, scheduler, step, and **data position** (Topic 33) —
  and it must be sharded and fast, or it dominates wall time.
- **Fault tolerance** — at scale, nodes fail. Expect restarts; automate resume.
- **Monitoring** — loss, gradient norm, learning rate, throughput (tokens/second), **MFU**, GPU
  utilization and memory. A loss spike or a falling MFU means intervene.
- **Cost control** — spot instances plus frequent checkpoints; measure MFU (40–50% is good) and treat a
  low number as recoverable money.

## 52.10 What this means for you

You will not train a frontier model. You *will*: fine-tune models that don't fit naively, diagnose
OOM errors correctly, choose between LoRA and full fine-tuning on evidence, configure
`accelerate`/DeepSpeed, and know whether a job is worth its cost. That's the practical payoff.

---

# Part 2 — Questions to implement

Build `training/` here. Single-GPU or CPU is enough for most of this; simulate multi-GPU where you
can't run it (and say so). `accelerate` and `deepspeed` are the tools.

### Q1. The memory calculator
**Build:** a function computing training memory — weights, gradients, optimizer state, master weights,
activations — for a given model size, batch size, sequence length, and precision.
**Check:** validate against a real run's measured peak memory.
**Explain:** how close was it? Which term did you underestimate?

### Q2. What fits
**Build:** for your hardware, compute the largest model you could train with (a) full fine-tuning,
(b) LoRA, (c) QLoRA.
**Check:** verify one of them empirically.
**Explain:** report the three sizes. Why is the gap so large?

### Q3. Optimizer state dominates
**Build:** measure memory for the same model with AdamW, SGD with momentum, and plain SGD.
**Check:** report peak memory for each.
**Explain:** report the differences. Why is AdamW worth its memory anyway?

### Q4. Mixed precision
**Build:** train the same small model in fp32, fp16, and bf16.
**Check:** report memory, throughput, and loss curves.
**Explain:** did fp16 need loss scaling? Did it ever produce `NaN`? Why doesn't bf16?

### Q5. Gradient checkpointing
**Build:** enable it and measure.
**Check:** report memory saved and throughput lost.
**Explain:** report the trade. At what point would you accept it?

### Q6. Activations vs weights
**Build:** measure peak memory at sequence lengths 512, 2048, 8192 for a fixed model.
**Check:** plot it.
**Explain:** where do activations overtake the weights? What does that mean for long-context training?

### Q7. Gradient accumulation equivalence
**Build:** batch size 32 directly, versus 8 with 4 accumulation steps.
**Check:** confirm the resulting updates and loss curves match closely.
**Explain:** where did you have to divide the loss, and what happens if you don't?

### Q8. Effective batch size and learning rate
**Build:** train at effective batch sizes 8, 32, and 128 with the same learning rate, then with a
scaled one.
**Check:** compare loss curves.
**Explain:** what happened at large batch with an unadjusted rate? What scaling rule did you apply?

### Q9. Data loading bottleneck
**Build:** deliberately slow the data pipeline (single worker, tokenizing on the fly), then optimize it
(pre-tokenized shards, prefetch, multiple workers).
**Check:** measure GPU utilization and tokens/second for both.
**Explain:** report utilization before and after. How much money would the slow version waste on a
rented GPU?

### Q10. MFU
**Build:** compute MFU — achieved FLOPs versus your hardware's peak — using the 6ND estimate
(Topic 13).
**Check:** report your number.
**Explain:** is it in the 40–50% range? If lower, what's the likely cause?

### Q11. Sharded training with accelerate/DeepSpeed
**Build:** configure ZeRO-2 or ZeRO-3 (multi-GPU if available, single-GPU with CPU offload otherwise).
**Check:** report memory and throughput against the unsharded run.
**Explain:** what did sharding buy, and what did communication cost?

### Q12. Simulate the parallelism trade-offs
**Build:** a model estimating, for a given cluster (GPU count, interconnect bandwidth), the memory and
communication cost of several (data, tensor, pipeline) configurations.
**Check:** find the best configuration for a 70B fine-tune on 8 and on 64 GPUs.
**Explain:** report the chosen splits and the reasoning. Which constraint decided it?

### Q13. Fault-tolerant training
**Build:** a training run with sharded checkpoints including data position, plus automatic resume. Kill
it twice at random points.
**Check:** the loss curve is continuous across both restarts and no data is re-seen.
**Explain:** what did checkpointing cost as a fraction of total wall time? How would you tune the
interval?

### Q14. LoRA vs full fine-tuning, decided
**Build:** the same task both ways on a model where both are feasible.
**Check:** report quality, memory, time, and cost.
**Explain:** was full fine-tuning worth it? At what model size does the answer become forced rather
than chosen?

---

# Done when you can answer

1. Why does training need ~16 bytes per parameter, and which term is biggest?
2. What does ZeRO/FSDP shard, and what does it cost?
3. Why does tensor parallelism stay within a node?
4. What is the pipeline bubble, and how is it reduced?
5. Why is bf16 preferred over fp16?
6. What does gradient checkpointing trade?
7. What is MFU, and what does a low value mean?

Write answers in `notes.md`.

---

**Phase 13 is complete.** You can customize and optimize models, and reason about training at scale.
Phase 14 turns to the research frontier: reasoning, memory, and learning systems.
