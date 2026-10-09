# AI Engineer Fundamentals

A beginner-to-advanced path for building real AI systems—not just calling APIs, but understanding data, models, evaluation, and production habits.

These notes start from absolute zero. If you can open a terminal and stay curious, you can follow along. The path moves from foundations through classical ML, deep learning, generative AI, production systems, and portfolio capstones.

## Learning path

| Part | Topic | Status |
|------|--------|--------|
| **[Part 1](part-01-foundations/00-overview.md)** | **Foundations** (Python, math intuition, data literacy) | **Complete** |
| **[Part 2](part-02-classical-ml/00-overview.md)** | **Classical ML** | **Available** |
| **[Part 3](part-03-deep-learning/00-overview.md)** | **Deep Learning** | **Available** |
| **[Part 4](part-04-generative-ai-llms/00-overview.md)** | **Generative AI / LLMs** | **Available** |
| **[Part 5](part-05-production-ai/00-overview.md)** | **Production AI systems** | **Available** |
| **[Part 6](part-06-capstones/00-overview.md)** | **Capstones** | **Available** |

## How to use these notes

1. **Read in order.** Start each part with its overview, then lessons in numbered order, then the exercises/checklist (or build guides in Part 6).
2. **Type the code yourself.** Copy-paste is fine for checking answers, but muscle memory comes from typing and breaking things.
3. **Do the mini practices** at the end of sections before moving on.
4. **Finish each part’s checklist** before starting the next part.
5. **Revisit freely.** Earlier parts are meant to be skimmed again when later material feels fuzzy.

Suggested pace: roughly **2–4 weeks per part** part-time for Parts 1–3, **6–8 weeks** for Part 4 (LLM systems depth), **4–6 weeks** for Part 5 (production depth: tradeoffs, eval, HITL), then **3–6 weeks** for a solid Part 6 capstone. Adjust for prior experience.

## All parts — file index

### Part 1 — Foundations

- [part-01-foundations/00-overview.md](part-01-foundations/00-overview.md) — Goals, prerequisites, study order, success criteria
- [part-01-foundations/01-python-programming.md](part-01-foundations/01-python-programming.md) — Python from zero for AI work
- [part-01-foundations/02-math-for-ai.md](part-01-foundations/02-math-for-ai.md) — Linear algebra, probability, calculus, statistics intuition
- [part-01-foundations/03-data-literacy.md](part-01-foundations/03-data-literacy.md) — Data types, cleaning, splits, metrics
- [part-01-foundations/04-exercises-and-checklist.md](part-01-foundations/04-exercises-and-checklist.md) — Exercises, mini capstone, “Ready for Part 2” checklist

### Part 2 — Classical ML

- [part-02-classical-ml/00-overview.md](part-02-classical-ml/00-overview.md) — Goals, prerequisites, study order, success criteria
- [part-02-classical-ml/01-ml-concepts.md](part-02-classical-ml/01-ml-concepts.md) — Supervised/unsupervised, features, overfitting, bias–variance
- [part-02-classical-ml/02-supervised-algorithms.md](part-02-classical-ml/02-supervised-algorithms.md) — Linear/logistic, trees, forests, boosting, k-NN, k-means
- [part-02-classical-ml/03-scikit-learn-workflow.md](part-02-classical-ml/03-scikit-learn-workflow.md) — Pipelines, CV, metrics, baselines
- [part-02-classical-ml/04-exercises-and-checklist.md](part-02-classical-ml/04-exercises-and-checklist.md) — Exercises, capstone, “Ready for Part 3” checklist

### Part 3 — Deep Learning

- [part-03-deep-learning/00-overview.md](part-03-deep-learning/00-overview.md) — Goals, prerequisites, study order, success criteria
- [part-03-deep-learning/01-neural-net-basics.md](part-03-deep-learning/01-neural-net-basics.md) — Neurons, layers, activations, loss, backprop, optimizers
- [part-03-deep-learning/02-pytorch-training-loop.md](part-03-deep-learning/02-pytorch-training-loop.md) — Tensors, Dataset/DataLoader, training loop, GPU note
- [part-03-deep-learning/03-cnns-and-transformers.md](part-03-deep-learning/03-cnns-and-transformers.md) — CNNs, attention/Transformers, when to use which
- [part-03-deep-learning/04-exercises-and-checklist.md](part-03-deep-learning/04-exercises-and-checklist.md) — MNIST MLP, CNN/fine-tune, tiny Transformer path, checklist

### Part 4 — Generative AI / LLMs

