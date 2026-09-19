# Topic 28 — Tool Design

**Why this topic:** the tools you expose define what your agent can do, how often it succeeds, and
what happens on its worst day. This topic is the discipline — boundaries, validation, permissions,
sandboxing, reliability — that makes agent tools safe to operate.

---

# Part 1 — Theory

## 28.1 Tool boundaries

The design question is granularity:

- **Too fine** — 40 tools, each trivial. Selection accuracy collapses and simple tasks take many
  round-trips.
- **Too coarse** — one `do_anything(instruction)` tool. The model can't reason about what it does,
  and you can't enforce anything.
- **Right** — one tool per *meaningful capability*, with parameters covering variation.

Heuristics that work:

- Name the tool after a **user intention**, not an implementation detail. `find_customer_orders`,
  not `query_orders_table_v2`.
- A tool should do **one thing completely**. Requiring three calls to accomplish one obvious
  operation is a design smell — and each extra round-trip is latency, cost, and a chance to fail.
- Merge tools that differ only by a parameter.
- Keep **read** and **write** separate even when they touch the same data, because they need
  different permissions and different approval rules.
- Design the tool for the *model's* convenience, not your API's shape. Your internal pagination
  scheme is not the model's problem — hide it.

## 28.2 Validation

Never trust the arguments. Validate in layers:

1. **Schema** — types, enums, required fields (Topic 18, plus strict mode).
2. **Semantic** — does this id exist? is this date range sane? is the amount within limits?
3. **Authorization** — may *this user's session* act on this resource? The model never decides
   this, and must not be able to influence it.
4. **Business rules** — does this violate a constraint (refund exceeding the order total)?

The critical principle: **the model is an untrusted client.** Not malicious by default, but
manipulable by anything in its context — a web page, a document, a user message. So your tool
implementation must enforce every rule it needs, exactly as if the arguments came from a public
HTTP endpoint. Authorization derived from the model's arguments (a `user_id` it supplies) is a
privilege-escalation bug; it must come from your session.

## 28.3 Permissions

Classify every tool:

| Class | Examples | Policy |
|---|---|---|
| read-only, safe | search, get, list | auto-allow |
| read-only, sensitive | read PII, read secrets | log, restrict, maybe approve |
| write, reversible | create draft, add comment | auto-allow with audit |
| write, hard to reverse | send email, charge card, delete | **require human approval** |
| destructive | drop data, deploy, spend money | approval + confirmation + limits |

Then: **least privilege** (the credential behind each tool has only what that tool needs),
**per-user scoping** (the agent acts *as* a user, with their permissions, never as an admin),
**budgets** (spending, call counts, rate limits per session), and **audit logs** (who, what, when,
which arguments, what result — for every call).

**Human-in-the-loop** design is part of this: show the human what will happen in plain language
including the actual arguments, make approval explicit, allow modification, and remember that
approval fatigue is real — ask only where it matters, or people will click through everything.

## 28.4 Sandboxing

Defence in depth for the tools that touch execution or the filesystem: process isolation
(containers or microVMs), no network by default with an explicit allowlist, a read-only filesystem
plus a writable scratch area, resource limits (CPU, memory, wall clock, disk, processes), no
credentials in the environment, and an ephemeral instance per session.

The mindset that matters: **assume the sandbox will be tested.** Not necessarily by an attacker —
by a confused model doing something drastic while trying to help. Your sandbox is what makes that
recoverable.

## 28.5 Reliability

Tools fail. An agent whose tools fail unpredictably is worse than no agent, because it fails
*plausibly*.

- **Timeouts on everything.** A hanging tool hangs the agent.
- **Retries for transient failures inside your wrapper** — the model shouldn't have to learn what
  a 503 is. But cap them, and never auto-retry a non-idempotent write.
- **Circuit breakers** — after N consecutive failures, stop calling and report the tool as
  unavailable, so one broken dependency doesn't consume the whole session.
- **Graceful degradation** — a clear "search is unavailable, I can't verify this" beats silent
  wrong answers.
- **Determinism where possible.** A tool returning different results for identical arguments makes
  debugging an agent nearly impossible.
- **Idempotency for writes.**

## 28.6 Documenting tools for the model

The description is a prompt (Topic 26) and deserves iteration. A good one states: what it does,
when to use it, when **not** to, what each parameter means with units and formats, what the result
looks like, and its limits ("returns at most 50 rows", "only orders from the last 90 days").

Write it for a capable colleague with no context. Then **test it**: build a set of requests where
the correct tool choice is known and measure selection accuracy. Descriptions are code — version
them, and evaluate changes rather than guessing.

## 28.7 Observability

