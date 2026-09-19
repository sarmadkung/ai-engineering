# Topic 22 — Document Processing

**Why this topic:** RAG quality is decided here, not in the clever retrieval logic. Garbage
chunks produce garbage answers no matter how good your model is — and this is the least glamorous,
most consequential topic in Phase 5.

---

# Part 1 — Theory

## 22.1 The pipeline

```
source file -> extract text -> clean -> chunk -> attach metadata -> embed -> store
```

Every stage can silently destroy information. The discipline of this topic is **looking at the
output of each stage** rather than trusting it.

## 22.2 PDF extraction, which is genuinely hard

A PDF describes where glyphs are painted on a page. It has no concept of a paragraph, a reading
order, or a table. Extraction is reconstruction, and it fails in predictable ways:

- **Multi-column layouts** interleave into nonsense if read left-to-right across the page.
- **Tables** collapse into space-separated soup where the row/column relationship — the entire
  meaning — is gone.
- **Headers, footers and page numbers** appear in the middle of your text every ~500 words.
- **Scanned PDFs** contain no text at all, only images. You need OCR (Phase 16) and must detect
  this case rather than silently indexing empty strings.
- **Ligatures, hyphenation and soft breaks** produce "ﬁ" and words split across lines.

Tools, roughly in order of sophistication: `pypdf` (fast, crude), `pdfplumber` (layout and table
awareness), `PyMuPDF` (fast and good), `unstructured` (heuristic document structure), then
layout-aware ML models or a vision model for the hard cases.

**Always eyeball the extracted text of a few documents.** Most bad RAG systems have never had
their extraction output read by a human.

## 22.3 HTML

The problem is the opposite: too much structure, mostly irrelevant. Navigation, cookie banners,
sidebars, related-article lists and footers can be 80% of a page's text and will pollute your
index.

Approach: extract the main content (`trafilatura`, `readability`), keep the semantic tags that
carry meaning (headings, lists, table structure), convert to Markdown rather than plain text so
structure survives into the chunk, and drop `script`/`style`/`nav`/`footer` outright.

## 22.4 Markdown — the useful intermediate

Convert everything to Markdown early. Headings give you a document hierarchy you can chunk along
and record as metadata; lists and tables keep their shape; it's compact (few tokens of overhead);
and LLMs read it natively.

## 22.5 Chunking — the decision that matters most

Why chunk at all? Embedding models have input limits, retrieval needs focused units, and you want
to fill the context with relevant text rather than whole documents.

Strategies, worst to best:

- **Fixed size by characters** — trivial, and splits mid-sentence and mid-word. Only acceptable as
  a baseline.
- **Fixed size with overlap** — each chunk repeats the last ~10–20% of the previous one, so a fact
  straddling a boundary survives in at least one chunk. Cheap insurance; the standard default.