- [part-04-generative-ai-llms/00-overview.md](part-04-generative-ai-llms/00-overview.md) — Goals, 6–8 week study order, system architecture mental model, success criteria
- [part-04-generative-ai-llms/01-foundation-models-basics.md](part-04-generative-ai-llms/01-foundation-models-basics.md) — Tokens, embeddings, chat APIs, decoding, cost/latency, failure modes
- [part-04-generative-ai-llms/02-prompting-deep-dive.md](part-04-generative-ai-llms/02-prompting-deep-dive.md) — Prompt anatomy, roles, prompting styles, task playbooks, parameters, iteration
- [part-04-generative-ai-llms/03-rag-deep-dive.md](part-04-generative-ai-llms/03-rag-deep-dive.md) — Chunking, hybrid retrieval, citations, ACL, RAG debugging
- [part-04-generative-ai-llms/04-tools-agents-and-structured-output.md](part-04-generative-ai-llms/04-tools-agents-and-structured-output.md) — Function calling, schemas, validation, constrained agents, least privilege
- [part-04-generative-ai-llms/05-fine-tuning-vs-rag-vs-prompt.md](part-04-generative-ai-llms/05-fine-tuning-vs-rag-vs-prompt.md) — Decision framework, scenarios, LoRA/PEFT, data and maintenance
- [part-04-generative-ai-llms/06-evaluation-for-llm-systems.md](part-04-generative-ai-llms/06-evaluation-for-llm-systems.md) — Golden sets, rubrics, regressions, online signals
- [part-04-generative-ai-llms/07-exercises-and-checklist.md](part-04-generative-ai-llms/07-exercises-and-checklist.md) — Exercises, prompt library + eval capstone, Ready-for-Part-5 checklist

### Part 5 — Production AI systems

- [part-05-production-ai/00-overview.md](part-05-production-ai/00-overview.md) — Goals, 4–6 week plan, production loop mental model, eval+HITL teaching contract
- [part-05-production-ai/01-production-mindset-and-decision-framework.md](part-05-production-ai/01-production-mindset-and-decision-framework.md) — Decision framework, tradeoffs, risk tiers, worked examples
- [part-05-production-ai/02-data-pipelines-and-mlops.md](part-05-production-ai/02-data-pipelines-and-mlops.md) — Versioning options, features/skew, registry, eval gates, data HITL
- [part-05-production-ai/03-serving-architectures.md](part-05-production-ai/03-serving-architectures.md) — Batch/API/queue/stream, FastAPI, caching, routing, rollouts
- [part-05-production-ai/04-monitoring-drift-and-cost.md](part-05-production-ai/04-monitoring-drift-and-cost.md) — Logging, quality monitors, drift, SLOs, review queues
- [part-05-production-ai/05-safety-privacy-and-hitl.md](part-05-production-ai/05-safety-privacy-and-hitl.md) — PII, injection, guardrails, HITL rates, safety eval, kill switches
- [part-05-production-ai/06-reliability-and-incident-response.md](part-05-production-ai/06-reliability-and-incident-response.md) — Timeouts, fallbacks, runbooks, rollback, postmortems
- [part-05-production-ai/07-exercises-and-checklist.md](part-05-production-ai/07-exercises-and-checklist.md) — Exercises, production design-doc capstone, Ready-for-Part-6 checklist

### Part 6 — Capstones

- [part-06-capstones/00-overview.md](part-06-capstones/00-overview.md) — How to pick and ship a portfolio project
- [part-06-capstones/01-project-catalog.md](part-06-capstones/01-project-catalog.md) — Project ideas with difficulty and skills
- [part-06-capstones/02-build-guide-docs-qa-rag.md](part-06-capstones/02-build-guide-docs-qa-rag.md) — End-to-end Docs Q&A RAG build guide
- [part-06-capstones/03-build-guide-tabular-plus-llm.md](part-06-capstones/03-build-guide-tabular-plus-llm.md) — Tabular ML + LLM report hybrid build guide
- [part-06-capstones/04-portfolio-and-next-steps.md](part-06-capstones/04-portfolio-and-next-steps.md) — Portfolio tips and what to learn next

## What you will build toward

By the end of the full path you should be able to:

- Clean and split data honestly
- Train and evaluate classical and deep models
- Work with generative models and LLMs thoughtfully (prompts, RAG, evaluation)
- Ship reliable, monitored AI features with safety and cost awareness
- Present a portfolio-ready capstone with metrics and limitations

Start with [Part 1 — Foundations](part-01-foundations/00-overview.md) if you are new. If foundations are already solid, begin at the first part whose checklist you cannot yet pass.
