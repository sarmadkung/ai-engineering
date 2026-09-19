# Topic 46 — Model Serving

**Why this topic:** running a model on your laptop is not serving it. This topic is the inference
service: many concurrent users, predictable latency, high utilization, and an interface your
application code doesn't have to care about.

---

# Part 1 — Theory

## 46.1 OpenAI-compatible APIs

The de facto standard: expose `/v1/chat/completions` and friends, and every existing client works.
vLLM, llama.cpp's server, Ollama, and most gateways speak it.

Why it matters: your application code stays provider-independent, you can switch between a local
model and an API by changing a base URL, and tooling (SDKs, proxies, observability) works unchanged.
This is also the practical basis of the fallback design in Topic 43.

Where compatibility ends: tool-calling formats differ subtly, structured-output support varies,
streaming event details differ, and provider-specific features (caching, thinking/effort settings) have
no equivalent. Compatible enough for most code; verify the features you depend on.

## 46.2 Batching — the core of serving economics

Generation is memory-bandwidth bound (Topic 45): each token requires reading the weights. If you read
them for one request, you may as well compute 32 requests' next tokens from the same read.

**Static batching** — collect N requests, run them together, return when all finish. Wasteful: short
requests wait for the longest one, and the batch slot idles.

**Continuous batching (in-flight batching)** — the real answer. Requests join and leave the batch per
token: as soon as one finishes, a queued request takes its place. Utilization stays high and
individual latency stays reasonable. This is what makes vLLM/TGI/SGLang fast, and it's the single
biggest difference between a toy server and a real one.

The throughput/latency trade: larger batches mean more total tokens/second and slightly slower
per-request generation. You tune it against your product's latency requirement, not to a universal
optimum.

## 46.3 KV cache management

The cache (Topic 12) is the binding memory constraint in serving. Per request it grows with sequence
length; across requests it multiplies by concurrency. Long contexts plus high concurrency exhausts
GPU memory long before compute runs out.

Techniques that matter:

- **PagedAttention** — allocate the cache in fixed-size blocks like virtual memory pages instead of
  one contiguous reservation per request. Eliminates the fragmentation and over-allocation that
  otherwise wastes most of the cache, and is why vLLM can hold far more concurrent requests.
- **Prefix caching** — share the cache for a common prompt prefix across requests. Huge win when
  every request carries the same long system prompt (this is the local equivalent of Topic 16's
  prompt caching, and the same prefix-stability discipline applies).
- **Eviction and preemption** — under pressure, evict or recompute a request's cache. Graceful
  degradation rather than OOM.
- **Quantized cache** — store K/V in 8 bits to fit more concurrency.
- **GQA** (Topic 12) — the architectural fix, decided when the model was trained.

## 46.4 Throughput and latency

Distinguish clearly:

- **Throughput** — total tokens/second across all users. Determines how many users one GPU serves,
  and therefore your cost per token.
- **Latency** — TTFT and per-token speed for one user. Determines how the product feels.

They trade against each other, and you must decide which you're optimizing. Metrics to track:
requests/second, tokens/second (prefill and decode separately), TTFT percentiles, inter-token
latency, queue time, batch size, and **GPU utilization**.

Low GPU utilization with a queue means your batching or memory configuration is wrong, not that you
need more hardware — the most common and most expensive misdiagnosis in serving.

## 46.5 Scaling and routing

- **Vertical** — a bigger GPU, or several with tensor parallelism (Phase 13).
- **Horizontal** — more replicas behind a load balancer. Stateless, so this is straightforward.
- **Prefix-aware routing** — route requests sharing a prefix to the same replica so prefix caching
  hits. A real and underused win.
- **Disaggregated prefill/decode** — separate pools for the compute-bound and memory-bound phases,
  sized independently. Advanced, and a clear expression of §45.3's asymmetry.
- **Multi-model serving** — LoRA adapters (Topic 14) let one base model serve many fine-tunes, since
  adapters are tiny and swappable per request. Far cheaper than a replica per variant.

## 46.6 Operating an inference service

Model loading is slow, so replicas must be warmed before receiving traffic; readiness must mean "can
generate", not "process started". Then: admission control and a bounded queue (rejecting fast beats
timing out slowly), request timeouts and cancellation (Topic 20 — a disconnected client's generation
should stop), autoscaling on queue depth rather than CPU, and a rollout strategy for model updates
(they change behaviour, so treat them like deploys with evaluation gates from Phase 9).