- **Recursive character splitting** — try to split on paragraph breaks, then sentences, then words,
  as needed to fit the size. A good default (this is what LangChain's recommended splitter does).
- **Structure-aware** — split on Markdown headings, code blocks, or document sections, so chunks
  align with the author's own units of meaning. Best quality when the structure exists.
- **Semantic chunking** — split where embedding similarity between consecutive sentences drops,
  i.e. where the topic changes. Expensive, sometimes better.

**Size** is a real trade-off: small chunks retrieve precisely but lack context to answer from;
large chunks carry context but dilute the embedding (one vector for many ideas) and waste context
window. 300–800 tokens is the usual range. You should decide it by measurement, not by copying a
tutorial.

Two failure modes to recognize: a chunk that answers nothing because the subject was named two
paragraphs earlier ("It costs $40" — what does?), and a chunk so broad its embedding matches
nothing specifically.

## 22.6 Metadata

Every chunk should carry: source (file, URL), position (page, section, heading path), timestamps,
document type, access permissions, and the id of the parent document.

Metadata is not bookkeeping — it's functionality:

- **Citations** require it, and citations are what make RAG trustworthy.
- **Filtering** requires it (Topic 23): "search only this customer's documents" is an access
  control requirement, not a feature.
- **Freshness** requires it (prefer the newest of two conflicting chunks).
- **Debugging** requires it — when an answer is wrong you need to find which chunk caused it.

Prepending the heading path to the chunk *text* (`"Billing > Refunds > Timeline: ..."`) is a cheap
trick that improves both retrieval and answer quality, because it restores the context the chunk
lost.

## 22.7 Cleaning and deduplication

Normalize whitespace and Unicode; strip repeated headers/footers; drop boilerplate; remove chunks
that are pure navigation or too short to mean anything; and **deduplicate**, because the same
paragraph appearing in 40 documents will occupy your entire top-k and crowd out everything else.

Phase 3's data-quality lessons apply directly — this is the same job at application scale.

## 22.8 Ingestion as a system

Real ingestion is a pipeline with operational requirements: **incremental** (re-index only what
changed, detected by content hash), **idempotent** (re-running produces no duplicates),
**observable** (counts per stage, so a drop is visible), **resumable** (a failure at document
9,000 of 10,000 must not restart from zero), and **versioned** (record which extractor, chunker
and embedding model produced each chunk, so you can migrate).

That last one is what makes changing your chunking strategy possible later without re-ingesting
blind.

---

# Part 2 — Questions to implement

Build `ingest.py` here. Get a genuinely messy corpus: 20+ real PDFs (papers, manuals, invoices),
some HTML pages, and at least one scanned PDF.

### Q1. Extract, and actually read the output
**Build:** extract text from 5 PDFs with two different libraries. Write both outputs to files.
**Check:** read them yourself, side by side.
**Explain:** list every defect you found (column interleaving, tables, headers, hyphenation).
Which library was better, and for which document type?

### Q2. Detect the unextractable
**Build:** a check that flags a PDF as needing OCR (e.g. almost no text extracted relative to page
count).
**Check:** your scanned PDF is flagged; normal ones are not.
**Explain:** what would have happened to that document without this check?

### Q3. Table damage
**Build:** extract a page containing a table, plainly and with a layout-aware tool.
**Explain:** paste both results in your notes. Can a model answer "what was the Q3 figure for
product B" from either? What would you have to do to make tables usable?

### Q4. HTML main-content extraction
**Build:** extract text from 5 web pages naively (all text) and with a main-content extractor.
**Check:** count characters for each.
**Explain:** what fraction was boilerplate? What would that boilerplate have done to your index?

### Q5. Markdown conversion
**Build:** convert HTML and PDF output to Markdown, preserving headings.
**Check:** the heading hierarchy is recoverable from the output.
**Explain:** why is preserving headings worth the effort?

### Q6. Four chunkers
**Build:** fixed-size, fixed-size-with-overlap, recursive, and heading-aware chunkers.
**Check:** run all four on the same document; print the first 3 chunks of each.
**Explain:** which produced chunks you'd be happy to hand to a model? Show one chunk that is
broken by the naive splitter and intact under the better one.

### Q7. The straddling fact
**Build:** find or construct a fact that lands on a chunk boundary. Retrieve it (using Topic 21's
search) with and without overlap.
**Check:** record whether the fact is retrievable in each case.
**Explain:** what overlap fraction did you need? What does overlap cost?

### Q8. Chunk size sweep
**Build:** index at 150, 400, 800, and 1500 tokens. Write 15 questions with known answers and
measure retrieval success for each size.
**Check:** tabulate hit rate per size.
**Explain:** which size won, and why did the extremes fail? Give the failure mode of each.

### Q9. The orphaned pronoun
**Build:** find a retrieved chunk whose meaning depends on text outside it ("it", "this
version", "the above").
**Explain:** show it. Then implement heading-path prepending and show the improved chunk. Did
retrieval improve too?

### Q10. Metadata and citations
**Build:** attach source, page, heading path, and hash to every chunk. Make your search return
citations with results.
**Check:** every answer can be traced to a file and page you can open and verify.
**Explain:** why is an uncitable RAG answer a product problem, not just a technical one?

### Q11. Deduplication
**Build:** detect near-duplicate chunks across your corpus and report the duplicate rate.
**Check:** print one duplicate group.
**Explain:** run a query where duplicates crowd the top-5 results. What did the user lose?

### Q12. Incremental, idempotent ingestion
**Build:** content-hash based ingestion: unchanged documents are skipped, changed ones are
re-chunked and their old chunks deleted.
**Check:** run twice — the second run does almost nothing and creates no duplicates. Modify one
file and confirm only it is reprocessed.
**Explain:** what would a non-idempotent pipeline do to your index after three runs?

### Q13. Pipeline observability
**Build:** report counts at each stage (documents in, extracted, chunks created, embedded, stored)
plus failures with reasons.
**Check:** deliberately corrupt one file and confirm the pipeline continues and reports it.
**Explain:** which stage lost the most content, and was that loss correct?

---

# Done when you can answer

1. Why is PDF extraction hard, and what are its typical failures?
2. Why convert everything to Markdown?
3. What is the trade-off between small and large chunks?
4. Why does overlap exist, and what does it cost?
5. Why is metadata functional rather than decorative?
6. Why must ingestion be incremental and idempotent?
7. Why is RAG quality mostly decided in this topic rather than in retrieval?

Write answers in `notes.md`.
