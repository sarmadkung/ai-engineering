# Topic 24 — RAG

**Why this topic:** RAG is how you give a model knowledge it wasn't trained on — your documents,
your data, today's facts. It is the most common architecture in applied AI, and the thing most
teams implement badly because they stop at the naive version.

---

# Part 1 — Theory

## 24.1 What RAG is and why it exists

```
question -> retrieve relevant text -> put it in the prompt -> model answers from it
```

That's all. The model doesn't "learn" your data; it *reads* it at question time.

Why not fine-tune instead (Topic 14)? Because fine-tuning teaches behaviour, not facts, and
because:

| | RAG | Fine-tuning |
|---|---|---|
| New facts | yes | unreliably |
| Update cost | re-index one document | retrain |
| Citations | natural | impossible |
| Per-user/private data | filter at query time | one model per user? no |
| Behaviour and format | weakly | this is its strength |

They're complementary, and the common mistake is reaching for fine-tuning when the problem is
missing knowledge.

## 24.2 The naive pipeline, and its five failure points

```
1. embed the question
2. vector search -> top k chunks
3. concatenate into a prompt
4. generate
```

This works well enough in a demo and fails in production at each step:

1. **Query mismatch** — the question is phrased nothing like the document. "Why is my card
   declining?" vs a policy page titled "Payment authorization failures".
2. **Retrieval miss** — the answer is in chunk 11 and you took the top 5.
3. **Bad chunks** — retrieved text is relevant but lacks the context needed to use it (Topic 22).
4. **Lost in the middle** — the right chunk is in the context but positioned where the model
   underweights it (Topic 19).
5. **Ungrounded generation** — the model answers from its own parameters instead of the provided
   text, and you cannot tell from the output.

Most of this topic is fixing points 1, 2 and 4.

## 24.3 Context construction

How you assemble retrieved text is a design decision, not string concatenation:

- **Order deliberately** — most relevant first *and* last, given lost-in-the-middle. Never bury
  the best chunk in the middle of ten.
- **Delimit clearly** — each chunk in tags with its source id, so the model can cite and so
  instructions inside a document are less likely to be obeyed (Topic 17, and prompt injection in
  Phase 10).
- **Budget** — cap total retrieved tokens, leaving room for the answer (Topic 19).
- **Instruct for grounding** — tell the model to answer *only* from the provided context, to cite
  sources, and to say "I don't know" when the context doesn't contain the answer. This last
  instruction is what makes the system honest, and it needs testing, not assuming.

## 24.4 Hybrid search

Embeddings and keywords fail in opposite directions (Topic 21), so run both and merge.

**BM25** is the keyword standard: term frequency weighted by rarity, normalized by document
length. Excellent on names, codes, rare terms and exact phrases.

Merging: **reciprocal rank fusion (RRF)** combines rankings without needing comparable scores —
each document scores `sum over lists of 1/(k + rank)`. Simple, robust, and it avoids the mistake
of averaging a cosine similarity with a BM25 score, which are not on the same scale.

Hybrid search is usually the **single largest quality improvement** over naive RAG, and it's
cheap. Do it before anything clever.

## 24.5 Reranking

Retrieval is fast and crude; reranking is slow and accurate. So retrieve widely (top 50) and
rerank down to the best 5.

The mechanism matters: an embedding model encodes query and document *separately* (a bi-encoder),
so it never compares them directly. A **cross-encoder** reranker puts query and document through
a model *together*, which is far more accurate about relevance and far too slow to run over a
whole corpus. Coarse-then-exact again (Topic 23).

Options: hosted rerank APIs, local cross-encoders (`bge-reranker`, `ms-marco-MiniLM`), or an LLM
prompted to score relevance (accurate, expensive, slow).

Reranking also fixes lost-in-the-middle indirectly: you send 5 excellent chunks instead of 20
mediocre ones.

## 24.6 Query rewriting and multi-query retrieval

The query you're given is often not the query you should search with.

- **Rewriting** — expand abbreviations, add context, make it declarative. Essential for follow-up
  questions in a conversation: "what about the second one?" is unsearchable, and must be rewritten
  into a standalone query using the history. **This is the most-skipped step in chat-based RAG**,
  and the reason conversational retrieval quality collapses after turn one.
- **Multi-query** — generate 3–5 paraphrases, retrieve for each, merge with RRF. Covers vocabulary
  mismatch cheaply.
- **Decomposition** — split a multi-part question ("how do X and Y differ?") into sub-questions,
  retrieve for each. Single-vector retrieval cannot serve a two-part question well.
- **HyDE** — have the model write a *hypothetical answer*, then embed that and search with it. A
  fake answer resembles a real document more than a question does. Surprisingly effective.
- **Step-back** — ask a more general version first to retrieve background, then the specific one.

