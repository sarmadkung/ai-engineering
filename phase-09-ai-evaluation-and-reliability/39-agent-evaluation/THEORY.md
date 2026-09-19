# Topic 39 — Agent Evaluation

**Why this topic:** an agent's output is a *trajectory*, not an answer. Two runs can both succeed
while one took 3 steps and the other 40 and nearly deleted a file. This topic measures the whole
path, which is the only way to improve agents systematically.

---

# Part 1 — Theory

## 39.1 Why agents are harder to evaluate

- The output is a **sequence of actions**, plus side effects in the world.
- **Multiple valid paths** exist to the same goal.
- **Variance is high** — the same task can take 3 or 30 steps.
- **Side effects are real** — evaluation runs write files, send requests, spend money.
- **Partial success** is the normal case (Topic 36).
- **Cost and latency vary per run**, so they're quality dimensions, not constants.

## 39.2 Task success

Start here. Define success **objectively and in advance**:

- **Deterministic outcome** — the file exists with the right contents, the tests pass, the record was
  created correctly. Always prefer this (Topic 36).
- **Judged outcome** — an LLM or human scores the result against a rubric. Necessary for open-ended
  tasks.
- **Self-reported** — worthless on its own; measure it only to compute the *false-success rate*.

Report success as a distribution over categories, not a single number: verified success, unverified
claimed success, honest failure, **false success**, limit hit, crash. False success — the agent says
done and it isn't — is the metric that predicts how much a human must re-check, and therefore how
much the agent is actually worth.

## 39.3 Tool accuracy

Per tool call: was the right tool chosen? were the arguments correct? was the result used properly?

Aggregate: selection accuracy, argument validity rate, error rate per tool, unnecessary calls
(retrieval when no retrieval was needed), and missing calls (answering from memory when it should
have looked something up).

This is where most agent quality problems originate, and it's the cheapest thing to fix — usually a
tool description (Topics 26, 28).

## 39.4 Trajectory evaluation

Beyond "did it work": *how* did it work?

- **Step count** versus an optimal or reference path — efficiency.
- **Redundancy** — repeated identical calls, re-reading the same file.
- **Recovery** — when a step failed, did it adapt or repeat?
- **Path validity** — were the steps sensible even if the outcome was right? (A right answer via a
  wrong path will not generalize.)
- **Safety** — did it attempt anything dangerous, even if blocked?

Two approaches: compare against a **reference trajectory** (brittle, since many paths are valid), or
score the trajectory with a **rubric-driven judge** given the trace (flexible, needs validation like
any judge, Topic 37).

The practical high-value metric here is **redundancy and step count**, because they map directly to
cost.

## 39.5 Failure analysis

The highest-value activity in this topic: read traces, classify failures, count, fix the biggest
category. A working taxonomy:

| Category | Signature |
|---|---|
| wrong tool | picked search when it needed SQL |
| bad arguments | right tool, wrong parameters |
| misread result | result was correct, conclusion wasn't |
| lost the goal | drifted from what was asked |
| loop | same action repeated |
| premature success | stopped before finishing |
| context exhaustion | ran out of window |
| gave up too early | stopped when it could have continued |
| unsafe attempt | tried something destructive |
| genuinely impossible | correct failure (not a bug) |

Note how many of these are **engineering** failures with specific fixes: caps, verification, context
policy, better descriptions. That's the point — failure analysis tells you which knob to turn.

## 39.6 Cost and efficiency as quality

For agents, report per run: total tokens, cost, wall time, step count, tool calls, and
**cost per successful task** (total cost including failures, divided by successes). That last number
is the one that decides whether an agent is viable, and it's often 3–10× the cost of a successful run
alone.

A useful comparison is always: what would the same task cost a human, and what would a fixed workflow
cost? An agent that succeeds 60% of the time at $2 per attempt may be worse than a workflow that
succeeds 50% at $0.05.

## 39.7 Safe evaluation environments

Agent evaluation runs actions, so you need: a sandbox (Topic 28), a reset mechanism so every run
starts from the same state, mocked external effects (no real emails, no real charges), and a budget
cap so a runaway evaluation doesn't produce a memorable invoice.

