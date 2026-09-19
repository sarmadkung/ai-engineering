# Topic 33 — Agent Orchestration

**Why this topic:** a long-running agent is a distributed system with a non-deterministic component.
It crashes, waits for humans, gets interrupted and must resume. This topic is the durable execution
machinery that makes that survivable.

---

# Part 1 — Theory

## 33.1 State management

The moment an agent runs longer than one request, its state must live outside the process.

What constitutes agent state: the goal, conversation and tool history, the plan and its progress,
accumulated results, budget consumed, current step, and status.

Two rules:

- **Explicit, typed, serializable state.** A dict you mutate freely is undebuggable and
  unrestorable. A typed state object you can save, diff and inspect is the foundation of everything
  else in this topic.
- **Transitions, not mutations.** Each step takes state and returns new state. That makes
  checkpointing, replay and time travel possible — and makes "what changed at step 7?" answerable.

## 33.2 Workflows vs agents, revisited

At orchestration level the distinction is about *where control lives*:

- **Workflow** — your code decides the sequence; the model fills in steps. Predictable, testable.
- **Agent** — the model decides. Flexible, unpredictable.

Real systems are hybrid: a workflow skeleton with agentic steps inside it. The engineering principle
is to **constrain the parts that must not go wrong** and leave discretion where it's valuable — the
same idea as Topic 31's state machines, applied to whole systems.

Composition patterns: sequential, parallel (fan-out/fan-in), conditional branches, loops with caps,
and subworkflows.

## 33.3 Checkpoints and durable execution

Save state after each step so a run can survive a crash, a deploy, or a pause for human input.

What to persist: the state object, the step index, timestamps, and enough information to resume
deterministically.

Design points that matter:

- **Checkpoint granularity** — per step is the usual right answer. Finer costs writes; coarser loses
  work.
- **Idempotency of steps.** This is the hard part. If you crash *after* sending an email but
  *before* checkpointing, resuming will send it again. Side-effecting steps need idempotency keys or
  an outbox pattern, exactly as in Topic 20.
- **Determinism on replay.** If a step's behaviour depends on `now()` or a random seed, replay
  diverges. Record such values in the state.

## 33.4 Human-in-the-loop

Not an exception path — a first-class feature, needed for approval of consequential actions,
disambiguation, missing information, and review of output.

Mechanically it forces **durable pause**: the run stops, state is persisted, and hours later a
human responds and it resumes — possibly in a different process. This is why checkpointing and HITL
are the same topic.

Design considerations: a clear presentation of what will happen (with real arguments), the ability
to approve, reject *with feedback the agent can use*, or edit; timeouts with a defined default; and
awareness of approval fatigue (Topic 28). A rejection should re-enter the agent's context as
information, not just abort.

## 33.5 Interruptions and cancellation

Users change their minds mid-run. Requirements:

- **Cooperative cancellation** — check a cancellation flag between steps; don't kill a process
  mid-side-effect.
- **Stop paying immediately.** Cancel the in-flight provider request too.
- **Clean partial state** — record what was completed, so a partially-done task is visible rather
  than silently abandoned.
- **Steering**, the more interesting case: new instructions arriving mid-run, which should be
  injected into the agent's context at the next step rather than restarting.

## 33.6 Recovery

Failures and their correct responses:

| Failure | Response |
|---|---|
| transient tool/API error | retry with backoff inside the step |
| model error (429/5xx) | retry; consider a fallback model |
| invalid model output | re-prompt with the validation error (Topic 18) |
| step fails repeatedly | mark failed, re-plan or escalate |
| process crash | resume from last checkpoint |
| agent stuck in a loop | detect, break, escalate (Topic 29) |
| budget exhausted | stop cleanly, report progress |

Two things separate a robust system: **retry at the right layer** (a transient HTTP failure is your
wrapper's problem, an invalid argument is the model's), and a **dead-letter destination** for runs
that cannot proceed, so they're visible rather than lost.

**Partial success is the normal outcome** of long agent runs. Design the reporting for it: "4 of 6
subtasks completed, 2 failed for these reasons" is a good outcome; "failed" is not a useful answer.

## 33.7 Observability for long runs

