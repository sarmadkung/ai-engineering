# Topic 16 — LLM APIs

**Why this topic:** you switch sides here. You stop building models and start building
products on top of someone else's. The model is now a network service with a price, a
latency, a rate limit, and failure modes — and those four things shape your architecture more
than anything else in this phase.

---

# Part 1 — Theory

## 16.1 What an API call actually is

One HTTPS POST. You send a JSON body containing the model name, a list of messages, and
settings; you get back JSON containing generated text and a token usage report.

Everything you built in Phases 1–2 happens on their servers: tokenizer, forward passes,
sampling, detokenizing. Your Phase 1 knowledge is what makes the parameters legible instead
of magic — `temperature` and `top_p` are exactly the knobs from Topic 5.

**But check whether the model still exposes them.** On the current Claude frontier models
(Opus 5, Sonnet 5, and the Fable family) `temperature`, `top_p` and `top_k` have been
**removed — sending one returns a 400**. Reasoning depth is set with `effort` instead
(§16.8). Claude Haiku 4.5 and older models still accept them. This is worth sitting with: the
decoding knobs are not a law of the API, they are a surface a provider chooses to expose, and
a frontier model can withdraw them. Knowing the pipeline underneath is what lets you read that
change as a design decision rather than a mystery.

**Use the official SDK rather than raw HTTP.** It handles retries, streaming assembly, and
typed errors. For Claude that's the `anthropic` package (Python) or `@anthropic-ai/sdk`
(TypeScript).

## 16.2 Messages and roles

A conversation is a list of messages, each with a role:

- **system** — instructions and persona. Not part of the dialogue; the highest-authority
  context. Separate from the message list in the Claude API (a top-level `system` field).
- **user** — input from the person.
- **assistant** — the model's previous replies. You send them back so the model can see what
  it said.

Two facts that follow directly from Phase 1:

- **The API is stateless.** Nothing is remembered between calls. Multi-turn chat works only
  because you resend the whole history every time. That's Topic 1's §1.5 as a billing line.
- Your Phase 3 chat-template knowledge is what the API is doing internally: roles are
  serialized into special tokens before the model sees them.

## 16.3 Models and choosing between them

A provider offers a tier of models trading capability against price and speed. Current
Claude models:

| Model | ID | Context | Input $/1M | Output $/1M |
|---|---|---|---|---|
| Claude Opus 5 | `claude-opus-5` | 1M | $5.00 | $25.00 |
| Claude Sonnet 5 | `claude-sonnet-5` | 1M | $2.00 | $10.00 |
| Claude Haiku 4.5 | `claude-haiku-4-5` | 200K | $1.00 | $5.00 |

**This table is a snapshot, verified 2026-09-24.** Model IDs, context windows, prices and
accepted parameters all change, and old model versions get retired. Check the provider's own
docs before relying on any row of it. Other providers have the same shape: a flagship, a mid
tier, a cheap fast tier.

**Output costs ~5× input.** That's not arbitrary — it's the prefill/decode asymmetry you
measured in Topic 10. Prompt tokens are processed in parallel; output tokens are generated
one at a time.

The practical method: prototype on the strongest model to find out whether the task is
possible at all, then try to move down a tier while measuring quality. Choosing the cheap
model first hides whether failures are your prompt or the model's ceiling.

## 16.4 Streaming

Non-streaming: you wait for the whole response. Streaming: you receive events as tokens are
produced. Same total time, far better perceived speed — and the reason is Topic 10's TTFT.

Streaming is not just a UX nicety. It also prevents HTTP timeouts on long responses, which is
why large `max_tokens` values effectively require it.

The cost is complexity: you handle a sequence of typed events rather than one object, errors
can arrive mid-stream after you've already shown text, and your client must assemble the
pieces. SDKs provide a helper to get the final assembled message when you don't need
per-event handling.

## 16.5 Failure modes, and handling them properly

The API is a network service and will fail. Handle these distinctly rather than catching
everything:

| Status | Meaning | Response |
|---|---|---|
| 400 | bad request (malformed, too many tokens) | fix your code — do **not** retry |
| 401 | bad credentials | fail loudly at startup |
| 404 | wrong model name | fail loudly |
| 429 | rate limited | retry with backoff, honouring `retry-after` |
| 500/529 | provider-side error or overload | retry with backoff |
| timeout | slow or stuck | retry, but beware duplicate work |

**Retry with exponential backoff and jitter** — 1s, 2s, 4s, 8s, each with a random offset so
a fleet of clients doesn't retry in lockstep. Only retry the retryable classes; retrying a
400 just wastes money and time. SDKs retry a couple of times by default, which is a floor,
not a strategy.

## 16.6 Rate limits

Providers limit requests per minute, input tokens per minute, and output tokens per minute.
You can hit the token limit while nowhere near the request limit.

Designing for them: queue and smooth your traffic rather than bursting, cap concurrency
deliberately, and track headroom. In a multi-tenant product, one heavy user can consume the
shared limit and degrade everyone — so per-user throttling is an application concern, not
just a provider one.

## 16.7 Cost management

The levers, most valuable first:

1. **Count tokens before sending.** Use the provider's token-counting endpoint
   (`messages.count_tokens` for Claude) — never a third-party tokenizer, which will be wrong
   for that model. Topic 2 tells you why: token counts are tokenizer-specific.
