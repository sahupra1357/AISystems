# Part 4 — Generative AI / LLMs: Overview

Deep learning taught you how neural nets train. **Generative AI** and **large language models (LLMs)** shift the day-to-day work: you often start from a **foundation model** trained by others, then adapt it with prompts, retrieval, tools, or light fine-tuning—and you ship that as a **system**, not a single API call.

This part is the **weeks 15–22** style curriculum: using foundation models well, deciding among prompt / RAG / fine-tune, evaluating seriously, and understanding the production shape of an LLM app (orchestrator, retriever, model, tools, logging, cost). Every lesson teaches **why** and **how**—not skimmy definitions.

## Goals for Part 4

By the end of Part 4, you should be able to:

- Explain how next-token prediction becomes chat behavior; reason about tokens, context windows, embeddings, temperature, cost, and latency
- Design strong prompts (anatomy, roles, styles, task playbooks) and iterate with a changelog and golden examples
- Design and debug a **RAG** pipeline: chunking tradeoffs, embedding choice, hybrid retrieval, citations, failure playbooks
- Use **tool / function calling**, structured outputs, and **constrained** agents with schemas, validation, and least privilege
- Choose among prompting, RAG, and fine-tuning with scenario-backed reasoning (including LoRA intuition and data needs)
- Evaluate LLM systems: golden sets, rubrics, pairwise compares, regression gates, online signals
- Sketch a production LLM app architecture that Part 5 can harden (serving, monitoring, safety)

## Prerequisites

Complete **Part 3 — Deep Learning**, or understand Transformers at a high level, train/val discipline, and comfortable Python for API clients and data wrangling.

You will need API access **or** a local open model for many exercises. Notes stay vendor-neutral.

```bash
python -m pip install openai  # or your provider's official SDK
python -m pip install numpy
# Later: vector store client / sentence-transformers / jsonschema / pydantic as needed
```

## Suggested time

About **6–8 weeks** part-time (this is a full systems block, not a quick API tour):

| Week | Focus |
|------|--------|
| 1 | Foundation models basics (01): tokens, embeddings, chat APIs, decoding, cost, failure modes |
| 2 | Prompting deep dive (02): anatomy, roles, first half of prompting styles |
| 3 | Prompting deep dive (02): remaining styles, task playbooks, parameters, iteration, anti-patterns |
| 4 | RAG deep dive (03): chunking, retrieval, hybrid search, citations, debugging |
| 5 | Tools, agents, structured output (04); production app shape |
| 6 | Fine-tune vs RAG vs prompt (05) |
| 7 | Evaluation for LLM systems (06) |
| 8 | Exercises, prompt library + eval capstone, Ready-for-Part-5 checklist (07) |

Experienced practitioners may compress weeks 1–2; still do prompting practice and golden-set work—those catch most production bugs.

## Order to study

1. **[01-foundation-models-basics.md](01-foundation-models-basics.md)**
2. **[02-prompting-deep-dive.md](../../05-prompt-and-context-engineering/learn/02-prompting-deep-dive.md)**
3. **[03-rag-deep-dive.md](../../06-rag/learn/03-rag-deep-dive.md)**
4. **[04-tools-agents-and-structured-output.md](../../07-agents-tools-memory/learn/04-tools-agents-and-structured-output.md)**
5. **[05-fine-tuning-vs-rag-vs-prompt.md](../../08-choosing-the-lever/learn/05-fine-tuning-vs-rag-vs-prompt.md)**
6. **[06-evaluation-for-llm-systems.md](../../09-evaluation/learn/06-evaluation-for-llm-systems.md)**
7. **[07-exercises-and-checklist.md](../../09-evaluation/learn/07-exercises-and-checklist.md)**

## Mental model: an LLM feature is a system

```text
                    ┌──────────────┐
 User request ──►   │ Orchestrator │ ──► response (+ citations / actions)
                    └──────┬───────┘
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
      Retriever         LLM API         Tools
      (RAG index)    (prompt+decode)  (APIs/DBs)
           │               │               │
           └───────────────┴───────────────┘
                           ▼
              Logging · eval hooks · cost meters
```

Lessons 01–02 deepen the LLM box. Lesson 03 deepens the Retriever. Lesson 04 deepens Tools and orchestration. Lessons 05–06 decide adaptation strategy and prove quality. Part 5 hardens the outer ring (serving, monitors, safety, privacy).



## Weeks 15–22 mapping (curriculum view)

If you are following a longer AI engineer path where Part 4 spans roughly weeks 15–22:

| Weeks | Lessons | Outcome |
|-------|---------|---------|
| 15 | 01 Foundation models basics | Comfortable with tokens, APIs, cost, failure modes |
| 16–17 | 02 Prompting deep dive | Style selection + shipping checklist habits |
| 18 | 03 RAG deep dive | Designed and debugged grounded Q&A |
| 19 | 04 Tools / agents / structured output | Safe tool schemas and capped orchestration |
| 20 | 05 Fine-tune vs RAG vs prompt | Decision memos with baselines |
| 21 | 06 Evaluation | Gold set + regression gate |
| 22 | 07 Capstone week | Prompt library + eval harness (+ optional RAG) |

## Success criteria before Part 5

You are ready when you can:

- Explain tokens, context, embeddings vs generative models, and cost/latency tradeoffs with failure modes
- Write prompts with clear anatomy and choose styles deliberately (not only zero-shot vibes)
- Design RAG with chunking/retrieval rationale and a debugging playbook
- Define tool schemas with validation and least privilege; explain agent loops vs single tool calls
- Defend prompt vs RAG vs fine-tune for concrete scenarios
- Ship a golden set and describe a regression gate
- Draw the orchestrator/retriever/LLM/tools/logging shape for a Docs Q&A or support assistant

Part 5 puts these systems behind APIs, monitors, and safety controls.
