# Topic 54 — Memory Systems

**Why this topic:** Topic 30 built practical agent memory. This topic goes deeper: memory as a
system with its own architecture, lifecycle and failure modes — the thing that would have to work
for an AI to accumulate knowledge over years rather than restart every session.

---

# Part 1 — Theory

## 54.1 The gap

A model's knowledge is frozen at training time; its context is temporary. Between the two there is
nothing — no mechanism for learning from experience and keeping it.

Humans consolidate: experience → short-term memory → selective long-term storage → retrieval when
relevant → revision as understanding improves. Every part of that loop has to be *built* for an AI
system, and no current design does it well. This is one of the genuinely open problems between here
and general intelligence.

## 54.2 External memory

The architecture:

```
experience -> extract what matters -> store (with structure + metadata)
                                        -> retrieve when relevant -> inject into context
                                        -> revise/consolidate/forget over time
```

Storage substrates and what each is good for:

- **Vector store** — semantic recall of episodes and passages. Fuzzy, scalable, no structure.
- **Relational** — facts with constraints, updates, and joins. Structured, requires a schema.
- **Knowledge graph** — entities and relationships, multi-hop traversal (Topic 25). Expressive,
  expensive to build and maintain.
- **Documents/files** — human-readable, diffable, agent-writable. Underrated: a notes file is often the
  most effective memory a coding agent has.
- **Key-value** — fast lookup for known keys.

Real systems combine them, and the combination is the design work.

## 54.3 Knowledge stores

The step beyond "remember text": turn experience into structured knowledge.

Requirements that make it hard: **entity resolution** (is this "IBM" the same as that one?),
**schema evolution** (you learn a new kind of fact), **provenance** (where did this come from, and how
confident are we?), **temporal validity** (true then, not now — "works at Acme" needs a time range),
and **contradiction handling**.

Provenance and temporality are the two most commonly omitted and most commonly needed. Without
provenance you cannot audit or retract; without time, your store accumulates facts that were true once
and are now wrong — and it will assert them confidently.

## 54.4 Retrieval, with the hard parts named

Topic 30 covered relevance, recency, importance and frequency. The deeper problems:

- **Cued recall** — a human remembers something because of an oblique association. Similarity search
  approximates this badly, which is why agents fail to recall relevant experience that a person would.
- **Retrieval of absence** — knowing you *don't* know something, and that you once tried and failed, is
  valuable and almost never stored.
