# Building AI Systems — Complete Reference (RAG + Agents)

A start-to-end walk, roughly in the order a system actually gets built: **ingest → parse → chunk → embed → index → retrieve → rerank → generate → evaluate → serve → operate**. Sections 0–8 are the core build; 9–15 are what separates a demo from production.

---

## 0. Ingestion & Parsing

The single highest-leverage stage. Garbage parsing caps the quality of everything downstream — no reranker or bigger model recovers a table that was flattened into word soup.

### 0.1 Source ingestion (before the parser)
- **Connectors** — S3/GCS, SharePoint, Confluence, Gmail, Drive, DBs, web crawl. Each needs auth, pagination, and rate-limit handling.
- **Change detection** — Store a content hash (SHA-256 of raw bytes) + source `etag`/`modified_at`. Only re-parse and re-embed what changed; full re-index of a corpus is the most common avoidable cost.

        The ladder

        walk gives you: path, size, mtime, ACL     ← metadata only, no file read

        (size, mtime) unchanged?  ──► SKIP entirely. Never open the file.
                │ changed
                ▼
        read bytes + compute SHA-256              ← one pass, see below
                │
        hash == stored hash?      ──► content is identical (resave / touch /
                │                     robocopy / restore). Update mtime, skip
                │ differs           parse + embed.
                ▼
        re-parse → re-chunk → re-embed
- **Deduplication** — Exact dedupe by hash, near-dupe by MinHash/SimHash. Duplicate docs poison retrieval: top-k fills with five copies of the same paragraph and the model sees one fact instead of five.
- **Document versioning** — Keep `doc_id + version`, soft-delete old chunks. Users ask "what does the *current* policy say" — stale chunks answering confidently is a classic prod incident.
- **Tenant / ACL capture at ingest** — Record `tenant_id`, `owner`, `visibility`, `source_permissions` on every chunk *at ingest time*. Retrofitting access control onto an existing index is painful and usually leaks.

### 0.2 OCR / digitization
Inspect and classify each page, then route to the right parser:
- **Digital document** → Docling (native text layer, preserves structure)
- **Scanned page** → PaddleOCR
- **Complex layout** (multi-column, forms) → PaddleOCR (layout model)
- **Formula-heavy page** → MinerU (LaTeX extraction)
- **Charts / images / diagrams** → VLM captioning (describe the figure into text so it becomes retrievable)

**Why route per page, not per document:** a 200-page PDF is usually mixed — a digital body with scanned appendices. Per-page routing gets you the fast path where possible and the accurate path where necessary.

### 0.3 Structure extraction (what the parser must preserve)
- **Reading order** — Multi-column pages read out of order destroy sentence continuity.
- **Tables** — Emit as HTML or Markdown, not flattened text. A flattened table loses row/column association, which is exactly the relationship the question is about.
- **Headings hierarchy** — Becomes chunk metadata and the breadcrumb prepended to each chunk.
- **Page numbers + bounding boxes** — The basis for citations that link to a highlighted region, which is what makes the answer verifiable.
- **Headers/footers/boilerplate stripping** — Repeated legal footers dominate embeddings and inflate token cost.

### 0.4 Validation
**Test-time, against a golden sample:**
- Character error rate (CER), word error rate (WER)
- Field-level precision, recall, F1
- Table-cell accuracy
- Layout / reading-order accuracy

**In production, per page (cheap heuristics, no ground truth available):**
- Is there enough text? (`minimum_text`)
- Does it look like decode garbage? (`replacement_characters`, `control_characters`)
- Is it internally consistent? (`reading_order`, `element_ids`, `coordinates`, `table_structure`)
- Did the parser self-report adequate confidence? (`confidence`)

### 0.5 Retry ladder
```
Primary parser → quality checks
  ├── passed → continue
  └── failed → secondary parser, then vision model (sequentially)
        └── still failing → human review queue (only if unresolved)
```
**Design note:** every escalation costs more and takes longer, so the ladder must be bounded and the failure state must be *visible* — a page that silently degrades to empty text is worse than a page that loudly fails, because empty text produces a confident wrong answer instead of an error.

