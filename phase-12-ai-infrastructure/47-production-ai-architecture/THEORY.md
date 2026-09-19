# Topic 47 — Production AI Architecture

**Why this topic:** you've built the parts. This topic is how they fit together as a system that a
team can operate, evolve and afford — and the architectural decisions that are expensive to change
later.

---

# Part 1 — Theory

## 47.1 The shape of a real AI system

```
client
  -> API gateway (auth, rate limit, routing)
  -> application service (business logic, orchestration)
  -> model gateway (provider abstraction, caching, budgets, fallback)
  -> models (hosted APIs and/or self-hosted inference)

supporting: Postgres (+pgvector), Redis, object storage, queue + workers,
            observability, evaluation pipeline
```

Two properties distinguish it from an ordinary web system: **the expensive dependency is external and
per-request priced**, and **quality is a runtime property you must measure continuously** (Phase 9).
Most of the architecture below exists to control those two facts.

## 47.2 AI services: where to draw boundaries

Options, with honest trade-offs:

- **Monolith** — everything in one service. Simplest, right for most teams, and the correct default.
- **Separate inference service** — self-hosted models behind their own API (Topic 46). Justified,
  because model serving has different hardware, scaling and deployment needs than your web tier.
- **Separate ingestion/indexing service** — long-running, bursty, CPU-heavy work (Phase 5) that you
  don't want competing with request serving.
- **Separate agent workers** — long-running stateful jobs (Topic 33) with different timeout and
  restart semantics.

The principle: split along **resource and lifecycle boundaries** (GPU vs CPU, request-scoped vs
long-running, bursty vs steady), not along conceptual ones. Splitting by concept gives you
distributed-systems problems with no operational benefit.

## 47.3 The model gateway

The single most valuable piece of AI-specific infrastructure, and the one people add too late. One
internal chokepoint through which every model call passes, giving you:

- **Provider abstraction** — swap models and providers without touching application code (Topic 43's
  fallback becomes trivial).
- **Centralized credentials** — keys in one place, rotatable.
- **Caching** — response and semantic caching in one implementation (Topic 20).
- **Rate limiting and budgets** — per user, per feature, per tenant.
- **Retries, timeouts, fallbacks** — implemented once, correctly (Topic 16).
- **Observability** — every call logged with cost and tokens, automatically (Topic 40).
- **A/B and canary routing** — send 5% of traffic to a new model or prompt version.
- **A kill switch** per feature.

Build it yourself (a thin library or service) or adopt a proxy (LiteLLM, Portkey, or a cloud gateway).
Either way: **no application code should call a provider SDK directly.** That one rule makes model
migration, cost control and evaluation possible; violating it means every future change is a
codebase-wide edit.

## 47.4 Queues and async work

Anything longer than a request belongs in a queue (Topic 20): ingestion, batch processing, long agent
runs, evaluation runs, and any retry-heavy work.

