# Phase 10 — AI Safety & Security

**Goal:** understand the vulnerability class that traditional security doesn't cover, and build systems
whose worst day is survivable. Defensive throughout — you attack only what you built.

---

## What I will do

| # | Topic | What I build | Status |
|---|---|---|---|
| 41 | [LLM Security](41-llm-security/) | `security/` — an attack suite against your own systems | ready |
| 42 | [Agent Security](42-agent-security/) | `agent_security/` — least privilege, egress control, approval, audit | ready |
| 43 | [AI Reliability](43-ai-reliability/) | `reliability/` — grounding, validation, guardrails, fallbacks | ready |

## Which things to learn

**41. LLM Security** — the root cause: the model cannot distinguish instructions from data, because it's
all one token stream; direct versus **indirect** prompt injection (the dangerous one, arriving through
content your own pipeline fetched); why **least privilege is the only reliable defence**; jailbreaking and
what's your problem versus the provider's; the five data-leakage paths; tool abuse; denial of wallet;
RAG poisoning; defence in depth.

**42. Agent Security** — the three ingredients that make an agent hazardous (untrusted input + private
data + the ability to act); **acting as the user** rather than the system; egress control as the
highest-value single control; secrets injected at the edge and never in context; error sanitization;
authorization from the session, never from model arguments; approval prompts that actually inform;
anomaly detection, kill switches, and an audit trail that can answer "what did it touch?"; the deployment
ladder.

**43. AI Reliability** — why hallucination is a consequence of the mechanism, not a bug; grounding, and
why it doesn't guarantee groundedness; the four validation layers; guardrails and their honest costs;
uncertainty signals and why stated confidence is poorly calibrated; fallbacks; **designing for wrongness**
— reversibility, blast-radius limits, transparency, honest UX; matching autonomy to consequence; a
reliability budget as a release gate.

## Prerequisites

Phases 5–8 (systems worth attacking) and Phase 9 (you cannot improve reliability without measurement).

**Scope discipline:** every attack in this phase targets your own systems, in a sandbox, to make them
stronger. That's the whole point.

**Next:** Phase 11 — bringing models onto your own hardware.
