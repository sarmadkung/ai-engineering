# Topic 34 — Multi-Agent Systems

**Why this topic:** several agents working together is the most over-applied idea in AI engineering
and occasionally exactly right. This topic teaches you to tell the difference, and to build the
version that works.

---

# Part 1 — Theory

## 34.1 When multiple agents actually help

The honest list of real reasons:

1. **Context isolation** — the strongest and most defensible reason. A subagent reads 50 documents
   and returns a 200-token summary; the main agent's context stays clean. This alone justifies most
   multi-agent designs.
2. **Parallelism** — independent subtasks run concurrently, cutting wall-clock time.
3. **Specialization** — different tools, prompts, permissions, or model sizes per role. A cheap
   model for extraction, an expensive one for judgement.
4. **Separation of concerns** — a reviewer that isn't the author gives a genuinely different
   perspective (Topic 31's critic).

And the bad reasons, which are common: it sounds sophisticated; org-chart cosplay ("CEO agent,
engineer agent"); or hoping that more agents will fix a prompt problem.

**The default should be one agent with good tools.** Multi-agent adds coordination overhead, cost
multiplication, harder debugging, and new failure modes. Reach for it when you can name which of the
four reasons above applies.

## 34.2 Roles and topologies

| Topology | Shape | Good for |
|---|---|---|
| **Supervisor** | one coordinator delegates to workers, workers report back | most real systems |
| **Pipeline** | fixed sequence of specialists | known procedure |
| **Peer/swarm** | agents hand off to each other freely | rarely justified |
| **Hierarchical** | supervisors of supervisors | very large tasks |
| **Blackboard** | agents read/write shared state | parallel contribution |

Supervisor is the workhorse: one place holds the goal and assembles results, so behaviour stays
predictable and failure is localized. Free-form peer topologies are where runaway loops and
mutual confusion live.

## 34.3 Communication

Agents "talk" by passing text or structured data. Design decisions:

- **Structured over prose.** A schema (Topic 18) for handoffs prevents the telephone game.
- **Full context or summary?** Passing everything defeats the context-isolation benefit; passing too
  little causes the worker to redo discovery. **Pass the task, the constraints, and only the
  necessary context** — a brief, not a transcript.
- **Task descriptions must be self-contained.** The most common multi-agent failure is a vague
  delegation ("research this") that the worker interprets differently than intended.
- **Return values must be structured** with status, result, and what couldn't be done.

Information loss compounds across hops. Two summarization steps and the original intent is gone —
which is a reason to keep topologies shallow.

## 34.4 Delegation

What the supervisor must decide: how to decompose the goal, which worker gets which subtask, what
context each needs, and whether subtasks can run in parallel.

Decomposition quality dominates everything else. Good subtasks are **independent** (no hidden
ordering), **verifiable** (you can tell if they succeeded), **bounded** (not "solve the problem"),
and **complete** (together they actually accomplish the goal). A supervisor that decomposes badly
cannot be rescued by excellent workers.

Practical guards: give each worker its own budget and iteration cap; make workers unable to delegate
further unless you intend hierarchy; and have the supervisor **verify** worker output rather than
trusting the claim of success (agents report success optimistically — Topic 29).

## 34.5 Coordination problems

The ones you will actually hit:

- **Duplicated work** — two workers do the same thing because the decomposition overlapped.
- **Conflicting output** — two workers produce incompatible results; somebody must resolve it, and
  that must be a defined role.
- **Hidden dependencies** — subtask B needed A's result, but they ran in parallel.
- **Cascading failure** — one worker's wrong output is accepted and built upon.
- **Cost explosion** — 5 workers × 20 steps × full context each. Multi-agent runs can cost 10–20×
  a single agent, which is the number that most often kills these designs in production.
- **Livelock** — agents handing work back and forth without progress.

Every one of these is a coordination-design problem, not a model problem.

## 34.6 Shared memory

Options: a shared store all agents read/write (a blackboard), message passing only, or a hybrid
(shared artifacts plus private scratch space).

Concerns that appear immediately: **write conflicts** (two agents editing the same artifact needs
locking or single-writer ownership), **staleness** (an agent acting on data another has changed), and
**visibility** (what should a worker be allowed to see, especially across tenants).

The simplest design that works: **single-writer ownership** — each artifact has exactly one agent
allowed to modify it, and others read. Avoids most conflicts without distributed-systems machinery.

## 34.7 Protocols

- **MCP** (Topic 32) — how agents access tools, not how they talk to each other.
- **A2A (agent-to-agent)** — emerging standards for agents from different systems to discover and
  call each other. Early; know the term.
- **In-practice** — inside one system, agents communicate by typed function calls over a shared
  state object. That's usually all you need, and it's testable.

---

# Part 2 — Questions to implement

Build `multiagent.py` here on top of Topic 33's orchestrator. Pick a task with genuinely separable
parts — e.g. "research 3 topics and produce a comparison", or "review this code for security,
performance and style".

### Q1. Single-agent baseline
**Build:** solve the task with one agent and good tools. Record success, steps, latency, cost.
**Explain:** these numbers are the bar. What specifically was hard for the single agent?

### Q2. Supervisor + workers
**Build:** a supervisor that decomposes, delegates to workers with fresh contexts, and assembles.
**Check:** same task, same metrics.
**Explain:** compare against Q1 on all four axes. Did quality improve? By how much did cost rise?

### Q3. The context-isolation measurement
**Build:** instrument main-agent context size for both designs on a research-heavy task.
**Check:** report peak context tokens for single-agent vs supervisor.
**Explain:** this is the core justification. Did it hold up on your task?

### Q4. Parallel workers
**Build:** run independent workers concurrently with a concurrency cap.
**Check:** wall time versus sequential; one worker failing doesn't lose the rest.
**Explain:** report the speedup. What limited it?

### Q5. Decomposition quality
**Build:** three decompositions of the same goal: good, overlapping, and incomplete.
**Check:** run all three.
**Explain:** report duplicated work and missed requirements for each. What does this say about where
to spend prompt effort?

### Q6. Structured handoffs
**Build:** typed task briefs and typed worker results (status, result, failures, sources).
**Check:** compare against free-text handoffs on 10 runs.
**Explain:** how many misinterpretations did structure eliminate?

### Q7. Information loss
**Build:** a 3-hop chain where each agent summarizes for the next. Compare the final output with the
original source material.
**Check:** identify what was lost or distorted.
**Explain:** report the loss. What does this imply about topology depth?

### Q8. Supervisor verification
**Build:** make the supervisor verify each worker's output instead of trusting it.
**Check:** count how often a worker claimed success incorrectly.
**Explain:** report the false-success rate. What happened downstream before verification existed?

### Q9. Conflicting results
**Build:** a scenario where two workers produce contradictory findings.
**Check:** observe the supervisor's behaviour, then implement explicit conflict resolution.
**Explain:** what did it do before? What's your resolution policy?

### Q10. Hidden dependency
**Build:** parallelize two subtasks where B actually needs A's result.
**Check:** observe the failure.
**Explain:** how would you detect such dependencies at decomposition time?

### Q11. Cost accounting
**Build:** track tokens and cost per agent, per run.
**Check:** report total cost for single-agent vs multi-agent on identical tasks.
**Explain:** report the multiple. At what task value does multi-agent become defensible?

### Q12. Shared memory with single-writer ownership
**Build:** a shared artifact store where each artifact has one owning writer; others read.
**Check:** an attempted conflicting write is rejected. Concurrent reads are fine.
**Explain:** what would concurrent writes have done to your artifact?

### Q13. Runaway prevention
**Build:** per-worker budgets and iteration caps, plus a global run budget; no worker may spawn
further workers.
**Check:** trigger each limit.
**Explain:** remove the limits and run a deliberately confusing task. What did it cost before you
stopped it? (Stop it.)

### Q14. The honest verdict
**Build:** one table comparing single-agent and multi-agent across quality, latency, cost, context,
lines of code, and debugging time.
**Explain:** which would you ship? Name the specific reason from §34.1 that applies — or admit that
none does.

---

# Done when you can answer

1. What are the four legitimate reasons for multiple agents?
2. Why is supervisor the default topology?
3. What makes a good subtask decomposition?
4. Why must handoffs be structured, and what should they contain?
5. Name five coordination failure modes.
6. Why does single-writer ownership avoid most shared-memory problems?
7. Roughly what cost multiple should you expect, and what does that imply?

Write answers in `notes.md`.