AI-specific requirements: **idempotency** (a retried job must not double-spend), **priority**
(interactive work ahead of batch backfills), **rate-limit awareness** (workers must share the
provider's budget, so a global limiter beats per-worker limits), **progress reporting**, and
**cancellation** (users abandon; you should stop paying).

Use the **Batch API** (Topic 16) for anything that can wait — roughly half price for the same work is
the cheapest optimization available to you.

## 47.5 Caching layers

Distinct caches with distinct behaviour:

| Cache | Key | Hit rate | Saves |
|---|---|---|---|
| provider prompt cache | stable prefix | high with stable prompts | most input cost |
| exact response cache | model + params + prompt version + messages | low–medium | whole call |
| semantic cache | embedding similarity | medium | whole call, with risk |
| embedding cache | text hash | high | embedding calls |
| retrieval cache | query + filters | medium | search latency |

Two cautions. **Semantic caching serves a *different* question's answer** when similarity is
mis-thresholded — great for FAQs, dangerous for anything specific or personalized; always include
tenant/user scope in the key (Topic 41's cache leakage). And every cache key must include the model
and prompt version, or a deploy silently serves stale behaviour.

## 47.6 Databases and storage

- **Postgres** — the system of record: users, conversations, messages, request logs, jobs, documents,
  evaluation results. Plus **pgvector** for embeddings unless you've measured a need for more
  (Topic 23).
- **Redis** — cache, rate limiting, queue broker, pub/sub for streaming.
- **Object storage** — uploaded documents, generated artifacts, model checkpoints, trace archives.
  Keep large blobs out of Postgres; store references.
- **Analytics store** — if request-log volume outgrows Postgres, move it to something columnar.
  Request logs grow fastest, so plan retention and partitioning early.

## 47.7 Multi-tenancy

Decide early, because retrofitting is painful: tenant id on every row, enforced at the storage layer
(Topic 23/42); per-tenant budgets and rate limits so one tenant can't exhaust shared capacity or your
provider quota; tenant-scoped caches; and per-tenant cost attribution, without which you cannot price
or identify unprofitable customers.

## 47.8 Evolution, and what to build first

The decisions that are expensive to reverse: the model gateway (or its absence), multi-tenancy, the
request-log schema, and whether prompts are versioned artifacts.

A sensible order: a monolith with a model gateway and full request logging → add queues and workers
when something takes too long → add caching when cost hurts → split out inference when you self-host
→ split further only when a resource boundary forces it.

Resist the opposite order. Microservices and a vector-database cluster on day one buy nothing; the
gateway and the logs buy everything.

---

# Part 2 — Questions to implement

Consolidate your earlier work into one coherent system in this folder. Docker Compose for dependencies.

### Q1. Architecture diagram and decision record
**Build:** a diagram of your intended system, plus a short decision record: what's one service, what's
separate, and why — with the resource/lifecycle reason for each split.
**Explain:** which boundary are you least sure about, and what evidence would settle it?

### Q2. The model gateway
**Build:** a gateway module/service every model call goes through: provider abstraction, credentials,
timeouts, retries, logging, cost accounting.
**Check:** no application file imports a provider SDK. Prove it with a grep in your test suite.
**Explain:** how many call sites did you have to change? What would this cost after another year of
growth?

### Q3. Model swapping
**Build:** switch your whole application from one provider to another (or to your local server from
Phase 11) by configuration only.
**Check:** everything still works; no application code changed.
**Explain:** what was still coupled and had to be special-cased?

### Q4. Gateway-level fallback and kill switch
**Build:** automatic fallback on failure, plus a per-feature kill switch.
**Check:** simulate an outage; trip the kill switch on one feature and confirm others are unaffected.
**Explain:** why do these belong in the gateway rather than in each feature?

### Q5. Budgets in the gateway
**Build:** per-user, per-feature and global spend limits enforced centrally, with clear errors when
exceeded.
**Check:** trigger each limit.
**Explain:** what would a runaway agent have cost without the global limit? (Use real numbers from
Phase 8.)

### Q6. Caching stack
**Build:** provider prompt caching, an exact response cache, and an embedding cache — all in the
gateway.
**Check:** measure hit rate and cost saving for each over realistic traffic.
**Explain:** report the saving per layer. Which gave the most per unit of effort?

### Q7. Semantic cache and its risk
**Build:** a semantic cache with a calibrated threshold (Topic 21).
**Check:** measure hit rate, and specifically count *wrong* hits — cases where a different question's
answer was served.
**Explain:** report both. Would you ship it? For which routes?

### Q8. Cache key discipline
**Build:** deliberately omit the prompt version from a cache key, then deploy a prompt change.
**Check:** observe stale behaviour being served.
**Explain:** what must a key contain, and what's the worst version of this bug?

### Q9. Queues with priority and shared limits
**Build:** interactive and batch queues with priority, plus a **global** provider rate limiter shared
by all workers.
**Check:** flood the batch queue and confirm interactive latency is unaffected and no 429 storm occurs.
**Explain:** why isn't a per-worker limiter sufficient?

### Q10. Batch API for deferrable work
**Build:** route an evaluation run or a bulk classification job through the provider's batch endpoint.
**Check:** compare cost and completion time against synchronous calls.
**Explain:** report both. What fraction of your total workload could use this?

### Q11. Multi-tenancy end to end
**Build:** tenant id on every table with storage-layer enforcement, tenant-scoped caches, per-tenant
budgets and cost reporting.
**Check:** a test suite attempting cross-tenant access through every path — query, cache, retrieval,
logs.
**Explain:** which path nearly leaked?

### Q12. Storage layout
**Build:** move large artifacts to object storage with references in Postgres; add partitioning or a
retention policy for request logs.
**Check:** measure database size before and after; confirm old logs are archived or dropped as
designed.
**Explain:** project your request-log growth for a year at 10× traffic. Does your plan hold?

### Q13. The full system, load tested
**Build:** run everything together under realistic load for 10 minutes.
**Check:** report p95 latency, error rate, cost, cache hit rates, and queue depths.
**Explain:** what was the bottleneck? Was it your service, the database, or the provider?

### Q14. The scaling plan
**Build:** a written plan for 10× and 100× current traffic: what breaks first at each level, and what
you'd change.
**Explain:** which change is architectural (must be done early) and which is operational (can wait)?

---

# Done when you can answer

1. What two properties distinguish AI systems from ordinary web systems?
2. Where should service boundaries go, and where shouldn't they?
3. What does a model gateway give you, and why is "no direct SDK calls" the key rule?
4. Which caching layers exist, and what must every key include?
5. Why is semantic caching risky?
6. Why must rate limiting be global rather than per worker?
7. Which architectural decisions are expensive to reverse?

Write answers in `notes.md`.
