# Topic 20 — AI Application Backend

**Why this topic:** an LLM feature is a backend with unusual properties — requests that take 30
seconds, responses that stream, costs per call, third-party rate limits, and non-deterministic
output. Ordinary web architecture needs adjusting for all five. This is the topic that turns
scripts into a service.

---

# Part 1 — Theory

## 20.1 What makes an AI backend different

| Normal API | LLM-backed API |
|---|---|
| 10–100 ms | 2–60 s |
| returns one payload | often streams |
| cost ≈ 0 per request | real money per request |
| your own DB is the bottleneck | a third-party rate limit is |
| deterministic | non-deterministic |
| errors are bugs | errors include "the model was wrong" |

Every design decision below follows from this table.

## 20.2 FastAPI and why async matters here

An LLM call is I/O-bound: your process waits seconds doing nothing. With synchronous handlers,
each waiting request occupies a worker, so a handful of users saturate the server. With async,
one process handles hundreds of concurrent waits.

This is the single biggest performance decision in the topic, and it's free if you get it right
from the start.

**The trap:** one blocking call inside an async handler blocks the entire event loop, stalling
every other request in the process. So: use the async SDK client, never the sync one; never call
`time.sleep` (use `asyncio.sleep`); and push genuinely CPU-bound work (PDF parsing, embedding
locally) to a thread or process pool. A single `requests.post` in an async handler will
mysteriously destroy your throughput.

FastAPI gives you async, typed request/response models via Pydantic (Topic 18's schemas doing
double duty), dependency injection for clients and sessions, and generated API docs.

## 20.3 Streaming over HTTP

Two mechanisms: **Server-Sent Events** (one-way, text, trivial to implement, reconnects
automatically — the right default) and **WebSockets** (bidirectional, needed for voice or
interruption).

Practical concerns that bite in production:

- **Errors mid-stream.** You've already sent a 200 and some text. You cannot change the status
  code, so your event protocol needs an explicit error event type, and the client must handle it.
- **Client disconnects.** The user closes the tab; you keep paying for tokens. Detect
  disconnection and cancel the upstream request.
- **Buffering proxies.** Nginx and some CDNs buffer responses by default, which silently
  destroys streaming. You have to disable it explicitly.
- **Save the full response** as you stream, or you have nothing to log or persist.

## 20.4 Authentication and multi-tenancy

Standard practice applies (API keys for machines, JWT/sessions for users), plus two things
specific to this domain:

- **Per-user quotas and rate limits.** Your provider limit is shared; one user's bulk job must
  not consume it. Enforce per-user concurrency and token budgets, not just request counts.
- **Cost attribution.** Every request must be logged against a user, or you cannot bill, cannot
  find the source of a spend spike, and cannot cut off abuse. This is a schema decision made on
  day one, painful to retrofit.

Never put the provider key in the client. An LLM proxy endpoint without auth is an open invoice.

## 20.5 Background jobs

Any operation longer than ~30 seconds — document ingestion, batch classification, long agent
runs — must not live in a request. HTTP clients, load balancers and browsers all time out.

The pattern: accept the work, return a job id immediately (202), process in a worker, let the
client poll or subscribe for status.

```
POST /jobs  -> 202 {job_id}
GET  /jobs/{id} -> {status: queued|running|done|failed, result?}
```

Queues: Celery or RQ with Redis, or Postgres-based queues for simplicity. Requirements that
matter for LLM work specifically: **idempotency** (a retried job must not double-charge you),
**visible progress** (long jobs need intermediate status), **bounded retries** with a dead-letter
destination, and **cancellation** (users abandon long jobs, and you should stop paying).

## 20.6 PostgreSQL

Your system of record. What an LLM app typically stores: users, conversations, messages,
requests (with token counts and cost), jobs, documents and their chunks, and evaluation runs.

Schema notes specific to this domain:

- **Store the whole request/response**, including model, parameters, prompt version, tokens, cost
  and latency. This table is how you debug quality, investigate cost, and build evaluation sets
  later. It's the highest-value table you'll have.
- **JSONB** for provider payloads and metadata that varies.
- **pgvector** for embeddings, so retrieval lives in the same database as everything else — the
  default choice in Phase 5 unless you have a reason otherwise.
- Conversations and messages need indexes on (user, created_at); they become your biggest tables.

## 20.7 Redis

Four distinct jobs, worth keeping distinct in your head:

- **Cache** — identical prompt → cached response. A semantic cache (embedding similarity) can go
  further, at the risk of serving a near-miss answer.
- **Rate limiting** — token buckets per user, atomically.
- **Queue broker** — for the background jobs above.
- **Pub/sub** — streaming a job's tokens to whichever web process holds the user's connection.
  Necessary as soon as you have more than one process.

Caching LLM responses is unusually effective because real traffic is repetitive — but decide your
TTL and whether the cache key includes the model, the parameters and the prompt version. It must.

## 20.8 Observability from day one

