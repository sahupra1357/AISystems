# Lesson 03 — RAG Deep Dive

Prompting steers behavior. It cannot reliably know **your** private documents, yesterday’s policy change, or a customer’s order row. **Retrieval-Augmented Generation (RAG)** supplies relevant evidence at request time, then asks the model to answer **conditioned on that evidence**.

This lesson teaches why RAG exists, how each pipeline stage works, tradeoffs you will actually tune, citation patterns, failure modes, and a debugging playbook. Aim to leave able to design a first Docs Q&A system and explain every box in it.

## Learning goals

- Explain why RAG beats “paste the wiki into the system prompt” and when RAG is the wrong tool
- Choose chunking strategies with precision/recall/context tradeoffs in mind
- Reason about embedding choice, vector indexes, hybrid search, and reranking
- Write grounded answer prompts and citation UX that trust metadata over model imagination
- Debug misses with a systematic playbook (retrieval vs generation vs access control)
- Sketch how RAG fits the production LLM app shape (orchestrator · retriever · LLM · logs)

---

## 1. Why RAG

### The problem RAG solves

Foundation models have three relevant limits:

1. **Parametric knowledge is frozen** at training time (and uneven)
2. **Context windows are finite**—you cannot paste every handbook every time
3. **Hallucination** remains possible when the model is asked for facts it was not given

RAG attacks all three by **fetching** just-in-time evidence from a corpus you control.

### One-picture pipeline

```text
User question
  → (optional) query rewrite / expand
  → embed query (+ maybe keyword query)
  → retrieve top-k chunks from index (dense and/or sparse)
  → (optional) rerank / compress
  → build prompt: instructions + chunks + question
  → LLM generates answer
  → return answer + citations (from retriever metadata)
  → log retrieval ids, scores, prompt version, cost
```

### Why not “always use the biggest context window”?

Long context helps, but:

- Cost and latency scale with tokens
- Noise dilutes attention; models still miss buried facts
- Access control is harder if you dump everything
- Freshness still requires an ingestion pipeline

RAG is **selective memory**, not merely “more tokens.”

### When RAG is the wrong (or incomplete) tool

| Situation | Prefer |
|-----------|--------|
| Stable label taxonomy with tons of history | Classifier / fine-tune (+ maybe RAG for explanations) |
| Need live inventory/price that changes per second | **Tools** / DB queries (Lesson 04), not static index alone |
| Need consistent weird XML voice and few facts | Prompting or fine-tune for style; RAG for facts |
| Tiny static FAQ (10 answers) | Hard-coded intents or simple search may beat RAG complexity |

---

## 2. Corpus design and metadata

Before chunking, decide what a **document** is and what metadata you will store.

Essential metadata fields (typical):

- `doc_id`, `chunk_id`
- `title`, `section_heading`, `source_uri` or path
- `updated_at`, `version`
- `acl` / `tenant_id` / `allowed_roles` (**critical**)
- Optional: language, product area, sensitivity label

**Why metadata matters:** citations, permission filters, freshness debugging, and analytics (“which docs get retrieved but never cited?”).

### Ingestion mindset

```text
source systems → clean/export → chunk → embed → upsert index → version the index
```

Treat re-ingest as a product feature: policies change; your index must too (Part 5 expands pipelines).

---

## 3. Chunking strategies and tradeoffs

Documents must be split into **chunks** that fit retrieval and prompt budgets.

### Strategies

| Approach | How | Strengths | Weaknesses |
|----------|-----|-----------|------------|
| Fixed token/character windows | Every N tokens | Simple, predictable | Splits mid-sentence/idea |
| Overlapping windows | Fixed size + overlap | Bridges boundary facts | Duplicate retrieval noise; more storage |
| Structure-aware | Split on headings, paragraphs, list blocks | Keeps coherent units | Needs decent structure in sources |
| Semantic / embedding-based splits | Cut when topic shifts | Can improve coherence | Extra complexity; tune carefully |
| Small chunks | ~100–300 tokens | Precise retrieval for facts | May lack surrounding context |
| Large chunks | ~800–2000 tokens | More narrative context | Lower precision; prompt bloat |

### How to choose (practical)

- **FAQ / policy facts:** smaller–medium chunks, structure-aware by heading, light overlap (10–20%)
- **Narrative handbooks:** medium chunks by section
- **Tables:** keep table row groups together; consider HTML/CSV-aware splitting; sometimes store both row text and parent table caption
- **Code:** split by function/class when possible—not arbitrary character cuts

### Parent-child / hierarchical patterns

Store small **child** chunks for retrieval precision, but expand to a **parent** section when building the LLM context. This often beats “one size fits all.”

```text
Retrieve child chunk c12 (high similarity)
  → map to parent section S4
  → send S4 (or S4 ± neighbors) to the LLM
```

