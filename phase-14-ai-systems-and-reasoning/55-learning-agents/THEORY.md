# Topic 55 — Learning Agents

**Why this topic:** every system you've built is static — it performs the same way on day 100 as on day
1 unless a human improves it. This topic is about systems that improve from their own experience, what
that actually requires, and where the current honest limits are.

---

# Part 1 — Theory

## 55.1 What "learning" can mean

Four distinct things, often conflated, in increasing difficulty:

1. **Learning within a session** — using what happened earlier in the same task. Real, and it's just
   context (Topic 19).
2. **Learning across sessions via memory** — storing and reusing experience. Real, and it's Topic 54.
   The weights don't change; behaviour does.
3. **Learning by self-modification** — the system changes its own prompts, tools or workflows based on
   measured outcomes. Possible, rarely done well, and the most interesting near-term direction.
4. **Learning by weight updates** — continual training from experience. Mostly unsolved in production,
   for reasons below.

Most "self-improving agent" claims are (1) or (2). The engineering value sits in (3).

## 55.2 Feedback loops

Nothing improves without a signal. Where signals come from, in descending reliability:

- **Deterministic outcomes** — tests pass, the transaction succeeded, the value reconciled. Best.
- **Explicit human feedback** — ratings, corrections, approvals and rejections. Sparse but high quality.
  A *correction* is far more valuable than a thumbs-down, because it tells you the right answer.
- **Implicit human behaviour** — did the user accept the suggestion, rephrase, retry, or abandon? Noisy,
  abundant, and the most underused signal in most products (Topic 40).
- **Self-evaluation** — the model judging its own work. Weak and possibly self-confirming.
- **Downstream consequences** — did the recommendation work out a week later? Valuable and hard to
  attribute.

Two pitfalls that bite in real systems: **attribution** (a 10-step agent run succeeded — which step
helped?) and **delay** (the signal arrives long after the action, so the state that produced it must be
stored).

## 55.3 Self-improvement mechanisms

What a system can actually change about itself, from safest to riskiest:

- **Memory** — accumulate facts and episodes (Topic 54). Safe, incremental.
- **Examples** — add successful interactions to a few-shot pool, retrieving relevant ones per request.
  Effective and low-risk: the model learns from a curated pool without any training.
