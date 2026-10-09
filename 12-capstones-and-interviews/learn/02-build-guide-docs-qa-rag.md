# Lesson 12.2 — Build Guide: Docs Q&A RAG System

This guide walks an end-to-end **documentation question-answering** system with retrieval-augmented generation. Follow in order. Swap libraries as you like; keep the stages.

## Outcome

A runnable project that:

1. Ingests a folder of markdown/text docs
2. Chunks and embeds them
3. Retrieves top-k chunks for a question
4. Generates an answer with citations
5. Evaluates on a golden set you create

## Suggested repo layout

```text
docs_qa_rag/
  README.md
  requirements.txt
  data/raw_docs/          # source documents
  data/processed/         # chunks jsonl
  eval/golden_set.jsonl
  src/
    ingest.py
    retrieve.py
    answer.py
    evaluate.py
    app.py                # optional FastAPI
  scripts/run_demo.sh
  reports/eval_report.md
```

## Step 0 — Proposal and corpus rights

- Pick docs you may use (your own notes, public documentation you mirror locally, internal docs with permission).
- 10–100 pages equivalent is enough for a portfolio MVP.
- Write the proposal template from the catalog.

## Step 1 — Environment

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
```

Example dependencies (adjust to your choices):

```text
numpy
pandas
scikit-learn
# choose an embedding path, e.g. sentence-transformers
# choose a vector index, e.g. faiss-cpu or chromadb
# choose an LLM client SDK for your provider
fastapi
uvicorn
pydantic
```

Pin versions in `requirements.txt` once something works.

## Step 2 — Normalize documents

Goals:

- UTF-8 text
- Stable `doc_id` and human title
- Preserve heading structure when possible

```python
# ingest sketch: walk data/raw_docs, write a list of {doc_id, title, text, source_path}
from pathlib import Path

def load_docs(root: str):
    docs = []
    for path in Path(root).rglob("*"):
        if path.suffix.lower() in {".md", ".txt"}:
            docs.append({
                "doc_id": path.stem,
                "title": path.stem.replace("_", " "),
                "text": path.read_text(encoding="utf-8"),
                "source_path": str(path),
            })
    return docs
```

Reject empty files. Log counts.

## Step 3 — Chunking

Start simple; improve later.

**Recommended MVP:** split on headings when present; otherwise overlapping windows of ~400–800 tokens with ~10–20% overlap.

Each chunk record:

```json
{
  "chunk_id": "handbook-refunds-003",
  "doc_id": "handbook-refunds",
  "title": "Refunds",
  "heading": "International orders",
  "text": "...",
  "source_path": "data/raw_docs/handbook-refunds.md"
}
```

Write `data/processed/chunks.jsonl` (one JSON object per line).

### Quality checks

- Spot-read 10 random chunks—do they stand alone enough to answer something?
- Ensure no chunk is huge enough to dominate the context window alone unless intentional.

## Step 4 — Embeddings and index

1. Embed all chunk texts with a single embedding model.
2. Store vectors + chunk metadata in your index.
3. Save model name and dimension in a `index_meta.json`.

```python
# Pseudocode
# vectors = embed([c["text"] for c in chunks])
# index.add(vectors, metadatas=chunks)
```

Rebuild the index in a single script so demos are reproducible (`python -m src.ingest`).

## Step 5 — Retriever

Implement `retrieve(query, k=5) -> list[chunk]`.

MVP scoring: dense cosine similarity.

Stretch: hybrid (keyword + dense) and a simple rerank (cross-encoder or LLM score—careful with cost).

Always apply **permission filters** if docs are not all public—even in a personal project, practice filtering by a `visibility` field.

Log `chunk_id`s returned for every query.

## Step 6 — Answering prompt

Build a prompt that:

- Instructs: use only context; abstain if missing; cite `[chunk_id]`
- Includes numbered context blocks
- Places the user question clearly

```text
You are a documentation assistant.
Use ONLY the context. If insufficient, say you do not know.
Cite chunk_ids like [handbook-refunds-003].

Context:
[1] (chunk_id=...)
...

Question:
...
```

Call the LLM at low temperature (for example 0–0.3). Validate that cited ids exist in the retrieved set—strip invented citations.

```python
def filter_citations(answer: str, allowed: set[str]) -> str:
    # keep only allowed citation ids; implementation left to you
    ...
```

## Step 7 — CLI demo

```bash
python -m src.answer "How many vacation days do new hires get?"
```

Print:

- Answer text
- Cited chunks (title + heading + short quote)
- Retrieval list

## Step 8 — Golden set evaluation

Create `eval/golden_set.jsonl`:

```json
{"id": "q001", "question": "...", "should_abstain": false, "relevant_chunk_ids": ["..."], "notes": "see refunds section"}
{"id": "q002", "question": "...", "should_abstain": true, "relevant_chunk_ids": [], "notes": "not in corpus"}
```

Aim for ≥20 questions: factual, multi-hop-ish, abstain, and adversarial phrasing.

Metrics (MVP):

| Metric | Definition |
|--------|------------|
| Retrieval hit@k | Fraction where ≥1 relevant chunk is in top-k |
| Abstain accuracy | Correct abstain vs not on labeled cases |
| Answer quality | Manual 0/1/2 rubric on a sample |

Automate retrieval hit@k in `evaluate.py`. Do manual grading in a spreadsheet or markdown table.

Write `reports/eval_report.md` with numbers and top failure themes.

## Step 9 — Optional FastAPI wrapper

Expose:

- `GET /health`
- `POST /ask` `{ "question": "..." }` → answer + citations + request_id

Add timeouts and basic input length limits. Do not log raw questions if they may contain secrets—redact or hash.

## Step 10 — README that hires well

Include:

- Problem statement and demo GIF/ASCII
- Architecture diagram
- How to ingest / ask / evaluate
- Eval numbers
- Limitations (hallucination cases you found)
- Next steps

## Acceptance checklist

- [ ] Fresh venv install works
- [ ] Ingest rebuilds index from raw docs
- [ ] Answers cite real chunk ids
- [ ] Golden-set retrieval metric computed
- [ ] Known failures documented
- [ ] No fake paper links or invented benchmarks

## Common stuck points

| Problem | Try |
|---------|-----|
| Bad answers, good docs | Improve chunking; increase k; hybrid search; tighter prompt |
| Good retrieval, bad answers | Lower temperature; require quotes; stronger abstain instruction |
| Everything abstains | Chunks too small/odd; embedding mismatch; query rewriting |
| Costly eval | Cache embeddings; use smaller model for draft; sample golden set in CI |

When finished, package for portfolio using [04-portfolio-and-next-steps.md](04-portfolio-and-next-steps.md).
