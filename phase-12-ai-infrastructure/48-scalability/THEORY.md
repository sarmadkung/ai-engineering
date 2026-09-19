# Topic 48 — Scalability

**Why this topic:** AI systems scale differently. The bottleneck is usually not your CPU — it's a
provider's rate limit, a GPU's memory, or your budget. This topic is about finding the real
constraint and designing around it.

---

# Part 1 — Theory

## 48.1 What actually limits you

In order of how often it's the true bottleneck:

1. **Provider rate limits** — requests and tokens per minute. Hit long before your servers strain.
2. **Cost** — you can often scale technically but not affordably. Budget is a capacity limit.
3. **GPU memory** (self-hosted) — the KV cache bounds concurrency (Topic 46).
4. **Model latency** — 2–30 seconds per call, which dominates your own processing entirely.
5. **Database** — vector search and request logs, eventually.
6. **Your application** — rarely, because it spends most of its time waiting.

**Measure before optimizing.** Teams routinely optimize their own code while the request queue is
waiting on a token-per-minute limit.

## 48.2 Concurrency

Since requests are I/O-bound (Topic 20), concurrency is nearly free — and dangerous without limits.

- **Async everywhere**, with no blocking call in an async path.
- **A concurrency cap** per provider and per model, chosen to sit just under your rate limit.
- **Bounded queues** with fast rejection rather than unbounded growth (Topic 46).
- **Backpressure** — when downstream is saturated, propagate it upward instead of accumulating work.

Sizing: your provider limit is the target, not your CPU count. If you're allowed 50 requests in
flight, a semaphore of 45 with a queue is a better design than 500 concurrent calls generating 429s.

## 48.3 Load balancing

Across providers and models: route by capability (cheap model for simple routes — Topic 44), spread
across providers to multiply effective rate limits, spill to a secondary when the primary is limited,
and pin requests with shared prefixes to the same replica when self-hosting so prefix caching hits
(Topic 46).

For self-hosted serving: replicas behind a balancer, autoscaling on queue depth (not CPU — a GPU
server saturated on memory may show modest CPU), and awareness that scale-up is slow because models
take minutes to load. Keep warm capacity for spikes, or accept the cold-start latency and say so.

## 48.4 Batch processing

The biggest cost lever available (Topic 16): the provider's batch endpoint is roughly half price, and
a great deal of real work is deferrable — backfills, bulk classification, evaluation runs, embedding
a corpus, nightly summaries.

Design: separate interactive from deferrable paths deliberately, with different queues, different
priority, and different SLAs. For self-hosted serving, batching is also the throughput mechanism
(Topic 46), so a batch workload is exactly what keeps your GPUs efficient.

## 48.5 Rate limiting your own users

Two separate jobs, often confused:

- **Protecting the provider quota** — a global limiter across all workers (Topic 47).
- **Protecting fairness and margin** — per-user and per-tenant limits, in requests *and* tokens, plus
  spend caps.

Token limits matter more than request limits: one user with a 200k-token prompt can consume more
quota than a thousand ordinary requests. A request-count limiter alone does not protect you.

Implementation: token buckets in Redis (atomic), tiered limits per plan, clear 429s with
`retry-after`, and a degraded mode (cheaper model, shorter answers) as an alternative to hard
refusal.

## 48.6 Cost optimization at scale

In the order you should apply them, cheapest first:

1. **Prompt caching** (Topic 16) — largest win, minimal effort, but audit it (a silent invalidation
   costs you everything it was saving).
2. **Right-size the model per route** (Topic 44) — most traffic doesn't need the flagship.
3. **Response caching** — free for repeated work.
4. **Batch API** for deferrable work — ~50%.
5. **Reduce output tokens** — the expensive half: cap `max_tokens`, ask for terse output, avoid
   regenerating whole documents when a diff would do.
6. **Reduce input tokens** — trim retrieved context (Topic 19), prune history, shorten instructions.
7. **Self-host** at sustained high volume (Topic 46), *if* utilization is high.
8. **Effort/thinking settings** — where supported, lower effort for easy routes.

The honest framing: cost per **completed task**, not per request. A cheap model that needs three
attempts or an agent that loops isn't cheap. And an optimization that degrades quality below your
Phase 9 bar isn't an optimization.

## 48.7 Latency at scale

Reduce it: streaming (Topic 16, perceived latency), smaller/faster models, shorter prompts (prefill),
caching, parallel independent calls, and speculative approaches (start retrieval before the user
finishes typing).

Accept it honestly where you can't: show progress, stream partial results, and set expectations. A
10-second operation with visible progress beats a 6-second one that looks frozen.

