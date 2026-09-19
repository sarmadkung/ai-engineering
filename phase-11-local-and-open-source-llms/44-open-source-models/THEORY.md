# Topic 44 — Open-Source Models

**Why this topic:** you've been renting intelligence through an API. Open-weight models let you own
it — with real trade-offs in quality, cost, privacy and operational burden. This topic is how to
choose, and how to know when the choice is wrong.

---

# Part 1 — Theory

## 44.1 What "open" means

Careful with the word. Distinguish:

- **Open weights** — you can download and run the weights. Training data and code usually not
  released. This is what most "open source" models actually are (Llama, Qwen, Mistral, Gemma).
- **Open source** in the strict sense — weights, code, and data, under a permissive licence. Rarer
  (OLMo, some others).
- **Licence** — varies enormously: Apache 2.0 (permissive), MIT, or custom community licences with
  restrictions on scale, use, or naming. **Read it before building a business on it**, because some
  restrict commercial use above a user threshold, or restrict training other models on outputs.

## 44.2 The families

Model names change fast, so learn the families and their characters rather than memorizing versions:

- **Llama (Meta)** — the ecosystem default; broadest tooling, fine-tuning recipes and community
  support. Custom licence.
- **Qwen (Alibaba)** — strong across sizes, excellent multilingual and coding variants, wide range of
  sizes. Usually Apache 2.0.
- **Mistral / Mixtral (Mistral AI)** — efficiency-focused; popularized MoE in open weights.
- **Gemma (Google)** — small, well-distilled, good quality per parameter.
- **DeepSeek** — strong reasoning and coding models, MoE architectures.
- **Phi (Microsoft)** — small models trained on curated data; strong for their size.

Also: **base vs instruct vs reasoning** variants of the same model. Base continues text (Topic 10);
instruct follows instructions (Topic 14); reasoning variants spend tokens thinking (Phase 14). Using
a base model where you wanted instruct is a classic beginner failure that looks like the model being
bad.

## 44.3 Size, and what fits

Sizes cluster around ~1B, 3–4B, 7–9B, 12–14B, 30–70B, and MoE models with large totals but small
active parameters (Topic 12).

Memory rule of thumb for inference:

```
fp16:    ~2 GB per billion parameters
8-bit:   ~1 GB per billion
4-bit:   ~0.5 GB per billion
```

Plus the KV cache, which grows with context and concurrency and is easy to forget until you OOM in
production (Topic 12).

So: a 7B model in 4-bit fits comfortably on a laptop; a 70B in 4-bit needs ~40 GB, i.e. a serious
GPU or a Mac with lots of unified memory. And note that an MoE's *memory* tracks total parameters
while its *speed* tracks active parameters — you need to hold all the experts even though most idle.

## 44.4 Capability expectations, honestly

A current small open model (7–14B) is genuinely good at: summarization, classification, extraction,
simple RAG answering, format conversion, and straightforward code. That covers a lot of real product
work.

It is meaningfully worse at: long multi-step reasoning, reliable tool use across many tools,
instruction-following under complex constraints, long-context reliability, and anything requiring
broad world knowledge.

The practical consequence is a **hybrid architecture**: small local models for high-volume,
well-defined tasks; a frontier API model for the hard path. That's usually the economically correct
answer, and it requires the routing you built in Topic 31.

## 44.5 Why choose local

Real reasons: **privacy and compliance** (data never leaves your infrastructure — often the deciding
factor), **cost at high volume** (fixed hardware beats per-token pricing past a break-even point),
**latency** (no network hop, and small models are fast), **no rate limits**, **offline operation**,
**customization** (fine-tuning the weights you own — Phase 13), and **stability** (the model doesn't
change under you, which matters when you've tuned prompts against its quirks).

Real costs: worse peak capability, hardware capital or rental, operational burden (you are now
running an inference service — Topic 46), and your own team's time, which is usually the largest
line item and the one most often omitted from the comparison.

## 44.6 Model selection, as a process

1. **Define the task and a measurable bar** — your Phase 9 evaluation set.
2. **Establish a ceiling** with a frontier API model. If the strongest model can't do it, a 7B
   certainly can't.
3. **Shortlist** by leaderboards and licence — as a filter, not a verdict.
4. **Evaluate candidates on your own set.** This is the only step that decides anything.
5. **Check latency and throughput** on your target hardware, at your target concurrency.
6. **Compute total cost** including hardware, operations and engineering time.
7. **Decide** — and record what would change the decision.

Benchmarks are contaminated and gamed (Topic 37). A model that ranks above another publicly may be
worse on your data, and only your evaluation set can tell you.

## 44.7 Cost break-even

The arithmetic that decides most local-vs-API debates:

```
API:   tokens × price per token
Local: hardware (or rental) + power + operations + engineering time
```

Local wins at sustained high volume with predictable load; API wins for spiky, low or uncertain
volume, and whenever peak capability matters. Do the calculation for your real volume — and include
the engineering time, because that is what makes most "cheaper locally" claims false at small scale.

---

# Part 2 — Questions to implement

Build `model_selection/` here. You'll need Ollama or `llama.cpp` (Topic 45 covers them; installing
one now is fine) and your Phase 9 evaluation set.