Reset is subtler than it looks: a *stateful* test suite where run 3 inherits run 2's mess produces
irreproducible results and false conclusions.

## 39.8 Continuous agent evaluation

- A fixed task suite (20–50 tasks) run on every meaningful change.
- Multiple runs per task (variance is high — one run tells you almost nothing).
- Trace storage, so a regression can be diagnosed rather than just observed.
- Production monitoring: success rate, step distribution, cost distribution, human-intervention rate.
- Every production failure becomes a suite task.

---

# Part 2 — Questions to implement

Build `agent_eval/` here, evaluating your Phase 7/8 agent. Use your Topic 33 traces.

### Q1. A task suite with objective success
**Build:** 20 tasks with deterministic verifiers, spanning easy, medium, hard, and 3 that are
impossible.
**Check:** every verifier accepts a known-good and rejects a near-miss.
**Explain:** which task was hardest to define success for, and how did you resolve it?

### Q2. Variance
**Build:** run each task 5 times.
**Check:** report success rate, step count spread, and cost spread per task.
**Explain:** what's the widest spread you saw? What does that imply about single-run conclusions?

### Q3. Outcome categories
**Build:** classify every run into §39.2's categories.
**Check:** report the distribution over 100 runs.
**Explain:** report your **false-success rate**. How much human re-checking does that imply?

### Q4. Tool accuracy
**Build:** per-call labelling — correct tool, correct arguments, result used properly. Aggregate per
tool.
**Check:** report selection accuracy, argument validity, unnecessary calls, missing calls.
**Explain:** which tool is worst? Is it a description problem or a boundary problem?

### Q5. Step efficiency
**Build:** write a reference (near-optimal) trajectory for 10 tasks; compare your agent's step count.
**Check:** report the ratio per task.
**Explain:** where was it most wasteful, and why?

### Q6. Redundancy detection
**Build:** detect repeated identical calls and repeated reads of the same resource within a run.
**Check:** report redundant calls as a fraction of all calls, and their cost.
**Explain:** what fraction of your spend is redundancy? What would fix it?

### Q7. Recovery behaviour
**Build:** inject a tool failure at a random step; observe.
**Check:** classify outcomes — adapted, retried identically, gave up, ignored and continued wrongly.
**Explain:** report the distribution. Which response was most common, and is it acceptable?

### Q8. Trajectory judge
**Build:** an LLM judge scoring a trace against a rubric (efficiency, validity, safety).
**Check:** validate it against your own scoring of 15 traces.
**Explain:** report agreement. Where did the judge miss something you caught by reading?

### Q9. Failure taxonomy
**Build:** classify every failure in 100 runs using §39.5's categories.
**Check:** report counts per category.
**Explain:** the top category is your next engineering task. What is it, and what's the specific
fix?

### Q10. Fix and re-measure
**Build:** implement the fix for your top failure category.
**Check:** re-run the suite; compare category counts before and after.
**Explain:** did it help? Did anything else get worse?

### Q11. Cost per successful task
**Build:** compute total cost across all runs (including failures) divided by verified successes.
**Check:** compare against cost per *successful* run alone.
**Explain:** report the multiple. Then compare against the cost of your Phase 7 fixed workflow.
Which is economically better?

### Q12. Safe evaluation harness
**Build:** sandboxed runs, full reset between runs, mocked side effects, budget cap.
**Check:** prove reset works — run a destructive task twice and get identical results.
**Explain:** what leaked between runs before reset was right?

### Q13. Regression suite
**Build:** the suite in CI: N runs per task, comparison against a stored baseline with a statistically
justified threshold, traces archived.
**Check:** introduce a deliberate regression (weaken a tool description) and confirm detection.
**Explain:** how many runs per task did you need for the signal to exceed the noise? Show your
reasoning.

---

# Done when you can answer

1. Why is an agent's output a trajectory rather than an answer?
2. What outcome categories should you report, and why is false success the key one?
3. What does tool accuracy measure, and why is it the cheapest thing to fix?
4. Why compare against a reference trajectory, and why is that brittle?
5. Why is cost per successful task the number that matters?
6. Why does agent evaluation need reset, and what goes wrong without it?
7. Why can't you conclude anything from a single agent run?

Write answers in `notes.md`.
