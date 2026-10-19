# Embeddings → Model choice (domain, language, dimensions, Matryoshka)

## What it is

The embedding model maps text to vectors such that semantically similar texts land close together. Everything about dense retrieval quality, index size, and query latency follows from this choice. Axes:

| Axis | What to check |
|---|---|
| **Quality on your data** | recall@k / nDCG on your golden queries — not leaderboard averages |
| **Domain** | legal, medical, code, finance, multilingual; general models can underperform on jargon-heavy corpora |
| **Language** | multilingual models (bge-m3, multilingual-e5, Cohere multilingual) vs English-only |
| **Max sequence length** | 512 vs 8k tokens; determines max chunk size and late-chunking feasibility ([late chunking](../2-chunking/late_chunking.md)) |
| **Dimensions** | 384 / 768 / 1024 / 1536 / 3072; memory and latency scale linearly, recall gains flatten |
| **Matryoshka (MRL)** | models trained so the first *d* dimensions are a usable embedding; truncate to 256 for a fast first pass, rescore with full dims |
| **Query/passage asymmetry** | instruction prefixes required? ([symmetric vs asymmetric](symmetric_vs_asymmetric.md)) |
| **Deployment** | API (OpenAI, Cohere, Voyage, Google) vs self-hosted (bge, e5, gte, nomic, jina, Qwen-embed) — cost, residency, latency, throughput |
| **Licence** | commercial use permitted? |
| **Stability** | API models get deprecated → forced re-index ([versioning & migration](embedding_versioning.md)) |

Typical outcome: a strong open model (bge-m3 / gte / e5-large / nomic) self-hosted for volume and residency, or a top API model (Voyage, Cohere, OpenAI text-embedding-3) for convenience; fine-tuned on your logs later ([fine-tuning embeddings on your own logs](fine_tuning_embeddings.md)).

## How to implement

1. **Build the retrieval golden set first** ([retrieval evals](../../../09-evaluation/apply/retrieval_evals.md)): 100–300 queries with relevant chunk IDs, stratified by source/query type.
2. **Shortlist 3–5 candidates** from MTEB (retrieval subset, your language) + domain-specific models + your current one.
3. **Embed the corpus sample** (or full corpus if feasible) with each; index; run the golden queries; compute recall@10/20, nDCG@10, MRR.
4. **Measure the operational side**: embedding throughput (docs/sec on your hardware or API rate), query embedding latency (p95), index size at the chosen dimension.
5. **Test dimension truncation** for MRL models: recall at 256/512/full dims; pick the smallest that holds recall, or use two-stage (truncated ANN → full-dim rescore).
6. **Pick, pin, and record** `embedding_model_version` on every chunk ([metadata on every chunk](../2-chunking/chunk_metadata.md)); write down the max sequence length in the chunker config.
7. **Plan the migration path**: how you'll re-index when you change ([versioning & migration](embedding_versioning.md)).

## Small example

```python
from sentence_transformers import SentenceTransformer
import numpy as np, time

CANDIDATES = {
    "bge-m3": ("BAAI/bge-m3", None, None),                                  # multilingual, 8k, 1024d, MRL-ish
    "e5-large": ("intfloat/multilingual-e5-large", "query: ", "passage: "), # needs prefixes
    "nomic-v1.5": ("nomic-ai/nomic-embed-text-v1.5", "search_query: ", "search_document: "),  # MRL, 8k
}

def evaluate(name, corpus, golden, dims=None):
    path, qp, pp = CANDIDATES[name]
    m = SentenceTransformer(path, trust_remote_code=True)
    t0 = time.time()
    D = m.encode([(pp or "") + c["text"] for c in corpus], normalize_embeddings=True, batch_size=64)
    thr = len(corpus) / (time.time() - t0)
    Q = m.encode([(qp or "") + g["query"] for g in golden], normalize_embeddings=True)
    if dims:                                                                # Matryoshka truncation
        D, Q = D[:, :dims], Q[:, :dims]
        D /= np.linalg.norm(D, axis=1, keepdims=True); Q /= np.linalg.norm(Q, axis=1, keepdims=True)
    S = Q @ D.T
    hits10 = [any(corpus[j]["id"] in g["relevant"] for j in np.argsort(-S[i])[:10]) for i, g in enumerate(golden)]
    return {"model": name, "dims": dims or D.shape[1], "recall@10": float(np.mean(hits10)), "docs_per_sec": thr}

for n in CANDIDATES:
    print(evaluate(n, corpus_sample, golden))
print(evaluate("nomic-v1.5", corpus_sample, golden, dims=256))
```

## Pros (of choosing deliberately)