Every run needs a durable trace: id, goal, per-step records (input, output, tool, tokens, cost,
latency), status transitions, human interactions, and the final outcome. You must be able to answer
"what happened in run X" hours later, from storage, and "which step usually fails" across all runs.

Live progress matters too: a run that takes ten minutes with no visible progress will be assumed
broken and killed.

---

# Part 2 — Questions to implement

Build `orchestrator.py` here — or do it in LangGraph and compare. Postgres for state (Topic 20).

### Q1. Typed state
**Build:** a serializable state object with goal, history, plan, progress, budget, status. Save and
load it.
**Check:** a round trip restores an identical object, including nested structures.
**Explain:** which field did you nearly leave out, and what would have broken on resume?

### Q2. Steps as transitions
**Build:** refactor your agent so each step is `(state) -> new_state`, with no hidden mutation.
**Check:** you can diff consecutive states and see exactly what changed each step.
**Explain:** print the diff for one run. What does this make possible that mutation didn't?

### Q3. Checkpoint and resume
**Build:** persist state after every step; resume from the latest checkpoint.
**Check:** kill the process at step 3 of 8 and resume — it completes without repeating work.
**Explain:** what did you have to store beyond the state itself?

### Q4. The double side-effect bug
**Build:** a step with an external side effect (write a file, send a fake email). Crash *after* the
effect but *before* the checkpoint, then resume.
**Check:** confirm the effect happens twice.
**Explain:** now fix it with an idempotency key or outbox. Which did you choose and why?

### Q5. Non-deterministic replay
**Build:** a step whose behaviour depends on `now()` or a random choice. Replay from a checkpoint.
**Check:** observe divergence.
**Explain:** how did you make replay deterministic?

### Q6. Parallel fan-out/fan-in
**Build:** a workflow that fans out to N independent subtasks and joins the results, with a
concurrency cap.
**Check:** measure wall time against sequential. One subtask failing does not lose the others.
**Explain:** report both times. How did you represent partial failure in state?

### Q7. Durable human approval
**Build:** pause before a destructive step, persist, exit the process entirely. In a *new* process,
approve, and resume.
**Check:** it completes correctly across the process boundary.
**Explain:** why is this harder than an in-memory prompt, and what did you need to store?

### Q8. Rejection with feedback
**Build:** allow reject-with-reason, feeding the reason back into the agent's context.
**Check:** the agent produces a different, informed second attempt.
**Explain:** compare with a plain abort. What did feedback change?

### Q9. Cancellation
**Build:** cooperative cancellation between steps, cancelling the in-flight model request too.
**Check:** from logs, prove no provider call was made after cancellation, and that partial progress
was recorded.
**Explain:** how much money did cancelling the in-flight request save on a long run?

### Q10. Steering mid-run
**Build:** accept a new instruction during a run and inject it at the next step.
**Check:** the agent changes behaviour without restarting.
**Explain:** what happened to the work already done? Was that correct?

### Q11. Recovery matrix
**Build:** handle each row of §33.6's table with the right strategy, and a dead-letter store for
unrecoverable runs.
**Check:** a test per row.
**Explain:** which failure was hardest to classify correctly at runtime?

### Q12. Partial success reporting
**Build:** a 6-subtask run where 2 subtasks are made to fail. Report per-subtask status with
reasons.
**Check:** the report is actionable — a human can see exactly what to retry.
**Explain:** how would a naive implementation have reported this?

### Q13. Run inspection
**Build:** durable traces plus a CLI or endpoint: list runs, show a run's steps, show cost and
duration, replay from step N.
**Check:** run 20 tasks including failures, then answer from storage alone: which step fails most,
p95 duration, total spend, how many needed a human.
**Explain:** report those four numbers and the most surprising one.

---

# Done when you can answer

1. Why must agent state be explicit and serializable?
2. Why are transitions better than mutation?
3. What's the hardest part of resuming from a checkpoint?
4. Why are human-in-the-loop and checkpointing the same problem?
5. How does cancellation differ from killing the process?
6. At which layer should each kind of retry happen?
7. Why is partial success the normal outcome, and how should it be reported?

Write answers in `notes.md`.
