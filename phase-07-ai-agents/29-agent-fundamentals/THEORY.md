# Topic 29 — Agent Fundamentals

**Why this topic:** you have all the parts — a model, tools, retrieval, context management. An agent
is what you get when you put them in a loop with a goal. This topic defines that precisely, and
makes you build the smallest honest version.

---

# Part 1 — Theory

## 29.1 What an agent is

> An agent is an LLM in a loop with tools, deciding its own next step toward a goal.

The distinction that matters:

| | Workflow | Agent |
|---|---|---|
| Control flow | you decide the steps | the model decides |
| Number of steps | fixed | unknown until it finishes |
| Cost/latency | predictable | variable |
| Debugging | straightforward | hard |
| Right when | you know the procedure | you can't specify the procedure |

A RAG pipeline is a workflow: retrieve, then generate, always. An agent given a `search` tool
decides whether to search, how many times, and with what queries.

**Most problems do not need an agent.** A fixed workflow is cheaper, faster, and far easier to test.
Use an agent when the steps genuinely cannot be specified in advance — and be honest about whether
that's true, because "agent" is fashionable and the failure modes are real.

## 29.2 The agent loop

```
observe   — what is the current state? (goal, history, last tool result)
think     — what should I do next?
act       — call a tool
observe   — read the result
... until the goal is met, a limit is hit, or it gives up
```

You already built this in Topic 26. The only thing added is that the loop is now driven by a **goal**
rather than a single request — which is what makes the number of iterations unbounded, and why
termination becomes a design problem rather than an afterthought.

## 29.3 Reasoning and acting together

The key idea (from the ReAct paper, Topic 31): interleave reasoning and action rather than planning
everything first.

```
Thought: I need the user's plan before I can answer about limits.
Action:  get_user(id=123)
Result:  {plan: "pro"}
Thought: Pro plan — now fetch the pro limits.
Action:  get_plan_limits(plan="pro")
```

