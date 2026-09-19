# Topic 53 — Reasoning Models

**Why this topic:** the biggest capability shift since instruction tuning. Models that spend tokens
thinking before answering behave differently enough that prompts and architectures designed for
earlier models are often wrong for them. This topic is what changed and how to use it.

---

# Part 1 — Theory

## 53.1 Reasoning tokens

A reasoning model generates an internal chain of thought before its answer. Mechanically it's the same
next-token prediction (Topic 1) — the difference is that the model was **trained** to use that space
productively, typically with reinforcement learning against verifiable outcomes (maths, code, logic),
rather than merely prompted to.

Consequences you must design around:

- **You pay for thinking tokens**, often many of them, and they're billed as output.
- **Latency rises substantially** — seconds to minutes on hard problems.
- **The raw chain of thought is usually not returned** (providers give a summary or nothing), so you
  can't inspect the reasoning directly. Don't build logic that depends on parsing it.
- **Hand-written "think step by step" scaffolding is often counterproductive** — it competes with
  trained behaviour. Prompts written for earlier models tend to be over-engineered for these.

The practical instruction: on a reasoning model, **state the task and the constraints, then get out of
the way.**

## 53.2 Test-time compute

The insight behind the shift: you can trade inference compute for quality *without retraining*.

Three ways to spend it: think longer (more reasoning tokens), sample several answers and choose, or
search over reasoning paths. Because this scaling is smooth and predictable, a smaller model thinking
longer can beat a larger model answering immediately — which changed the economics of the field.

The engineering implication: **quality becomes a dial you control per request.** Cheap-and-fast for
easy routes, expensive-and-thorough for hard ones — which is Topic 31's routing, now applied to effort
rather than model choice. Modern APIs expose this as an effort or thinking setting rather than a fixed
token budget.

Diminishing returns are real: doubling compute doesn't double quality, and beyond a point it buys
nothing. Measure where your ceiling is (Phase 9) rather than defaulting to maximum.

## 53.3 Search

Instead of one reasoning path, explore several:

- **Best-of-N** — sample N answers, pick the best by a verifier or reward model. Simple, effective,
  and the quality depends entirely on the picker.
- **Beam search over reasoning steps** — keep the top-k partial paths.
- **Tree of Thoughts** — branch, evaluate states, backtrack. Powerful for puzzles; expensive.
- **Monte Carlo Tree Search** — simulate forward, prioritize promising branches. What game-playing
  systems use, adapted to reasoning.

The binding constraint for all of these is the **evaluator**. Search is only as good as your ability to
tell a good partial path from a bad one — which is why search works spectacularly where verification is
cheap (code with tests, maths with checkable answers) and poorly where it isn't. Same lesson as
Topic 36, arrived at from a different direction.

## 53.4 Verification

Again the central mechanism:

- **Deterministic** — tests, a compiler, a solver, unit checks. Strongest.
- **Process reward models** — score each reasoning *step*, not just the final answer. Catch errors
  early, and give search a usable signal at every node.
- **Outcome reward models** — score the final answer only. Easier to train, less informative.
- **Self-verification** — ask the model to check its own work. Weak but non-zero; much better when
  grounded in a tool.

The general rule holds: **generation is easier than verification for open-ended tasks, and verification
is easier than generation for checkable ones.** Where verification is cheap, you can buy quality with
compute. Where it isn't, you can't — and that asymmetry explains why AI progress is so uneven across
domains.

## 53.5 Self-consistency

Sample N answers at temperature > 0 and take the majority. Works because errors are often distributed
and randomly distributed errors don't agree, while correct answers do.

Requires a comparable answer (a number, a label, a choice) — it doesn't apply to open-ended prose. Cost
is N× and gains flatten quickly (most of the benefit by ~5 samples). Its other use is as an
**uncertainty signal**: disagreement is a genuine warning (Topic 43).

## 53.6 When reasoning models are worth it

**Worth it:** multi-step logic, maths, complex code, planning, analysis with interdependent
constraints, and anything where a wrong answer is expensive.

**Not worth it:** classification, extraction, summarization, format conversion, simple retrieval
answering, and latency-sensitive interactions. Here they cost more, take longer and add nothing — using
one for a classifier is a common and expensive mistake.

Architecturally: route by difficulty (Topic 31), use effort settings rather than model swaps where
available (one cache namespace, one prompt to maintain — Topic 16), and set timeouts and budgets
appropriate to long thinking.

## 53.7 What this changes for agents

