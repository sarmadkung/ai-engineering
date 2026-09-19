# Project 6 — Tool-Using AI Agent

**Depends on:** Phase 6 (Topics 26–28), Phase 7 (Topics 29–31), Phase 10 for the parts that make it
safe.

**What you prove:** that you can give a model real capability in a real system without giving it the
ability to cause real damage. This is the project where security stops being theoretical.

---

# Part 1 — What you are building

An agent that accomplishes goals in a domain with actual side effects. Pick one with genuine stakes:

- a support agent that looks up orders, checks policy, and issues refunds within limits
- a data agent that queries a database, analyses results, and writes reports
- a devops agent that inspects logs and metrics and proposes (or applies) fixes
- a research agent that searches, reads, and produces a sourced brief

The requirement is that at least one action is **consequential** — it changes something or costs
something — because that's what forces the permissions, approval and audit work.

---

# Part 2 — Design decisions

| Decision | Options | Consider |
|---|---|---|
| Architecture | ReAct / router / planner-executor / state machine | Topic 31 — match it to your task |
| Tool granularity | many specific / few parameterized | selection accuracy versus flexibility |
| Write policy | read-only / approval / auto within limits | start restrictive |
| Identity | acts as the system / as the user | as the user, always, if possible (Topic 42) |
| Verification | none / check outcomes | check outcomes (Topic 36) |
| Failure behaviour | retry / escalate / report | all three, in a defined ladder |

---

# Part 3 — Milestones

### M1 — Tool registry (Topic 28)
Tools declared with name, description, schema, permission class, timeout and handler; provider payload
generated, not hand-written.
**Check:** adding a tool is one declaration. Permission class determines whether it's even offered.

### M2 — Read-only agent (Topics 26, 29)
The loop with read tools only, iteration and budget caps, full tracing.
**Check:** 20 realistic tasks. Report success, steps, cost, and tool selection accuracy.

### M3 — Tool descriptions, measured (Topic 26)
A test set of 30 requests with known-correct tool choices.
**Check:** measure selection accuracy; iterate on your two worst descriptions and re-measure.
**Explain:** report before and after per tool.

### M4 — Layered validation (Topic 28)
Schema, semantic, authorization (identity from the session — never from model arguments), and business
rules.
**Check:** a test per layer. Then craft an injection that tries to change the acting user — it must fail
at the database, not the prompt.

### M5 — Write actions with approval (Topics 28, 33)
A consequential tool behind approval: plain-language description with resolved arguments, approve /
reject-with-reason / edit, and durable pause so approval can happen minutes later in another process.
**Check:** approve one, reject one with feedback that the agent then uses, and resume across a process
restart.

### M6 — Limits (Topics 28, 42)
Per-session quantity and value caps, semantic limits (never exceed the order total), and a global budget.
**Check:** attempt to breach each. Report what the agent does when told it's out of budget.

### M7 — Verification and honest failure (Topics 29, 36)
Verify outcomes rather than claims; an escalation ladder (retry informed → different approach → human);
honest give-up.
**Check:** measure false-success rate before and after verification. Give it an impossible task and
confirm it says so.

### M8 — Security (Topics 41, 42)
Indirect injection through tool results, egress control, secrets unreadable, error sanitization, audit
trail.
**Check:** run a 15-attack red-team suite. Report which control stopped each and what's still open.

### M9 — Evaluation (Topic 39)
20 tasks with objective verifiers, 3 runs each, outcome categories, failure taxonomy, cost per successful
task.
**Check:** the full report. Fix the top failure category and re-measure.

### M10 — Operations (Topics 33, 40)
Durable state with resume, cancellation, per-run traces, kill switch, and a dashboard of runs with
outcomes and cost.
**Check:** kill a run mid-way and resume it. Answer "what did session X touch?" from the audit trail
alone.

---

# Part 4 — Experiments

1. **Architecture comparison** — the same tasks under ReAct, router, and planner-executor (Topic 31).
2. **Tool count** — 5 versus 20 tools available. Selection accuracy and token cost.
3. **General versus specific tools** — one parameterized search versus four narrow ones.
4. **Verification value** — false-success rate with and without outcome checking.
5. **Approval policy** — approve-everything versus approve-consequential-only. Count prompts and measure
   what a user would tolerate.
6. **Model comparison** — reasoning versus standard model on the same suite (Topic 53).

---

# Part 5 — Deliverables

- The agent, working against a realistic (ideally seeded, resettable) environment
- `EVALUATION.md` — suite, outcomes, failure taxonomy, cost per successful task
- `SECURITY.md` — threat model, red-team results, remaining risks, deployment-ladder position
  (Topic 42)
- `ARCHITECTURE.md` — architecture choice with the evidence for it
- Audit log samples and run traces

---

# Part 6 — Done when

- It reliably completes tasks with verified outcomes, and you know the rate.
- A fully compromised agent could not cause unacceptable harm, and you've written down why.
- Consequential actions cannot happen without an approval a human actually understands.
- You can answer "what did it do?" for any past run.
- You can state which rung of Topic 42's deployment ladder it's ready for.

---

# Stretch

MCP server for the tools so other clients can use them (Topic 32); multi-step workflows with dependencies
(Topic 36); learning from approvals — building an allowlist from patterns a human repeatedly approved
(Topic 55); a web UI for the approval queue.
