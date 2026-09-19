# Topic 43 — AI Reliability

**Why this topic:** the defining failure of LLM systems is a confident wrong answer. This topic is
about making a system whose mistakes are visible, bounded and recoverable — which is what "production
ready" actually means for AI.

---

# Part 1 — Theory

## 43.1 Why models hallucinate

Not a bug — a consequence of the mechanism (Topic 1). The model samples the most plausible
continuation. A plausible-sounding citation, API method or statistic is, by construction, a likely
continuation. The model has no separate "do I know this?" signal to consult.

Contributing factors: gaps in training data, pressure to answer (alignment rewards helpfulness), very
long contexts, ambiguous questions, and a fluency bias — the same confident tone whether right or
wrong. The tone carries no information about correctness, and users read it as if it does.

**The reframing that matters:** you cannot eliminate hallucination. You can make it detectable,
bounded and rare. Every technique below does one of those three.

## 43.2 Grounding

The primary mitigation: provide the facts and require the answer to come from them.

- **Retrieval** (Phase 5) — supply the source material.
- **Explicit instructions** — answer only from the context; say you don't know otherwise.
- **Citations** — every claim attributed to a source, which makes verification possible.
- **Verification** — check that claims are actually supported (Topic 38's faithfulness).

Grounding does not guarantee groundedness (Topic 24, Q13): a model given context can still answer
from its priors, and will sometimes prefer its training over your document. So grounding must be
*measured*, not assumed.

## 43.3 Validation

Check the output before the user sees it. In order of strength:

- **Structural** — valid JSON, schema-conformant (Topic 18).
- **Semantic** — do referenced ids exist? do numbers reconcile? are dates plausible?
- **Factual** — do citations support their claims? are quoted passages actually present in the source?
  (String matching a quotation against the source document is cheap and catches a lot.)
- **Policy** — no forbidden content, no promises your business can't keep, no advice you're not
  licensed to give.

Where it can be automated and deterministic, automate it. Every unvalidated output path is a place
where a wrong answer reaches a user unchallenged.

## 43.4 Guardrails

Controls at the boundaries:

**Input** — length limits, injection detection (Topic 41), topic/scope checks, PII detection.
**Output** — content filters, schema validation, fact checks, the exfiltration-channel blocking from
Topic 41, and a confidence gate.

Guardrails have costs you must weigh honestly: latency (each check is time), false positives (blocking
legitimate use is a real product harm), and maintenance. Applying them everywhere degrades the
product; applying them where consequences are severe is right.

## 43.5 Uncertainty, and its limits

Signals worth using:

- **Retrieval score** — low top-score means you probably lack the material (Topic 25).
- **Self-consistency** — sample several answers; disagreement indicates uncertainty. Costly and
  genuinely informative.
- **Token logprobs** where available — low probability on key tokens correlates with uncertainty.
- **Explicit asking** — "how confident are you?" Poorly calibrated and easily gamed, but weakly
  informative.
- **Judge-based** — a second model assesses whether the answer is supported.

The honest caveat: **LLMs are poorly calibrated.** Stated confidence correlates weakly with
correctness. Self-consistency and retrieval scores are more trustworthy than self-reported
confidence, and both are imperfect. Don't build a system that depends on the model knowing what it
doesn't know.

## 43.6 Fallbacks

What happens when something fails:

| Failure | Fallback |
|---|---|
| provider outage / rate limit | another provider or model, with retries |
| model too slow | a faster model, or a cached/degraded answer |
| validation failure | retry with the error, then escalate |
| low confidence | say so, ask a clarifying question, or hand to a human |
| unanswerable | admit it, and offer what you *can* do |
| tool failure | alternative tool, or proceed without and say so |

Two design principles: **fail visibly, not silently** (a wrong answer is worse than "I can't do this
right now"), and **degrade gracefully** (partial results with caveats beat nothing).

Multi-provider fallback is worth building for anything critical, and costs mainly abstraction
discipline: keep your prompts and logic independent of a single provider's specifics.

## 43.7 Designing for wrongness

The architectural principle of the topic: **assume the output is wrong some percentage of the time,
and design the surrounding system so that's survivable.**

- **Human review** where stakes are high — the model drafts, a person approves.
- **Reversibility** — soft deletes, drafts, undo, staged changes. Reversible mistakes are
  dramatically cheaper than irreversible ones.
- **Blast radius limits** — per-operation caps, so one bad decision affects one record rather than
  ten thousand.
- **Transparency** — show sources, show what the system did, make it easy to check. This converts a
  hidden error into a visible one.
- **Honest UX** — don't present generated content with more authority than it deserves.

Match autonomy to consequence: high-stakes and irreversible needs a human; low-stakes and reversible
can be automatic. Most product failures in AI come from getting this mapping wrong, not from the
model being bad.

## 43.8 The reliability budget

Set explicit targets and measure them (Phase 9): acceptable hallucination rate, required citation
accuracy, acceptable refusal rate, latency and error-rate budgets. Then treat them as release gates.

Without targets, "reliable" is a feeling — and quality drifts downward with every well-intentioned
change.

---

# Part 2 — Questions to implement

Build `reliability/` here, hardening one of your earlier systems. Reuse Phase 9's evaluation harness
— you cannot do this topic without measurement.

### Q1. Measure your hallucination rate
**Build:** 40 questions — 25 answerable from your corpus, 15 not. Measure how often the system invents
an answer.
**Check:** report the rate for each group separately.
**Explain:** report your baseline. Would you have guessed it?

### Q2. Fluency carries no signal
**Build:** collect 10 confident-sounding wrong answers and 10 correct ones. Strip them of context and
try to tell them apart.
**Explain:** could you? What does that mean for your users?

### Q3. Grounding instructions
**Build:** compare plain prompting against explicit grounding instructions plus required citations.
**Check:** measure hallucination rate on both groups from Q1.
**Explain:** report the improvement and the residual rate.

### Q4. Quote verification
**Build:** require the model to quote the supporting passage, then string-match the quote against the
source.
**Check:** count fabricated or altered quotes.
**Explain:** report the number. Why is this check so cheap and so effective?

### Q5. Semantic validation
**Build:** validators for your domain — ids exist, totals reconcile, dates are plausible, references
resolve.
**Check:** each rejects a crafted bad output. Measure how often real outputs fail validation.
**Explain:** which validator fired most often, and what was the underlying model error?

### Q6. Retrieval confidence gate
**Build:** refuse or hedge when the top retrieval score is below a threshold you calibrated (Topic 21).
**Check:** measure on unanswerable questions *and* confirm answerable ones still get answered.
**Explain:** report both rates. What did over-tightening the threshold cost?

### Q7. Self-consistency as an uncertainty signal
**Build:** sample 5 answers; measure agreement; flag low-agreement cases.
**Check:** compare agreement against actual correctness on 30 questions.
**Explain:** does disagreement predict wrongness? Report the correlation. Is 5× cost justified for
your use case?

### Q8. Calibration check
**Build:** ask the model for a confidence score with every answer; compare stated confidence against
measured correctness across 40 cases.
**Check:** bucket by stated confidence and report accuracy per bucket.
**Explain:** is it calibrated? Would you gate on it?

### Q9. Provider fallback
**Build:** automatic fallback to a second model/provider on outage, rate limit, or timeout.
**Check:** simulate each failure; confirm the request succeeds.
**Explain:** how much did your code have to change to be provider-independent? What's still coupled?

### Q10. Graceful degradation
**Build:** with retrieval disabled, produce a clearly-caveated degraded answer rather than a silent
ungrounded one.
**Check:** compare user-visible output with and without degradation handling.
**Explain:** which version would a user trust more *appropriately*?

### Q11. Reversibility
**Build:** convert one destructive operation in your agent into a reversible one (draft, staged
change, soft delete, undo).
**Check:** demonstrate recovery from a wrong action.
**Explain:** how does this change the approval policy you need?

### Q12. Blast radius
**Build:** per-operation limits (max records affected, max spend, max messages sent).
**Check:** attempt a bulk operation exceeding them.
**Explain:** what's the worst single mistake your system can now make? Is that acceptable?

### Q13. Guardrail cost
**Build:** measure latency and false-positive rate for your full guardrail stack on 50 legitimate
requests.
**Check:** report added latency and how many legitimate requests were blocked or hedged.
**Explain:** which guardrail was most expensive per unit of protection? Would you remove any?

### Q14. The reliability budget
**Build:** written targets for hallucination rate, citation accuracy, refusal rate, latency and error
rate; measure your system against them; wire them into CI as release gates.
**Explain:** which target do you currently miss? What's your plan, and what would you tell a user in
the meantime?

---

# Done when you can answer

1. Why do models hallucinate, and why can't it be eliminated?
2. Why doesn't grounding guarantee groundedness?
3. What are the four validation layers?
4. Which uncertainty signals are trustworthy, and which aren't?
5. Why is failing visibly better than a wrong answer?
6. What does "design for wrongness" mean concretely?
7. How should autonomy be matched to consequence?

Write answers in `notes.md`.

---

**Phase 10 is complete.** Your systems can be attacked and survive, and fail safely. Phase 11 brings
the models onto your own hardware.
