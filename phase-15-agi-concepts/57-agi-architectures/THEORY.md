# Topic 57 — AGI Architectures

**Why this topic:** proposals for how general intelligence might be built. Most are speculative; the
value is that they clarify what a bare LLM lacks, and several ideas have already turned into the
engineering you've been doing for nine phases.

---

# Part 1 — Theory

## 57.1 Cognitive architectures

Before LLMs, decades of work tried to build general intelligence by specifying the components of
cognition: **SOAR** and **ACT-R** (working memory, long-term memory, production rules, goal stacks),
and **LIDA** (attention and consciousness-inspired cycles).

They were transparent, principled, and brittle — everything had to be hand-specified, and they never
handled the messiness of the real world.

The lesson worth taking, though, is a genuine one: their component analysis was reasonable. Perception,
working memory, long-term memory, attention, goals, action selection, learning — a modern agent stack
reimplements most of that list around an LLM, with the LLM supplying the flexible reasoning the old
systems lacked. Your Phase 7/8 agent is closer to ACT-R than you might expect.

## 57.2 World models

A world model is an internal representation of how things work, enabling prediction of what happens
next and simulation of the consequences of actions.

Why it matters: planning requires it (you cannot evaluate an action without predicting its effect),
and it's the difference between reacting and deliberating.

Do LLMs have one? Contested. Evidence for: they predict physical and social outcomes better than
chance, and probing studies find internal representations of structured state (a board position, a
spatial layout) in models trained only on text sequences. Evidence against: they fail on simple physical
reasoning, are inconsistent across phrasings of the same situation, and have no persistent updated
state.

Alternative approaches: learned latent world models trained by prediction (as in model-based RL), and
**JEPA**-style architectures that predict in representation space rather than generating pixels or
tokens — with the argument that predicting every detail wastes capacity on what doesn't matter.

## 57.3 Agentic architectures

The direction that's actually working, and it's what you built: an LLM as a reasoning core, wrapped in
tools, memory, planning, and feedback loops.

```
perception  <- tools, retrieval, observations
reasoning   <- the LLM
memory      <- external stores (Topic 54)
planning    <- decomposition and revision (Topics 29, 36)
action      <- tools with permissions (Phase 6)
learning    <- feedback loops and gated self-modification (Topic 55)
```

This is the pragmatic bet: general capability comes from scaffolding a strong general reasoner, not
from a new cognitive theory. What's notable is how far it goes — and what still limits it: long-horizon
coherence, reliable memory integration, and self-correction that doesn't require a human-designed
verifier.

## 57.4 Long-term memory as an architectural component

Every serious architecture proposal includes memory as a first-class component, not a bolt-on
(Topic 54). What it must do: write selectively, retrieve by association, consolidate and abstract,
handle contradiction, forget, and support reasoning *over* memory rather than just recall.

Nothing does all of this well. It's the most frequently identified missing piece across otherwise
disagreeing proposals — which is a reason to take it seriously.

## 57.5 Planning systems

The recurring proposal is hybrid: an LLM for open-ended, loosely-specified goals; a verifier or
classical solver for correctness. Neural for proposal, symbolic for validation.

This is Topic 53's asymmetry stated architecturally, and it's the same pattern as every reliable system
in this roadmap: generate flexibly, verify rigorously. If you take one architectural idea from Phase 15
into your engineering, this is the one.

## 57.6 Other proposals worth knowing

- **Neurosymbolic** — combine learned pattern recognition with symbolic manipulation, for reliability on
  exactly the systematic generalization LLMs fumble (Topic 56).
- **Multi-agent minds** — specialized modules (society of mind). Appealing; in practice most multi-agent
  systems underperform a single strong agent with tools (Topic 34).
- **Embodiment** — grounding meaning in sensorimotor interaction, on the view that text alone cannot
  supply causal understanding.
- **Scale maximalism** — no new architecture needed; more compute, data and post-training. Has won
  repeatedly against predictions, which is itself evidence worth weighing.