Why interleaving works: each observation informs the next decision, so the agent adapts instead of
executing a plan that was wrong from step 2. And the reasoning text is in the context, so it
conditions subsequent steps (Topic 17's chain of thought, applied inside a loop).

On modern reasoning models this happens internally between tool calls, so explicit "Thought:"
scaffolding is often unnecessary — but understanding it explains what the model is doing.

## 29.4 Planning

Three approaches, each with a real trade:

- **Implicit** — no plan; decide each step as it comes. Simple and adaptive; can wander.
- **Upfront plan** — produce a plan, then execute it. Auditable, better for multi-step tasks, and
  brittle when reality diverges from the plan.
- **Plan-and-revise** — plan, execute, re-plan when something unexpected happens. Best for long
  tasks; more complex and more expensive.

A **todo list** the agent maintains is a practical middle ground: it externalizes the plan into
context, survives compaction, and lets a human see intent. This is what most good coding agents do.

## 29.5 Termination — the part beginners get wrong

An agent must stop. Conditions to implement, all of them:

- **Success** — the goal is met. Needs a definition of done: a verification step, or structured
  output declaring completion.
- **Iteration cap** — a hard ceiling.
- **Budget cap** — tokens or dollars.
- **Time cap** — wall clock.
- **Explicit give-up** — the agent decides it cannot proceed and says why. **A useful agent must be
  able to fail honestly**, or it will invent success.
- **Loop detection** — the same action repeated with the same arguments means stuck.

Without these, the characteristic failure is a confident infinite loop with a real invoice attached.

## 29.6 Where agents actually fail

Worth knowing before you build, because you'll see all of these:

- **Compounding errors** — an early mistake poisons every later step (Topic 1, §1.3).
- **Loops** — repeating the same failing action, or oscillating between two.
- **Premature success** — declaring the task done without verifying.
- **Context exhaustion** — tool results fill the window and the goal scrolls out of view.
- **Wrong tool, confidently** — see Topic 26.
- **Silent skipping** — quietly dropping part of a multi-part task and reporting success on the rest.

Notice that most of these are *not* model-intelligence failures. They're engineering failures:
missing verification, missing limits, missing context management.

## 29.7 The minimum viable agent

What you need for something genuinely useful:

```
goal + system prompt (role, constraints, definition of done)
tools (Phase 6, with permissions)
loop with all termination conditions
context management (Topic 19: history budget, clear old tool results)
verification (did it actually work?)
tracing (every thought, call, result, timing, cost)
```

Tracing is not optional. An agent without a trace is undebuggable, because the interesting behaviour
is a *sequence*, and you will be asked why it did something on step 7.

---

# Part 2 — Questions to implement

Build `agent.py` here. Reuse your Phase 6 tools. Pick a task with a checkable outcome — e.g. "find
the three cheapest options in this dataset and write them to a file", or a small research task.

### Q1. Workflow vs agent
**Build:** the same task twice: as a fixed pipeline, and as an agent with tools.
**Check:** run both on 10 inputs. Record success rate, steps, latency, cost, and variance.
**Explain:** which was better? Would you ship the agent? On what evidence?

### Q2. The bare loop
**Build:** the smallest agent: goal, tools, loop, iteration cap. Log every iteration.
**Check:** it completes a simple task. Print the full trace.
**Explain:** annotate the trace — where did it reason, where did it act, where did it decide to
stop?

### Q3. Reasoning visible vs suppressed
**Build:** two variants — one instructed to state its reasoning before each action, one told to act
without commentary.
**Check:** measure success rate, steps, and tokens on 10 tasks.
**Explain:** did visible reasoning help quality? What did it cost? Was it worth it on your model?

### Q4. No plan vs upfront plan
**Build:** implicit stepping, and plan-first execution.
**Check:** run both on a 5+ step task.
**Explain:** which handled a surprise better (introduce one: make a tool return unexpected data)?

### Q5. A todo list in context
**Build:** the agent maintains an explicit todo list it updates as it works.
**Check:** on a task with 4 independent subtasks, verify none is silently dropped.
**Explain:** run the same task without the todo list. Did anything get skipped? Why does
externalizing the plan help?

### Q6. Make it loop, then stop it
**Build:** a scenario that induces a loop (a tool that always fails the same way). Then add loop
detection based on repeated (tool, arguments) pairs.
**Check:** the uncapped version loops; detection stops it and reports the problem.
**Explain:** what did the uncapped run cost before you killed it?

### Q7. All the termination conditions
**Build:** success, iteration cap, token budget, wall clock, explicit give-up, loop detection —
each with a distinct exit status.
**Check:** write a test triggering each.
**Explain:** which was hardest to define, and why? (It's "success" — say why.)

### Q8. Honest failure
**Build:** give the agent an impossible task (data it has no tool to reach).
**Check:** does it say so, or invent an answer?
**Explain:** report what happened. What in the system prompt makes honest failure more likely?

### Q9. Verification
**Build:** a verification step — after declaring success, the agent (or your code) checks the actual
outcome: does the file exist, does the value match, does the test pass.
**Check:** on 10 runs, count how often declared success was real.
**Explain:** report the rate of *premature success* before verification. What does that mean for
agents you can't verify?

### Q10. Context exhaustion
**Build:** a task whose tool results are large enough to fill the window. Run without context
management.
**Check:** observe the failure.
**Explain:** describe exactly what went wrong. Now add a history budget and clearing of old tool
results (Topic 19), and report the difference.

### Q11. Compounding error
**Build:** inject a wrong result at step 2 (a tool that lies once).
**Check:** trace what the agent does for the rest of the run.
**Explain:** did it ever recover? What mechanism would let it?

### Q12. Tracing
**Build:** structured traces: per step — thought, tool, arguments, result summary, tokens, cost,
latency; per run — outcome, totals.
**Check:** save traces for 20 runs and answer, from traces alone: which step usually fails, what's
the average step count, where does the money go.
**Explain:** report those three answers.

### Q13. Twenty runs, honestly assessed
**Build:** run your agent on 20 varied inputs of the same task.
**Check:** categorize every outcome: succeeded and verified, succeeded unverified, failed honestly,
failed claiming success, hit a limit, looped.
**Explain:** report the distribution. Would you put this in front of a user? What single change
would most improve the numbers?

---

# Done when you can answer

1. What distinguishes an agent from a workflow?
2. When should you *not* build an agent?
3. Why interleave reasoning and action rather than planning it all first?
4. Name six termination conditions.
5. Why is defining "success" the hardest part?
6. Name four agent failure modes that are engineering problems, not model problems.
7. Why is tracing mandatory rather than nice to have?

Write answers in `notes.md`.
