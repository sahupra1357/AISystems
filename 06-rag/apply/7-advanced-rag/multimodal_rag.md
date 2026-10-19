# Advanced RAG → Multimodal RAG (page images, ColPali, VLM descriptions)

## What it is

Two complementary approaches to making visual content retrievable:

**1. Caption-and-index (the text route, [charts, images, diagrams](../1-ingestion-parsing/charts_images_vlm_captioning.md)).** A VLM describes each figure/page in text; that text is embedded and indexed in your normal pipeline. Cheap, works with a text-only stack, but lossy — the description is a bottleneck, and a question the captioner didn't anticipate won't match.

**2. Visual embedding (ColPali / ColQwen-style).** Skip parsing entirely: render each page as an image, feed it to a vision-language model, and store **multi-vector page embeddings** (one vector per image patch, ColBERT-style late interaction). At query time, the text query's token embeddings are scored against the page's patch embeddings with MaxSim. The page image itself is then passed to a VLM to generate the answer.

```
Text route:   page → parse → OCR/VLM caption → text → embed → search text
Visual route: page → image → ColPali → [patch vectors] → MaxSim against query tokens → page image → VLM answers
```

ColPali's premise is that the entire parsing pipeline — OCR, layout analysis, table structure, figure captioning — is a lossy, expensive detour when the model can look at the page directly. On visual-document benchmarks (ViDoRe) it substantially outperforms text pipelines while being simpler at ingest.

## How to implement

1. **Render pages to images** at consistent resolution ([scanned page](../1-ingestion-parsing/scanned_page_ocr.md)), stored in object storage keyed by `doc_id/version/page`.
2. **Visual route**: embed each page with ColPali/ColQwen → ~1,000 patch vectors of 128 dims per page. Store in a multi-vector-capable index (Vespa, Qdrant multivector, Weaviate, or a custom MaxSim layer over a flat store).
3. **Query**: embed the query text with the same model (token-level), compute MaxSim against candidate pages, take top-k pages.
4. **Two-stage for scale**: pool patch vectors into one dense vector per page for ANN retrieval of ~100 candidates, then MaxSim-rerank those exactly. Full MaxSim over millions of pages is not tractable.
5. **Generate**: pass the retrieved *page images* to a VLM along with the question. The answer is grounded in the pixels, not in a text approximation.
6. **Keep the text route in parallel** for hybrid: BM25 over OCR text catches exact identifiers that visual embeddings miss ([hybrid search](../6-retrieval-ranking/hybrid_search.md)).
7. **Citations** are page-level and natural: the retrieved page image *is* the citation, with bounding-box highlighting optional ([page numbers + bounding boxes](../1-ingestion-parsing/page_numbers_bounding_boxes.md)).

## Small example

```python
import torch
from colpali_engine.models import ColQwen2, ColQwen2Processor

model = ColQwen2.from_pretrained("vidore/colqwen2-v1.0", torch_dtype=torch.bfloat16, device_map="cuda").eval()
proc = ColQwen2Processor.from_pretrained("vidore/colqwen2-v1.0")

def embed_pages(images):                      # list[PIL.Image]
    batch = proc.process_images(images).to(model.device)
    with torch.no_grad():
        return model(**batch)                 # [n_pages, n_patches, dim] multi-vector

def embed_query(text):
    batch = proc.process_queries([text]).to(model.device)
    with torch.no_grad():
        return model(**batch)[0]              # [n_query_tokens, dim]

def maxsim(q_vecs, page_vecs):
    """ColBERT-style late interaction: each query token takes its best-matching patch."""
    return (q_vecs @ page_vecs.transpose(-1, -2)).max(dim=-1).values.sum().item()

def retrieve_visual(question, caller, k=5, ann_candidates=100):
    qv = embed_query(question)
    pooled_q = qv.mean(0)
    cand = vec.search(pooled_q.cpu().numpy(), ann_candidates, build_acl_filter(caller))   # stage 1: cheap ANN
    scored = [(c, maxsim(qv, load_page_vectors(c.id))) for c in cand]                     # stage 2: exact MaxSim
    top = sorted(scored, key=lambda x: -x[1])[:k]
    return [{"page_id": c.id, "score": s, "image_url": page_image_url(c.id),
             "doc_id": c.payload["doc_id"], "page": c.payload["page"]} for c, s in top]

def answer_visual(question, pages):
    content = [{"type": "image", "source": {"type": "base64", "media_type": "image/png",
                                            "data": b64(fetch_image(p["image_url"]))}} for p in pages[:4]]
    content.append({"type": "text", "text": f"Answer the question using only these pages. Cite page numbers.\n\n{question}"})
    return client.messages.create(model="claude-sonnet-5", max_tokens=1000,
                                  messages=[{"role": "user", "content": content}])
```

## Pros

- **No parsing pipeline to break.** Charts, complex layouts, handwritten annotations, stamps, engineering drawings — all retrievable without OCR, layout models, or table extraction.
- **Higher accuracy on visual documents** than text pipelines on standard benchmarks.
- **Radically simpler ingestion**: render → embed. No per-page routing, no retry ladder, no table-structure model.
- **Grounding in the actual page** at generation time; the citation is the page the user can see.
- Robust to parser failures that would silently produce empty text ([production per-page heuristics](../1-ingestion-parsing/validation_production_heuristics.md)).

## Cons

