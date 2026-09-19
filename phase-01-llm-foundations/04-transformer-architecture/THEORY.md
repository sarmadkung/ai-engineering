# Topic 4 — Transformer Architecture

**Why this topic:** this is the answer to the wall you hit in Topic 1. Counting needs an
exact context match; attention compares contexts *loosely*, so similar situations help
each other. This is the single most important topic in Phase 1.

---

# Part 1 — Theory

## 4.1 The idea in one paragraph

To predict the next token, some earlier tokens matter a lot and most matter very little.
In *"the dog that chased the cat was ___"*, the important word is **dog** (so: "was", not
"were"), even though "cat" is closer. Attention is a mechanism that, for each position,
**looks at every earlier token and decides how much to care about each one**, then builds a
summary weighted by that decision. It learns what to care about from data.

## 4.2 Query, Key, Value

The mechanism is a soft dictionary lookup. Each token produces three vectors, each by
multiplying its embedding by a learned matrix:

- **Query (Q)** — "what am I looking for?"
- **Key (K)** — "what do I offer?"
- **Value (V)** — "what do I contribute if chosen?"

For one position, the steps are:

```
1. score every token:   score_j = Q_current · K_j        (dot product = how well they match)
2. scale:               score_j / sqrt(d_k)              (keeps numbers in a sane range)
3. mask the future      (see 4.5)
4. softmax the scores   -> attention weights summing to 1
5. output = sum over j of  weight_j * V_j                (weighted average of values)
```

Step 4 should look familiar: it's the same softmax that turns logits into a distribution
in Topic 1 — here it turns match scores into "how much attention".

**Why sqrt(d_k):** dot products of long vectors get large, and large numbers make softmax
almost one-hot (all attention on a single token, tiny gradients). Dividing by the square
root of the key dimension keeps the spread reasonable.

**This is the generalization Topic 1 lacked.** "cat" and "dog" have similar embeddings, so
they produce similar keys, so they attract similar attention. Evidence transfers. No exact
string match required.

## 4.3 Multi-head attention

One attention operation gives one summary — one opinion about what matters. Models need
several simultaneously: grammar agreement, subject tracking, topic, position.

So run attention **several times in parallel** with separate Q/K/V matrices, each on a
smaller slice of the dimensions, then concatenate the results and pass them through one
more learned matrix. Each copy is a **head**. 8 heads of size 64 rather than 1 head of
512: same total work, several independent views.

## 4.4 What goes around attention

Attention mixes information **between positions**. The rest of the block processes each
position **on its own**:

**Feed-forward network (FFN/MLP)** — two linear layers with a nonlinearity between them,
applied identically at every position. It expands to ~4× the width and comes back down.
This is where most of a model's parameters live, and it is where "thinking about" a token
happens after the mixing.

**Residual connections** — every sublayer *adds* its result to its input rather than
replacing it (`x = x + attention(x)`). Two benefits: gradients have a short path back
through many layers, so deep stacks train at all; and each layer only has to learn a
*modification*, not a whole representation.

**Layer normalization** — rescales each position's vector to a stable mean and variance,
so numbers don't drift as they pass through dozens of layers.

One block, therefore:

```
x = x + attention(norm(x))
x = x + ffn(norm(x))
```

Stack that block N times (12, 32, 80...). **Depth is just repetition of this block** — the
architecture doesn't get more complicated as models get bigger.

Finally, an output layer maps the last position's vector to one number per vocabulary
entry — **logits** — and softmax turns them into the distribution from Topic 1.

## 4.5 Causal masking

A model that predicts the next token must not see it. During training the whole passage is
present at once, so position 5 could simply look at position 6 and copy the answer — and
learn nothing.

Fix: before softmax, set the score of every **future** position to negative infinity.
After softmax those weights are 0. Each position sees only itself and the past.

This is what makes the model **causal** (decoder-only). And it's why training can be
parallel: with the mask in place, all positions can be computed simultaneously and each one
is still only predicting from its own past.

## 4.6 Encoder, decoder, both

| Type | Sees | Good for | Example |
|---|---|---|---|
| **Decoder-only** | past only (masked) | generating text | GPT, Llama, Claude |
| **Encoder-only** | whole input, both directions | understanding, classification, embeddings | BERT |
| **Encoder-decoder** | encoder reads all, decoder writes with **cross-attention** to it | translation, summarizing | T5, original Transformer |

