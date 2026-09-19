# Phase 7 — AI Agents

**Goal:** build agents that work — with memory, structure, and honest limits. Also to learn when *not* to
build one, which is most of the time and the judgement that separates good engineers here from
enthusiastic ones.

---

## What I will do

| # | Topic | What I build | Status |
|---|---|---|---|
| 29 | [Agent Fundamentals](29-agent-fundamentals/) | `agent.py` — the loop with every termination condition and tracing | ready |
| 30 | [Agent Memory](30-agent-memory/) | `memory.py` — conversation, semantic, episodic memory | ready |
| 31 | [Agent Architectures](31-agent-architectures/) | ReAct, router, planner/executor, reflection, state machine | ready |
| 32 | [Agent Frameworks](32-agent-frameworks/) | the same agent in LangChain, LangGraph, LlamaIndex, plus an MCP server | ready |

## Which things to learn

**29. Agent Fundamentals** — agent versus workflow, and when a workflow is the right answer; the
observe-think-act loop; interleaved reasoning; planning styles and the todo list as a middle ground; the
six termination conditions (and why defining "success" is hardest); the failure modes that are
*engineering* problems — compounding errors, loops, premature success, context exhaustion; why tracing is
mandatory.

**30. Agent Memory** — that there is no model memory, only storage plus retrieval; short-term context
management; semantic, episodic and procedural memory; the five conversation-memory patterns; retrieval by
relevance, recency and importance; **writes must retrieve first**, or contradictions accumulate; why a
wrong memory is worse than none; what deletion actually requires.

**31. Agent Architectures** — ReAct and its weaknesses; routers (the most underrated pattern);
planner/executor with plan approval; reflection, and why self-critique without grounding is unreliable;
critic systems, with tool critics as the strongest; state machines for invariants a prompt can't
guarantee; subagents as a context-management technique.

**32. Agent Frameworks** — what frameworks give and take; LangChain's useful pieces; LangGraph's
checkpointing, human-in-the-loop and time travel; LlamaIndex for retrieval-heavy systems; **MCP** as a
protocol (and why a third-party MCP server is a security decision); keeping an exit from any framework.

## Prerequisites

Phase 6. For Topic 32: `langchain`, `langgraph`, `llama-index`, the MCP SDK. Postgres for memory.

**A recurring theme:** measure the agent against the simpler thing it replaced. Several topics here end
with "would you ship it?" and the honest answer is sometimes no.

**Next:** Phase 8 — making agents survive contact with reality.
