# Topic 38 — RAG Evaluation

**Why this topic:** RAG has two stages that fail differently, and a single end-to-end score tells you
nothing about which one to fix. This topic is the diagnostic toolkit — and it's what turns Phase 5
from guesswork into engineering.

---

# Part 1 — Theory

## 38.1 Two stages, separate metrics

```
question -> [RETRIEVAL] -> context -> [GENERATION] -> answer
```

A wrong answer has three possible causes, and the fix is different for each:

| Retrieval | Generation | Diagnosis |
|---|---|---|
| correct chunks | wrong answer | generation problem — prompt, grounding, model |
| wrong chunks | wrong answer | retrieval problem — chunking, embeddings, search |
| wrong chunks | right answer | **the model answered from its own knowledge** — dangerous |

That third row is the one people miss: your system appears to work while your retrieval is broken,
until a question arrives that the model's training doesn't cover. So always measure both stages.

## 38.2 Retrieval metrics

You need a labelled set: for each question, which chunk(s) contain the answer.

- **Hit rate / recall@k** — was a relevant chunk in the top k? The first number to look at. If this
  is low, nothing downstream can save you.
- **MRR (mean reciprocal rank)** — 1/rank of the first relevant chunk, averaged. Rewards ranking it
  *high*, which matters because of lost-in-the-middle (Topic 19).
- **NDCG@k** — rank-weighted quality with graded relevance. Use when relevance isn't binary.
- **Precision@k / context precision** — what fraction of retrieved chunks were relevant. Low
  precision means wasted tokens and diluted attention.
- **Context recall** — of everything needed to answer, how much was retrieved? A question needing two
  facts is only answerable if both arrive.

The standard trade: raising k raises recall and lowers precision. Measure both, and choose k on
end-to-end outcome rather than on recall alone.

## 38.3 Generation metrics

- **Faithfulness / groundedness** — is every claim supported by the retrieved context?
  **This is the most important metric in RAG**, because it measures hallucination directly. Measured
  by decomposing the answer into claims and checking each against the context (an LLM judge does the
  checking).
- **Answer relevance** — does it address the question actually asked?
- **Correctness** — does it match the known answer? Needs ground truth.
- **Citation accuracy** — do the cited sources actually support the claims attributed to them? A
  plausible citation that doesn't support its claim is worse than none, because it manufactures
  false confidence.
- **Refusal appropriateness** — does it say "I don't know" exactly when the context lacks the answer?
  Measured on deliberately unanswerable questions.

## 38.4 Building the evaluation set

Per case you want: the question, the answer, the chunk ids that support it, and a label for whether
it's answerable at all.

How to get one:

- **From real usage** — best. Mine your logs for actual questions.
- **Synthetically** — have a model generate questions *from* a chunk, so the source is known by
  construction. Fast and cheap; systematically easier than real questions, because they're phrased
  using the chunk's own vocabulary. Useful, but don't mistake a high synthetic score for quality.
- **Hand-written** — expensive, best quality, essential for edge cases.

**Include unanswerable questions (~20%).** A system that always answers scores well on answerable
questions and hallucinates on the rest, and you will not notice without them.

## 38.5 Diagnosing with the matrix

The workflow that makes this topic practical: for every failure, classify it by §38.1's table, then
count.

Then act on where the mass is:

- Mostly retrieval failures → chunking, embeddings, hybrid search, reranking (Topics 22, 24).
- Mostly generation failures → grounding instructions, context ordering, model (Topics 19, 24).
- Mostly "right answer, wrong context" → your retrieval is worse than it looks; fix it before it
  bites.

## 38.6 Tools and their place

`ragas` (faithfulness, relevance, context precision/recall, with synthetic set generation),
`trulens`, `deepeval`, and platform-native evaluation tooling. They save time; they also hide what's
being computed. Implement faithfulness by hand once — claim decomposition and per-claim checking —
so you know what the number means and where it can be wrong.

## 38.7 Continuous evaluation in production

An offline score decays: documents change, users ask new things, the model gets updated. So:

- Log every query with retrieved chunk ids, scores, and the answer.
- Collect feedback (thumbs, corrections, whether the user rephrased — a rephrase is a strong negative
  signal).
