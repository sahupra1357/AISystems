# The AI Engineering Guide

One path from "I can open a terminal" to "I run AI systems in production." The repository is laid out in the same 12 stages as this guide, so each section below matches a top-level folder:

```text
01-foundations/                      07-agents-tools-memory/
02-classical-ml/                     08-choosing-the-lever/
03-deep-learning/                    09-evaluation/
04-llm-foundations/                  10-production/
05-prompt-and-context-engineering/   11-architecture-patterns/
06-rag/                              12-capstones-and-interviews/
```

Every stage folder has the same three parts:

- **`README.md`** lists everything in the stage, in reading order.
- **`learn/`** holds the *curriculum*: lessons, exercises and checklists that explain **why** things work and build habits. Lessons are numbered by stage, so Lesson 6.1 is the first lesson in stage 6 and lives in `06-rag/learn/01-rag-deep-dive.md`. Each stage that has an overview starts with `00-overview.md`; the overview in stage 4 covers stages 4 to 9, and the one in stage 10 covers the production lessons.
- **`apply/`** holds the *field manual*: one file per production scenario, each with implementation steps, an example, trade-offs, scale notes, a checklist and metrics. It tells you **what to do** when you are building. File numbers such as `04.1.5` match the section numbers in [`AI_System.md`](AI_System.md), the one-page version of the whole field manual, which stays at the root.

Stages 1 to 3 are curriculum only.

This guide stitches the stages together. Each section below gives you the core ideas in a few paragraphs, the rules worth memorising, and links into both `learn/` and `apply/` so you can go deep exactly where you need to.

---

## Contents

