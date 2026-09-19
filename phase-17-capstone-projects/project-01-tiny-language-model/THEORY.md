# Project 1 — Tiny Language Model From Scratch

**Depends on:** Phases 1–2 (Topics 1–10). **Do this one first.**

**What you prove:** that you understand LLMs mechanically, not by analogy. Everything in later phases
is easier once this exists and works.

---

# Part 1 — What you are building

A complete, trained, working language model with no borrowed model code:

```
your tokenizer -> your dataset pipeline -> your transformer -> your training loop -> your sampler
```

PyTorch for tensors and autograd; every layer written by you. Target: a model of 5–50M parameters
trained on 10–100 MB of text, producing fluent-shaped English with real words and plausible sentence
rhythm.

**Deliberately not the goal:** a useful model. At this scale the output will be locally plausible and
globally meaningless. That is the correct outcome, and mistaking it for failure is the main way people
quit at this stage.

---

# Part 2 — Design decisions to make deliberately

Write your choice and reason for each in `DECISIONS.md` before you code:

| Decision | Options | What it affects |
|---|---|---|
| Tokenizer | character / byte-level BPE | vocabulary size, context in characters, training speed |
| Vocabulary size | 1k / 8k / 32k | embedding size, sequence length, output layer cost |
| Corpus | books / code / your own writing | what the output sounds like |
| Model size | layers × width × heads | quality ceiling, training time |
| Context length | 128 / 256 / 512 | memory, what the model can reference |
| Position encoding | learned / sinusoidal / RoPE | context extensibility |
| Norm and activation | LayerNorm+GELU / RMSNorm+SwiGLU | modernity, minor quality |
| Optimizer and schedule | AdamW + warmup + cosine | stability, final loss |

Sensible first configuration: byte-level BPE at 8k vocabulary, 6 layers, 384 wide, 6 heads, context 256,
~15M parameters. It trains in hours on a laptop GPU or an hour on a rented one.

---

# Part 3 — Milestones

Each milestone has a check you must pass before continuing. Do not skip them; every one catches a bug
class that is painful to diagnose later.

### M1 — Tokenizer (Topics 2, 6)
Train BPE on your corpus; `encode`/`decode`/`save`/`load`.
**Check:** exact round trip on prose, code, emoji, and non-English text. Report bytes per token.

### M2 — Dataset (Topic 7)
Token stream, `<eos>` between documents, fixed-length chunks, input/target pairs shifted by one,
batching, train/validation split by document.
**Check:** decode one (input, target) pair and read it aloud — target must be input shifted left by one.
This single check prevents the most common silent disaster in the project.

### M3 — Model (Topics 4, 8, 12)
Embeddings + position, causal multi-head attention, feed-forward, residuals, norm, output projection.
**Checks, all of them:**
- untrained loss ≈ `ln(vocab_size)` — if it's much lower, your causal mask is broken
- changing the last input token does not change earlier positions' outputs
- parameter count matches your estimate
- **it can overfit a single batch to near-zero loss** — if not, the bug is in the model

### M4 — Training (Topic 9)
Loop with AdamW, warmup + cosine schedule, gradient clipping, train/validation logging, checkpoints
(model + optimizer + step + config), resume.
**Check:** kill it mid-run and resume with a continuous loss curve. Loss falls from `ln(V)` steadily.

### M5 — Inference (Topics 5, 10)
Generation loop, temperature/top-k/top-p, stop conditions, streaming output.
**Check:** it produces text. Same seed and settings → identical output.

### M6 — KV cache (Topics 10, 12)
**Check:** cached and uncached generation produce **identical** text at a fixed seed, and generation of
400 tokens is several times faster. This equality test is the whole point — a cache that changes output
is a bug, not an optimization.

### M7 — Train it properly
Run to convergence on your full corpus. Keep the best-validation checkpoint.
**Check:** report initial loss, best validation loss, perplexity, tokens/second, and wall time.

---

# Part 4 — Experiments worth running

These are what turn a working project into understanding:

1. **Scale** — train 3 sizes (small/medium/large by your standards) on the same data. Plot validation
   loss against parameters. Do you see a scaling curve?
2. **Data quantity** — the same model on 10%, 50%, 100% of the corpus. Which is your bottleneck —
   capacity or data?
3. **Context length** — 64 vs 256 vs 512. Does more context help, and what does it cost?
4. **Modern components** — RMSNorm + RoPE + SwiGLU versus your baseline at matched parameters. Report
   the difference honestly; at this scale it may be small.
5. **Tokenizer** — character-level versus BPE. Compare perplexity (noting it isn't directly comparable),
   tokens/second, and readability of samples.
6. **Sampling** — the same checkpoint at greedy, T=0.8+top_p=0.9, T=1.5. Which settings flatter your
   model most, and which expose it?

---

# Part 5 — Deliverables

- `tokenizer.py`, `dataset.py`, `model.py`, `train.py`, `generate.py` — all yours
- `DECISIONS.md` — your configuration choices and reasons
- `RESULTS.md` — configuration, parameter count, corpus size, loss curves, perplexity, tokens/second,
  and 5 sample generations at different settings
- `EXPERIMENTS.md` — the six experiments with numbers and conclusions
- A trained checkpoint you can load and sample from

---

# Part 6 — Done when

- The model trains from scratch with one command and produces recognizable English-shaped text.
- Every milestone check passes, including the cache-equality test.
- You can explain, without notes: what each tensor shape means, why loss starts at `ln(V)`, what the
  causal mask prevents, why generation is sequential, and what the KV cache changes.
- You can point at the exact lines implementing attention, the loss, and the sampler.

---

# Common failures, and what they mean

| Symptom | Almost certainly |
|---|---|
| initial loss far below `ln(V)` | causal mask broken — the model sees the future |
| loss falls then output is gibberish | off-by-one in input/target shift |
| loss `NaN` early | learning rate too high, or no warmup |
| cannot overfit one batch | bug in the model, not the data |
| output is one repeated token | greedy decoding, or a collapsed model |
| cached output ≠ uncached | cache indexing bug |
| training loss great, samples terrible | overfitting; check validation loss |
