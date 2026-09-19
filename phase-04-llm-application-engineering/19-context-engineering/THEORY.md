# Topic 19 — Context Engineering

**Why this topic:** the context window is a fixed budget, and deciding what goes in it is the
central engineering problem of LLM applications. Prompt engineering is what you *write*;
context engineering is what you *choose to include*. Every quality, cost and latency problem in
later phases comes back to this.

---

# Part 1 — Theory

## 19.1 Context is a budget

Every request has a hard token limit shared by system prompt, tool definitions, conversation
history, retrieved documents, and the space left for the answer. Modern models offer very large
windows (1M tokens on current Claude models), which changes the problem but doesn't remove it:

- **Cost is linear in tokens.** A million-token context on every request is ruinous at scale.
- **Latency rises with input length**, because prefill still has to process it.
- **Quality does not improve monotonically with more context.** More is not better.

So the job is not "fill the window" but "spend the budget on what earns its place".

## 19.2 Lost in the middle

The single most important empirical fact in this topic: models attend most reliably to
information at the **beginning** and **end** of a long context, and least reliably to the
middle.

Consequences:

- Put instructions early and the actual question **last**.
- Put the most important retrieved document first or last, never buried at position 7 of 15.
- More retrieved chunks can *lower* accuracy by pushing the good one into the middle and
  diluting attention.

This is why "just retrieve 50 documents" fails, and why reranking (Phase 5) exists.

## 19.3 Context rot

As a context grows, quality degrades in ways that aren't about hitting the limit:

- **Distraction** — irrelevant text competes with relevant text.
- **Contradiction** — an early "the user prefers Python" and a later "the user prefers Go" leave
  the model guessing; stale content is worse than missing content.
- **Instruction dilution** — a system prompt 100k tokens back has less influence than one right
  before the question.