- **Storage explosion**: ~1,000 patch vectors × 128 dims per page. Even quantised, this is 10–100× a single dense vector per page. Pooling and binary quantisation are essential and lossy.
- **Retrieval cost**: MaxSim is far more expensive than cosine; needs two-stage retrieval and specialised index support.
- **Generation cost**: page images are expensive in tokens (a page image can be 1–2k tokens), and you can only fit a handful.
- **Page-level granularity**: you retrieve a whole page, not a paragraph — less precise context, more tokens.
- **Weak on exact identifiers** — visual similarity doesn't do exact string matching; hybrid with BM25 over OCR text is still needed.
- **Immature ecosystem** relative to text retrieval: fewer stores, fewer rerankers, less operational experience.
- No text means no text-based metadata filtering unless you also parse.

## When to use

- **Visually dense corpora**: slide decks, financial reports with charts, scanned forms, engineering drawings, product catalogues, marketing collateral, medical imaging reports.
- When your parsing eval ([test-time validation against a golden sample](../1-ingestion-parsing/validation_golden_sample.md)) shows the text pipeline losing significant content, and the retry ladder is expensive.
- As a **parallel retrieval leg** fused with text retrieval for mixed corpora.

## When NOT to use

- Text-dominant corpora (contracts, policies, wikis) — a text pipeline is cheaper, more precise, and filterable.
- Large corpora with tight storage budgets — model the vector storage before committing.
- When precise sub-page citation is a requirement and page-level isn't acceptable.
- When you need rich metadata filtering that only parsing can supply.

## Production at scale

**Storage is the constraint that decides feasibility.** Model it first: pages × patches × dims × bytes. Mitigations, in order: pooling patch vectors (to ~32–128 per page), binary/int8 quantisation, and two-stage retrieval with full vectors on disk. A million pages at 1,000×128 float32 is ~500 GB before any compression; the same with pooling to 64 vectors and int8 is ~8 GB.

**Two-stage retrieval is mandatory** beyond a few hundred thousand pages: pooled-vector ANN for candidates, exact MaxSim to rerank. This mirrors the compressed-then-rescore pattern in [index types: Flat, HNSW, IVF-PQ](../4-indexing/index_types_flat_hnsw_ivfpq.md).

**Run both routes and fuse.** Visual retrieval for layout/chart questions, BM25 over OCR text for identifiers and exact phrases, dense text retrieval for paraphrase — RRF over all three ([hybrid search](../6-retrieval-ranking/hybrid_search.md)). Track which leg contributes to cited results; on mixed corpora each earns its keep on different query types.

**Generation cost per answer** rises sharply with images. Cap the number of page images (3–5), downscale to the model's optimal resolution, and cache the base64 payloads. Track image tokens as a distinct cost line ([cost levers](../../../10-production/apply/1-serving-cost-latency/cost_levers.md)).

**ACLs**: page images are documents. The same tenant/ACL filters apply at retrieval, and the image-serving endpoint must enforce them independently ([page numbers + bounding boxes](../1-ingestion-parsing/page_numbers_bounding_boxes.md), [authentication & authorization](../../../10-production/apply/6-security-privacy-governance/authn_authz_permission_filtered_retrieval.md)) — serving a page image is serving the whole page.

**Hybrid ingestion**, in practice: still parse for metadata (dates, doc types, page labels, ACLs) even when using visual retrieval, because filtering and citation metadata come from parsing. Visual retrieval replaces the *content* extraction, not the *metadata* extraction.

**Evaluate per query type.** Build the golden set with a visual-question subset (chart reading, layout, forms) and a text-question subset. The visual route wins the first decisively and often loses the second.

**Version everything**: the vision embedder version is an embedding-model version — a change means re-embedding every page ([versioning & migration](../3-embeddings/embedding_versioning.md)), and page vectors are big, so plan the migration window carefully.

## Prod checklist

- [ ] Storage modelled: pages × patches × dims; pooling and quantisation decided before build
- [ ] Two-stage retrieval (pooled ANN → exact MaxSim) for anything beyond ~100k pages
- [ ] Page images stored, versioned, and served through an ACL-enforcing endpoint
- [ ] Fused with BM25/dense text legs; per-leg contribution to cited results tracked
- [ ] Page-image count and resolution capped at generation; image tokens tracked as a cost line
- [ ] Metadata still extracted by parsing for filters and citations
- [ ] Golden set split into visual and text question subsets
- [ ] Vision embedder version treated as an embedding-model version with a migration plan

## Metrics to watch

| Metric | Why |
|---|---|
| Visual-question accuracy vs text pipeline | Justification |
| Storage GB per 100k pages | The main constraint |
| Retrieval p95 (two-stage) | MaxSim cost |
| Image tokens and $ per answer | Generation economics |
| Per-leg contribution to cited results | Whether hybrid legs earn their keep |

## Related

- [VLM captioning](../1-ingestion-parsing/charts_images_vlm_captioning.md) — the cheaper text route
- [Per-page routing](../1-ingestion-parsing/per_page_routing.md) — what visual retrieval lets you skip
- [Index types](../4-indexing/index_types_flat_hnsw_ivfpq.md) — two-stage compressed-then-rescore
- [Hybrid search](../6-retrieval-ranking/hybrid_search.md) — fuse visual with text legs
- [Page images and citations](../1-ingestion-parsing/page_numbers_bounding_boxes.md)

## Video takeaway (30 seconds)

"Your slide deck's key fact is a bar chart, and every step of your parsing pipeline is a lossy attempt to turn that picture into words. ColPali-style retrieval skips it: render the page, embed the image as a thousand patch vectors, and match query tokens against patches directly — then hand the actual page image to a vision model to answer. It beats text pipelines on visual documents and the ingest is just 'render and embed'. The price is storage: a thousand vectors per page instead of one. Pool them, quantise them, use two-stage retrieval, and keep a keyword leg alongside — visual embeddings still can't match an exact part number."
