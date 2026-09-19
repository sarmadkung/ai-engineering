# Topic 37 — LLM Evaluation

**Why this topic:** without evaluation you are guessing. Every improvement you've made so far — a
prompt change, a retrieval tweak, a model swap — was either measured or believed. This topic makes it
measured, and it's the skill that most distinguishes senior AI engineers.

---

# Part 1 — Theory

## 37.1 Why this is hard

Normal software: one input, one correct output, `assert`. LLM software:

- **Many valid outputs.** There are a thousand good summaries of a paragraph.
- **Non-deterministic.** The same input gives different outputs at temperature > 0.
- **Graded quality**, not pass/fail.
- **Subjective** dimensions — tone, helpfulness, appropriateness.
- **Silent regressions.** A prompt change fixes one case and breaks four others, and nothing throws.

So evaluation is **statistical**, not assertive: measure a distribution over a set, compare
distributions, and treat single examples as anecdotes.

## 37.2 Golden datasets

Your test set. Properties that matter:

- **Real inputs.** Sampled from actual usage where possible. Invented test cases are systematically
  easier and more polite than real ones, so they flatter your system.
- **Representative distribution**, including the boring majority, not just interesting edge cases.
- **Edge cases and failures deliberately included** — ambiguous input, adversarial input, and inputs
  your system *should refuse or decline*.
- **Expected outputs or acceptance criteria.** Sometimes an exact answer, sometimes a rubric,
  sometimes required/forbidden elements.
- **Versioned in git**, like code.
- **Train/validation/test discipline.** Iterate against dev; keep a test slice untouched. Phase 2's
  overfitting lesson applies exactly — iterating against one set until it passes is overfitting by
  hand.

Size: 20 cases is enough to catch gross regressions; 50–200 for confident comparisons; more for
subtle differences. Start small and grow it from **real failures** — every production bug becomes a
permanent test case. That habit is worth more than a large synthetic set.

## 37.3 Automated evaluation

**Deterministic checks** (cheap, reliable — use wherever possible): exact match, valid JSON, schema
validation, required substrings, forbidden substrings, numeric tolerance, latency and token budgets,
and structural checks (did it cite a source? is the code parseable? do the tests pass?).

**Similarity metrics**: embedding similarity to a reference (Phase 5), and older overlap metrics
(BLEU/ROUGE) which are weak for open-ended generation.

**LLM as judge** — for the open-ended dimensions. Pass the input, the output, and a rubric to a
model and get a score. What makes a judge usable:

- **A specific rubric** with defined levels, not "rate 1–10".
- **Reasoning before the score** (Topic 17), which improves the score and lets you audit it.
- **Pairwise comparison** where possible — "which is better, A or B?" is far more reliable than
  absolute scoring, and is what arena-style evaluation uses.
- **Validation against human labels.** A judge you haven't checked against your own judgement is an
  unvalidated instrument. Measure agreement, and know the number.

Known judge biases: verbosity (longer is rated better), position (order in a pairwise comparison
affects the verdict — mitigate by running both orders), self-preference (a model may favour its own
style), and leniency. Use a strong model as judge, and never judge with the same call that produced
the output.

## 37.4 Human evaluation

The ground truth, and expensive. Use it to: validate your automated metrics, judge what automation
can't, and settle disagreements.

Practices that make it worthwhile: written guidelines (which *define* quality — vague guidelines
produce noise), multiple raters with measured agreement, pairwise comparison, and blind
presentation. If raters agree only 70% of the time, no automated metric can meaningfully exceed 70%
— that's your ceiling, and it's worth knowing before you chase a point of accuracy.

## 37.5 Benchmarking and its limits

Public benchmarks (MMLU, GSM8K, HumanEval, MT-Bench, and their successors) are useful for coarse
model selection and useless for your application. Reasons: **contamination** (benchmarks leak into
training data), a distribution unlike your users', saturation, and gaming.

**Your evaluation set is the only benchmark that matters for your product.** Use public ones to
shortlist models; use yours to choose.

## 37.6 Regression testing in CI

The operational payoff:

```
change -> run eval set -> compare with baseline -> block or flag on regression
```

