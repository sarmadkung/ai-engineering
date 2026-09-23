# Topic 1 — Language Modeling: Theory

Read Part 1 first. It is the whole idea of language models in plain words.
Then do Part 2, which is a list of things to build.

No code here. You write the code.

---

# Part 1 — Theory

## 1.1 What a language model is

A language model reads some text and answers one question:

> **What token probably comes next?**

It does not answer with one token. It answers with a **probability for every token it
knows**. Like this:

```
Text so far: "the capital of France is"

Model's answer:
   " Paris"   92%
   " a"        3%
   " in"       1%
   " Berlin"   0.4%
   ... and a tiny number for every other token
```

All those numbers add up to 100%. That list of numbers is called a **distribution**.

That is it. That is a language model. Text in, distribution out.

A "token" is just a piece of text. In this topic, one token = one character. Later it
becomes a piece of a word. It does not matter yet.

**Why this matters:** ChatGPT and Claude do exactly this. Nothing more. Everything else
— chat, agents, coding help — is built on top of this one small step, repeated.

The model you build in this topic will have the same shape as a real LLM: text in,
distribution out. The only difference is *how* it calculates the numbers.

---

## 1.2 Why guessing the next token is enough to learn a lot

This sounds too simple to work. Here is why it works.

To guess the next token well, you are **forced** to learn things:

| To predict this... | You must know... |
|---|---|
| "probabil**i**ty" | how words are spelled |
| "the capital of France is **Paris**" | a fact about the world |
| "2 + 2 = **4**" | arithmetic |
| "if (x) **{**" | code syntax |
| "Dear Sir or **Madam**" | writing style |

There is no way to get good at predicting text without picking up spelling, facts,
grammar, and style. Skills come along for free, as a side effect of getting good at
prediction.

**And the training data needs no human help.** Take any sentence:

```
"the cat sat on the mat"
```

You already know all the right answers:

```
after "the"                -> "cat"
after "the cat"            -> "sat"
after "the cat sat"        -> "on"
...
```

The text is its own answer key. Nobody has to label anything. This is called
**self-supervised learning**, and it is why models can be trained on the whole internet.

---

## 1.3 How text gets generated (autoregression)

The model only gives you **one** token at a time. So to write a paragraph, you run it
again and again in a loop:

```
1. Give the model the text so far
2. It returns a distribution
3. Pick one token from it
4. Add that token to the text
5. Go back to step 1
```

Example, one line per run of the model:

```
"once upon a"        -> picks " time"
"once upon a time"   -> picks " there"
"once upon a time there" -> picks " was"
```

Each answer becomes part of the next question. The name for this is **autoregressive
generation**. A 1000-word answer means the model ran 1000 times.

Two important side effects:

**1. Generating is slow, and cannot be sped up by doing it all at once.**
Step 10 needs step 9's token first. So the steps must happen one after another.
(Training does not have this problem — see 1.4.) This is why big AI systems care so much
about making each step faster.

**2. Mistakes stick.**
If the model picks a wrong token, that token is now part of the text. Every later step
reads it and treats it as true. The model cannot go back and fix it. This is one reason
AI answers sometimes go off the rails and keep going.

---

## 1.4 Training vs using the model (inference)

These are two completely different activities. Mixing them up causes a lot of confusion.

**Training** = the model reads lots of text and adjusts itself to predict better.
The correct next token is already known (it is right there in the text), so the model
just compares its guess to the truth and improves.

**Inference** = you give the model a prompt and it produces text.
Now the correct answer is *not* known. It just predicts.

| | Training | Inference (using it) |
|---|---|---|
| Is the right answer known? | Yes, it's in the text | No |
| Does it produce new text? | No | Yes |
| Does the model change? | Yes | **Never** |
| Can it do all positions at once? | Yes | No, one at a time |
| When does it happen? | Once, before release | Every time you send a message |

**The most useful thing to take from this table:** when you chat with an AI, it is
**not learning** from you. The model is frozen. It only looks smart about your earlier
messages because that earlier text is placed in front of it again on every turn. *How* the
system does that — resending the messages, keeping them on the server, sending a summary —
is an implementation detail. Either way, the model itself starts from nothing but what it
is shown.

---

## 1.5 Context: the model's only memory

**Context** = the text the model can see right now, while predicting.

That is all the memory it has. No notes. No memory of yesterday. No idea who you are.
If something is not in the context, the model does not know it.

The context has a **maximum size** (a "context window"). In the model you build, the
window will be tiny — maybe 4 characters. A real LLM's window is huge, but the rule is
the same: there is a limit, and past the limit, text is gone.

