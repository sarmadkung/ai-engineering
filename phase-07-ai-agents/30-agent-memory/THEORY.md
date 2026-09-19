# Topic 30 — Agent Memory

**Why this topic:** the model has no memory (Topic 1, §1.5). Everything that feels like memory is
machinery you build. This topic is that machinery — and it's what separates an assistant that knows
you from one that starts fresh every time.

---

# Part 1 — Theory

## 30.1 There is no memory

The model is stateless. Weights are frozen. Between two API calls it retains nothing.

So "memory" always means: **storage outside the model, plus a retrieval strategy that puts the right
part back into the context.** Every design below is a different answer to "what do we store, and
what do we put back?"

## 30.2 Short-term memory (working memory)

The current context window: the conversation so far, recent tool results, the current plan. It's
fast (already in context), complete (full detail), and bounded — it fills up and costs tokens on
every request.

Managing it is Topic 19: a turn/token budget, clearing old tool results, rolling summaries. For
agents specifically, tool results are usually the bulk and the least useful later.

## 30.3 Long-term memory

Anything that must survive beyond the current context. Split by *what kind of thing* is remembered:

**Semantic memory — facts.**
"The user's name is Sarmad." "They deploy on Fridays." "Production is on AWS eu-west-1."
Stored as structured records or short statements, retrieved by relevance or always injected if small.
**Facts must be updatable** — a memory store that only appends will eventually hold both "prefers
Python" and "prefers Go" and the agent will guess.

**Episodic memory — experiences.**
"On 12 March we tried X and it failed because Y." Stored as summaries of past sessions with
timestamps, retrieved by similarity to the current situation. This is what lets an agent say "we
tried this before".

**Procedural memory — how to do things.**
Learned workflows and skills: "to deploy, run these steps". Sometimes stored as instructions the
agent can retrieve, sometimes as reusable tools it writes for itself.

## 30.4 Conversation memory: the practical patterns

For chat specifically, in increasing sophistication:

1. **Full history** — resend everything. Perfect recall, cost grows without bound.
2. **Sliding window** — last N turns. Constant cost, forgets abruptly.
3. **Summary + window** — rolling summary plus recent turns verbatim. The standard.
4. **Retrieval over history** — embed all past turns; retrieve the relevant ones. Recalls turn 3 at
   turn 400 without carrying everything.
5. **Extracted facts + window** — pull durable facts into a store, keep recent turns verbatim.
   The most robust, and best for facts that must not drift.

Patterns 4 and 5 compose well: facts for identity and preferences, retrieval for "what did we
discuss about X?".

## 30.5 Memory retrieval

Storing is easy; retrieving the *right* memory is the hard part. Signals to combine:

- **Relevance** — semantic similarity to the current situation (Phase 5's machinery).
- **Recency** — newer memories usually matter more.
- **Importance** — some memories are critical regardless of relevance (a hard constraint, an
  allergy, a security rule). Score them at write time.
- **Frequency** — repeatedly accessed memories are probably important.

A common scoring function is a weighted combination of these, which is exactly what the
"generative agents" line of research demonstrated.

Then: **how much to inject?** Memory competes with the task for context space, and irrelevant
memories are actively harmful (Topic 19's distraction). A small number of well-chosen memories beats
a large dump.

## 30.6 Writing memories: the hard problems

**What to store?** Not everything — an agent that remembers every utterance has an unsearchable
pile. Extract only durable, reusable facts. Deciding this is itself an LLM task ("is anything here
worth remembering long-term?").

**When to write?** After each turn (expensive, current) or at session end (cheap, loses detail).

**Conflict and update.** When a new fact contradicts a stored one, you must *update*, not append.
That requires finding the related memory first — which means retrieval on write, not just on read.
This is the single most-skipped step in memory implementations.

**Decay and forgetting.** Old, unused, superseded memories should age out. Unbounded growth degrades
retrieval precision for everyone.

**Wrong memories are worse than no memories.** A hallucinated or misextracted "fact" becomes
permanent context that the agent will assert confidently forever. Validate before writing, and store
provenance (which session, which message) so a bad memory can be traced and removed.

## 30.7 Where memory lives

- **Postgres** for structured facts — with real updates, constraints, and audit trails (Topic 20).
- **Vector store** for semantic retrieval over episodes and past turns (Topic 23).
- **A file** the agent reads and writes, which is how coding agents keep project notes — simple,
  inspectable, and version-controllable.
- **Provider memory tooling** where available.

A hybrid is normal: structured facts in a table, episodes in a vector index, and a small
always-injected profile.

## 30.8 Privacy

Memory is persistent personal data, which brings real obligations: users must be able to see, edit
and delete their memories; deletion must be genuine (including from the vector index — Topic 23's
consistency problem); memories must never leak across users or tenants; and sensitive categories
should not be stored at all by default. "The agent remembers" is a feature request that arrives
attached to a compliance requirement.

---

# Part 2 — Questions to implement

Build `memory.py` here, integrated with your Topic 29 agent. Postgres plus your vector store from
Phase 5.

### Q1. Prove statelessness again, at agent level
**Build:** tell your agent a fact in session 1. Start session 2 and ask about it.
**Check:** it doesn't know.
**Explain:** exactly what would have to be stored and injected for it to know?

### Q2. Four conversation strategies
**Build:** full history, sliding window, summary+window, and retrieval-over-history.
**Check:** a 40-turn conversation. At turn 40, ask about something from turn 3. Record tokens per
turn for each strategy.
**Explain:** tabulate recall and cost. Which failed at recall, which at cost?

### Q3. Fact extraction
**Build:** after each turn, extract durable facts as structured records (Topic 18), with
provenance.
**Check:** run a 20-turn conversation and print the resulting store.
**Explain:** how many extracted facts were genuinely durable? How many were noise or wrong?

### Q4. Wrong memories
**Build:** from Q3's output, find or force an incorrect extraction. Then start a new session where
that memory is injected.
**Check:** observe the agent asserting it.
**Explain:** describe the damage. What validation would have caught it at write time?

### Q5. Updates and conflicts
**Build:** the user changes a preference. Implement retrieval-on-write so the new fact *updates*
the old one rather than appending.
**Check:** the store holds exactly one current value, with history if you keep it.
**Explain:** what did append-only behaviour do before the fix?

### Q6. Episodic memory
**Build:** at session end, store a summary of what happened, with outcome and timestamp. Retrieve
relevant episodes at the start of new sessions.
**Check:** run a session that fails, then a similar one. Does the agent reference the earlier
failure?
**Explain:** did it actually use it, or just have it in context? How can you tell?

### Q7. Retrieval scoring
**Build:** a scorer combining relevance, recency, and importance, with tunable weights.
**Check:** on 10 situations, compare memories selected by pure similarity vs your scorer.
**Explain:** which weighting was best? Show a case where recency should beat similarity.

### Q8. How much memory to inject
**Build:** inject 0, 3, 10, and 30 memories.
**Check:** measure task success, tokens, and any distraction effects.
**Explain:** where was the peak? Show one case where an irrelevant memory caused a wrong action.

### Q9. Importance at write time
**Build:** score importance when writing (an LLM call or heuristics), and always inject
high-importance memories regardless of relevance.
**Check:** a critical constraint ("never email customers directly") is respected in a session where
it isn't semantically related to the task.
**Explain:** what happened before you did this?

### Q10. Decay
**Build:** age out memories that are old and unaccessed; keep high-importance ones.
**Check:** simulate 100 sessions of accumulation and measure retrieval precision before and after
decay.
**Explain:** report the precision change. What did you have to be careful not to delete?

### Q11. Procedural memory
**Build:** let the agent store a successful procedure and retrieve it for similar tasks later.
**Check:** measure steps and cost on the second occurrence of a similar task.
**Explain:** did it get faster? What's the risk of reusing a stored procedure blindly?

### Q12. Isolation and deletion
**Build:** per-user memory isolation, plus a real delete that removes from Postgres *and* the vector
index.
**Check:** a test attempting cross-user retrieval fails. After deletion, the memory cannot be
retrieved by any route.
**Explain:** where did the "deleted" memory nearly survive? (Topic 23's consistency trap.)

### Q13. The comparison
**Build:** your agent with no memory, with conversation summary only, and with the full system
(facts + episodes + procedures).
**Check:** a 10-session scenario with continuity requirements. Score continuity, tokens per
session, and cost.
**Explain:** tabulate. Which memory type gave the most value per token? Which would you build first
in a real product?

---

# Done when you can answer

1. Why is there no such thing as model memory?
2. What are semantic, episodic and procedural memory?
3. What are the five conversation-memory patterns, and what does each cost?
4. Which signals decide what to retrieve, and why isn't similarity enough?
5. Why must writes retrieve first?
6. Why are wrong memories worse than no memory?
7. What does deletion actually require?

Write answers in `notes.md`.