### Mini practice

For a 50-page employee handbook used for FAQ-style questions, argue for either ~200-token or ~1,000-token chunks. Discuss precision, context continuity, and citation granularity.

---

## 4. Embeddings for retrieval

### What you embed

Usually the chunk text—sometimes prefixed with title/section for disambiguation:

```text
Title: Refund Policy | Section: Digital goods
Digital goods may be refunded within 14 days of purchase...
```

### Choosing an embedding model

Consider:

- **MTEB-style quality** on retrieval tasks (directionally—still validate on *your* corpus)
- Dimension size (storage/latency)
- Multilingual needs
- Hosting (API vs local)
- Whether query and document encoders differ (some asymmetric retrieval models)

**Rule:** the embedding model that ranks your golden questions’ correct chunks highest wins—not the one with the flashiest card.

### Similarity metrics

Cosine similarity and dot product are common (sometimes embeddings are normalized so they coincide). Consistency between **how you index** and **how you query** matters more than the brand name of the metric.

### Index structures (intuition)

| Structure | Intuition | Notes |
|-----------|-----------|-------|
| Flat / exact | Brute force nearest neighbors | Fine for small corpora |
| HNSW / ANN | Approximate neighbors, fast | Default for many vector DBs |
| IVF / clustering | Probe some clusters | Tune recall vs latency |

Approximate search can miss neighbors—measure recall@k on a labeled set when quality matters.

---

## 5. Retrieval: dense, sparse, hybrid, rerank

### Dense retrieval

Embed query → nearest chunk vectors. Captures paraphrases (“vacation days” ≈ “PTO balance”).

### Sparse / keyword (BM25-style)

Lexical match. Excels at IDs, exact product codes, rare proper nouns dense models may blur.

### Hybrid search

Combine dense + sparse scores (weighted sum, reciprocal rank fusion, etc.). **Often stronger than either alone** on real corporate corpora.

```text
score = α * dense_score + (1-α) * sparse_score
# or fuse ranked lists with RRF
```

Tune α on a retrieval golden set—not on vibes.

### Query rewriting

Optional LLM step: expand “that policy?” using chat history into a standalone search query. Helps multi-turn; costs an extra call; can drift—log both.

### Reranking

Retrieve a broad set (for example top 50), then apply a **cross-encoder / reranker** (or LLM rerank carefully) to pick top_k for the prompt (for example 5). Improves precision at the cost of latency/spend.

### How many chunks (k)?

| k too low | k too high |
|-----------|------------|
| Miss evidence | Noise, cost, dilution, conflicting snippets |

Start with k=4–8 after rerank; measure grounded answer quality and retrieval recall.

---

## 6. Building the grounded prompt

### Contract patterns

```text
You are a docs assistant for internal policy Q&A.
Use ONLY the Context below.
If the answer is not in the Context, say: "I do not know based on the documents I was given."
Cite evidence using chunk ids like [c12].
Do not invent policies or URLs.

Context:
[c11] Title: ...
...
[c12] Title: ...
...

Question: ...
```

### Soft vs hard grounding

- **Hard:** only context (safer for policy bots)
- **Soft:** prefer context; if using general knowledge, mark it clearly—riskier; needs careful UX

### Compression and packing

If chunks are long:

- Put the **most relevant** chunks first (some models attend unevenly)
- Truncate low-value boilerplate
- Use extractive compression (“keep sentences that mention refund”) carefully—over-compression drops answers

### Answer vs free chat (recap from Lesson 02)

RAG-answer prompts differ: abstain path, citation contract, lower temperature, no encouragement to freestyle company law.

---

## 7. Citations that users can trust

**Ideal pattern:** the UI shows citations from **retriever metadata** (title, link, chunk id). The model references ids you inserted. You map ids → URLs in code.

**Fragile pattern:** “cite sources” with no ids and hope the model outputs real URLs—it may invent them.

```python
def format_context(chunks):
    blocks = []
    for c in chunks:
        blocks.append(f"[{c['chunk_id']}] {c['title']} | {c['heading']}\n{c['text']}")
    return "\n\n".join(blocks)

def citations_for_ui(chunks):
    return [
        {"id": c["chunk_id"], "title": c["title"], "uri": c["source_uri"]}
        for c in chunks
    ]
```

Optional: ask the model to quote short spans; still verify spans occur in the chunk text with a string check for high-stakes flows.

---

## 8. Access control (non-negotiable)

Filter by permissions **before** generation:

```text
candidate_chunks = retrieve(query)
visible = [c for c in candidate_chunks if user_may_read(user, c["acl"])]
prompt = build_prompt(visible)
```

Never “retrieve everything and ask the model not to reveal secrets.” Prompt rules are not ACLs.

Multi-tenant indexes: prefer hard filters (`tenant_id = …`) in the query itself.