Cost accounting for a self-hosted model is different from an API: you pay for **capacity**, not
tokens. Your effective cost per token is hardware cost divided by tokens actually produced — so
utilization *is* your unit economics. A GPU at 15% utilization is roughly 6× more expensive per
token than the same GPU at 90%.

---

# Part 2 — Questions to implement

Use vLLM if you have an NVIDIA GPU, else llama.cpp's server (it also has continuous batching). Build
`serving/` and a load-testing script here.

### Q1. Stand up a server
**Build:** run an OpenAI-compatible inference server and point your Phase 4 application at it by
changing only the base URL.
**Check:** chat, streaming, and usage reporting all work.
**Explain:** what broke, if anything? Which features were missing?

### Q2. Compatibility audit
**Build:** test tool calling, structured outputs, streaming details, and usage fields against both your
local server and an API provider.
**Check:** tabulate what works where.
**Explain:** which incompatibility would most affect your code?

### Q3. Single-stream baseline
**Build:** measure TTFT, tokens/second, and total latency for one request at a time.
**Explain:** report the numbers. This is the best-case latency you can promise.

### Q4. Load test
**Build:** a load generator with configurable concurrency and realistic prompt/output lengths.
**Check:** run at concurrency 1, 4, 16, 64.
**Explain:** report aggregate tokens/second, TTFT p50/p95, and inter-token latency for each. Where did
aggregate throughput plateau?

### Q5. Continuous vs static batching
**Build:** if your server supports both, compare them with a mix of short and long requests.
**Check:** report throughput and the latency of *short* requests in each mode.
**Explain:** what happened to short requests behind a long one under static batching?

### Q6. Find the saturation point
**Build:** increase concurrency until latency degrades unacceptably or requests fail.
**Check:** record throughput, latency and GPU utilization at each step.
**Explain:** what's your maximum safe concurrency? What failed first — memory, queue, or latency?

### Q7. KV cache limits
**Build:** measure maximum concurrency at context lengths 1k, 8k, 32k.
**Check:** report the numbers and the memory used.
**Explain:** show the arithmetic linking context length to concurrency. Which product decision does
this constrain?

### Q8. Prefix caching
**Build:** requests sharing a 2000-token system prompt, with prefix caching on and off.
**Check:** measure TTFT and throughput for both.
**Explain:** report the improvement. Then change one token at the start of the prefix and report what
happens.

### Q9. Utilization diagnosis
**Build:** deliberately misconfigure the server (tiny batch size or over-reserved memory) and observe.
**Check:** record GPU utilization, queue depth and throughput.
**Explain:** you now have low utilization *and* a queue. How would you distinguish this from "need
more hardware"?

### Q10. Admission control
**Build:** a bounded queue that rejects with 429 when full, plus per-request timeouts and cancellation
on client disconnect.
**Check:** overload it and confirm fast rejection rather than slow timeouts; confirm a disconnected
client's generation stops.
**Explain:** why is fast rejection better for the caller?

### Q11. Horizontal scaling
**Build:** two replicas behind a load balancer.
**Check:** measure aggregate throughput and the effect on prefix cache hit rate.
**Explain:** did throughput double? What did naive round-robin do to your cache hits, and what would
prefix-aware routing fix?

### Q12. LoRA multi-tenancy
**Build:** serve two LoRA adapters (Topic 14) on one base model, selected per request.
**Check:** both behave correctly; measure the overhead versus a single-adapter server.
**Explain:** compare cost against running two full model replicas.

### Q13. Cost per token
**Build:** compute your effective cost per million tokens: hourly hardware cost divided by measured
throughput, at 20%, 50% and 90% utilization.
**Check:** compare against an API provider's price.
**Explain:** at which utilization do you actually beat the API? What sustained volume does that
require?

### Q14. Model rollout
**Build:** a swap to a new model version with a warm replica, an evaluation gate (Phase 9), and
rollback.
**Check:** no dropped requests during the swap; the gate blocks a deliberately worse model.
**Explain:** why should a model change be treated as a deploy rather than a config edit?

---

# Done when you can answer

1. Why does an OpenAI-compatible API matter, and where does compatibility end?
2. Why does continuous batching beat static batching?
3. What does PagedAttention solve?
4. What does prefix caching buy, and what breaks it?
5. Why is low utilization with a queue not a hardware problem?
6. What is the throughput/latency trade, and who decides it?
7. Why is utilization your unit economics when self-hosting?

Write answers in `notes.md`.

---

**Phase 11 is complete.** You can run and serve your own models. Phase 12 is the production
infrastructure around all of it.
