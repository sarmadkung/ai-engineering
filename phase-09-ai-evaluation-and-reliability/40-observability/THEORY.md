# Topic 40 — Observability

**Why this topic:** evaluation tells you how your system performs on tests. Observability tells you
what it's doing **right now, for real users**. In LLM systems it's more important than in ordinary
software, because failures are silent: no exception, no 500 — just a confidently wrong answer.

---

# Part 1 — Theory

## 40.1 Why LLM observability is different

- **Failures are silent.** The worst outcome is a plausible wrong answer, which no status code
  reports.
- **Cost is per request**, so spend is an operational metric with a spike risk, not a monthly
  constant.
- **Non-determinism** means you cannot reproduce a user's problem by re-running their input.
- **Multi-step flows** mean one user action may be 15 model calls, and the interesting failure is in
  the middle.
- **Quality is the real SLO**, and it isn't visible in infrastructure metrics.

Consequence: you must **log the content** — prompts, completions, retrieved context, tool calls — not
just timings and counts. Without content you can't diagnose a quality problem, and quality problems
are most of your problems.

## 40.2 Tracing

A trace represents one logical operation; spans are its nested steps.

```
trace: "user asked about refunds"
  span: retrieve            120 ms
    span: embed query        30 ms
    span: vector search      80 ms
  span: rerank              200 ms
  span: generate           2400 ms   1,850 in / 320 out  $0.012
```

What each LLM span should carry: model, parameters, input messages, output, token counts (including
cached), cost, latency, TTFT, stop reason, prompt version, and error if any. Agent spans add: step
index, tool name, arguments, result size, and outcome.

**One trace id, propagated everywhere** — across services, background jobs, and retries — is what
makes a multi-step failure reconstructable. Retrofitting it is painful; do it from the start
(Topic 20).

OpenTelemetry is the standard, with emerging semantic conventions for GenAI attributes; LLM-specific
platforms (LangSmith, Langfuse, Phoenix, Braintrust, Helicone, W&B Weave) build on the same idea with
prompt-aware UIs and built-in evaluation.

## 40.3 Logging

Log at the boundary of every model interaction, with structure (JSON), not prose. Include the trace
id, user/session, model and version, prompt version, full input and output, usage, cost, latency, and
outcome.

Two hard constraints:

- **PII.** You are logging user content, which is personal data. Decide what's redacted, how long
  it's retained, who can read it, and how a deletion request is honoured — including in your
  observability vendor. This is a design decision, not an afterthought.
- **Volume and cost.** Full prompt logging is expensive at scale. Sampling (log all errors, N% of
  successes) is the standard compromise.

## 40.4 Metrics

**Usage** — requests, tokens in/out/cached, by model, route and user.
**Cost** — per request, per user, per feature, per day, with a **projection** and alerts. Cost
spikes come from prompt-length regressions, cache misses, retry storms, and agent loops — all of which
are invisible without this.
**Performance** — TTFT, total latency, tokens/second, queue time. Use percentiles, never averages; the
p99 is where the timeouts live.
**Reliability** — error rate by class, retry rate, rate-limit hits, fallback usage, timeouts.
**Quality proxies** — refusal rate, empty/truncated responses (stop reason = max_tokens), user
regenerations, thumbs-down rate, escalation to human, conversation abandonment. These are the
cheapest signals you have that quality is drifting, because they need no labels.
**Agent-specific** — steps per run, tool error rate, loop detections, human interventions, budget
exhaustions.

## 40.5 Token and cost tracking

Attribute every token: per request, per user, per feature, per prompt version. Then you can answer
"which feature is expensive", "which user is unprofitable", "did that prompt change increase cost".

Watch **cache hit rate** specifically (Topic 16) — a silent cache invalidation is one of the most
common and most expensive regressions, and it shows up nowhere else.

Then set: budget alerts, per-user caps, anomaly detection on spend, and a kill switch for a runaway
feature.

## 40.6 Latency

Break it down — queue, prefill (scales with input length), decode (scales with output length),
tool calls, retrieval, your own processing. This tells you where to optimize: a long prompt hurts
TTFT, a long answer hurts total time, and a slow tool hurts more than the model does in most agent
systems.

For streaming, TTFT is the metric users feel (Topic 16); report it separately from total duration.

## 40.7 Online evaluation

The bridge from Phase 9's offline work to production:

- **Automated checks on live traffic**: faithfulness on a sample (no ground truth needed —
  Topic 38), schema validity, forbidden content, refusal appropriateness.
- **Implicit user signals**: regeneration, rephrasing, abandonment, copy actions, thumbs.
- **Sampled human review** of a small daily slice — the only way to catch what your automation
  doesn't measure.