And measure percentiles (Topic 40). The p99 is where your timeouts, retries and angry users live.

## 48.8 Capacity planning

Work out: peak concurrent requests, tokens per minute at peak, required provider limits (and request
increases *before* you need them — approval takes time), GPU count for self-hosted, database size and
IOPS, and your monthly spend projection with a variance range.

Then load test to find the real breaking point rather than the assumed one, and know what your system
does when it's exceeded: reject cleanly, degrade, or queue.

---

# Part 2 — Questions to implement

Build `scale/` here: a load generator, limiters, and a capacity model. Use the system from Topic 47.

### Q1. Find the real bottleneck
**Build:** load test with increasing concurrency, instrumenting your service, the database, and
provider responses.
**Check:** identify what fails or saturates first.
**Explain:** report it. Was it what you expected? What would you have optimized by guessing?

### Q2. Concurrency sizing
**Build:** a semaphore per provider, with the cap configurable.
**Check:** measure throughput and 429 rate at caps of 5, 20, 50, 200.
**Explain:** which cap maximized *successful* throughput? What did the highest cap achieve?

### Q3. Backpressure
**Build:** bounded queues with fast rejection, propagating saturation upward.
**Check:** overload the system and confirm rejection rather than unbounded latency growth.
**Explain:** compare user-visible behaviour with and without backpressure.

### Q4. Multi-provider load balancing
**Build:** spread traffic across two providers (or a provider plus your local server) with spillover
on rate limits.
**Check:** measure effective throughput against a single provider.
**Explain:** report the improvement. What new problems did you introduce (quality differences,
prompt-compatibility, cost variance)?

### Q5. Per-user rate limiting, requests and tokens
**Build:** Redis token buckets limiting both, per plan tier.
**Check:** one user hitting a limit doesn't affect others. Then have one user send a 100k-token
request and confirm the *token* limiter catches what the request limiter wouldn't.
**Explain:** report what a request-only limiter would have allowed.

### Q6. Degraded mode
**Build:** instead of refusing at the limit, fall back to a cheaper model and shorter outputs.
**Check:** measure quality and cost in degraded mode.
**Explain:** would users prefer this to a 429? For which features?

### Q7. Caching audit
**Build:** measure prompt-cache hit rate across your real traffic and find every silent invalidator
(Topic 16).
**Check:** report hit rate before and after fixing them.
**Explain:** report the cost change. How was the invalidation introduced?

### Q8. Model right-sizing
**Build:** classify your routes by difficulty and move the easy ones to a cheaper model, gated by your
Phase 9 evaluation.
**Check:** report quality per route and total cost change.
**Explain:** what fraction of traffic moved down a tier, and what did it save?

### Q9. Output token reduction
**Build:** apply three techniques: cap `max_tokens`, instruct for terseness, return diffs/deltas
instead of full documents.
**Check:** measure output tokens and quality before and after.
**Explain:** report savings and any quality cost.

### Q10. Batch path
**Build:** split one workload into interactive and deferrable, sending the deferrable part to the
batch endpoint.
**Check:** compare cost and completion time.
**Explain:** report the saving and what fraction of your workload qualifies.

### Q11. Cost per completed task
**Build:** measure cost per *successful* outcome (including failures and retries) for two
configurations: a cheap model with more retries, and an expensive model with fewer.
**Check:** report both.
**Explain:** which is actually cheaper? Does the naive per-request price mislead?

### Q12. Latency percentiles and the tail
**Build:** report p50/p95/p99 latency under load, and identify what makes the p99 slow.
**Check:** trace three of your slowest requests.
**Explain:** what caused the tail — retries, long outputs, queueing, a slow tool?

### Q13. Capacity model
**Build:** a spreadsheet or script: users → requests/minute → tokens/minute → required provider limits
→ GPU count → monthly cost, with a peak-to-average ratio.
**Check:** validate the current-scale numbers against measurements.
**Explain:** at what user count do you need a provider limit increase? Have you requested it?

### Q14. Break it deliberately
**Build:** push to 5× your planned peak.
**Check:** record exactly what happens — rejections, timeouts, data loss, cost spike.
**Explain:** was the failure graceful? What would a user have seen, and what would you fix first?

---

# Done when you can answer

1. What usually limits an AI system, and what usually doesn't?
2. How should you size concurrency?
3. Why are token limits more important than request limits?
4. What's the cheapest-first order of cost optimizations?
5. Why is cost per completed task the right metric?
6. Why autoscale self-hosted inference on queue depth rather than CPU?
7. What must you know before you hit a capacity ceiling?

Write answers in `notes.md`.
