# Stage 6: Retrieval-augmented generation

The RAG pipeline in build order: ingest, parse, chunk, embed, index, query, retrieve, rerank, generate and ground.

Summary, rules and context: [AI Engineering Guide, Stage 6](../AI_ENGINEERING_GUIDE.md#stage-6-retrieval-augmented-generation).

## Learn

Curriculum lessons, in reading order.

- [Lesson 6.1 — RAG Deep Dive](learn/01-rag-deep-dive.md)

## Apply

Field-manual scenarios: implementation steps, an example, trade-offs, production notes, a checklist and metrics.

**1-ingestion-parsing/**

- [Source ingestion → Connectors](apply/1-ingestion-parsing/connectors.md)
- [Source ingestion → Change detection (the size/mtime → SHA-256 ladder)](apply/1-ingestion-parsing/change_detection.md)
- [Source ingestion → Deduplication (exact + near-duplicate)](apply/1-ingestion-parsing/deduplication.md)
- [Source ingestion → Document versioning](apply/1-ingestion-parsing/document_versioning.md)
- [Source ingestion → Tenant / ACL capture at ingest](apply/1-ingestion-parsing/tenant_acl_capture_at_ingest.md)
- [OCR / digitization → Per-page classification and routing](apply/1-ingestion-parsing/per_page_routing.md)
- [OCR / digitization → Digital document (native text layer → Docling)](apply/1-ingestion-parsing/digital_document_docling.md)
- [OCR / digitization → Scanned page (PaddleOCR)](apply/1-ingestion-parsing/scanned_page_ocr.md)
- [OCR / digitization → Complex layout (multi-column, forms → layout model)](apply/1-ingestion-parsing/complex_layout_ocr.md)
- [OCR / digitization → Formula-heavy page (MinerU / LaTeX extraction)](apply/1-ingestion-parsing/formula_heavy_mineru.md)
- [OCR / digitization → Charts, images, diagrams (VLM captioning)](apply/1-ingestion-parsing/charts_images_vlm_captioning.md)
- [Structure extraction → Reading order](apply/1-ingestion-parsing/reading_order.md)
- [Structure extraction → Tables (emit as HTML/Markdown, never flattened)](apply/1-ingestion-parsing/tables.md)
- [Structure extraction → Headings hierarchy (breadcrumbs)](apply/1-ingestion-parsing/headings_hierarchy.md)
- [Structure extraction → Page numbers + bounding boxes (verifiable citations)](apply/1-ingestion-parsing/page_numbers_bounding_boxes.md)
- [Structure extraction → Headers / footers / boilerplate stripping](apply/1-ingestion-parsing/boilerplate_stripping.md)
- [Validation → Test-time validation against a golden sample](apply/1-ingestion-parsing/validation_golden_sample.md)
- [Validation → Production per-page heuristics (no ground truth)](apply/1-ingestion-parsing/validation_production_heuristics.md)
- [Retry ladder → Bounded escalation with visible failure](apply/1-ingestion-parsing/retry_ladder.md)

**2-chunking/**

- [Chunking → Fixed-size + overlap](apply/2-chunking/fixed_size_overlap.md)
- [Chunking → Recursive character splitting](apply/2-chunking/recursive_character_splitting.md)
- [Chunking → Layout / structure-aware](apply/2-chunking/layout_structure_aware.md)
- [Chunking → Semantic chunking](apply/2-chunking/semantic_chunking.md)
- [Chunking → Parent–child (small-to-big)](apply/2-chunking/parent_child_small_to_big.md)
- [Chunking → Contextual retrieval (LLM-generated chunk context)](apply/2-chunking/contextual_retrieval.md)
- [Chunking → Late chunking](apply/2-chunking/late_chunking.md)
- [Chunking → Metadata on every chunk](apply/2-chunking/chunk_metadata.md)

**3-embeddings/**

- [Embeddings → Model choice (domain, language, dimensions, Matryoshka)](apply/3-embeddings/embedding_model_choice.md)
- [Embeddings → Symmetric vs asymmetric (query/passage prefixes)](apply/3-embeddings/symmetric_vs_asymmetric.md)
- [Embeddings → Fine-tuning embeddings on your own logs](apply/3-embeddings/fine_tuning_embeddings.md)
- [Embeddings → Versioning & migration (dual-index cutover)](apply/3-embeddings/embedding_versioning.md)

**4-indexing/**

- [Indexing → Index types: Flat, HNSW, IVF-PQ (and tuning the recall/latency curve)](apply/4-indexing/index_types_flat_hnsw_ivfpq.md)
- [Indexing → Vector store choice](apply/4-indexing/vector_store_choice.md)
- [Indexing → Filtered search (pre-filter vs post-filter)](apply/4-indexing/filtered_search.md)

**5-query-understanding/**

- [Query understanding → Query rewriting (coreference over chat history)](apply/5-query-understanding/query_rewriting.md)
- [Query understanding → Multi-query + Reciprocal Rank Fusion](apply/5-query-understanding/multi_query_rrf.md)
- [Query understanding → HyDE (Hypothetical Document Embeddings)](apply/5-query-understanding/hyde.md)
- [Query understanding → Decomposition (multi-hop into sub-questions)](apply/5-query-understanding/decomposition.md)
- [Query understanding → Self-query (natural language → metadata filters)](apply/5-query-understanding/self_query_metadata_filters.md)
- [Query understanding → Routing (vector, SQL, graph, web, or no retrieval)](apply/5-query-understanding/query_routing.md)

**6-retrieval-ranking/**

- [Retrieval & ranking → Hybrid search (BM25 + dense, fused)](apply/6-retrieval-ranking/hybrid_search.md)
- [Retrieval & ranking → Reranking (cross-encoder / late interaction)](apply/6-retrieval-ranking/reranking.md)
- [Retrieval & ranking → MMR / diversity](apply/6-retrieval-ranking/mmr_diversity.md)
- [Retrieval & ranking → Score thresholding & abstention](apply/6-retrieval-ranking/score_thresholding_abstention.md)
- [Retrieval & ranking → Recursive / agentic retrieval (bounded follow-up searches)](apply/6-retrieval-ranking/recursive_agentic_retrieval.md)

**7-advanced-rag/**

- [Advanced RAG → Agentic RAG (retrieval as a tool)](apply/7-advanced-rag/agentic_rag.md)
- [Advanced RAG → Corrective RAG (CRAG): grade, then rewrite or fall back](apply/7-advanced-rag/corrective_rag.md)
- [Advanced RAG → Self-RAG (reflection tokens: retrieve? supported? useful?)](apply/7-advanced-rag/self_rag.md)
- [Advanced RAG → GraphRAG / knowledge graph](apply/7-advanced-rag/graphrag.md)
- [Advanced RAG → Multimodal RAG (page images, ColPali, VLM descriptions)](apply/7-advanced-rag/multimodal_rag.md)
- [Advanced RAG → Structured + unstructured hybrid (SQL for numbers, vectors for narrative)](apply/7-advanced-rag/structured_unstructured_hybrid.md)

**8-generation-grounding/**

- [Generation & grounding → Grounded prompting](apply/8-generation-grounding/grounded_prompting.md)
- [Generation & grounding → Citations (span-level, validated)](apply/8-generation-grounding/citations.md)
- [Generation & grounding → Faithfulness check (is each claim entailed?)](apply/8-generation-grounding/faithfulness_check.md)
- [Generation & grounding → Conflict handling (when sources disagree)](apply/8-generation-grounding/conflict_handling.md)
- [Generation & grounding → Refusal / no-answer path](apply/8-generation-grounding/refusal_no_answer_path.md)

---

Previous: [Stage 5: Prompt and context engineering](../05-prompt-and-context-engineering/README.md) · Next: [Stage 7: Tools, agents and memory](../07-agents-tools-memory/README.md)
