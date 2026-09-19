# AI Engineering & LLM — Learning Workspace

My working repository for [AI_ENGINEERING_LLM_ROADMAP.md](AI_ENGINEERING_LLM_ROADMAP.md): 17 phases,
62 topics, 10 projects, from next-token prediction to a production AI platform.

**How this works:** every topic folder contains a `THEORY.md` — plain-language theory, then numbered
implementation questions with a check for each and a written question to answer. **I write all the code
and all the `notes.md` files.** No solution code exists anywhere in this repository, deliberately.

---

## Progress

Mark topics off as you finish them (all checks passing and `notes.md` written).

### Phase 1 — [LLM Foundations](phase-01-llm-foundations/)
- [ ] 1 Language Modeling · [ ] 2 Tokenization · [ ] 3 Embeddings · [ ] 4 Transformer Architecture · [ ] 5 LLM Generation

### Phase 2 — [Build a Tiny LLM](phase-02-build-a-tiny-llm/)
- [ ] 6 Tokenizer From Scratch · [ ] 7 Dataset Preparation · [ ] 8 Tiny Language Model · [ ] 9 Training · [ ] 10 Inference

### Phase 3 — [Modern LLMs](phase-03-modern-llms/)
- [ ] 11 Transformer Deep Dive · [ ] 12 Modern Architecture · [ ] 13 LLM Training · [ ] 14 Fine-Tuning · [ ] 15 Alignment

### Phase 4 — [LLM Application Engineering](phase-04-llm-application-engineering/)
- [ ] 16 LLM APIs · [ ] 17 Prompt Engineering · [ ] 18 Structured Outputs · [ ] 19 Context Engineering · [ ] 20 Backend

### Phase 5 — [Embeddings & RAG](phase-05-embeddings-and-rag/)
- [ ] 21 Embeddings · [ ] 22 Document Processing · [ ] 23 Vector Databases · [ ] 24 RAG · [ ] 25 Advanced RAG

### Phase 6 — [Tool Use & Function Calling](phase-06-tool-use-and-function-calling/)
- [ ] 26 Tool Calling · [ ] 27 External Tools · [ ] 28 Tool Design

### Phase 7 — [AI Agents](phase-07-ai-agents/)
- [ ] 29 Agent Fundamentals · [ ] 30 Agent Memory · [ ] 31 Agent Architectures · [ ] 32 Agent Frameworks

### Phase 8 — [Advanced Agent Systems](phase-08-advanced-agent-systems/)
- [ ] 33 Orchestration · [ ] 34 Multi-Agent · [ ] 35 Harnesses · [ ] 36 Autonomous Workflows

### Phase 9 — [Evaluation & Reliability](phase-09-ai-evaluation-and-reliability/)
- [ ] 37 LLM Evaluation · [ ] 38 RAG Evaluation · [ ] 39 Agent Evaluation · [ ] 40 Observability

### Phase 10 — [Safety & Security](phase-10-ai-safety-and-security/)
- [ ] 41 LLM Security · [ ] 42 Agent Security · [ ] 43 AI Reliability

### Phase 11 — [Local & Open-Source LLMs](phase-11-local-and-open-source-llms/)
- [ ] 44 Open-Source Models · [ ] 45 Local Inference · [ ] 46 Model Serving

### Phase 12 — [AI Infrastructure](phase-12-ai-infrastructure/)
- [ ] 47 Production Architecture · [ ] 48 Scalability · [ ] 49 Deployment

### Phase 13 — [Advanced Model Engineering](phase-13-advanced-model-engineering/)
- [ ] 50 Model Fine-Tuning · [ ] 51 Model Optimization · [ ] 52 Training Systems

### Phase 14 — [AI Systems & Reasoning](phase-14-ai-systems-and-reasoning/)
- [ ] 53 Reasoning Models · [ ] 54 Memory Systems · [ ] 55 Learning Agents

### Phase 15 — [AGI Concepts](phase-15-agi-concepts/)
- [ ] 56 Intelligence · [ ] 57 AGI Architectures · [ ] 58 Research Topics · [ ] 59 Safety & Alignment

### Phase 16 — [Multimodal AI](phase-16-multimodal-ai/)
- [ ] 60 Vision · [ ] 61 Audio · [ ] 62 Video

### Phase 17 — [Capstone Projects](phase-17-capstone-projects/)
- [ ] 1 Tiny LM · [ ] 2 CLI Assistant · [ ] 3 Chat App · [ ] 4 Document RAG · [ ] 5 Coding Assistant
- [ ] 6 Tool Agent · [ ] 7 Memory Agent · [ ] 8 Research System · [ ] 9 SWE Agent · [ ] 10 Platform

---

## Folder layout

```
phase-NN-name/
  README.md              what to do in this phase, and which things to learn
  NN-topic-name/
    THEORY.md            theory + implementation questions (provided)
    <sample data>        corpora and fixtures where relevant (provided)
    <my code>            I write this
    notes.md             my answers, measurements, and conclusions
```

## The method

From the roadmap, for every topic: understand the concept → learn the mechanism → implement a minimal
version from scratch → use a production library or API → build something small → test and evaluate →
write down what was learned → move on.

Step seven is not optional. `notes.md` is where the topic actually lands, and the "Done when" questions
at the end of each `THEORY.md` are the test.

## Setup

Phases 1 and most of 2 need only Python 3 and the standard library — deliberately, so the mechanisms
stay visible. From Topic 8 onward:

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install torch                      # Topic 8
pip install transformers peft trl datasets     # Phase 3
pip install fastapi uvicorn pydantic redis     # Phase 4
pip install sentence-transformers chromadb pgvector   # Phase 5
```

API keys go in environment variables, never in code, never committed.

## Three habits this roadmap is really teaching

1. **Measure before believing.** Almost every topic ends with a number, and several end with "would you
   ship it?" where the honest answer is no.
2. **Build the minimal version yourself first**, then use the library. You can only evaluate a tool you
   could have written.
3. **Write down what you learned**, including the experiments that failed and the intuitions that turned
   out wrong.

## Asking for help

Bring code for review, ask for a hint, or ask to be quizzed on a topic's "Done when" questions. The one
thing that doesn't happen here is someone else writing the implementation.