Every LLM you will use in later phases is decoder-only. Cross-attention (decoder queries,
encoder keys/values) is the one extra mechanism in the third row.

---

# Part 2 — Questions to implement

Build `attention.py` in this folder. Standard library only — write the matrix helpers
yourself. Small numbers, printed and inspected by hand. **Forward pass only**; training
comes in Phase 2.

### Q1. Matrix helpers
**Build:** matrix–vector multiply, matrix–matrix multiply, transpose, and softmax over a
list. Make softmax numerically safe by subtracting the maximum before exponentiating.
**Check:** softmax output sums to 1 and is unchanged if you add a constant to every input.
Multiplication shapes behave as expected.
**Explain:** why does subtracting the max not change the result, and what does it prevent?

### Q2. One attention head, one position
**Build:** given a small sequence of embeddings (say 4 tokens, dimension 8) and random
Q/K/V matrices, compute the attention output for the **last** position. Print the weights.
**Check:** weights sum to 1; output has the same dimension as a value vector.
**Explain:** in your own words, what did steps 1–5 of §4.2 do to produce that output?

### Q3. Prove attention generalizes
**Build:** hand-set two token embeddings to be **nearly identical** (your "cat" and "dog")
and a third to be very different. Attend from a query that matches "cat".
**Check:** "dog" receives almost as much attention as "cat"; the unrelated token gets very
little.
**Explain:** this is the fix for Topic 1's failure. Say exactly why the counting model
could not do this.

### Q4. The scaling factor
**Build:** compute attention weights with and without dividing by `sqrt(d_k)`, using
dimension 8 and then dimension 512 (random vectors).
**Check:** unscaled weights at dimension 512 collapse to nearly all-on-one-token.
**Explain:** why is a near-one-hot attention pattern a training problem?

### Q5. Causal mask
**Build:** extend Q2 to all positions at once, with future scores set to `-inf` before
softmax. Print the full weight matrix.
**Check:** it is lower-triangular — every entry above the diagonal is 0. Row 0 attends only
to itself.
**Explain:** what would the model learn without this mask, and why does the mask make
parallel training possible?

### Q6. Multi-head
**Build:** run 4 heads of dimension 4 over a model dimension of 16, concatenate, then apply
a final output matrix.
**Check:** output dimension equals model dimension. Different heads produce visibly
different weight rows.
**Explain:** why several small heads instead of one big one?

### Q7. Feed-forward network
**Build:** two linear layers with a nonlinearity between (ReLU is fine), expanding to 4×
and back, applied per position.
**Check:** feeding two different positions gives two independent results — no mixing.
**Explain:** which part of the block mixes information across positions, and which part
does not? Why do you need both?

### Q8. Residuals and normalization
**Build:** layer normalization (subtract mean, divide by standard deviation, per position),
then assemble a full block: `x = x + attn(norm(x))`, `x = x + ffn(norm(x))`.
**Check:** normalized vectors have mean ≈ 0 and standard deviation ≈ 1. Removing the
residual (`x = attn(norm(x))`) changes output magnitudes noticeably as you stack blocks.
**Explain:** stack 10 blocks with residuals and 10 without, and compare the output scale.
What would that do to training?

### Q9. A full forward pass
**Build:** embeddings + positional encoding (reuse Topic 3) → N stacked blocks → output
projection to vocabulary size → softmax. Feed a real tokenized string from Topic 2.
**Check:** you get a valid probability distribution over your vocabulary, summing to 1.
The predictions are nonsense — the weights are random.
**Explain:** every component of a real LLM is now in your file. What single thing is
missing that would make the output meaningful?

### Q10. Cost accounting
**Build:** for sequence length 10, 100, 1000, count how many attention scores must be
computed.
**Explain:** how does the count grow with length? What does that imply about the price of
long context, and which part of the model is responsible?

---

# Done when you can answer

1. What do Q, K, and V each represent?
2. Why divide by sqrt(d_k)?
3. What does causal masking prevent, and what does it enable?
4. Why multiple heads?
5. Which part mixes positions and which part doesn't?
6. Why do residual connections matter in deep stacks?
7. Why does attention generalize where Topic 1's counting could not?
8. Why does long context cost grow faster than linearly?

Write answers in `notes.md`.