- **Drift monitoring**: input distribution, retrieval score distribution, output length, refusal
  rate. A shift in any of these is an early warning.

Then close the loop: every production failure becomes an evaluation case (Topic 37). That loop is
what makes a system improve instead of just age.

## 40.8 Alerting

Alert on things that are actionable: error-rate spike, latency p95 breach, cost anomaly, cache
hit-rate collapse, refusal-rate jump, quality-check failure rate, provider outage. Avoid alerting on
individual failures in a stochastic system — you'll teach your team to ignore alerts.

---

# Part 2 — Questions to implement

Add observability to your Phase 4 backend and Phase 7/8 agent. Build `observability/` here, or wire
up a platform and compare.

### Q1. Trace ids everywhere
**Build:** generate a trace id per request and propagate it through every model call, tool call,
retrieval and background job.
**Check:** pick a user action causing 5+ model calls and reconstruct the whole thing from logs.
**Explain:** where did propagation nearly break?

### Q2. Structured spans
**Build:** span records with all §40.2 fields, persisted.
**Check:** print a full trace tree with durations and costs.
**Explain:** which single field turned out most useful in practice?

### Q3. Content logging with redaction
**Build:** log full prompts and completions, with PII redaction for a defined set of patterns, plus a
retention policy and a per-user delete.
**Check:** a test confirms redaction works and that deletion removes everything for a user.
**Explain:** what did you choose to redact, and what diagnostic ability did you lose by doing so?

### Q4. Sampling
**Build:** log all errors and N% of successes, configurable.
**Check:** measure log volume at 100% and at 5%.
**Explain:** report the volume and cost difference. What can you no longer investigate at 5%?

### Q5. Cost attribution
**Build:** cost per request, per user, per feature, per prompt version — queryable.
**Check:** run varied traffic and answer: most expensive feature, most expensive user, cost per
successful task.
**Explain:** report all three. Which surprised you?

### Q6. Catch a cost regression
**Build:** cache hit-rate tracking plus an alert. Then break caching deliberately (insert a timestamp
early in the prompt).
**Check:** the alert fires; cost per request rises.
**Explain:** how long did detection take? Without this metric, when would you have noticed?

### Q7. Latency breakdown
**Build:** instrument queue, retrieval, prefill, decode, tool time, and your own processing.
**Check:** report p50/p95 per stage.
**Explain:** where does the time actually go? What would you optimize first?

### Q8. Percentiles vs averages
**Build:** report mean, p50, p95 and p99 latency for the same traffic.
**Explain:** how different are mean and p99? What would you have concluded from the mean alone?

### Q9. Quality proxies without labels
**Build:** track refusal rate, truncation (stop reason), empty responses, and regeneration rate.
**Check:** deliberately degrade quality (a worse prompt) and see which proxies move.
**Explain:** which was the most sensitive early warning?

### Q10. Online faithfulness
**Build:** run your Topic 38 faithfulness check on a 10% sample of live traffic, and alert on a drop.
**Check:** introduce a retrieval bug and confirm the metric moves before users would complain.
**Explain:** why can this metric run in production when correctness cannot?

### Q11. Agent observability
**Build:** per-run metrics — steps, tools, errors, loops, interventions, cost — plus a per-run trace
viewer.
**Check:** find your slowest and most expensive runs and explain each from its trace.
**Explain:** what was the most common cause of expensive runs?

### Q12. Drift detection
**Build:** monitor input length distribution, retrieval top-score distribution, output length and
refusal rate over time.
**Check:** simulate a week where users start asking about topics your corpus lacks.
**Explain:** which signal detected it first, and how much earlier than a user complaint would have?

### Q13. Dashboard and alerts
**Build:** one dashboard — usage, cost with projection, latency percentiles, error rates, quality
proxies, agent metrics — plus the alerts from §40.8.
**Check:** trigger three alerts deliberately.
**Explain:** which alerts would page someone at 3am, and which are for the morning? Justify.

### Q14. Close the loop
**Build:** a pipeline turning production failures into evaluation cases (Topic 37).
**Check:** take 5 real failures, add them to your eval set, and confirm your system's score drops.
**Explain:** why is this loop the thing that makes a system improve rather than just age?

---

# Done when you can answer

1. Why is LLM observability different from ordinary observability?
2. Why must you log content, and what obligations does that create?
3. What does a trace id make possible that per-call logs don't?
4. Which metrics catch a cost regression, and what causes them?
5. Why percentiles rather than averages?
6. Which quality signals need no labels?
7. Which quality metric can run on live traffic, and why?

Write answers in `notes.md`.

---

**Phase 9 is complete.** You can measure and monitor everything you build. Phase 10 covers what
happens when someone attacks it.