Reasoning models are substantially better at agentic work — planning, tool selection, recovering from
errors — which shifts the balance established in Phase 7: less need for elaborate prompt scaffolding
and explicit ReAct formatting, more value in giving the model good tools and a clear goal. Some
multi-agent complexity that existed to compensate for weak reasoning is no longer needed.

What does *not* change: permissions, verification, budgets, observability. A model that reasons better
still acts in the world, and Phases 6, 9 and 10 remain load-bearing.

---

# Part 2 — Questions to implement

Build `reasoning/` here. You need access to a reasoning-capable model (via effort settings on a current
API model) and a task set with **objectively checkable answers** — maths word problems, logic puzzles,
or code with tests.

### Q1. Build a checkable task set
**Build:** 30 problems with verifiable answers, spanning easy/medium/hard.
**Check:** an automatic verifier.
**Explain:** why is objective verification essential for this whole topic?

### Q2. Reasoning vs non-reasoning
**Build:** run the set with thinking/effort off (or low) and high.
**Check:** report accuracy, tokens, latency and cost for each.
**Explain:** report the accuracy gain and the cost multiple. Where was the gain concentrated?

### Q3. The effort dial
**Build:** run at every effort level your provider supports.
**Check:** tabulate accuracy, output tokens, latency, cost.
**Explain:** plot accuracy against cost. Where are the diminishing returns, and which level would you
default to?

### Q4. Difficulty interaction
**Build:** break Q3's results down by problem difficulty.
**Explain:** did high effort help on easy problems at all? What routing policy do these numbers imply?

### Q5. Scaffolding is counterproductive
**Build:** three prompts on a reasoning model: bare task; task plus "think step by step"; task plus an
elaborate reasoning framework.
**Check:** report accuracy and tokens for each.
**Explain:** did scaffolding help or hurt? Now run the same three on a non-reasoning model and compare.
What does the contrast tell you about inherited prompts?

### Q6. Self-consistency
**Build:** sample 1, 3, 5, and 10 answers at temperature 0.8 and take the majority.
**Check:** report accuracy and cost per setting.
**Explain:** where did gains flatten? Compare the cost of 5-sample consistency against one high-effort
call at equal spend — which bought more accuracy?

### Q7. Disagreement as uncertainty
**Build:** from Q6's samples, measure whether disagreement predicts incorrectness.
**Check:** report accuracy for unanimous versus split cases.
**Explain:** report both. Is this a usable confidence signal (Topic 43)?

### Q8. Best-of-N with a verifier
**Build:** sample 8 answers and select using your deterministic verifier.
**Check:** report accuracy against a single answer.
**Explain:** now select using an *LLM judge* instead of the verifier and report again. How much of
best-of-N's value came from the picker?

### Q9. A weak verifier ruins search
**Build:** deliberately degrade your verifier (make it wrong 30% of the time) and re-run best-of-N.
**Check:** report accuracy.
**Explain:** state the lesson about search and verification in one sentence.

### Q10. Step-level checking
**Build:** have the model produce numbered reasoning steps, then check each step (with a tool or a
judge) and intervene on the first failure.
**Check:** compare accuracy against checking only the final answer.
**Explain:** did catching errors early help? What did it cost?

### Q11. Tree search on a hard problem
**Build:** a simple tree-of-thoughts implementation on 5 genuinely hard problems, with branching and
state evaluation.
**Check:** report accuracy and cost against single-path high-effort reasoning.
**Explain:** was the extra complexity worth it? Under what conditions would it be?

### Q12. The wrong tool for the job
**Build:** run a simple classification task with a reasoning model at high effort and with a cheap
model.
**Check:** report accuracy, latency, cost.
**Explain:** report the cost multiple for equal accuracy. Why is this a common mistake?

### Q13. Reasoning models in an agent
**Build:** run your Phase 7 agent on the same 20 tasks with a reasoning model and a non-reasoning one.
**Check:** report success rate, steps, cost, and failure categories.
**Explain:** which failure modes disappeared? Which scaffolding in your agent became unnecessary? What
still mattered regardless?

### Q14. The routing policy
**Build:** a difficulty router choosing model and effort per request, gated on your measurements.
**Check:** report end-to-end accuracy and cost against always-high-effort and always-low.
**Explain:** how much did routing save at what quality cost? Would you ship it?

---

# Done when you can answer

1. What are reasoning tokens, and how were they trained?
2. What does test-time compute let you trade, and why does that matter economically?
3. Why is the evaluator the binding constraint on search?
4. Why does verification difficulty explain uneven AI progress across domains?
5. Why does self-consistency work, and where can't it apply?
6. When is a reasoning model the wrong choice?
7. What did reasoning models change about agent design, and what did they not?

Write answers in `notes.md`.
