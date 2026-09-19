# Project 9 — Autonomous Software Engineering Agent

**Depends on:** Project 5, Phase 8 (Topics 33–36), Phase 9, Phase 10.

**What you prove:** that you can build long-horizon autonomy where it's actually feasible — and that you
understand the reliability arithmetic well enough to know where the human checkpoints belong.

---

# Part 1 — What you are building

An agent that takes an issue and produces a reviewed-ready pull request:

```
issue -> understand codebase -> plan -> implement across files -> test -> fix
      -> self-review -> commit -> PR with description
```

This is Project 5 scaled up: multi-file changes, longer horizons, real git operations, and a verification
loop that has to hold together over dozens of steps.

The honest framing from Topic 36: at 95% per-step reliability, a 20-step task succeeds about 36% of the
time. **Verification-and-retry is what breaks that multiplication** — so it is the centre of this project,
not an add-on.

---

# Part 2 — Design decisions

| Decision | Options | Consider |
|---|---|---|
| Task scope | bug fixes / small features / refactors | narrow scope, high success beats the reverse |
| Isolation | worktree / container | must be complete; this agent runs git commands |
| Plan representation | prose / structured task list | structured, with dependencies (Topic 36) |
| Verification | tests / + types / + lint / + self-review | as much objective signal as you can get |
| Human checkpoints | none / plan / per-risky-step / PR only | measure it (Topic 36, Q14) |
| Recovery | retry / re-plan / escalate | a full ladder |

---

# Part 3 — Milestones

### M1 — The benchmark and the verifier (Topic 36)
20 real tasks from your own repositories with **objective** success criteria (a failing test that must
pass, a behaviour that must change). Write the verifier before the agent.
**Check:** each verifier accepts a known-good patch and rejects three near-misses.
**Explain:** why the verifier comes first.

### M2 — Isolated workspace
A git worktree or container per run, with the agent able to run git but unable to touch your real
branches or push anywhere.
**Check:** attempt `git push`, editing outside the worktree, and network access. All blocked.

### M3 — Codebase understanding (Phase 5, Topic 35)
Repository exploration: structure, search, targeted reads. Optionally an index over the codebase.
**Check:** the agent locates the relevant files for 10 issues without reading the whole repo. Report
tokens used.

### M4 — Structured planning (Topic 36)
A task list with ids, dependencies, status and attempts, persisted and queryable. Re-planning when
reality diverges.
**Check:** print a plan mid-run. Introduce a surprise and confirm re-planning happens.

### M5 — Multi-file implementation
Coordinated edits across files, with each step verified where possible.
**Check:** a task needing 3+ file changes completes coherently. No file is left half-edited on failure.

### M6 — The verification loop (Topic 36)
Run tests/types/lint after each meaningful change; feed failures back; detect identical repeated failures
and change approach; cap retries.
**Check:** compare success rate with informed retry versus naive re-run. Report the per-step reliability
you achieve.

### M7 — The reliability arithmetic, measured (Topic 36)
Using M6's per-step rate, predict end-to-end success for 5, 10, 20 steps. Then run tasks of those lengths.
**Check:** compare prediction with reality.
**Explain:** did the arithmetic hold? What did verification-and-retry do to the effective per-step rate?

### M8 — Self-review (Topic 31)
Before finishing, the agent reviews its own diff against the issue, with a tool critic (tests, lint) in
addition to its own reading.
**Check:** measure how often self-review catches a real problem, and how often it approves a bad diff.
Report both.

### M9 — Durable execution (Topic 33)
Checkpoints including plan state and data position; resume after a crash; cancellation; no duplicated side
effects.
**Check:** kill a run mid-implementation and resume. Confirm no double commits.

### M10 — Output: a reviewable PR
Commit with a sensible message, PR description stating what changed and why, what was verified, what's
uncertain, and what a reviewer should look at.
**Check:** hand three PRs to someone (or review them yourself cold). Are they reviewable without the
trace?

### M11 — Honest reporting (Topic 36)
Per-subtask status with evidence; partial success reported as such; explicit "needs human attention".
**Check:** on a run where 2 of 6 subtasks fail, the report is actionable.

### M12 — Evaluation (Topic 39)
The 20-task suite, 3 runs each: outcome categories, false-success rate, steps, cost per successful task,
failure taxonomy.
**Check:** the full report. Fix the top failure category and re-measure.

---

# Part 4 — Experiments

1. **Human checkpoint placement** — fully autonomous versus plan-approval versus per-risky-step. Measure
   success and *human time spent* (Topic 36, Q14).
2. **Task length** — success rate by required step count. Where does it collapse?
3. **Verification strength** — tests only versus tests+types+lint+self-review.
4. **Model comparison** — reasoning versus standard model (Topic 53). Which failures disappear?
5. **Retry strategy** — naive versus informed versus escalation ladder.
6. **Cost per successful task** — including failed attempts. Compare with your own time at your hourly
   rate.

---

# Part 5 — Deliverables

- The agent, producing PRs on real issues in real repositories
- `BENCHMARK.md` — the 20 tasks, verifiers, and full results
- `RELIABILITY.md` — per-step rate, the arithmetic, and how verification changed it
- `RESULTS.md` — experiment tables, cost per successful task
- `SECURITY.md` — sandbox verification, git safety
- 5 example PRs, with the traces that produced them

---

# Part 6 — Done when

- It produces PRs you'd actually review, with tests passing as evidence.
- You know the per-step reliability, the end-to-end rate by task length, and the false-success rate.
- It cannot push, cannot escape the worktree, and cannot double-commit after a crash.
- Partial failure is reported honestly and actionably.
- You can say where the human checkpoint belongs, with numbers behind the answer.

---

# Stretch

Responding to PR review comments; running in CI on labelled issues; learning from merged versus rejected
PRs (Topic 55); cross-repository changes; a dashboard of runs with cost and outcomes.
