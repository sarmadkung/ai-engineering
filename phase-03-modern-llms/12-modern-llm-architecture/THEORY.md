# Topic 12 — Modern LLM Architecture

**Why this topic:** the five substitutions that turn your Topic 8 model into a 2025-shaped
one. Each is an efficiency win inside the same skeleton. After this topic you can read any
open-source model's code without surprises.

---

# Part 1 — Theory

## 12.1 RMSNorm instead of LayerNorm

LayerNorm subtracts the mean, divides by the standard deviation, then scales and shifts with
two learned vectors.

**RMSNorm** skips the mean entirely: divide by the root-mean-square of the vector, then scale
by one learned vector.

```
LayerNorm:  (x - mean) / std * gamma + beta
RMSNorm:    x / sqrt(mean(x^2)) * gamma
```

Why it won: the mean-subtraction and the shift turn out not to matter for quality, and removing
them saves computation and one parameter vector per layer. Simpler, faster, same result — used
in Llama, Mistral, Qwen, Gemma.

## 12.2 RoPE — rotary position embeddings

The problem with learned positions: position 4096 has a vector only if you trained one, so
context length is frozen at training time. And an *absolute* position vector is a strange thing
to add — what matters linguistically is usually **relative** distance.

**RoPE** injects position by **rotating** the query and key vectors by an angle proportional to
their position, in 2-D pairs of dimensions, before computing attention scores.

The elegant consequence: when you dot a rotated query at position `m` with a rotated key at
position `n`, the result depends only on `m - n` — the **relative** distance. Absolute position
drops out of the arithmetic.

Practical wins: no position parameters at all; attention naturally expresses "how far apart";
and context can be extended after training by scaling the rotation frequencies (this is how
"32k context" versions of 4k models are made). RoPE is applied to Q and K only, not V, and
inside every layer rather than once at the input.

## 12.3 SwiGLU — a better feed-forward

Standard FFN: expand to 4×, apply an activation, contract back.

**SwiGLU** uses a *gate*: compute two projections from the same input, pass one through a
sigmoid-based activation (SiLU/Swish), multiply them elementwise, then contract.

```
standard:  down( relu( up(x) ) )
SwiGLU:    down( silu(gate(x)) * up(x) )
```

The multiplication lets the network decide, per dimension, how much signal to let through —
a learned, input-dependent filter rather than a fixed nonlinearity. It costs a third
projection, so implementations shrink the hidden width (4× → ~2.7×) to keep parameters
roughly equal. Better quality at the same budget.

## 12.4 KV cache

The inference optimization you felt the need for in Topic 10.

Generating token `n` recomputes keys and values for tokens 1..n-1, which never change. So
store them:

```
without cache:  step n processes n tokens   -> total work grows with n^2
with cache:     step n processes 1 token    -> total work grows with n
```

The cost is memory, and it is substantial:

```
cache size ≈ 2 (K and V) × layers × heads × head_dim × sequence_length × batch × bytes
```

For long contexts and many concurrent users this can exceed the model weights. That single
fact drives most of Phase 11/12's serving concerns — and motivates the next section.

## 12.5 Grouped-query attention (GQA)

The KV cache scales with the number of heads, so shrinking heads shrinks the cache.

- **Multi-head attention (MHA)** — every head has its own K and V. Best quality, biggest cache.
- **Multi-query attention (MQA)** — all heads *share* one K and V. Cache shrinks by the head
  count; quality drops noticeably.
- **Grouped-query attention (GQA)** — heads are divided into groups, each group sharing one K
  and V. E.g. 32 query heads, 8 KV heads: 4× smaller cache, quality close to MHA.

GQA is the standard compromise in modern open models. Queries stay many; keys and values get
few.

## 12.6 Mixture of Experts (MoE)

Most parameters live in the FFN. MoE replaces one FFN with many "experts" plus a small router
that picks a couple per token.

