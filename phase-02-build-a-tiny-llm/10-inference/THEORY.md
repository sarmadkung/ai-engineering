# Topic 10 — Inference

**Why this topic:** you have a trained model. Now make it talk. This is the same generation
loop as Topic 1, but against a neural network — and the point where you first see text your
own model learned to produce.

---

# Part 1 — Theory

## 10.1 Inference mode

Two things must be switched before generating, and both are easy to forget:

- `model.eval()` — disables dropout. Left in train mode, your output is randomly damaged by
  dropped activations.
- `torch.no_grad()` — stops recording operations for autograd. Without it you waste memory
  and time building a graph you'll never use, and long generations can exhaust memory.

Both are pure inference hygiene. Neither changes the weights; forgetting either degrades
output or performance for no reason.

## 10.2 Prompting a base model

Your model is a **base model**: it continues text. It has never been taught to answer
questions or follow instructions. So:

```
prompt: "The capital of France is"        -> works, it continues plausibly
prompt: "What is the capital of France?"  -> may continue with more questions
```

A base model given a question may well produce a list of similar questions, because that's
what follows a question in its training data. Instruction-following is *added later* by
fine-tuning (Phase 3, Topic 14). Understanding this distinction now explains why raw
pretrained models on Hugging Face feel broken compared to chat models.

For a base model, the effective technique is to write a prompt whose natural continuation is
what you want — including few-shot examples (Phase 4, Topic 17 formalizes this).

## 10.3 The generation loop

```
1. tokenize the prompt              -> ids
2. crop ids to the last block_size  (the model cannot see further)
3. forward pass                     -> logits (B, T, V)
4. take logits at the LAST position -> (B, V)
5. apply temperature / top-k / top-p
6. softmax, sample one token
7. append to ids, go to 2
8. stop on <eos>, max_tokens, or a stop string
9. decode
```

Two mistakes to avoid: using logits from all positions instead of only the last (you get
predictions for tokens you already have), and forgetting to crop, which crashes once the
sequence exceeds the trained position embeddings.

## 10.4 Why naive generation is slow, and what a KV cache is

Step 3 re-runs the whole sequence every time. To generate token 500 you recompute the
representations of tokens 1–499, which haven't changed. Cost grows quadratically with output
length for no reason.

The fix — **KV caching** — stores each position's computed keys and values, so each new step
only computes K and V for the one new token and attends against the stored rest. Generation
becomes roughly linear in length instead of quadratic, typically a 10×+ speedup.

Details live in Phase 3 (Topic 12), but the idea belongs here because you feel the pain here.
The memory cost is real: cache size grows with sequence length × layers × heads, and for long
contexts it can exceed the model weights themselves. That is why serving systems (Phase 11)
care so much about it.

## 10.5 Sampling settings, now on a real model

Everything from Topic 5 applies, and now you can judge it on output that actually means
something:

- Low temperature / greedy → repetitive but coherent. Small models loop badly.
- High temperature → drifts off topic and into non-words.
- top-p ~0.9 with temperature ~0.8 → a reasonable default to compare against.

Expect your tiny model to produce *shapes* of English — plausible word fragments, correct
spacing, sentence-like rhythm — without real meaning. That is the correct outcome for this
scale, and recognizing it prevents you from chasing a bug that isn't there.

## 10.6 Streaming

Users perceive speed by **time to first token**, not total time. Since generation is
one-token-at-a-time anyway, you can yield each token as it's produced rather than waiting for
the whole response. Same total duration, dramatically better experience.

One catch from Topic 6: a token may be a partial UTF-8 character, so streaming must buffer
incomplete byte sequences rather than decoding blindly per token. This is exactly why you
sometimes see a broken glyph flicker in a streaming UI.

## 10.7 Measuring inference

Report these, because every serving decision in later phases refers to them:

