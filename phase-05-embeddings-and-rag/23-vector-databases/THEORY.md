# Topic 23 — Vector Databases

**Why this topic:** exact search over a million vectors is too slow. This topic is how vector
search actually scales, and how to choose and operate the storage layer without being sold to.

---

# Part 1 — Theory

## 23.1 Why a special database

Your Topic 21 search compares the query against **every** vector. That's exact and simple, and it
costs O(n) per query. At 10M vectors of 1536 dimensions that's ~60 GB of arithmetic per query —
hopeless.

A vector database provides: an **approximate nearest neighbour (ANN) index** for sub-linear
search, **metadata filtering** alongside vector search, persistence, and the usual operational
machinery (updates, deletes, backups, concurrency).

**The core trade:** ANN gives up exactness for speed. You get "almost always the right top-k",
measured as **recall** — the fraction of true nearest neighbours your index actually returned. You
choose where to sit on the recall/speed curve, and you should know your number.

## 23.2 The index types

**Flat (brute force)** — no index, exact, O(n). The right choice below ~10k–100k vectors, and it's
the baseline you measure recall against. Don't skip it out of ambition.

**IVF (inverted file)** — cluster the vectors, then search only the nearest few clusters. Fast and
memory-light; misses neighbours that sit just across a cluster boundary. `nprobe` (clusters
searched) is the recall/speed dial.

**HNSW (hierarchical navigable small world)** — a multi-layer graph you walk, coarse layer to fine.
Excellent recall and speed, high memory use, slower to build. The most common default.
`M` (connections per node) and `ef_construction` set build quality; `ef_search` is the query-time
recall/speed dial.

**Quantization** — compress vectors to shrink memory: **scalar/int8** (~4× smaller, small recall
cost), **product quantization** (much smaller, larger recall cost), **binary** (32× smaller, used
as a fast first pass then re-ranked exactly). Combined with HNSW in most production systems.

Know the pattern: **coarse-and-fast first, exact re-scoring second**. It recurs everywhere in
retrieval, including Topic 24's reranking.

## 23.3 The options

**pgvector** — vectors as a Postgres column type, with IVF and HNSW indexes.
*The right default for most applications*, because your vectors live beside your relational data:
one database, real transactions, real joins, arbitrary SQL filtering, one backup story, no sync
problem. Scales to millions of vectors comfortably.

**Chroma** — embedded, Python-native, trivial to start. Excellent for learning and prototyping;
limited for production scale.

**Qdrant / Weaviate / Milvus** — purpose-built vector databases (self-hostable). Stronger filtering
and index tuning, better at very large scale, at the cost of running another system and keeping it
in sync with your source of truth.

**Pinecone** — fully managed. No ops, per-vector pricing, your data leaves your infrastructure.

**FAISS** — a library, not a database. Fast indexes with no persistence, filtering, or API.

**How to choose, honestly:** start with pgvector if you already have Postgres, or Chroma to learn.
Move to a dedicated store when you have a measured reason — scale, filtering performance, or a
specific index feature. "It's a vector database" is not an architecture.

## 23.4 Metadata filtering, and why it's the hard part

You rarely want "nearest neighbours in the whole corpus". You want them **within a subset**: this
user's documents, this date range, this language, documents this person may read.

Two implementations, and the difference matters:

- **Post-filtering** — retrieve top-k by vector, then discard those failing the filter. Simple,
  and catastrophic with selective filters: if only 0.1% of documents match, your top-100 may
  contain zero of them and you return nothing.
- **Pre-filtering** — restrict the search to matching vectors. Correct, but hard to do efficiently
  inside a graph index, since the graph's connectivity assumes all nodes are present.

This is the strongest technical argument for pgvector-style storage: SQL `WHERE` clauses compose
naturally with the vector search, and a selective filter can use a normal index.

**Access control is a filter.** Multi-tenant retrieval where the filter is advisory is a data
breach waiting to happen — a query that leaks another customer's chunk is a security incident, not
a relevance bug.

## 23.5 Operating the thing

Realities that don't appear in the marketing:

- **Index builds are expensive.** HNSW over millions of vectors takes time and a lot of memory.
- **Updates and deletes degrade an index.** Deletions are often tombstones, and graph quality drifts
  after heavy churn; periodic rebuilds are normal.
- **Re-embedding is a full reindex** (Topic 21). Plan for dual-write or a versioned collection so
  you can migrate without downtime.
