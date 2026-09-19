# Topic 56 — Intelligence

**Why this topic:** Phase 15 steps back from building to thinking. This topic examines what the
capability words actually mean — generalization, reasoning, memory, learning — because vague use of
them produces both overclaiming and underclaiming about what these systems do. The questions here are
conceptual, with experiments you can actually run.

---

# Part 1 — Theory

## 56.1 What we mean by intelligence

No agreed definition. Useful framings:

- **Goal achievement across varied environments** — intelligence as generality of competence
  (Legg & Hutter).
- **Skill-acquisition efficiency** — not what a system can do, but how little data and experience it
  needs to learn something new (Chollet). Under this framing, a system trained on the entire internet
  demonstrating a skill shows less intelligence than a child learning it from three examples.
- **Compression** — intelligence as finding the shortest description of experience, which is literally
  what next-token training optimizes.
- **Prediction** — modelling the world well enough to anticipate it.

These matter practically, because the framing you adopt determines what counts as progress. Under
"goal achievement", LLMs are clearly intelligent. Under "skill-acquisition efficiency", the picture is
much more mixed — and both statements are defensible.

## 56.2 Generalization

The heart of the matter. Levels worth distinguishing:

- **Interpolation** — handling inputs within the training distribution. LLMs excel.
- **Extrapolation** — beyond the distribution. Much weaker.
- **Systematic generalization** — if you know "A is B" you should know "B is A" and be able to compose
  known rules in new ways. LLMs do this unreliably, which is one of the sharpest known limitations
  (the "reversal curse" is a concrete, reproducible instance).
- **Transfer** — applying learning from one domain in another.

The hard empirical question with no clean answer: how much of an LLM's ability is genuine
generalization and how much is sophisticated retrieval from an enormous training set? Contamination
(Topic 37) makes this extremely difficult to measure, and it's why benchmark scores are weak evidence
about understanding.

**What's clear:** these systems generalize far better than previous approaches, and less reliably
than a competent human — and they fail in ways that don't match human failure patterns, which is what
makes them hard to supervise.

## 56.3 Transfer learning

Two senses:

- **Technical** — pretraining then fine-tuning (Topic 14). Solved, routine.
- **Cognitive** — using knowledge from one domain to reason about another. Partially present:
  in-context learning transfers a pattern shown in the prompt, code training improves non-code
  reasoning, and multilingual models transfer across languages.

The limit is *reliability*. A system that transfers correctly most of the time and fails unpredictably
is difficult to build on — which is why so much of Phases 9 and 10 exists.

## 56.4 Reasoning

A word doing too much work. Distinguish:

- **Deductive** — following rules to a conclusion. LLMs are decent, and better with explicit chains.
- **Inductive** — inferring a rule from examples. Reasonable, and what in-context learning does.
- **Abductive** — best explanation for an observation. Weaker.
- **Causal** — distinguishing correlation from cause, reasoning about interventions. Notably weak, and
  probably not learnable from text alone.
- **Analogical** — mapping structure between domains. Mixed.
- **Counterfactual** — what if things had been different. Weak.

The open debate: is chain-of-thought *reasoning*, or the appearance of it? Evidence for: it improves
accuracy on genuinely novel problems, and the improvement scales with compute (Topic 53). Evidence
against: reasoning chains can be unfaithful (the stated reasoning doesn't match what drove the
answer), performance degrades on superficial perturbations of the same problem, and errors are
sometimes ones no reasoner would make.

A defensible position: it is a real and useful computational process that is not identical to human
reasoning, and treating it as either "just autocomplete" or "thinking like us" leads to wrong
predictions about where it will fail.

## 56.5 Planning

Sequencing actions toward a goal under constraints (Topic 29). Current systems do short plans
reasonably, and degrade with horizon length, constraint interaction, and the need to revise mid-plan.
Classical planners are far better at verifiable planning; LLMs are far better at loosely-specified
real-world goals. Hybrids — LLM proposes, solver verifies — are the practical direction, and another
instance of Topic 53's verification asymmetry.

## 56.6 Memory

Covered in Topic 54. The conceptual point for this topic: memory is not storage, it's the **integration**
of experience into usable knowledge. Current systems have no native mechanism for it, and every
implementation is scaffolding. This is one of the clearest structural gaps between current AI and
anything that learns over a lifetime.

## 56.7 Learning

Humans learn continuously from few examples with grounded feedback. LLMs learn in one enormous batch
from vast data, then stop. The mismatch defines the gap (Topic 55).

Note something genuinely surprising: **in-context learning** — acquiring a new pattern from a few
examples, with no weight change — was not designed, but emerged from scale. It's the closest current
systems come to human-like rapid learning, and it doesn't persist beyond the context window.

## 56.8 What's missing

An honest list, with the caveat that each is contested:

