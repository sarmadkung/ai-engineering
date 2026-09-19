# Topic 51 — Model Optimization

**Why this topic:** making a model smaller, faster or cheaper without making it noticeably worse.
These four techniques are how a model that needs a data-centre GPU ends up on a laptop — and each one
trades something specific.

---

# Part 1 — Theory

## 51.1 Quantization

Fewer bits per weight (Topic 45's practical view; here the mechanics).

**Post-training quantization (PTQ)** — quantize an already-trained model. Cheap, no retraining.
Methods differ in how carefully they choose scales: naive rounding, then **GPTQ** (layer-wise,
minimizing output error on calibration data), **AWQ** (protects the weights that matter most for
activations), and **SmoothQuant** (shifts difficulty from activations to weights so both quantize
well).

**Quantization-aware training (QAT)** — simulate quantization during training so the model adapts.
Better quality at low bit-widths; requires a training run.

Practical facts worth carrying: weights quantize far better than activations; a small number of
**outlier** weights/channels cause most of the damage, which is why methods that treat them specially
work; per-channel scales beat per-tensor; and 4-bit is the usual sweet spot, with acceleration
below that coming increasingly from hardware support (FP8/FP4) rather than clever rounding.

Quantizing the **KV cache** is a separate, valuable lever for serving (Topic 46), since the cache
often dominates memory.

## 51.2 Distillation

Train a small **student** to imitate a large **teacher**.

- **Response distillation** — the student trains on the teacher's outputs. Simple, effective, and
  exactly what Topic 50 called distillation. Works through an API.
- **Logit distillation** — the student matches the teacher's full output *distribution*, which carries
  far more information per example than a single sampled token (the teacher's "second choice" is a
  signal). Better results; needs access to the teacher's logits, so usually requires local weights.
- **Feature distillation** — match intermediate representations too. More constraining, needs
  compatible architectures.

Why it works: the teacher's distribution encodes graded similarity between options, which is much
richer supervision than a hard label — the student learns *how wrong* each alternative is.

Reality: a distilled small model can approach the teacher **on the distilled task distribution** and
will not match it in general. So distillation is for narrowing scope, not for free capability. Most
small "surprisingly good" open models are distilled from larger ones.

## 51.3 Pruning

Remove weights or structures.

- **Unstructured** — zero out individual small weights. High sparsity possible, but irregular sparsity
  gives little real speedup without hardware support.
- **Structured** — remove whole heads, channels, or layers. Less compressible, but actually faster on
  normal hardware because the shapes shrink.
- **Depth pruning** — drop entire layers. Surprisingly tolerable in large models, especially in the
  middle of the stack.
- **Experts** — for MoE models, drop rarely-used experts (Topic 12).

Usually: prune → fine-tune to recover → repeat gradually. Aggressive one-shot pruning damages models
badly.

Honest assessment: pruning is less used in practice for LLMs than quantization and distillation,
because quantization gives a bigger win for far less effort. Know it; reach for it last.

## 51.4 Speculative decoding

The one optimization here that changes **latency** rather than size, and it's lossless.

```
1. a small draft model proposes the next k tokens (cheap)
2. the big model verifies all k in ONE forward pass (parallel)
3. accept the longest correct prefix; reject the rest and continue
```

Why it works: verification is compute-bound and parallel, while generation is memory-bandwidth bound
and serial (Topic 45). Checking 5 tokens costs barely more than generating 1, so if the draft is
usually right you get several tokens per expensive pass.

Crucially, **the output distribution is unchanged** — with correct acceptance sampling the result is
exactly what the big model would have produced. This is rare among optimizations: pure speedup, no
quality trade.

Typical gains 1.5–3×, depending on how well the draft agrees. Variants: a separate small model,
**self-speculation** (early layers draft), **Medusa** (extra heads predict several tokens), and
**n-gram/lookup** drafting (cheap, works well on repetitive text like code).

Costs: the draft model's memory and complexity, and reduced benefit at high batch sizes (where the big
model's capacity is already saturated with useful work).

## 51.5 Other levers

- **Flash attention** and fused kernels — faster, more memory-efficient attention. Free if your stack
  supports it.
- **Continuous batching, PagedAttention, prefix caching** — serving-level throughput (Topic 46), often
  a bigger practical win than model-level optimization.
- **Compilation** (`torch.compile`, TensorRT, ONNX) — kernel fusion and graph optimization.
- **Early exit / cascades** — try a small model, escalate on low confidence (Topic 44's routing).

## 51.6 Choosing what to optimize

Start by measuring what's actually binding:

| Symptom | Reach for |
|---|---|
| doesn't fit in memory | quantization, then a smaller/distilled model |
| single-stream generation too slow | quantization, speculative decoding |
| throughput too low under load | batching, PagedAttention (Topic 46) |
| prompt processing slow | shorter prompts, prefix caching |
| cost too high | smaller model, caching, batch API (Topic 48) |
| latency spikes under load | admission control, more replicas |

And the rule: **measure quality after every optimization, on your own task** (Phase 9). Each technique
has a characteristic failure — quantization degrades reasoning first, distillation loses
generality, pruning damages rare capabilities. A benchmark score that holds up is not evidence that
your use case does.

---

# Part 2 — Questions to implement

Build `optimize/` here. You need a model you can run locally and your Phase 9 evaluation set. Some
questions need a GPU; where you can't run something, work the arithmetic and say so.

### Q1. Measure before optimizing
**Build:** baseline for your model: memory, tokens/second, TTFT, evaluation score, cost per 1000
requests.
**Explain:** which constraint actually binds for your intended use?

### Q2. PTQ methods compared
**Build:** quantize to 4-bit with two different methods (e.g. a GGUF Q4 variant and AWQ/GPTQ).
**Check:** tabulate memory, speed, and evaluation score for each plus fp16.
**Explain:** did the smarter method preserve quality better? Was the difference worth the extra work?

### Q3. Where quantization hurts
**Build:** evaluate fp16 vs 4-bit vs 2-3 bit separately on classification, extraction, summarization,
and multi-step reasoning.
**Check:** report a score per task per precision.
**Explain:** which capability degraded first? Explain why reasoning is the canary.

### Q4. KV cache quantization
**Build:** enable 8-bit KV cache if your stack supports it.
**Check:** measure maximum concurrency and quality at a long context.
**Explain:** report the concurrency gain. Was quality affected?

### Q5. Response distillation
**Build:** generate 500 outputs from a strong model for your task; fine-tune a small model on them
(Topic 50).
**Check:** evaluate student, teacher, and the un-tuned small model.
**Explain:** how much of the gap did the student close on the task?

### Q6. Distillation doesn't generalize
**Build:** evaluate your student on 15 questions *outside* the distilled task.
**Check:** compare with the teacher and with the base small model.
**Explain:** report the numbers. What exactly did distillation buy, and what did it not?

### Q7. Logit vs response distillation
**Build:** if you can access teacher logits locally, train a second student on the distribution.
**Check:** compare with the response-only student at equal data size.
**Explain:** did it help? Explain the information difference.

### Q8. Layer pruning
**Build:** remove 2–4 middle layers from a model and evaluate. Then fine-tune briefly to recover.
**Check:** report score, memory and speed at each stage.
**Explain:** how much did pruning cost, and how much did recovery return? Would you use this?

### Q9. Pruning vs quantization at equal memory
**Build:** compare a pruned model and a quantized model of roughly equal memory footprint.
**Check:** evaluate both.
**Explain:** which won? Does this match §51.3's assessment?

### Q10. Speculative decoding
**Build:** a small draft model with a larger target, with configurable draft length k.
**Check:** measure tokens/second and the **acceptance rate** at k = 2, 4, 8.
**Explain:** report the speedup curve. Where's the optimum, and why does too-large k stop helping?

### Q11. Speculative decoding is lossless
**Build:** generate the same prompt with and without speculation at temperature 0.
**Check:** the outputs are identical.
**Explain:** why can this be lossless when quantization cannot?

### Q12. Draft quality matters
**Build:** speculation with a well-matched draft model and with a poorly-matched one.
**Check:** report acceptance rate and speedup for both.
**Explain:** what makes a good draft model?

### Q13. Batch size interaction
**Build:** measure speculative decoding's benefit at batch size 1, 8, and 32.
**Explain:** why does the gain shrink with batch size? Relate to memory bandwidth versus compute.

### Q14. The optimization decision
**Build:** for one real deployment target, combine the techniques that help and measure the total
effect against your Q1 baseline.
**Check:** report memory, speed, quality, and cost per 1000 requests.
**Explain:** what did you achieve overall, what did you give up, and which technique earned the most
per hour of your effort?

---

# Done when you can answer

1. What do GPTQ and AWQ do that naive rounding doesn't, and why do outliers matter?
2. Why does logit distillation carry more information than response distillation?
3. What does distillation buy, and what does it not?
4. Why is structured pruning faster in practice than unstructured?
5. Why is speculative decoding lossless, and why does it work at all?
6. Why does speculation help less at large batch sizes?
7. Which capability degrades first under quantization, and how would you detect it?

Write answers in `notes.md`.
