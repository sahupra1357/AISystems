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
- **`apply/`** holds the *field manual*: one file per production scenario, each with implementation steps, an example, trade-offs, scale notes, a checklist and metrics. It tells you **what to do** when you are building. [`AI_System.md`](AI_System.md), at the root, is the one-page version of the whole field manual.

Stages 1 to 3 are curriculum only.

This guide stitches the stages together. Each section below gives you the core ideas in a few paragraphs, the rules worth memorising, and links into both `learn/` and `apply/` so you can go deep exactly where you need to.

---

## Contents

- [How to use this guide](#how-to-use-this-guide)
- [The three mental models](#the-three-mental-models)
- [Stage 1: Foundations](#stage-1-foundations)
- [Stage 2: Classical machine learning](#stage-2-classical-machine-learning)
- [Stage 3: Deep learning](#stage-3-deep-learning)
- [Stage 4: Foundation models and LLMs](#stage-4-foundation-models-and-llms)
- [Stage 5: Prompt and context engineering](#stage-5-prompt-and-context-engineering)
- [Stage 6: Retrieval-augmented generation](#stage-6-retrieval-augmented-generation)
- [Stage 7: Tools, agents and memory](#stage-7-tools-agents-and-memory)
- [Stage 8: Choosing the lever (prompt, RAG, fine-tune)](#stage-8-choosing-the-lever)
- [Stage 9: Evaluation](#stage-9-evaluation)
- [Stage 10: Production engineering](#stage-10-production-engineering)
- [Stage 11: Architecture patterns](#stage-11-architecture-patterns)
- [Stage 12: Capstones, portfolio and interviews](#stage-12-capstones-portfolio-and-interviews)
- [Cheat sheets](#cheat-sheets)
- [Master index: topic to file](#master-index-topic-to-file)

---

## How to use this guide

**If you are new to AI:** follow stages 1 to 12 in order. For each stage, read the curriculum lessons first (the "Learn" links), do the exercises, pass the checklist, then skim the field-manual files (the "Apply" links) so you know they exist.

**If you already ship software and want to build LLM features:** skim stages 1 to 3 against their checklists, then start properly at stage 4. Spend most of your time on stages 6, 9 and 10.

**If you are building something right now:** jump to the stage that matches the problem, read the "Rules" block, then open the field-manual file for the exact scenario. Use [the master index](#master-index-topic-to-file) to find it.

**If you are preparing for interviews or design reviews:** read [the three mental models](#the-three-mental-models), [Stage 10](#stage-10-production-engineering) and [the cheat sheets](#cheat-sheets), then drill the 60 scenarios in [Lesson 12.5](12-capstones-and-interviews/learn/05-scenario-based-prod-ai-questions.md).

Every field-manual file follows the same shape: what it is, how to implement, small example, pros and cons, when to use, production at scale, checklist and metrics, and a 30-second takeaway. Every curriculum part follows: overview, lessons, exercises and a "ready for the next part" checklist.

---

## The three mental models

Almost every decision in AI engineering is easier once these three pictures are in your head.

### An AI feature is a system, not a model call

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

### Production AI is a loop

```text
Data & labels → Train / adapt → Serve → Observe → Improve (eval gates + HITL) → back to data
```

A model that works in a notebook is not a product. You need versioned data, rollbackable releases, monitors for silent failure, explainable cost, safety controls, and explicit decisions about when humans review. Sources: [Stage 10 overview](10-production/learn/00-overview.md), [data flywheel](10-production/apply/8-data-flywheel/data_flywheel.md).

### The end-to-end reference architecture

```text
OFFLINE   sources → ingest (dedupe, hash, ACL) → parse (route per page, validate)
          → chunk (parent/child, contextual) → embed (versioned) → index (vector + BM25 + graph)

ONLINE    user → authn/z → input guardrails → query understanding (rewrite, multi-query,
          decompose, filters) → route → hybrid retrieve (ACL-filtered) → rerank → top-k
          → context assembly (budget, order, dedupe, cite) → LLM / agent loop (tools, HITL)
          → grounding check → output guardrails → stream to user
          → traces, cost, feedback → eval set → flywheel
```

Stages 4 to 10 below walk this diagram left to right. Sources: [AI_System.md reference architecture](AI_System.md#reference-architecture-end-to-end), [reference architecture](11-architecture-patterns/apply/reference_architecture.md).

---

## Stage 1: Foundations

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

## Stage 2: Classical machine learning

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

## Stage 3: Deep learning

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

## Stage 4: Foundation models and LLMs

**Goal:** reason about what an LLM is doing, what it costs, and how it fails, before building on top of it.

**Core ideas**

- **Next-token prediction becomes chat behaviour** through instruction tuning and preference tuning. The model predicts tokens; everything else is scaffolding.
- **Tokens and context windows.** You pay per token, latency grows with output tokens, and the window caps what the model can see at once.
- **Embedding models versus generative models.** Embedders turn text into vectors for search; generators produce text. A RAG system uses both.
- **Decoding knobs.** Temperature and top-p trade determinism for diversity. Use temperature 0 for extraction, classification and evals.
- **Failure modes to expect:** hallucination, format drift, instruction-following lapses, sensitivity to prompt wording, stale knowledge, and confident wrong answers.

**Choosing models** ([model selection](04-llm-foundations/apply/model_selection.md), [routing and escalation](04-llm-foundations/apply/routing_escalation.md))

- Pick by capability, cost, latency and context window. Small models with good prompts and good retrieval routinely beat large models with bad context.
- A cheap fast model handles classification, routing, extraction, query rewriting and light reranking. A reasoning model handles final answers, planning and critique.
- Route cheap-first and escalate on low confidence. If more than about 40% of traffic escalates, the router is not earning its complexity.
- **Build versus buy:** managed retrieval gets a demo in a day; you own the pipeline once you need custom chunking, reranking or ACLs ([build vs buy](04-llm-foundations/apply/build_vs_buy.md)).

**Learn:** [foundation models basics](04-llm-foundations/learn/01-foundation-models-basics.md) (includes a worked cost estimate for a naive support bot and a "the model is being weird" debugging playbook)

---

## Stage 5: Prompt and context engineering

**Goal:** get reliable behaviour from a model through what you put in its window.

### Prompt engineering

- **Structure:** system/role → task → constraints → examples → output format. Put **stable content first** so the prompt-cache prefix hits ([prompt structure](05-prompt-and-context-engineering/apply/1-prompt-engineering/prompt_structure.md)).
- **Delimit data from instructions.** Retrieved text, user uploads and tool output go inside clearly marked blocks the instructions say to treat as data ([delimiters](05-prompt-and-context-engineering/apply/1-prompt-engineering/delimiters_data_vs_instructions.md)).
- **Techniques:** zero/few-shot ([Zero-shot and few-shot prompting](05-prompt-and-context-engineering/apply/1-prompt-engineering/zero_few_shot.md)), chain-of-thought when reasoning pays and not for extraction ([Chain-of-thought](05-prompt-and-context-engineering/apply/1-prompt-engineering/chain_of_thought.md)), self-consistency ([Self-consistency](05-prompt-and-context-engineering/apply/1-prompt-engineering/self_consistency.md)), decomposition, negative examples and prefill ([Decomposition, negative examples, and prefilling](05-prompt-and-context-engineering/apply/1-prompt-engineering/decomposition_negative_examples_prefill.md)).
- **Dynamic few-shot:** retrieve the k most similar solved cases. It is a mini-RAG and beats static examples on domain tasks ([Dynamic few-shot selection](05-prompt-and-context-engineering/apply/1-prompt-engineering/dynamic_few_shot_selection.md)).
- **Structured outputs:** JSON schema, function calling, and validate-then-retry with the error fed back. Track schema violations as a metric ([Structured outputs & tool use](05-prompt-and-context-engineering/apply/1-prompt-engineering/structured_outputs_tool_use.md)).
- **Prompts are code:** version them, diff them, run the eval suite on every change, and log the prompt version with every call ([Prompt versioning & evaluation](05-prompt-and-context-engineering/apply/1-prompt-engineering/prompt_versioning_evaluation.md)).

### Context engineering

Context engineering manages *everything* in the window: instructions, retrieved documents, tool results, memory and history. The goal is the right information, in the right position, with minimal noise ([what it is](05-prompt-and-context-engineering/apply/2-context-engineering/what_is_context_engineering.md)).

- Keep a rolling summary plus the last N turns verbatim ([compaction](05-prompt-and-context-engineering/apply/2-context-engineering/compaction_summarization.md)).
- Put the most relevant chunk first or last, never buried in the middle ([lost in the middle](05-prompt-and-context-engineering/apply/2-context-engineering/lost_in_the_middle.md)).
- Cap tokens per source so one verbose tool result cannot crowd out the rest ([budgets](05-prompt-and-context-engineering/apply/2-context-engineering/per_source_truncation_budgets.md)).
- Cache static prefixes; often the single biggest cost lever in a chat product ([prompt caching](05-prompt-and-context-engineering/apply/2-context-engineering/prompt_caching.md)).
- Give subagents their own window instead of appending to the orchestrator's ([isolation](05-prompt-and-context-engineering/apply/2-context-engineering/context_isolation.md)).
- Name the failure modes: poisoning, distraction, confusion and clash ([failure modes](05-prompt-and-context-engineering/apply/2-context-engineering/context_failure_modes.md)).

**Learn:** [prompting deep dive](05-prompt-and-context-engineering/learn/01-prompting-deep-dive.md) (anatomy, roles, styles, task playbooks, parameters, iteration method, anti-patterns, and a worked case from playground to template).

---

## Stage 6: Retrieval-augmented generation

**Goal:** answer from private, fresh or large knowledge with citations a user can trust.

RAG is a pipeline, and quality is capped by its weakest stage. Work through it in build order.

### Ingestion and parsing (the highest-leverage stage)

Garbage parsing caps everything downstream. No reranker recovers a table flattened into word soup.

- **Ingest:** connectors with auth, pagination and rate limits ([Connectors](06-rag/apply/1-ingestion-parsing/connectors.md)); change detection with a size/mtime then SHA-256 ladder so you only re-embed what changed ([Change detection](06-rag/apply/1-ingestion-parsing/change_detection.md)); exact and near-duplicate removal ([Deduplication](06-rag/apply/1-ingestion-parsing/deduplication.md)); `doc_id + version` with soft-deleted stale chunks ([Document versioning](06-rag/apply/1-ingestion-parsing/document_versioning.md)); tenant and ACL captured at ingest, never retrofitted ([Tenant / ACL capture at ingest](06-rag/apply/1-ingestion-parsing/tenant_acl_capture_at_ingest.md)).
- **Route per page, not per document** ([Per-page classification and routing](06-rag/apply/1-ingestion-parsing/per_page_routing.md)): digital pages to a native text-layer parser ([Digital document](06-rag/apply/1-ingestion-parsing/digital_document_docling.md)), scans to OCR ([Scanned page](06-rag/apply/1-ingestion-parsing/scanned_page_ocr.md)), complex layouts to a layout model ([Complex layout](06-rag/apply/1-ingestion-parsing/complex_layout_ocr.md)), formulas to LaTeX extraction ([Formula-heavy page](06-rag/apply/1-ingestion-parsing/formula_heavy_mineru.md)), charts to VLM captions ([Charts, images, diagrams](06-rag/apply/1-ingestion-parsing/charts_images_vlm_captioning.md)).
- **Preserve structure:** reading order ([Reading order](06-rag/apply/1-ingestion-parsing/reading_order.md)), tables as HTML/Markdown ([Tables](06-rag/apply/1-ingestion-parsing/tables.md)), heading breadcrumbs ([Headings hierarchy](06-rag/apply/1-ingestion-parsing/headings_hierarchy.md)), page numbers and bounding boxes for citations ([Page numbers + bounding boxes](06-rag/apply/1-ingestion-parsing/page_numbers_bounding_boxes.md)), boilerplate stripped ([Headers / footers / boilerplate stripping](06-rag/apply/1-ingestion-parsing/boilerplate_stripping.md)).
- **Validate:** CER/WER and table-cell accuracy on a golden sample ([Test-time validation against a golden sample](06-rag/apply/1-ingestion-parsing/validation_golden_sample.md)); cheap per-page heuristics in production ([Production per-page heuristics](06-rag/apply/1-ingestion-parsing/validation_production_heuristics.md)); a bounded retry ladder that ends in a visible human queue, never silent empty text ([Bounded escalation with visible failure](06-rag/apply/1-ingestion-parsing/retry_ladder.md)).

### Chunking

| Strategy | Use it when | File |
|---|---|---|
| Fixed-size + overlap | Baseline only | [Fixed-size + overlap](06-rag/apply/2-chunking/fixed_size_overlap.md) |
| Recursive character splitting | Good general default | [Recursive character splitting](06-rag/apply/2-chunking/recursive_character_splitting.md) |
| Layout/structure-aware | The parser preserved headings and tables | [Layout / structure-aware](06-rag/apply/2-chunking/layout_structure_aware.md) |
| Semantic | Cohesion matters more than ingest cost | [Semantic chunking](06-rag/apply/2-chunking/semantic_chunking.md) |
| **Parent–child (small-to-big)** | **Highest-ROI upgrade in most systems** | [Parent–child](06-rag/apply/2-chunking/parent_child_small_to_big.md) |
| Contextual retrieval | Chunks full of pronouns and bare numbers | [Contextual retrieval](06-rag/apply/2-chunking/contextual_retrieval.md) |
| Late chunking | You have a long-context embedder | [Late chunking](06-rag/apply/2-chunking/late_chunking.md) |

Every chunk carries metadata: `doc_id`, title, breadcrumb, page, section, dates, tenant, ACL, source URL ([Metadata on every chunk](06-rag/apply/2-chunking/chunk_metadata.md)).

### Embeddings and indexing

- Choose the embedder for your domain and language, and validate on *your* golden set, not only a leaderboard ([Model choice](06-rag/apply/3-embeddings/embedding_model_choice.md)). Get query/passage prefixes right ([Symmetric vs asymmetric](06-rag/apply/3-embeddings/symmetric_vs_asymmetric.md)). Fine-tuning the embedder on your own triplets is often a bigger win than a bigger LLM ([Fine-tuning embeddings on your own logs](06-rag/apply/3-embeddings/fine_tuning_embeddings.md)). A model change means a full re-index; store the version per chunk and dual-index during migration ([Versioning & migration](06-rag/apply/3-embeddings/embedding_versioning.md)).
- HNSW is the default index; flat for small corpora; IVF-PQ when memory-bound ([Index types: Flat, HNSW, IVF-PQ](06-rag/apply/4-indexing/index_types_flat_hnsw_ivfpq.md)). Choose the store by what else you need: pgvector for joins and transactional ACLs, Qdrant/Weaviate/Milvus for scale and filtering, a managed service for no ops, OpenSearch when you also need BM25 and aggregations ([Vector store choice](06-rag/apply/4-indexing/vector_store_choice.md)). Prefer native filtered ANN; post-filtering by tenant can return zero rows ([Filtered search](06-rag/apply/4-indexing/filtered_search.md)).

### Query understanding

Rewrite follow-ups using chat history, or multi-turn RAG falls apart by turn three ([Query rewriting](06-rag/apply/5-query-understanding/query_rewriting.md)). Add multi-query with Reciprocal Rank Fusion for cheap recall ([Multi-query + Reciprocal Rank Fusion](06-rag/apply/5-query-understanding/multi_query_rrf.md)), HyDE for short queries against verbose documents ([HyDE](06-rag/apply/5-query-understanding/hyde.md)), decomposition for multi-hop questions ([Decomposition](06-rag/apply/5-query-understanding/decomposition.md)), self-query to turn language into hard filters ([Self-query](06-rag/apply/5-query-understanding/self_query_metadata_filters.md)), and routing so that not every question hits the index ([Routing](06-rag/apply/5-query-understanding/query_routing.md)).

### Retrieval and ranking

- **Hybrid search** (BM25 + dense, fused) beats either alone on virtually every real corpus, because IDs, part numbers and names are where dense retrieval fails ([Hybrid search](06-rag/apply/6-retrieval-ranking/hybrid_search.md)).
- **Rerank:** retrieve top-50 cheaply, cross-encoder rerank to top-5. Usually the largest single quality jump ([Reranking](06-rag/apply/6-retrieval-ranking/reranking.md)).
- Diversify with MMR ([MMR / diversity](06-rag/apply/6-retrieval-ranking/mmr_diversity.md)), abstain below a tuned score floor ([Score thresholding & abstention](06-rag/apply/6-retrieval-ranking/score_thresholding_abstention.md)), and bound any follow-up searches ([Recursive / agentic retrieval](06-rag/apply/6-retrieval-ranking/recursive_agentic_retrieval.md)).

### Advanced patterns

Reach for these only when the basics are measured and insufficient: agentic RAG ([Agentic RAG](06-rag/apply/7-advanced-rag/agentic_rag.md)), corrective RAG ([Corrective RAG (CRAG): grade, then rewrite or fall back](06-rag/apply/7-advanced-rag/corrective_rag.md)), Self-RAG ([Self-RAG](06-rag/apply/7-advanced-rag/self_rag.md)), GraphRAG for global and multi-hop questions ([GraphRAG / knowledge graph](06-rag/apply/7-advanced-rag/graphrag.md)), multimodal RAG for slides and scans ([Multimodal RAG](06-rag/apply/7-advanced-rag/multimodal_rag.md)), and SQL for aggregates alongside vectors for narrative, because many "RAG failures" are analytics questions ([Structured + unstructured hybrid](06-rag/apply/7-advanced-rag/structured_unstructured_hybrid.md)).

### Generation and grounding

Answer only from context, then verify ([Grounded prompting](06-rag/apply/8-generation-grounding/grounded_prompting.md)). Require citations in the output schema and validate every cited ID ([Citations](06-rag/apply/8-generation-grounding/citations.md)). Run a cheap faithfulness pass ([Faithfulness check](06-rag/apply/8-generation-grounding/faithfulness_check.md)). Surface conflicting sources with dates instead of picking one ([Conflict handling](06-rag/apply/8-generation-grounding/conflict_handling.md)). Design and evaluate the refusal path ([Refusal / no-answer path](06-rag/apply/8-generation-grounding/refusal_no_answer_path.md)).

**Rules**

- Permission-filter retrieval at query time with the caller's identity. Access control is non-negotiable and never lives in the prompt.
- Log retrieved chunk IDs and scores on every request. You cannot debug an answer without knowing what it read.
- Evaluate retrieval separately from generation. If the answer was never retrieved, generation metrics are noise.

**Learn:** [RAG deep dive](06-rag/learn/01-rag-deep-dive.md) (corpus design, chunking workshop, hybrid fusion sketch, citations, ACLs, debugging playbook, worked handbook Q&A). **Build:** [Docs Q&A RAG capstone](12-capstones-and-interviews/learn/02-build-guide-docs-qa-rag.md).

---

## Stage 7: Tools, agents and memory

**Goal:** let models take actions safely, with bounded cost and a clear stop point.

### Tools and structured output

Tools beat hallucinated actions: the model proposes a call, your code validates and executes it. A good tool has a precise name, a description that says when *not* to use it, a strict argument schema, and an explicit side-effect contract. Prefer fewer, well-described, idempotent tools over one "supertool" ([tool design](07-agents-tools-memory/apply/1-agents/tool_design.md)).

### Agent shapes, simplest first

1. **Single tool call** inside a fixed workflow. Most production "agents" should be this.
2. **Plan-then-execute or a DAG** for predictable pipelines.
3. **Single agent loop** (LLM → tool → observe → repeat) with a termination condition, max iterations, per-step timeouts and errors fed back to the model ([Single agent loop](07-agents-tools-memory/apply/1-agents/single_agent_loop.md)).
4. **Orchestrator and subagents** with explicit task contracts, narrow scope and isolated context ([Task decomposition](07-agents-tools-memory/apply/1-agents/task_decomposition.md), [Orchestrator / subagent](07-agents-tools-memory/apply/1-agents/orchestrator_subagent.md)), fan-out/fan-in with partial-failure handling ([Fan-out / fan-in](07-agents-tools-memory/apply/1-agents/fan_out_fan_in.md)), and bounded reflection ([Loop-back / reflection](07-agents-tools-memory/apply/1-agents/loop_back_reflection.md)).

Choose ReAct for exploration, plan-then-execute for predictable work, and search only when steps are cheap and reversible ([planning styles](07-agents-tools-memory/apply/1-agents/planning_styles.md)).

### Making agents safe to run

- **Human-in-the-loop** approval before irreversible actions (send, pay, delete, deploy), with a clear resume path ([Human-in-the-loop](07-agents-tools-memory/apply/1-agents/human_in_the_loop.md)).
- **Durability:** checkpoint after each step and use idempotency keys so a resumed run does not double-send ([State & durability](07-agents-tools-memory/apply/1-agents/state_durability.md)).
- **Sandboxing:** code runs in a container with no secrets, egress policy, resource limits and read-only mounts ([Sandboxing](07-agents-tools-memory/apply/1-agents/sandboxing.md)).
- **Interoperability:** MCP for tool discovery and reuse, A2A-style handoffs, or a shared blackboard ([Multi-agent communication](07-agents-tools-memory/apply/1-agents/multi_agent_communication_mcp.md)).
- **Budgets:** hard caps on tokens, tool calls, wall clock and dollars, enforced by the runtime ([Cost & step budgets](07-agents-tools-memory/apply/1-agents/cost_step_budgets.md)).
- **Least privilege:** a read-only agent with a read-only token cannot be talked into a delete ([excessive agency](10-production/apply/5-guardrails/excessive_agency.md)).

### Memory

Short-term (the window), working (a scratchpad), episodic (past interactions), semantic (durable facts) and procedural (how-tos) ([Memory types](07-agents-tools-memory/apply/2-memory/memory_types.md)). Write only what an extraction step judges memory-worthy ([What to write to memory](07-agents-tools-memory/apply/2-memory/what_to_write.md)); retrieve by relevance and recency with a cap ([When to retrieve memory](07-agents-tools-memory/apply/2-memory/when_to_retrieve.md)); forget with TTLs and user controls ([How to forget](07-agents-tools-memory/apply/2-memory/how_to_forget.md)); supersede changed facts with `valid_from` ([Memory conflict resolution](07-agents-tools-memory/apply/2-memory/conflict_resolution.md)); and treat memory as data, because it is an injection surface ([Memory security](07-agents-tools-memory/apply/2-memory/memory_security.md)).

**Learn:** [tools, agents and structured output](07-agents-tools-memory/learn/01-tools-agents-and-structured-output.md) (argument validation catalogue, tool result hygiene, deterministic test stubs, cost model, worked refund assistant, and a pre-production orchestrator checklist).

---

## Stage 8: Choosing the lever

**Goal:** fix the right problem with the right tool.

| Lever | Changes | Cost to iterate | Use when |
|---|---|---|---|
| **Prompting** | Behaviour: tone, format, procedure | Minutes, no training | Always first |
| **RAG** | Knowledge: fresh, private or too large for the weights | Days; ongoing index ops | "It doesn't know" |
| **Fine-tuning** | Style, format or a narrow domain, at scale; or cost via distillation | Weeks; retraining and eval burden | "It knows but says it wrong," and prompting does not scale |
| **Change the product** | The task itself | Varies | The model is being asked to do something it should not |

**Rule of thumb:** if the failure is *"it doesn't know,"* use RAG. If it is *"it knows but says it wrong,"* fix the prompt, and fine-tune only if prompting cannot scale. Production systems usually combine all three.

**Fine-tuning, when you get there:** SFT on a few hundred to a few thousand high-quality pairs ([SFT](08-choosing-the-lever/apply/fine-tuning/sft.md)); LoRA/QLoRA adapters as the default ([PEFT / LoRA / QLoRA](08-choosing-the-lever/apply/fine-tuning/peft_lora_qlora.md)); DPO for subjective quality ([Preference tuning](08-choosing-the-lever/apply/fine-tuning/preference_tuning_dpo_rlhf.md)); distillation to cut cost once behaviour is stable ([Distillation](08-choosing-the-lever/apply/fine-tuning/distillation.md)). Do not fine-tune for changing knowledge, unstable tasks, or anything a better prompt fixes ([When *not* to fine-tune](08-choosing-the-lever/apply/fine-tuning/when_not_to_fine_tune.md)). Version datasets with checkpoints and keep a held-out split from day one ([Fine-tuning ops](08-choosing-the-lever/apply/fine-tuning/fine_tuning_ops.md)).

**Learn:** [fine-tuning vs RAG vs prompt](08-choosing-the-lever/learn/01-fine-tuning-vs-rag-vs-prompt.md) (decision flowchart, scenario workshop, LoRA intuition, data pipeline, eval gates, a copy-paste decision worksheet). **Apply:** [prompting vs RAG vs fine-tuning](08-choosing-the-lever/apply/prompting_vs_rag_vs_finetuning.md).

---

## Stage 9: Evaluation

**Goal:** know whether a change made things better, before users tell you.

Evaluation is the skill that most separates AI engineers from people who call APIs. Treat it as a first-class release control.

- **Golden dataset first:** 50 to 200 real, hand-labelled cases beat any generic benchmark. Grow it from production failures; every incident becomes a test case ([Golden dataset first](09-evaluation/apply/golden_dataset.md)).
- **Prompt evaluation:** run variants against the golden set and track regressions per prompt version ([Prompt evaluation](09-evaluation/apply/prompt_evaluation.md)).
- **LLM-as-judge:** use it for scale, calibrate it against human labels, report agreement, and watch for position, length and self-preference bias. Never the sole truth ([LLM-as-judge](09-evaluation/apply/llm_as_judge.md)).
- **Separate the failure sources in RAG:** retrieval (recall@k, MRR, nDCG) versus generation (faithfulness, relevance, citation accuracy) ([Retrieval evals](09-evaluation/apply/retrieval_evals.md), [Generation evals](09-evaluation/apply/generation_evals.md)).
- **Agents:** evaluate the trajectory and the cost, not only the final answer. A 2% gain for 4× the tool calls is usually a loss ([Agent evals](09-evaluation/apply/agent_evals.md)).
- **Component then end-to-end:** end-to-end-only evals tell you something broke, not what ([Component vs end-to-end](09-evaluation/apply/component_vs_end_to_end.md)).
- **Statistical hygiene:** temperature 0, multiple seeds where sampling matters, confidence intervals, and scepticism about a 3-point move on 50 cases ([Statistical hygiene](09-evaluation/apply/statistical_hygiene.md)).
- **Online signals:** thumbs, edit distance to what the user shipped, task completion and escalation rate, with production traffic sampled back into the eval set ([Online eval](09-evaluation/apply/online_eval.md)).
- **Score abstention correctly.** Most RAG systems are never measured on whether they correctly decline.
- **Report by slice** (customer tier, language, document type), because averages hide the segment that is failing.

**Testing pyramid:** unit tests for tools and parsers → eval suites for prompts → integration tests for agent flows with mocked tools → red-team tests → evals in CI that block merges on regression ([The testing pyramid](09-evaluation/apply/testing_pyramid.md)).

**Learn:** [evaluation for LLM systems](09-evaluation/learn/01-evaluation-for-llm-systems.md) (metric formulas, rubrics, pairwise comparison, regression gates, human review session recipe, eval harness sketch, release checklist). **Practise:** [Stages 4–9 capstone: prompt library + eval set](09-evaluation/learn/02-exercises-and-checklist.md).

---

## Stage 10: Production engineering

**Goal:** ship AI features that are reliable, observable, affordable and safe, and that improve over time.

### Decide deliberately

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

### Data pipelines and MLOps

Version datasets and RAG corpora, track experiments, and promote models, prompts and indexes through a registry with eval gates. Separate dev, staging and prod indexes; never evaluate against the prod index you are mutating ([Lesson 10.2](10-production/learn/02-data-pipelines-and-mlops.md), [environments](10-production/apply/1-serving-cost-latency/environments.md)).

### Serving

- **Shapes:** batch, sync API, async queue, streaming. Add a worker queue when work exceeds a request timeout or needs retries and review ([Lesson 10.3](10-production/learn/03-serving-architectures.md)).
- **Cost levers:** prompt caching, batching, routing, per-request token budgets, shorter system prompts, trimming top-k after reranking. Know your $/task and $/user/month ([Cost levers](10-production/apply/1-serving-cost-latency/cost_levers.md)).
- **Latency levers:** streaming (perceived latency is what users judge), semantic caching, parallel retrieval and tool calls, smaller models on the critical path ([Latency levers](10-production/apply/1-serving-cost-latency/latency_levers.md)).
- **Self-hosting:** vLLM/TGI with continuous batching, paged KV cache, quantisation and tensor parallelism ([Self-hosted serving](10-production/apply/1-serving-cost-latency/self_hosted_serving.md)). Rate limits, per-tenant throttling, backpressure and load shedding ([Capacity & limits](10-production/apply/1-serving-cost-latency/capacity_limits.md)).
- **Rollouts:** prompts, models, indexes and chunking configs are versioned artefacts. Shadow, canary or A/B; pin model versions; keep rollback one command away ([Versioning & rollout](10-production/apply/1-serving-cost-latency/versioning_rollout.md)).
- **Cloud deployment:** host the app on Fargate, run small stateless tasks on Lambda, and call the model through Bedrock ([AWS: Fargate, Lambda, Bedrock](10-production/apply/3-deployment/aws_bedrock_fargate_lambda.md)).

### GPU capacity

On GPUs the card is the unit of capacity and most clouds cannot hot-swap GPU type or count, so a resize is a controlled fleet replacement. **Exhaust software packing first, then horizontal replicas, then a bigger SKU, then tensor parallelism.** Size VRAM as weights + KV cache (context × concurrent sequences) + activations + 10–20% headroom; at production concurrency the KV cache usually dominates. Never autoscale on GPU utilisation; scale on queue depth and time-to-first-token, and keep generation pools warm. Read the playbook in [GPUResizing.md](10-production/apply/2-gpu/GPUResizing.md) and the detail in [`2-gpu/details/`](10-production/apply/2-gpu/details/).

### Monitoring, drift and cost

Log enough to replay any request: prompt version, model, tokens, latency, cost, tool calls, retrieved chunk IDs and scores, outcome, and a trace ID ([observability](10-production/apply/4-observability-reliability/observability_tracing.md)). Monitor quality, drift, latency and cost against SLOs; route thumbs-down, empty retrieval and low-confidence traffic to human review queues; and run a weekly operating rhythm ([Lesson 10.4](10-production/learn/04-monitoring-drift-and-cost.md)).

### Safety, privacy and security

Start with a threat model, then layer defences ([Lesson 10.5](10-production/learn/05-safety-privacy-and-hitl.md)):

- **Input guardrails:** PII detection and masking, topic and abuse classification ([Input guardrails](10-production/apply/5-guardrails/input_guardrails.md)). **Output guardrails:** leakage, toxicity, schema, banned claims ([Output guardrails](10-production/apply/5-guardrails/output_guardrails.md)).
- **Prompt injection:** all retrieved content, tool output and memory is data, never instructions; privileged actions sit behind a policy check the model does not control ([Prompt-injection defence](10-production/apply/5-guardrails/prompt_injection_defense.md)).
- **Access and tenancy:** filter at query time with the caller's identity; isolation belongs in the index ([Authentication & authorization](10-production/apply/6-security-privacy-governance/authn_authz_permission_filtered_retrieval.md)).
- **Privacy and governance:** data residency and retention ([Data residency & retention](10-production/apply/6-security-privacy-governance/data_residency_retention.md)), PII with reversible mapping ([PII](10-production/apply/6-security-privacy-governance/pii.md)), queryable audit trails ([Auditability](10-production/apply/6-security-privacy-governance/auditability.md)), pinned and vetted dependencies and MCP servers ([Model & supply-chain hygiene](10-production/apply/6-security-privacy-governance/supply_chain_hygiene.md)), and the OWASP LLM Top 10 as a checklist ([Threat checklist](10-production/apply/6-security-privacy-governance/owasp_llm_top10.md)).
- **HITL rate:** set it from error cost, volume and reviewer capacity (see [starting HITL rates](#starting-hitl-rates)), keep kill switches, and use an escalation ladder: auto-allow → soft flag → hard flag (HITL required) → deny and log.

### Reliability and incidents

Timeouts on every external call, retries with exponential backoff and jitter, idempotency, circuit breakers, fallback models across providers, and graceful degradation (return the sources with "couldn't summarise" rather than a 500) ([Failure handling](10-production/apply/4-observability-reliability/failure_handling.md)). Write runbooks for AI-specific failures, version everything so rollback is real, and run postmortems that ask which eval would have caught the incident ([Lesson 10.6](10-production/learn/06-reliability-and-incident-response.md)).

### The data flywheel

```text
prod traffic → traces + feedback → weekly triage → new golden cases
  → prompt / retrieval / model fix → eval → canary → prod
```

Bucket failures into parse, retrieval, ranking, generation and tooling; the buckets tell you where to spend. Mine hard negatives to fine-tune the embedder or reranker. Every fixed bug becomes a permanent regression test ([Data flywheel](10-production/apply/8-data-flywheel/data_flywheel.md), [Feedback capture & triage](10-production/apply/8-data-flywheel/feedback_capture_triage.md)).

**Practise:** [Stage 10 exercises and production design-doc capstone](10-production/learn/07-exercises-and-checklist.md).

---

## Stage 11: Architecture patterns

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

For the RAG and agent patterns, the field-manual counterpart is the [reference architecture](11-architecture-patterns/apply/reference_architecture.md).

---

## Stage 12: Capstones, portfolio and interviews

**Ship one excellent project rather than five half-finished clones.** A capstone is done when someone else can run it from the README, you report metrics on a held-out or golden set, you document failures honestly, and you can explain the architecture in five minutes without slides.

- Pick a project: [project catalogue](12-capstones-and-interviews/learn/01-project-catalog.md).
- Build a Docs Q&A RAG system end to end: [build guide](12-capstones-and-interviews/learn/02-build-guide-docs-qa-rag.md). Upgrade it with the field manual: parent–child chunking, hybrid search, reranking, citations, abstention, and a golden set.
- Build a tabular ML + LLM report hybrid: [build guide](12-capstones-and-interviews/learn/03-build-guide-tabular-plus-llm.md).
- Package it and plan what is next: [portfolio and next steps](12-capstones-and-interviews/learn/04-portfolio-and-next-steps.md).

**Interview and design-review drills:** [60 scenario questions](12-capstones-and-interviews/learn/05-scenario-based-prod-ai-questions.md) covering classical ML in production, RAG, agents, serving, data, safety, organisation and multi-tenant cost crises. Answer aloud in 8–12 minutes, name at least two rejected options, and always end with eval, HITL and rollback.

---

## Cheat sheets

### Decision rules worth memorising

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

### Metrics by stage

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

More: [metrics cheat sheet](10-production/apply/7-metrics-failure-modes/metrics_cheat_sheet.md).

### Failure modes and fixes

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

More: [common failure modes](10-production/apply/7-metrics-failure-modes/common_failure_modes.md).

### Starting HITL rates

| Scenario | Starting point (calibrate) |
|---|---|
| Low-tier drafts, ~10k/day | 0.5–1% random + all user flags |
| Medium support answers, ~1k/day | 2–5% random + 100% thumbs-down + empty retrieval |
| High-tier refunds, ~50/day | 100% pre-action; relax only after months of clean audits |
| Canary week | 2–5× normal sample |
| After a serious incident | 100% temporarily |

---

## Master index: topic to file

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
| Prompt vs RAG vs fine-tune | [08-choosing-the-lever](08-choosing-the-lever/README.md) | [decision framework](08-choosing-the-lever/learn/01-fine-tuning-vs-rag-vs-prompt.md) | [Prompting vs RAG vs fine-tuning](08-choosing-the-lever/apply/prompting_vs_rag_vs_finetuning.md), [fine-tuning/](08-choosing-the-lever/apply/fine-tuning/) |
| Evaluation and testing | [09-evaluation](09-evaluation/README.md) | [evaluation for LLM systems](09-evaluation/learn/01-evaluation-for-llm-systems.md) | [evals and testing pyramid](09-evaluation/apply/) |
| Decision framework and risk tiers | [10-production](10-production/README.md) | [production mindset](10-production/learn/01-production-mindset-and-decision-framework.md) | [build vs buy](04-llm-foundations/apply/build_vs_buy.md) |
| Data pipelines and MLOps | [10-production](10-production/README.md) | [data pipelines and MLOps](10-production/learn/02-data-pipelines-and-mlops.md) | [environments](10-production/apply/1-serving-cost-latency/environments.md), [fine-tuning ops](08-choosing-the-lever/apply/fine-tuning/fine_tuning_ops.md) |
| Serving, cost, latency | [10-production](10-production/README.md) | [serving architectures](10-production/learn/03-serving-architectures.md) | [1-serving-cost-latency/](10-production/apply/1-serving-cost-latency/), [3-deployment/](10-production/apply/3-deployment/) |
| GPU capacity | [10-production](10-production/README.md) | [PyTorch devices](03-deep-learning/learn/02-pytorch-training-loop.md) | [GPUResizing.md](10-production/apply/2-gpu/GPUResizing.md), [2-gpu/details/](10-production/apply/2-gpu/details/) |
| Monitoring and drift | [10-production](10-production/README.md) | [monitoring, drift, cost](10-production/learn/04-monitoring-drift-and-cost.md) | [4-observability-reliability/](10-production/apply/4-observability-reliability/), [7-metrics-failure-modes/](10-production/apply/7-metrics-failure-modes/) |
| Guardrails, security, HITL | [10-production](10-production/README.md) | [safety, privacy, HITL](10-production/learn/05-safety-privacy-and-hitl.md) | [5-guardrails/](10-production/apply/5-guardrails/), [6-security-privacy-governance/](10-production/apply/6-security-privacy-governance/), [HITL](07-agents-tools-memory/apply/1-agents/human_in_the_loop.md) |
| Reliability and incidents | [10-production](10-production/README.md) | [reliability and incidents](10-production/learn/06-reliability-and-incident-response.md) | [failure handling](10-production/apply/4-observability-reliability/failure_handling.md), [failure modes](10-production/apply/7-metrics-failure-modes/common_failure_modes.md) |
| Data flywheel | [10-production](10-production/README.md) | [monitoring, drift, cost](10-production/learn/04-monitoring-drift-and-cost.md) | [8-data-flywheel/](10-production/apply/8-data-flywheel/) |
| Architecture patterns | [11-architecture-patterns](11-architecture-patterns/README.md) | [15 production patterns](11-architecture-patterns/learn/01-ai-architecture-patterns-for-prod.md) | [reference architecture](11-architecture-patterns/apply/reference_architecture.md) |
| Capstones | [12-capstones-and-interviews](12-capstones-and-interviews/README.md) | [overview](12-capstones-and-interviews/learn/00-overview.md) | [reference architecture](11-architecture-patterns/apply/reference_architecture.md) |
| Scenario drills | [12-capstones-and-interviews](12-capstones-and-interviews/README.md) | [60 scenarios](12-capstones-and-interviews/learn/05-scenario-based-prod-ai-questions.md) | All of the above |
