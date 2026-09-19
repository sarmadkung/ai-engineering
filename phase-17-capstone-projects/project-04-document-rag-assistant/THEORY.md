# Project 4 — PDF / Document RAG Assistant

**Depends on:** Phase 5 (Topics 21–25), Phase 9 (Topics 37–38), Topic 60 for hard documents.

**What you prove:** that you can build retrieval that actually works, and that you can *prove* it works
with numbers. This is the most commercially common AI application there is — and the one most often
built badly.

---

# Part 1 — What you are building

Upload documents, ask questions, get answers with citations you can verify.

```
upload -> extract -> chunk -> embed -> store (pgvector)
question -> rewrite -> hybrid retrieve -> rerank -> answer with citations
```

The difference between this and a weekend demo is measurement: you will have a labelled evaluation set
and separate retrieval and generation metrics, and every improvement will be justified by numbers.

---

# Part 2 — Design decisions

| Decision | Options | Consider |
|---|---|---|
| Extraction | pypdf / pdfplumber / PyMuPDF / VLM | quality versus cost per page |
| Chunking | fixed+overlap / recursive / heading-aware / parent-child | the biggest quality lever |
| Chunk size | 200 / 400 / 800 tokens | measure it, don't copy a tutorial |
| Embeddings | hosted / local | cost, privacy, and it locks in your index |
| Store | pgvector / Chroma / dedicated | pgvector unless you have a measured reason |
| Retrieval | vector / hybrid / hybrid+rerank | hybrid is usually the biggest win |
| Answer style | strict grounding / allow model knowledge | decide and test it |

---

# Part 3 — Milestones

### M1 — Evaluation set first (Topics 37, 38)
**Before building retrieval:** 50 questions over your corpus with expected answers and the chunk(s)
that support them. Include 10 that are unanswerable.
**Check:** a harness reporting retrieval metrics (hit rate@k, MRR) and generation metrics (correctness,
faithfulness) **separately**.
**Explain:** why this comes first — everything after is measured against it.

### M2 — Ingestion (Topic 22)
Extract, clean, chunk with metadata (source, page, heading path), embed, store. Incremental and
idempotent by content hash. OCR-needed detection.
**Check:** re-running changes nothing. A scanned PDF is flagged rather than silently indexed as empty.

### M3 — Naive RAG baseline (Topic 24)
Embed question → top-5 → answer.
**Check:** report hit rate and accuracy. Classify 10 failures by cause (retrieval / generation /
answered-from-training).

### M4 — Grounding and citations (Topics 24, 43)
Answer only from context, cite chunk ids, say "I don't know" when the context lacks the answer. Verify
citations actually support their claims.
**Check:** hallucination rate on your 10 unanswerable questions, before and after. Report both.

### M5 — Hybrid search (Topic 24)
BM25 plus vector, merged with reciprocal rank fusion.
**Check:** hit rate for vector-only, BM25-only, and hybrid. Show one query each method wins.

### M6 — Reranking (Topic 24)
Retrieve 50, rerank to 5 with a cross-encoder.
**Check:** hit rate@5 before and after, plus added latency. Was it worth it?

### M7 — Query rewriting (Topic 24)
Rewrite follow-up questions into standalone queries using conversation history.
**Check:** hit rate on pronoun follow-ups with and without rewriting. This is where conversational RAG
usually collapses — show the gap.

### M8 — Parent-child retrieval (Topic 25)
Index small chunks, return larger parents.
**Check:** hit rate and answer accuracy against M6. Show one question that only worked with expansion.

### M9 — Hard documents (Topic 60)
A cascade: text extraction first, detect failure, escalate those pages to a vision model for tables and
scans.
**Check:** accuracy on your worst documents, and cost per page for the cascade versus vision-for-all.

### M10 — Product (Topics 20, 47)
Upload UI, streaming answers, clickable citations that open the source page, multi-user isolation, and
per-user document scoping enforced at the storage layer.
**Check:** a test attempting cross-user retrieval fails. Every answer's citation opens to a page where
you can see the claim.

---

# Part 4 — The ablation that matters

One table, every configuration measured on the same evaluation set:

| Config | hit rate@5 | faithfulness | correctness | latency | cost/query |
|---|---|---|---|---|---|
| naive | | | | | |
| + grounding | | | | | |
| + hybrid | | | | | |
| + rerank | | | | | |
| + rewriting | | | | | |
| + parent-child | | | | | |

Then answer honestly: which stages earned their place, and which fashionable technique did nothing on
your corpus?

Also run: chunk-size sweep (4 sizes), k sweep, position effects (Topic 19), and synthetic versus
real questions (Topic 38).

---

# Part 5 — Deliverables

- Working application with upload, chat, and verifiable citations
- `EVALUATION.md` — the labelled set, methodology, and the full ablation table
- `ARCHITECTURE.md` — pipeline and decisions
- `RESULTS.md` — final metrics, cost per query, ingestion cost per document
- The evaluation harness, runnable with one command

---

# Part 6 — Done when

- Every configuration change you shipped is justified by a number.
- Hallucination rate on unanswerable questions is measured and low.
- Every citation can be opened and checked by a human.
- Retrieval and generation metrics are reported separately.
- Cross-user document leakage is impossible and you have a test proving it.

---

# Stretch

Graph RAG for multi-hop questions (Topic 25); agentic retrieval where the model decides when to search
(Topic 25); table-aware extraction into queryable structured data; production faithfulness monitoring on
live traffic (Topic 40); multilingual documents.
