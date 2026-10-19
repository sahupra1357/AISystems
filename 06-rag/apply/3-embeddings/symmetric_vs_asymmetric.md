# Embeddings → Symmetric vs asymmetric (query/passage prefixes)

## What it is

**Symmetric** retrieval: both sides are the same kind of text (sentence ↔ sentence similarity, duplicate detection, example retrieval). One embedding function for both.

**Asymmetric** retrieval: a short query ↔ a long passage. The model must map "how many sick days" close to "Employees receive 10 paid sick days per calendar year" even though the texts look nothing alike. Models trained for this often use **role prefixes** or **instructions**:

| Model family | Query side | Document side |
|---|---|---|
| e5 / multilingual-e5 | `query: ` | `passage: ` |
| bge v1.5 (en) | instruction: `Represent this sentence for searching relevant passages: ` | (none) |
| bge-m3 | (none) | (none) |
| nomic-embed | `search_query: ` | `search_document: ` |
| gte-Qwen / instruction-tuned | `Instruct: <task>\nQuery: ` | (none) |
| OpenAI / Cohere APIs | `input_type="search_query"` | `input_type="search_document"` (Cohere) |
| Voyage | `input_type="query"` | `input_type="document"` |

Omit the prefix and the model still returns vectors — just worse ones, silently. Use the *document* prefix for queries and it's worse again. The mistake survives code review because nothing errors.

## How to implement

1. **Read the model card** and record the exact query/document prefix (or API `input_type`) in the model config next to the model ID.
2. **Two functions, always**: `embed_queries(texts)` and `embed_documents(texts)`; each applies its prefix. Nobody calls the raw model.
3. **Ingest path uses `embed_documents`**; query path (and HyDE pseudo-docs, [hyDE](../5-query-understanding/hyde.md); multi-query paraphrases, [multi-query + Reciprocal Rank Fusion](../5-query-understanding/multi_query_rrf.md)) uses the correct side — HyDE hypothetical *answers* are document-like; paraphrased *questions* are query-like.
4. **Contextual/breadcrumb-enriched text** is still a document ([contextual retrieval](../2-chunking/contextual_retrieval.md)).
5. **Example stores** ([dynamic few-shot selection](../../../05-prompt-and-context-engineering/apply/1-prompt-engineering/dynamic_few_shot_selection.md)) that match query-to-query are symmetric — use the query prefix on both sides, or a symmetric model.
6. **Test**: a unit test that embeds a query with and without the prefix and asserts the golden recall difference is what the model card promises (or at least that the prefixed version is not worse).
7. **Record the prefix version** in `embedding_model_version` (e.g. `e5-large-v2|q:query:|d:passage:`) — changing a prefix changes vectors → re-index.

## Small example

```python
from dataclasses import dataclass
from sentence_transformers import SentenceTransformer

@dataclass(frozen=True)
class EmbeddingConfig:
    model_id: str; query_prefix: str; doc_prefix: str; max_len: int; version: str

CFG = EmbeddingConfig("intfloat/multilingual-e5-large", "query: ", "passage: ", 512, "e5ml-large|q=query:|d=passage:")
_model = SentenceTransformer(CFG.model_id)

def embed_queries(texts: list[str]):
    return _model.encode([CFG.query_prefix + t for t in texts], normalize_embeddings=True)

def embed_documents(texts: list[str]):
    return _model.encode([CFG.doc_prefix + t for t in texts], normalize_embeddings=True, batch_size=64)

# ingest:  vec = embed_documents([chunk.text_indexed])[0]
# query:   qv  = embed_queries([rewritten_query])[0]
# HyDE:    hv  = embed_documents([hypothetical_answer])[0]      # answer text is passage-like
# example store (query↔query): embed_queries on both sides

def test_prefix_effect(golden, corpus_vecs):
    with_p = recall_at_k(embed_queries([g["query"] for g in golden]), corpus_vecs, golden, 10)
    no_p = recall_at_k(_model.encode([g["query"] for g in golden], normalize_embeddings=True), corpus_vecs, golden, 10)
    assert with_p >= no_p, f"prefix hurts?! {with_p} < {no_p}"
    print("recall@10 with prefix", with_p, "without", no_p)
```

