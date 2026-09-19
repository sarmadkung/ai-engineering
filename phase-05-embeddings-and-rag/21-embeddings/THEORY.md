# Topic 21 — Embeddings

**Why this topic:** Topic 3 covered embeddings *inside* a model. This topic is embeddings as a
*product*: turning documents and queries into vectors so you can search by meaning. It's the
foundation of everything in Phase 5.

---

# Part 1 — Theory

## 21.1 Text embeddings vs token embeddings

Same idea, different scope:

| | Token embeddings (Topic 3) | Text embeddings (here) |
|---|---|---|
| One vector per | token | sentence, paragraph, document |
| Lives | inside the model | in your database |
| Produced by | the model's first layer | a dedicated embedding model |
| Used for | making the model work | search, clustering, classification |

A text embedding model is usually an **encoder** (Topic 11) — bidirectional, since it needs to
understand the whole text at once, not continue it. It compresses meaning into a fixed-length
vector, typically 384 to 3072 numbers, regardless of input length.

## 21.2 What "meaning" means here

Two texts with similar embeddings are texts that would appear in similar contexts. That's it —
and it has consequences worth knowing before you debug your first retrieval failure:

- "How do I cancel my subscription?" and "I want to stop paying for this" land close together
  despite sharing almost no words. This is the win over keyword search.
- "The flight was delayed" and "The flight was not delayed" land *very* close together, because
  negation barely moves an embedding. This is a real and permanent weakness.
- Numbers, dates, ids and codes embed poorly. "Invoice 4471" and "Invoice 4417" are nearly
  identical vectors. Never rely on embeddings to match identifiers.

## 21.3 Choosing an embedding model

The axes that matter:

- **Quality** — check a current leaderboard (MTEB) for your language and task, not a blog post
  from two years ago.
- **Dimensions** — more is usually slightly better and definitely more expensive to store and
  search. 768–1536 is the common sweet spot.
- **Max input length** — often 512 tokens. Text longer than this is silently truncated, which
  means the end of your chunk contributes nothing. A frequent and invisible bug.
- **Hosted vs local** — API (simple, per-token cost, data leaves your network) or local
  (`sentence-transformers`, free after setup, runs on CPU for small models).
- **Symmetric vs asymmetric** — some models are trained for query↔document matching where the two
  sides look different (a short question vs a long passage) and expect a **prefix** like
  `"query: "` / `"passage: "`. Using the wrong prefix, or none, quietly degrades results.

**The rule that matters most:** the embedding model is part of your index. Changing it invalidates
every stored vector, so you must re-embed everything. Choose deliberately, record which model
produced each vector, and treat a model change as a migration.

## 21.4 Similarity metrics

You built these in Topic 3:

- **Cosine similarity** — angle between vectors, ignores length. The default.
- **Dot product** — identical to cosine when vectors are normalized, and faster.
- **Euclidean distance** — straight-line gap; ranks the same as cosine on normalized vectors.

Most embedding models emit normalized vectors, in which case all three agree on ranking. Know
which your model and your database use, because mixing metrics silently produces bad rankings.

**Calibrate your scores before trusting thresholds.** Cosine similarity between two random
English sentences is often 0.5–0.7, not 0. So "0.8 means similar" is not a universal truth — you
must measure the distribution for *your* model and *your* corpus before choosing a cutoff.

## 21.5 Semantic search, end to end

```
indexing (once):   documents -> chunks -> embed -> store vectors + text + metadata
querying (each):   query -> embed -> compare against all vectors -> top k -> return text
```

Two properties to internalize:

- **Retrieval always returns something.** There is no "no results" — the top match of an
  irrelevant query is still the top match. You must apply a score threshold yourself, or your
  system will confidently answer from unrelated documents.
- **Exact search is linear** in corpus size. Fine for thousands of vectors, too slow for millions,
  which is what the approximate indexes in Topic 23 solve.

## 21.6 Where embeddings beat keywords, and where they don't

Embeddings win on paraphrase, synonyms, cross-lingual matching, and vague questions. Keyword
search (BM25) wins on exact terms, rare words, names, codes, and identifiers — precisely where
embeddings are weakest.

They fail in *opposite* directions, which is why serious systems use both and combine the results
(hybrid search, Topic 24). Knowing this now saves you from concluding "RAG doesn't work" when what
you actually needed was a keyword match on a product code.

