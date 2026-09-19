# Phase 14 — AI Systems & Reasoning

**Goal:** the capability frontier — models that think before answering, memory that accumulates, and
systems that improve from their own experience. Practical where it can be, honest about the limits where
it can't.

---

## What I will do

| # | Topic | What I build | Status |
|---|---|---|---|
| 53 | [Reasoning Models](53-reasoning-models/) | `reasoning/` — effort dials, search, self-consistency, routing | ready |
| 54 | [Memory Systems](54-memory-systems/) | `memory_system/` — provenance, temporality, consolidation, forgetting | ready |
| 55 | [Learning Agents](55-learning-agents/) | `learning/` — feedback loops and gated self-improvement | ready |

## Which things to learn

**53. Reasoning Models** — reasoning tokens and how they were trained; **test-time compute** as quality you
can buy per request; why hand-written "think step by step" scaffolding is often counterproductive now;
search (best-of-N, beam, tree) and why **the evaluator is the binding constraint**; verification levels;
self-consistency and disagreement as an uncertainty signal; when a reasoning model is the wrong choice;
what they changed about agent design and what they didn't.

**54. Memory Systems** — the consolidation loop humans have and AI systems lack; storage substrates and
what each is for; **provenance and temporal validity** as the two most-omitted essentials; entity
resolution and contradiction reconciliation; the hard retrieval problems (cued, absence, compositional);
consolidation operations, with over-generalization as the dangerous one; **silent retrieval failure** as
the most insidious bug; forgetting as necessary; and evaluating a memory system like any component.

**55. Learning Agents** — the four meanings of "learning" and which are practical; feedback signals,
attribution and delay; what a system can safely change about itself (memory, examples, procedures, tools,
prompts, weights) in order of risk; **every self-modification gated by measurement**, or it drifts
downhill; the four environment properties learning requires; why continual weight updating is unsolved;
error amplification; and what genuinely improves today.

## Prerequisites

Phases 7–9 (agents, memory, evaluation). A domain with **objective verification** for Topics 53 and 55 —
code with tests is ideal.

**Next:** Phase 15 — stepping back from engineering to concepts.
