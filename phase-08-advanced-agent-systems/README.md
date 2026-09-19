# Phase 8 — Advanced Agent Systems

**Goal:** agents that survive crashes, wait for humans, run for hours, coordinate, and verify their own
work. This is where agents stop being demos.

---

## What I will do

| # | Topic | What I build | Status |
|---|---|---|---|
| 33 | [Agent Orchestration](33-agent-orchestration/) | `orchestrator.py` — durable state, checkpoints, resume, HITL | ready |
| 34 | [Multi-Agent Systems](34-multi-agent-systems/) | `multiagent.py` — supervisor and workers, honestly compared | ready |
| 35 | [Agent Harnesses](35-agent-harnesses/) | `harness/` — environment, tools, context, permissions, state | ready |
| 36 | [Autonomous Workflows](36-autonomous-workflows/) | `autonomous.py` — decomposition, verification, retry, reporting | ready |

## Which things to learn

**33. Agent Orchestration** — explicit serializable state and transitions instead of mutation;
checkpointing and the double-side-effect bug; deterministic replay; human-in-the-loop as **durable pause**;
cooperative cancellation and mid-run steering; the recovery matrix and retrying at the right layer;
partial success as the normal outcome, reported usefully.

**34. Multi-Agent Systems** — the four legitimate reasons (context isolation first) and the bad ones;
supervisor as the default topology; structured handoffs and self-contained task briefs; decomposition
quality dominating everything; the coordination failure modes; single-writer shared memory; and the cost
multiple — usually 3–10× — that decides most of these designs in practice.

**35. Agent Harnesses** — that **the harness decides how much of the model's capability you get**;
execution environments and session lifecycle; managing a large tool surface; context policy (references
instead of content, compaction that preserves the plan, persistent notes); permission modes and
allowlists; files as state; replay as the debugging tool.

**36. Autonomous Workflows** — **verification, not planning, is what makes autonomy work**; decomposition
as data; levels of verification and why coding agents have it easy; informed retry and escalation
ladders; the reliability arithmetic (0.95^10 ≈ 60%) and how verification breaks the multiplication;
honest reporting; and where the human checkpoints belong.

## Prerequisites

Phase 7. Postgres for durable state. Docker for harness isolation. A task domain with **objective
verification** — code with tests is strongly recommended for Topic 36.

**Next:** Phase 9 — proving any of this works.
