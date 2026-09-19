# Topic 49 — Deployment

**Why this topic:** shipping and operating AI systems. Mostly standard practice, with three
differences that matter: prompts and models are deployable artifacts, GPUs are expensive and awkward,
and quality regressions don't announce themselves.

---

# Part 1 — Theory

## 49.1 What's different about deploying AI

- **Prompts are code.** They change behaviour as much as code does, so they need versioning, review,
  evaluation and rollback.
- **Models are dependencies that change under you.** A provider updates a model and your carefully
  tuned prompts behave differently. Pin versions where the provider allows it, and re-run evaluation
  when you move.
- **Quality regressions are silent.** No test fails; the system just gets worse. Hence evaluation as a
  release gate (Phase 9).
- **GPU deployment is slow and costly** — minutes to load a model, expensive to keep warm, awkward to
  autoscale.
- **Cost is a deploy risk.** A prompt change can multiply your bill without any error appearing.

## 49.2 Containers

Standard practice, with AI-specific notes:

- **Keep model weights out of the image.** Multi-gigabyte images are slow to build, push and pull.
  Mount a volume or download to a cache on start.
- **Multi-stage builds** and pinned dependencies — the Python AI stack is large and breaks on minor
  version drift.
- **CUDA images** for GPU work: the driver lives on the host, the toolkit in the container, and the
  versions must be compatible. This is where most first-time GPU deployments fail.
- **Health checks that mean something**: readiness should require that the model can generate, not
  that the process started (Topic 46).

## 49.3 Kubernetes, and whether you need it

Useful for multi-service systems, autoscaling, and GPU scheduling (node selectors, taints, device
plugins, and requesting GPUs as resources). Realities: GPU nodes are expensive so bin-packing
matters; scale-up is slow because of image pulls and model loading; and interrupting a long-running
agent job needs graceful termination windows and cooperative shutdown (Topic 33).

Honest guidance: **most AI applications do not need Kubernetes.** A managed platform for the web tier
plus a managed inference endpoint is less work and fewer failure modes. Adopt it when you're running
several services, self-hosting GPUs at scale, or your organization already lives there.

## 49.4 Cloud GPUs

Options: hyperscalers (integrated, expensive, quota-limited), GPU-specialist clouds (cheaper,
simpler, fewer guarantees), serverless GPU (pay per second, cold starts), and managed inference
endpoints (no ops, highest per-token cost).