```
dense:  every token -> the one FFN                       (all parameters used)
MoE:    every token -> router -> 2 of 64 experts         (a fraction used)
```

So a model can hold 400B parameters but *use* 30B per token: huge capacity, modest compute per
token. Different tokens take different routes, and the router learns to specialize experts.

The catches are real: all experts must sit in memory even though most are idle, routing can
collapse onto a few favourites (needing a load-balancing loss), and training is fiddlier. Used
by Mixtral, DeepSeek, and most frontier models.

**The terminology that trips people up:** "total parameters" vs "active parameters". An MoE's
quality tracks total; its speed and its per-token compute track active.

---

# Part 2 — Questions to implement

Modify your Topic 8 model in this folder (`modern_model.py`). Change one thing at a time, and
measure each: same data, same steps, same seed.

### Q1. RMSNorm
**Build:** RMSNorm, and swap it in for LayerNorm.
**Check:** output vectors have RMS ≈ 1. Parameter count drops by one vector per norm. Loss
curve is comparable.
**Explain:** report step-time and parameter difference. Was quality affected?

### Q2. RoPE
**Build:** rotary embeddings applied to Q and K inside attention. Remove position embeddings
entirely.
**Check:** the model still trains. The position embedding table is gone from the parameter
count. Verify the relative-distance property numerically: the score between positions (5, 7)
matches that between (105, 107) for the same content vectors.
**Explain:** that numerical check is the whole point of RoPE. Say what it proves.

### Q3. Context extension
**Build:** train with block_size 128, then generate with 256 by scaling RoPE frequencies.
**Check:** it runs, and output degrades gracefully rather than crashing.
**Explain:** try the same with learned position embeddings. What happens, and why is that the
key advantage?

### Q4. SwiGLU
**Build:** the gated FFN, with hidden width reduced so total parameters stay ~equal.
**Check:** parameter counts within a few percent; validation loss compared fairly.
**Explain:** did it help at equal parameters? What does the gate let the network do that ReLU
cannot?

### Q5. KV cache memory
**Build:** compute the cache-size formula for your model at sequence lengths 128, 1k, 8k, 32k,
with batch 1 and batch 32. Print a table in megabytes.
**Explain:** where does the cache exceed your model's weight size? What would you do about it
if you were serving this model?

### Q6. GQA
**Build:** make the number of KV heads configurable: equal to query heads (MHA), one (MQA), and
a divisor like 4 (GQA).
**Check:** all three train. Cache size scales exactly with KV head count.
**Explain:** tabulate loss and cache size for the three. Which trade would you ship?

### Q7. A small MoE
**Build:** replace the FFN with 4 experts and a top-2 router. Track how often each expert is
selected.
**Check:** total parameters rise substantially; per-token compute does not.
**Explain:** report total vs active parameters and your expert usage histogram. Is routing
balanced? If not, what would you add?

### Q8. Everything together
**Build:** one model with RMSNorm + RoPE + SwiGLU + GQA. Train against your Topic 8 baseline at
matched parameters.
**Check:** report parameters, step time, best validation loss, and inference tokens/sec for
both.
**Explain:** which change gave the biggest win on which axis? Which mattered least for a model
your size, and why might it matter more at scale?

### Q9. Read the real thing
**Build:** open a modern open-source model's modelling file (Llama or Qwen in `transformers`)
and locate each of the five features in the code.
**Explain:** note one thing they do that you did not, and what it is for.

---

# Done when you can answer

1. What does RMSNorm drop, and why is that fine?
2. How does RoPE encode position, and what property makes it special?
3. Why can RoPE extend context when learned embeddings cannot?
4. What does the gate in SwiGLU do?
5. Why does a KV cache change generation from quadratic to linear, and what does it cost?
6. How does GQA shrink the cache, and what is traded?
7. Total vs active parameters in an MoE — what does each predict?

Write answers in `notes.md`.
