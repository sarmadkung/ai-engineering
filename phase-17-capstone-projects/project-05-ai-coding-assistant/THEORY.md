# Project 5 — AI Coding Assistant

**Depends on:** Phases 6–7 (Topics 26–32), Topic 35, Topic 36.

**What you prove:** that you can build a harness. Coding is the domain where agents work best, because
verification is free — tests, compilers and type checkers give you the objective signal that Topic 36
says everything depends on.

---

# Part 1 — What you are building

An agent that reads a codebase, makes changes, and verifies them:

```
task -> explore (read, search) -> plan -> edit -> run tests -> fix -> report
```

Scope it to one thing it does well: fix failing tests, implement a function from a docstring, add test
coverage, or perform a mechanical refactor. A narrow agent that works beats a general one that doesn't.

---

# Part 2 — Design decisions

| Decision | Options | Consider |
|---|---|---|
| Tool surface | bash only / specific file tools / both | bash is powerful and huge; specific tools are safer |
| Edit mechanism | whole-file rewrite / search-replace / patch | whole-file wastes tokens and loses unrelated changes |
| Isolation | git worktree / container / direct | never let it edit your real working tree unsandboxed |
| Context strategy | inline files / references + re-read | references scale (Topic 35) |
| Verification | tests / types / lint / all | more signal is better |
| Autonomy | approve each edit / approve the plan / full auto | start restrictive |

The edit mechanism matters more than it looks: search-replace or patches are token-efficient and
preserve surrounding code, while whole-file rewrites silently drop things and cost a fortune on large
files.

---

# Part 3 — Milestones

### M1 — Sandboxed environment (Topics 28, 35)
A git worktree or container per session, with the agent confined to it. Resource limits, no network by
default.
**Check:** path traversal, writes outside the root, and network egress all fail. Your real repository is
untouchable.

### M2 — Read and explore tools (Topic 27)
Read file (with line ranges), list directory, grep/search, and a repository-structure overview.
**Check:** the agent can answer "how does X work in this codebase?" using only these tools. Report tokens
consumed.

### M3 — Edit tools
Search-replace or patch application, with exact-match validation that fails loudly rather than guessing.
**Check:** an edit whose target text doesn't match is rejected with a clear error the agent can act on.

### M4 — Verification (Topic 36)
Run tests, type checker and linter as tools, returning structured results the agent can parse.
**Check:** a failing test produces an actionable error message; the agent fixes it on the next turn.

### M5 — The agent loop (Topics 29, 35)
Goal, tools, iteration cap, budget cap, loop detection, honest failure. Full tracing of every step.
**Check:** 20 real tasks. Report verified success, false success, honest failure, and limit hits.

### M6 — Context management (Topics 19, 35)
References plus summaries rather than inlined files; clear old tool results; compaction that preserves
the goal and plan; a persistent notes/plan file.
**Check:** a 40-step session with compaction still knows its task and doesn't drop subtasks. Report peak
context tokens with and without these measures.

### M7 — Verification-driven correction (Topic 36)
On test failure, feed the actual error into the retry; detect identical repeated failures and change
approach.
**Check:** compare success rate with informed retry versus naive re-run. Report both.

### M8 — Permissions (Topics 28, 42)
Read auto-allowed; edits shown as diffs for approval (or auto-approved within scope); destructive
commands blocked or confirmed; secrets unreadable.
**Check:** attempt to read `.env`, run `git push`, and `rm -rf`. Report what stopped each.

### M9 — Evaluation (Topic 39)
A fixed suite of 20 tasks with deterministic verifiers, run 3× each, with traces archived.
**Check:** report success rate, steps, cost per successful task, and a failure taxonomy.

### M10 — Report and diff output
Final output: what changed (as a diff), what was verified and how, what failed and why, cost.
**Check:** a human can review the work without reading the trace.

---

# Part 4 — Experiments

1. **Bash versus specific tools** — same tasks, both surfaces. Success rate, steps, tokens (Topic 35).
2. **Edit mechanism** — search-replace versus whole-file rewrite. Tokens, and how often unrelated code was
   damaged.
3. **Verification value** — with and without running tests. Report the false-success rate in each.
4. **Context strategy** — inline files versus references. Peak context, cost, success.
5. **Model comparison** — a reasoning model versus a standard one (Topic 53). Which failure modes
   disappear?
6. **Autonomy levels** — approve-each-edit versus approve-plan versus full auto. Success per unit of human
   attention (Topic 36, Q14).

---

# Part 5 — Deliverables

- The agent, runnable against a repository in an isolated worktree
- `EVALUATION.md` — the task suite, results, failure taxonomy
- `ARCHITECTURE.md` — harness design and tool surface
- `RESULTS.md` — success rates, cost per successful task, experiment tables
- `SECURITY.md` — sandbox verification results
- Traces for 20 runs

---

# Part 6 — Done when

- It completes real tasks in your own codebase with tests passing as proof.
- False-success rate is measured and low.
- It cannot escape its sandbox, read secrets, or touch your real working tree.
- Cost per successful task is known.
- You would let it work on a branch unsupervised — and can say why.

---

# Stretch

Multi-file refactors; PR creation with a written description; codebase-wide understanding via RAG
(Phase 5) over the repo; a review mode that only comments; MCP server exposure so other clients can use
your tools (Topic 32).