1. [How to use this guide](#1-how-to-use-this-guide)
2. [The three mental models](#2-the-three-mental-models)
3. [Stage 1: Foundations](#3-stage-1-foundations)
4. [Stage 2: Classical machine learning](#4-stage-2-classical-machine-learning)
5. [Stage 3: Deep learning](#5-stage-3-deep-learning)
6. [Stage 4: Foundation models and LLMs](#6-stage-4-foundation-models-and-llms)
7. [Stage 5: Prompt and context engineering](#7-stage-5-prompt-and-context-engineering)
8. [Stage 6: Retrieval-augmented generation](#8-stage-6-retrieval-augmented-generation)
9. [Stage 7: Tools, agents and memory](#9-stage-7-tools-agents-and-memory)
10. [Stage 8: Choosing the lever (prompt, RAG, fine-tune)](#10-stage-8-choosing-the-lever)
11. [Stage 9: Evaluation](#11-stage-9-evaluation)
12. [Stage 10: Production engineering](#12-stage-10-production-engineering)
13. [Stage 11: Architecture patterns](#13-stage-11-architecture-patterns)
14. [Stage 12: Capstones, portfolio and interviews](#14-stage-12-capstones-portfolio-and-interviews)
15. [Cheat sheets](#15-cheat-sheets)
16. [Study plans](#16-study-plans)
17. [Master index: topic to file](#17-master-index-topic-to-file)

---

## 1. How to use this guide

**If you are new to AI:** follow stages 1 to 12 in order. For each stage, read the curriculum lessons first (the "Learn" links), do the exercises, pass the checklist, then skim the field-manual files (the "Apply" links) so you know they exist.

**If you already ship software and want to build LLM features:** skim stages 1 to 3 against their checklists, then start properly at stage 4. Spend most of your time on stages 6, 9 and 10.

**If you are building something right now:** jump to the stage that matches the problem, read the "Rules" block, then open the field-manual file for the exact scenario. Use [section 17](#17-master-index-topic-to-file) to find it.

**If you are preparing for interviews or design reviews:** read [section 2](#2-the-three-mental-models), [section 12](#12-stage-10-production-engineering) and [section 15](#15-cheat-sheets), then drill the 60 scenarios in [Lesson 12.5](12-capstones-and-interviews/learn/05-scenario-based-prod-ai-questions.md).

Every field-manual file follows the same shape: what it is, how to implement, small example, pros and cons, when to use, production at scale, checklist and metrics, and a 30-second takeaway. Every curriculum part follows: overview, lessons, exercises and a "ready for the next part" checklist.

---

## 2. The three mental models

Almost every decision in AI engineering is easier once these three pictures are in your head.

### 2.1 An AI feature is a system, not a model call

```text
                    ┌──────────────┐
 User request ──►   │ Orchestrator │ ──► response (+ citations / actions)
                    └──────┬───────┘
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
      Retriever         LLM / model      Tools
      (RAG index)    (prompt + decode)  (APIs, DBs)
           └───────────────┴───────────────┘
                           ▼
              Logging · eval hooks · cost meters
```

The model is one box. Quality comes from what you put around it: the data it reads, the tools it can call, the checks on its output, and the logs that let you debug it. Source: [Stages 4–9 overview](04-llm-foundations/learn/00-overview.md).

### 2.2 Production AI is a loop

```text
Data & labels → Train / adapt → Serve → Observe → Improve (eval gates + HITL) → back to data
```

A model that works in a notebook is not a product. You need versioned data, rollbackable releases, monitors for silent failure, explainable cost, safety controls, and explicit decisions about when humans review. Sources: [Stage 10 overview](10-production/learn/00-overview.md), [data flywheel](10-production/apply/8-data-flywheel/12.1_data_flywheel.md).

### 2.3 The end-to-end reference architecture

```text
OFFLINE   sources → ingest (dedupe, hash, ACL) → parse (route per page, validate)
          → chunk (parent/child, contextual) → embed (versioned) → index (vector + BM25 + graph)

ONLINE    user → authn/z → input guardrails → query understanding (rewrite, multi-query,
          decompose, filters) → route → hybrid retrieve (ACL-filtered) → rerank → top-k
          → context assembly (budget, order, dedupe, cite) → LLM / agent loop (tools, HITL)
          → grounding check → output guardrails → stream to user
          → traces, cost, feedback → eval set → flywheel
```

Sections 6 to 12 of this guide walk this diagram left to right. Sources: [AI_System.md §13](AI_System.md#13-reference-architecture-end-to-end), [reference architecture](11-architecture-patterns/apply/13.1_reference_architecture.md).

---

## 3. Stage 1: Foundations

**Goal:** write and run Python confidently, have working intuition for the math that training uses, and handle data honestly.

**Core ideas**

- **Python is the daily tool.** Virtual environments, `pip`, reading stack traces, files and JSON, Git basics, and a first look at NumPy and pandas.
- **Math you actually use.** Vectors and dot products (embeddings are vectors; similarity is a dot product), matrices (a layer is a matrix multiply), derivatives and gradients (training is walking downhill on a loss), probability (outputs are distributions), and statistics (mean, variance, bias versus variance).
- **Data literacy decides projects.** Structured versus unstructured data, cleaning (missing values, duplicates, outliers), train/validation/test splits, and choosing metrics that match the job.

**Rules**

- Split before you do anything else. Normalising or imputing on the full dataset before splitting is leakage.
- Pick the metric before the model: accuracy only for balanced classes; precision, recall and F1 when classes are skewed or errors cost differently; RMSE or MAE for regression.
- Honest evaluation beats a high number. The test set is touched once.

**Learn:** [overview](01-foundations/learn/00-overview.md) · [Python](01-foundations/learn/01-python-programming.md) · [math for AI](01-foundations/learn/02-math-for-ai.md) · [data literacy](01-foundations/learn/03-data-literacy.md) · [exercises and checklist](01-foundations/learn/04-exercises-and-checklist.md)

**You are ready to move on when** you can load a CSV, clean it, compute stats, write a JSON summary, explain a train/val/test split in one sentence, and spot a leakage bug.

---

## 4. Stage 2: Classical machine learning

**Goal:** train, compare and evaluate models on tabular data the professional way.

**Core ideas**

- **Supervised versus unsupervised.** Labels or no labels. Features are inputs; the label is what you predict.
- **Overfitting, underfitting and bias–variance.** Fight overfitting with more data, regularisation, simpler models, and cross-validation.
- **The algorithms you will actually use:** linear and logistic regression (fast, interpretable baselines), decision trees, random forests, gradient boosting (the default winner on tabular data), k-NN, and k-means for clustering.
- **The scikit-learn workflow:** `Pipeline` (preprocessing fitted only on training data), cross-validation, hyperparameter search, metrics that match the job, and saving models.

**Rules**

- Always beat a dumb baseline (majority class, mean prediction) before celebrating.
- Put preprocessing inside the `Pipeline`, so cross-validation cannot leak.
- On tabular problems, try gradient boosting before deep learning, and often before an LLM.

**Why this matters later:** classical ML still powers fraud scoring, churn, ranking and routing, and many production LLM systems use a small classifier as a cheap router or guardrail. See [cascade classifiers](11-architecture-patterns/learn/01-ai-architecture-patterns-for-prod.md#13-ensemble--cascade-classifiers) and the [tabular + LLM capstone](12-capstones-and-interviews/learn/03-build-guide-tabular-plus-llm.md).

**Learn:** [overview](02-classical-ml/learn/00-overview.md) · [ML concepts](02-classical-ml/learn/01-ml-concepts.md) · [supervised algorithms](02-classical-ml/learn/02-supervised-algorithms.md) · [scikit-learn workflow](02-classical-ml/learn/03-scikit-learn-workflow.md) · [exercises and checklist](02-classical-ml/learn/04-exercises-and-checklist.md)

---

## 5. Stage 3: Deep learning

**Goal:** understand what training is doing, and write a PyTorch training loop you can debug.

**Core ideas**

- **Neurons, layers, activations, loss, backpropagation, optimisers.** A network is stacked matrix multiplies with non-linearities; backprop computes gradients; SGD or Adam updates the weights; minibatches and epochs organise the work.
- **The training loop shape** you will reuse forever: `model.train()`, forward, loss, `zero_grad`, backward, step; then `model.eval()` and `torch.no_grad()` for validation.
- **Devices.** Move both model and tensors to the GPU; a GPU helps when the work is large, parallel matrix maths.
- **Architectures.** CNNs exploit spatial locality in images. Transformers use attention to relate every token to every other token, which is why they dominate language and why LLMs exist.

**Rules**

- Plot train and validation loss together. Diverging curves mean overfitting; both flat and high means underfitting or a bug.
- Overfit a single small batch first. If you cannot, the bug is in the loop, not the data.
- Fine-tuning a pretrained model usually beats training from scratch.

**Learn:** [overview](03-deep-learning/learn/00-overview.md) · [neural net basics](03-deep-learning/learn/01-neural-net-basics.md) · [PyTorch training loop](03-deep-learning/learn/02-pytorch-training-loop.md) · [CNNs and Transformers](03-deep-learning/learn/03-cnns-and-transformers.md) · [exercises and checklist](03-deep-learning/learn/04-exercises-and-checklist.md)

---

## 6. Stage 4: Foundation models and LLMs

**Goal:** reason about what an LLM is doing, what it costs, and how it fails, before building on top of it.

**Core ideas**

- **Next-token prediction becomes chat behaviour** through instruction tuning and preference tuning. The model predicts tokens; everything else is scaffolding.
- **Tokens and context windows.** You pay per token, latency grows with output tokens, and the window caps what the model can see at once.
- **Embedding models versus generative models.** Embedders turn text into vectors for search; generators produce text. A RAG system uses both.
- **Decoding knobs.** Temperature and top-p trade determinism for diversity. Use temperature 0 for extraction, classification and evals.
- **Failure modes to expect:** hallucination, format drift, instruction-following lapses, sensitivity to prompt wording, stale knowledge, and confident wrong answers.

**Choosing models** ([model selection](04-llm-foundations/apply/01.1_model_selection.md), [routing and escalation](04-llm-foundations/apply/01.2_routing_escalation.md))

- Pick by capability, cost, latency and context window. Small models with good prompts and good retrieval routinely beat large models with bad context.
- A cheap fast model handles classification, routing, extraction, query rewriting and light reranking. A reasoning model handles final answers, planning and critique.
- Route cheap-first and escalate on low confidence. If more than about 40% of traffic escalates, the router is not earning its complexity.
- **Build versus buy:** managed retrieval gets a demo in a day; you own the pipeline once you need custom chunking, reranking or ACLs ([build vs buy](04-llm-foundations/apply/01.4_build_vs_buy.md)).

**Learn:** [foundation models basics](04-llm-foundations/learn/01-foundation-models-basics.md) (includes a worked cost estimate for a naive support bot and a "the model is being weird" debugging playbook)

---

## 7. Stage 5: Prompt and context engineering

**Goal:** get reliable behaviour from a model through what you put in its window.

### 7.1 Prompt engineering

- **Structure:** system/role → task → constraints → examples → output format. Put **stable content first** so the prompt-cache prefix hits ([prompt structure](05-prompt-and-context-engineering/apply/1-prompt-engineering/02.1_prompt_structure.md)).
- **Delimit data from instructions.** Retrieved text, user uploads and tool output go inside clearly marked blocks the instructions say to treat as data ([delimiters](05-prompt-and-context-engineering/apply/1-prompt-engineering/02.2_delimiters_data_vs_instructions.md)).
- **Techniques:** zero/few-shot ([02.3](05-prompt-and-context-engineering/apply/1-prompt-engineering/02.3_zero_few_shot.md)), chain-of-thought when reasoning pays and not for extraction ([02.4](05-prompt-and-context-engineering/apply/1-prompt-engineering/02.4_chain_of_thought.md)), self-consistency ([02.5](05-prompt-and-context-engineering/apply/1-prompt-engineering/02.5_self_consistency.md)), decomposition, negative examples and prefill ([02.6](05-prompt-and-context-engineering/apply/1-prompt-engineering/02.6_decomposition_negative_examples_prefill.md)).
- **Dynamic few-shot:** retrieve the k most similar solved cases. It is a mini-RAG and beats static examples on domain tasks ([02.7](05-prompt-and-context-engineering/apply/1-prompt-engineering/02.7_dynamic_few_shot_selection.md)).
- **Structured outputs:** JSON schema, function calling, and validate-then-retry with the error fed back. Track schema violations as a metric ([02.8](05-prompt-and-context-engineering/apply/1-prompt-engineering/02.8_structured_outputs_tool_use.md)).
- **Prompts are code:** version them, diff them, run the eval suite on every change, and log the prompt version with every call ([02.9](05-prompt-and-context-engineering/apply/1-prompt-engineering/02.9_prompt_versioning_evaluation.md)).

### 7.2 Context engineering

Context engineering manages *everything* in the window: instructions, retrieved documents, tool results, memory and history. The goal is the right information, in the right position, with minimal noise ([what it is](05-prompt-and-context-engineering/apply/2-context-engineering/03.1_what_is_context_engineering.md)).

- Keep a rolling summary plus the last N turns verbatim ([compaction](05-prompt-and-context-engineering/apply/2-context-engineering/03.2_compaction_summarization.md)).
- Put the most relevant chunk first or last, never buried in the middle ([lost in the middle](05-prompt-and-context-engineering/apply/2-context-engineering/03.3_lost_in_the_middle.md)).
- Cap tokens per source so one verbose tool result cannot crowd out the rest ([budgets](05-prompt-and-context-engineering/apply/2-context-engineering/03.4_per_source_truncation_budgets.md)).
- Cache static prefixes; often the single biggest cost lever in a chat product ([prompt caching](05-prompt-and-context-engineering/apply/2-context-engineering/03.5_prompt_caching.md)).
- Give subagents their own window instead of appending to the orchestrator's ([isolation](05-prompt-and-context-engineering/apply/2-context-engineering/03.6_context_isolation.md)).
- Name the failure modes: poisoning, distraction, confusion and clash ([failure modes](05-prompt-and-context-engineering/apply/2-context-engineering/03.7_context_failure_modes.md)).

**Learn:** [prompting deep dive](05-prompt-and-context-engineering/learn/01-prompting-deep-dive.md) (anatomy, roles, styles, task playbooks, parameters, iteration method, anti-patterns, and a worked case from playground to template).

---

## 8. Stage 6: Retrieval-augmented generation

**Goal:** answer from private, fresh or large knowledge with citations a user can trust.

RAG is a pipeline, and quality is capped by its weakest stage. Work through it in build order.

### 8.1 Ingestion and parsing (the highest-leverage stage)

Garbage parsing caps everything downstream. No reranker recovers a table flattened into word soup.

- **Ingest:** connectors with auth, pagination and rate limits ([00.1.1](06-rag/apply/1-ingestion-parsing/00.1.1_connectors.md)); change detection with a size/mtime then SHA-256 ladder so you only re-embed what changed ([00.1.2](06-rag/apply/1-ingestion-parsing/00.1.2_change_detection.md)); exact and near-duplicate removal ([00.1.3](06-rag/apply/1-ingestion-parsing/00.1.3_deduplication.md)); `doc_id + version` with soft-deleted stale chunks ([00.1.4](06-rag/apply/1-ingestion-parsing/00.1.4_document_versioning.md)); tenant and ACL captured at ingest, never retrofitted ([00.1.5](06-rag/apply/1-ingestion-parsing/00.1.5_tenant_acl_capture_at_ingest.md)).
- **Route per page, not per document** ([00.2.0](06-rag/apply/1-ingestion-parsing/00.2.0_per_page_routing.md)): digital pages to a native text-layer parser ([00.2.1](06-rag/apply/1-ingestion-parsing/00.2.1_digital_document_docling.md)), scans to OCR ([00.2.2](06-rag/apply/1-ingestion-parsing/00.2.2_scanned_page_ocr.md)), complex layouts to a layout model ([00.2.3](06-rag/apply/1-ingestion-parsing/00.2.3_complex_layout_ocr.md)), formulas to LaTeX extraction ([00.2.4](06-rag/apply/1-ingestion-parsing/00.2.4_formula_heavy_mineru.md)), charts to VLM captions ([00.2.5](06-rag/apply/1-ingestion-parsing/00.2.5_charts_images_vlm_captioning.md)).
- **Preserve structure:** reading order ([00.3.1](06-rag/apply/1-ingestion-parsing/00.3.1_reading_order.md)), tables as HTML/Markdown ([00.3.2](06-rag/apply/1-ingestion-parsing/00.3.2_tables.md)), heading breadcrumbs ([00.3.3](06-rag/apply/1-ingestion-parsing/00.3.3_headings_hierarchy.md)), page numbers and bounding boxes for citations ([00.3.4](06-rag/apply/1-ingestion-parsing/00.3.4_page_numbers_bounding_boxes.md)), boilerplate stripped ([00.3.5](06-rag/apply/1-ingestion-parsing/00.3.5_boilerplate_stripping.md)).
- **Validate:** CER/WER and table-cell accuracy on a golden sample ([00.4.1](06-rag/apply/1-ingestion-parsing/00.4.1_validation_golden_sample.md)); cheap per-page heuristics in production ([00.4.2](06-rag/apply/1-ingestion-parsing/00.4.2_validation_production_heuristics.md)); a bounded retry ladder that ends in a visible human queue, never silent empty text ([00.5.1](06-rag/apply/1-ingestion-parsing/00.5.1_retry_ladder.md)).

### 8.2 Chunking

| Strategy | Use it when | File |
|---|---|---|
| Fixed-size + overlap | Baseline only | [04.1.1](06-rag/apply/2-chunking/04.1.1_fixed_size_overlap.md) |
| Recursive character splitting | Good general default | [04.1.2](06-rag/apply/2-chunking/04.1.2_recursive_character_splitting.md) |
| Layout/structure-aware | The parser preserved headings and tables | [04.1.3](06-rag/apply/2-chunking/04.1.3_layout_structure_aware.md) |
| Semantic | Cohesion matters more than ingest cost | [04.1.4](06-rag/apply/2-chunking/04.1.4_semantic_chunking.md) |
| **Parent–child (small-to-big)** | **Highest-ROI upgrade in most systems** | [04.1.5](06-rag/apply/2-chunking/04.1.5_parent_child_small_to_big.md) |
| Contextual retrieval | Chunks full of pronouns and bare numbers | [04.1.6](06-rag/apply/2-chunking/04.1.6_contextual_retrieval.md) |
| Late chunking | You have a long-context embedder | [04.1.7](06-rag/apply/2-chunking/04.1.7_late_chunking.md) |

Every chunk carries metadata: `doc_id`, title, breadcrumb, page, section, dates, tenant, ACL, source URL ([04.1.8](06-rag/apply/2-chunking/04.1.8_chunk_metadata.md)).

### 8.3 Embeddings and indexing

- Choose the embedder for your domain and language, and validate on *your* golden set, not only a leaderboard ([04.2.1](06-rag/apply/3-embeddings/04.2.1_embedding_model_choice.md)). Get query/passage prefixes right ([04.2.2](06-rag/apply/3-embeddings/04.2.2_symmetric_vs_asymmetric.md)). Fine-tuning the embedder on your own triplets is often a bigger win than a bigger LLM ([04.2.3](06-rag/apply/3-embeddings/04.2.3_fine_tuning_embeddings.md)). A model change means a full re-index; store the version per chunk and dual-index during migration ([04.2.4](06-rag/apply/3-embeddings/04.2.4_embedding_versioning.md)).
- HNSW is the default index; flat for small corpora; IVF-PQ when memory-bound ([04.3.1](06-rag/apply/4-indexing/04.3.1_index_types_flat_hnsw_ivfpq.md)). Choose the store by what else you need: pgvector for joins and transactional ACLs, Qdrant/Weaviate/Milvus for scale and filtering, a managed service for no ops, OpenSearch when you also need BM25 and aggregations ([04.3.2](06-rag/apply/4-indexing/04.3.2_vector_store_choice.md)). Prefer native filtered ANN; post-filtering by tenant can return zero rows ([04.3.3](06-rag/apply/4-indexing/04.3.3_filtered_search.md)).

### 8.4 Query understanding

Rewrite follow-ups using chat history, or multi-turn RAG falls apart by turn three ([04.4.1](06-rag/apply/5-query-understanding/04.4.1_query_rewriting.md)). Add multi-query with Reciprocal Rank Fusion for cheap recall ([04.4.2](06-rag/apply/5-query-understanding/04.4.2_multi_query_rrf.md)), HyDE for short queries against verbose documents ([04.4.3](06-rag/apply/5-query-understanding/04.4.3_hyde.md)), decomposition for multi-hop questions ([04.4.4](06-rag/apply/5-query-understanding/04.4.4_decomposition.md)), self-query to turn language into hard filters ([04.4.5](06-rag/apply/5-query-understanding/04.4.5_self_query_metadata_filters.md)), and routing so that not every question hits the index ([04.4.6](06-rag/apply/5-query-understanding/04.4.6_query_routing.md)).

### 8.5 Retrieval and ranking

- **Hybrid search** (BM25 + dense, fused) beats either alone on virtually every real corpus, because IDs, part numbers and names are where dense retrieval fails ([04.5.1](06-rag/apply/6-retrieval-ranking/04.5.1_hybrid_search.md)).
- **Rerank:** retrieve top-50 cheaply, cross-encoder rerank to top-5. Usually the largest single quality jump ([04.5.2](06-rag/apply/6-retrieval-ranking/04.5.2_reranking.md)).
- Diversify with MMR ([04.5.3](06-rag/apply/6-retrieval-ranking/04.5.3_mmr_diversity.md)), abstain below a tuned score floor ([04.5.4](06-rag/apply/6-retrieval-ranking/04.5.4_score_thresholding_abstention.md)), and bound any follow-up searches ([04.5.5](06-rag/apply/6-retrieval-ranking/04.5.5_recursive_agentic_retrieval.md)).

### 8.6 Advanced patterns

Reach for these only when the basics are measured and insufficient: agentic RAG ([04.6.1](06-rag/apply/7-advanced-rag/04.6.1_agentic_rag.md)), corrective RAG ([04.6.2](06-rag/apply/7-advanced-rag/04.6.2_corrective_rag.md)), Self-RAG ([04.6.3](06-rag/apply/7-advanced-rag/04.6.3_self_rag.md)), GraphRAG for global and multi-hop questions ([04.6.4](06-rag/apply/7-advanced-rag/04.6.4_graphrag.md)), multimodal RAG for slides and scans ([04.6.5](06-rag/apply/7-advanced-rag/04.6.5_multimodal_rag.md)), and SQL for aggregates alongside vectors for narrative, because many "RAG failures" are analytics questions ([04.6.6](06-rag/apply/7-advanced-rag/04.6.6_structured_unstructured_hybrid.md)).

### 8.7 Generation and grounding

Answer only from context, then verify ([04.7.1](06-rag/apply/8-generation-grounding/04.7.1_grounded_prompting.md)). Require citations in the output schema and validate every cited ID ([04.7.2](06-rag/apply/8-generation-grounding/04.7.2_citations.md)). Run a cheap faithfulness pass ([04.7.3](06-rag/apply/8-generation-grounding/04.7.3_faithfulness_check.md)). Surface conflicting sources with dates instead of picking one ([04.7.4](06-rag/apply/8-generation-grounding/04.7.4_conflict_handling.md)). Design and evaluate the refusal path ([04.7.5](06-rag/apply/8-generation-grounding/04.7.5_refusal_no_answer_path.md)).

**Rules**

- Permission-filter retrieval at query time with the caller's identity. Access control is non-negotiable and never lives in the prompt.
- Log retrieved chunk IDs and scores on every request. You cannot debug an answer without knowing what it read.
- Evaluate retrieval separately from generation. If the answer was never retrieved, generation metrics are noise.

**Learn:** [RAG deep dive](06-rag/learn/01-rag-deep-dive.md) (corpus design, chunking workshop, hybrid fusion sketch, citations, ACLs, debugging playbook, worked handbook Q&A). **Build:** [Docs Q&A RAG capstone](12-capstones-and-interviews/learn/02-build-guide-docs-qa-rag.md).

---

## 9. Stage 7: Tools, agents and memory

**Goal:** let models take actions safely, with bounded cost and a clear stop point.

### 9.1 Tools and structured output

Tools beat hallucinated actions: the model proposes a call, your code validates and executes it. A good tool has a precise name, a description that says when *not* to use it, a strict argument schema, and an explicit side-effect contract. Prefer fewer, well-described, idempotent tools over one "supertool" ([tool design](07-agents-tools-memory/apply/1-agents/05.7_tool_design.md)).

### 9.2 Agent shapes, simplest first

1. **Single tool call** inside a fixed workflow. Most production "agents" should be this.
2. **Plan-then-execute or a DAG** for predictable pipelines.
3. **Single agent loop** (LLM → tool → observe → repeat) with a termination condition, max iterations, per-step timeouts and errors fed back to the model ([05.1](07-agents-tools-memory/apply/1-agents/05.1_single_agent_loop.md)).
4. **Orchestrator and subagents** with explicit task contracts, narrow scope and isolated context ([05.2](07-agents-tools-memory/apply/1-agents/05.2_task_decomposition.md), [05.3](07-agents-tools-memory/apply/1-agents/05.3_orchestrator_subagent.md)), fan-out/fan-in with partial-failure handling ([05.4](07-agents-tools-memory/apply/1-agents/05.4_fan_out_fan_in.md)), and bounded reflection ([05.5](07-agents-tools-memory/apply/1-agents/05.5_loop_back_reflection.md)).

Choose ReAct for exploration, plan-then-execute for predictable work, and search only when steps are cheap and reversible ([planning styles](07-agents-tools-memory/apply/1-agents/05.6_planning_styles.md)).

### 9.3 Making agents safe to run

- **Human-in-the-loop** approval before irreversible actions (send, pay, delete, deploy), with a clear resume path ([05.8](07-agents-tools-memory/apply/1-agents/05.8_human_in_the_loop.md)).
- **Durability:** checkpoint after each step and use idempotency keys so a resumed run does not double-send ([05.9](07-agents-tools-memory/apply/1-agents/05.9_state_durability.md)).
- **Sandboxing:** code runs in a container with no secrets, egress policy, resource limits and read-only mounts ([05.10](07-agents-tools-memory/apply/1-agents/05.10_sandboxing.md)).
- **Interoperability:** MCP for tool discovery and reuse, A2A-style handoffs, or a shared blackboard ([05.11](07-agents-tools-memory/apply/1-agents/05.11_multi_agent_communication_mcp.md)).
- **Budgets:** hard caps on tokens, tool calls, wall clock and dollars, enforced by the runtime ([05.12](07-agents-tools-memory/apply/1-agents/05.12_cost_step_budgets.md)).
- **Least privilege:** a read-only agent with a read-only token cannot be talked into a delete ([excessive agency](10-production/apply/5-guardrails/08.5_excessive_agency.md)).

### 9.4 Memory

Short-term (the window), working (a scratchpad), episodic (past interactions), semantic (durable facts) and procedural (how-tos) ([06.1](07-agents-tools-memory/apply/2-memory/06.1_memory_types.md)). Write only what an extraction step judges memory-worthy ([06.2](07-agents-tools-memory/apply/2-memory/06.2_what_to_write.md)); retrieve by relevance and recency with a cap ([06.3](07-agents-tools-memory/apply/2-memory/06.3_when_to_retrieve.md)); forget with TTLs and user controls ([06.4](07-agents-tools-memory/apply/2-memory/06.4_how_to_forget.md)); supersede changed facts with `valid_from` ([06.5](07-agents-tools-memory/apply/2-memory/06.5_conflict_resolution.md)); and treat memory as data, because it is an injection surface ([06.6](07-agents-tools-memory/apply/2-memory/06.6_memory_security.md)).

**Learn:** [tools, agents and structured output](07-agents-tools-memory/learn/01-tools-agents-and-structured-output.md) (argument validation catalogue, tool result hygiene, deterministic test stubs, cost model, worked refund assistant, and a pre-production orchestrator checklist).

---

## 10. Stage 8: Choosing the lever

**Goal:** fix the right problem with the right tool.

| Lever | Changes | Cost to iterate | Use when |
|---|---|---|---|
| **Prompting** | Behaviour: tone, format, procedure | Minutes, no training | Always first |
| **RAG** | Knowledge: fresh, private or too large for the weights | Days; ongoing index ops | "It doesn't know" |
| **Fine-tuning** | Style, format or a narrow domain, at scale; or cost via distillation | Weeks; retraining and eval burden | "It knows but says it wrong," and prompting does not scale |
| **Change the product** | The task itself | Varies | The model is being asked to do something it should not |

**Rule of thumb:** if the failure is *"it doesn't know,"* use RAG. If it is *"it knows but says it wrong,"* fix the prompt, and fine-tune only if prompting cannot scale. Production systems usually combine all three.

**Fine-tuning, when you get there:** SFT on a few hundred to a few thousand high-quality pairs ([11.1](08-choosing-the-lever/apply/fine-tuning/11.1_sft.md)); LoRA/QLoRA adapters as the default ([11.2](08-choosing-the-lever/apply/fine-tuning/11.2_peft_lora_qlora.md)); DPO for subjective quality ([11.3](08-choosing-the-lever/apply/fine-tuning/11.3_preference_tuning_dpo_rlhf.md)); distillation to cut cost once behaviour is stable ([11.4](08-choosing-the-lever/apply/fine-tuning/11.4_distillation.md)). Do not fine-tune for changing knowledge, unstable tasks, or anything a better prompt fixes ([11.5](08-choosing-the-lever/apply/fine-tuning/11.5_when_not_to_fine_tune.md)). Version datasets with checkpoints and keep a held-out split from day one ([11.6](08-choosing-the-lever/apply/fine-tuning/11.6_fine_tuning_ops.md)).

**Learn:** [fine-tuning vs RAG vs prompt](08-choosing-the-lever/learn/01-fine-tuning-vs-rag-vs-prompt.md) (decision flowchart, scenario workshop, LoRA intuition, data pipeline, eval gates, a copy-paste decision worksheet). **Apply:** [prompting vs RAG vs fine-tuning](08-choosing-the-lever/apply/01.3_prompting_vs_rag_vs_finetuning.md).

---

## 11. Stage 9: Evaluation

**Goal:** know whether a change made things better, before users tell you.

Evaluation is the skill that most separates AI engineers from people who call APIs. Treat it as a first-class release control.

- **Golden dataset first:** 50 to 200 real, hand-labelled cases beat any generic benchmark. Grow it from production failures; every incident becomes a test case ([07.1](09-evaluation/apply/07.1_golden_dataset.md)).
- **Prompt evaluation:** run variants against the golden set and track regressions per prompt version ([07.2](09-evaluation/apply/07.2_prompt_evaluation.md)).
- **LLM-as-judge:** use it for scale, calibrate it against human labels, report agreement, and watch for position, length and self-preference bias. Never the sole truth ([07.3](09-evaluation/apply/07.3_llm_as_judge.md)).
- **Separate the failure sources in RAG:** retrieval (recall@k, MRR, nDCG) versus generation (faithfulness, relevance, citation accuracy) ([07.4](09-evaluation/apply/07.4_retrieval_evals.md), [07.5](09-evaluation/apply/07.5_generation_evals.md)).
- **Agents:** evaluate the trajectory and the cost, not only the final answer. A 2% gain for 4× the tool calls is usually a loss ([07.6](09-evaluation/apply/07.6_agent_evals.md)).
- **Component then end-to-end:** end-to-end-only evals tell you something broke, not what ([07.7](09-evaluation/apply/07.7_component_vs_end_to_end.md)).
- **Statistical hygiene:** temperature 0, multiple seeds where sampling matters, confidence intervals, and scepticism about a 3-point move on 50 cases ([07.8](09-evaluation/apply/07.8_statistical_hygiene.md)).
- **Online signals:** thumbs, edit distance to what the user shipped, task completion and escalation rate, with production traffic sampled back into the eval set ([07.9](09-evaluation/apply/07.9_online_eval.md)).
- **Score abstention correctly.** Most RAG systems are never measured on whether they correctly decline.
- **Report by slice** (customer tier, language, document type), because averages hide the segment that is failing.

**Testing pyramid:** unit tests for tools and parsers → eval suites for prompts → integration tests for agent flows with mocked tools → red-team tests → evals in CI that block merges on regression ([08.1](09-evaluation/apply/08.1_testing_pyramid.md)).

**Learn:** [evaluation for LLM systems](09-evaluation/learn/01-evaluation-for-llm-systems.md) (metric formulas, rubrics, pairwise comparison, regression gates, human review session recipe, eval harness sketch, release checklist). **Practise:** [Stages 4–9 capstone: prompt library + eval set](09-evaluation/learn/02-exercises-and-checklist.md).

---

## 12. Stage 10: Production engineering

**Goal:** ship AI features that are reliable, observable, affordable and safe, and that improve over time.

### 12.1 Decide deliberately

Use this framework for every significant decision, and leave a one-page memo behind ([Lesson 10.1](10-production/learn/01-production-mindset-and-decision-framework.md)):

```text
1. Problem       Who is harmed or helped? What job is the model doing?
2. Constraints   Latency, cost, privacy, compliance, team skill, deadline
3. Options       At least 2–3 real alternatives
4. Trade-offs    Cost · latency · quality · complexity · risk · skill fit
5. Eval plan     Golden set, metrics, pass bars
6. HITL plan     Who reviews what, sample rate, escalation
7. Ship criteria What must be true to promote
8. Rollback      Exact previous artefact versions + kill switch
```

**Match verification to risk:**

| Tier | Examples | Release path |
|---|---|---|
| Low | Internal drafts, editable suggestions | Offline smoke → ship |
| Medium | Customer-facing answers, product ranking | Offline gates → shadow/canary → sampled HITL → ship |
| High | Money, access, regulated advice, irreversible actions | Offline gates → red-team → shadow → HITL on actions → slow canary → ship |

If a wrong answer can move money, change access or give regulated advice, treat it as high until someone accountable says otherwise.

### 12.2 Data pipelines and MLOps

Version datasets and RAG corpora, track experiments, and promote models, prompts and indexes through a registry with eval gates. Separate dev, staging and prod indexes; never evaluate against the prod index you are mutating ([Lesson 10.2](10-production/learn/02-data-pipelines-and-mlops.md), [environments](10-production/apply/1-serving-cost-latency/09.6_environments.md)).

### 12.3 Serving

- **Shapes:** batch, sync API, async queue, streaming. Add a worker queue when work exceeds a request timeout or needs retries and review ([Lesson 10.3](10-production/learn/03-serving-architectures.md)).
- **Cost levers:** prompt caching, batching, routing, per-request token budgets, shorter system prompts, trimming top-k after reranking. Know your $/task and $/user/month ([09.1](10-production/apply/1-serving-cost-latency/09.1_cost_levers.md)).
- **Latency levers:** streaming (perceived latency is what users judge), semantic caching, parallel retrieval and tool calls, smaller models on the critical path ([09.2](10-production/apply/1-serving-cost-latency/09.2_latency_levers.md)).
- **Self-hosting:** vLLM/TGI with continuous batching, paged KV cache, quantisation and tensor parallelism ([09.3](10-production/apply/1-serving-cost-latency/09.3_self_hosted_serving.md)). Rate limits, per-tenant throttling, backpressure and load shedding ([09.4](10-production/apply/1-serving-cost-latency/09.4_capacity_limits.md)).
- **Rollouts:** prompts, models, indexes and chunking configs are versioned artefacts. Shadow, canary or A/B; pin model versions; keep rollback one command away ([09.5](10-production/apply/1-serving-cost-latency/09.5_versioning_rollout.md)).
- **Cloud deployment:** host the app on Fargate, run small stateless tasks on Lambda, and call the model through Bedrock ([16.1](10-production/apply/3-deployment/16.1_aws_bedrock_fargate_lambda.md)).

### 12.4 GPU capacity

On GPUs the card is the unit of capacity and most clouds cannot hot-swap GPU type or count, so a resize is a controlled fleet replacement. **Exhaust software packing first, then horizontal replicas, then a bigger SKU, then tensor parallelism.** Size VRAM as weights + KV cache (context × concurrent sequences) + activations + 10–20% headroom; at production concurrency the KV cache usually dominates. Never autoscale on GPU utilisation; scale on queue depth and time-to-first-token, and keep generation pools warm. Read the playbook in [GPUResizing.md](10-production/apply/2-gpu/GPUResizing.md) and the detail in [`2-gpu/details/`](10-production/apply/2-gpu/details/).

### 12.5 Monitoring, drift and cost

Log enough to replay any request: prompt version, model, tokens, latency, cost, tool calls, retrieved chunk IDs and scores, outcome, and a trace ID ([observability](10-production/apply/4-observability-reliability/08.6_observability_tracing.md)). Monitor quality, drift, latency and cost against SLOs; route thumbs-down, empty retrieval and low-confidence traffic to human review queues; and run a weekly operating rhythm ([Lesson 10.4](10-production/learn/04-monitoring-drift-and-cost.md)).

### 12.6 Safety, privacy and security

Start with a threat model, then layer defences ([Lesson 10.5](10-production/learn/05-safety-privacy-and-hitl.md)):

- **Input guardrails:** PII detection and masking, topic and abuse classification ([08.2](10-production/apply/5-guardrails/08.2_input_guardrails.md)). **Output guardrails:** leakage, toxicity, schema, banned claims ([08.3](10-production/apply/5-guardrails/08.3_output_guardrails.md)).
- **Prompt injection:** all retrieved content, tool output and memory is data, never instructions; privileged actions sit behind a policy check the model does not control ([08.4](10-production/apply/5-guardrails/08.4_prompt_injection_defense.md)).
- **Access and tenancy:** filter at query time with the caller's identity; isolation belongs in the index ([10.1](10-production/apply/6-security-privacy-governance/10.1_authn_authz_permission_filtered_retrieval.md)).
- **Privacy and governance:** data residency and retention ([10.2](10-production/apply/6-security-privacy-governance/10.2_data_residency_retention.md)), PII with reversible mapping ([10.3](10-production/apply/6-security-privacy-governance/10.3_pii.md)), queryable audit trails ([10.4](10-production/apply/6-security-privacy-governance/10.4_auditability.md)), pinned and vetted dependencies and MCP servers ([10.5](10-production/apply/6-security-privacy-governance/10.5_supply_chain_hygiene.md)), and the OWASP LLM Top 10 as a checklist ([10.6](10-production/apply/6-security-privacy-governance/10.6_owasp_llm_top10.md)).
- **HITL rate:** set it from error cost, volume and reviewer capacity (see [section 15.4](#154-starting-hitl-rates)), keep kill switches, and use an escalation ladder: auto-allow → soft flag → hard flag (HITL required) → deny and log.

### 12.7 Reliability and incidents

Timeouts on every external call, retries with exponential backoff and jitter, idempotency, circuit breakers, fallback models across providers, and graceful degradation (return the sources with "couldn't summarise" rather than a 500) ([08.7](10-production/apply/4-observability-reliability/08.7_failure_handling.md)). Write runbooks for AI-specific failures, version everything so rollback is real, and run postmortems that ask which eval would have caught the incident ([Lesson 10.6](10-production/learn/06-reliability-and-incident-response.md)).

### 12.8 The data flywheel

```text
prod traffic → traces + feedback → weekly triage → new golden cases
  → prompt / retrieval / model fix → eval → canary → prod
```

Bucket failures into parse, retrieval, ranking, generation and tooling; the buckets tell you where to spend. Mine hard negatives to fine-tune the embedder or reranker. Every fixed bug becomes a permanent regression test ([12.1](10-production/apply/8-data-flywheel/12.1_data_flywheel.md), [12.2](10-production/apply/8-data-flywheel/12.2_feedback_capture_triage.md)).

**Practise:** [Stage 10 exercises and production design-doc capstone](10-production/learn/07-exercises-and-checklist.md).

---

## 13. Stage 11: Architecture patterns

Patterns compose. A multi-tenant RAG SaaS product is typically patterns 15 + 6 + 12 + 9 + 8 below. Full treatment with options, trade-offs and eval requirements in [Lesson 11.1](11-architecture-patterns/learn/01-ai-architecture-patterns-for-prod.md).

| # | Pattern | Latency | Typical risk |
|---|---|---|---|
| 1 | Batch scoring pipeline | Hours to a day | Low–medium |
| 2 | Real-time sync inference API | ms to a few s | Medium |
| 3 | Async job / queue inference | Seconds to minutes | Medium–high |
| 4 | Feature store + online serving | ms to s | Medium |
| 5 | Model gateway / multi-model router | ms to s | Medium |
| 6 | RAG production architecture | 1–10 s | Medium–high |
| 7 | RAG + tools / constrained agent | 2–30 s+ | High |
| 8 | HITL review workflow | Human-paced | Medium–high |
| 9 | Shadow / canary / A/B topology | Same as target | Any |
| 10 | Event-driven AI (stream → score → act) | Sub-second to seconds | Medium–high |
| 11 | Edge / on-device + cloud hybrid | Local ms + cloud | Privacy, offline |
| 12 | LLM gateway (policy, cache, cost) | Adds overhead | Cross-cutting |
| 13 | Ensemble / cascade classifiers | ms to s | Medium |
| 14 | Offline train → registry → online serving | Release cycle | All MLOps |
| 15 | Multi-tenant SaaS AI | Same as product | High |

For the RAG and agent patterns, the field-manual counterpart is the [reference architecture](11-architecture-patterns/apply/13.1_reference_architecture.md).

---

## 14. Stage 12: Capstones, portfolio and interviews

**Ship one excellent project rather than five half-finished clones.** A capstone is done when someone else can run it from the README, you report metrics on a held-out or golden set, you document failures honestly, and you can explain the architecture in five minutes without slides.

- Pick a project: [project catalogue](12-capstones-and-interviews/learn/01-project-catalog.md).
- Build a Docs Q&A RAG system end to end: [build guide](12-capstones-and-interviews/learn/02-build-guide-docs-qa-rag.md). Upgrade it with the field manual: parent–child chunking, hybrid search, reranking, citations, abstention, and a golden set.
- Build a tabular ML + LLM report hybrid: [build guide](12-capstones-and-interviews/learn/03-build-guide-tabular-plus-llm.md).
- Package it and plan what is next: [portfolio and next steps](12-capstones-and-interviews/learn/04-portfolio-and-next-steps.md).

**Interview and design-review drills:** [60 scenario questions](12-capstones-and-interviews/learn/05-scenario-based-prod-ai-questions.md) covering classical ML in production, RAG, agents, serving, data, safety, organisation and multi-tenant cost crises. Answer aloud in 8–12 minutes, name at least two rejected options, and always end with eval, HITL and rollback.

---

## 15. Cheat sheets

### 15.1 Decision rules worth memorising

| Situation | Rule |
|---|---|
| Model doesn't know something | RAG, not fine-tuning |
| Model knows but says it wrong | Prompt first, fine-tune if prompting doesn't scale |
| Tabular prediction | Gradient boosting before deep learning or an LLM |
| First RAG upgrade | Parent–child chunking, then hybrid search, then a reranker |
| Multi-turn RAG breaks at turn 3 | Add a query-rewriting step |
| Choosing an agent design | Fixed workflow with LLM steps; free-form agent only when the path can't be enumerated |
| Irreversible action | Human approval gate and least-privilege credentials |
| Cost is 10× the estimate | Prompt caching, rerank-then-trim, step budgets |
| GPU capacity | Pack → replicate → bigger SKU → tensor parallel; scale on queue + TTFT |
| Any release | Golden set gate, canary, pinned versions, one-command rollback |

### 15.2 Metrics by stage

| Stage | Metric | Why it matters |
|---|---|---|
| Parsing | CER/WER, table-cell accuracy, reading-order accuracy | Caps everything downstream |
| Chunking | Chunk-level recall on golden queries | Detects splits that orphan the answer |
| Retrieval | recall@k, precision@k, MRR, nDCG | Was the answer even present? |
| Reranking | nDCG@5 vs baseline, position of first relevant | Isolates the rerank gain |
| Generation | Faithfulness, answer relevance, citation accuracy | Hallucination and grounding |
| Refusal | Correct-abstention rate, false-refusal rate | The most-skipped RAG metric |
| Agent | Task success, trajectory validity, steps/run, cost/run | Accuracy alone hides cost blowups |
| Classical ML | Precision/recall/F1, AUC, RMSE vs baseline, calibration | Honest comparison to a dumb baseline |
| Production | p50/p95 latency, TTFT, $/request, error rate, cache-hit rate | SLOs and unit economics |
| Business | Deflection, time saved, human-escalation rate | The numbers leadership funds |

More: [metrics cheat sheet](10-production/apply/7-metrics-failure-modes/14.1_metrics_cheat_sheet.md).

### 15.3 Failure modes and fixes

| Symptom | Usual cause | Fix |
|---|---|---|
| Right doc exists, never retrieved | Chunk lacks context or keywords; dense-only search | Hybrid + contextual retrieval + rewriting |
| Answer cites the wrong section | Chunks too large, no reranking | Parent–child + cross-encoder rerank |
| Confidently wrong on a fresh question | No abstention path | Score threshold + faithfulness check |
| Correct at turn 1, wrong at turn 3 | No query rewriting over history | Coreference-resolving rewrite |
| Costs 10× the estimate | No caching, top-k too large, agent loops | Prompt cache, rerank-then-trim, step budgets |
| Works in dev, fails in prod | Eval set isn't representative | Sample real traffic into the golden set |
| Table questions always wrong | Table flattened at parse time | Structure-preserving parser, table-aware chunks |
| Agent takes a destructive action | Excessive agency + injection | Least-privilege tokens, HITL gate, data ≠ instructions |
| Great offline score, bad in prod (classical) | Leakage or training/serving skew | Fit preprocessing on train only; share feature code |
| Quality silently drops after a vendor update | Unpinned model version | Pin versions; canary every upgrade |

More: [common failure modes](10-production/apply/7-metrics-failure-modes/15.1_common_failure_modes.md).

### 15.4 Starting HITL rates

| Scenario | Starting point (calibrate) |
|---|---|
| Low-tier drafts, ~10k/day | 0.5–1% random + all user flags |
| Medium support answers, ~1k/day | 2–5% random + 100% thumbs-down + empty retrieval |
| High-tier refunds, ~50/day | 100% pre-action; relax only after months of clean audits |
| Canary week | 2–5× normal sample |
| After a serious incident | 100% temporarily |

---

## 16. Study plans

### 16.1 Full path (part-time, roughly 6–8 months)

| Weeks | Focus | Material |
|---|---|---|
| 1–4 | Foundations | Stage 1 |
| 5–7 | Classical ML | Stage 2 |
| 8–11 | Deep learning | Stage 3 |
| 12–19 | LLMs: models, prompting, RAG, tools, levers, evaluation | Stages 4–9 |
| 20–25 | Production AI and architecture patterns | Stages 10–11 |
| 26–31 | Capstone and portfolio | Stage 12 |

### 16.2 Fast track for working software engineers (about 8 weeks)

| Week | Focus |
|---|---|
| 1 | Checklists for Stages 1–3; patch any gaps. Read section 2 of this guide. |
| 2 | Foundation models and prompting (stages 4–5) |
| 3 | RAG end to end (stage 6), building a minimal Docs Q&A |
| 4 | Golden set and eval harness for that build (stage 9) |
| 5 | Tools and a constrained agent (stage 7); the lever decision (stage 8) |
| 6 | Serving, cost, monitoring (stage 10.1–10.5) |
| 7 | Safety, reliability, flywheel (stage 10.6–10.8); architecture patterns |
| 8 | Polish the capstone; drill 10 scenario questions |

### 16.3 Interview sprint (about 2 weeks)

Day 1–2: sections 2, 12 and 15 of this guide. Day 3–4: [Lesson 11.1 patterns](11-architecture-patterns/learn/01-ai-architecture-patterns-for-prod.md). Day 5–12: four scenario questions a day from [Lesson 12.5](12-capstones-and-interviews/learn/05-scenario-based-prod-ai-questions.md), cross-checking answers against the field manual. Day 13–14: rehearse a five-minute walkthrough of your capstone.

---

## 17. Master index: topic to file

"Learn" points to the curriculum lesson; "Apply" points to the field-manual scenarios. Each stage folder's `README.md` lists every file in it.

| Topic | Stage folder | Learn | Apply |
|---|---|---|---|
| Python, math, data | [01-foundations](01-foundations/README.md) | [overview](01-foundations/learn/00-overview.md) | — |
| Classical ML | [02-classical-ml](02-classical-ml/README.md) | [overview](02-classical-ml/learn/00-overview.md) | — |
| Deep learning | [03-deep-learning](03-deep-learning/README.md) | [overview](03-deep-learning/learn/00-overview.md) | — |
| Model basics and selection | [04-llm-foundations](04-llm-foundations/README.md) | [foundation models basics](04-llm-foundations/learn/01-foundation-models-basics.md) | [model selection, routing, build vs buy](04-llm-foundations/apply/) |
| Prompting | [05-prompt-and-context-engineering](05-prompt-and-context-engineering/README.md) | [prompting deep dive](05-prompt-and-context-engineering/learn/01-prompting-deep-dive.md) | [1-prompt-engineering/](05-prompt-and-context-engineering/apply/1-prompt-engineering/) |
| Context engineering | [05-prompt-and-context-engineering](05-prompt-and-context-engineering/README.md) | [prompting deep dive](05-prompt-and-context-engineering/learn/01-prompting-deep-dive.md) | [2-context-engineering/](05-prompt-and-context-engineering/apply/2-context-engineering/) |
| Ingestion and parsing | [06-rag](06-rag/README.md) | [RAG deep dive §2](06-rag/learn/01-rag-deep-dive.md) | [1-ingestion-parsing/](06-rag/apply/1-ingestion-parsing/) |
| RAG pipeline | [06-rag](06-rag/README.md) | [RAG deep dive](06-rag/learn/01-rag-deep-dive.md) | [2-chunking/ through 8-generation-grounding/](06-rag/apply/) |
| Tools and agents | [07-agents-tools-memory](07-agents-tools-memory/README.md) | [tools, agents, structured output](07-agents-tools-memory/learn/01-tools-agents-and-structured-output.md) | [1-agents/](07-agents-tools-memory/apply/1-agents/) |
| Memory | [07-agents-tools-memory](07-agents-tools-memory/README.md) | [conversation state](04-llm-foundations/learn/01-foundation-models-basics.md) | [2-memory/](07-agents-tools-memory/apply/2-memory/) |
| Prompt vs RAG vs fine-tune | [08-choosing-the-lever](08-choosing-the-lever/README.md) | [decision framework](08-choosing-the-lever/learn/01-fine-tuning-vs-rag-vs-prompt.md) | [01.3](08-choosing-the-lever/apply/01.3_prompting_vs_rag_vs_finetuning.md), [fine-tuning/](08-choosing-the-lever/apply/fine-tuning/) |
| Evaluation and testing | [09-evaluation](09-evaluation/README.md) | [evaluation for LLM systems](09-evaluation/learn/01-evaluation-for-llm-systems.md) | [evals and testing pyramid](09-evaluation/apply/) |
| Decision framework and risk tiers | [10-production](10-production/README.md) | [production mindset](10-production/learn/01-production-mindset-and-decision-framework.md) | [build vs buy](04-llm-foundations/apply/01.4_build_vs_buy.md) |
| Data pipelines and MLOps | [10-production](10-production/README.md) | [data pipelines and MLOps](10-production/learn/02-data-pipelines-and-mlops.md) | [environments](10-production/apply/1-serving-cost-latency/09.6_environments.md), [fine-tuning ops](08-choosing-the-lever/apply/fine-tuning/11.6_fine_tuning_ops.md) |
| Serving, cost, latency | [10-production](10-production/README.md) | [serving architectures](10-production/learn/03-serving-architectures.md) | [1-serving-cost-latency/](10-production/apply/1-serving-cost-latency/), [3-deployment/](10-production/apply/3-deployment/) |
| GPU capacity | [10-production](10-production/README.md) | [PyTorch devices](03-deep-learning/learn/02-pytorch-training-loop.md) | [GPUResizing.md](10-production/apply/2-gpu/GPUResizing.md), [2-gpu/details/](10-production/apply/2-gpu/details/) |
| Monitoring and drift | [10-production](10-production/README.md) | [monitoring, drift, cost](10-production/learn/04-monitoring-drift-and-cost.md) | [4-observability-reliability/](10-production/apply/4-observability-reliability/), [7-metrics-failure-modes/](10-production/apply/7-metrics-failure-modes/) |
| Guardrails, security, HITL | [10-production](10-production/README.md) | [safety, privacy, HITL](10-production/learn/05-safety-privacy-and-hitl.md) | [5-guardrails/](10-production/apply/5-guardrails/), [6-security-privacy-governance/](10-production/apply/6-security-privacy-governance/), [HITL](07-agents-tools-memory/apply/1-agents/05.8_human_in_the_loop.md) |
| Reliability and incidents | [10-production](10-production/README.md) | [reliability and incidents](10-production/learn/06-reliability-and-incident-response.md) | [failure handling](10-production/apply/4-observability-reliability/08.7_failure_handling.md), [failure modes](10-production/apply/7-metrics-failure-modes/15.1_common_failure_modes.md) |
| Data flywheel | [10-production](10-production/README.md) | [monitoring, drift, cost](10-production/learn/04-monitoring-drift-and-cost.md) | [8-data-flywheel/](10-production/apply/8-data-flywheel/) |
| Architecture patterns | [11-architecture-patterns](11-architecture-patterns/README.md) | [15 production patterns](11-architecture-patterns/learn/01-ai-architecture-patterns-for-prod.md) | [reference architecture](11-architecture-patterns/apply/13.1_reference_architecture.md) |
| Capstones | [12-capstones-and-interviews](12-capstones-and-interviews/README.md) | [overview](12-capstones-and-interviews/learn/00-overview.md) | [reference architecture](11-architecture-patterns/apply/13.1_reference_architecture.md) |
| Scenario drills | [12-capstones-and-interviews](12-capstones-and-interviews/README.md) | [60 scenarios](12-capstones-and-interviews/learn/05-scenario-based-prod-ai-questions.md) | All of the above |