- **Grounding** — meaning learned from text versus from interaction with a world.
- **Continual learning** — accumulating knowledge without retraining.
- **Reliable systematic generalization** — composition and symbol manipulation that always works.
- **Causal models** — intervention and counterfactuals, not just correlation.
- **Calibrated self-knowledge** — knowing what you don't know (Topic 43).
- **Long-horizon coherence** — maintaining goals over very long timescales.
- **Sample efficiency** — learning from few examples.

Some may dissolve with scale — several capabilities people declared missing did. Some may require
different architectures. **Predicting which is the core disagreement in the field**, and honest
uncertainty is the right posture.

---

# Part 2 — Questions to investigate

These are experiments and written analyses rather than systems to build. Keep findings in `notes.md`
with your data. Use a current model via API, and use models of different sizes where you can.

### Q1. Definitions and consequences
**Write:** for each of §56.1's four framings, state whether current LLMs qualify as intelligent and why.
**Explain:** which framing do you find most useful for *engineering* decisions, and what does it predict
that the others don't?

### Q2. The reversal curse
**Build:** test bidirectional facts — obscure ones where you can check both directions ("X's mother is
Y" / "Y's son is X"). 20 pairs.
**Check:** report accuracy in each direction.
**Explain:** did you reproduce asymmetry? What does it suggest about how facts are represented?

### Q3. Interpolation vs extrapolation
**Build:** a task with a clean rule (e.g. arithmetic in an unusual base, or a novel string
transformation). Test inside the plausible training distribution and well outside it.
**Check:** report accuracy for both.
**Explain:** where did it break down? Is that a reasoning limit or a data limit — and how could you
tell?

### Q4. Systematic composition
**Build:** teach two simple rules in-context, then test each alone and both composed.
**Check:** report accuracy for single and composed application.
**Explain:** did composition work? This is the systematic-generalization question — report your evidence.

### Q5. Superficial perturbation
**Build:** take 15 problems it solves reliably. Change only surface features — names, units, irrelevant
added detail, reordered clauses.
**Check:** report accuracy before and after.
**Explain:** how much did performance drop? What does that imply about whether the original solution was
reasoning or matching?

### Q6. Chain-of-thought faithfulness
**Build:** on problems it gets right, insert a subtle error into its stated reasoning and ask it to
continue; separately, compare its stated reasoning against what actually changes the answer (e.g. by
perturbing an input the reasoning claims is irrelevant).
**Check:** record whether stated reasoning matches actual dependence.
**Explain:** was the reasoning faithful? What follows for using reasoning traces as explanations?

### Q7. Causal reasoning
**Build:** 10 problems distinguishing correlation from causation, and 10 intervention questions ("if we
forced X, what happens to Y?").
**Check:** report accuracy for each type.
**Explain:** where did it fail, and what kind of information would be needed to do better?

### Q8. Counterfactuals
**Build:** questions about well-known situations with a single premise altered.
**Check:** does it reason from the altered premise, or revert to the factual world?
**Explain:** report the failure pattern.

### Q9. Analogical transfer
**Build:** present a problem structure in one domain, then an isomorphic problem in an unrelated domain.
**Check:** does prior exposure help?
**Explain:** report with and without. What does this say about transfer?

### Q10. Planning horizon
**Build:** planning tasks of increasing length and constraint interaction (3, 6, 12 steps).
**Check:** report validity of plans at each length.
**Explain:** where did it degrade, and how? Did it fail by omission, contradiction, or invalid steps?

### Q11. In-context learning limits
**Build:** a genuinely novel pattern (an invented transformation). Test with 1, 5, and 20 examples.
**Check:** report accuracy per count.
**Explain:** how quickly did it learn? Compare with how many examples *you* needed.

### Q12. Calibration
**Build:** 40 questions spanning easy to obscure. Ask for an answer and a confidence.
**Check:** bucket by stated confidence and report accuracy per bucket.
**Explain:** is it calibrated? Relate to Topic 43's engineering consequences.

### Q13. Emergence, if you can test it
**Build:** if you can access several sizes of one model family, test a task that small models fail
entirely.
**Check:** report accuracy by size.
**Explain:** was the transition gradual or sharp? (Note the debate: apparent sharpness can be an
artifact of the metric — does yours change with a smoother metric?)

### Q14. Your own position
**Write:** 1,000 words: what current systems can and cannot do, which limitations you think scale
removes and which need something new, and what evidence would change your mind.
**Explain:** cite your own experiments from Q2–Q13. Date it, and plan to revisit it.

---

# Done when you can answer

1. What are the main definitions of intelligence, and what does each imply about LLMs?
2. What are the levels of generalization, and where do LLMs sit?
3. Why is it hard to tell generalization from retrieval?
4. Which kinds of reasoning are LLMs weak at?
5. What is unfaithful reasoning, and why does it matter?
6. Why is in-context learning surprising?
7. What is missing, and which gaps might scale close?

Write answers in `notes.md`.
