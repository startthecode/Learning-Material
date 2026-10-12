# 6-Month AI Systems Engineer Roadmap
### For a Java / Spring Boot / Kafka / PostgreSQL / Docker developer

This roadmap keeps your backend foundation intact and layers AI systems engineering on top of it. Fine-tuning sits in Month 5, evaluation runs continuously from Month 1 onward, and model serving is folded into Month 6 alongside security. Full ML/DL theory (backprop derivations, CNNs, RNNs, optimizer internals) is deliberately deferred to an optional Month 7+ track — you get just enough ML vocabulary early to not be lost once fine-tuning starts, without spending weeks on math before building anything.

---

## Month 1 — Python (fast) · LLM Fundamentals · Basic ML Concepts
**Topics:** Python syntax for developers coming from another language · tokens · context windows · embeddings · attention · transformers · inference · structured output · hallucination · temperature · train/test split · loss · overfitting · classification basics · precision/recall · gradient descent (conceptual, not derived) · tiny eval set for your own test prompts

You already think in types, APIs, and services — Python syntax is days, not weeks, for you. Spend the bulk of the month on LLM concepts and just enough ML vocabulary that fine-tuning in Month 5 won't feel foreign.

**Courses:**
- Claude 101 — Anthropic Academy (free)
- AI Capabilities and Limitations — Anthropic Academy (free) — covers next-token prediction, pretraining vs. fine-tuning, knowledge limitations, context/working-memory limits
- Any short "Python for developers from another language" crash course for syntax

**Deliverable:** a small set of test prompts with expected behavior — your first (tiny) eval set. Keep it; you'll extend it every month.

---

## Month 2 — RAG · Retrieval · Retrieval Evaluation
**Topics:** chunking · embeddings · vector search · BM25 · hybrid search · reciprocal rank fusion · reranking · metadata filtering · context limits · citation/grounding · retrieval evaluation metrics

Learn retrieval, not just "how to use a vector database" — understand *why* retrieval fails before you trust any RAG pipeline.

**Courses:**
- Retrieval Augmented Generation — DeepLearning.AI (Zain Hasan) — semantic search, BM25, hybrid retrieval, chunking, evaluation, production/deployment
- Building and Evaluating Advanced RAG — DeepLearning.AI (Jerry Liu)

**Project (Part 1):** AI documentation assistant for your own Spring Boot microservices — feed it your Product/Inventory API docs, Kafka event schemas, outbox pattern notes, architecture decisions. Test with ~50 real questions and score retrieval quality, not just final answers.

---

## Month 3 — Tool Calling · MCP · Tool-Selection Evaluation
**Topics:** tool schemas · function calling · validation · authorization · retries · timeouts · idempotency · error handling · tool result formats · MCP architecture (client-server) · MCP tools/resources/prompts · building and deploying an MCP server · tool-selection eval

Learn tool architecture first, MCP second — MCP is a standard for something you should already understand conceptually.

**Courses:**
- Building with the Claude API — Anthropic Academy (free)
- MCP: Build Rich-Context AI Apps with Anthropic — DeepLearning.AI (free) — note: this course had SDK-breaking changes in July 2026, a good reminder to learn the protocol/concepts rather than copy exact code

**Project (Part 2, start):** wrap 3–4 Spring Boot endpoints (`getProduct`, `getInventory`, `getOrder`) as callable tools with validation, timeouts, and permission checks. Add a handful of test cases checking the model picks the *correct* tool for a given question.

---

## Month 4 — Agents · Context Engineering · Agent-Trajectory Evaluation
**Topics:** agent loops · planning · tool selection · memory · multi-step workflows · human-in-the-loop approval · guardrails · context prioritization · context compression/summarization · state management · agent-trajectory evaluation

The agent loop itself is the easy part. The hard question is what information the model should receive at each step — put context engineering ahead of framework syntax.

