# Topic 32 — Agent Frameworks

**Why this topic:** you've now built by hand what these frameworks provide. That's the right order —
you can evaluate them instead of being shaped by them. This topic is about knowing what each one
actually does, and when writing it yourself is the better engineering decision.

---

# Part 1 — Theory

## 32.1 What frameworks provide, and what they cost

Provide: provider abstraction, a prepackaged agent loop, tool registration, memory implementations,
retrieval pipelines, tracing integrations, and streaming plumbing.

Cost: abstraction between you and the API (so the provider's newest feature may not be exposed), a
debugging layer to learn on top of the problem you're debugging, dependency churn in a
fast-moving ecosystem, and hidden prompts you didn't write but are being billed for.

The honest guidance: **frameworks are worth it for standard shapes and a fast start, and a liability
for anything unusual.** Many production systems end up as a thin, hand-written loop against the
provider SDK, using libraries for specific jobs (a text splitter, a vector client, a tracer) rather
than one framework for everything. Having built it yourself, you can tell which situation you're in.

## 32.2 LangChain

The best-known ecosystem. Core ideas:

- **Runnables / LCEL** — compose steps into pipelines with a shared interface (`invoke`, `stream`,
  `batch`), so streaming and batching come free.
- **Model and embedding wrappers** — swap providers behind one interface.
- **Document loaders, splitters, retrievers** — Phase 5's machinery, prebuilt. The text splitters
  and loaders are genuinely useful even if you use nothing else.
- **Tools and agents** — the loop, prepackaged.

Reality: enormous surface, rapid change, and multiple generations of API in tutorials — much of
what you'll find online is for a version that no longer exists. Its abstractions can obscure the
prompt actually being sent, which is exactly the thing you need to see when quality is wrong.

Use it for: document loading, splitting, and quick prototypes. Be careful with: the higher-level
agent abstractions, unless you've read what they send.

## 32.3 LangGraph

LangChain's answer to Topic 31's state machines, and the more interesting library.

You define a **graph**: nodes (functions or model calls), edges (transitions), conditional edges
(the routing decision), and a typed **state** object passed between nodes.

What it gives you that a hand-rolled loop doesn't, for free:

- **Checkpointing** — state persisted per step, so a run can be resumed after a crash.
- **Human-in-the-loop** — interrupt before a node, wait for approval, resume (Phase 8).
- **Time travel** — rewind to a previous state and take a different path. Genuinely useful for
  debugging agents.
- **Streaming** of state updates as the graph executes.
- **Cycles**, which is what makes it an agent framework rather than a pipeline tool.

This is the most defensible choice for production agents in the Python ecosystem, precisely because
it makes control flow explicit rather than hiding it.

## 32.4 LlamaIndex

RAG-first, and strongest exactly where Phase 5 was hard:

- Many data connectors (files, APIs, databases, SaaS).
- Sophisticated indexes: vector, summary, tree, knowledge graph, and **composable** indexes over
  multiple sources.
- Advanced retrieval out of the box: parent-child/auto-merging, sentence-window, recursive
  retrieval, reranking (Topic 25's techniques, implemented).
- Query engines and query-planning over multiple indexes.
- Agents, but retrieval remains its centre of gravity.

Use it when retrieval is the hard part of your system. Its built-in implementations of Topic 25's
advanced patterns will save you real time — and you'll understand what they do, having built them.

## 32.5 Model Context Protocol (MCP)

Not a framework — a **protocol**, and the most strategically important item in this topic.

The problem it solves: every agent needs tools, and every tool integration was being rewritten for
every agent. MCP standardizes the interface between a model host and a tool provider.

```
MCP server  — exposes tools, resources (readable data), prompts
MCP client  — an agent/host that connects and consumes them
transport   — stdio (local) or HTTP/SSE (remote)
```

Why it matters: one server (say, for your internal API) serves any MCP-capable client; the ecosystem
of ready-made servers (filesystem, databases, GitHub, browsers) becomes available to your agent
immediately; and tools become deployable, versioned units rather than code embedded in one app.

What it does *not* do: it doesn't make tools safe. Everything in Topic 28 still applies — an MCP
server has permissions, needs validation, and its results are untrusted input. A third-party MCP
server is third-party code with access to your agent's context, which deserves the scrutiny you'd
give any dependency, and more.

## 32.6 Choosing, and the exit strategy

| Need | Reasonable choice |
|---|---|
| Learning how agents work | hand-rolled (you did this) |
| Complex, stateful, resumable agents | LangGraph |
| Retrieval-heavy application | LlamaIndex |
| Document loading and splitting | LangChain pieces |
| Exposing tools to many clients | MCP server |
| Full control, minimal dependencies | provider SDK + your own loop |

Whatever you choose, keep an exit: put your prompts, tool logic and business rules in **your** code,
not in framework-specific constructs. Frameworks change; your domain logic shouldn't have to.

---

# Part 2 — Questions to implement

Reimplement work you've already done, so the comparison is real rather than theoretical. Install
`langchain`, `langgraph`, `llama-index`, and the MCP SDK.

### Q1. Your baseline, documented
**Build:** count the lines of code and dependencies in your hand-rolled agent (Topic 29/31) and
record its metrics on your 20-task set.
**Explain:** this is the thing frameworks must beat. On what axes would you accept losing?

### Q2. The same agent in LangChain
**Build:** reimplement it with LangChain's agent abstractions.
**Check:** same tasks, same metrics. Also capture the *actual prompt sent* (via a callback or
tracing).
**Explain:** compare lines of code, dependencies, and metrics. Was the prompt what you expected?
How many tokens did the framework add?

### Q3. The same agent in LangGraph
**Build:** as an explicit graph with typed state.
**Check:** same tasks, same metrics.
**Explain:** what became clearer than in your hand-rolled loop? What became more ceremonial?

### Q4. Checkpointing and resume
**Build:** add a checkpointer; kill the process mid-run; resume.
**Check:** it continues from the last completed node rather than restarting.
**Explain:** how much work would this have been by hand? Where would you have got it wrong?

### Q5. Human-in-the-loop interrupt
**Build:** interrupt before a destructive node, require approval, then resume.
**Check:** approve one run, reject one.
**Explain:** compare with your Topic 28 approval implementation. Which is better and why?

### Q6. Time travel
**Build:** rewind a completed run to an earlier state and take a different branch.
**Check:** both histories are inspectable.
**Explain:** debug one real failure this way. Did it help?

### Q7. RAG in LlamaIndex
**Build:** reimplement your Phase 5 pipeline with LlamaIndex.
**Check:** measure hit rate and answer accuracy against your own implementation, on the same
evaluation set.
**Explain:** report both. Where did the framework do better, and can you tell *why* from the code?

### Q8. Advanced retrieval, prebuilt
**Build:** use LlamaIndex's auto-merging/sentence-window retrieval.
**Check:** compare against your Topic 25 implementations.
**Explain:** did the prebuilt version beat yours? What did it do that you hadn't thought of?

### Q9. Build an MCP server
**Build:** expose 3 of your Phase 6 tools as an MCP server over stdio.
**Check:** connect with an MCP-capable client and call each tool.
**Explain:** what was involved? What did you have to decide that the protocol doesn't decide for
you?

### Q10. Consume MCP
**Build:** connect your agent to your own server *and* to a third-party one (filesystem or similar).
**Check:** the agent uses tools from both.
**Explain:** what permissions does the third-party server have? What could it see of your context?
Would you run it in production?

### Q11. Framework overhead, measured
**Build:** for one identical task, capture exact input tokens and latency for: hand-rolled,
LangChain, LangGraph.
**Check:** tabulate.
**Explain:** report the hidden-token cost per call and extrapolate to 1M calls. Is it material?

### Q12. Debugging under abstraction
**Build:** deliberately introduce the same bug (a tool returning malformed data) in your
hand-rolled agent and in a framework agent.
**Check:** time how long each takes to diagnose, and note how many layers you had to read.
**Explain:** which was faster to debug, and what does that imply for on-call?

### Q13. The decision
**Build:** a short recommendation document: for the kind of system you intend to build, what do you
use and why — including which pieces you keep hand-rolled.
**Explain:** defend it with your measured numbers. Name the one thing that would change your mind.

---

# Done when you can answer

1. What do frameworks give you, and what do they take?
2. What does LangGraph add over a hand-written loop?
3. What is LlamaIndex best at?
4. What problem does MCP solve, and what does it *not* solve?
5. Why is a third-party MCP server a security decision?
6. How do you keep an exit from a framework?
7. When is hand-rolling the better engineering choice?

Write answers in `notes.md`.

---

**Phase 7 is complete.** You can build, remember, structure and ship agents. Phase 8 makes them
robust, multi-agent, and long-running.