---

## 9. RAG failure modes and mitigations

| Failure | Symptoms | Likely cause | Mitigations |
|---------|----------|--------------|-------------|
| Bad chunking | Partial answers, weird fragments | Splits mid-idea; tables broken | Structure-aware; parent-child; overlap |
| Weak retrieval | Abstains or invents | Poor embeddings; lexical miss; low k | Hybrid; rewrite; rerank; more k; better emb |
| Wrong doc version | Outdated policy | Stale index | Re-ingest; version metadata; TTL |
| Context stuffing | Vague/contradictory answers | Too many chunks | Rerank; compress; lower k |
| Hallucination despite context | Fluent wrong | Weak instructions; conflicts in chunks | Hard grounding; require quotes; verify |
| Citation theater | Pretty fake links | Model-invented URLs | Metadata citations only |
| ACL bug | Cross-tenant leak | Filter after gen or not at all | Filter pre-prompt; tests |
| Query drift | Off-topic retrieve | Bad rewrite / coreference | Log rewrite; gold multi-turn cases |
| Embedding mismatch | Systematic misses | Doc/query model skew; bad normalization | Align models; re-embed after changes |

---

## 10. Debugging playbook

When an answer is wrong, decide **which stage failed**:

```text
1) Was the needed evidence in the corpus at all?
   no  → ingestion / content gap
   yes → continue
2) Did top-k (before ACL) include a relevant chunk?
   no  → retrieval problem (hybrid, emb, rewrite, chunking)
   yes → continue
3) Did ACL filter remove it incorrectly?
   yes → authz metadata bug
   no  → continue
4) Did the prompt include it clearly (not truncated)?
   no  → packing / token budget
   yes → continue
5) Did the model ignore it or conflict with another chunk?
   → generation/prompt/conflict resolution issue
```

Instrument logs:

- query (and rewrite)
- retrieved ids + scores
- ids after ACL
- prompt template version / hash
- model + parameters
- final answer + citation ids used

Without logs, RAG debugging is folklore.

### Retrieval metrics intuition

Build a set of questions with **known good chunk ids**:

| Metric | Meaning |
|--------|---------|
| recall@k | Fraction of questions where ≥1 labeled chunk appears in top-k |
| MRR | How high is the first relevant hit on average |
| nDCG | Ranking quality (if you have graded relevance) |

You can have great recall@k and still bad answers (generation bug)—measure both stages.

---

## 11. Evaluation hooks for RAG (preview of Lesson 06)

Minimum viable:

1. 30–100 golden questions with expected chunk ids and/or answer rubrics
2. retrieval recall@k dashboard
3. answer faithfulness review (supported by context?)
4. regression run on chunker/embedder/prompt changes

Faithfulness ≠ user delight. An answer can be faithful and still unhelpful (wrong granularity). Use rubrics with both dimensions.

---

## 12. Architecture: where RAG sits in the app

```text
┌────────────────────────────────────────────┐
│ Orchestrator (your code)                   │
│  - authn/authz                             │
│  - build retrieval query                   │
│  - call retriever                          │
│  - render prompt template                  │
│  - call LLM                                │
│  - validate output                         │
│  - attach citations from metadata          │
│  - log cost + ids                          │
└────────────────────────────────────────────┘
         │                │              │
         ▼                ▼              ▼
   Vector+BM25 index   LLM API      (optional tools)
   + doc store                          │
         ▲                              │
         │ reindex job                  │
   Source connectors ───────────────────┘
```

Own the orchestrator. Avoid frameworks that hide retrieval ids and prompts until you can log them.

---

## 13. Worked example: internal handbook Q&A

**Corpus:** markdown handbook, ~80 pages, clear headings.

**Chunking:** split on `##` / `###`; aim 200–600 tokens; store heading path as metadata; overlap 0–50 tokens only when sections are tiny.

**Index:** hybrid BM25 + embeddings; filter `tenant=internal`.

**Retrieve:** top 20 hybrid → rerank to 5 → parent expand.

**Prompt:** hard grounding + [chunk_id] citations; temp 0.2; max tokens 400.

**Gold set:** 40 questions: 30 answerable, 10 should abstain; each answerable item lists acceptable chunk ids.

**Failure drill:** “What’s the parental leave for contractors?” if handbook only covers FTE—gold expects abstain, not improvisation.

---

## 14. Anti-patterns

| Anti-pattern | Why | Prefer |
|--------------|-----|--------|
| One giant chunk per PDF | Terrible precision | Structure-aware splits |
| Embedding once, never refreshing | Stale answers | Scheduled re-ingest |
| k=20 into the prompt “to be safe” | Dilution + cost | Rerank to small k |
| Trusting model-written URLs | Hallucinated citations | Metadata mapping |
| No ACL filter | Data leaks | Pre-generation filters |
| Evaluating only answer prose | Misses retrieval rot | recall@k + answer rubrics |
| Silent truncation of context | Hidden misses | Log token counts + overflows |

