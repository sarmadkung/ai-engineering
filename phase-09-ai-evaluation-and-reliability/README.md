# Phase 9 — AI Evaluation & Reliability

**Goal:** stop guessing. This phase is the skill that most distinguishes senior AI engineers: knowing
whether a change helped, and being able to prove it.

**If you only master one phase after Phase 4, make it this one.** Everything you built earlier gets
better once you can measure it.

---

## What I will do

| # | Topic | What I build | Status |
|---|---|---|---|
| 37 | [LLM Evaluation](37-llm-evaluation/) | `eval/` — golden dataset, metrics, judge, CI harness | ready |
| 38 | [RAG Evaluation](38-rag-evaluation/) | `rag_eval/` — retrieval and generation metrics, separately | ready |
| 39 | [Agent Evaluation](39-agent-evaluation/) | `agent_eval/` — trajectories, outcome categories, failure taxonomy | ready |
| 40 | [Observability](40-observability/) | tracing, cost tracking, dashboards, online evaluation | ready |

## Which things to learn

**37. LLM Evaluation** — why evaluation is statistical, not assertive; golden datasets from *real* inputs
(invented cases flatter you); deterministic checks before judges; LLM-as-judge with rubrics, pairwise
comparison, and **validation against your own labels**; judge biases; the **noise floor** you must measure
before claiming an improvement; regression gates in CI; why public benchmarks can't tell you if your
system is good.

**38. RAG Evaluation** — the two stages fail differently, so measure them separately; hit rate, MRR,
context precision and recall; **faithfulness** as the most important metric; citation accuracy; the
dangerous case where retrieval fails and the model answers from training anyway; why synthetic questions
are systematically easier; production faithfulness monitoring, which needs no ground truth.

**39. Agent Evaluation** — the output is a trajectory; outcome categories with **false success** as the key
number; tool accuracy as the cheapest thing to fix; step efficiency and redundancy; failure taxonomies
that tell you which knob to turn; **cost per successful task**; safe, resettable evaluation environments;
why a single run tells you nothing.

**40. Observability** — why LLM failures are silent; tracing with propagated ids; logging content (and the
obligations that creates); the metrics that catch a cost regression; percentiles not averages; quality
proxies that need no labels; online faithfulness; drift detection; and closing the loop by turning
production failures into evaluation cases.

## Prerequisites

Something built to evaluate — your Phase 5 RAG system and Phase 7/8 agent are the natural subjects. A
budget for evaluation runs: they cost real money, which is itself a design constraint.

**Next:** Phase 10 — what happens when someone attacks it.
