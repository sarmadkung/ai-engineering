# Project 8 — Multi-Agent Research System

**Depends on:** Phase 8 (Topics 33–34), Phase 7, Phase 9 for measurement.

**What you prove:** that you can build a multi-agent system *and* judge honestly whether it beat a single
agent. Most multi-agent projects skip the second half. Yours won't.

---

# Part 1 — What you are building

A system that researches a topic and produces a sourced report:

```
question -> supervisor decomposes into sub-questions
         -> workers research in parallel (search, fetch, extract)
         -> supervisor verifies and assembles
         -> report with citations
```

The legitimate justification here is **context isolation** (Topic 34): each worker reads many documents
and returns a summary, so the supervisor's context stays clean. Parallelism is the second benefit.
State both explicitly, and then test whether they materialize.

---

# Part 2 — Design decisions

| Decision | Options | Consider |
|---|---|---|
| Topology | supervisor / pipeline / peer | supervisor, almost certainly (Topic 34) |
| Worker context | full history / task brief only | brief only, or you lose the whole benefit |
| Handoff format | prose / structured | structured (Topic 18) |
| Worker model | same as supervisor / cheaper | cheap workers, strong supervisor is the usual win |
| Verification | trust workers / supervisor verifies | verify — workers report success optimistically |
| Parallelism | sequential / concurrent with cap | concurrent, with a cap |
| Sources | web / your corpus / both | both is more interesting |

---

# Part 3 — Milestones

### M1 — Evaluation set and single-agent baseline
10 research questions with rubrics for a good answer (coverage, accuracy, sourcing). Then solve them with
**one** agent with the same tools.
**Check:** report quality scores, wall time, cost, and peak context tokens.
**Explain:** this is the bar. What specifically was hard for the single agent?

### M2 — Research tools (Topic 27)
Search, fetch-and-extract, and retrieval over your own corpus. Untrusted-content handling.
**Check:** the single agent can answer a question with real citations.

### M3 — Supervisor and decomposition (Topic 34)
A supervisor that decomposes a question into independent, verifiable, bounded, complete sub-questions,
represented as data (ids, dependencies, status).
**Check:** inspect three decompositions by hand. Are the subtasks genuinely independent and complete?

### M4 — Workers with clean contexts
Workers receive a structured brief (task, constraints, only necessary context) and return a structured
result (findings, sources, what couldn't be done).
**Check:** compare main-context peak tokens against M1's single agent. **This is the core claim — report
the number.**

### M5 — Parallel execution (Topics 33, 34)
Concurrent workers with a cap; one failure doesn't lose the others; partial results preserved.
**Check:** report wall time versus sequential, and confirm a deliberately failing worker doesn't sink the
run.

### M6 — Supervisor verification (Topic 34)
The supervisor checks worker output rather than trusting claims, and can re-dispatch a failed subtask.
**Check:** count how often a worker claimed success incorrectly. Report the false-success rate.

### M7 — Conflict resolution (Topic 34)
Two workers return contradictory findings — detect and resolve explicitly (by source quality, recency, or
flagging the disagreement in the report).
**Check:** construct the case. What did it do before you handled it?

### M8 — Report assembly with citations (Topic 43)
A structured report: findings, evidence, sources, and explicit uncertainty where sources disagreed or
coverage was thin.
**Check:** verify 20 citations actually support their claims. Report the unsupported rate.

### M9 — Cost and budget control (Topics 33, 34)
Per-worker budgets, a global run budget, no recursive worker spawning, and per-agent cost tracking.
**Check:** report total cost per report and the multiple versus the single agent. Trigger each limit.

### M10 — Orchestration robustness (Topic 33)
Durable state, resume after a crash, cancellation, per-run traces.
**Check:** kill a run with 3 of 5 workers finished; resume and confirm only the unfinished work re-runs.

---

# Part 4 — The honest comparison

The deliverable that matters most. One table on the same 10 questions:

| | quality | wall time | cost | peak context | LOC | debug time |
|---|---|---|---|---|---|---|
| single agent | | | | | | |
| multi-agent | | | | | | |

Then answer, in writing:

1. Did quality improve, and by how much relative to your measurement noise (Topic 37)?
2. What was the cost multiple? (Expect 3–10×, Topic 34.)
3. Did context isolation actually materialize in the numbers?
4. How much wall-clock did parallelism save?
5. Which of Topic 34's four legitimate reasons applied — or did none?
6. **Would you ship it?**

Also run: decomposition quality (good/overlapping/incomplete), information loss across hops, and
structured versus prose handoffs.

---

# Part 5 — Deliverables

- The system, producing real reports on real questions
- `EVALUATION.md` — questions, rubrics, scores for both architectures
- `COMPARISON.md` — the table above and your honest verdict
- `ARCHITECTURE.md` — topology, brief/result schemas, budgets
- 3 example reports with verified citations
- Run traces showing the full supervisor/worker interaction

---

# Part 6 — Done when

- It produces genuinely useful sourced reports.
- You have measured it against a single agent on every axis and written down the verdict — including if
  the verdict is "the single agent was better".
- Cost per report is known and bounded by enforced budgets.
- Citations are verified, and disagreement between sources appears in the report rather than being
  silently resolved.
- A crashed run resumes without redoing completed work.

---

# Stretch

Hierarchical supervisors for very large questions; a dedicated critic agent reviewing the report
(Topic 31); iterative deepening where thin sections trigger more research; human-in-the-loop plan
approval before expensive research (Topic 33); scheduled recurring research with change detection.