### Q1. Licences
**Build:** for 4 models you might use, tabulate: licence, commercial restrictions, attribution
requirements, limits on training other models on outputs.
**Explain:** which would you be comfortable building a paid product on, and which not?

### Q2. Base vs instruct
**Build:** download the base and instruct variants of one small model. Give both the same
question-style prompt.
**Check:** record both outputs.
**Explain:** describe the difference. How would a newcomer misdiagnose this?

### Q3. Memory arithmetic
**Build:** a calculator: parameters × precision + KV cache (given context length and concurrency) → memory.
**Check:** validate against a real load — run a model and measure actual memory use.
**Explain:** how close was your estimate? What did you forget?

### Q4. What fits on your machine
**Build:** determine the largest model you can run at a usable speed, and at what quantization.
**Check:** measure tokens/second for each size you can run.
**Explain:** report the size/speed curve. Where does it become unusable for interactive work?

### Q5. Establish the ceiling
**Build:** run your Phase 9 evaluation set against a frontier API model.
**Check:** record the score, latency and cost.
**Explain:** this is your upper bound. What's the minimum score you'd accept from a local model?

### Q6. Evaluate local candidates
**Build:** the same evaluation set against 3 local models of different sizes/families.
**Check:** one table — score, latency, tokens/second, memory.
**Explain:** report the quality gap to the frontier model. Is any local model above your Q5 bar?

### Q7. Where they fail
**Build:** for your best local model, classify every failure against the frontier model's answer.
**Check:** group failures by type (reasoning, instruction-following, format, knowledge, tool use).
**Explain:** does the pattern match §44.4? What kind of task would you never send locally?

### Q8. Task-by-task comparison
**Build:** evaluate both on four task types separately: classification, extraction, summarization,
multi-step reasoning.
**Check:** report per-task scores for local and frontier.
**Explain:** on which tasks is local *good enough*? This is your routing policy — write it out.

### Q9. Hybrid routing
**Build:** a router (Topic 31) sending easy tasks locally and hard ones to the API.
**Check:** measure end-to-end quality and cost against all-API and all-local.
**Explain:** report all three. What fraction of traffic stayed local, and what did you save?

### Q10. Structured output and tool use locally
**Build:** ask a local model for schema-conformant JSON, then for a tool call, over 20 attempts each.
**Check:** report valid-output rates.
**Explain:** compare with the API model's rates (Topic 18). What does this mean for local agents?

### Q11. Leaderboard vs your data
**Build:** pick two models where public rankings disagree with your evaluation.
**Check:** report both rankings.
**Explain:** did the leaderboard order hold on your set? What does that tell you about model
selection by reputation?

### Q12. Break-even
**Build:** a cost model: API price at your volume versus local hardware (buy or rent) plus power plus
an honest estimate of engineering hours.
**Check:** find the monthly token volume where local becomes cheaper.
**Explain:** state the number. Are you above or below it? Does the answer change if you exclude
engineering time — and is excluding it honest?

### Q13. The decision
**Build:** a written recommendation for one real use case: which model, hosted where, why, and what
you're giving up.
**Explain:** name the single measurement that would reverse your decision.

---

# Done when you can answer

1. What's the difference between open weights and open source, and why do licences matter?
2. What are base, instruct and reasoning variants for?
3. How do you estimate memory for a given model, and what's easy to forget?
4. What are small open models genuinely good at, and where do they fail?
5. What are the real reasons to run locally — and the real costs?
6. Why can't a leaderboard choose your model?
7. What determines the local-vs-API break-even?

Write answers in `notes.md`.
