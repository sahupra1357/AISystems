# Lesson 12.1 — Project Catalog

Use this catalog to pick a capstone. Prefer projects where you can access data legally and finish an MVP.

Difficulty assumes you finished Stages 1–11.

## How to choose

Ask:

1. Which skills do I want on a resume *with proof*?
2. Can I get data this week?
3. Can I demo in 5 minutes?
4. Is the scope cuttable (MVP vs stretch)?

Write a one-paragraph proposal before coding.

## Project ideas

### 1) Docs Q&A RAG assistant

| | |
|--|--|
| **Difficulty** | Intermediate |
| **Skills** | Chunking, embeddings, retrieval, prompting, citations, golden-set eval, basic serving |
| **MVP** | Ingest a small docs folder; answer questions with citations; 20-question eval table |
| **Stretch** | Hybrid search, reranking, FastAPI UI, auth, monitoring hooks |
| **Build guide** | [02-build-guide-docs-qa-rag.md](02-build-guide-docs-qa-rag.md) |

### 2) Tabular ML + LLM executive report

| | |
|--|--|
| **Difficulty** | Intermediate |
| **Skills** | scikit-learn pipelines, metrics, baselines, structured LLM summarization, reliability of text over numbers |
| **MVP** | Train a model; export metrics/feature importances; LLM generates a report that only uses provided stats |
| **Stretch** | Scheduled batch job; HTML email; HITL approval |
| **Build guide** | [03-build-guide-tabular-plus-llm.md](03-build-guide-tabular-plus-llm.md) |

### 3) Image classifier with deployment sketch

| | |
|--|--|
| **Difficulty** | Intermediate |
| **Skills** | PyTorch CNN or fine-tune, evaluation curves, FastAPI/Docker sketch, error analysis |
| **MVP** | Train on a public image dataset; report test metrics; serve `/predict` locally |
| **Stretch** | ONNX export; canary notes; confusion-matrix-driven remediations |

### 4) Support-ticket triage system

| | |
|--|--|
| **Difficulty** | Intermediate–Advanced |
| **Skills** | Text classification (classical or LLM), class imbalance, cost-sensitive thresholds, human handoff rules |
| **MVP** | Classify tickets into ≥4 categories; baseline vs model; policy for low-confidence → human queue |
| **Stretch** | Active learning loop; RAG snippets per category; latency/cost dashboard |

### 5) Personal research digest (constrained tools)

| | |
|--|--|
| **Difficulty** | Intermediate |
| **Skills** | Tool calling, caching, structured output, safety constraints |
| **MVP** | Given URLs or local PDFs you provide, extract claims into JSON and write a digest with links |
| **Stretch** | Weekly cron; dedupe; reliability timeouts; no unbounded browsing agents |

## Scope cutters (use liberally)

- Fewer documents / fewer classes
- Offline CLI instead of polished UI
- Mock provider with recorded fixtures for CI
- Evaluate on 20 hard examples instead of 2,000

## Anti-catalog (avoid for first capstone)

- “Fully autonomous multi-agent company”
- Training a huge LLM from scratch
- Projects requiring private medical/financial data you should not have
- Anything whose success depends on a paid API you cannot afford to evaluate

## Proposal template

```markdown
# Capstone proposal

**Title:**
**Problem user:**
**MVP success metric:**
**Data source & rights:**
**Non-goals:**
**Timeline (weeks):**
**Risks:**
```

Fill this before opening the build guides’ implementation steps.
