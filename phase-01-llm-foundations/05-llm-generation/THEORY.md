# Topic 5 — LLM Generation

**Why this topic:** Topic 1 sampled from probabilities. Real models emit **logits**, and
the full decoding stack lives between logits and the token you see. This closes Phase 1:
you will have built every stage from text to text.

---

# Part 1 — Theory

## 5.1 Logits

The last layer of a transformer produces one raw number per vocabulary entry. Those are
**logits**. They are unbounded — positive, negative, any size — and they are not
probabilities.

```
logits: [ 2.1, -0.5, 8.7, 0.3, -3.2, ... ]     one per vocabulary entry
```

Bigger = the model prefers that token. Only the *differences* between logits matter, not
their absolute values.

## 5.2 Softmax

Softmax converts logits to a probability distribution:

```
p_i = exp(logit_i) / sum over all j of exp(logit_j)
```

Three properties worth understanding rather than memorizing:

- **Everything becomes positive** (exp of any number is positive) and **sums to 1**.
- **It exaggerates gaps.** Because exp grows fast, a logit lead of 2 becomes a large
  probability lead. A modest preference turns into a confident answer.
- **Shifting all logits by a constant changes nothing.** This is also the trick for
  computing it safely: subtract the maximum logit first, so `exp` never overflows.

## 5.3 Temperature, done properly

Divide the logits by T **before** softmax:

```
p = softmax(logits / T)
```

- `T < 1` → logits spread further apart → distribution sharpens → safe, repetitive.
- `T = 1` → the model's own distribution.
- `T > 1` → logits squeezed together → distribution flattens → creative, then incoherent.
- `T = 0` → undefined mathematically (division by zero); by convention it means greedy.

In Topic 1 you approximated this by raising probabilities to `1/T`. Same effect, one step
later. Doing it on logits is the real thing and is cheaper.

## 5.4 Greedy decoding

Always take the highest logit. Deterministic and reproducible, and the right choice when
there is one correct answer (classification, extraction, structured output).

Its failure is structural: identical context must produce an identical next token, so once
the model re-enters a state it has been in, it repeats forever. You saw this in Topic 1.

## 5.5 Sampling, top-k, top-p

**Sampling** — draw a token in proportion to its probability. Variety, and the possibility
of a bad draw.

**top-k** — keep the k highest-probability tokens, renormalize over them, sample. Simple,
but k is fixed: when the model is genuinely certain, k=50 still admits 49 bad options; when
it is genuinely uncertain, k=50 may cut off good ones.

**top-p (nucleus)** — sort by probability, keep adding tokens until their cumulative
probability reaches p (e.g. 0.9), discard the rest, renormalize, sample. This **adapts**:
a confident step keeps 1–2 tokens, an open-ended step keeps hundreds. This is why top-p is
the common default.

**min-p** — keep tokens whose probability is at least some fraction of the top token's.
Another adaptive variant you'll see in local-model tooling (Phase 11).

Order of operations matters, and the standard order is: temperature → top-k → top-p →
sample.

## 5.6 Repetition controls

Models fall into loops even when sampling. Three patches, all applied to logits before
softmax:

- **Repetition penalty** — divide (or subtract from) the logit of any token already in the
  context. Blunt: it also penalizes words that *should* repeat, like "the".
- **Frequency penalty** — subtract an amount proportional to how many times the token
  already appeared. Scales with the actual problem.
- **Presence penalty** — subtract a flat amount if the token appeared at all, pushing the
  model toward new vocabulary.
- **No-repeat n-gram blocking** — forbid any token that would complete an n-gram already
  present. Hard guarantee, but it can block legitimate repeats (names, code).

These are patches for a symptom, not cures. The underlying cause is that a high-probability
loop is genuinely high-probability under the model.

## 5.7 Stopping

Generation has to end somehow:

- **EOS token** — the model emits a special end-of-text token it was trained to produce.
  The natural stop.
- **max_tokens** — a hard cap you set. Hitting it means truncation mid-sentence, which
  looks like a model failure but is a settings failure.
- **stop sequences** — you supply strings ("\nUser:"); generation halts when one appears.
  Essential when you are formatting the prompt into a transcript yourself.

## 5.8 The full pipeline

Every LLM call, in order:

