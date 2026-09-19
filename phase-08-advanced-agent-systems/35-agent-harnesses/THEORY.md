# Topic 35 — Agent Harnesses

**Why this topic:** the harness is everything around the model — tools, context, permissions,
sandbox, state. Claude Code, Cursor and Devin are harnesses over models you can also call. Model
quality sets the ceiling; **the harness decides how much of it you get**. This topic is how to build
one.

---

# Part 1 — Theory

## 35.1 What a harness is

```
harness = execution environment + tool management + context management
        + permissions + persistent state + observability
```

Same model, different harness, wildly different capability. That's why two products built on the
same API feel decades apart. If you want to know where the engineering value is in agent products,
it's here.

## 35.2 Execution environments

Where the agent's actions actually happen:

- **In-process** — tools are Python functions in your app. Simple; no isolation; a bug or a bad
  action hits your service.
- **Container per session** — the standard. Isolated filesystem and network, disposable, resource
  limits (Topic 28).
- **MicroVM** (Firecracker, gVisor) — stronger isolation for untrusted code, slightly slower to
  start.
- **Remote/cloud sandbox** — a provider runs it; you don't operate infrastructure.

Lifecycle decisions matter as much as the technology: **per-session** environments (created, used,
destroyed) are the sane default; persistent ones accumulate state and drift; pooled warm
environments trade isolation for startup latency, which is a real product concern when cold start is
several seconds.