## Pros

- **Free recall**: correct prefixes are a several-point gain on many models, at zero cost.
- **Explicit API** (`embed_queries` / `embed_documents`) prevents an entire class of silent bugs.
- Instruction-tuned models let you tailor the task ("retrieve legal clauses relevant to the question") for a further gain.

## Cons

- **Easy to get wrong**, and nothing fails when you do.
- **Prefix conventions differ per model**; migrations must update them (and re-index).
- Instructions add tokens to every query (small).
- Some models are sensitive to the exact prefix string (trailing space, capitalisation).

## When to use

- Every asymmetric retrieval setup (RAG) — check the model card and apply the convention, even if the answer is "no prefix".
- Any time you embed non-query text on the query path (HyDE) or non-document text on the ingest path (example stores) — pick the right side deliberately.

## When NOT to use (asymmetric prefixes)

- Symmetric tasks (dedupe, clustering, example retrieval by similarity of inputs): use a symmetric configuration or the same prefix on both sides.

## Production at scale

**One embedding client, two entry points**, shared by ingestion workers and the query service, so prefixes can't drift between the two code paths. Version the config; a prefix change is a re-index event ([versioning & migration](embedding_versioning.md)).

**Migration trap**: switching models often means switching prefix conventions; forgetting to update the *query* side while the index was built with new document prefixes yields a big, confusing recall drop. Make the prefix part of `embedding_model_version` and assert the query service's version equals the index's.

**Instruction tuning per call site**: instruction-tuned embedders allow different task strings per retrieval use case (policy Q&A vs code search). Treat the instruction as config, evaluated on the golden set, and versioned; each distinct document-side instruction is a separate index.

**Monitoring**: recall on a canary golden subset run hourly in production catches a deploy that dropped the prefix ([online eval](../../../09-evaluation/apply/online_eval.md)).

**Mixed corpora**: documents of very different kinds (tables, code, prose) may benefit from different document instructions; measure before adding complexity.

## Prod checklist

- [ ] Prefix/`input_type` convention recorded in the embedding config with the model ID
- [ ] `embed_queries` / `embed_documents` are the only entry points; shared by ingest and query services
- [ ] HyDE, multi-query, contextual text, example stores use the correct side
- [ ] Prefix encoded into `embedding_model_version`; version equality asserted at query time
- [ ] Unit test proving prefixed recall ≥ unprefixed on the golden set
- [ ] Hourly canary recall check in production

## Metrics to watch

| Metric | Why |
|---|---|
| Golden recall with vs without prefix | Verifies the convention |
| Canary recall in prod | Catches regressions |
| Version mismatch alerts (query vs index) | Migration safety |

## Related

- [Model choice](embedding_model_choice.md), [Versioning](embedding_versioning.md)
- [HyDE](../5-query-understanding/hyde.md) — document-side embedding of a pseudo-answer
- [Dynamic few-shot](../../../05-prompt-and-context-engineering/apply/1-prompt-engineering/dynamic_few_shot_selection.md) — symmetric case
- [Retrieval evals](../../../09-evaluation/apply/retrieval_evals.md)

## Video takeaway (30 seconds)

"A question and the passage that answers it look nothing alike, so many embedding models want to be told which one they're looking at: 'query:' on one side, 'passage:' on the other, or an input-type flag in the API. Leave it off and you still get vectors — just worse ones, and nothing errors. Wrap the model in two functions, embed-queries and embed-documents, make them the only way anyone embeds anything, bake the prefix into the version string, and test that the prefixed recall is actually higher. It's several points of recall for free."
