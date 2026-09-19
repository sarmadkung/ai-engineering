# Project 10 — Production-Grade AI Platform

**Depends on:** everything. Especially Phases 9, 10, 12.

**What you prove:** that you can build the layer other engineers build on — the model gateway,
evaluation, observability and governance that a company needs once more than one team is shipping AI
features. This is the capstone.

---

# Part 1 — What you are building

Internal infrastructure, not an end-user feature:

```
application teams
  -> your gateway API (auth, routing, budgets, caching, fallback)
  -> providers (hosted models + your self-hosted inference)
  + prompt registry, evaluation service, observability, admin console
```

The users are engineers. Success means they ship faster and the organization can answer questions it
couldn't before: what does AI cost us, is quality improving, who is using what, and what happens when a
provider fails.

---

# Part 2 — Design decisions

| Decision | Options | Consider |
|---|---|---|
| Gateway form | library / sidecar / service | a service centralizes control; a library is lower latency |
| API surface | OpenAI-compatible / your own | compatible means existing SDKs work (Topic 46) |
| Multi-tenancy | team-scoped keys and budgets | required from day one (Topic 47) |
| Prompt storage | in app repos / central registry | central registry enables evaluation and rollback |
| Self-hosting | none / vLLM for cheap routes | only with measured utilization (Topic 46) |
| Evaluation | per-team / platform service | a shared harness is a major multiplier |

---

# Part 3 — Milestones

### M1 — Gateway core (Topics 16, 47)
OpenAI-compatible endpoints, provider abstraction, retries, timeouts, streaming, per-request logging with
tokens and cost.
**Check:** an existing application switches to your gateway by changing a base URL and key. Measure the
latency overhead you add — keep it small and report it.

### M2 — Auth, budgets, limits (Topics 42, 48)
Per-team keys, request and **token** rate limits, daily/monthly spend caps, and a global provider-quota
limiter shared across all callers.
**Check:** one team exhausting its budget doesn't affect others, and the global limiter prevents a 429
storm.

### M3 — Routing and model aliases (Topics 44, 47)
Logical names (`fast`, `smart`, `cheap`) mapped to concrete models per environment, changeable centrally
without application deploys.
**Check:** repoint an alias and confirm every caller follows, with the change audited.

### M4 — Caching (Topics 16, 47)
Provider prompt caching support, exact response caching keyed on model + params + prompt version +
messages + tenant, and an embedding cache.
**Check:** report hit rates and cost saved. Prove a tenant cannot receive another tenant's cached
response (Topic 41).

### M5 — Reliability (Topic 43)
Fallback across providers and models, circuit breakers per provider, and a per-feature kill switch.
**Check:** simulate a provider outage — traffic fails over with no application changes. Trip a kill switch.

### M6 — Prompt registry (Topic 49)
Versioned prompts with templates, retrievable by name and version, with the version recorded on every
request.
**Check:** roll back a prompt centrally without a code deploy; the logs show exactly which version served
which request.

### M7 — Evaluation service (Topics 37, 38)
A shared harness: teams register datasets and metrics, runs are triggered on demand or in CI, results are
stored with prompt/model versions, and regressions are flagged against a baseline with statistically
justified thresholds.
**Check:** a deliberately worse prompt is caught. Two teams' datasets run independently.

### M8 — Observability (Topic 40)
Traces with propagated ids, structured logs with redaction, and dashboards for usage, cost with
projection, latency percentiles, error rates, cache hit rate, and quality proxies — sliced by team,
feature and model.
**Check:** answer from the platform alone: which team spends most, which feature has the worst p95, did
cost per request change this week, which model has the highest error rate.

### M9 — Self-hosted route (Topics 45, 46)
One route served by your own inference server, behind the same gateway API.
**Check:** measure cost per million tokens at your real utilization and compare with the API (Topic 46).
Report whether it's actually cheaper.

### M10 — Governance (Topics 42, 59)
Audit logs of who called what, PII redaction policy with retention, per-team data-handling settings, model
cards for approved models, and an approval path for adding a new provider or model.
**Check:** produce an audit report for one team's last month. Honour a data-deletion request end to end.

### M11 — Developer experience
A client library, a quickstart that gets a new team running in under 10 minutes, and honest docs including
known limitations.
**Check:** hand the docs to someone unfamiliar and watch them integrate. Time it; fix what they stumble on.

### M12 — Operations (Topics 48, 49)
Load tested, deployed with CI, rollback rehearsed, runbooks for provider outage, cost spike, rate
limiting, and degraded quality.
**Check:** a game day exercising two runbooks. Report what the runbooks got wrong.

---

# Part 4 — The value case

Measure what the platform is worth, because this is what distinguishes infrastructure from a hobby:

1. **Cost saved** — caching, routing and batch, as a percentage of total spend.
2. **Integration time** — hours for a team to ship an AI feature, before and after.
3. **Incidents prevented** — outages absorbed by fallback, cost overruns stopped by budgets, regressions
   caught by evaluation gates.
4. **Overhead added** — your gateway's latency and operational cost, honestly stated.
5. **Questions now answerable** — list what the organization can see that it couldn't before.

---

# Part 5 — Deliverables

- The platform, deployed, with at least two of your earlier projects migrated onto it
- `ARCHITECTURE.md` — components, decisions, and the reasons
- `RESULTS.md` — the value case above, with numbers
- `RUNBOOKS.md` — operational procedures, tested
- `GOVERNANCE.md` — audit, retention, redaction, approval process
- Developer docs and a client library
- Dashboards

---

# Part 6 — Done when

- Two or more of your own projects run through it and are simpler for it.
- A provider outage is absorbed without any application change.
- You can answer cost, quality and usage questions per team and per feature from one place.
- A quality regression is caught by CI before it ships.
- A new team can integrate in under 10 minutes from the docs alone.
- You can state the platform's own overhead and still defend its value with numbers.

---

# Part 7 — After this

You have built every layer: the model, the application, retrieval, tools, agents, evaluation, security,
serving, infrastructure, and the platform. What's left is depth and judgement, which come from operating
real systems with real users.

Two habits worth keeping permanently: **measure before believing**, and **read the primary sources**
for anything you depend on. Both are what this roadmap was really teaching.
