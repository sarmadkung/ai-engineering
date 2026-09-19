# Phase 12 — AI Infrastructure

**Goal:** the production system around the models — the architecture, the scaling decisions, and the
deployment discipline that let a team operate AI features without surprises.

---

## What I will do

| # | Topic | What I build | Status |
|---|---|---|---|
| 47 | [Production AI Architecture](47-production-ai-architecture/) | a consolidated system with a **model gateway** | ready |
| 48 | [Scalability](48-scalability/) | `scale/` — limiters, load tests, a capacity model | ready |
| 49 | [Deployment](49-deployment/) | containers, CI with evaluation and cost gates, rollback, runbooks | ready |

## Which things to learn

**47. Production AI Architecture** — the two properties that make AI systems different (an externally
priced dependency, and quality as a runtime property); service boundaries along resource and lifecycle
lines; **the model gateway** — one chokepoint for credentials, caching, budgets, fallback, observability
and A/B routing, with the rule that no application code calls a provider SDK directly; queues; the
caching layers and what every key must include; multi-tenancy decided early; what's expensive to reverse.

**48. Scalability** — what actually limits you (provider rate limits and cost, rarely your CPU);
concurrency sizing against the provider's limit; backpressure; multi-provider balancing; **token limits
matter more than request limits**; the cheapest-first order of cost optimizations; cost per *completed
task*; latency percentiles and the tail; capacity planning, including requesting quota before you need it.

**49. Deployment** — prompts and models as deployable artifacts; containers without model weights inside;
whether you need Kubernetes (usually not); CI with an **evaluation gate** and a **cost gate**; prompt
versioning with hot rollback; canaries; model pinning and migration; automatic rollback; feature flags and
kill switches; runbooks rehearsed before the incident; and quality incidents treated as real incidents.

## Prerequisites

Phases 4, 9, 10, and ideally 11. A deployment target with a **budget alarm set before you start**, and
Docker.

**Next:** Phase 13 — back to the models: customizing and optimizing them.