Each costs an extra model call. Measure whether it earns its latency.

## 24.7 The full pipeline

```
question + history
 -> rewrite (standalone query)
 -> multi-query expansion
 -> hybrid retrieve (vector + BM25) per query
 -> merge (RRF), dedupe
 -> rerank (cross-encoder) -> top 5
 -> assemble context (ordered, delimited, budgeted)
 -> generate with grounding instructions + citations
 -> optionally verify the answer is supported
```

Build it incrementally and measure each addition (Topic 25 covers evaluation properly). Not every
stage earns its place in every application, and the only way to know is numbers.

---

# Part 2 — Questions to implement

Build `rag.py` here, on top of Topics 21–23. **First build an evaluation set: 30 questions with
known answers and known source chunks.** Without it, everything below is guesswork.

### Q1. Evaluation set first
**Build:** 30 questions over your corpus, each with the expected answer and the chunk id(s) that
contain it. Include 5 questions your corpus *cannot* answer.
**Check:** a harness that measures retrieval hit rate (was the right chunk retrieved?) and answer
correctness.
**Explain:** why do the 5 unanswerable questions matter more than the other 25?

### Q2. Naive RAG baseline
**Build:** embed → top-5 → concatenate → answer.
**Check:** report retrieval hit rate and answer accuracy.
**Explain:** inspect 5 failures and attribute each to one of §24.2's five failure points.

### Q3. Grounding instructions
**Build:** two prompts — plain, and one instructing "answer only from the context, cite the source
id, say you don't know if it isn't there".
**Check:** measure accuracy, and specifically what happens on your 5 unanswerable questions.
**Explain:** did the naive version hallucinate answers to the unanswerable ones? Report the rate
before and after.

### Q4. Citations
**Build:** make the model cite chunk ids, and verify each citation actually contains the claim.
**Check:** count unsupported citations.
**Explain:** did the model ever cite a real chunk that didn't support its claim? Why is that worse
than no citation?

### Q5. k sweep
**Build:** k = 1, 3, 5, 10, 20.
**Check:** tabulate hit rate, answer accuracy, tokens, cost.
**Explain:** where did accuracy peak? Did more retrieval ever make answers *worse*? Explain via
Topic 19.

### Q6. Position effects
**Build:** with k=10, place the known-correct chunk first, middle, and last.
**Check:** measure accuracy for each position.
**Explain:** report three numbers, and state your ordering policy as a result.

### Q7. BM25 and hybrid
**Build:** BM25 search, then hybrid with RRF.
**Check:** hit rate for vector-only, BM25-only, and hybrid.
**Explain:** report all three. Find one query where BM25 won and one where vectors won, and explain
each.

### Q8. Reranking
**Build:** retrieve 50, rerank to 5 with a cross-encoder.
**Check:** hit rate@5 before and after reranking; added latency.
**Explain:** was the latency worth it? Why can't you rerank the entire corpus?

### Q9. Query rewriting for conversation
**Build:** a two-turn conversation where turn 2 is a pronoun follow-up ("and the second one?").
Retrieve with the raw query, then with a rewritten standalone query.
**Check:** hit rate for both.
**Explain:** report the gap. Why does naive conversational RAG fail here?

### Q10. Multi-query and HyDE
**Build:** both, merged with RRF.
**Check:** hit rate and cost for each against the hybrid baseline.
**Explain:** which helped more on your corpus? Why might HyDE work better on technical documents?

### Q11. Decomposition
**Build:** handle a multi-part question by splitting it, retrieving per part, and answering from
the union.
**Check:** compare against single-query retrieval on 5 such questions.
**Explain:** what specifically failed in the single-query version?

### Q12. The full pipeline, ablated
**Build:** assemble everything, then measure with each stage removed one at a time.
**Check:** a table: configuration → hit rate → accuracy → latency → cost per query.
**Explain:** rank the stages by value per unit of latency. Which would you actually ship, and which
did not earn its place?

### Q13. Adversarial grounding
**Build:** give the model context that *contradicts* well-known facts (e.g. a document stating your
product's refund window is 90 days when the model's prior would say 30).
**Check:** does it follow the context or its training?
**Explain:** which behaviour do you want, and how would you enforce it? What does this tell you
about relying on RAG for authoritative answers?

---

# Done when you can answer

1. What does RAG do that fine-tuning cannot, and vice versa?
2. Name the five failure points of naive RAG.
3. Why is hybrid search usually the biggest single win?
4. What's the difference between a bi-encoder and a cross-encoder, and why does it matter?
5. Why must conversational queries be rewritten before retrieval?
6. How does HyDE work, and why should it work at all?
7. How do you make a RAG system say "I don't know"?

Write answers in `notes.md`.
