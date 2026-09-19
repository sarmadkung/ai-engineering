# Topic 31 — Agent Architectures

**Why this topic:** "agent" isn't one design. These are the named patterns you'll meet in papers,
frameworks and job interviews, each solving a specific weakness of the plain loop. Knowing them lets
you choose deliberately instead of reinventing them badly.

---

# Part 1 — Theory

## 31.1 ReAct (reason + act)

The baseline, and what you built in Topic 29: interleave a reasoning step with each action.

```
Thought -> Action -> Observation -> Thought -> Action -> ... -> Answer
```

Strengths: simple, adaptive, and the trace is legible — you can see *why* it acted.
Weaknesses: no global plan, so it can wander; and it has no mechanism for noticing its own bad
answer.

Everything below is ReAct plus one addition.

## 31.2 Router agents

One cheap model call classifies the request and dispatches it to the right handler:

```
request -> router -> { simple Q&A | RAG pipeline | SQL agent | coding agent | refuse }
```

This is the most underrated pattern in applied AI. Most production "agents" are really a router in
front of a few specialized workflows, because that's cheaper, faster and more testable than one
agent with thirty tools.

Design notes: route with a small model (classification is easy), make routes explicit and few, have
a fallback route, and **log routing decisions** — misrouting is invisible otherwise. Consider
confidence thresholds so ambiguous requests go to the more capable path.

## 31.3 Planner / executor

Separate the thinking from the doing:

```
planner (strong model)  -> a plan of steps
executor (cheap model)  -> executes each step with tools
```

Benefits: the plan is inspectable and approvable before anything happens; you pay for the expensive
model once; and the executor's job is narrow enough for a small model. Costs: plans go stale when
reality differs, so you need re-planning; and the split adds latency and complexity.

Best for multi-step tasks with side effects, where a human wants to approve the plan first.

## 31.4 Reflection

The agent reviews its own output and revises:

```
generate -> critique -> revise -> (repeat)
```

It works because critiquing is an easier task than producing, and because the critique enters the
context as new information. Reliable gains on writing, code, and analysis.

Limits worth knowing: gains usually plateau after 1–2 rounds; each round costs a full generation;
and **self-critique without external feedback is weakly correlated with actual correctness** — a
model often cannot see its own error. Reflection grounded in a real signal (test results,
validation errors, a retrieval check) is dramatically better than reflection on vibes.

## 31.5 Critic systems

Separate the critic from the producer, so the reviewer isn't the author:

- **Model-as-critic** — a second call, with a rubric, ideally a different/stronger model.
- **Tool-as-critic** — tests, a linter, a type checker, a schema validator. Objective, cheap, and
  the best kind when available.
- **Human-as-critic** — the gold standard, expensive.

Architecturally: producer → critic → (accept | revise with feedback) → loop with a cap. The pattern
that wins in practice is a *tool* critic, because it's grounded: "the tests fail" is not an opinion.

## 31.6 State machines

Instead of letting the model pick any next step, define explicit states and legal transitions; the
model decides *within* a state.

```
gather_info -> validate -> confirm_with_user -> execute -> report
```

Advantages: control, testability, guaranteed invariants (you cannot reach `execute` without passing
`confirm`), and easy human-in-the-loop insertion. Cost: rigidity — unanticipated situations have no
state.

This is the pattern behind graph-based agent frameworks (Topic 32) and the sane choice for
business-critical flows. **Constrain what must never go wrong; leave the rest to the model.**

## 31.7 Other patterns worth knowing

- **Tool-use loop with verification** — plain ReAct plus a mandatory check before declaring success.
  Cheap, and fixes Topic 29's premature-success failure.
- **Tree of Thoughts** — explore several reasoning branches, evaluate, keep the best. Expensive;
  occasionally worth it for puzzles and search problems.
- **Debate** — two agents argue, a judge decides. Interesting, rarely cost-effective.
- **Subagents** — delegate a bounded subtask to a fresh agent with its own clean context, and return
  only a summary. This is primarily a *context management* technique and is genuinely useful (Phase
  8).
- **Hierarchical** — a supervisor decomposes and delegates to specialists (Phase 8).

## 31.8 Choosing

