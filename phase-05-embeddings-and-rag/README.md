# Phase 5 — Embeddings & RAG

**Goal:** make a model answer from *your* data, with citations, and prove it works. RAG is the most
common architecture in applied AI — and the one most often built badly, because most people skip the
measurement.

---

## What I will do

| # | Topic | What I build | Status |
|---|---|---|---|
| 21 | [Embeddings](21-embeddings/) | `embed.py`, `search.py` — semantic search from scratch | ready |
| 22 | [Document Processing](22-document-processing/) | `ingest.py` — extraction, chunking, metadata, incremental ingest | ready |
| 23 | [Vector Databases](23-vector-databases/) | `store.py` — two backends behind one interface, with measured recall | ready |
| 24 | [RAG](24-rag/) | `rag.py` — hybrid search, reranking, query rewriting | ready |
| 25 | [Advanced RAG](25-advanced-rag/) | parent-child, multi-vector, graph and agentic retrieval | ready |

## Which things to learn

**21. Embeddings** — text embeddings versus token embeddings; what similarity captures and what it misses
(negation, identifiers); choosing a model, and why it locks in your index; cosine similarity and
calibrating thresholds on your own corpus; why retrieval never returns "nothing"; where keyword search
wins.

**22. Document Processing** — why PDF extraction is genuinely hard; HTML boilerplate; Markdown as the
useful intermediate; chunking strategies and the size trade-off; overlap; metadata as functionality
(citations, filtering, freshness); deduplication; incremental, idempotent, observable ingestion.

**23. Vector Databases** — why exact search doesn't scale; ANN indexes (flat, IVF, HNSW) and measuring
**recall**; quantization; pgvector as the sensible default; pre- versus post-filtering; access control as
a storage-layer concern; the operational life of an index.

**24. RAG** — what RAG does that fine-tuning can't; the five failure points of the naive pipeline;
context construction and ordering; hybrid search with reciprocal rank fusion (usually the biggest win);
cross-encoder reranking; query rewriting for conversations; HyDE and multi-query.

**25. Advanced RAG** — the small-chunk dilemma and parent-child retrieval; multi-vector and hypothetical
questions; contextual retrieval; Graph RAG for multi-hop; agentic RAG; and RAG evaluation — retrieval and
generation measured separately, with faithfulness as the key metric.

## Prerequisites

Phase 4. An embedding model (`sentence-transformers` locally, or an API), Postgres with pgvector,
`chromadb`, and a real corpus — messy PDFs and HTML, including at least one scanned document.

**Build your evaluation set before your pipeline.** Topic 24 starts with it for a reason.

**Next:** Phase 6 — letting the model act, not just read.