**Careful: "the model" is not "the AI app."** Keep these apart from now on, because almost
every later phase lives in the outer box, not the inner one:

```
                    AI APPLICATION
  ┌────────────────────────────────────────────────┐
  │  conversation state · memory · retrieval       │
  │  tools · agent loop · prompt assembly          │
  │                                                │
  │  all of it exists to decide ONE thing:         │
  │  what text goes into the box below             │
  │                     │                          │
  │                     ▼                          │
  │            ┌──────────────────┐                │
  │            │      MODEL       │                │
  │            │  frozen weights  │                │
  │            │  + this context  │                │
  │            └──────────────────┘                │
  └────────────────────────────────────────────────┘
```

An application can have databases, notes about you, and years of history. **The model has
weights plus whatever text is in front of it right now** — nothing else. When people say
"the AI remembers me," they mean the application put something in the context. Everything
in Phases 4, 5, 7 and 14 is work in the outer box.

Three real-world consequences:

- Every turn, earlier messages have to be put back into the context somehow — resent,
  held server-side, summarized, or retrieved. Systems differ in the mechanism; none of them
  escape the rule, because the model has no other way to see them.
- Long conversations cost more money, because you are sending more text each time.
- When a chat gets very long, early parts fall out of the window and are simply gone.

Later in the roadmap, "context engineering" (Phase 4) and "RAG" (Phase 5) are both just
about one thing: **choosing wisely what to put in this limited space.**

---

## 1.6 Turning the distribution into one token (decoding)

The model gives you probabilities. Somebody still has to **pick one token**. That
picking is your job, not the model's — and it changes the output a lot.

Say the model returns:

```
" Paris" 60%   " Lyon" 20%   " Nice" 15%   " banana" 0.001%  ...
```

Ways to pick:

**Greedy** — always take the highest one (" Paris").
Same input always gives same output. Problem: it gets stuck in loops. If the model comes
back to text it has seen before, it must make the same choice again, so it repeats
itself forever: *"is a prediction is a prediction is a prediction..."*

**Sampling** — pick randomly, but respecting the percentages. " Paris" 60% of the time,
" Lyon" 20% of the time. More variety, less predictable.

**Temperature** — a dial that reshapes the numbers *before* you pick:

```
low temperature  (0.2)  ->  " Paris" 95%, others tiny     = safe, boring, repetitive
normal           (1.0)  ->  the numbers as they are
high temperature (1.8)  ->  " Paris" 30%, " banana" 2%    = creative, then nonsense
```

Temperature 0 is the same as greedy.

**top-k** — throw away everything except the k best options, then pick.
**top-p** — keep the best options until their total reaches p (say 90%), throw away the
rest, then pick.

Why top-k and top-p exist: a model may give each bad token a tiny chance, but there are
*thousands* of bad tokens. Added up, that is a real chance of picking garbage. Cutting
off the tail removes that risk.

**Remember this:** the same model can look brilliant or broken depending only on these
settings. When output looks bad, check the settings before blaming the model.

---

## 1.7 How to tell if a language model is any good

There is a simple, honest test. Show the model text it has **never seen**. For each
token, ask: *how much probability did you give to the token that actually appeared?*

- High probability = the model was not surprised = good.
- Low probability = the model was surprised = bad.

Average the surprise over all tokens. That average is called **cross-entropy loss**.

**Perplexity** is that number made easier to read (`perplexity = e^cross-entropy`).
Read it as: *how many options did the model feel it was choosing between?*

```
perplexity 1    = certain every time (perfect)
perplexity 2    = like flipping a coin at each token
perplexity 45   = with a 45-token vocabulary, this means it learned nothing
```

So a small number is good, and your vocabulary size is the "learned nothing" baseline.

**One rule you must not break: test on text the model did not train on.** A model that
memorized its training text looks perfect on that text and is useless on anything new.
So always split your data: most of it for training, a slice held back for testing.

This same measurement is what trains real LLMs. In this topic you only measure it. In
Phase 2 you make the model reduce it.

---

## 1.8 Why the model you're building is weak (and what fixes it)

Your model will work by **counting**. It reads the corpus and remembers things like:

```
after "the c"  ->  "a" happened 12 times, "o" happened 5 times, "l" happened 2 times
```

To predict, it looks up the counts and turns them into percentages. That is a real
language model. But it has two problems:

