# Project 7 — Long-Term Memory Agent

**Depends on:** Topic 30, Topic 54, Phase 5 for retrieval, Topic 20 for storage.

**What you prove:** that you can build memory that genuinely accumulates — facts that update rather than
pile up, experiences that inform later sessions, and deletion that actually deletes. This is one of the
hardest problems in applied AI and the least well solved in real products.

---

# Part 1 — What you are building

An assistant that knows you across months of use:

```
conversation -> extract durable facts -> store (structured + episodic)
new session -> retrieve relevant memories -> inject -> respond with continuity
periodically -> consolidate, reconcile, forget
```

Pick a domain where continuity obviously matters: a personal assistant, a tutor tracking what you've
learned, a project companion, or a health/habit tracker.

---

# Part 2 — Design decisions

| Decision | Options | Consider |
|---|---|---|
| Fact storage | Postgres / documents / graph | updates and constraints argue for relational |
| Episode storage | vector store | semantic recall of past sessions |
| Extraction timing | per turn / per session | cost versus freshness |
| Retrieval scoring | similarity / +recency +importance | similarity alone is not enough (Topic 30) |
| Injection budget | fixed count / token budget | memory competes with the task (Topic 19) |
| Consolidation | never / scheduled / threshold | necessary eventually (Topic 54) |
| Forgetting | never / decay | unbounded stores degrade precision |

---

# Part 3 — Milestones

### M1 — The continuity test set (Topic 54)
**Before building:** a scripted 20-session scenario with facts stated early and needed late,
preferences that change, and experiences that should inform later sessions. Plus 30 checkable questions
for the end.
**Check:** a harness that runs the scenario and scores continuity.
**Explain:** which requirements do you predict will fail?

### M2 — No-memory baseline
Run the scenario with no memory at all.
**Check:** report the continuity score.
**Explain:** this is the bar. Which failures look most fixable?

### M3 — Fact extraction (Topics 18, 30)
Structured extraction of durable facts per turn, with provenance (session, message, time) and confidence.
**Check:** run 5 sessions and read the store. Report how many extracted facts were durable, noise, or
wrong.

### M4 — False memory handling (Topic 30)
From M3's output, find a wrong extraction. Add validation before writing.
**Check:** demonstrate the damage a false memory does, then show your validation catching it.

### M5 — Updates, not appends (Topics 30, 54)
Retrieval-on-write so a new fact updates a contradicting one; temporal validity (`valid_from`,
`valid_to`); reconciliation by recency plus confidence.
**Check:** the user changes a preference; the store holds one current value plus history. Ask "what is
true now?" and "what was true last month?"

### M6 — Episodic memory (Topic 30)
Session summaries with outcomes, retrieved by similarity to the current situation.
**Check:** run a session that fails, then a similar one. Does the agent reference the earlier attempt?

### M7 — Retrieval scoring (Topics 30, 54)
Combine relevance, recency and importance, with importance scored at write time and high-importance
memories always injected.
**Check:** compare selected memories against pure similarity on 10 situations. Show a case where recency
should win.

### M8 — Injection budget (Topic 19)
Test 0, 3, 10, 30 memories per turn.
**Check:** report task success and any distraction effects. Show a case where an irrelevant memory caused
a wrong action.

### M9 — Consolidation (Topic 54)
Scheduled deduplication, merging with rising confidence, episode summarization, and cautious abstraction
into rules.
**Check:** report store size and retrieval precision before and after. Then force an over-generalization
scenario and report what it concluded.

### M10 — Forgetting and privacy (Topics 41, 54)
Decay by age, access and importance; per-user isolation; genuine deletion from Postgres **and** the
vector index; a user-facing view of their own memories.
**Check:** a cross-user retrieval test fails. After deletion, the memory is unreachable by any route.
Simulate 100 sessions and report precision with and without decay.

---

# Part 4 — Experiments

1. **Memory strategies** — none / summary-only / facts+episodes. Continuity score, tokens per session,
   cost.
2. **Recall measurement** — 30 questions whose answers are definitely stored. Report recall, and for the
   failures determine whether storage or retrieval was at fault (Topic 54's insidious failure).
3. **Compositional recall** — 5 questions needing two stored facts combined. Plain retrieval versus
   multi-hop.
4. **Consolidation effect** — precision and store size over 100 simulated sessions, with and without.
5. **Summary drift** — after 40 turns of rolling summarization, compare the summary against the
   transcript. What changed or was invented?

---

# Part 5 — Deliverables

- The agent with working memory across sessions
- `EVALUATION.md` — the 20-session scenario, continuity scores, and memory metrics (recall, precision,
  correctness, staleness)
- `ARCHITECTURE.md` — storage design and schemas
- `RESULTS.md` — experiment tables
- `PRIVACY.md` — what's stored, retention, isolation, deletion, and the user's view of their data

---

# Part 6 — Done when

- It remembers what matters across 20 sessions and you can prove it with a score.
- A changed preference updates rather than accumulating a contradiction.
- Memory recall is measured, and you know whether failures are storage or retrieval.
- Wrong memories are caught before they become permanent.
- Deletion is complete, and cross-user leakage is impossible with a test proving it.

---

# Stretch

Graph-structured memory for multi-hop questions (Topic 25); procedural memory the agent writes and
reuses (Topic 30); memory an LLM can *edit* through a tool rather than only read; confidence decay for
unverified facts; a memory review UI where the user corrects the agent's beliefs.