- **Procedures** — store a workflow that worked and reuse it (Topic 30's procedural memory).
- **Tools** — write a new tool for a repeated subtask. Powerful; needs review, since a self-written tool
  is unreviewed code.
- **Prompts** — modify its own instructions based on measured outcomes. Requires an evaluation harness
  and guardrails, or it drifts.
- **Weights** — fine-tune on accumulated successful interactions (Topic 50). Slow, expensive, risky.

The pattern across all of these: **changes must be gated by measurement.** An agent that modifies
itself without an evaluation gate will drift, and drift is usually downhill because a stochastic system
has more ways to get worse than better.

## 55.4 Evaluation-driven improvement

The honest version of self-improvement is a closed loop you'd recognize from Phase 9:

```
run in production -> log everything -> detect failures -> add to eval set
  -> propose a change (prompt, examples, tools, data) -> measure on the eval set
  -> ship if better, discard if not -> repeat
```

This is not glamorous and it is what actually works. The automatable parts: failure detection, eval-set
growth, change proposal (a model can suggest a prompt edit), measurement, and the ship/discard
decision. The parts that should stay human, at least for now: which failures matter, what counts as
better, and approval of anything with side effects.

Guardrails that make it safe: a held-out test set the optimizer cannot see (or it overfits — Phase 2's
lesson again), a minimum improvement threshold above your noise floor (Topic 37), rollback, and bounded
autonomy (it may edit a prompt, not grant itself a new permission).

## 55.5 Environment interaction

A learning agent needs an environment it can act in and get feedback from. Properties that determine
whether learning is feasible:

- **Verifiable** — you can tell success from failure. Without this, no learning signal exists.
- **Resettable** — repeated attempts from a known state. Without this, experiments contaminate each
  other (Topic 39).
- **Safe** — failures are cheap and reversible (Topic 43).
- **Fast** — iteration speed bounds learning speed.

This is exactly why coding is the domain where self-improving agents work best: tests give
verification, git gives reset, sandboxes give safety, and a test run takes seconds. Where an
environment lacks these, "learning" is aspiration.

## 55.6 Continual learning and why it's hard

Updating weights from a stream of experience, which would be the real thing:

- **Catastrophic forgetting** — new learning overwrites old (Topic 14).
- **Stability/plasticity trade-off** — learn fast and forget, or learn slowly and barely adapt.
- **No clean signal** — production data isn't labelled, and success is often ambiguous.
- **Error amplification** — training on your own outputs entrenches your own mistakes; do it repeatedly
  and quality collapses (model collapse).
- **Safety and auditability** — a model that changes daily is a model whose behaviour you cannot
  certify, and whose regressions you cannot bisect.
- **Cost** — training runs versus a prompt edit that takes seconds.

Research directions: parameter-efficient continual updates, replay buffers mixing old and new,
modular/adapter-per-domain approaches, and RL from real outcomes. Nothing here is a solved production
pattern, and treating it as one is how you ship a system that silently gets worse.

## 55.7 The realistic position

What you can build today that genuinely improves:

- Memory that accumulates verified facts.
- A curated example pool grown from successes.
- An evaluation set grown from real failures.
- Prompt and tool changes proposed automatically and gated by measurement.
- Periodic fine-tuning on verified good interactions, evaluated like any release.

That combination is a system that measurably improves over months — with humans in the loop for
judgement, and every change gated by evidence. That's the state of the art, and it's a great deal
better than static.

---

# Part 2 — Questions to implement

Build `learning/` here, on top of your agent, memory and evaluation work. **Choose a verifiable domain**
— code with tests is strongly recommended.

### Q1. Static baseline
**Build:** your agent on 30 tasks, with full logging. Freeze it.
**Check:** report success rate, steps, cost.
**Explain:** this is the "no learning" reference. What kinds of failure look learnable?

### Q2. A verifiable, resettable environment
**Build:** an environment with deterministic verification, reset to a known state, safety, and fast
iteration.
**Check:** the same task run twice from reset gives identical starting conditions.
**Explain:** which of the four properties was hardest to provide?

### Q3. Feedback collection
**Build:** capture all available signals — verification outcomes, explicit feedback, and implicit
behaviour (retries, rephrasings, abandonment).
**Check:** run 30 tasks and report how many produced each signal type.
**Explain:** which signal was most abundant? Which most reliable? How do you handle one arriving late?

### Q4. The attribution problem
**Build:** for multi-step runs, attempt to attribute success or failure to specific steps.
**Check:** on 10 runs, record your attribution and whether you believe it.
**Explain:** how confident are you? What would make attribution easier?

### Q5. Example-pool learning
**Build:** store successful interactions; retrieve the most relevant as few-shot examples per request.
**Check:** measure success rate as the pool grows (0, 10, 50 examples).
**Explain:** report the curve. Where did it flatten, and did any example ever hurt?

### Q6. Example quality control
**Build:** deliberately admit some *unverified* successes into the pool.
**Check:** measure the effect.
**Explain:** what happened? What's your admission criterion now?

### Q7. Procedural learning
**Build:** store successful procedures and reuse them for similar tasks.
**Check:** measure steps and cost on repeated task types.
**Explain:** did it get faster? Did a stale procedure ever cause a failure?

### Q8. Self-written tools
**Build:** let the agent write a tool for a repeated subtask, with human review before it becomes
available.
**Check:** measure the effect on the tasks it targets.
**Explain:** read the tool. Would you have approved it? What would an unreviewed version have risked?

### Q9. Automated prompt improvement
**Build:** a loop — analyse failures, propose a prompt change, measure on a dev eval set, keep only if
it improves beyond your noise floor.
**Check:** run 5 iterations, recording each proposal and its measured effect.
**Explain:** how many proposals actually helped? What did the failures look like?

### Q10. Overfitting the optimizer
**Build:** after Q9, evaluate on a held-out test set the loop never saw.
**Check:** compare dev and test improvements.
**Explain:** how much of your gain was real? What does this say about automated optimization loops
generally?

### Q11. Drift without gates
**Build:** run the same self-modification loop with **no** evaluation gate, accepting every proposed
change for 10 iterations.
**Check:** measure quality at each step.
**Explain:** what happened to quality over 10 iterations? State the lesson.

### Q12. Fine-tuning on accumulated successes
**Build:** collect verified successful interactions and fine-tune a small model on them (Topic 50).
**Check:** evaluate against the base model on task *and* general ability.
**Explain:** did it improve? Compare the cost and iteration speed with Q9's prompt loop.

### Q13. Error amplification
**Build:** fine-tune on the previous model's outputs, then again on *that* model's outputs, two or three
generations deep.
**Check:** evaluate each generation.
**Explain:** report the trajectory. What does this predict about training on your own outputs at scale?

### Q14. The whole loop, over time
**Build:** run the complete system — memory + examples + eval growth + gated prompt changes — over 100
tasks in batches, measuring after each batch.
**Check:** plot success rate, cost per successful task, and human interventions over time.
**Explain:** did it measurably improve? Which mechanism contributed most per unit of effort and risk?
And what still required a human?

---

# Done when you can answer

1. What are the four meanings of "learning", and which are practical today?
2. Which feedback signals are reliable, and what makes attribution hard?
3. What can a system safely change about itself, and in what order of risk?
4. Why must every self-modification be gated by measurement?
5. What four properties must an environment have for learning to be feasible?
6. Why is continual weight updating still unsolved in production?
7. What does error amplification predict about self-training?

Write answers in `notes.md`.

---

**Phase 14 is complete.** Phase 15 steps back from engineering to the concepts that frame where all of
this is going.