**Problem 1: it cannot spot similar situations.**
Suppose it has seen `the cat sat` many times. Now it sees `the dog sat`.
A human sees an obvious pattern. Your model sees a completely unknown string — for it,
"cat" and "dog" have nothing in common. So it has learned nothing useful and has to
guess blindly.

**Problem 2: more context makes this worse, not better.**
You might think looking at 8 previous characters beats looking at 4. But long
situations almost never repeat. If the model needs an exact match for 8 characters, it
will almost never find one in its counts — so it is blind most of the time.

You will see this happen with real numbers in Question 9. It is the key discovery of
this topic.

**The fix, which is the rest of Phase 1:** stop treating text as exact strings to match.
Turn tokens into **numbers in space** (embeddings, Topic 3) so that "cat" and "dog" land
near each other. Then use **attention** (Topic 4) to compare situations loosely instead
of exactly. Then "the dog sat" can benefit from having seen "the cat sat".

Everything in Part 1 stays true for real LLMs. Only the counting gets replaced.

---

## Quick summary

- A language model turns text into probabilities for the next token.
- Predicting the next token forces it to learn spelling, facts, grammar, style.
- Text is its own answer key, so no human labelling is needed.
- To write longer text, run the model in a loop, one token per run.
- Training and using the model are separate; the model does not learn while you chat.
- Context is the model's only memory, and it is limited.
- You choose how to pick a token: greedy, sampling, temperature, top-k, top-p.
- Measure quality with cross-entropy / perplexity, on text it has never seen.
- Counting works but cannot generalize. Embeddings and attention fix that.

---

# Part 2 — Questions to implement

Build one file: **`ngram_lm.py`** in this folder. Standard library only (`math`,
`random`, `collections`). Nothing to install.

Data provided: `corpus.txt` (3.7 KB) and `corpus_alt.txt` (1.5 KB).

Do these in order. Each one has:
- **Build** — what to write
- **Check** — how to know it works, before moving on
- **Explain** — a question to answer in your own words in `notes.md`

## Group A — Setup

### Q1. Load and split the data
**Build:** read `corpus.txt`. Split it into two pieces: about 90% for training, the last
10% kept aside for testing. Print both lengths.
**Check:** both pieces are non-empty, and the two lengths add up to the file size.
**Explain:** why must the model never see the test piece during training?

### Q2. Tokenizer
**Build:** two functions — text to a list of tokens, and a list of tokens back to text.
Use single characters as tokens. Also build the **vocabulary**: the sorted list of
distinct characters in the training text.
**Check:** converting text to tokens and back gives the exact original text. Print the
vocabulary size (expect a few dozen).
**Explain:** characters are very small tokens. What does the model have to do more often
because of that?

## Group B — Training

### Q3. Model setup
**Build:** a class with one setting, `order` = how many previous tokens the model looks
at, plus one. So `order = 5` means it looks at 4 previous tokens. **Those 4 tokens are
its entire context window.**
You need a way to store, for each context, how many times each token followed it.
Hint: a dictionary where the key is a context and the value counts the next tokens. Keys
must be hashable, so use a tuple of tokens, not a list.
**Explain:** what does `order = 1` mean — how much does the model look at?

### Q4. Training
**Build:** a `fit(text)` method. Walk through the training tokens once. At each position,
record "this context was followed by this token".
One detail: the first few positions do not have enough previous tokens. Solve it by
adding a special start token (like `"<s>"`) at the front, enough times to fill the window.
**Check:** print how many different contexts you recorded. Then pick one context by hand
(for example the 4 characters before the `i` in "probability") and check its counts look
right.
**Explain:** your `fit` produces no text at all. Looking at the table in 1.4, which
column is it?

## Group C — Prediction

### Q5. `distribution(context)`
**Build:** given a context, return a probability for **every token in the vocabulary**,
adding up to 1. You need to handle:
- a context longer than the window → use only the last few tokens
- a context shorter than the window → pad the front with the start token
- a context never seen in training → must not crash or divide by zero
- a token that never followed this context → must not get probability exactly 0,
  because you will take `log` of it later and `log(0)` breaks everything

Fix the last two with **add-k smoothing**: before turning counts into percentages, add a
small number `k` (start with 0.01) to the count of *every* token, including the zeros.
Make `k` a setting you can change.

**Check:** the numbers add up to 1.0 for: a normal context, an unseen context, an empty
context, and a too-long context. Then print the top few predictions after `"probabil"` —
one should be very high. If not, your context handling is wrong.
**Explain:** with add-k, what does your model return for a context it has never seen? Is
that a reasonable thing for it to say?

