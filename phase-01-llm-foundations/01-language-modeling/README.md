# Topic 1 — Language Modeling

**What this folder is for:** building your first language model from scratch, so that
"LLM" stops being a black box and becomes a function you have written yourself.

Phase 1, Topic 1 of `AI_ENGINEERING_LLM_ROADMAP.md`.

---

## What I will do here

1. Read the theory in [THEORY.md](THEORY.md), Part 1 — eight short sections, once, end to end.
2. Work through the questions in THEORY.md Part 2 (Q1–Q12, in order; Q13–Q15 optional), building
   one file: **`ngram_lm.py`** (does not exist yet — I create it).
3. Run the four experiments in Group F and write up what the numbers mean.
4. Answer the eight "You are done when" questions in my own words in **`notes.md`**
   (does not exist yet — I create it).

## Files

| File | Who writes it | Status |
|---|---|---|
| `THEORY.md` | provided | ready — plain-language theory + the questions to implement |
| `corpus.txt` | provided | 3.7 KB training text |
| `corpus_alt.txt` | provided | 1.5 KB text in a different voice, for experiment Q12 |
| `ngram_lm.py` | **me** | to create — the model |
| `notes.md` | **me** | to create — my answers and measured results |

No solution file exists anywhere in this repo. That is deliberate.

## What I am building

A character-level **n-gram language model**: it predicts the next character from the
previous few by counting what followed what in a training corpus. Pure Python standard
library — `math`, `random`, `collections`. No PyTorch, no NumPy, nothing to install.

It is a deliberately primitive model, and that is the point. It is a *complete* language
model in the formal sense — it has a probability distribution, a training procedure, an
autoregressive generation loop, decoding controls, and a measurable loss — so every
concept transfers directly to a transformer. It is also weak in a specific, instructive
way, and experiment Q9 makes you discover that weakness with your own numbers. That
discovery is what makes attention (Topic 4) feel necessary rather than arbitrary.

## Pieces I will implement

Roughly in this order, each with a check to pass before moving on:

- a **tokenizer** (text ↔ tokens) and a **vocabulary**
- **`fit`** — the training pass: read the corpus, count context → next-token
- **`distribution`** — context in, a probability for every vocabulary token out
- **`choose`** — the sampler: greedy, temperature, top-k
- **`generate`** — the autoregressive loop
- **`cross_entropy` / `perplexity`** — evaluation on held-out text

## Concepts I should own by the end

Next-token prediction · autoregressive generation · training vs inference · context
windows · vocabulary · distributions over tokens · greedy vs sampling · temperature ·
top-k · cross-entropy loss · perplexity · overfitting vs generalization · why counting
models hit a wall.

## How to work

- Run the check after each question. Don't write three pieces then debug.
- Write the *written* answers down as you go, not at the end — they are most of the value.
- When stuck: ask me for a hint or a review of your code. I will not write the
  implementation, and I will push on vague reasoning instead of handing over answers.
- Expect a few hours. It is a small amount of code and a large amount of thinking.

## Done when

`ngram_lm.py` runs, every check in Groups A–E passes, the four experiments are recorded
with numbers, and `notes.md` answers the eight questions at the end of THEORY.md without
looking back at the theory.

**Next:** Topic 2 — Tokenization, which replaces this topic's character tokenizer with a
BPE tokenizer built from scratch, against the same interface.