- **Recursive self-improvement** — a system that improves its own design. The mechanism behind
  fast-takeoff scenarios, and currently unrealized (Topic 55 explains why it's hard).

## 57.7 How to hold these ideas

Every proposal above has advocates who are not fools, and the disagreements are about empirical
questions nobody has settled. Two failure modes to avoid: dismissing architecture research because
scaling works, and dismissing scaling because it "isn't real intelligence".

The engineer's version of this topic: **the gaps these architectures target are the same gaps you
work around in production** — memory, verification, planning, grounding. Reading the proposals tells
you which of your scaffolding is a permanent part of the design and which is a temporary patch. That's
genuinely useful, and it's why this topic is in a practical roadmap.

---

# Part 2 — Questions to investigate

Reading, analysis, and a few implementable experiments. Keep everything in `notes.md`.

### Q1. Map your own agent onto a cognitive architecture
**Write:** take ACT-R or SOAR's component list and map each component onto a part of your Phase 8 agent.
**Explain:** which components do you have? Which are missing or fake? Which does the LLM absorb entirely?

### Q2. Probe for a world model
**Build:** a simple state-tracking task — describe a sequence of moves in a small world (objects moved
between containers, or a 3×3 grid) and query the final state at increasing lengths.
**Check:** report accuracy by sequence length.
**Explain:** does it maintain state? Where does it break? Is the failure memory or modelling?

### Q3. Consistency of the world model
**Build:** ask the same physical/causal question in 5 different phrasings.
**Check:** count distinct answers.
**Explain:** what does inconsistency imply about whether there's a single underlying model?

### Q4. Prediction vs simulation
**Build:** ask for (a) what happens next and (b) what would happen if a specific intervention were made,
in the same scenario.
**Check:** compare quality.
**Explain:** is it better at prediction than at simulating alternatives? What would that mean
architecturally?

### Q5. Neural proposal, symbolic verification
**Build:** a task where an LLM proposes a solution and a deterministic checker validates it (a
constraint puzzle, a scheduling problem, or code with tests), with retry on failure.
**Check:** report accuracy for LLM alone, checker alone (if feasible), and the hybrid.
**Explain:** report all three. Why did the hybrid win, and what does that generalize to?

### Q6. Where a solver beats an LLM
**Build:** a constraint problem (scheduling, packing) solved by an LLM and by an actual solver.
**Check:** compare correctness and time.
**Explain:** what does the LLM contribute that the solver can't, and vice versa? Design the ideal
division of labour.

### Q7. Memory as a first-class component
**Write:** using §57.4's list, audit your Topic 54 memory system.
**Explain:** which capabilities do you have, which are absent, and which would be hardest to add? Does
this match what architecture proposals identify as missing?

### Q8. Long-horizon coherence
**Build:** a task requiring goal maintenance over 50+ steps with distractions.
**Check:** measure goal drift — does it still pursue the original objective?
**Explain:** what mechanisms in your harness (Topic 35) prevented drift? Which failed?

### Q9. Multi-agent vs single agent, again
**Write:** using your Topic 34 numbers, evaluate the "society of mind" proposal.
**Explain:** did specialization help in your measurements? What would have to be true for it to help
more?

### Q10. The grounding argument
**Write:** state the strongest case that text-only training cannot yield causal understanding, and the
strongest case that it can.
**Explain:** which do you find more convincing, and what experiment would distinguish them?

### Q11. Read the primary sources
**Build:** read one paper from three different camps (e.g. a scaling-laws paper, a JEPA/world-model
paper, and a neurosymbolic paper).
**Explain:** summarize each in 200 words, then state what each claims the others get wrong.

### Q12. Which of your scaffolding is permanent?
**Write:** list the workarounds you've built across Phases 5–14 (retrieval, memory, verification,
caps, approval, evaluation).
**Explain:** for each, predict whether it's a permanent architectural component or a temporary patch
that better models will make unnecessary. Justify each prediction.

### Q13. Design an architecture
**Write:** a concrete architecture for a system that could learn continuously and act over long horizons
— components, interfaces, and how it handles memory, verification and learning.
**Explain:** name the part you're least confident about, and what experiment would test it.

### Q14. Position statement
**Write:** which approach do you think gets furthest in the next five years, and why?
**Explain:** commit to a falsifiable prediction, date it, and note what would prove you wrong.

---

# Done when you can answer

1. What did classical cognitive architectures get right and wrong?
2. What is a world model, and what's the evidence LLMs have one?
3. What are the components of the agentic architecture, and what still limits it?
4. What must a first-class memory component do?
5. Why do planning proposals converge on neural-plus-symbolic?
6. What does scale maximalism claim, and what supports it?
7. Which of your own scaffolding do you expect to be permanent?

Write answers in `notes.md`.
