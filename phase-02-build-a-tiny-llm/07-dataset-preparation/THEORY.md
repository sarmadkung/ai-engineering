# Topic 7 — Dataset Preparation

**Why this topic:** a model trains on batches of fixed-length integer arrays, not on text.
This topic is the conversion, and it is where a lot of real training bugs live.

---

# Part 1 — Theory

## 7.1 From corpus to training examples

The pipeline:

```
raw text files
 -> cleaned text
 -> one long stream of token IDs
 -> fixed-length chunks
 -> (input, target) pairs
 -> batches of those pairs
```

Nothing here is glamorous, and all of it determines whether training works.

## 7.2 Input/target pairs: the shift by one

The model must predict the next token at **every position at once**. So a chunk of tokens
becomes two arrays offset by one:

```
chunk:   [ 12, 45, 8, 91, 3, 67 ]

input:   [ 12, 45, 8, 91, 3 ]
target:  [ 45, 8, 91, 3, 67 ]
```

Read it position by position: given 12 predict 45; given 12,45 predict 8; and so on. One
chunk of length N yields N-1 training signals, not one — which is why language model
training is so data-efficient per pass.

**An off-by-one here is the classic disaster.** Shift the wrong way and the model learns to
copy its input: loss drops beautifully, output is worthless. Always print one pair and read
it aloud before training.

## 7.3 Context windows and chunking

Pick a context length (block size). Then decide how to cut the stream:

- **Non-overlapping chunks** — cheap, but each token is seen in one context only, and
  tokens at a chunk boundary never learn about what preceded them.
- **Overlapping (sliding window with a stride)** — more examples, better coverage of
  boundaries, more compute. Stride = block size means no overlap; stride = block size / 2
  means 50% overlap.

Note what *doesn't* happen: the stream is not cut at sentence or document boundaries by
default. A chunk can span two unrelated documents, which teaches a false continuation. The
standard fix is inserting an `<eos>` token between documents so the model can learn "that
was the end" instead.

## 7.4 Batching

Training processes several chunks simultaneously, as a batch, for two reasons: hardware is
far more efficient on batched work, and averaging the gradient over several examples makes
the training signal less noisy.

A batch is a rectangle: `batch_size × block_size`. All rows must be the same length —
hence fixed-size chunks.

**Padding** appears when rows can't be equal (fine-tuning on variable-length examples,
Phase 3). Short rows get filled with `<pad>`, and you must then supply a **mask** so the
loss ignores padded positions. Training on padding teaches the model to predict padding.

## 7.5 Splits

Split before you look at the data, and never train on the test portion:

- **train** (~90%) — what the model learns from
- **validation** (~5–10%) — checked during training, to see overfitting as it happens
- **test** — touched once, at the end

Split by **document**, not by random chunk. Random chunk splitting lets nearly-identical
neighbouring chunks land in both train and validation, so validation loss looks great and
means nothing. This is **leakage**, and it is the most common way people fool themselves.

## 7.6 Cleaning, and why data quality dominates

Real corpora contain duplicates, boilerplate, navigation menus, encoding damage, and
machine-generated junk. Basic steps: normalize whitespace and Unicode, drop near-duplicate
documents, remove very short and very long documents, and strip obvious boilerplate.

**Deduplication matters more than most beginners expect.** Duplicated text is memorized
rather than learned, wastes compute, and inflates your evaluation if a duplicate straddles
the train/test split. You will feel this in your own results in Question 8.

## 7.7 Shuffling and reproducibility

Shuffle the *order of chunks* so the model doesn't see all of one topic at once — ordered
data makes the model lurch as the topic changes. Never shuffle tokens inside a chunk;
that destroys the language.

Seed every random choice (shuffling, batch order, splits). Training runs you cannot
reproduce cannot be debugged or compared.

---

# Part 2 — Questions to implement

Build `dataset.py` in this folder, importing your Topic 6 tokenizer. Standard library only.

### Q1. Load and clean
**Build:** read your corpus from one or more files. Normalize whitespace, strip control
characters, drop empty documents. Report characters before and after.
**Explain:** name one cleaning step you chose *not* to apply, and the risk you accepted.

### Q2. One long stream of IDs
**Build:** tokenize all documents and concatenate into one list of IDs, with `<eos>`
inserted between documents.
**Check:** total length is plausible against your corpus size and bytes-per-token from Topic
6. Count the `<eos>` tokens — it should equal your document count.
**Explain:** what does the model learn from those `<eos>` tokens, and what goes wrong
without them?

### Q3. Input/target pairs — verify by reading
**Build:** given a block size, produce `(input, target)` pairs shifted by one.
**Check:** take one pair, decode both sides, and print them aligned. Read it aloud: target
must be input moved left by exactly one.
**Explain:** describe the model's behaviour if the shift were reversed, and why the loss
curve would look *good* while the model is useless.

### Q4. Chunking strategies
**Build:** chunk the stream both non-overlapping and with a stride you choose.
**Check:** count examples produced by each for the same corpus and block size.
**Explain:** what does overlap buy, and what does it cost?

### Q5. Batching
**Build:** a batch generator yielding `batch_size × block_size` input and target arrays.
**Check:** every row has exactly block_size entries; the generator covers the dataset once
per epoch without repeating or dropping examples (except a documented final partial batch).
**Explain:** why must all rows be the same length?

### Q6. Padding and masking
**Build:** support variable-length examples: pad to the longest in the batch with `<pad>`,
and return a mask marking real positions.
**Check:** the mask has exactly as many true values as real tokens.
**Explain:** what does the model learn if you train on padded positions without a mask?

### Q7. Splits without leakage
**Build:** split **by document** into train/validation/test.
**Check:** no document appears in two splits. Report token counts per split.
**Explain:** construct the leakage scenario explicitly — describe how random chunk
splitting would make validation loss lie to you.

### Q8. Duplicates
**Build:** count exact duplicate documents (hash them) and report what fraction of tokens
they represent. Remove them and re-measure.
**Explain:** what would those duplicates have done to training, and to your validation
number?

### Q9. Shuffle and seed
**Build:** shuffle chunk order with a seeded random source. Support a `seed` parameter.
**Check:** the same seed gives identical batch order; a different seed differs. Tokens
inside a chunk are untouched.
**Explain:** why shuffle chunks but never tokens?

### Q10. Inspect before you trust
**Build:** a `describe()` function printing: token counts per split, number of batches,
block size, batches per epoch, and a decoded sample batch row.
**Check:** the decoded row reads like real text from your corpus.
**Explain:** Phase 2 training starts next. Which single thing in this file, if wrong, would
silently ruin it?

---

# Done when you can answer

1. Why are inputs and targets offset by one, and how would you catch an off-by-one?
2. How many training signals does one chunk of length N give?
3. What does `<eos>` between documents teach the model?
4. Why must batches be rectangular, and what are padding and masking for?
5. What is leakage, and why does splitting by document prevent it?
6. Why does deduplication matter so much?
7. Why shuffle chunk order but not tokens?

Write answers in `notes.md`.
