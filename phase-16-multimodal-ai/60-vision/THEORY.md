# Topic 60 — Vision

**Why this topic:** most real information isn't text — it's screenshots, scans, diagrams, photos and
PDFs. Vision-language models let your systems read them, and they unlock the document problems that
defeated Topic 22's text extraction.

---

# Part 1 — Theory

## 60.1 How a vision-language model works

The architecture is simpler than it sounds, and it's Phase 1's machinery reused:

```
image -> vision encoder -> patch embeddings -> projection into the LLM's embedding space
                                            -> treated as tokens alongside text tokens
```

An image is cut into patches (say 14×14 pixels), each patch becomes a vector, and those vectors are
projected so the language model can attend over them exactly as it does over text (Topic 4). There is
no separate "image reasoning" module — it's one transformer attending over a mixed sequence.

Consequences that matter in practice:

- **Images cost tokens.** A high-resolution image can be a thousand tokens or more, and you pay for them
  like any input. Resolution is a cost dial.
- **Resolution is limited.** Images are downscaled to a supported size, so fine detail — small text, thin
  lines — can be lost before the model ever sees it. Many models handle this by tiling a large image
  into several crops, which multiplies tokens.
- **Text and image tokens are interchangeable to the model**, which is why it can reason about a diagram
  and a paragraph together.

## 60.2 What they're good at, and what they aren't

**Reliable:** describing a scene, reading clear text in an image (OCR-by-model), understanding charts
and diagrams, answering questions about a screenshot, extracting fields from a form, judging layout and
UI, and comparing two images.

**Unreliable:** precise spatial relations and exact coordinates, counting more than a few objects,
reading small or low-contrast text, fine-grained measurement, and anything requiring pixel-level
precision. Also: they will confidently describe things that aren't there, in the same way they
hallucinate text (Topic 43), and a plausible-sounding description of a chart with invented numbers is a
real and dangerous failure.

The rule: **use vision for understanding, not for measurement.** If a number's exact value matters, get
it from the source data or verify it.

## 60.3 Image understanding in applications

Common real uses: document processing (the big one), UI/screenshot analysis for agents and testing,
visual QA in support flows, content moderation, accessibility descriptions, and chart/diagram
interpretation.

Prompting them well is mostly the same discipline as text (Topic 17), with additions: ask for a
structured output (Topic 18) rather than prose; tell the model what to look for rather than asking it
to describe everything; and when accuracy matters, ask it to quote the text it is reading so you can
verify it.

## 60.4 OCR: model or engine?

Two approaches, and the choice matters:

- **Traditional OCR** (Tesseract, cloud OCR services) — returns text plus **bounding boxes and
  confidence scores**, is cheap, fast, deterministic, and excellent on clean printed text. It does not
  understand the document.
- **Vision-language models** — understand layout, tables, handwriting and context; can output structured
  data directly; cost more, are slower, are non-deterministic, and give you no coordinates or
  confidence.

The practical answer is often **both**: OCR for a cheap, verifiable text layer, and a VLM for structure
and semantics — with OCR output used to check the model's reading. This is the same generate-and-verify
pattern as everywhere else in this roadmap.

## 60.5 Documents: the payoff

Topic 22's hardest failures — multi-column layout, tables collapsing into soup, scanned pages — are
largely solved by feeding the *page image* to a VLM and asking for structured output. A table becomes
JSON; a form becomes fields; a scanned invoice becomes records.

The cost is real: per-page model calls versus a text extraction that's nearly free. The sensible
architecture is a cascade — cheap text extraction first, detect when it has failed (Topic 22's OCR
check), and escalate only those pages to vision.

## 60.6 Vision embeddings

