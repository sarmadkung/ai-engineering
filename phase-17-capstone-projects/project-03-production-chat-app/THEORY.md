# Project 3 — Production AI Chat Application

**Depends on:** Phase 4 (Topics 16–20), Topic 19, plus Phases 9–10 for the parts that make it
production-grade.

**What you prove:** that you can ship a real multi-user AI product — auth, streaming, persistence,
cost control, abuse resistance, and observability. This is the project most directly comparable to
professional work.

---

# Part 1 — What you are building

A web chat application with real users:

```
browser (streaming UI)
  -> FastAPI (auth, rate limits, quotas)
  -> model gateway (caching, budgets, fallback)
  -> provider
Postgres (users, conversations, messages, request logs) + Redis (cache, limits, pub/sub)
```

Features: sign-in, multiple conversations per user, streaming responses, conversation titles, message
history, per-user quotas, and an admin view of cost and usage.

---

# Part 2 — Design decisions

| Decision | Options | Consider |
|---|---|---|
| Streaming transport | SSE / WebSocket | SSE is simpler and sufficient unless you need bidirectional |
| Frontend | server-rendered + HTMX / React | SSE consumption is easy in both |
| Auth | sessions / JWT / a provider | you need per-user attribution either way |
| History strategy | full / window / summary+window | cost versus continuity (Topic 19) |
| Multi-tenancy | none / per-user / per-org | decide now; retrofitting is painful (Topic 47) |
| Model routing | one model / tier by request | cost versus quality |

---

# Part 3 — Milestones

### M1 — Backend skeleton (Topic 20)
FastAPI, async throughout, Postgres and Redis via Compose, health checks that verify dependencies
separately.
**Check:** 20 concurrent requests are handled concurrently — prove it with timings against a blocking
version.

### M2 — Schema
Users, conversations, messages, `llm_requests` (model, prompt version, tokens in/out/cached, cost,
latency, status, user).
**Check:** you can answer "cost per user this week" and "p95 latency by model" in SQL.

### M3 — Streaming chat
SSE endpoint, token-by-token to the browser, full response persisted as it streams.
**Check:** TTFT under 1 s. Kill the browser mid-stream and confirm from logs that the upstream request
was cancelled — you should not pay for abandoned generations.

### M4 — Conversations and context (Topic 19)
Load history, apply a token budget, summarize or window when it grows, generate titles automatically.
**Check:** a 50-turn conversation stays coherent with bounded input tokens. Report tokens per turn.

### M5 — Auth and quotas (Topics 20, 48)
Sign-in, per-user rate limits on **requests and tokens**, daily spend caps, clear 429s with
`retry-after`.
**Check:** one user hitting a limit doesn't affect others. A single 100k-token request is caught by the
token limiter.

### M6 — Model gateway (Topic 47)
All model calls through one module: provider abstraction, retries with backoff, timeouts, fallback,
caching, cost accounting, kill switch.
**Check:** no route imports a provider SDK. Switch providers by config alone.

### M7 — Reliability (Topics 16, 43)
Retry the retryable classes only, handle mid-stream errors with an explicit error event, degrade to a
cheaper model on overload, and show honest errors in the UI.
**Check:** simulate rate limits, provider 500s, and timeouts. The user always gets something truthful.

### M8 — Security (Topics 41, 42)
Input length caps, injection-aware system prompt structure, output filtering for exfiltration channels,
per-user cache scoping, and no secrets in prompts.
**Check:** run your own attack suite — direct injection, cross-user cache leakage, cost attacks. Report
what was blocked.

### M9 — Observability (Topic 40)
Trace id per request through every call and log line; dashboard for usage, cost with projection, latency
percentiles, error rates, and quality proxies (refusals, truncations, regenerations).
**Check:** reconstruct one user's session end to end from logs. Trigger a cost alert deliberately.

### M10 — Deploy (Topic 49)
Containerized, deployed, TLS, secrets from a store, CI with tests and an evaluation gate.
**Check:** a deliberately bad prompt change is blocked by CI. Roll back in one command.

---

# Part 4 — Experiments

1. **Async proof** — throughput with blocking versus async clients at 50 concurrent users.
2. **Context strategies** — full history versus summary+window over 50 turns: cost, coherence, latency.
3. **Cache economics** — hit rate and saving for prompt caching and exact-response caching on realistic
   traffic.
4. **Load test** — 100 concurrent users for 10 minutes. Report p50/p95/p99, error rate, total cost, and
   what saturated first.
5. **Cost per active user per month** — measure it, then project it at 1,000 users. Is the product
   viable?

---

# Part 5 — Deliverables

- A deployed, working application others can sign into
- `ARCHITECTURE.md` with a diagram and decision record
- `RESULTS.md` — load test numbers, cost per user, cache hit rates, latency percentiles
- `SECURITY.md` — your threat model, attack suite results, and remaining risks
- Admin dashboard screenshots

---

# Part 6 — Done when

- Multiple real people can use it at once without interfering with each other.
- You know your cost per user and can prove it from your own logs.
- Abandoned streams stop costing money.
- Your attack suite runs and you can state what's still open.
- A deploy can be rolled back in one command, and CI blocks a quality regression.

---

# Stretch

File uploads with vision (Topic 60); voice input and output (Topic 61); RAG over user documents
(Project 4); shared/team conversations with proper tenant isolation; usage-based billing.