Log for every LLM call: request id, user, model, prompt version, input/output/cached tokens, cost,
latency, TTFT, stop reason, and whether it failed and why. Trace multi-step operations with a
shared id so a single user action can be reconstructed across calls (Phase 9 formalizes this).

Health checks should verify the provider is reachable, not just that your process is alive.

---

# Part 2 — Questions to implement

Build a small but real service in this folder. Install `fastapi`, `uvicorn`, `httpx`, `pydantic`,
`asyncpg`/`sqlalchemy`, `redis`. Docker Compose for Postgres and Redis is the easy path.

### Q1. Skeleton and health
**Build:** a FastAPI app with `/health` that also checks provider reachability, Postgres, and
Redis, reporting each separately.
**Check:** stop Redis and confirm the endpoint reports it as unhealthy without crashing.
**Explain:** why check dependencies separately rather than returning a single boolean?

### Q2. A typed chat endpoint
**Build:** `POST /chat` taking a Pydantic request model, calling the model, returning a typed
response including usage.
**Check:** an invalid body returns 422 with a useful message; a valid one returns the answer.
**Explain:** where else in this roadmap did you define schemas, and why is it the same tool?

### Q3. Prove the async trap
**Build:** two endpoints — one using a blocking client inside an async handler, one using the
async client. Fire 20 concurrent requests at each and measure total wall time.
**Check:** the blocking version is dramatically slower.
**Explain:** report both numbers and explain the mechanism precisely.

### Q4. Streaming
**Build:** `POST /chat/stream` with SSE, plus a minimal HTML page or curl command to consume it.
**Check:** tokens appear progressively. Measure TTFT versus the non-streaming endpoint.
**Explain:** what did you have to do to stop proxies or buffering from breaking it (or what would
you have to do in production)?

### Q5. Mid-stream errors and disconnects
**Build:** an explicit error event in your SSE protocol, and cancellation of the upstream call
when the client disconnects.
**Check:** kill the client mid-stream and confirm from your logs that the provider request was
cancelled, not left running.
**Explain:** why can't you just return a 500 when a stream fails halfway?

### Q6. Persistence
**Build:** Postgres tables for users, conversations, messages, and llm_requests (model, prompt
version, tokens, cost, latency, status). Persist every call.
**Check:** after 20 varied requests, query total cost per user and average latency per model.
**Explain:** which single column would you most regret not having added? Why?

### Q7. Multi-turn conversation
**Build:** `POST /conversations/{id}/messages` that loads history from Postgres, applies the
Topic 19 context budget, calls the model, and stores both messages.
**Check:** a 20-turn conversation stays coherent and input tokens stay bounded.
**Explain:** which Topic 19 strategy did you implement, and what does it give up?

### Q8. Auth and per-user limits
**Build:** API-key auth, and a Redis token-bucket limiter enforcing both requests/minute and
tokens/day per user.
**Check:** exceeding either returns 429 with a `retry-after`. One user hitting their limit does
not affect another.
**Explain:** why limit tokens as well as requests?

### Q9. Response caching
**Build:** a Redis cache keyed on (model, parameters, prompt version, messages hash), with a TTL.
**Check:** measure latency and cost for a repeated request, cached and uncached.
**Explain:** why must the model and prompt version be in the key? Give a concrete bug from
omitting each.

### Q10. Background jobs
**Build:** `POST /jobs` for a long task (classify 200 items), returning a job id; a worker; and
`GET /jobs/{id}` for status with progress.
**Check:** the endpoint returns immediately; progress advances; results are retrievable.
**Explain:** how did you make retries idempotent, and what would a non-idempotent retry cost you
in real money?

### Q11. Job cancellation
**Build:** `DELETE /jobs/{id}` that stops the work.
**Check:** verify from logs that no further provider calls are made after cancellation.
**Explain:** why is cancellation a cost feature as much as a UX feature?

### Q12. Tracing
**Build:** a request id generated per HTTP request, attached to every log line and every LLM call
record, including inside background jobs.
**Check:** pick one user action that caused 3 model calls and reconstruct the whole thing from
logs alone.
**Explain:** what would you still be unable to reconstruct? (Phase 9 answers this.)

### Q13. Load test
**Build:** hammer the service with 50 concurrent users for a minute.
**Check:** record p50/p95 latency, error rate, and total cost of the test.
**Explain:** what broke first — your service, the provider's rate limit, or the database? What
would you fix first, and why?

---

# Done when you can answer

1. Name five ways an LLM backend differs from an ordinary one.
2. Why is async essential, and what one mistake negates it?
3. How do you report an error that happens mid-stream?
4. Why must every request be attributed to a user?
5. When does work belong in a background job, and what must that job guarantee?
6. What do you store in Postgres, and what belongs in Redis?
7. What must a cache key include, and why?

Write answers in `notes.md`.

---

**Phase 4 is complete.** You can now build a real application on top of a hosted model. Phase 5
gives it access to your own data.
