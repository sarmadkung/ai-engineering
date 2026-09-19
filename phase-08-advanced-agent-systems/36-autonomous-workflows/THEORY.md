# Topic 36 — Autonomous Workflows

**Why this topic:** the capstone of agent engineering — a system that takes a goal, decomposes it,
executes, checks its own work, and recovers, with little supervision. This is also where honesty
about reliability matters most: autonomy multiplies both capability and failure.

---

# Part 1 — Theory

## 36.1 What autonomy requires

```
goal -> decompose -> execute -> verify -> (retry | self-correct) -> report
```

The mechanism that makes autonomy work isn't better planning — it's **verification**. An agent that
can check its own work can recover; one that can't will confidently report a wrong result, and that
single property determines whether autonomy is usable.

Which means the first design question for any autonomous system is: **how will success be checked,
objectively?** If you cannot answer it, don't build the autonomous version.

## 36.2 Task decomposition

Good subtasks are independent, verifiable, bounded, and complete (Topic 34). Beyond that:

- **Static vs dynamic decomposition** — plan everything upfront (auditable, brittle) or decompose
  incrementally as you learn (adaptive, harder to audit). Hybrid: a coarse upfront plan, refined
  per-step.
- **Depth limits.** Recursive decomposition without a limit is a common runaway: subtasks spawning
  subtasks until the budget is gone.
- **Represent the plan as data** — a task list with ids, dependencies, status, and results. Not prose
  in a context window, because you need to query it, resume it, and show it to a human.
- **Dependencies** must be explicit, or parallelization breaks silently (Topic 34).

## 36.3 Execution

Per subtask: assemble the context it needs, run (agent or workflow), verify, record the outcome, and
update the plan. Then: run independent tasks in parallel, respect a global budget, and keep going
after a failure rather than aborting the whole run.

**Partial progress must be preserved.** A 20-subtask run that fails at 18 should keep the 17
completed results, so a retry costs one subtask rather than eighteen.

## 36.4 Verification — the heart of it

Kinds, from strongest to weakest:

| Verification | Example | Trust |
|---|---|---|
| **Deterministic check** | tests pass, code compiles, schema validates, totals reconcile | high |
| **Tool-based** | linter, type checker, API returns 200 | high |
| **LLM judge with a rubric** | "does this answer the question, citing sources?" | medium |
| **Self-assessment** | "did I do it right?" | low |
| **None** | assume success | none |

**Push verification as far up this table as your task allows.** The reason coding agents work
noticeably better than general ones is that code has deterministic verification built in: tests,
compilers, types. Where you can manufacture such a signal, autonomy becomes viable; where you
can't, keep a human in the loop.

Practical rules: verify *outcomes* not *claims* (does the file exist and contain what was intended,
not "the agent said it wrote it"); verify at every level (subtask and whole goal); and treat a
verification failure as information for the retry, not just a status.

## 36.5 Retry and self-correction

Retry is not "run it again" — that repeats the same failure. Effective retry feeds the failure
*reason* into the next attempt.

```
attempt -> verify -> fail -> analyse why -> change approach -> attempt again
```

Strategies: same approach (only for transient failures), modified approach (informed by the error),
decompose further, use a different tool, escalate to a stronger model, or escalate to a human.

Guards you need: a retry cap per subtask, **detection of identical repeated failures** (three
identical errors means the approach cannot work — change it or stop), and a global budget so retry
loops can't consume the run.

Self-correction is strongest when grounded: "the test says `expected 4, got 5`" produces a real fix;
"maybe I should reconsider" produces churn. This is Topic 31's lesson, made load-bearing.

## 36.6 Reporting

An autonomous run's report is the product. It must state: what was accomplished, what wasn't and why,
what was verified and how, what needs human attention, what it cost, and where the artifacts are.

**Never report undifferentiated success.** "Done" hides the 2 of 20 subtasks that quietly failed, and
that habit is what destroys trust in autonomous systems. Report per-subtask status, with evidence.

## 36.7 The reliability arithmetic

The fact that governs autonomous design: per-step reliability compounds.

```
95% per step, 10 steps  -> 0.95^10 ≈ 60% end-to-end
99% per step, 10 steps  -> 90%
95% per step, 50 steps  -> 8%
```

So long autonomous chains fail *usually*, not occasionally, unless each step is extremely reliable
**or** verification-plus-retry restores it. Verification is what breaks the multiplication: a step
that is checked and retried until it passes behaves like a much more reliable step.

This is also why short chains with human checkpoints outperform long fully-autonomous ones in
practice, and why the useful question is not "can it run alone?" but "where should the checkpoints
be?"

