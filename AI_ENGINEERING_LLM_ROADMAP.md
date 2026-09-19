# AI Engineering & LLM Learning Roadmap

## Goal

Progress from existing Python/ML/PyTorch knowledge to practical AI Engineering, LLMs, AI Agents, advanced agent systems, and AGI concepts.

---

# Phase 1 — LLM Foundations

## 1. Language Modeling
- What is a language model
- Next-token prediction
- Autoregressive generation
- Training vs inference
- Context and sequence

## 2. Tokenization
- Words vs tokens
- Subword tokenization
- BPE
- SentencePiece
- Token IDs
- Vocabulary

## 3. Embeddings
- Token embeddings
- Positional embeddings
- Positional encoding
- Vector representations
- Similarity

## 4. Transformer Architecture
- Encoder and decoder
- Self-attention
- Query, Key, Value
- Multi-head attention
- Feed-forward networks
- Residual connections
- Layer normalization
- Causal masking

## 5. LLM Generation
- Logits
- Softmax
- Temperature
- Greedy decoding
- Sampling
- Top-k
- Top-p
- Repetition controls

---

# Phase 2 — Build a Tiny LLM

## 6. Tokenizer From Scratch
- Vocabulary
- Token-to-ID mapping
- ID-to-token mapping
- Encoding
- Decoding

## 7. Dataset Preparation
- Text corpus
- Sequences
- Context windows
- Input/target pairs
- Batching

## 8. Tiny Language Model
- Embedding layer
- Attention
- Transformer block
- Output projection
- Loss function

## 9. Training
- Forward pass
- Cross-entropy loss
- Backpropagation
- Optimizers
- Learning rate
- Training loop
- Checkpoints

## 10. Inference
- Prompting
- Token generation
- Sampling
- Generation loop

---

# Phase 3 — Modern LLMs

## 11. Transformer Deep Dive
- Transformer evolution
- GPT architecture
- Decoder-only models
- Encoder-only models
- Encoder-decoder models

## 12. Modern LLM Architecture
- RMSNorm
- RoPE
- SwiGLU
- KV cache
- Grouped-query attention
- Mixture of Experts

## 13. LLM Training
- Pretraining
- Data pipelines
- Distributed training
- Compute
- Scaling laws
- Checkpointing

## 14. Fine-Tuning
- Transfer learning
- Instruction tuning
- Supervised fine-tuning
- LoRA
- QLoRA
- PEFT

## 15. Alignment
- RLHF
- Preference datasets
- DPO
- Reward models
- Safety alignment

---

# Phase 4 — LLM Application Engineering

## 16. LLM APIs
- API requests
- Messages
- System/user/assistant roles
- Streaming
- Error handling
- Retries
- Rate limits
- Cost management

## 17. Prompt Engineering
- System prompts
- Few-shot prompting
- Structured prompting
- Reasoning strategies
- Prompt templates
- Prompt evaluation

## 18. Structured Outputs
- JSON outputs
- Schemas
- Validation
- Typed responses
- Function/tool schemas

## 19. Context Engineering
- Context windows
- Context selection
- Context compression
- Conversation history
- Long-context strategies

## 20. AI Application Backend
- FastAPI
- Async Python
- Authentication
- Streaming APIs
- Background jobs
- PostgreSQL
- Redis

---

# Phase 5 — Embeddings & RAG

## 21. Embeddings
- Text embeddings
- Embedding models
- Similarity metrics
- Cosine similarity
- Semantic search

## 22. Document Processing
- PDF extraction
- HTML
- Markdown
- Chunking
- Metadata
- Cleaning
- Document ingestion

## 23. Vector Databases
- Vector indexes
- pgvector
- Chroma
- Pinecone
- Weaviate
- Metadata filtering

## 24. RAG
- Basic RAG
- Retrieval
- Context construction
- Generation
- Hybrid search
- Reranking
- Query rewriting
- Multi-query retrieval

## 25. Advanced RAG
- Parent-child retrieval
- Multi-vector retrieval
- Graph RAG
- Agentic RAG
- RAG evaluation

---

# Phase 6 — Tool Use & Function Calling

## 26. Tool Calling
- Function definitions
- Tool schemas
- Tool selection
- Tool execution
- Tool results
- Error handling

## 27. External Tools
- Web search
- Databases
- APIs
- File systems
- Code execution
- Browser automation

## 28. Tool Design
- Tool boundaries
- Tool validation
- Permissions
- Sandboxing
- Reliability

---

# Phase 7 — AI Agents

## 29. Agent Fundamentals
- What is an agent
- Agent loop
- Reasoning and acting
- Planning
- Tool use
- Observation
- Execution

## 30. Agent Memory
- Short-term memory
- Long-term memory
- Conversation memory
- Episodic memory
- Semantic memory
- Memory retrieval

## 31. Agent Architectures
- ReAct
- Router agents
- Planner/executor
- Reflection
- Critic systems
- State machines