**Courses:**
- Functions, Tools and Agents with LangChain — DeepLearning.AI (learn the patterns, not "become a LangChain expert" — frameworks change, concepts don't)
- Anthropic's *Effective Context Engineering for AI Agents* engineering post (no dedicated course exists yet)

**Project (Part 2, finish):** full agent loop over your Month 3 tools. Add trajectory checks — did it call tools in a sensible order, did it stop in a reasonable number of steps.

---

## Month 5 — Fine-Tuning · Post-Training · Model Evaluation
**Topics:** when to fine-tune vs. prompt/RAG · dataset preparation and validation · supervised fine-tuning (SFT) · instruction tuning · PEFT · LoRA · QLoRA · quantization · learning rate/epochs/batch size (practical, not derived) · RLHF and reward modeling (conceptual) · PPO/GRPO (conceptual) · synthetic data · reward hacking · baseline-vs-fine-tuned comparison · model evaluation

The point isn't that you'll fine-tune weekly — it's understanding the full lifecycle (base model → dataset → fine-tune → evaluate → compare) well enough to know when *not* to fine-tune, which is most of the time.

**Courses:**
- Finetuning Large Language Models — DeepLearning.AI (Sharon Zhou) — fundamentals, when to fine-tune, training and evaluation
- Fine-tuning & RL for LLMs: Intro to Post-training — DeepLearning.AI + AMD (Sharon Zhou), ~6h10m — goes beyond basic fine-tuning into SFT, RLHF, reward modeling, PPO/GRPO, LoRA, evaluation, error analysis, red teaming, and production go/no-go decisions
- Evaluating AI Agents — DeepLearning.AI (built with Arize AI) — code-based vs. LLM-as-judge evaluators, experiments, monitoring

**Project (Part 3, start):** take a small open base model (7–8B range) → validate your dataset → evaluate the baseline → fine-tune with QLoRA on a few hundred examples → evaluate again → compare. If it looks better, check whether it's genuinely better or your eval set is the weak link — this is the step people skip.

---

## Month 6 — MLOps · Model Serving · Security · Continuous Evaluation
**Topics:** quantization for serving · inference servers · latency and throughput · batching · GPU memory and concurrency · streaming responses · autoscaling · model versioning · prompt versioning · tracing · monitoring · continuous/regression evaluation pipelines · prompt injection (direct and indirect) · data exfiltration · tool-permission scoping (read/write/delete boundaries) · secrets management · tenant isolation · audit logging · red teaming

Model serving is the piece most roadmaps skip — understanding how an open model actually gets into production, even at a conceptual level, closes the gap between "I fine-tuned a model" and "I can ship one."

**Courses:**
- Red Teaming LLM Applications — DeepLearning.AI — prompt injection, jailbreaks, systematic testing
- Agent Observability & Evaluation — LangChain Academy (LangSmith) — tracing and production debugging, turning traces into repeatable regression tests

**Project (Part 3, finish) — capstone:** the full AI E-commerce Operations Platform — RAG over your docs, tool calls into Spring Boot services via MCP, Kafka event awareness, a model router choosing between the general model and your Month 5 fine-tuned model, a continuous eval suite running across all components, tracing/observability, and explicit read/write/delete permission boundaries per tool.

---

## Quick reference table

| Month | Focus | Continuous thread | Best course(s) | Cost |
|---|---|---|---|---|
| 1 | Python + LLM fundamentals + basic ML concepts | Start tiny eval set | Claude 101 + AI Capabilities & Limitations | Free |
| 2 | RAG + retrieval | Retrieval evaluation | Retrieval Augmented Generation (DeepLearning.AI) | Free |
| 3 | Tool calling + MCP | Tool-selection evaluation | Building with the Claude API + MCP course | Free |
| 4 | Agents + context engineering | Agent-trajectory evaluation | Functions, Tools and Agents with LangChain | Free |
| 5 | Fine-tuning + post-training | Model evaluation | Finetuning LLMs + Fine-tuning & RL for LLMs: Intro to Post-training + Evaluating AI Agents | Free |
| 6 | MLOps + model serving + security | Continuous/regression evaluation | Red Teaming LLM Applications + LangSmith Observability | Free |

---

## What's deliberately excluded

Backpropagation derivations, optimizer internals, CNNs, RNNs, GANs, and formal linear algebra/calculus are not in this 6-month plan. You get the practical vocabulary (loss, overfitting, gradient descent as a concept) needed to not be lost during fine-tuning — not the underlying math. That depth is optional, and belongs in Month 7+ only if you decide to specialize toward LLM Engineer or ML Engineer rather than AI Systems Engineer.

## One system, not three toy projects

The three project phases above are not separate demos — they're one system built incrementally: a RAG-and-tool-calling assistant over your own Spring Boot/Kafka services, which gains a fine-tuned model option in Month 5 and production hardening in Month 6. By the end, your portfolio demonstrates RAG + agents + MCP + Spring Boot + Kafka + fine-tuning + evaluation + observability + security + production deployment — not "I built a chatbot."

## Career framing

Keep your identity as a **Java/Spring Boot backend engineer who specializes in AI systems** — that combination (distributed systems + databases + Kafka + concurrency, now paired with LLMs + RAG + agents + fine-tuning + evals + MLOps) is harder to replace than either half alone. Target titles: **AI Engineer**, **AI Systems Engineer**, or **GenAI Engineer** — avoid narrowing to "AI Agent Developer," which undersells the systems work.