```
text
 -> tokenizer            (Topic 2)
 -> token IDs
 -> embeddings + position (Topic 3)
 -> N transformer blocks  (Topic 4)
 -> logits               (this topic)
 -> temperature / top-k / top-p / penalties
 -> softmax -> sample    -> one token
 -> append, repeat       (Topic 1's loop)
 -> detokenize
text
```

When you meet `temperature` and `top_p` as API parameters in Phase 4, you will know exactly
which line of this pipeline each one touches — and that they are settings you own, not
properties of the model.

---

# Part 2 — Questions to implement

Build `decoding.py` in this folder. Standard library only. You can drive it with logits from
Topic 4's forward pass, or with hand-written logit lists — hand-written is easier to reason
about, so start there.

### Q1. Safe softmax
**Build:** `softmax(logits)` with the max-subtraction trick.
**Check:** sums to 1; unchanged by adding 100 to every logit; does not overflow on
`[1000, 1001]`; without the trick, confirm that the same input does overflow.
**Explain:** why is subtracting the max safe, and what exactly breaks without it?

### Q2. Logits are not probabilities
**Build:** print logits and their softmax side by side for `[2.0, 1.0, 0.0]` and for
`[20.0, 10.0, 0.0]`.
**Explain:** the logit gaps are 10× larger in the second case. What happened to the
probabilities, and what does that tell you about how softmax treats gaps?

### Q3. Temperature on logits
**Build:** `apply_temperature(logits, T)` dividing before softmax. Handle `T = 0` as greedy
explicitly rather than crashing.
**Check:** tabulate the resulting distribution at T = 0.1, 0.5, 1, 2, 10 for one logit
vector. At T = 10 it should be nearly uniform.
**Explain:** why does T = 0 need special handling?

### Q4. top-k and top-p
**Build:** both filters, each taking logits and returning a filtered distribution.
**Check:** top-k=1 equals greedy. For top-p, use a **peaked** logit vector and a **flat**
one, and print how many tokens survive in each.
**Explain:** from those two counts, state precisely why top-p adapts and top-k does not.

### Q5. The standard pipeline
**Build:** one function: logits → temperature → top-k → top-p → softmax → sample, with a
seedable random source.
**Check:** the same seed gives the same token. Disabling all filters reproduces plain
sampling.
**Explain:** why this order? What goes wrong if you apply top-p before temperature?

### Q6. Repetition penalties
**Build:** frequency and presence penalties, applied to logits given the tokens generated
so far.
**Check:** feed a context where one token repeats 5 times and show its logit dropping.
**Explain:** name one case where each penalty damages legitimate output.

### Q7. Stopping conditions
**Build:** a generation loop that halts on any of: EOS token, `max_tokens`, or a stop
string appearing in the decoded text.
**Check:** all three trigger in tests. The stop string is not left in the returned output.
**Explain:** how would a user tell a `max_tokens` truncation apart from a model that simply
finished? Why does that distinction matter in a product?

### Q8. Put Phase 1 together
**Build:** connect Topic 2's tokenizer, Topic 3's embeddings, Topic 4's forward pass, and
this decoder into one `generate(prompt)` function.
**Check:** it produces tokens end to end without crashing. The output is gibberish — the
weights are still random.
**Explain:** you now have a complete untrained LLM. List which parts are *structure* and
which are *learned numbers*, and state what Phase 2 has to add.

### Q9. Settings sweep, judged by eye
**Build:** with your Topic 1 n-gram model as the probability source (it at least produces
real words), generate at: greedy; T=0.7 + top_p=0.9; T=1.3; top_k=5; and T=0.7 + top_p=0.9
+ frequency penalty.
**Explain:** rank the outputs. Which setting would you pick for a factual answer, and which
for creative writing? Justify from what you observed, not from convention.

---

# Done when you can answer

1. What is a logit, and how does it differ from a probability?
2. What does softmax do to gaps between logits?
3. Where does temperature apply, and why is it better there than on probabilities?
4. Why does top-p adapt where top-k cannot?
5. Why do repetition penalties treat a symptom rather than a cause?
6. What are the three ways generation can stop?
7. Name every stage from input text to output text.

Write answers in `notes.md`.

---

**Phase 1 is complete when this file's questions are answered.** You will have built, by
hand: a tokenizer, an embedding layer, attention, a transformer block, and a decoder. Phase
2 makes the numbers learn.