Practicalities: cost (eval runs spend money — use a small fast set on every commit and the full set
nightly), noise (temperature 0 where possible, or run N times and average), and thresholds (define
what counts as a regression in advance, statistically — a 1-point move on 30 cases is noise).

Version everything together: prompt version, model, retrieval config, code commit, score. Otherwise
a regression is untraceable.

## 37.7 The metrics that matter

Beyond quality: cost per request, p50/p95 latency, error rate, refusal rate, and **task success** (did
the user get what they needed?). A system that is 2% more accurate and 3× more expensive may be worse.

The discipline: define your metric *before* you make changes, not after you see the numbers.

---

# Part 2 — Questions to implement

Build `eval/` here: `dataset.py`, `metrics.py`, `judge.py`, `runner.py`. Evaluate something you
built earlier — your Phase 5 RAG system or a Phase 4 feature.

### Q1. A golden dataset
**Build:** 40 cases from real or realistic usage — including 5 ambiguous, 5 adversarial, and 5 that
*should* be refused. Store as versioned JSON with acceptance criteria.
**Check:** split into dev (30) and test (10); don't look at test.
**Explain:** how did you choose the cases? What kind of input is under-represented?

### Q2. Deterministic metrics
**Build:** every cheap check that applies: schema validity, required/forbidden content, citation
presence, latency and token budgets.
**Check:** run on your system and report a score per metric.
**Explain:** what fraction of your quality concerns can these cover? What's left?

### Q3. Measure non-determinism first
**Build:** run the same 10 cases 5 times at your production temperature.
**Check:** report the variance in your score.
**Explain:** what is the *noise floor* of your evaluation? Any improvement smaller than this is
unmeasurable — state the number.

### Q4. An LLM judge
**Build:** a judge with a specific rubric and reasoning-before-score.
**Check:** run it over your dev set.
**Explain:** read 5 of its judgements. Do you agree? Where is it wrong?

### Q5. Validate the judge
**Build:** hand-label 25 outputs yourself, then measure judge–human agreement.
**Check:** report agreement as a percentage, and the cases of disagreement.
**Explain:** report the number. Is your judge trustworthy enough to make decisions with?

### Q6. Judge biases
**Build:** test three biases — (a) same content, one verbose one terse; (b) pairwise comparison in
both orders; (c) an answer that is confident and wrong versus hedged and right.
**Check:** record the judge's behaviour in each.
**Explain:** which biases did you reproduce? How will you mitigate them?

### Q7. Pairwise vs absolute
**Build:** score 20 output pairs both ways (absolute 1–5 each, and pairwise A/B).
**Check:** compare consistency with your own judgement.
**Explain:** which method was more reliable, and by how much?

### Q8. A real comparison
**Build:** evaluate two versions of your system (two prompts, or two models) on the full dev set.
**Check:** report all metrics plus cost and latency, and whether the difference exceeds your Q3
noise floor.
**Explain:** is the difference real? What would you need to be confident?

### Q9. Held-out test
**Build:** after iterating on dev, run your best version on the untouched test slice.
**Explain:** compare dev and test scores. Did you overfit? By how much?

### Q10. Grow the set from failures
**Build:** find 5 genuine failures not represented in your dataset; add them as cases.
**Check:** your score drops.
**Explain:** why is a dropping score good news here?

### Q11. CI integration
**Build:** a script run on every change: fast subset, compare to baseline, exit non-zero on
regression beyond a defined threshold.
**Check:** make a deliberately bad prompt change and confirm it's caught. Make a trivial change and
confirm it isn't flagged.
**Explain:** what threshold did you pick, and how did you justify it statistically?

### Q12. Full dashboard
**Build:** a report: quality per metric, cost per request, p50/p95 latency, error rate, refusal
rate — with version metadata for prompt, model and code.
**Check:** produce it for three versions of your system.
**Explain:** which version would you ship, considering all axes rather than quality alone?

---

# Done when you can answer

1. Why is LLM evaluation statistical rather than assertive?
2. What makes a good golden dataset, and why are invented cases misleading?
3. When should you use deterministic checks rather than a judge?
4. How do you validate an LLM judge, and what biases must you test for?
5. Why is pairwise comparison more reliable than absolute scoring?
6. What is a noise floor, and why must you measure it first?
7. Why can't public benchmarks tell you whether your system is good?

Write answers in `notes.md`.