## 21.7 Other uses

Beyond search: clustering (group similar documents), classification (embed + train a small
classifier, often beating an LLM on cost for high-volume work), deduplication (near-identical
vectors), recommendation, and anomaly detection. A useful fallback to remember — for a
high-volume classification task, embeddings plus logistic regression may be 100× cheaper than an
LLM call and just as accurate.

---

# Part 2 — Questions to implement

Build `embed.py` and `search.py` here. Use a local model (`sentence-transformers`,
`all-MiniLM-L6-v2` is small and fine) or an API. You need a corpus of a few hundred text
snippets — your own notes, documentation, or scraped articles.

### Q1. First embeddings
**Build:** embed 5 sentences. Print the vector length and the first few values.
**Check:** all vectors have identical length regardless of input length.
**Explain:** what does "fixed length regardless of input" imply about what gets lost?

### Q2. Similarity behaves as expected
**Build:** cosine similarity for: identical sentences, paraphrases, related topics, unrelated
sentences.
**Check:** the ordering is sensible.
**Explain:** report the four numbers. Is unrelated close to 0? What does that tell you about
choosing thresholds?

### Q3. Calibrate your score distribution
**Build:** compute similarity for 200 random pairs from your corpus and plot or bucket the
distribution.
**Explain:** what is the mean and spread for *unrelated* text in your corpus? Now pick a
principled threshold and justify it with these numbers.

### Q4. The negation failure
**Build:** compare "The flight was delayed" with "The flight was not delayed", and a few similar
pairs of your own.
**Check:** similarity is very high.
**Explain:** why does this happen? Name a real application where this would cause a serious bug.

### Q5. The identifier failure
**Build:** compare "Invoice 4471" with "Invoice 4417" and with "Invoice 9982".
**Explain:** report the numbers. What must you use instead for id lookup?

### Q6. Truncation, caught in the act
**Build:** embed a text longer than the model's max input, then embed only its first half.
**Check:** the two vectors are nearly identical.
**Explain:** what happened to the second half? How would you detect this in a pipeline (and what
should the pipeline do about it)?

### Q7. Semantic search
**Build:** embed your whole corpus, then a search function returning top-k by cosine similarity.
**Check:** 10 queries of your own return sensible results.
**Explain:** which query worked best, which worst, and why?

### Q8. Embeddings vs keywords, head to head
**Build:** a simple keyword search (word overlap or BM25) over the same corpus. Run the same 10
queries through both.
**Check:** tabulate which method won per query.
**Explain:** characterize the queries where each wins. Does the split match §21.6?

### Q9. No-results handling
**Build:** query with something completely unrelated to your corpus.
**Check:** results are still returned, with scores.
**Explain:** apply your Q3 threshold. What score cutoff correctly rejects this query without
rejecting good ones? How did you verify that?

### Q10. Model comparison and the migration cost
**Build:** index the same corpus with two different embedding models. Run the same queries.
**Check:** compare result quality, vector dimensions, index size, and embedding time.
**Explain:** now suppose you must switch models in production with 10M documents indexed. Write
out what that migration involves.

### Q11. Asymmetric prefixes
**Build:** if your model documents query/passage prefixes, index and search with them and without.
**Check:** compare retrieval quality on the same queries.
**Explain:** report the difference. Why would getting this wrong be hard to notice?

### Q12. Clustering your corpus
**Build:** k-means over your embeddings, then print a few members of each cluster.
**Explain:** do the clusters correspond to real topics? What did this tell you about your corpus
that you didn't know?

### Q13. Embeddings as a cheap classifier
**Build:** label 100 items into 3 classes, embed them, train logistic regression, and evaluate.
Compare accuracy and cost against asking an LLM to classify the same held-out items.
**Explain:** report accuracy and cost per 1000 items for both. At what volume does the embedding
approach obviously win?

---

# Done when you can answer

1. How do text embeddings differ from token embeddings?
2. What kinds of similarity do embeddings capture — and what do they miss?
3. Why does the embedding model choice lock in your whole index?
4. Why can't you pick a similarity threshold without measuring?
5. Why does retrieval never return "nothing", and what must you do about it?
6. Where does keyword search beat embeddings?
7. When would you use embeddings instead of an LLM entirely?

Write answers in `notes.md`.
