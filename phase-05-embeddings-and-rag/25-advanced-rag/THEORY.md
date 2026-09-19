# Topic 25 — Advanced RAG

**Why this topic:** the techniques that fix the structural limits of Topic 24 — chunks too small to
answer from, questions that need multiple hops, and the fact that you can't improve any of it
without measurement. This is also where RAG stops being a pipeline and becomes an agent.

---

# Part 1 — Theory

## 25.1 The small-chunk dilemma

Small chunks retrieve well (focused embedding) but answer badly (missing context). Large chunks
answer well but retrieve badly (diluted embedding). You cannot win by picking a size.

The insight that resolves it: **the unit you search does not have to be the unit you read.**

## 25.2 Parent-child (small-to-big) retrieval

Index small chunks; return their larger parents.

```
search over:   200-token children (precise matching)
send to model: the 1500-token parent section that contains the hit
```

Variants: **sentence-window** (retrieve a sentence, expand to N sentences either side) and
**hierarchical** (child → section → document, expanding as needed). Both give precise retrieval
with sufficient context, and both need deduplication when several children share a parent.

This is usually the second-biggest quality win in RAG after hybrid search, and it's cheap.

## 25.3 Multi-vector retrieval

One chunk, several vectors — because a chunk can be relevant for several different reasons.

- **Summary embedding** — embed a generated summary alongside the raw text; summaries match
  high-level questions better.
- **Hypothetical questions** — generate the questions a chunk answers, embed those, and match
  question-to-question rather than question-to-prose. Removes the query/document asymmetry at the
  root.
- **Title + body**, or **per-section** vectors.
- **ColBERT-style late interaction** — a vector per token, matched token-wise. Much better recall,
  much more storage.

Cost: more vectors to store and generate (the hypothetical-question variant needs an LLM call per
chunk at ingestion). Ingestion cost buys query quality — often the right trade, because you ingest
once and query forever.

## 25.4 Graph RAG

Vector search retrieves *similar* text. It cannot answer questions that require *connecting*
facts across documents — "which of our suppliers are affected by the sanctions we discussed in the
Q3 memo?" needs a join, not a similarity.

Graph RAG builds a knowledge graph — entities as nodes, relationships as edges, extracted from the
corpus by an LLM — then traverses it, optionally combining with vector search over node
descriptions. Community summaries (clusters of related entities) support genuinely global questions
("what are the main themes across these 500 documents?") that no top-k retrieval can answer.

The honest assessment: expensive to build (an LLM pass over the entire corpus), hard to keep
current, and it fails when entity extraction is inconsistent ("IBM" vs "I.B.M." as two nodes).
Worth it for multi-hop and global questions over a stable corpus; overkill for a FAQ bot.

## 25.5 Agentic RAG

Stop treating retrieval as a fixed pipeline step, and give the model retrieval as a **tool** it
decides how to use (Phase 6 and 7 build this properly).

What that enables:

- **Deciding whether to retrieve at all.** "Hello" needs no search; naive RAG retrieves anyway,
  wasting money and polluting context.
- **Iterating.** Search, read, notice what's missing, search again with a better query.
- **Choosing a source.** Documentation vs code vs tickets vs the web.
- **Self-correction.** "These results don't answer the question" → reformulate.
- **Multi-hop by construction.** Find A, use it to search for B.

The cost is variable latency, variable spend, and loops that can go wrong. You need iteration
caps, a budget, and tracing (Phase 9). This is the direction the field has moved, and the reason
Phase 7 follows Phase 5 in the roadmap.

## 25.6 Other techniques worth knowing

- **Contextual retrieval** — prepend an LLM-generated sentence situating each chunk in its document
  before embedding it ("This chunk is from the 2024 refund policy, discussing the appeal process").
  Substantially improves retrieval for a one-off ingestion cost; a strong, simple upgrade.
- **Self-RAG / CRAG** — the model grades retrieved documents for relevance and its own answer for
  support, retrying when either fails.
- **Corrective fallback** — if retrieval confidence is low, fall back to web search or to an
  explicit "I don't know".
- **Query routing** — classify the query and send it to the right index, to SQL, or to no retrieval
  at all.

## 25.7 RAG evaluation

You cannot improve what you don't measure, and RAG has **two** stages that fail differently. Always
evaluate them separately — a wrong answer from correct retrieval is a generation problem, and
tuning your chunker will not fix it.

**Retrieval metrics**

- **hit rate / recall@k** — was a correct chunk retrieved at all? The first thing to measure.
- **MRR** — how high up was the first correct chunk?
- **NDCG** — rank-weighted quality when relevance is graded rather than binary.
- **context precision** — what fraction of retrieved context was actually relevant? (Wasted tokens.)

**Generation metrics**

- **faithfulness / groundedness** — is every claim in the answer supported by the context? The most
  important metric in RAG, because it measures hallucination directly.
- **answer relevance** — does it address the question asked?
- **correctness** — does it match the known answer?

