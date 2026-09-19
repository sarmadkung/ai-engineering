# Project 2 — LLM-Powered CLI Assistant

**Depends on:** Phase 4 (Topics 16–20), plus Topic 26 for tools.

**What you prove:** that you can build a real, pleasant, cost-aware tool on a hosted model — the
smallest complete product in this roadmap, and the one you'll actually use daily.

---

# Part 1 — What you are building

A command-line assistant with: streaming answers, multi-turn conversations that persist, piped input,
a few tools, configurable models, and honest cost reporting.

```
$ ai "explain what this error means" < error.log
$ git diff | ai "write a commit message"
$ ai chat                      # interactive session, resumable
$ ai cost --today
```

The point is that it's *good to use*: fast to start, streams immediately, remembers context, and never
surprises you with a bill.

---

# Part 2 — Design decisions

| Decision | Options | Consider |
|---|---|---|
| Language | Python / Go / Rust / TypeScript | startup time matters a lot for a CLI |
| Conversation storage | SQLite / JSON files | SQLite for search and history |
| Config | file + env + flags | precedence order must be obvious |
| Model default | cheap vs capable | you'll run this constantly; default matters |
| Tool scope | none / read-only / shell access | shell access is powerful and dangerous |
| Output | plain / markdown rendering | terminal markdown is nice, adds latency |

Startup latency is the quality signal users feel first. If your CLI takes 800 ms to print its first
token because of imports, nobody will use it — including you.

---

# Part 3 — Milestones

### M1 — One-shot query (Topic 16)
`ai "question"` → streamed answer. API key from environment or config.
**Check:** first token appears in well under a second. Measure and record TTFT.

### M2 — Piped input
Read stdin when present; combine with the prompt argument. Handle large input by warning about token
count rather than silently sending it.
**Check:** `cat file | ai "summarize"` works; a 500 KB file is refused or truncated with a clear message.

### M3 — Conversations (Topics 19, 20)
Persistent sessions with history, `ai chat` interactive mode, `--continue` to resume the last
conversation, and a context budget so long conversations don't grow without bound.
**Check:** a 30-turn conversation stays coherent and input tokens stay bounded. Show the numbers.

### M4 — Configuration and models
Config file, env vars, and flags with documented precedence. Multiple model presets (`--fast`,
`--smart`). A system prompt you can set per profile.
**Check:** switching models is one flag; the active model is visible in verbose mode.

### M5 — Cost tracking (Topics 16, 40)
Log every call: model, tokens in/out/cached, cost, latency. `ai cost` reports today, this week, by
model.
**Check:** your reported cost matches the provider's dashboard closely.

### M6 — Prompt caching (Topic 16)
Structure requests so the stable prefix (system prompt, any included files) is cacheable.
**Check:** report cache hit rate and cost saving over 20 similar calls.

### M7 — Tools (Topic 26)
2–4 tools: read a file, list a directory, run a shell command (with confirmation), search the web.
Approval for anything with side effects (Topic 28).
**Check:** the assistant can answer "what's in this project?" by reading files. A destructive command
requires explicit confirmation showing the exact command.

### M8 — Polish
Errors that make sense (no stack traces), retries with backoff, `--verbose`, `--json` for scripting,
shell completion, and a real `--help`.
**Check:** unplug your network and run it. The error message should tell a user what to do.

---

# Part 4 — Experiments

1. **Model comparison** — run 20 of your real queries through three model tiers. Compare quality
   (your judgement, written down), latency, and cost. Which should be the default?
2. **Caching impact** — cost per call with and without a cacheable prefix, over a realistic session.
3. **Context strategy** — full history versus summary+window on a long conversation. Report tokens and
   whether coherence held.
4. **Startup latency** — profile it. Lazy-import the heavy dependencies and measure the improvement.
5. **Tool value** — 10 tasks with and without tools. Where did tools actually help?

---

# Part 5 — Deliverables

- Installable CLI (`pipx install .` or a single binary)
- `README.md` with real usage examples
- `DECISIONS.md` — your design choices
- `RESULTS.md` — TTFT, startup time, cost per call, cache hit rate, model comparison
- Your own week of usage logs, summarized

---

# Part 6 — Done when

- You use it daily without wishing it were faster.
- Cost is visible and your reported numbers match the provider's.
- A destructive tool call cannot happen without you seeing exactly what will run.
- It fails gracefully with no network, a bad key, a rate limit, and an oversized input.
- Someone else can install it from your README.

---

# Stretch

Shell-history awareness; project-aware context (read a config file for conventions); a `--diff` mode
that proposes edits as patches you approve; local model support via an OpenAI-compatible base URL
(Phase 11); MCP client support (Topic 32).