---

## 1. Foundations

- **Model selection & tradeoffs** — Pick by capability / cost / latency / context window. Small models with good prompts and good retrieval routinely beat large models with bad context. Practical split:
    - Cheap, fast model → classification, routing, extraction, reranking-lite, query rewriting
    - Reasoning model → final answer generation, planning, critique
    - **Routing/escalation** — cheap model first, escalate when confidence is low or the task is flagged complex. Measure the escalation rate; if it's >40% the router isn't earning its complexity.
- **Prompting vs RAG vs fine-tuning** — Three different levers for three different problems:
    - **Prompting** → changes *behavior* (tone, format, procedure). Fastest to iterate, zero training cost.
    - **RAG** → supplies *knowledge* that is fresh, private, or too large for the weights. Use when the org has internal documents (contracts, policies, tickets, wikis).
    - **Fine-tuning** → bakes in *style, format, or a narrow domain* at scale. Example: LoRA on medical transcripts so the model reliably emits the house SOAP-note format. Also used for latency/cost — distill a big model's behavior into a small one.
    - **Rule of thumb:** if the failure is "it doesn't know," that's RAG. If it's "it knows but says it wrong," that's prompting, then fine-tuning if prompting doesn't scale.
- **Build vs buy** — Managed retrieval (Vertex/Bedrock KB) gets you to a demo in a day; you own the pipeline the moment you need custom chunking, reranking, or ACLs.

---

## 2. Prompt engineering

- **Prompt structure** — System/role → task → constraints → examples (few-shot) → output format, in that order.
    - Put **stable content first** so the prompt-cache prefix hits (large static system prompts + tool defs are the win).
    - Use **XML or markdown delimiters** to separate instructions from data. Retrieved content goes inside a clearly-marked block that the instructions say to treat as *data, not commands*.
- **Techniques** — Zero/few-shot, chain-of-thought, self-consistency (sample n, majority vote), decomposition, negative examples ("do not…"), prefilling the assistant turn to force a format.
    - CoT buys reasoning accuracy and costs latency + tokens; skip it for classification and extraction where it adds nothing.
- **Few-shot selection** — Static examples are fine for format; **dynamic** examples (retrieve the k most similar solved cases) is meaningfully better for domain tasks and is itself a mini-RAG.
- **Structured outputs / tool use** — JSON-schema enforcement, function calling, and **validate + retry on parse failure** with the error fed back into the retry. This is the mechanism by which agents actually act; treat schema violations as a first-class metric.
- **Prompt versioning & evaluation** — Prompts are code: version them, diff them, run the eval suite on every change, and log the version ID with every call so a production regression can be traced to a specific edit.

---

## 3. Context engineering

- **Definition** — Managing *everything* in the window: instructions, retrieved docs, tool results, memory, conversation history. The goal is the right information, in the right position, with minimal noise.
- **Tactics**
    - **Compaction/summarization** of long history; keep a rolling summary plus the last N verbatim turns.
    - **"Lost in the middle"** — models attend best to the start and end of the window. Put the most relevant chunk first or last, never buried at position 7 of 12.
    - **Per-source truncation budgets** — cap tokens per tool result / per document so one verbose source can't crowd out the rest.
    - **Prompt caching** of static prefixes — often the single biggest cost lever in a chat product.
    - **Context isolation** — hand a subagent its own window rather than appending to the orchestrator's; prevents pollution and keeps the main thread cheap.
- **Failure modes to name explicitly:** context *poisoning* (a wrong fact enters and persists), *distraction* (irrelevant retrieved text), *confusion* (too many tools), *clash* (retrieved docs contradict each other and the model picks arbitrarily).

---

## 4. Retrieval / RAG pipeline

