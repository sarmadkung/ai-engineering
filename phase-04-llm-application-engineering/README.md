# Phase 4 — LLM Application Engineering

**Goal:** build real applications on hosted models. This is the phase whose skills you will use every
working day — and the one where your Phase 1–3 knowledge stops being academic, because every API
parameter is something you implemented yourself.

---

## What I will do

| # | Topic | What I build | Status |
|---|---|---|---|
| 16 | [LLM APIs](16-llm-apis/) | `client.py`, `retry.py`, `costs.py` — a properly-built API layer | ready |
| 17 | [Prompt Engineering](17-prompt-engineering/) | `prompts.py`, `evaluate.py` — versioned prompts with a test harness | ready |
| 18 | [Structured Outputs](18-structured-outputs/) | `extract.py`, `schemas.py` — reliable typed output | ready |
| 19 | [Context Engineering](19-context-engineering/) | `context.py` — a budgeted context assembler | ready |
| 20 | [AI Application Backend](20-ai-application-backend/) | a real FastAPI service with Postgres and Redis | ready |

## Which things to learn

**16. LLM APIs** — requests and messages; system/user/assistant roles; statelessness and what resending
history costs; model tiers and why output tokens cost more; streaming and TTFT; which errors to retry and
how; rate limits; token counting; **prompt caching** and what silently breaks it; the numbers you must
log from day one.

**17. Prompt Engineering** — why a prompt conditions rather than instructs; system prompts and cache
stability; few-shot examples and consistency; delimiters and structure; chain of thought and why order
matters; templates as versioned artifacts; and prompt evaluation, without which quality plateaus.

**18. Structured Outputs** — why malformed JSON is intermittent; ask-nicely, defensive parsing,
validate-and-retry, constrained decoding; schema design (enums, optionality, escape hatches); why
constrained decoding guarantees shape but not correctness; tool schemas as the same idea.

**19. Context Engineering** — the window as a budget; lost-in-the-middle; context rot; selection by
relevance, recency and importance; truncation, summarization, rolling summaries and fact extraction;
long-context strategies (map-reduce, refine, retrieval); deciding long context versus retrieval with
numbers.

**20. AI Application Backend** — what makes an AI backend different; async and the one mistake that
negates it; SSE streaming with mid-stream errors and disconnect handling; auth, per-user quotas and cost
attribution; background jobs with idempotency and cancellation; Postgres and Redis; observability from
day one.

## Prerequisites

An API key (set as an environment variable — never in code). `fastapi`, `uvicorn`, `pydantic`, `redis`,
a Postgres instance (Docker is easiest), and the provider SDK.

**Next:** Phase 5 — giving your application access to your own data.
