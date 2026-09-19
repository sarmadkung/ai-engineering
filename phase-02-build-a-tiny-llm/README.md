# Phase 2 — Build a Tiny LLM

**Goal:** train a real language model end to end. Phase 1 built the pieces by hand with random weights;
this phase makes them learn. After this, nothing about LLMs is a black box to you.

**This is where PyTorch enters.** Create a virtual environment in Topic 8 (`python3 -m venv .venv`,
`pip install torch`). Topics 6 and 7 are still standard library.

---

## What I will do

| # | Topic | What I build | Status |
|---|---|---|---|
| 6 | [Tokenizer From Scratch](06-tokenizer-from-scratch/) | `tokenizer.py` — byte-level BPE, saveable, exact round trip | ready |
| 7 | [Dataset Preparation](07-dataset-preparation/) | `dataset.py` — token stream, chunks, input/target pairs, batches | ready |
| 8 | [Tiny Language Model](08-tiny-language-model/) | `model.py` — the transformer in PyTorch, with a loss | ready |
| 9 | [Training](09-training/) | `train.py` — the loop, schedules, checkpoints, loss curves | ready |
| 10 | [Inference](10-inference/) | `generate.py` — generation, KV cache, streaming | ready |

Each folder has a `THEORY.md` (theory + numbered questions with checks). You write the code and `notes.md`.

## Which things to learn

**6. Tokenizer From Scratch** — a tokenizer as a trained, saveable artifact; byte-level BPE so nothing is
unknown; stable token IDs; special tokens; measuring compression and fertility; why the wrong tokenizer
makes a model produce fluent nonsense.

**7. Dataset Preparation** — cleaning; one token stream with document separators; the input/target shift
by one (and how an off-by-one produces a beautiful loss curve and a useless model); chunking and overlap;
batching, padding and masking; splitting without leakage; deduplication; seeded shuffling.

**8. Tiny Language Model** — PyTorch and autograd; tensor shapes as the main source of bugs; embeddings,
causal attention, blocks, output projection, weight tying; cross-entropy; why untrained loss should be
`ln(vocab_size)`; parameter counting; dropout and train/eval modes; "overfit one batch" as the readiness
test.

**9. Training** — the five-step loop; learning rate as the setting that matters most; warmup and cosine
decay; AdamW; reading train-versus-validation curves; gradient clipping and accumulation; checkpoints
with optimizer state; telling a capacity limit from a bug.

**10. Inference** — eval mode and `no_grad`; base models versus instruction-following; the generation
loop; why naive generation is quadratic and what a KV cache changes; proving the cache is correct;
stopping conditions; streaming and TTFT; prefill versus decode, and why output tokens cost more.

## Prerequisites

Phase 1 complete. A machine that can train a small model — a laptop is enough for a few million
parameters; a rented GPU makes Topic 9's experiments faster.

**Next:** Phase 3 — Modern LLMs, which explains what real models add on top of what you just built.