- **Recall on your corpus** can differ by 10–20 points between models that are neighbours on MTEB.
- **Right dimension** halves or quarters index memory and query latency.
- **Multilingual/domain fit** avoids silent failure on non-English or jargon queries.
- Pinning + versioning makes migrations planned events.

## Cons

- **Evaluation effort** upfront (golden set, multiple embeddings of a sample).
- **Self-hosting** is ops: GPUs, batching, autoscaling.
- **API models**: deprecations, rate limits, cost at ingestion scale, data leaving your environment.
- Small-dimension savings can cost recall on hard queries; must be measured, not assumed.

## When to use (re-evaluate)

- At project start; on major corpus changes (new language, new domain); when retrieval evals show a ceiling; when a new model class arrives; when the current model is deprecated.

## When NOT to use (churn)

- Don't swap embedding models for marginal leaderboard gains — every swap is a full re-index and a migration risk. Fine-tune ([fine-tuning embeddings on your own logs](fine_tuning_embeddings.md)) or improve chunking/hybrid/rerank first; those usually move the needle more.

## Production at scale

**Throughput math.** Corpus of 50M chunks × 300 tokens: on an A10-class GPU a base model does ~1–3k chunks/sec → hours to a day; an API at 1–5k RPM with batching → days and $. Plan ingestion capacity and re-index windows around this number.

**Serving the embedder.** Query embedding is on the request critical path: p95 must be ~10–30 ms. Self-host behind a batched inference server (TEI, vLLM, Triton, ONNX Runtime) with warm replicas; or an API with regional endpoints and a fallback. Cache query embeddings by normalised query text (short TTL) for repeated queries.

**Dimensions and index cost.** HNSW memory ≈ n × (d × 4 bytes + graph overhead). 50M × 1024d ≈ 200 GB + graph; at 256d ≈ 50 GB. Matryoshka two-stage (256d ANN → top-200 → full-dim rescore from a disk-resident store) is a common large-scale pattern ([index types: Flat, HNSW, IVF-PQ](../4-indexing/index_types_flat_hnsw_ivfpq.md)).

**Max sequence length** is a hard constraint on chunk size; over-length inputs are silently truncated by many libraries. Assert in the chunker ([fixed-size + overlap](../2-chunking/fixed_size_overlap.md)).

**Multilingual routing.** Detect query language; if using per-language models, route; if one multilingual model, verify cross-lingual recall (query in FR, doc in EN) on the golden set.

**Residency and privacy.** API embedding sends every chunk out; for regulated tenants use self-hosted or in-region endpoints with zero retention ([data residency & retention](../../../10-production/apply/6-security-privacy-governance/data_residency_retention.md)). Per-tenant model overrides may be needed — that means per-tenant indexes.

**Deprecation tracking.** API embedding models get sunset dates; a forced migration with no plan is a fire drill. Track dates; keep the migration runbook ready ([versioning & migration](embedding_versioning.md)).

**Quantisation** (int8/binary embeddings) cuts memory 4–32×; recall loss is model-dependent; evaluate with rescoring.

## Prod checklist

- [ ] Retrieval golden set built before model selection
- [ ] Candidates evaluated on recall/nDCG + throughput + query latency + index size
- [ ] Dimension (and MRL truncation / two-stage) chosen by measurement
- [ ] Model pinned; `embedding_model_version` on every chunk; max length asserted in chunker
- [ ] Embedding server with batching, warm replicas, p95 SLO; query-embedding cache
- [ ] Ingestion throughput and re-index window planned
- [ ] Residency policy per tenant; deprecation dates tracked
- [ ] Migration runbook ready ([versioning & migration](embedding_versioning.md))

## Metrics to watch

| Metric | Why |
|---|---|
| recall@k / nDCG on golden set per model version | Quality |
| Query embedding p95 latency | Critical path |
| Ingest embeddings/sec and $/1M chunks | Capacity/cost |
| Index memory per 1M chunks | Dimension choice |

## Related

- [Symmetric vs asymmetric](symmetric_vs_asymmetric.md) — prefixes
- [Fine-tuning embeddings](fine_tuning_embeddings.md) — bigger win than swapping
- [Versioning](embedding_versioning.md) — migration
- [Index types](../4-indexing/index_types_flat_hnsw_ivfpq.md) — dimension/quantisation interplay
- [Retrieval evals](../../../09-evaluation/apply/retrieval_evals.md)

## Video takeaway (30 seconds)

"The leaderboard tells you which embedding models are worth trying. Your golden set tells you which one to use — models that are neighbours on MTEB can differ by fifteen points of recall on your contracts. Check domain, language, max input length, and dimensions: a thousand dimensions costs four times the memory of two-fifty-six, and Matryoshka models let you truncate and rescore. Then pin the version, stamp it on every chunk, and don't swap models for marginal gains — every swap is a full re-index. Fine-tune or fix chunking first."