### 4.1 Chunking
- **Fixed-size + overlap** — Baseline. Simple, breaks semantics mid-sentence.
- **Recursive character splitting** — Split on paragraph → sentence → word boundaries in order. Good default.
- **Layout/structure-aware** — Split on headings, sections, table boundaries. Best when the parser preserved structure.
- **Semantic chunking** — Split where consecutive-sentence embedding similarity drops. Higher cost, better cohesion.
- **Parent–child (small-to-big)** — Embed small precise chunks, but return the larger parent section to the LLM. Retrieval precision without generation starvation. **This is the highest ROI chunking upgrade in most systems.**
- **Contextual retrieval** — Prepend an LLM-generated one-line "this chunk is from X, about Y" header to each chunk before embedding. Substantially reduces failed retrieval on chunks full of pronouns and bare numbers.
- **Late chunking** — Embed the whole document with a long-context embedder, then pool per chunk, so each chunk vector carries document-level context.
- **Metadata on every chunk** — `doc_id`, `title`, breadcrumb, page, section, dates, tenant, ACL, source URL. Metadata is what makes filtering and citation possible.

### 4.2 Embeddings
- **Model choice** — Match domain and language; check MTEB but validate on *your* golden set. Dimension size trades recall against index size and latency; Matryoshka embeddings let you truncate dimensions on the fly.
- **Symmetric vs asymmetric** — Query and document may need different prefixes/instructions (`query: …` / `passage: …`); getting this wrong quietly costs recall.
- **Fine-tuning embeddings** — Train on (query, positive, hard-negative) triplets from your own logs. Usually a bigger win than swapping the LLM.
- **Versioning** — An embedding-model change requires a full re-index. Store `embedding_model_version` per chunk and support dual-index during migration.

### 4.3 Indexing & vector stores
- **Index types** — Flat (exact, small corpora), **HNSW** (fast, memory-hungry, the default), IVF-PQ / quantized (large corpora, memory-constrained). Tune `ef_search`/`nprobe` for the recall-vs-latency curve.
- **Store choice** — pgvector (already have Postgres, need joins + transactional ACLs), Qdrant/Weaviate/Milvus (scale + filtering), Pinecone/Turbopuffer (managed), Elasticsearch/OpenSearch (you also need BM25 and aggregations).
- **Filtered search** — Pre-filter vs post-filter matters: post-filtering a top-100 by tenant can return zero rows. Prefer stores with native filtered-ANN.

### 4.4 Query understanding
- **Query rewriting** — Resolve pronouns and coreference from chat history ("what about *its* renewal term?" → "what is the renewal term of the Acme MSA?"). Without this, multi-turn RAG falls apart at turn 3.
- **Multi-query / fusion** — Generate 3–5 paraphrases, retrieve for each, merge with Reciprocal Rank Fusion. Cheap recall boost.
- **HyDE** — Generate a hypothetical answer, embed *that*, and search with it. Helps when queries are short and documents are verbose.
- **Decomposition** — Split multi-hop questions into sub-questions, retrieve per sub-question, then synthesize.
- **Self-query / metadata extraction** — Pull filters out of natural language ("contracts signed after 2023" → `signed_date > 2023-01-01`) and apply them as hard filters.
- **Routing** — Decide per query: vector search, SQL/text2SQL, graph, web, or answer directly with no retrieval. Not every question needs the index.

### 4.5 Retrieval & ranking
- **Hybrid search** — BM25 (exact terms, IDs, names, codes) + dense vectors (paraphrase, concept), fused with RRF or weighted scores. Hybrid beats either alone on virtually every real corpus, because part numbers and proper nouns are where dense retrieval fails.
- **Reranking** — A cross-encoder (Cohere Rerank, BGE-reranker) or ColBERT late-interaction model scores query×chunk jointly. Retrieve top-50 cheaply, rerank to top-5. Typically the largest single quality jump in a RAG pipeline.
- **MMR / diversity** — Penalize near-duplicate chunks so top-k covers more ground instead of restating one paragraph.
- **Score thresholding & abstention** — If the best score is below a floor, return "I don't have that" instead of forcing an answer from irrelevant text. Tune the threshold on the golden set.
- **Recursive / agentic retrieval** — Let the model issue follow-up searches after seeing the first results; bounded by a max-hop count.

