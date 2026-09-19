# Phase 1 — LLM Foundations

**Goal of this phase:** understand what a large language model actually *is* — as a
mechanism, not an analogy — and be able to build the pieces yourself. Everything in
Phases 2 through 17 assumes these five topics are solid.

You write all the code in this phase. Each topic folder provides theory, implementation
questions, and sample data; the implementation and notes are yours.

---

## What I will do in this phase

Five topics, in order. Each is one folder with a `README.md` (what to do), a
`THEORY.md` (theory + questions to implement), and any sample data.

| # | Topic | What I build | Status |
|---|---|---|---|
| 1 | [Language Modeling](01-language-modeling/) | `ngram_lm.py` — a complete character-level n-gram language model: training, generation, decoding, perplexity | **ready to start** |
| 2 | Tokenization | a BPE tokenizer from scratch, replacing Topic 1's character tokenizer behind the same interface | not created yet |
| 3 | Embeddings | token + positional embeddings; vector similarity by hand | not created yet |
| 4 | Transformer Architecture | self-attention and a transformer block, from Q/K/V up | not created yet |
| 5 | LLM Generation | logits → softmax → sampling: the full decoding stack, properly | not created yet |

Topic folders are created one at a time, when you reach them — so that each one can
build on what you actually wrote in the previous one rather than on a guess.

## Which things to learn, by topic

**1. Language Modeling** — what a language model is (context in, distribution over next
tokens out); next-token prediction and why so narrow an objective yields broad
capability; autoregressive generation and its two consequences (sequential latency,
compounding errors); training vs inference as distinct activities; context as the model's
only memory; decoding strategies as your choice, not the model's; cross-entropy and
perplexity; and why counting models hit a wall that attention was invented to solve.

**2. Tokenization** — why characters are too small and words too many; subword
tokenization; the BPE merge algorithm; SentencePiece; token IDs and vocabulary
construction; why token count, not character count, is the unit of cost and context in
every LLM product.

**3. Embeddings** — tokens as vectors rather than symbols; why that is what lets similar
contexts share evidence (the exact failure you will have measured in Topic 1);
positional embeddings and encodings, and why order has to be injected deliberately;
similarity and distance between vectors.

**4. Transformer Architecture** — self-attention as learned, soft context matching;
query, key and value; multi-head attention; feed-forward layers; residual connections
and layer normalization; causal masking and why a decoder must not see the future;
encoder vs decoder vs encoder-decoder.

**5. LLM Generation** — logits and softmax as the real output of a model; temperature,
greedy decoding, sampling, top-k, top-p and repetition controls, implemented properly
against logits rather than probabilities. This closes the loop back to Topic 1's
sampler with the real machinery underneath.

## How to work through it

Follow the roadmap's Learning Method for each topic: understand the concept → learn the
mechanism → implement a minimal version from scratch → (later phases) use the production
library → build something small → test and evaluate → write down what you learned → move
on. The "write down what you learned" step is not optional; it is where the topic
actually lands.

Ask for hints, code review, or to be quizzed on a topic's questions at any point.
Nobody writes the implementation but you.

## Prerequisites

Python 3 and a text editor. Nothing to install for Topics 1–3; Topic 4 is where a
virtual environment with PyTorch starts to earn its place, and Phase 2 requires it.