Practicalities: **quota** must be requested in advance and is often the real constraint; **spot
instances** are much cheaper but can be reclaimed, so only use them for interruptible work with
checkpointing (Phase 13); and **reserved capacity** pays off only at high, sustained utilization
(Topic 46's arithmetic).

## 49.5 CI/CD for AI systems

```
commit
 -> lint, type check, unit tests
 -> prompt/schema validation
 -> fast evaluation subset (Phase 9)
 -> build and push image
 -> deploy to staging
 -> full evaluation suite + cost check
 -> canary in production
 -> full rollout
```

The AI-specific gates: an **evaluation gate** that blocks a quality regression beyond a defined
threshold, and a **cost gate** that fails the build if tokens per request jump — because that's how
budget disasters ship.

Practical issues: evaluation costs money (small set per commit, full set nightly or pre-release),
evaluation is non-deterministic (multiple runs, statistical thresholds — Topic 37), and you need test
credentials with their own budget so CI can't spend production money.

## 49.6 Deploying prompts and models

Treat both as versioned artifacts:

- Prompts in version control, with an id and version attached to every request log (Topic 40).
- The ability to roll back a prompt without a code deploy — but keep the rollback auditable.
- **Canary** a new prompt or model on a small traffic slice, comparing quality proxies and cost before
  full rollout.
- Model version pinning, with a planned, evaluated migration when a provider deprecates one.
  Deprecation notices arrive with deadlines; an unpinned system can change behaviour overnight.

## 49.7 Monitoring and rollback

Watch after every deploy (Topic 40): error rate, latency, cost per request, cache hit rate, refusal
rate, and quality proxies. Then: automatic rollback on error/latency breach, one-command manual
rollback, feature flags to disable an AI feature without deploying, and the kill switch from
Topic 47.

Deliberate practice worth having: **a rehearsed rollback**. The first time you roll back should not be
during an incident.

## 49.8 Operating it

- **Runbooks** for the predictable incidents: provider outage, rate limiting, cost spike, quality
  complaint, GPU node failure, queue backlog.
- **On-call** that includes cost alerts, because a runaway agent loop is a 3am problem.
- **Secrets management** — provider keys in a real secret store, rotated, never in images or repos
  (Topic 42).
- **Regular re-evaluation** — model updates, corpus drift and user behaviour change all degrade a
  system that isn't re-measured.
- **Postmortems** that include quality incidents, not just outages. "The answers got worse for three
  weeks and nobody noticed" is an incident with a root cause.

---

# Part 2 — Questions to implement

Deploy the system from Topic 47 somewhere real (a cheap VPS, a managed platform, or a cloud account
with a budget alarm set first).

### Q1. Containerize
**Build:** a Dockerfile with pinned dependencies and a multi-stage build; run the full stack with
Compose.
**Check:** it starts from scratch on a clean machine. Report image size and build time.
**Explain:** what did you exclude from the image, and why?

### Q2. Meaningful health checks
**Build:** liveness (process alive) and readiness (can actually serve, dependencies reachable).
**Check:** break a dependency; readiness fails while liveness passes.
**Explain:** why must these be different for a service that loads a model?

### Q3. Deploy it
**Build:** deploy to a real environment with secrets from a secret store, not environment files in the
repo.
**Check:** it serves traffic over TLS; no secret appears in the image, the repo, or the logs.
**Explain:** what's your key rotation procedure?

### Q4. CI pipeline
**Build:** lint, tests, prompt validation, and a fast evaluation subset on every commit.
**Check:** a deliberately bad prompt change fails the pipeline.
**Explain:** how long does the pipeline take, and what does each run cost in tokens?

### Q5. Evaluation gate with a real threshold
**Build:** block deploys when quality drops more than a defined amount, using multiple runs and your
Topic 37 noise floor.
**Check:** a small regression passes; a real one is blocked.
**Explain:** justify your threshold statistically. How many runs did you need?

### Q6. Cost gate
**Build:** fail the build if average tokens per request on the evaluation set rises beyond a threshold.
**Check:** add a verbose instruction to a prompt and confirm it's caught.
**Explain:** what would this have caught in your earlier work?

### Q7. Prompt versioning and hot rollback
**Build:** versioned prompts, with the version logged per request and the ability to roll back without
a code deploy.
**Check:** roll back a prompt in production and confirm the logs show the change.
**Explain:** how do you keep a hot change auditable rather than untracked?

### Q8. Canary
**Build:** route 5% of traffic to a new prompt or model, comparing cost and quality proxies.
**Check:** run a canary where the new version is deliberately worse, and detect it from metrics alone.
**Explain:** how long did it take to detect, and how much traffic was affected?

### Q9. Model pinning and migration
**Build:** pin a model version in config, then migrate to a different one with a full evaluation
comparison.
**Check:** report quality, latency and cost differences.
**Explain:** what changed unexpectedly? What would have happened without pinning?

### Q10. GPU deployment (or a documented decision not to)
**Build:** deploy your Phase 11 inference server on a rented GPU with a working CUDA container, or
write up the comparison that led you to a managed endpoint instead.
**Check:** measure cold start, throughput, and hourly cost.
**Explain:** what was your cost per million tokens, and how does it compare to the API?

### Q11. Automatic and manual rollback
**Build:** automatic rollback on error-rate or latency breach, plus a one-command manual rollback.
**Check:** deploy a broken version and watch it revert; then rehearse the manual path.
**Explain:** how long was the bad version live? What would shorten that?

### Q12. Feature flags and kill switch
**Build:** flags to disable each AI feature independently, without deploying.
**Check:** disable one and confirm the rest are unaffected and the UI degrades sensibly.
**Explain:** which feature would you kill first in a cost emergency?

### Q13. Runbooks and a game day
**Build:** runbooks for provider outage, rate limiting, cost spike, quality complaint, and queue
backlog. Then simulate two of them.
**Check:** follow your own runbook and note where it's wrong or incomplete.
**Explain:** which runbook failed under practice, and how did you fix it?

### Q14. A week of operation
**Build:** run the system for a week with real or simulated traffic, monitoring everything.
**Check:** produce a weekly report: uptime, cost, latency percentiles, quality proxies, incidents.
**Explain:** what drifted over the week? What would you change before real users arrived?

---

# Done when you can answer

1. What makes AI deployment different from ordinary deployment?
2. Why keep model weights out of container images?
3. Do you need Kubernetes, and when?
4. What are the two AI-specific CI gates, and what do they catch?
5. How should prompts be versioned and rolled back?
6. What do you watch after a deploy, and what triggers rollback?
7. Why are quality incidents real incidents?

Write answers in `notes.md`.

---

**Phase 12 is complete.** You can build, ship and operate production AI systems. Phase 13 returns to
the models themselves — customizing and optimizing them.