### 4.6 Advanced RAG patterns
- **Agentic RAG** — Retrieval is a *tool* the agent calls repeatedly with refined queries, rather than a fixed prefetch step.
- **Corrective RAG (CRAG)** — Grade retrieved docs for relevance; if they fail, rewrite the query or fall back to web search before generating.
- **Self-RAG** — Model emits reflection tokens deciding *whether* to retrieve and whether its own output is supported.
- **GraphRAG / knowledge graph** — Extract entities and relations into a graph; answer global questions ("what themes recur across all incidents?") that top-k chunking structurally cannot, plus multi-hop traversal.
- **Multimodal RAG** — Index page images with a vision embedder (ColPali-style) or index VLM-generated descriptions; essential for slide decks, charts, and scanned forms.
- **Structured + unstructured hybrid** — Route numeric/aggregate questions to SQL over a warehouse and narrative questions to vector search; many "RAG failures" are actually analytics questions.

### 4.7 Generation & grounding
- **Grounded prompting** — "Answer only from the context; if it isn't there, say so." Then *verify* rather than trust.
- **Citations** — Require span-level or chunk-level citations in the output schema and validate that every cited ID exists. Unverifiable citations are worse than none.
- **Faithfulness check** — A cheap second pass (NLI model or small LLM) asking "is each claim entailed by the retrieved context?" Catches hallucination before the user sees it.
- **Conflict handling** — When sources disagree, surface both with dates and sources rather than silently picking one.
- **Refusal / no-answer path** — Explicitly designed, explicitly evaluated. Most RAG systems are never measured on whether they correctly *decline*.

---

## 5. Agents

- **Single agent loop** — LLM → decide tool → execute → observe → repeat until done. Key design points: **termination condition**, max iterations, per-step timeout, and error handling that feeds the error text back to the model instead of crashing.
- **Task decomposition ("define down")** — The orchestrator breaks a goal into subtasks with explicit contracts (input/output schema, success criteria) *before* dispatching. Bad decomposition cannot be rescued by good subagents.
- **Orchestrator / subagent** — Orchestrator holds the plan and synthesizes; subagents get narrow scope, their own context window, and their own tools. Prevents context pollution and lets you run cheaper models per subtask.
- **Fan-out / fan-in** — Fan-out runs independent subtasks in parallel (per-document review, per-section extraction). Fan-in aggregates, dedupes, resolves conflicts, and ranks. Must handle **partial failure** (one of eight subagents times out) and **deterministic result ordering**.
- **Loop-back / reflection** — A critic or verifier reviews output and returns it with feedback; bounded retries. Also covers retry-on-tool-error and replanning when a step fails. Bound it — unbounded reflection loops burn budget and often oscillate.
- **Planning styles** — ReAct (interleaved reason/act) for exploratory work, plan-then-execute for predictable pipelines, tree/graph search when steps are reversible and cheap. Most production agents are far more scripted than the literature suggests — prefer a fixed workflow with LLM steps inside it, and reach for a free-form agent only when the path genuinely can't be enumerated.
- **Tool design** — Fewer, well-named, well-described tools beat many overlapping ones. Descriptions are prompt real estate: they carry when to use it, when *not* to, and what the return shape means. Make tools idempotent and side-effect-explicit.
- **Human-in-the-loop** — Approval gates before high-risk or irreversible actions (send, pay, delete, deploy), escalation on low confidence, and a clear resume path after approval. Knowing where you *stop* the agent is a design decision, not an afterthought.
- **State & durability** — Checkpoint agent state after each step so a crashed run resumes instead of restarting; make tool calls idempotent (idempotency keys) so a resumed run doesn't double-send.
- **Sandboxing** — Code-execution and shell tools run in a container with no secrets, network egress policy, CPU/memory/time limits, and a read-only mount by default.
- **Multi-agent communication** — MCP for standardized tool discovery/invocation across agents (write a tool once, reuse everywhere); A2A-style handoff protocols; a shared blackboard/state object when agents must collaborate on one artifact.
- **Cost & step budgets** — Hard caps on tokens, tool calls, wall-clock, and dollars per run, enforced by the runtime, not by asking the model nicely.