**How to measure:** a golden set of 50–200 (question, answer, source chunk) triples, built from
real user questions where possible. Grade with exact/keyword checks where you can, and an **LLM
judge** for faithfulness and relevance — with the caveats that a judge needs its own validation
against human labels, is biased toward verbose answers, and should be a strong model.

Frameworks (`ragas`, `trulens`, `deepeval`) implement these. Build your harness once by hand
first, so you know what the numbers mean. Phase 9 covers evaluation as a discipline.

---

# Part 2 — Questions to implement

Extend Topic 24's pipeline here. Your 30-question evaluation set is the instrument for every
question below — expand it to 50 if you can.

### Q1. Separate your metrics
**Build:** a harness reporting retrieval metrics (hit rate@k, MRR, context precision) and
generation metrics (correctness, faithfulness) **separately**.
**Check:** find one case of good retrieval + wrong answer, and one of bad retrieval + right answer.
**Explain:** what would you have concluded if you'd only measured end-to-end accuracy?

### Q2. An LLM judge for faithfulness
**Build:** a judge that, given context + answer, flags every unsupported claim.
**Check:** validate it — hand-label 20 answers yourself and measure the judge's agreement with you.
**Explain:** report agreement. Where did the judge disagree with you, and who was right?

### Q3. Parent-child retrieval
**Build:** index 200-token children, return 1500-token parents, deduplicated.
**Check:** hit rate and answer accuracy against Topic 24's baseline.
**Explain:** report both. Show one question that only worked with parent expansion, and say why.

### Q4. Sentence-window
**Build:** retrieve single sentences, expand to ±3 sentences.
**Check:** compare against parent-child on the same questions.
**Explain:** which suited your corpus better, and what property of the corpus decided it?

### Q5. Hypothetical-question embeddings
**Build:** generate 3 questions per chunk at ingestion, embed them, and search against them.
**Check:** hit rate versus embedding the chunk text. Record the ingestion cost.
**Explain:** did question-to-question matching help? Was the one-off ingestion cost worth it at
your query volume? Show the arithmetic.

### Q6. Contextual retrieval
**Build:** prepend an LLM-generated contextualizing sentence to each chunk before embedding.
**Check:** hit rate before and after; ingestion cost.
**Explain:** report the improvement. Which chunks benefited most?

### Q7. Multi-vector
**Build:** store both a summary vector and a raw-text vector per chunk; search both and merge.
**Check:** measure on high-level questions and on specific-detail questions separately.
**Explain:** which vector won for which question type? What does that suggest about routing?

### Q8. A multi-hop question that defeats vector search
**Build:** write 5 questions requiring information from two different documents to answer.
**Check:** measure naive RAG's accuracy on them.
**Explain:** show why top-k retrieval structurally cannot answer one of them.

### Q9. Graph RAG, small scale
**Build:** extract entities and relationships from 50 documents with an LLM, store as a graph, and
answer your multi-hop questions by traversal plus retrieval.
**Check:** accuracy on Q8's questions versus naive RAG. Record build cost.
**Explain:** did it work? Count your duplicate/inconsistent entities. What would maintaining this
graph over changing documents require?

### Q10. Agentic RAG
**Build:** give the model a `search(query)` tool and let it decide when and how often to call it,
with an iteration cap.
**Check:** measure accuracy, number of searches, latency and cost on your full set — including on
questions needing no retrieval, and on Q8's multi-hop questions.
**Explain:** report all of it. Where did it beat the fixed pipeline, and where was it just more
expensive?

### Q11. Retrieval confidence and fallback
**Build:** a confidence check on retrieval (score threshold or judge) that falls back to "I don't
know" rather than answering.
**Check:** measure on your unanswerable questions and confirm you didn't break the answerable ones.
**Explain:** report both rates. What's the cost of being too cautious?

### Q12. Query routing
**Build:** a classifier routing each query to: no retrieval, document retrieval, or structured
lookup/SQL.
**Check:** routing accuracy, plus end-to-end accuracy and cost against always-retrieve.
**Explain:** how much did you save by not retrieving when unnecessary?

### Q13. The final ablation
**Build:** measure every configuration you've built: naive, +hybrid, +rerank, +parent-child,
+contextual, +rewriting, +agentic.
**Check:** one table — hit rate, faithfulness, correctness, latency, cost per query.
**Explain:** pick the configuration you would ship and defend it on the numbers. Which fashionable
technique did *not* earn its place on your corpus?

---

# Done when you can answer

1. What is the small-chunk dilemma, and how does parent-child resolve it?
2. Why would you store more than one vector per chunk?
3. What can Graph RAG answer that vector search cannot, and what does it cost?
4. What does agentic RAG add over a fixed pipeline, and what risks come with it?
5. Why must retrieval and generation be evaluated separately?
6. What is faithfulness, and why is it the most important RAG metric?
7. How do you validate an LLM judge?

Write answers in `notes.md`.

---

**Phase 5 is complete.** Your application can now use your own data, and you can measure whether
it does so well. Phase 6 gives the model the ability to act, not just read.