2. **Prompt caching.** Mark a stable prefix (long system prompt, documents, tool definitions)
   as cacheable and pay a small fraction for repeated reads. This is the single biggest win in
   most real applications, and it is nearly free to adopt. It's prefix-matched, so **any** byte
   change early in the prompt invalidates everything after it — a timestamp in your system
   prompt silently destroys your cache.
3. **Batch API** — ~50% discount for work that can wait (bulk classification, backfills,
   evaluations).
4. **Model choice per route** — a cheap model for classification and routing, an expensive one
   for the hard path.
5. **Cap `max_tokens`** to what the task needs, since output is the expensive half.

**Instrument from day one:** log tokens in, tokens out, cached tokens, model, latency, and
cost per request. Without that you cannot answer "why did the bill triple", and you will be
asked.

## 16.8 Other capabilities worth knowing now

You'll meet these properly in later topics, but know they exist:

- **Tool use / function calling** — the model can request that you run a function (Phase 6).
- **Structured outputs** — constrain the response to a schema (Topic 18).
- **Thinking / reasoning effort** — modern models can spend extra tokens reasoning before
  answering; on current Claude models, adaptive thinking plus an `effort` setting rather than a
  fixed token budget. Better answers on hard problems, more tokens.
- **Vision and documents** — images and PDFs as input (Phase 16).

---

# Part 2 — Questions to implement

Build a small package in this folder: `client.py`, `retry.py`, `costs.py`. You need an API key
in an environment variable — **never** in code, and never committed.

### Q1. First call
**Build:** one non-streaming request. Print the response text and the full usage object.
**Check:** you see input tokens, output tokens, and the model name.
**Explain:** compute the cost of that single call by hand from the usage numbers and the price
table.

### Q2. Prove statelessness
**Build:** send "My name is Sarmad." then, in a *separate* call with no history, "What is my
name?". Then repeat, resending the full history.
**Explain:** describe both results. What does this tell you about where conversation memory
actually lives, and who pays for it?

### Q3. Roles
**Build:** the same user question with three different system prompts (a terse expert, a
patient teacher, a pirate).
**Explain:** what changed? Now put the same instruction in a user message instead. Any
difference in how well it's followed?

### Q4. Growing conversation cost
**Build:** a 10-turn conversation, logging input tokens per turn.
**Check:** input tokens grow every turn.
**Explain:** plot or tabulate it. If a conversation ran for 100 turns, what would happen to
cost per turn? Name two strategies to fix it (Topic 19 is about this).

### Q5. Streaming
**Build:** the same request with streaming, printing text as it arrives. Measure TTFT and total
time for both versions.
**Check:** total times are similar; TTFT is dramatically different.
**Explain:** report all four numbers. Which would a user call "faster", and why?

### Q6. Parameters you already understand
**Build:** the same prompt at temperature 0 and 1.0 (three times each), and with a low
`max_tokens`. Use a model that still accepts sampling parameters — Claude Haiku 4.5 works and
is the cheap tier anyway. Then send `temperature` to a frontier model (Opus 5 or Sonnet 5) as
well, and read the error it returns.
**Explain:** relate each observation to Topic 5. What does the low `max_tokens` output look
like, and how would your application detect that case? Then: you just saw the same parameter
accepted by one model and rejected by another in the same family. What does that tell you
about where decoding settings actually live?

### Q7. Retry with backoff
**Build:** a wrapper with exponential backoff and jitter that retries 429/5xx/timeout and
does **not** retry 400/401/404. Log every attempt.
**Check:** simulate failures (a fake client that throws) and verify the delay sequence and
that 400s fail immediately.
**Explain:** why jitter? And why is retrying a 400 actively harmful?

### Q8. Token counting before sending
**Build:** use the provider's count-tokens endpoint to estimate input cost before a call, and
refuse to send if it exceeds a budget you set.
**Check:** the count matches the `usage.input_tokens` of an actual call closely.
**Explain:** why can't you use a generic tokenizer library for this?

### Q9. Prompt caching, measured
**Build:** a request with a long stable prefix (a few thousand tokens of document or
instructions) marked cacheable. Call it 5 times and log cached vs uncached input tokens and
cost.
**Check:** cache reads appear from the second call onward.
**Explain:** report the saving. Then deliberately break it by inserting the current timestamp
at the **start** of the prompt, and explain what you observe.

### Q10. Cost tracking
**Build:** a logger recording per call: model, input/output/cached tokens, latency, computed
cost, and a request id. Write to a file or SQLite.
**Check:** run 20 varied calls and produce a summary: total cost, mean latency, cost by model.
**Explain:** which single request was most expensive, and why?

### Q11. Model comparison on your own task
**Build:** pick one real task with 10 test inputs. Run it on three model tiers, recording
quality (your judgement, written down), latency, and cost.
**Explain:** tabulate. Which would you ship? At what volume would that answer change?

### Q12. Concurrency and rate limits
**Build:** send 20 requests sequentially, then concurrently with a semaphore capping
concurrency. Measure wall time for both.
**Check:** concurrency is much faster; raising the cap high enough eventually produces 429s
that your Q7 wrapper handles.
**Explain:** how would you choose the cap for a real service? What breaks if you don't have
one?

---

# Done when you can answer

1. What is in an API request, and what comes back?
2. Why is the API stateless, and what does that cost?
3. Why do output tokens cost more than input tokens?
4. What does streaming change, and what doesn't it change?
5. Which errors should you retry, and how?
6. What is prompt caching, and what silently breaks it?
7. Which numbers must you log from day one, and why?

Write answers in `notes.md`.
