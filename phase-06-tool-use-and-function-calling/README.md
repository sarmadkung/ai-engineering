# Phase 6 — Tool Use & Function Calling

**Goal:** give a model the ability to act — and the discipline to do it safely. This is the single
mechanism that turns a language model into an agent, so everything in Phases 7 and 8 stands on it.

---

## What I will do

| # | Topic | What I build | Status |
|---|---|---|---|
| 26 | [Tool Calling](26-tool-calling/) | `tools.py`, `loop.py` — the protocol and the agentic loop, by hand | ready |
| 27 | [External Tools](27-external-tools/) | `tools/` — web, database, API, filesystem, code execution, browser | ready |
| 28 | [Tool Design](28-tool-design/) | `registry.py`, `validation.py`, `permissions.py`, `audit.py` | ready |

## Which things to learn

**26. Tool Calling** — that the model only ever *requests* and **your code executes** (the basis of every
security property later); tool definitions, with the description as a prompt; the agentic loop and what
must be appended each iteration; parallel calls in one message; tool selection and why it degrades with
tool count; results as context; **errors as information the model can act on**; why "no results" isn't an
error.

**27. External Tools** — web search and untrusted retrieved content; three designs for database access
and what must guard generated SQL; API tools with credentials held in your code and idempotent writes;
filesystem tools and path traversal; sandboxed code execution; browser automation as a last resort;
designing a small, coherent tool surface.

**28. Tool Design** — granularity; the four validation layers; **the model as an untrusted client**, so
identity comes from the session and never from its arguments; permission classes and human approval
(without approval fatigue); sandboxing as a boundary you assume will be tested; timeouts, circuit
breakers, graceful degradation; descriptions as code, evaluated rather than guessed; per-call audit.

## Prerequisites

Phase 4 (Topic 18's schemas do double duty as tool schemas). Docker for sandboxing. A database you can
safely break — seed it, don't use anything real.

**Safety note:** everything you attack in this phase is your own. Sandboxes are built before the tools
that need them, not after.

**Next:** Phase 7 — goals, loops and memory: agents.