- **Its own output** — a long transcript of the model's earlier mistakes conditions it to
  continue in that vein (Topic 1's compounding errors, at conversation scale).

The practical upshot: **actively curate**. Deliberately drop what no longer helps.

## 19.4 Context selection

Given more candidate content than fits, choose:

- **Relevance** — semantic similarity to the current question (Phase 5's retrieval).
- **Recency** — recent turns usually matter more than old ones.
- **Importance** — some facts must always be present (the user's account tier, their name,
  a hard constraint).
- **Explicit pinning** — let the system (or user) mark items as always-included.

A standard layout that respects §19.2:

```
[system prompt + tools]        stable, cacheable, early
[pinned facts / summary]       small, always present
[retrieved context]            selected per request, most relevant first
[recent conversation turns]    a bounded window
[the current question]         LAST
```

Stable content first is not just tidiness — it's what makes prompt caching work (Topic 16).

## 19.5 Compression

Four techniques, in increasing aggressiveness:

- **Truncation** — drop the oldest turns. Cheap, and loses things irreversibly.
- **Summarization** — replace old turns with a summary. Keeps the gist; loses detail and can
  introduce errors, which then become "facts" in the context.
- **Rolling summary** — maintain a running summary updated each turn, plus the last N turns
  verbatim. The standard approach for long chat, and what "compaction" features implement
  server-side.
- **Extraction to structured memory** — pull durable facts out into a store (name, preferences,
  decisions) and inject only those. The most robust, and the bridge to Phase 7's agent memory.

The key judgement: summarize the *narrative*, extract the *facts*. Facts survive compression
badly when they're inside prose.

## 19.6 Conversation history

Design decisions you must make explicitly, because defaults are bad:

- **How many turns to keep verbatim?** A fixed number, or a token budget (better).
- **What to do with tool calls and results?** They're often the bulk of an agent's history and
  the least useful later. Clearing old tool results is a standard move — some APIs offer it as
  a feature.
- **System prompt position** — resent every turn; keep it first and byte-stable.
- **Do you keep the model's failed attempts?** Usually no: they teach the wrong pattern.

## 19.7 Long-context strategies

When input genuinely exceeds what you can or should send:

- **Chunk and retrieve** — index it, fetch only relevant parts (Phase 5). The default answer.
- **Map-reduce** — process chunks independently, then combine the results. Good for
  summarizing a book; bad for questions needing cross-chunk reasoning.
- **Refine** — process chunks in sequence, carrying a running answer. Handles dependencies;
  serial and slow.
- **Hierarchical** — summarize sections, then summarize the summaries; drill down on demand.
- **Let the model navigate** — give it tools to search and read on demand rather than
  pre-loading everything. This is the agentic answer (Phase 7), and increasingly the right one.

**Choosing between long context and retrieval:** long context is simpler and better when the
material is small enough and questions need global understanding. Retrieval wins on cost at
scale, on corpora that don't fit at any price, and on freshness. It is a cost/quality decision
you should be able to defend with numbers, not a matter of fashion.

---

# Part 2 — Questions to implement

Build `context.py` in this folder: a context manager that assembles requests within a token
budget. Use your Topic 16 token counter for every measurement.

### Q1. Measure the budget
**Build:** a function reporting, for a planned request: tokens for system prompt, tools,
history, retrieved content, and remaining space for output.
**Check:** the total matches the API's reported `input_tokens` closely.
**Explain:** for your typical request, what fraction of the budget does each part consume?

### Q2. Reproduce lost-in-the-middle
**Build:** a long context of ~20 similar paragraphs where exactly one contains a specific fact
("the access code is 7741"). Ask for that fact with the target paragraph at position 1, middle,
and last. Run each 10 times.
**Check:** record accuracy per position.
**Explain:** report your three numbers. Did you reproduce the effect? What does it imply for
how you order retrieved documents?

### Q3. More context, worse answers
**Build:** answer a question with 1, 3, 10, and 30 retrieved paragraphs, where only one is
relevant. Score accuracy and record tokens and latency.
**Explain:** find the point where adding context stopped helping or started hurting. Why does
it hurt?

### Q4. Turn-window vs token-budget history
**Build:** two history managers — keep the last 10 turns, and keep as many turns as fit in a
token budget.
**Check:** run a conversation with wildly varying message sizes through both.
**Explain:** which one fails badly, and in which direction? Which would you ship?

### Q5. Truncation vs summarization
**Build:** a 30-turn conversation. At turn 20, compress with (a) truncation and (b)
summarization. Then ask a question whose answer was in turn 3.
**Check:** record whether each can answer.
**Explain:** what did each lose? Which loss is more dangerous, and why?

### Q6. Rolling summary
**Build:** maintain a running summary plus the last 5 turns verbatim, updating the summary as
turns fall out of the window.
**Check:** over 40 turns, tokens per request stay roughly flat instead of growing.
**Explain:** plot tokens per turn for this versus resending everything. What is the cost curve
difference?

### Q7. Summary drift
**Build:** run your rolling summary for 40 turns, re-summarizing each time. Then compare the
final summary against the actual transcript.
**Check:** look for facts that changed, disappeared, or got invented.
**Explain:** what did repeated summarization do? Why is this an argument for extraction over
summarization?

### Q8. Structured memory extraction
**Build:** after each turn, extract durable facts into a structured store (Topic 18's typed
output), and inject only those facts plus recent turns.
**Check:** ask about a fact from turn 2 at turn 40 and get it right, with a much smaller context
than the full transcript.
**Explain:** compare tokens and accuracy against Q6. What does this approach lose?

### Q9. Contradiction
**Build:** a conversation where the user changes their mind ("use Python" → later "actually use
Go"). Ask a question whose answer depends on the current preference.
**Check:** does the model use the latest? Now put the old preference in a summary and the new
one in recent turns.
**Explain:** what happened, and how should a memory system handle updates rather than
accumulation?

### Q10. Caching-aware assembly
**Build:** restructure your context builder so stable content is always first and byte-identical
across requests, with a cache breakpoint before the volatile part. Measure cached tokens over 20
requests in a conversation.
**Explain:** report the cost saving. Which of your components were you tempted to put early and
had to move?

### Q11. Long document, three ways
**Build:** take a document far too long to send (a book, a long spec). Answer the same question
with map-reduce, refine, and chunk-and-retrieve.
**Check:** record quality, cost, latency and number of calls for each.
**Explain:** which won for a *local* question ("what does section 4 say")? Which for a *global*
one ("what is the author's overall argument")? Explain the difference.

### Q12. Long context vs retrieval, decided with numbers
**Build:** for a corpus that *does* fit in the window, answer 10 questions both ways: whole
corpus in context, and retrieval of the top 3 chunks.
**Explain:** tabulate accuracy, cost, and latency. State the corpus size at which your answer
would flip, and show the arithmetic.

---

# Done when you can answer

1. Why is a bigger context window not a solution to context management?
2. What is lost-in-the-middle, and how does it change your layout?
3. Name four ways context degrades as it grows.
4. What order should the parts of a request go in, and why does caching constrain it?
5. When do you summarize, and when do you extract facts instead?
6. What are map-reduce, refine, and retrieval each good for?
7. How would you decide between long context and retrieval for a real system?

Write answers in `notes.md`.