| Situation | Pattern |
|---|---|
| Distinct request types | router |
| Known procedure | workflow or state machine |
| Multi-step with side effects | planner/executor + approval |
| Output quality matters | reflection with a tool critic |
| Must never violate a rule | state machine |
| Unpredictable exploration | ReAct with caps |
| Context fills up with subtask detail | subagents |

Two rules that hold generally: **start with the simplest pattern that could work** and add structure
only where measurement shows you need it; and **prefer grounded feedback over more model calls.**

---

# Part 2 — Questions to implement

Build one module per pattern here. Use the same task set for all of them so comparisons are fair,
and keep your Topic 29 traces and metrics.

### Q1. ReAct baseline
**Build:** your Topic 29 agent, cleaned up, with a fixed evaluation set of 20 tasks.
**Check:** record success rate, steps, latency, cost.
**Explain:** these are your baseline numbers. Which failure mode dominated?

### Q2. Router
**Build:** a router over 4 handlers (direct answer, RAG, SQL/data, agent). Use a cheap model.
**Check:** measure routing accuracy on 40 labelled requests, plus end-to-end cost versus sending
everything to the full agent.
**Explain:** report routing accuracy and cost saved. What did misrouting cost when it happened?

### Q3. Router fallbacks
**Build:** add a confidence threshold and a fallback route.
**Check:** feed 10 deliberately ambiguous requests.
**Explain:** what did the router do before the fallback existed?

### Q4. Planner/executor
**Build:** a strong model plans; a cheap model executes each step.
**Check:** success rate and cost against the ReAct baseline.
**Explain:** report both. How often was the plan wrong from the start?

### Q5. Re-planning
**Build:** add re-planning when a step fails or returns something unexpected.
**Check:** introduce a surprise (a tool returning unexpected data) and compare with and without
re-planning.
**Explain:** what did the static plan do with the surprise?

### Q6. Plan approval
**Build:** show the plan to a human for approval/editing before execution.
**Check:** approve one, reject one, edit one.
**Explain:** for which class of task is this mandatory rather than nice?

### Q7. Reflection
**Build:** generate → self-critique → revise, with 1, 2, and 3 rounds.
**Check:** score output quality per round, and cost per round.
**Explain:** where did it plateau? Was round 3 ever worth it?

### Q8. Ungrounded vs grounded critique
**Build:** two critics for the same task: self-critique on vibes, and a tool critic (tests, schema
validation, or a retrieval check).
**Check:** measure actual correctness improvement for each.
**Explain:** report both. How often did self-critique approve a wrong answer?

### Q9. Separate critic model
**Build:** a distinct critic (different or stronger model) with an explicit rubric.
**Check:** compare against self-critique on the same outputs.
**Explain:** did separation help? Was the cost justified?

### Q10. State machine
**Build:** a 5-state machine for a task with a hard invariant (e.g. nothing is sent before explicit
confirmation). Enforce legal transitions in code.
**Check:** write a test that *tries* to reach `execute` without `confirm` and fails.
**Explain:** why can't a prompt instruction give you this guarantee?

### Q11. Rigidity cost
**Build:** feed your state machine 5 inputs it wasn't designed for.
**Check:** observe the behaviour.
**Explain:** what happened? Where's the line between "constrain it" and "let the model decide"?

### Q12. Subagent for context
**Build:** delegate a research-heavy subtask to a subagent with a fresh context, returning only a
summary.
**Check:** compare main-agent context size and total cost against doing it inline.
**Explain:** report the context saving. What information was lost in the summary, and did it matter?

### Q13. The comparison table
**Build:** run every pattern on the same 20 tasks.
**Check:** one table — success rate, steps, latency, cost, and failure modes per pattern.
**Explain:** which pattern would you ship for this task, and why? Which was more complexity than it
was worth?

---

# Done when you can answer

1. What is ReAct, and what are its two weaknesses?
2. Why are routers underrated?
3. What does separating planner from executor buy, and what does it cost?
4. Why is reflection on self-critique alone unreliable?
5. What's the best kind of critic, and why?
6. What can a state machine guarantee that a prompt cannot?
7. What problem do subagents actually solve?

Write answers in `notes.md`.
