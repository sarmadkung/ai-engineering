# Topic 3 — Embeddings

**Why this topic:** in Topic 1 your model failed because "cat" and "dog" were unrelated
symbols. Embeddings are the fix. This is the idea that makes neural language models work.

---

# Part 1 — Theory

## 3.1 The problem being solved

A token ID is a label, not information. ID 1820 ("the") and ID 1821 ("dog") are as
different as any other pair of numbers — and nothing about the number 1820 tells you what
"the" means. Counting models suffer exactly this: seeing `the cat sat` teaches them
nothing about `the dog sat`.

We want a representation where **similar tokens are close together**, so that learning
about one automatically informs the other.

## 3.2 Token embeddings

Give every token in the vocabulary a **list of numbers** (a vector):

```
"cat"  ->  [ 0.21, -0.88,  0.42, ... ]     (say 256 numbers)
"dog"  ->  [ 0.19, -0.81,  0.47, ... ]     close to "cat"
"the"  ->  [-0.55,  0.12, -0.90, ... ]     far from both
```

That table — vocabulary size × embedding dimension — is called the **embedding matrix**,
and it is usually the first layer of a language model. Looking up a token is just picking
a row.

The crucial part: **nobody writes these numbers by hand.** They start random and are
learned during training, because better vectors mean better next-token prediction. Words
used in similar contexts drift together, since that reduces the loss.

The number of dimensions is a design choice (a few hundred to a few thousand). More
dimensions = more capacity to express distinctions, and more parameters to train.

## 3.3 What lives in the space

Directions in this space end up meaning something. The classic demonstration:

```
king - man + woman  ≈  queen
```

Meaning is captured as relative position, not as a definition. Nothing in the model
"knows" what a queen is; it knows where the token sits relative to others. This is worth
holding onto when you later reason about what an LLM does and does not understand.

## 3.4 Two different things called "embeddings"

Don't confuse them:

- **Token embeddings** (this topic) — one vector per token, inside the model, learned as
  part of it. A building block.
- **Text embeddings** (Phase 5, Topic 21) — one vector for a whole sentence or document,
  produced by a separate model, used for search and RAG. A product feature.

Same underlying idea — meaning as position in space — used for two different jobs.

## 3.5 Position has to be added deliberately

Here is something surprising: the attention mechanism you build in Topic 4 treats its
input as a **set**, not a sequence. By itself it cannot tell `dog bites man` from
`man bites dog`. Order is invisible to it.

So position must be injected into the vectors themselves. Two families:

**Learned positional embeddings** — a second table, one vector per position (position 0,
position 1, ...), added to the token vector:

```
input at position 5  =  embedding("dog")  +  embedding(position 5)
```

Simple, and used by GPT-2-style models. Limitation: positions beyond the trained maximum
have no vector, so the context length is baked in.

**Positional encoding (fixed, sinusoidal)** — compute the position vector from sine and
cosine waves of different frequencies instead of learning it. No parameters, and it
extends to positions never seen in training. This is what the original Transformer paper
used.

(Modern models mostly use **RoPE**, which rotates vectors by an angle based on position so
that only *relative* distance matters. That's Topic 12 — the idea builds directly on this.)

## 3.6 Comparing vectors: similarity

To ask "are these two vectors similar?", the standard answer is **cosine similarity** —
the cosine of the angle between them:

```
 1.0  = same direction  (very similar)
 0.0  = perpendicular   (unrelated)
-1.0  = opposite
```

It cares about **direction, not length**, which is what you want: a long document vector
and a short query vector can still point the same way. Computed as the dot product of the
two vectors divided by both their lengths.

Alternatives: dot product (fast, but length affects the score) and Euclidean distance
(straight-line gap; on normalized vectors it ranks the same as cosine). All of Phase 5's
semantic search runs on this one operation.

---

# Part 2 — Questions to implement

Build `embeddings.py` in this folder. Standard library only (`math`, `random`). No NumPy
— doing the loops by hand once is the point.

### Q1. Vector basics
**Build:** functions for dot product, vector length (magnitude), addition, subtraction,
scaling, and cosine similarity. Take plain lists of floats.
**Check:** cosine of a vector with itself is 1.0. Cosine of `[1,0]` and `[0,1]` is 0.
Cosine of `[1,0]` and `[-1,0]` is -1. Cosine of `[1,0]` and `[5,0]` is 1 (length ignored).
**Explain:** why does cosine ignore length, and why is that the behaviour you want?

### Q2. An embedding table
**Build:** for the vocabulary from Topic 2, create a table of random vectors (dimension
16 to start). Write `lookup(token)` returning its vector.
**Check:** the table has vocab_size rows, each of the same length. Random vectors have
near-zero cosine similarity to each other.
**Explain:** these vectors are random and therefore meaningless. What process is supposed
to make them meaningful, and why would prediction error drive them to move?

### Q3. Similarity by hand, with real meaning
**Build:** you cannot train yet, so hand-build a tiny 2-D space: place ~8 words yourself
(cat, dog, kitten, puppy, car, truck, the, of) at coordinates you choose to reflect
meaning. Then write `nearest(word, k)` using cosine similarity.
**Check:** cat's nearest neighbours are kitten and dog, not car.
**Explain:** what did you have to know to place those points — and where would that
knowledge come from in a real model?

### Q4. Vector arithmetic
**Build:** in your hand-made space, compute `kitten - cat + dog` and find the nearest
word to the result.
**Explain:** did you get puppy? What does the answer say about how meaning is stored?

### Q5. Learned positional embeddings
**Build:** a second table, one random vector per position, up to a maximum length. Write a
function that takes a token list and returns, for each position, `token_vec + pos_vec`.
**Check:** the same token at two positions produces two different vectors. A sequence
longer than your maximum raises a clear error rather than silently misbehaving.
**Explain:** why is that error the honest behaviour, and what real-world limit does it
correspond to?

### Q6. Sinusoidal positional encoding
**Build:** the fixed version: for position `p` and dimension index `i`, alternate sine and
cosine of `p / (10000 ** (2i/d))`. No parameters.
**Check:** it works for any position, including ones beyond your earlier maximum. Print
cosine similarity between position vectors for positions (0,1), (0,5), (0,50) — nearby
positions should be more similar than distant ones.
**Explain:** what does your similarity table show about what this encoding tells the model,
and what advantage does it have over the learned table?

### Q7. Prove order is invisible without position
**Build:** compute the **sum** of token vectors for `dog bites man` and for
`man bites dog`, with no positional information added. Compare.
**Check:** identical.
**Explain:** this is the exact reason positional information exists. State the reason in
one sentence.

### Q8. Dimension sweep
**Build:** with random vectors in dimensions 2, 16, 256, measure the average cosine
similarity between random pairs.
**Explain:** it shrinks toward 0 as dimensions grow. Why is "random things are far apart in
high dimensions" useful for a model that needs many distinct meanings?

---

# Done when you can answer

1. Why is a token ID not enough, and what does an embedding add?
2. Where do embedding values come from?
3. What does cosine similarity measure, and why not use length?
4. Why must position be added separately, and what breaks if you don't?
5. Learned vs sinusoidal positions — what does each buy you?
6. What's the difference between token embeddings and text embeddings?

Write answers in `notes.md`.