- **Compositional recall** — combining several stored facts to answer something none contains
  (Topic 25's multi-hop problem).
- **Context-appropriate depth** — a passing reference versus full detail.

## 54.5 Consolidation

Raw memory accumulates and degrades retrieval. Consolidation is the process of turning many
experiences into fewer, better memories:

- **Summarization** — many episodes into one account. Loses detail (Topic 19's drift).
- **Abstraction** — many specifics into a general rule ("deployments on Friday tend to fail"). The most
  valuable operation, and the hardest to do safely: over-generalizing from two incidents produces a
  confident false belief that is then permanent.
- **Deduplication and merging** — several statements of one fact into one, with a confidence that rises
  with corroboration.
- **Reconciliation** — resolving contradictions, ideally by recency plus provenance rather than
  arbitrarily.
- **Forgetting** — deliberate removal of the stale, superseded and unused. Necessary; not merely a
  capacity measure. An unbounded store has worse precision for everyone.

When to consolidate: at session end, on a schedule, or when the store grows past a threshold. Note the
cost — consolidation is LLM work over your whole store, which doesn't scale linearly.

## 54.6 Failure modes

Worth naming because they're specific to memory systems:

- **False memory** — a wrong extraction becomes permanent and authoritative (Topic 30).
- **Over-generalization** — a rule abstracted from too little evidence.
- **Staleness** — the store asserts what used to be true.
- **Retrieval failure** — the memory exists but never surfaces. Invisible, and therefore the most
  insidious: your system looks like it has no memory rather than a broken index.
- **Context pollution** — too many memories injected, degrading reasoning (Topic 19).
- **Privacy leakage** — memory crossing user or tenant boundaries (Topic 41).

A functioning memory system needs to be *evaluated* like any other component: recall of things it
should know, precision of what it injects, and correctness of what it stores.

## 54.7 What this gestures at

The research directions: continual learning (updating weights without catastrophic forgetting),
retrieval-augmented pretraining, memory-augmented architectures with learned read/write, and very
long context as a substitute for memory (simpler, but no consolidation and no forgetting — and
lost-in-the-middle still applies).

The honest position: nobody has solved this. Memory is one of the clearest gaps between current
systems and anything that learns over a lifetime — which is why it recurs in Phase 15.

---

# Part 2 — Questions to implement

Build `memory_system/` here, extending Topic 30. Postgres, a vector store, and a scenario spanning many
sessions.

### Q1. A multi-session scenario
**Build:** a scripted 20-session scenario with continuity requirements: facts stated early and needed
late, preferences that change, experiences that should inform later ones.
**Check:** a test set of 30 questions checkable at the end.
**Explain:** which requirements do you expect to fail, and why?

### Q2. Hybrid storage
**Build:** structured facts in Postgres, episodes in a vector store, and an agent-writable notes file.
**Check:** all three are used in one session.
**Explain:** which memory type went where, and why?

### Q3. Provenance and confidence
**Build:** every memory records source session, source message, extraction time, and a confidence score.
**Check:** trace one memory back to the exact message that produced it.
**Explain:** what does this make possible that a bare fact doesn't?

### Q4. Temporal validity
**Build:** facts with `valid_from` and `valid_to`; queries ask "what is true now?" and "what was true
on date X?"
**Check:** a superseded fact is not returned as current but is still auditable.
**Explain:** show a case where ignoring time would produce a confidently wrong answer.

### Q5. Entity resolution
**Build:** deliberately introduce variant names for the same entity, then implement resolution.
**Check:** report duplicates before and after.
**Explain:** which variants did your resolver miss? What would a wrong merge cost?

### Q6. Contradiction reconciliation
**Build:** a policy resolving contradictions by recency plus provenance confidence.
**Check:** feed conflicting facts and verify the right one wins and the other is retained as history.
**Explain:** what would arbitrary resolution have done over 20 sessions?

### Q7. Retrieval failure, measured
**Build:** a recall test — ask 30 questions whose answers are definitely in the store.
**Check:** report recall.
**Explain:** report your number. For the failures, was the memory badly stored or badly retrieved? Why
is this the most insidious failure mode?

### Q8. Retrieval of absence
**Build:** store failed attempts ("tried X, it didn't work") and retrieve them when a similar approach
comes up.
**Check:** the agent avoids a previously failed approach.
**Explain:** what happened before you stored failures?

### Q9. Compositional recall
**Build:** 5 questions requiring two or more stored facts combined.
**Check:** measure accuracy with plain retrieval, then with multi-hop retrieval or graph traversal.
**Explain:** report both. Why does single-vector retrieval struggle here?

### Q10. Consolidation
**Build:** a scheduled job that deduplicates, merges corroborated facts (raising confidence), summarizes
old episodes, and abstracts repeated patterns into rules.
**Check:** report store size and retrieval precision before and after.
**Explain:** what improved? What was lost?

### Q11. Over-generalization
**Build:** a scenario where two similar incidents could support a false general rule. Let consolidation
run.
**Check:** did it produce the bad rule?
**Explain:** report what it concluded. What evidence threshold would you require for abstraction, and how
would you allow a rule to be retracted?

### Q12. Forgetting
**Build:** decay based on age, access frequency and importance, with protection for high-importance
facts.
**Check:** simulate 100 sessions; report store size and retrieval precision with and without decay.
**Explain:** what did you almost delete that mattered?

### Q13. Context pollution
**Build:** inject 3, 10, and 30 retrieved memories per turn.
**Check:** measure task success and any degradation.
**Explain:** where was the optimum? Show a case where an irrelevant memory caused a wrong action.

### Q14. Evaluate the memory system
**Build:** a proper evaluation: recall (does it know what it should?), precision (is what it injects
relevant?), correctness (is what it stored true?), and staleness rate.
**Check:** report all four across your 20-session scenario.
**Explain:** which metric is worst? What would you fix first, and what does that say about the state of
memory systems generally?

---

# Done when you can answer

1. What loop would an AI need to accumulate knowledge, and which parts are missing?
2. What do provenance and temporal validity add, and what breaks without them?
3. What is retrieval of absence, and why is it valuable?
4. What are the consolidation operations, and which is most dangerous?
5. Why is silent retrieval failure the most insidious failure mode?
6. Why is forgetting necessary rather than just economical?
7. Why isn't a very long context a substitute for a memory system?

Write answers in `notes.md`.
