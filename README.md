# AISystems

A beginner-to-production path for becoming an AI engineer, organised as 12 stages.

**Start here: [`AI_ENGINEERING_GUIDE.md`](AI_ENGINEERING_GUIDE.md).** It summarises each stage, gives the rules worth memorising, and links into every file below.

## Stages

| # | Stage | What it covers |
|---|---|---|
| 1 | [Foundations](01-foundations/README.md) | Python, math intuition, data literacy |
| 2 | [Classical ML](02-classical-ml/README.md) | Core algorithms, scikit-learn workflow, honest evaluation |
| 3 | [Deep learning](03-deep-learning/README.md) | Neural nets, the PyTorch loop, CNNs and Transformers |
| 4 | [LLM foundations](04-llm-foundations/README.md) | Tokens, cost, failure modes, model selection and routing |
| 5 | [Prompt and context engineering](05-prompt-and-context-engineering/README.md) | Prompt structure, techniques, structured output, context management |
| 6 | [RAG](06-rag/README.md) | Ingestion and parsing through chunking, retrieval, reranking and grounding |
| 7 | [Agents, tools and memory](07-agents-tools-memory/README.md) | Tool design, agent loops, HITL, sandboxing, budgets, memory |
| 8 | [Choosing the lever](08-choosing-the-lever/README.md) | Prompt vs RAG vs fine-tune; SFT, LoRA, DPO, distillation |
| 9 | [Evaluation](09-evaluation/README.md) | Golden sets, LLM-as-judge, RAG and agent evals, testing pyramid |
| 10 | [Production](10-production/README.md) | Decision framework, MLOps, serving, GPUs, monitoring, safety, reliability |
| 11 | [Architecture patterns](11-architecture-patterns/README.md) | 15 production patterns and the reference architecture |
| 12 | [Capstones and interviews](12-capstones-and-interviews/README.md) | Build guides, portfolio, 60 scenario drills |

Each stage folder has:

- **`learn/`**: curriculum lessons, exercises and checklists that explain why things work.
- **`apply/`**: field-manual scenarios, each with implementation steps, an example, trade-offs, production notes, a checklist and metrics (stages 4 to 12).
- **`README.md`**: every file in the stage, in reading order.

Also at the root:

- **[`AI_System.md`](AI_System.md)**: the whole field manual on one page, in build order (ingest → parse → chunk → embed → index → retrieve → rerank → generate → evaluate → serve → operate). The `apply/` file numbers match its section numbers.

## How to use these notes

1. **Read in order.** Start each stage with its README, then the lessons in `learn/`, then the exercises and checklist.
2. **Type the code yourself.** Copy-paste is fine for checking answers, but muscle memory comes from typing and breaking things.
3. **Use `apply/` when you build.** Skim it during study so you know it exists, then come back to the exact scenario when you need it.
4. **Finish each checklist** before moving on, and revisit earlier stages freely.

Suggested pace, part-time: 2–4 weeks each for stages 1–3, 6–8 weeks for stages 4–9, 4–6 weeks for stages 10–11, and 3–6 weeks for a solid capstone in stage 12. The guide also has a fast track and an interview sprint.