## 32. Agent Frameworks
- LangChain
- LangGraph
- LlamaIndex
- Model Context Protocol (MCP)

---

# Phase 8 — Advanced Agent Systems

## 33. Agent Orchestration
- State management
- Workflows
- Checkpoints
- Human-in-the-loop
- Interruptions
- Recovery

## 34. Multi-Agent Systems
- Agent roles
- Communication
- Delegation
- Coordination
- Shared memory
- Agent protocols

## 35. Agent Harnesses
- Execution environments
- Tool management
- Context management
- Permissions
- Sandboxing
- Persistent state

## 36. Autonomous Workflows
- Task decomposition
- Planning
- Execution
- Verification
- Retry
- Self-correction

---

# Phase 9 — AI Evaluation & Reliability

## 37. LLM Evaluation
- Golden datasets
- Automated evaluation
- Human evaluation
- Benchmarking
- Regression testing

## 38. RAG Evaluation
- Retrieval quality
- Relevance
- Faithfulness
- Context precision
- Context recall

## 39. Agent Evaluation
- Task success
- Tool accuracy
- Trajectory evaluation
- Failure analysis

## 40. Observability
- Tracing
- Logging
- Metrics
- Token usage
- Latency
- Cost tracking

---

# Phase 10 — AI Safety & Security

## 41. LLM Security
- Prompt injection
- Jailbreaking
- Data leakage
- Tool abuse
- Model manipulation

## 42. Agent Security
- Permissions
- Sandboxing
- Secrets
- Tool authorization
- Human approval

## 43. AI Reliability
- Hallucination mitigation
- Grounding
- Validation
- Guardrails
- Fallback systems

---

# Phase 11 — Local & Open-Source LLMs

## 44. Open-Source Models
- Llama
- Qwen
- Mistral
- Gemma
- Model selection

## 45. Local Inference
- Ollama
- llama.cpp
- vLLM
- Quantization
- GPU/CPU inference

## 46. Model Serving
- OpenAI-compatible APIs
- Batching
- KV cache
- Throughput
- Latency

---

# Phase 12 — AI Infrastructure

## 47. Production AI Architecture
- AI services
- Model gateways
- Queues
- Caching
- Databases
- Object storage

## 48. Scalability
- Concurrent inference
- Load balancing
- Batch processing
- Rate limiting
- Cost optimization

## 49. Deployment
- Docker
- Kubernetes
- Cloud GPUs
- CI/CD
- Monitoring

---

# Phase 13 — Advanced Model Engineering

## 50. Model Fine-Tuning
- Dataset creation
- Data quality
- SFT
- LoRA
- QLoRA
- Evaluation

## 51. Model Optimization
- Quantization
- Distillation
- Pruning
- Speculative decoding

## 52. Training Systems
- Distributed training
- Data parallelism
- Model parallelism
- Pipeline parallelism
- Mixed precision

---

# Phase 14 — AI Systems & Reasoning

## 53. Reasoning Models
- Reasoning tokens
- Test-time compute
- Search
- Verification
- Self-consistency

## 54. Memory Systems
- External memory
- Knowledge stores
- Memory retrieval
- Memory consolidation

## 55. Learning Agents
- Feedback loops
- Self-improvement
- Evaluation-driven improvement
- Environment interaction

---

# Phase 15 — AGI Concepts

## 56. Intelligence
- Generalization
- Transfer learning
- Planning
- Reasoning
- Memory
- Learning

## 57. AGI Architectures
- Cognitive architectures
- World models
- Agentic systems
- Long-term memory
- Planning systems

## 58. AGI Research Topics
- Scaling
- Emergent capabilities
- Reasoning
- Continual learning
- Embodied intelligence
- Multi-modal intelligence

## 59. AGI Safety & Alignment
- Alignment
- Interpretability
- Robustness
- Corrigibility
- Governance

---

# Phase 16 — Multimodal AI

## 60. Vision
- Vision-language models
- Image understanding
- Vision embeddings
- OCR

## 61. Audio
- Speech recognition
- Text-to-speech
- Audio understanding

## 62. Video
- Video understanding
- Temporal reasoning
- Multimodal agents

---

# Phase 17 — Capstone Projects

## Project 1
Tiny language model from scratch

## Project 2
LLM-powered CLI assistant

## Project 3
Production AI chat application

## Project 4
PDF/document RAG assistant

## Project 5
AI coding assistant

## Project 6
Tool-using AI agent

## Project 7
Long-term memory agent

## Project 8
Multi-agent research system

## Project 9
Autonomous software engineering agent

## Project 10
Production-grade AI platform

---

# Learning Method

For every topic:

1. Understand the concept
2. Learn the underlying mechanism
3. Implement a minimal version from scratch
4. Use a production library/API
5. Build a small project
6. Test and evaluate it
7. Document what was learned
8. Move to the next topic