### Q6. `choose(distribution)`
**Build:** takes a distribution, returns one token. Support:
- greedy (treat temperature 0 as "take the highest")
- temperature
- top-k

For top-k plus temperature, do it in this order: sort, keep the top k, then apply
temperature and re-scale so the kept ones add to 1.
Hint for random picking: pick a random number between 0 and the total, then walk through
the options adding up their weights until you pass it.
Pass in a `random.Random` object so you can set a seed and repeat the same result.

**Check:** temperature 0 gives the same token every time. `top_k = 1` matches greedy.
Very low temperature almost always gives the top token. Now a stronger test: pick 10,000
times at temperature 1 and count results — the counts should roughly match the input
percentages.
**Explain:** why does this belong outside the model instead of inside it?

## Group D — Generating text

### Q7. `generate(prompt, max_tokens)`
**Build:** the loop from 1.3. Turn the prompt into tokens, then repeat: get a
distribution, choose a token, add it to the list. Stop after `max_tokens`.
**Check:** generate 200 characters. It should look like broken English — real words,
sentences that make no sense. If it is complete gibberish, Q5 has a bug.
**Then:** print one line per step showing the context, the top 3 candidates with their
percentages, and the token chosen. Watch the context move forward by one each line, and
watch each chosen token appear in the next line's context. That picture is autoregression.
**Explain:** the prompt is not an instruction. What is it, really? And what would have to
change for the model to learn from what it just wrote?

## Group E — Measuring

### Q8. `cross_entropy(text)` and `perplexity(text)`
**Build:** walk the text the same way `fit` did (same padding, same context). But instead
of counting, look up the probability your model gave to the token that actually appeared,
take `-log` of it, and average over all positions. Perplexity is `exp` of that average.
Decide what to do about characters that are not in the vocabulary, and remember what you
chose.
**Check:** run it on the training text and on the test text. Training should be lower.
Both should be well under the vocabulary size. If the test score is *better* than
training, you have a bug.
**Explain:** write down your test perplexity and your vocabulary size. What does that
number mean in "how many options did it feel it had" terms?

## Group F — Experiments (this is where the learning happens)

### Q9. Does more context help?
**Build:** train separate models with `order` = 1, 2, 3, 4, 5, 6, 7, 8 on the same text.
Make a table: order, number of contexts stored, training perplexity, test perplexity.
**Explain:** training perplexity keeps improving. Test perplexity does not — it gets
better, then turns around and gets worse. Find where it turns. Then explain **why more
context makes it worse**, using section 1.8. Save this answer; Topic 4 needs it.

### Q10. Same model, different picking settings
**Build:** same prompt, same seed. Generate at temperature 0, 0.3, 0.8, 1.0, 1.6, and
with `top_k = 3`.
**Explain:** greedy repeats itself — explain why that is guaranteed, not bad luck. And
find the temperature where the text stops being readable.

### Q11. Is temperature really the problem?
**Build:** for a context seen only once, calculate how much of the total probability
add-k handed out to tokens that never actually appeared. Then re-run Q10 with `k` 100
times smaller, and measure test perplexity again.
**Explain:** the generated text gets better, but test perplexity gets *worse*. Explain
both. Then write one sentence about what this teaches you regarding metrics and settings.

### Q12. The data is the model
**Build:** train on `corpus_alt.txt` instead, same settings, and generate.
**Explain:** the style of the output changes completely, and you changed no code. What
does that tell you about training data?

## Group G — Optional extras

### Q13. Backoff
When a context has never been seen, instead of giving up, try again with one fewer token
of context. Re-run Q9 and report how much the test perplexity improves.

### Q14. top-p sampling
Add it next to top-k: keep the best tokens until their total reaches p, then pick.
Compare both on the same seed.

### Q15. Word tokens
Change Q2 to split on spaces instead of characters. Report the new vocabulary size and
how many contexts were seen only once. This pain is exactly why Topic 2 exists.

---

# You are done when you can answer these without looking

1. What does a language model actually output?
2. Why does predicting the next token teach a model facts and grammar?
3. Why is generating text slow, while training is not?
4. What exactly is your model's context window?
5. Why does greedy picking get stuck repeating itself?
6. Your test perplexity was about 20 with a 45-token vocabulary. What does that mean?
7. Why did counting stop working when you gave it more context?
8. Name one time the picking settings made the model look worse than it was.

Write your answers in `notes.md`. If one comes out fuzzy, re-read that section. Ask me to
quiz you and I will ask questions, not give answers.