## 36.8 Knowing when to stop

An autonomous system must be able to say: this is done and verified; this is blocked and here's why;
this needs a decision I shouldn't make; I've spent the budget, here's the progress.

An agent that cannot stop and ask is not more autonomous — it's less useful, because everything it
produces has to be re-checked by hand.

---

# Part 2 — Questions to implement

Build `autonomous.py` here on top of Topics 33–35. Choose a goal with **deterministic verification**
— e.g. "implement these 5 functions so the provided tests pass", or "produce a dataset satisfying
these checkable constraints".

### Q1. Verification first
**Build:** before any agent code, write the verifier for your task. It must decide success
objectively.
**Check:** it correctly accepts a known-good result and rejects three kinds of near-miss.
**Explain:** why build this first? What would you have measured without it?

### Q2. Decomposition as data
**Build:** a plan structure — tasks with ids, dependencies, status, attempts, results — persisted and
queryable.
**Check:** you can print the plan at any point and see exactly what's done, in progress, and blocked.
**Explain:** why not keep the plan in the prompt?

### Q3. Sequential execution with per-task verification
**Build:** execute tasks in dependency order, verifying each.
**Check:** run on a 6-subtask goal.
**Explain:** report per-task success on the first attempt. What was your per-step reliability?

### Q4. The compounding arithmetic, measured
**Build:** using Q3's per-step rate, predict end-to-end success for 6, 10, and 20 steps. Then
actually run a 15-step goal 5 times.
**Check:** compare prediction with reality.
**Explain:** did the arithmetic hold? What does that mean for how long your chains can be?

### Q5. Verify outcomes, not claims
**Build:** log both the agent's claim of success and the verifier's result for every subtask.
**Check:** count disagreements over 20 subtasks.
**Explain:** report the false-success rate. What would have happened if you'd trusted claims?

### Q6. Informed retry
**Build:** on failure, feed the verification error into the retry, with a cap.
**Check:** compare success rate against naive re-run-the-same-thing retry.
**Explain:** report both. How many tasks succeeded on attempt 2 with feedback and not without?

### Q7. Identical-failure detection
**Build:** detect the same error occurring N times and force a change of approach (or escalate).
**Check:** construct a task that cannot succeed with the obvious approach.
**Explain:** what did it do before detection existed, and what did that cost?

### Q8. Escalation ladder
**Build:** on repeated failure, escalate: decompose further → different tool → stronger model →
human.
**Check:** each rung triggers in a crafted scenario.
**Explain:** which rung resolved most failures? Which was never worth it?

### Q9. Parallel execution with dependencies
**Build:** run independent tasks concurrently while respecting the dependency graph.
**Check:** wall time versus sequential; dependent tasks never start early.
**Explain:** report the speedup and how you verified ordering was respected.

### Q10. Partial progress across a crash
**Build:** persist completed subtask results; crash at subtask 8 of 12; resume.
**Check:** the resumed run redoes only what's unfinished.
**Explain:** how much work and money did persistence save?

### Q11. Weak vs strong verification
**Build:** the same goal verified three ways: deterministic check, LLM judge, self-assessment.
**Check:** measure how often each verifier's verdict matches the deterministic truth.
**Explain:** report agreement rates. What's the practical consequence of relying on the weakest one?

### Q12. Honest reporting
**Build:** a report with per-subtask status, evidence of verification, what needs human attention,
cost, and artifact locations.
**Check:** run a goal where 2 of 8 subtasks genuinely fail.
**Explain:** compare your report with a naive "done"/"failed". What would a user have done wrong
with the naive version?

### Q13. Budgets and stopping
**Build:** global caps on cost, time and steps, with a clean stop and a progress report.
**Check:** trigger each.
**Explain:** what does the agent say when it runs out? Is it useful?

### Q14. Where to put the human
**Build:** three variants — fully autonomous, human checkpoint at the plan, and human checkpoint per
risky subtask.
**Check:** measure end-to-end success, human time spent, and cost for each on 10 goals.
**Explain:** which delivered the best *usable* outcome per unit of human attention? This is the real
design question of the topic — answer it with your numbers.

---

# Done when you can answer

1. Why is verification, not planning, the key to autonomy?
2. What makes a good subtask?
3. What are the levels of verification, and why do coding agents have it easy?
4. Why must retry include the failure reason?
5. What does the reliability arithmetic say about long chains, and how does verification change it?
6. Why is undifferentiated "success" a dangerous report?
7. When should an autonomous system stop and ask?

Write answers in `notes.md`.

---

**Phase 8 is complete.** You can build agent systems that survive contact with reality. Phase 9 makes
you able to prove it.