---

## 15. Practice

1. Write a chunking policy for a mix of PDFs (slides + policies). Name metadata fields.
2. Given recall@5 = 0.55 on gold, list four concrete experiments to try next week.
3. Draft the exact system/user template for hard-grounded answers with ids.
4. Describe how you would prevent tenant A from retrieving tenant B’s chunks—even if the model is “well behaved.”
5. Sketch logs for a single request (fields only).

---



## 16. Chunking workshop: before/after

**Document snippet (policy):**

```text
## Refunds
### Digital goods
Digital goods may be refunded within 14 days of purchase if unused.
### Hardware
Hardware refunds require an RMA within 30 days. Opened bundles are final sale.
```

**Bad chunking (fixed 40-token windows, illustrative):** might split “14 days” away from “Digital goods,” so a query about digital refunds retrieves a fragment that only says “of purchase if unused.”

**Better chunking:** one chunk per `###` subsection, metadata `heading_path=["Refunds","Digital goods"]`, text includes the subsection body plus the parent title in the embedding prefix.

**Before/after retrieval intuition:**

| Query | Bad index risk | Better index |
|-------|----------------|--------------|
| “digital refund window” | Hits hardware fragment | Hits digital subsection |
| “RMA” | May miss if split oddly | Hits hardware subsection |

## 17. Hybrid fusion sketch (code-level intuition)

```python
def rrf(rank_lists, k=60):
    """Reciprocal Rank Fusion over multiple ranked id lists."""
    scores = {}
    for ranking in rank_lists:
        for rank, doc_id in enumerate(ranking, start=1):
            scores[doc_id] = scores.get(doc_id, 0.0) + 1.0 / (k + rank)
    return sorted(scores, key=scores.get, reverse=True)

dense_hits = ["c12", "c4", "c9", "c1"]
sparse_hits = ["c9", "c12", "c22", "c4"]
fused = rrf([dense_hits, sparse_hits])
# fused might start with c12, c9, ...
```

Use fusion **before** ACL filter or carefully ensure filters apply to each candidate list—never fuse in a forbidden doc.

## 18. Conflicting evidence

Chunks can disagree (old vs new policy both retrieved). Options:

1. Prefer higher `updated_at` in packing order and tell the model the dates
2. Deduplicate near-identical chunks
3. Ask the model to surface conflicts explicitly (“Document A says 14 days; Document B (older) says 7”)
4. Fix ingestion so obsolete versions leave the default index

```text
If Context contains conflicting rules, say so and cite both ids.
Prefer the chunk with the newer updated_at when dates are present.
```

## 19. Multimodal and non-text corpora (awareness)

RAG is not only markdown:

- Tables → row-aware chunks + caption metadata
- Slides → per-slide text with slide numbers
- Images/PDFs → OCR / layout parsing quality dominates retrieval quality
- Code repos → symbol-aware chunking; still consider tools for “go to definition”

Garbage parsing → garbage embeddings → confident wrong answers.

## 20. Cost and latency playbook for RAG

| Lever | Quality impact | Cost/latency impact |
|-------|----------------|---------------------|
| Lower k after rerank | Often neutral/positive if rerank is good | Lower |
| Skip rewrite | May hurt multi-turn | Lower |
| Cache embeddings for docs | Neutral | Lower ingest CPU |
| Cache frequent query results | Risk staleness | Lower |
| Smaller embedding model | Maybe lower recall | Lower |
| Smaller chat model for easy intents | Task-dependent | Lower |

Measure end-to-end: **p95 latency**, **$/answer**, **recall@k**, **faithfulness pass rate**.

## 21. From prototype to “boring reliable”

Checklist to graduate a notebook RAG into something teammates can trust:

- [ ] Deterministic doc ids and chunk ids
- [ ] Re-ingest job documented
- [ ] ACL filters tested with negative cases
- [ ] Gold set with retrieval + answer labels
- [ ] Logs for ids/scores/template version
- [ ] Prompt template in version control
- [ ] Known failure list in README
- [ ] Owner for corpus freshness

Part 5 will add stronger ops (monitoring, rollbacks); this checklist is the bridge.


## Key vocabulary

RAG; chunking; overlap; parent-child chunks; embeddings; dense retrieval; BM25 / sparse; hybrid search; reranker; recall@k; grounded generation; citation metadata; ACL filtering; query rewrite; index freshness

## What is next

**[04-tools-agents-and-structured-output.md](../../07-agents-tools-memory/learn/04-tools-agents-and-structured-output.md)** — when the model must take actions or fetch live state: function calling, JSON schemas, validation loops, constrained agents, and least privilege.