---

## 6. Memory

- **Types**
    - **Short-term** — the context window itself.
    - **Working** — an explicit scratchpad/state object the agent reads and writes across steps.
    - **Episodic** — past interactions and their outcomes ("last time the user asked X, we did Y").
    - **Semantic** — durable facts about the user/domain (role, preferences, entities).
    - **Procedural** — learned how-tos, skills, reusable prompts.
    - Long-term typically lands in a vector store (fuzzy recall) plus a KV/relational store (exact facts).
- **Mechanics**
    - **What to write** — an extraction step decides what is memory-worthy; writing everything makes recall useless.
    - **When to retrieve** — relevance + recency, with a cap on injected memories.
    - **How to forget** — TTL/decay, explicit deletion, and user-visible controls (GDPR isn't optional).
    - **Conflict resolution** — when a fact changes, supersede rather than accumulate; store `valid_from` and prefer the newest.
    - **Security** — memory injection is a prompt-injection surface: content written by one user (or one document) and later read as instructions is a real attack path. Treat memory as data.

---

## 7. Evaluation

- **Golden dataset first** — 50–200 real, hand-labeled cases beat any generic benchmark. Grow it from production failures; every incident becomes a test case.
- **Prompt evaluation** — Run variants against the golden set and score. Metrics: exact match, rubric scores, task success rate. Track regressions per prompt version.
- **LLM-as-judge** — A strong model grades outputs against a rubric or pairwise. **Calibrate against human labels** (report agreement rate) and watch for position bias, length bias, and self-preference. Use it for scale, never as sole truth.
- **RAG evals — separate the two failure sources:**
    - *Retrieval:* recall@k, precision@k, MRR, nDCG, hit rate. If the answer wasn't retrieved, generation metrics are noise.
    - *Generation:* faithfulness/groundedness, answer relevance, context precision/recall, citation accuracy (RAGAS-style).
- **Agent evals** — Trajectory-level: did it pick the right tools, in the right order, at reasonable cost? Plus end-to-end task success and step-level checks. Report cost and step count alongside accuracy; a 2% gain for 4× the tool calls is usually a loss.
- **Component vs end-to-end** — Test the parser, chunker, retriever, and generator independently, then together. End-to-end-only evals tell you something is broken but not what.
- **Statistical hygiene** — Temperature 0 for evals, multiple seeds where sampling matters, report confidence intervals, and beware of a 3-point move on a 50-case set meaning nothing.
- **Online eval** — Thumbs up/down, edit-distance between the answer and what the user actually shipped, task-completion and escalation-to-human rates. Sample production traffic into the eval set continuously.

---

## 8. Testing & reliability

- **Testing pyramid**
    - Unit tests for tools and parsers (deterministic).
    - Eval suites for prompts (statistical, temperature 0).
    - Integration tests for agent flows with mocked tools.
    - Red-team tests for prompt injection and jailbreaks.
    - Run evals in CI on every prompt/model/index change; block the merge on regression.
- **Guardrails & safety**
    - Input filters (PII detection and masking, topic/abuse classification).
    - Output filters (PII leakage, toxicity, schema validation, banned-claim checks).
    - **Prompt-injection defense** — treat all retrieved content, tool output, and memory as data, never instructions; delimit it; strip instruction-like patterns; never let retrieved text authorize an action. Keep privileged actions behind a separate, non-model-controlled policy check.
    - **Excessive agency** — the agent's credentials should be the minimum for the task; a read-only agent with a read-only token cannot be talked into a delete.
- **Observability**
    - Trace every LLM call: prompt version, model, tokens in/out, latency, cost, tool calls, retrieval hits, final outcome, and a trace ID linking the whole run.
    - Log retrieved chunk IDs and scores — you cannot debug a bad answer without knowing what it read.
    - Sample production outputs into eval sets. LangSmith / Langfuse / OpenTelemetry GenAI conventions / Arize.
- **Failure handling** — Timeouts on every external call, retries with exponential backoff + jitter, circuit breakers, fallback models across providers, graceful degradation (return retrieved sources with "couldn't summarize" rather than a 500).

---

## 9. Production, cost & latency

- **Cost levers** — Prompt caching (biggest single win for long static prefixes), batching for offline work, model routing, token budgets per request, shortening system prompts, trimming top-k after reranking. **Know your $/task number** and your $/user/month.
- **Latency levers** — Streaming (perceived latency is what users judge), semantic caching of near-identical queries, parallel retrieval + tool calls, speculative/prefetch retrieval while the user types, smaller models on the critical path, and moving reranking to a hosted low-latency endpoint.
- **Serving (self-hosted)** — vLLM/TGI with continuous batching, PagedAttention KV cache, quantization (AWQ/GPTQ/FP8), tensor parallelism, and GPU sizing from concurrent-request × context-length math.
- **Capacity & limits** — Provider rate limits and quotas, per-tenant throttling, queueing with backpressure, and load-shedding rules for spikes.
- **Versioning & rollout** — Prompts, models, indexes, and chunking configs are all versioned artifacts. Ship with A/B or canary, compare on the eval suite *and* online metrics, keep rollback one command away. Pin model versions — an unpinned model upgrade is a silent behavior change.
- **Environments** — Separate dev/staging/prod indexes; never eval against the prod index you're mutating.

---

## 10. Security, privacy & governance

- **Authentication & authorization** — Retrieval must be permission-filtered *at query time* with the caller's identity, not filtered after generation. Multi-tenant isolation belongs in the index (tenant-scoped namespaces or mandatory filters), not in prompt instructions.
- **Data residency & retention** — Where embeddings live, how long logs keep prompts, whether provider training is disabled (zero-retention endpoints), and per-region routing.
- **PII** — Detect and mask before it reaches a third-party model; keep a reversible mapping if the answer must contain real values.
- **Auditability** — Who asked what, what was retrieved, what the model said, what action was taken. Regulated domains need this at rest and queryable.
- **Model/supply-chain hygiene** — Pin model and dependency versions, verify third-party MCP servers before granting tools, and review tool descriptions for injected instructions.
- **Threat checklist** — OWASP LLM Top 10: prompt injection, insecure output handling (rendering model output as HTML/SQL/shell), training-data poisoning, model DoS, supply chain, sensitive-info disclosure, insecure plugin design, excessive agency, overreliance, model theft.

---

## 11. Fine-tuning (when you get there)

- **SFT** — Supervised fine-tuning on (input, ideal output) pairs. Needs a few hundred to a few thousand high-quality examples; data quality dominates quantity.
- **PEFT / LoRA / QLoRA** — Train low-rank adapters instead of full weights: cheap, fast, swappable per domain, and the default for most teams.
- **Preference tuning (DPO/RLHF)** — Train on (chosen, rejected) pairs to shape subjective quality that SFT can't express.
- **Distillation** — Use a strong model to generate training data for a small one; the standard route to cutting cost and latency once behavior is stable.
- **When *not* to** — Knowledge that changes (use RAG), tasks not yet stable (you'll retrain constantly), and anything a better prompt fixes. Fine-tuning also makes evaluation harder, not easier.
- **Ops** — Version datasets alongside model checkpoints, hold out a test split from day one, and re-run the same eval suite you use for prompts.

---

## 12. Data flywheel

The compounding loop that separates systems that improve from systems that plateau:

```
prod traffic → traces + user feedback → triage failures
   → new golden cases → prompt/retrieval/model fix → eval → canary → prod
```
- Capture explicit feedback (thumbs, corrections) and implicit signals (copy, edit, retry, abandon, escalate to human).
- Triage weekly: bucket failures into parse / retrieval / ranking / generation / tooling — the buckets tell you where to spend.
- Mine hard negatives from failed retrievals to fine-tune the embedder or reranker.
- Every fixed bug becomes a permanent regression test.

---

## 13. Reference architecture (end to end)

```
                          ┌──────────── Offline / batch ────────────┐
 sources ──► ingest ──► parse ──► chunk ──► embed ──► index (vector + BM25 + graph)
 (S3, SP,     dedupe,   route     parent/    version     metadata + ACL + tenant
  Confluence, hash,     per page  child,     pinned
  DB, web)    ACLs      validate  contextual

                          ┌──────────── Online / request ───────────┐
 user ──► authn/z ──► guardrails(in) ──► query understanding ──► route
                                          (rewrite, multi-query,   │
                                           decompose, filters)     │
                                                                   ▼
                                       hybrid retrieve (BM25 + vector, ACL-filtered)
                                                    │
                                                    ▼
                                          rerank (cross-encoder) ──► top-k
                                                    │
                                                    ▼
                              context assembly (budget, order, dedupe, cite)
                                                    │
                                                    ▼
                                    LLM / agent loop (tools, HITL gates)
                                                    │
                                                    ▼
                             grounding check ──► guardrails(out) ──► stream to user
                                                    │
                                                    ▼
                              traces, cost, feedback ──► eval set ──► flywheel
```

---

## 14. Metrics cheat sheet

| Stage | Metric | Why it matters |
|---|---|---|
| Parsing | CER/WER, table-cell accuracy, reading-order accuracy | Caps everything downstream |
| Chunking | Chunk-level recall on golden queries | Detects splits that orphan the answer |
| Retrieval | recall@k, precision@k, MRR, nDCG | Was the answer even present? |
| Reranking | nDCG@5 vs baseline, position of first relevant | Isolates the rerank gain |
| Generation | Faithfulness, answer relevance, citation accuracy | Hallucination and grounding |
| Refusal | Correct-abstention rate, false-refusal rate | The most-skipped RAG metric |
| Agent | Task success, trajectory validity, steps/run, cost/run | Accuracy alone hides cost blowups |
| Production | p50/p95 latency, TTFT, $/request, error rate, cache-hit rate | SLO and unit economics |
| Business | Deflection rate, time saved, human-escalation rate | The only numbers leadership funds |

---

## 15. Common failure modes (and the fix)

| Symptom | Usual cause | Fix |
|---|---|---|
| Right doc exists, never retrieved | Chunk lacks context/keywords; dense-only search | Hybrid + contextual retrieval + rewriting |
| Answer cites the wrong section | Chunks too large, no reranking | Parent–child + cross-encoder rerank |
| Confidently wrong on a fresh question | No abstention path | Score threshold + faithfulness check |
| Correct at turn 1, wrong at turn 3 | No query rewriting over history | Coreference-resolving rewrite step |
| Costs 10× the estimate | No caching, top-k too large, agent loops | Prompt cache, rerank-then-trim, step budgets |
| Works in dev, fails in prod | Eval set isn't representative | Sample real traffic into the golden set |
| Table questions always wrong | Table flattened at parse time | Structure-preserving parser, table-aware chunks |
| Agent takes a destructive action | Excessive agency + injection | Least-privilege tokens, HITL gate, data≠instructions |


AWS Bedrock Deployment
----------------------

Fargate = host your application. Lambda = execute small stateless functions/tasks. Bedrock = execute the LLM/managed agent.