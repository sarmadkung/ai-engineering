# Phase 17 — Capstone Projects

**Goal:** ten projects that turn 62 topics of knowledge into things that exist, work, and can be shown to
other people. Each has a `THEORY.md` with design decisions, milestones with checks, experiments, and
done-when criteria.

**You don't have to do all ten.** Projects 1, 3, 4 and 6 cover the most ground; 5 and 9 are the most
impressive if you want to work on agents; 10 is the one that demonstrates seniority.

---

## The projects

| # | Project | Depends on | What it proves |
|---|---|---|---|
| 1 | [Tiny Language Model](project-01-tiny-language-model/) | Phases 1–2 | you understand LLMs mechanically |
| 2 | [LLM CLI Assistant](project-02-llm-cli-assistant/) | Phase 4 | you can ship a small, good product |
| 3 | [Production Chat App](project-03-production-chat-app/) | Phases 4, 9, 10 | you can ship a real multi-user AI product |
| 4 | [Document RAG Assistant](project-04-document-rag-assistant/) | Phases 5, 9 | retrieval that works, proven with numbers |
| 5 | [AI Coding Assistant](project-05-ai-coding-assistant/) | Phases 6–8 | you can build a harness |
| 6 | [Tool-Using Agent](project-06-tool-using-agent/) | Phases 6, 7, 10 | real capability without real danger |
| 7 | [Long-Term Memory Agent](project-07-long-term-memory-agent/) | Topics 30, 54 | memory that accumulates correctly |
| 8 | [Multi-Agent Research System](project-08-multi-agent-research-system/) | Phase 8 | multi-agent, honestly evaluated |
| 9 | [Autonomous SWE Agent](project-09-autonomous-swe-agent/) | Projects 5 + Phase 8 | long-horizon autonomy where it's feasible |
| 10 | [Production AI Platform](project-10-production-ai-platform/) | everything | you can build what others build on |

## A suggested order

1. **Project 1** as soon as Phase 2 is done — it's the foundation everything else rests on.
2. **Project 2** after Phase 4 — small, immediately useful, and you'll use it while doing the rest.
3. **Project 4** after Phase 5 — the most commercially common application.
4. **Project 3** after Phase 9/10 — it needs evaluation and security to be worth calling production.
5. **Projects 5–8** after Phases 7–8, in whatever order interests you.
6. **Projects 9 and 10** last — they build on the others.

## What every project must have

Not optional, because these are what distinguish a project from a demo:

- **A `DECISIONS.md`** written *before* the code, recording the choices and reasons.
- **Measurement.** An evaluation set, and numbers for every claim you make.
- **A `RESULTS.md`** with the experiments and their outcomes — including the ones that didn't work.
- **Honest limitations.** What it fails at, what you'd fix next, what you'd need to ship it for real.
- **Cost figures.** Per request, per task, or per user. If you don't know, you haven't finished.

## The habit worth keeping

Every project's `THEORY.md` ends with a "would you ship it?" question, and for several the right answer
will be no. Being able to say that, with numbers behind it, is more valuable than a portfolio of demos
that were never measured.