- Run a **faithfulness check on a sample of live traffic** (it needs no ground truth, only the
  context and the answer — which makes it the one quality metric you can measure in production).
- Watch retrieval score distributions for drift; a falling mean top-score often means new questions
  your corpus doesn't cover.
- Feed every real failure into the evaluation set (Topic 37).

---

# Part 2 — Questions to implement

Build `rag_eval/` here, evaluating your Phase 5 system. Reuse Topic 37's harness.

### Q1. A labelled evaluation set
**Build:** 50 cases: question, answer, supporting chunk ids, answerable flag. Include 10
unanswerable ones.
**Check:** verify by hand that the labelled chunks really contain the answers.
**Explain:** how many of your own labels were wrong on inspection? What does that suggest about
trusting synthetic labels?

### Q2. Synthetic vs real questions
**Build:** generate 20 questions synthetically from chunks, and write 20 yourself in the voice of a
real user.
**Check:** measure retrieval hit rate on both sets.
**Explain:** report both numbers. Why is the synthetic set easier? What does that mean for reported
RAG benchmarks?

### Q3. Retrieval metrics
**Build:** hit rate@k, MRR, precision@k, context recall.
**Check:** report all of them at k = 1, 3, 5, 10, 20.
**Explain:** where does recall saturate? Where does precision collapse? Which k would you choose,
and on what basis?

### Q4. The diagnostic matrix
**Build:** for every failing case, classify it as retrieval failure, generation failure, or
right-answer-wrong-context.
**Check:** report the counts.
**Explain:** where is your mass? What will you fix first, and what would you have fixed if you'd
only seen end-to-end accuracy?

### Q5. The dangerous third case
**Build:** find cases where the answer is right but retrieval missed. Then ask those same questions
with retrieval *disabled*.
**Check:** how many does the model answer correctly with no context at all?
**Explain:** report the number. What does it mean for your measured "RAG accuracy"?

### Q6. Faithfulness by hand
**Build:** decompose answers into atomic claims, check each against the provided context, and score
the fraction supported.
**Check:** run on 30 answers; read 5 decompositions yourself.
**Explain:** report the faithfulness score. Show one unsupported claim you found.

### Q7. Validate your faithfulness judge
**Build:** hand-label 20 answers for groundedness and compare with your judge.
**Check:** report agreement.
**Explain:** where did it disagree? Is the metric trustworthy enough to gate releases?

### Q8. Citation verification
**Build:** check that each cited chunk actually supports the claim attributed to it.
**Check:** count correct citations, missing citations, and *wrong* citations.
**Explain:** report all three. Why is a wrong citation the worst of the three?

### Q9. Refusal on unanswerable questions
**Build:** measure behaviour on your 10 unanswerable cases, with and without grounding instructions.
**Check:** report hallucination rate for both.
**Explain:** report both numbers. What did the instruction change, and is the remaining rate
acceptable?

### Q10. Ablation with proper metrics
**Build:** evaluate naive, +hybrid, +rerank, +parent-child, +rewriting — with retrieval *and*
generation metrics for each.
**Check:** one table.
**Explain:** did any change improve retrieval while *hurting* faithfulness? What would end-to-end
accuracy alone have hidden?

### Q11. Compare with a framework
**Build:** run `ragas` (or similar) on the same data and compare with your hand-built metrics.
**Check:** do the numbers agree?
**Explain:** where they differ, which do you trust and why?

### Q12. Production monitoring
**Build:** log queries with retrieved ids and scores; run faithfulness on a 10% sample; track the
top-score distribution.
**Check:** simulate a week of traffic including questions your corpus can't answer.
**Explain:** did your monitoring detect the coverage gap? What signal fired first?

---

# Done when you can answer

1. Why must retrieval and generation be measured separately?
2. What are the three failure combinations, and which is most dangerous?
3. What do hit rate, MRR and context precision each tell you?
4. Why is faithfulness the most important RAG metric?
5. Why are synthetic evaluation questions systematically easier?
6. Why must unanswerable questions be in your set?
7. Which quality metric can you measure in production without ground truth, and why?

Write answers in `notes.md`.