- **latency** — total time to finish a response
- **time to first token (TTFT)** — how responsive it feels
- **tokens per second** — throughput
- **prefill vs decode** — prefill processes the whole prompt in parallel (fast per token);
  decode produces one token at a time (slow per token). Long prompts are cheap; long
  *outputs* are expensive. This asymmetry drives API pricing, where input tokens cost less
  than output tokens.

---

# Part 2 — Questions to implement

Build `generate.py` in this folder, loading your Topic 9 checkpoint.

### Q1. Load and verify
**Build:** load the checkpoint, rebuild the model from the saved config, restore weights, set
`eval()`.
**Check:** re-compute validation loss and confirm it matches what training reported.
**Explain:** if it doesn't match, name three plausible causes.

### Q2. The generation loop
**Build:** the loop from §10.3, using only the last position's logits, with cropping.
**Check:** it produces text. Feed a prompt longer than block_size and confirm no crash.
**Explain:** print the shape of the logits and say which slice you used and why.

### Q3. Look at the output honestly
**Build:** generate 5 samples of 200 tokens from the same prompt.
**Explain:** describe what the model got right (spacing? word shapes? sentence rhythm?) and
what it got wrong. Is this the expected quality for your parameter count and corpus size?
Justify your answer.

### Q4. Base model vs instruction
**Build:** generate from `"The capital of France is"` and from `"What is the capital of
France?"`.
**Explain:** compare the behaviour. Explain the difference using §10.2 — what would have to
happen for the question form to work properly?

### Q5. Sampling settings on real output
**Build:** generate at greedy, T=0.5, T=0.8+top_p=0.9, T=1.2, top_k=10 — same prompt, same
seed.
**Explain:** rank them. Which setting hides the model's weaknesses, and which exposes them?

### Q6. Measure the naive cost
**Build:** time generation of 50, 100, 200, 400 tokens, and record tokens per second for each.
**Check:** tokens per second *falls* as output grows.
**Explain:** why does it fall? Which step of the loop is responsible?

### Q7. Implement a KV cache
**Build:** cache keys and values per layer; on each step feed only the new token and attend
against the cache.
**Check:** with a fixed seed, cached and uncached generation produce **identical** text —
this is the correctness test and it must pass. Then re-run Q6's timing.
**Explain:** report your speedup at 400 tokens. Why did tokens per second stop degrading?
Where does the memory go?

### Q8. Stopping
**Build:** stop on `<eos>`, on `max_tokens`, and on a user-supplied stop string.
**Check:** all three fire. The stop string is removed from the returned text.
**Explain:** how should an application behave differently when it stopped on max_tokens
versus `<eos>`?

### Q9. Streaming
**Build:** make generation a generator that yields text as it goes, buffering incomplete
UTF-8 byte sequences.
**Check:** printing the stream produces the same final text as the batch version. Feed it text
with an emoji and confirm no broken output.
**Explain:** why is TTFT the number users actually feel?

### Q10. Prefill vs decode
**Build:** measure, separately, the time to process a 500-token prompt and the time to
generate 500 tokens.
**Explain:** compare them. Now explain why APIs charge less for input tokens than output
tokens.

### Q11. Phase 2 wrap-up
**Build:** a short `README.md` in this folder recording: your model config, parameter count,
corpus size, best validation loss and perplexity, tokens/sec with and without cache, and 3
sample generations.
**Explain:** name the single change you would make first to improve quality, and why you
believe it would be the biggest win.

---

# Done when you can answer

1. What must you switch on before inference, and what happens if you don't?
2. Why does a base model handle a question poorly?
3. Which logits do you sample from, and why only those?
4. Why is naive generation quadratic, and what does a KV cache change?
5. How do you *prove* a KV cache is correct?
6. What is TTFT, and why does streaming matter if total time is unchanged?
7. Why are output tokens more expensive than input tokens?

Write answers in `notes.md`.

---

**Phase 2 is complete when this works.** You will have built and trained a working language
model end to end: tokenizer, dataset, model, training loop, inference. Phase 3 explains what
real LLMs add on top.