Per call, log: tool name, arguments, result size, latency, success or error class, iteration index,
session id, and (for approvals) who approved. Then you can answer the questions that actually
arise: which tool fails most, which is slow, where do agents loop, which tool never gets used,
which arguments are most often invalid.

Per session, track: total calls, cost, duration, and outcome. This is the raw material for Phase 9's
agent evaluation — and a tool layer you can't observe is one you can't improve.

---

# Part 2 — Questions to implement

Refactor your Topic 26/27 tools into a proper framework here: `registry.py`, `validation.py`,
`permissions.py`, `audit.py`.

### Q1. Boundary audit
**Build:** list every tool you've built. For each: is it too fine, too coarse, or right? Propose a
consolidated surface.
**Check:** implement the consolidation.
**Explain:** how many tools before and after? Which merges were obvious in hindsight?

### Q2. Granularity, measured
**Build:** the same capability as 6 narrow tools and as 2 parameterized ones. Run 20 requests
through each.
**Check:** measure selection accuracy, round-trips per task, and tokens.
**Explain:** report all three. Which design won, and was it close?

### Q3. A tool registry
**Build:** a registry where a tool declares name, description, schema, permission class, timeout,
and handler — and generates the provider-format definition automatically.
**Check:** adding a tool takes one declaration; the provider payload is generated, not hand-written.
**Explain:** what bug class does generating the schema eliminate?

### Q4. Layered validation
**Build:** all four layers from §28.2 for one write-capable tool.
**Check:** a test for each layer, each rejecting a crafted bad input.
**Explain:** show a case that passes schema validation and fails semantic validation.

### Q5. The authorization attack
**Build:** a tool taking `user_id` as a model-supplied argument. Then craft a request that makes the
agent pass a *different* user's id.
**Check:** confirm you can read another user's data.
**Explain:** now fix it by deriving identity from the session instead. Why is the first version a
privilege-escalation bug rather than a prompt problem?

### Q6. Permission classes and approval
**Build:** classify every tool; auto-allow safe ones; require approval for destructive ones with a
plain-language prompt showing the real arguments.
**Check:** a destructive action cannot proceed without explicit approval, and a denial is reported
back to the model as a result it can reason about.
**Explain:** what did the agent do after being denied?

### Q7. Approval fatigue
**Build:** deliberately require approval for everything, then run a 20-step task.
**Explain:** count the prompts. What would a real user do by prompt 15? Where exactly would you draw
the line, and why?

### Q8. Budgets
**Build:** per-session limits: max tool calls, max spend, max wall-clock. Enforce them and report
clearly when hit.
**Check:** trigger each one.
**Explain:** how does the agent behave when it's told it's out of budget? Is that acceptable?

### Q9. Sandbox verification
**Build:** your strongest sandbox for code execution and file access.
**Check:** a test suite that *attacks* it: network egress, host file read, fork bomb, memory bomb,
infinite loop, writing outside the scratch area, reading environment variables.
**Explain:** report pass/fail for each. Which control stopped which attack?

### Q10. Timeouts and circuit breaker
**Build:** per-tool timeouts and a circuit breaker opening after 3 consecutive failures.
**Check:** simulate a dependency that hangs, then one that fails repeatedly.
**Explain:** what did the agent do in each case? How much time and money did the breaker save?

### Q11. Graceful degradation
**Build:** with a tool forcibly disabled, compare two behaviours: silent failure (empty result) and
explicit unavailability.
**Check:** record the agent's final answer in each case.
**Explain:** which produced a *confidently wrong* answer? Why is that the worse failure?

### Q12. Description evaluation
**Build:** a test set of 30 requests with known-correct tool choices, and a harness measuring
selection accuracy. Then iterate on your two worst descriptions.
**Check:** report accuracy before and after per tool.
**Explain:** which description change helped most? What was wrong with it?

### Q13. Audit trail and dashboard
**Build:** log every call (tool, arguments, result size, latency, outcome, session, approver) and a
small report: calls by tool, error rate by tool, p95 latency, unused tools, most common validation
failures.
**Check:** run 50 varied tasks and produce the report.
**Explain:** what surprised you? Which tool would you fix or delete first, on the evidence?

---

# Done when you can answer

1. How do you decide tool granularity?
2. What are the four validation layers, and which can a schema never do?
3. Why is the model an untrusted client, and what must never be derived from its arguments?
4. What are the permission classes, and which need a human?
5. What must a sandbox enforce, and why assume it will be tested?
6. What is a circuit breaker, and what does it protect?
7. Why is silent failure worse than reported unavailability?

Write answers in `notes.md`.

---

**Phase 6 is complete.** Your model can now act, safely. Phase 7 gives it goals, loops, and memory
— which is to say, makes it an agent.