Models like CLIP embed images and text into a **shared** space, so you can search images with text
queries and vice versa (Phase 5's machinery applied to pixels).

Uses: image search over a corpus, zero-shot classification (embed the candidate labels as text, compare),
deduplication, and multimodal RAG — retrieving relevant figures and diagrams alongside text passages,
which is genuinely useful for technical documentation.

Same caveats as text embeddings (Topic 21): calibrate your thresholds, and remember similarity is not
relevance.

## 60.7 Engineering concerns

- **Cost** — measure tokens per image at each resolution and decide deliberately. Downscaling before
  sending is the cheapest optimization available.
- **Caching** — image inputs are large and often repeated; cache by content hash.
- **Privacy** — images contain faces, documents, screens full of personal data. The same obligations as
  text logging (Topic 40), with higher stakes.
- **Untrusted content** — text inside an image can carry prompt injection, and it bypasses text-based
  input filters entirely. A screenshot containing "ignore your instructions" is a real attack
  (Topic 41).
- **Verification** — for extracted data that matters, verify against a second source or a deterministic
  check.

---

# Part 2 — Questions to implement

Build `vision/` here. Use a current vision-capable API model. You need real material: photos, screenshots,
scanned documents, charts, forms, and a multi-column PDF.

### Q1. First calls and token cost
**Build:** send one image with a question. Then send the same image at three resolutions.
**Check:** report input tokens and cost for each.
**Explain:** how many tokens is an image worth in your setup? What's the cheapest resolution that still
answers correctly?

### Q2. Structured extraction from an image
**Build:** extract fields from 10 scanned forms or invoices into a validated schema (Topic 18).
**Check:** report per-field accuracy against hand-labelled ground truth.
**Explain:** which field was least reliable, and why?

### Q3. The table that defeated Topic 22
**Build:** take the PDF page whose table collapsed in Topic 22's text extraction. Send the page image and
ask for the table as JSON.
**Check:** compare against the actual table.
**Explain:** did it work? Report cell-level accuracy. What does this change about your document pipeline?

### Q4. Multi-column layout
**Build:** the multi-column page that interleaved into nonsense in Topic 22.
**Check:** compare VLM reading order against text extraction.
**Explain:** report both. What does this cost per page, and when is it worth it?

### Q5. Hallucinated detail
**Build:** ask for detailed descriptions of 10 images, including a few that are ambiguous, low-quality, or
nearly empty.
**Check:** verify every claim against the image yourself.
**Explain:** how many invented details did you find? What kind of image provoked them?

### Q6. Chart reading, verified
**Build:** ask for the values in 10 charts where you know the underlying data.
**Check:** report numeric accuracy.
**Explain:** how often were numbers approximately right but not exact? What does this mean for using
chart reading in anything that computes?

### Q7. Counting and spatial relations
**Build:** counting tasks (5, 12, 30 objects) and spatial questions (left/right, above, overlapping).
**Check:** report accuracy per category.
**Explain:** where did it break down? Restate §60.2's rule in your own words with your data behind it.

### Q8. Small text
**Build:** the same document image at decreasing resolutions, and with small-font regions.
**Check:** find the point where text becomes unreadable to the model.
**Explain:** report the threshold. How would a pipeline detect this failure rather than accepting wrong
text?

### Q9. OCR engine vs model
**Build:** run 10 documents through traditional OCR and through a VLM.
**Check:** compare accuracy, cost, latency, and what each returns (boxes? confidence? structure?).
**Explain:** report the table. For which documents did each win?

### Q10. Cascade pipeline
**Build:** text extraction first, a failure detector (Topic 22's OCR check plus a quality heuristic), and
vision escalation only for failures.
**Check:** measure accuracy and cost per page against vision-for-everything.
**Explain:** report both. What fraction of pages needed escalation, and what did the cascade save?

### Q11. Cross-verification
**Build:** for extracted numeric data, verify VLM output against OCR output and flag disagreements.
**Check:** report the disagreement rate and, when they disagree, who was right.
**Explain:** is this a usable confidence signal?

### Q12. Vision embeddings and multimodal retrieval
**Build:** embed a set of images with CLIP-style embeddings; search them with text queries. Then extend
your Phase 5 RAG to retrieve relevant figures alongside text.
**Check:** measure retrieval quality on 10 queries.
**Explain:** did retrieved figures improve answers? For what kind of question?

### Q13. Injection through an image
**Build:** an image containing text instructing the model to ignore its instructions. Send it through your
normal pipeline.
**Check:** does it work? Do your text-based input filters see it at all?
**Explain:** report both. What defence would catch this? (Topic 41.)

### Q14. A real document pipeline
**Build:** end to end: ingest mixed documents, cascade extraction, structured output, validation,
citations back to page images, and cost tracking.
**Check:** run 50 pages of varied documents; report accuracy, cost per page, and failure rate.
**Explain:** compare against your Phase 5 text-only pipeline's numbers. Was vision worth it, and for
which document types?

---

# Done when you can answer

1. How does an image become tokens a language model can attend over?
2. Why are images expensive, and what's the resolution trade-off?
3. What are VLMs reliable at, and what should you never trust them for?
4. When do you use traditional OCR instead of (or alongside) a model?
5. Why is a cascade better than sending every page to vision?
6. What do vision embeddings enable?
7. Why does an image bypass your text input filters?

Write answers in `notes.md`.
