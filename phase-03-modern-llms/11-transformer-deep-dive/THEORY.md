# Topic 11 — Transformer Deep Dive

**Why this topic:** you built *a* transformer. This topic places it in the family — what the
original paper did, what GPT changed, and why one shape won.

---

# Part 1 — Theory

## 11.1 How the architecture evolved

**Before 2017: RNNs and LSTMs.** Process text one token at a time, carrying a hidden state.
Two fatal problems: they cannot be parallelized over a sequence (token 100 needs token 99's
state), and information from far back gets diluted through hundreds of updates.

**2017: "Attention Is All You Need".** Remove recurrence entirely. Every position attends to
every other directly, so distance is no longer a barrier and all positions compute in
parallel. That parallelism is what made training on internet-scale data possible — the real
breakthrough was as much about hardware efficiency as about modelling.

**2018 onward:** the three branches split (BERT: encoder-only; GPT: decoder-only; T5:
encoder-decoder), and by ~2020 decoder-only had won for generation.

## 11.2 The three shapes

| | Encoder-only | Decoder-only | Encoder-decoder |
|---|---|---|---|
| Attention | bidirectional | causal (masked) | encoder bidirectional, decoder causal + cross |
| Trained by | masking random tokens | predicting the next token | reconstructing corrupted spans |
| Can generate? | no | yes | yes |
| Best at | classification, embeddings, retrieval | everything generative | seq2seq with a clear input/output split |
| Examples | BERT, embedding models | GPT, Llama, Claude, Qwen | T5, BART |

**Encoder-only** sees the whole input at once, so its representation of a word uses both left
and right context — better for understanding, useless for generation (there is no "next").
These are still what powers Phase 5's embedding models.

**Encoder-decoder** adds **cross-attention**: decoder positions form queries while keys and
values come from the encoder's output. Natural for translation, where input and output are
distinct objects.

## 11.3 Why decoder-only won

Not because it's better at any one task, but because it's more *general*:

- **One objective, all data.** Next-token prediction works on any text, so training data is
  effectively unlimited.
- **Input/output distinction disappears.** Translation, summarizing, Q&A and chat are all just
  "continue this text" with the right prefix. No architectural change per task.
- **Simplicity scales.** One stack of identical blocks is easier to make wider, deeper and
  faster than two interacting stacks.
- **In-context learning emerged.** At scale, decoder-only models started solving tasks from
  examples in the prompt, with no weight updates — a capability nobody designed in.

## 11.4 GPT-style architecture, concretely

What GPT-2/3 (and your Topic 8 model) look like:

```
tokens -> token embedding + learned position embedding
       -> N x [ pre-norm attention + residual, pre-norm FFN + residual ]
       -> final layer norm
       -> linear to vocabulary (often weight-tied)
```

Scaling is uniform: GPT-2 small is 12 layers / 768 wide / 12 heads; GPT-3 is 96 layers / 12288
wide / 96 heads. Same diagram. Head dimension stays ~64–128 as width grows, so wider models
get more heads rather than bigger ones.

Two conventions worth knowing because you'll meet them in code:
- **Pre-norm** (normalize before each sublayer) replaced the paper's post-norm because deep
  post-norm stacks need careful warmup to train at all.
- **Learned positions** replaced sinusoidal in GPT-2, and were later replaced again by RoPE
  (Topic 12) because learned positions can't extend past their trained length.

## 11.5 What changed, and what didn't

Between the 2017 paper and a 2025 model, the *diagram* barely moved. What changed:

- normalization: LayerNorm → RMSNorm
- position: sinusoidal/learned → RoPE
- activation: ReLU/GELU → SwiGLU
- attention: multi-head → grouped-query
- sometimes: dense FFN → mixture of experts

All of these are **efficiency substitutions inside the same skeleton** — Topic 12's subject.
The conceptual content of Topic 4 remains the whole story.

---

# Part 2 — Questions to implement

Extend your Topic 8 model in this folder (`variants.py`). PyTorch. Small models, short runs —
you are comparing shapes, not chasing quality.

### Q1. Causal vs bidirectional, measured
**Build:** take your model and remove the causal mask, then train briefly with the same
next-token objective.
**Check:** loss falls dramatically lower than the masked version.
**Explain:** the unmasked model looks far better and is worthless. Explain exactly what it
learned, and why loss is a lying metric here.

### Q2. A masked-language-model objective (encoder-style)
**Build:** on the unmasked model, switch the task: replace 15% of input tokens with a `<mask>`
token and predict only those positions.
**Check:** loss is computed only at masked positions, and now bidirectional attention is
legitimate.
**Explain:** why can this model never generate text, and what is it good for instead?

### Q3. Cross-attention
**Build:** a cross-attention module: queries from the decoder's sequence, keys and values from
a separate encoder output.
**Check:** with decoder length 5 and encoder length 8, attention weights are shape (5, 8) and
each row sums to 1. No causal mask is needed on the encoder side.
**Explain:** why is causal masking irrelevant for the encoder side?

### Q4. A tiny encoder-decoder
**Build:** wire encoder → decoder with cross-attention on a toy task (reverse the input
sequence, or copy it).
**Check:** it learns the toy task.
**Explain:** how would you do the same task with a decoder-only model, and what does that
comparison say about §11.3?

### Q5. Pre-norm vs post-norm
**Build:** both block layouts. Train 12-layer versions of each with **no warmup**.
**Check:** post-norm is less stable or fails; pre-norm trains.
**Explain:** relate what you saw to gradient flow through residual connections.

### Q6. Width vs depth at fixed budget
**Build:** two models with similar parameter counts — one wide and shallow (2 layers, wide),
one narrow and deep (8 layers, narrow). Train both identically.
**Check:** record best validation loss and step time for each.
**Explain:** which won, and which was faster per step? What does that suggest about how real
models are shaped?

### Q7. Head count at fixed width
**Build:** at model dimension 256, train with 1, 4, and 16 heads.
**Check:** parameter counts are nearly identical; losses differ.
**Explain:** why does head count matter if the parameter count doesn't change? Relate to
head dimension.

### Q8. Read a real config
**Build:** find the published config for GPT-2 small and for a Llama model (any size) and
tabulate: layers, width, heads, head dimension, vocabulary, context length, parameters.
**Explain:** which numbers scale together, which stay constant, and where your own model sits.

---

# Done when you can answer

1. What did the transformer remove from RNNs, and why did that matter for training?
2. What are the three architecture families, and what is each good for?
3. What is cross-attention, precisely?
4. Give three reasons decoder-only won.
5. Why pre-norm?
6. What changed between the 2017 paper and a modern model — and what didn't?

Write answers in `notes.md`.