- **Consistency**: if your vectors live outside your main database, a failed write leaves them
  disagreeing. A chunk deleted from Postgres but still in the vector store will keep being
  retrieved, and will happily cite a document you deleted for legal reasons.
- **Measure recall on your own data.** Vendor benchmarks use datasets you don't have.

---

# Part 2 — Questions to implement

Build `store.py` with a common interface and at least two backends. You need a corpus of at least
50k chunks to see real behaviour — synthesize by duplicating with variation if needed. Install
`pgvector` (with Docker Postgres) and `chromadb`.

### Q1. Exact baseline
**Build:** brute-force search over your vectors in plain Python/NumPy, and record its results as
ground truth. Measure query latency at 1k, 10k, 100k vectors.
**Check:** latency grows linearly.
**Explain:** extrapolate to 10M. What's your per-query latency, and is that shippable?

### Q2. A common interface
**Build:** `add(chunks)`, `search(query, k, filters)`, `delete(ids)`, `count()` — implemented over
Chroma and pgvector.
**Check:** the same test suite passes against both.
**Explain:** which operation was most awkward to make uniform, and why?

### Q3. Measure recall
**Build:** using Q1's exact results as ground truth, compute recall@10 for an ANN index over 100
queries.
**Check:** report a number, not an impression.
**Explain:** what recall did you get at default settings? Is that acceptable for your use case?

### Q4. The recall/speed dial
**Build:** sweep `ef_search` (HNSW) or `nprobe` (IVF) across a wide range, recording recall@10 and
latency at each point.
**Check:** plot or tabulate the curve.
**Explain:** where's the knee? Which setting would you ship, and what recall are you accepting?

### Q5. HNSW vs IVF vs flat
**Build:** all three over the same 100k vectors.
**Check:** tabulate build time, memory, query latency, recall@10.
**Explain:** which would you choose for 100k vectors? For 10M? For 5k?

### Q6. Quantization
**Build:** an index with int8 or binary quantization.
**Check:** report memory saved, recall lost, and latency change.
**Explain:** was the trade worth it? How would a re-scoring pass change your answer?

### Q7. Post-filter failure
**Build:** deliberately use post-filtering with a filter matching ~0.1% of documents.
**Check:** count how often you return fewer than k results, or none.
**Explain:** show a query that returns nothing despite matching documents existing. Why does this
happen?

### Q8. Pre-filtering
**Build:** the same filtered query with pgvector and a SQL `WHERE` clause.
**Check:** the correct results are returned; compare latency to the unfiltered query.
**Explain:** why can SQL do this well? What did the dedicated vector index struggle with?

### Q9. Tenant isolation
**Build:** multi-tenant data with a `tenant_id`, and search that enforces it at the storage layer —
not in application code after the fact.
**Check:** write a test that *tries* to leak across tenants and fails to.
**Explain:** why must this be enforced below the application, and what's the consequence of getting
it wrong?

### Q10. Churn
**Build:** delete 30% of your vectors and insert 30% new ones.
**Check:** measure recall and latency before and after; then rebuild the index and measure again.
**Explain:** did quality degrade? What is your operational plan for this?

### Q11. Consistency failure
**Build:** deliberately delete a chunk from Postgres but not from the vector store, then query for
it.
**Check:** the deleted content is returned.
**Explain:** describe the real-world incident this represents. How does single-database storage
prevent it?

### Q12. Re-embedding migration
**Build:** migrate your index to a different embedding model with no downtime (versioned
collections or dual-write plus cutover).
**Check:** searches keep working throughout; afterwards all vectors come from the new model.
**Explain:** write the runbook you'd hand a colleague.

### Q13. Decide with numbers
**Build:** a short comparison document: for your actual corpus size and query volume, tabulate
latency, recall, memory, operational burden and cost for pgvector vs one dedicated store.
**Explain:** which do you choose, and what measured fact would change your mind?

---

# Done when you can answer

1. Why is exact search unusable at scale, and what does ANN trade away?
2. What is recall, and why must you measure it on your own data?
3. How do HNSW and IVF differ, and what's each one's dial?
4. Why is pgvector the sensible default for most applications?
5. Why does post-filtering break with selective filters?
6. Why is access control a storage-layer concern?
7. What operational work does a vector index need over its life?

Write answers in `notes.md`.
