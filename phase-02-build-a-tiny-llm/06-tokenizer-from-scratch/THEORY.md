# Topic 6 — Tokenizer From Scratch

**Why this topic:** Phase 2 builds a real, trainable language model. Step one is a
production-shaped tokenizer: a saveable, loadable artifact with a fixed vocabulary that
your training and inference code both depend on.

You built BPE in Topic 2. Here you make it *solid*.

---

# Part 1 — Theory

## 6.1 A tokenizer is a trained artifact, not a function

This trips people up. A tokenizer has **state** — its vocabulary and merge list — learned
from a corpus. That state must be saved to disk and shipped alongside the model.

If the tokenizer changes, every token ID changes meaning, and the model becomes garbage
without its weights changing at all. Token ID 4219 means "ing" only because *this*
tokenizer says so. This is why models and tokenizers are versioned together, and why
loading a model with the wrong tokenizer produces fluent nonsense.

## 6.2 The four operations

Everything downstream needs exactly these:

```
train(corpus, vocab_size)   -> build vocabulary + merges          (once, offline)
encode(text)                -> list of token IDs                  (every request)
decode(ids)                 -> text                               (every response)
save(path) / load(path)     -> persist and restore the artifact
```

`encode` and `decode` must be **exact inverses** for any input. A tokenizer that mangles
whitespace or drops an emoji introduces errors nothing downstream can fix.

## 6.3 Vocabulary and ID assignment

The vocabulary is the two-way map: token text ↔ integer ID. Two rules that save pain later:

- **IDs must be stable.** Assign them once, in a deterministic order, and never re-shuffle.
  A dict iteration order or a set is not a stable source of IDs.
- **Reserve special token IDs first**, before any learned token, so they can never collide:

| Token | Purpose |
|---|---|
| `<pad>` | fills short sequences in a batch (Topic 7) |
| `<eos>` | end of text — how the model says "I'm done" (Topic 10) |
| `<unk>` | unknown; with byte-level BPE you should never need it |
| `<bos>` | beginning of text, optional |

## 6.4 Byte-level, so nothing can break it

Your Topic 2 version started from characters, so a character never seen in training had no
representation. Real tokenizers start from **bytes** — all 256 possible values.

Since any text encodes to bytes (UTF-8), any input can be tokenized: emoji, Arabic, corrupt
data, binary. No unknown token is ever needed. The cost is that non-English text uses more
tokens per character, because its bytes are less common and merge less.

One wrinkle: a single UTF-8 character can be several bytes, so a token may hold **part** of
a character. Decoding must therefore collect all the bytes first and convert to text at the
very end — never per-token. (This is also why streaming APIs sometimes deliver a broken
character mid-stream: the remaining bytes haven't arrived yet.)

## 6.5 Encoding efficiently

Naively, each merge scans the whole text — fine for a toy, slow for a real corpus. Two
practical improvements:

- Split text into small chunks first (on whitespace/punctuation boundaries) and tokenize
  each independently, since merges never cross those boundaries anyway.
- Cache chunk → IDs in a dictionary. Real text repeats words constantly, and the cache
  turns most of the work into a lookup.

## 6.6 Measuring a tokenizer

Two numbers tell you whether it's good:

- **Compression ratio** — bytes per token (higher is better; ~4 is typical for English).
  Directly proportional to cost and to how much text fits in context.
- **Fertility** — average tokens per word. Near 1.0 for common English; much higher for
  other languages or code, which is a real fairness and cost issue.

Measure both on text the tokenizer was *not* trained on.

---

# Part 2 — Questions to implement

Build `tokenizer.py` in this folder — the version you will use for the rest of Phase 2.
Standard library only. Get a larger corpus this time (a public-domain book, your own notes,
a few MB of anything).

### Q1. Bytes, not characters
**Build:** convert text to a list of byte values and back, using UTF-8.
**Check:** round-trips exactly for English, an emoji, and non-Latin script. A multi-byte
character produces more than one byte value.
**Explain:** why must decoding assemble all bytes before converting to text?

### Q2. Training, with progress you can watch
**Build:** BPE training over bytes to a target vocabulary size. Print every 100th merge.
**Check:** first merges are common byte pairs; later merges are whole words. Final
vocabulary size = 256 + number of merges + number of special tokens.
**Explain:** why does the vocabulary size formula look like that?

### Q3. Special tokens and stable IDs
**Build:** reserve IDs 0–3 for `<pad>`, `<eos>`, `<unk>`, `<bos>`, then bytes, then merges,
in deterministic order.
**Check:** train twice on the same corpus and get identical ID assignments. Special IDs
never appear from encoding ordinary text.
**Explain:** what breaks downstream if IDs shift between two training runs?

### Q4. Encode and decode
**Build:** `encode(text) -> ids` and `decode(ids) -> text`.
**Check:** `decode(encode(x)) == x` for: prose, code with indentation, double spaces,
newlines, emoji, and a non-English sentence. Put these in a test function you can re-run.
**Explain:** which case was most likely to break, and why?

### Q5. Save and load
**Build:** `save(path)` and `load(path)` (JSON is fine; bytes need care in JSON — decide how).
**Check:** encode a string, save, load in a fresh process, encode again — identical IDs.
**Explain:** name a bug that could only appear after a save/load round trip.

### Q6. Speed
**Build:** chunk-splitting plus a chunk→IDs cache. Time encoding 1 MB before and after.
**Check:** output is identical with and without the cache; the cache is meaningfully faster.
**Explain:** why is caching so effective on natural text? When would it not help?

### Q7. Measure it
**Build:** on held-out text, report bytes per token and tokens per word for vocabulary sizes
1000, 5000, 10000.
**Explain:** where do the gains flatten? What is the cost of a larger vocabulary — both in
the model and at training time?

### Q8. Fertility across languages
**Build:** encode the same sentence in English and in one other language (or code).
**Explain:** compare token counts. What does the difference mean for cost and for the
context window of a user writing in that language?

### Q9. Ready for Phase 2
**Build:** a clean interface: `train`, `encode`, `decode`, `save`, `load`, `vocab_size`, and
the special-token IDs as attributes.
**Check:** `import tokenizer` from a different folder and encode a string in three lines.
**Explain:** Topics 7–10 will all import this. Which detail, if you change it later, forces
you to retrain the model?

---

# Done when you can answer

1. Why is a tokenizer a trained artifact rather than a function?
2. What exactly gets saved to disk?
3. Why byte-level, and what does it guarantee?
4. Why must a token sometimes hold part of a character, and what must decoding do about it?
5. What are bytes-per-token and fertility, and why do they matter commercially?
6. What happens if a model is loaded with the wrong tokenizer?

Write answers in `notes.md`.