What the environment must provide: a working directory, the tools the agent needs installed,
credentials (scoped, never the agent's own), network policy, and resource limits.

## 35.3 Tool management

Beyond Topic 28's per-tool design, a harness must manage the tool *surface*:

- **Registration and discovery** — a registry, possibly dynamic per session or per user permissions.
- **Scaling to many tools.** Model accuracy degrades past ~20 tools and definitions consume input
  tokens on *every* call. Solutions: group tools, load definitions on demand (a tool-search
  mechanism), or expose a general interface instead (bash instead of 40 file operations — one tool
  with enormous surface, which is exactly why coding agents are built on it).
- **Versioning** — changing a tool's schema changes agent behaviour; treat it as an interface change.
- **Per-session filtering** — a read-only session should not be *offered* write tools, rather than
  being refused when it tries.

## 35.4 Context management in a harness

The harness owns the context window, and this is where harness quality shows most:

- **Budget allocation** — how much for system prompt, tools, history, tool results, memory, and the
  answer.
- **Compaction** — when history approaches the limit, summarize and continue (Topic 19). The
  transition must be seamless: the agent shouldn't lose its plan.
- **Tool result pruning** — old results are the bulk and the least useful. Clearing them is standard.
- **File/state references instead of content** — keep a path and a summary in context rather than a
  50k-token file. The agent re-reads on demand. This is the single most effective context technique
  in coding agents.
- **Persistent notes** — a file the agent maintains (plan, findings) that survives compaction. The
  cheapest form of durable memory, and it's inspectable by humans.
- **Caching-aware layout** — stable prefix first, or you pay full price every turn (Topic 16).

## 35.5 Permissions in a harness

Topic 28's classes, now session-scoped:

- **Modes** — read-only, ask-before-write, auto-approve-within-scope, full auto. Users should be able
  to choose, and the default should be cautious.
- **Scoping** — the agent acts within a directory, a project, a tenant, a user's own permissions.
- **Allowlists** — remembered approvals ("always allow `npm test`") to fight approval fatigue
  without granting everything.
- **Credentials** — provided by the environment, scoped minimally, never visible in context (which
  means never printable by a tool the model can call).
- **Audit** — every action, with who approved it.

The framing that helps: the harness is the **operating system** for the agent. It decides what
exists, what's permitted, and what's recorded.

## 35.6 Persistent state

Across a session and across sessions:

- **Working state** — plan, progress, findings. In state or in files.
- **Artifacts** — what the agent produced.
- **Memory** — Topic 30's stores.
- **Configuration** — user preferences, project conventions (this is what a `CLAUDE.md`-style file
  is: persistent instructions loaded into every session).
- **Checkpoints** — Topic 33's resumability.

A design worth noting: **files as state**. Using the filesystem for the agent's own notes and
artifacts is simple, inspectable, diffable, version-controllable, and survives compaction. It's what
most successful coding agents converge on.

## 35.7 Observability and the feedback loop

A harness must produce, per session: every step, tool calls and results, context size over time,
compaction events, approvals, cost, and outcome. Then a **replay** capability, so a failure can be
re-run and diagnosed.

Because the harness is where you can actually improve things: you cannot retrain the model, but you
can fix a tool description, a context policy, or a permission default — and measure it. Phase 9's
evaluation applied to your own harness is the loop that makes agents better over time.

---

# Part 2 — Questions to implement

Build a real harness in this folder: `harness/` with environment, registry, context, permissions,
state and tracing modules. Target task: a coding agent that can read, modify and test code in a
sandboxed project directory.

### Q1. Session lifecycle
**Build:** create a session (container or isolated temp directory), run an agent in it, destroy it.
**Check:** nothing persists after destruction except what you explicitly saved. Measure startup time.
**Explain:** is your startup latency acceptable for interactive use? What would pooling change?

### Q2. Environment isolation
**Build:** filesystem confined to a working directory, no network by default, resource limits.
**Check:** attack tests — read outside the root, open a socket, fork bomb, fill the disk.
**Explain:** report which control stopped each.

### Q3. Tool registry with per-session filtering
**Build:** a registry where a session's mode (read-only vs write) determines which tools are even
*offered*.
**Check:** in read-only mode the write tools are absent from the request payload, not just refused.
**Explain:** why is absence better than refusal?

### Q4. Tool count and token cost
**Build:** measure input tokens and selection accuracy with 5, 15, and 40 tools registered.
**Check:** tabulate.
**Explain:** report degradation and the per-call token overhead. What's your plan for 100 tools?

### Q5. General vs specific tools
**Build:** a task solved with 12 specific file tools, then with one `bash` tool.
**Check:** measure success rate, steps, and tokens.
**Explain:** which won? What did you give up with bash, and what did you gain?

### Q6. Context budget allocation
**Build:** explicit budgets per component, enforced, with a report of actual usage per turn.
**Check:** on a long session, no component can crowd out the rest.
**Explain:** which component wanted more than its share?

### Q7. Compaction that doesn't lose the plan
**Build:** compaction at a threshold, preserving the goal, the plan, and recent turns.
**Check:** run a 50-step session through several compactions and verify the agent still knows its
task.
**Explain:** what did a naive compaction lose? How did you fix it?

### Q8. References instead of content
**Build:** for file-reading tools, keep a path plus a summary in context instead of full content, and
let the agent re-read ranges on demand.
**Check:** compare peak context and total cost against inlining full files.
**Explain:** report both. When did the agent need to re-read, and was that cheaper overall?

### Q9. Persistent notes file
**Build:** a notes/plan file the agent maintains, reloaded after compaction.
**Check:** on a 4-subtask session with compaction in the middle, nothing is silently dropped.
**Explain:** compare with Topic 29's in-context todo list. What does the file add?

### Q10. Permission modes
**Build:** read-only, ask, and auto modes, plus a remembered allowlist for repeated safe commands.
**Check:** each mode behaves correctly; the allowlist reduces prompts on a real task.
**Explain:** count approval prompts per mode on the same task. Which default would you ship?

### Q11. Credential safety
**Build:** provide a credential via the environment, and attempt to make the agent reveal it (print
env, read the config file, echo it).
**Check:** it cannot.
**Explain:** how many paths did you have to close? Which one did you nearly miss?

### Q12. Project configuration
**Build:** a per-project instructions file loaded into every session (conventions, commands, don'ts).
**Check:** the agent follows a project-specific convention it could not have guessed.
**Explain:** how did behaviour differ without it?

### Q13. Tracing and replay
**Build:** full session traces and the ability to replay a session step by step.
**Check:** run 20 sessions, then diagnose one failure entirely from the trace, and replay it.
**Explain:** what was the root cause, and would you have found it without replay?

### Q14. Harness ablation
**Build:** the same model and task with (a) a bare loop and (b) your full harness.
**Check:** success rate, steps, cost, and context usage for both.
**Explain:** report the gap. Which single harness feature contributed most? This is the topic's
central claim — did your numbers support it?

---

# Done when you can answer

1. What are the six components of a harness?
2. Why is per-session environment lifecycle the sane default?
3. How do you handle a large tool surface?
4. Why keep references instead of content in context?
5. What survives compaction, and how?
6. Why offer fewer tools rather than refusing calls?
7. Why is the harness where you can actually improve agent quality?

Write answers in `notes.md`.
