# Stage 6: Retrieval-augmented generation

The RAG pipeline in build order: ingest, parse, chunk, embed, index, query, retrieve, rerank, generate and ground.

Summary, rules and context: [AI Engineering Guide, section 8](../AI_ENGINEERING_GUIDE.md#8-stage-6-retrieval-augmented-generation).

## Learn

Curriculum lessons, in reading order.

- [Lesson 6.1 — RAG Deep Dive](learn/01-rag-deep-dive.md)

## Apply

Field-manual scenarios: implementation steps, an example, trade-offs, production notes, a checklist and metrics.

**1-ingestion-parsing/**

- [0.1.1 Source ingestion → Connectors](apply/1-ingestion-parsing/00.1.1_connectors.md)
- [0.1.2 Source ingestion → Change detection (the size/mtime → SHA-256 ladder)](apply/1-ingestion-parsing/00.1.2_change_detection.md)
- [0.1.3 Source ingestion → Deduplication (exact + near-duplicate)](apply/1-ingestion-parsing/00.1.3_deduplication.md)
- [0.1.4 Source ingestion → Document versioning](apply/1-ingestion-parsing/00.1.4_document_versioning.md)
- [0.1.5 Source ingestion → Tenant / ACL capture at ingest](apply/1-ingestion-parsing/00.1.5_tenant_acl_capture_at_ingest.md)
- [0.2.0 OCR / digitization → Per-page classification and routing](apply/1-ingestion-parsing/00.2.0_per_page_routing.md)
- [0.2.1 OCR / digitization → Digital document (native text layer → Docling)](apply/1-ingestion-parsing/00.2.1_digital_document_docling.md)
- [0.2.2 OCR / digitization → Scanned page (PaddleOCR)](apply/1-ingestion-parsing/00.2.2_scanned_page_ocr.md)
- [0.2.3 OCR / digitization → Complex layout (multi-column, forms → layout model)](apply/1-ingestion-parsing/00.2.3_complex_layout_ocr.md)
- [0.2.4 OCR / digitization → Formula-heavy page (MinerU / LaTeX extraction)](apply/1-ingestion-parsing/00.2.4_formula_heavy_mineru.md)
- [0.2.5 OCR / digitization → Charts, images, diagrams (VLM captioning)](apply/1-ingestion-parsing/00.2.5_charts_images_vlm_captioning.md)
- [0.3.1 Structure extraction → Reading order](apply/1-ingestion-parsing/00.3.1_reading_order.md)
- [0.3.2 Structure extraction → Tables (emit as HTML/Markdown, never flattened)](apply/1-ingestion-parsing/00.3.2_tables.md)
- [0.3.3 Structure extraction → Headings hierarchy (breadcrumbs)](apply/1-ingestion-parsing/00.3.3_headings_hierarchy.md)
- [0.3.4 Structure extraction → Page numbers + bounding boxes (verifiable citations)](apply/1-ingestion-parsing/00.3.4_page_numbers_bounding_boxes.md)
- [0.3.5 Structure extraction → Headers / footers / boilerplate stripping](apply/1-ingestion-parsing/00.3.5_boilerplate_stripping.md)
- [0.4.1 Validation → Test-time validation against a golden sample](apply/1-ingestion-parsing/00.4.1_validation_golden_sample.md)
- [0.4.2 Validation → Production per-page heuristics (no ground truth)](apply/1-ingestion-parsing/00.4.2_validation_production_heuristics.md)
- [0.5.1 Retry ladder → Bounded escalation with visible failure](apply/1-ingestion-parsing/00.5.1_retry_ladder.md)

**2-chunking/**

- [4.1.1 Chunking → Fixed-size + overlap](apply/2-chunking/04.1.1_fixed_size_overlap.md)
- [4.1.2 Chunking → Recursive character splitting](apply/2-chunking/04.1.2_recursive_character_splitting.md)
- [4.1.3 Chunking → Layout / structure-aware](apply/2-chunking/04.1.3_layout_structure_aware.md)
- [4.1.4 Chunking → Semantic chunking](apply/2-chunking/04.1.4_semantic_chunking.md)
- [4.1.5 Chunking → Parent–child (small-to-big)](apply/2-chunking/04.1.5_parent_child_small_to_big.md)
- [4.1.6 Chunking → Contextual retrieval (LLM-generated chunk context)](apply/2-chunking/04.1.6_contextual_retrieval.md)
- [4.1.7 Chunking → Late chunking](apply/2-chunking/04.1.7_late_chunking.md)
- [4.1.8 Chunking → Metadata on every chunk](apply/2-chunking/04.1.8_chunk_metadata.md)

**3-embeddings/**

- [4.2.1 Embeddings → Model choice (domain, language, dimensions, Matryoshka)](apply/3-embeddings/04.2.1_embedding_model_choice.md)
- [4.2.2 Embeddings → Symmetric vs asymmetric (query/passage prefixes)](apply/3-embeddings/04.2.2_symmetric_vs_asymmetric.md)
- [4.2.3 Embeddings → Fine-tuning embeddings on your own logs](apply/3-embeddings/04.2.3_fine_tuning_embeddings.md)
- [4.2.4 Embeddings → Versioning & migration (dual-index cutover)](apply/3-embeddings/04.2.4_embedding_versioning.md)

**4-indexing/**

- [4.3.1 Indexing → Index types: Flat, HNSW, IVF-PQ (and tuning the recall/latency curve)](apply/4-indexing/04.3.1_index_types_flat_hnsw_ivfpq.md)
- [4.3.2 Indexing → Vector store choice](apply/4-indexing/04.3.2_vector_store_choice.md)
- [4.3.3 Indexing → Filtered search (pre-filter vs post-filter)](apply/4-indexing/04.3.3_filtered_search.md)

**5-query-understanding/**

- [4.4.1 Query understanding → Query rewriting (coreference over chat history)](apply/5-query-understanding/04.4.1_query_rewriting.md)
- [4.4.2 Query understanding → Multi-query + Reciprocal Rank Fusion](apply/5-query-understanding/04.4.2_multi_query_rrf.md)
- [4.4.3 Query understanding → HyDE (Hypothetical Document Embeddings)](apply/5-query-understanding/04.4.3_hyde.md)
- [4.4.4 Query understanding → Decomposition (multi-hop into sub-questions)](apply/5-query-understanding/04.4.4_decomposition.md)
- [4.4.5 Query understanding → Self-query (natural language → metadata filters)](apply/5-query-understanding/04.4.5_self_query_metadata_filters.md)
- [4.4.6 Query understanding → Routing (vector, SQL, graph, web, or no retrieval)](apply/5-query-understanding/04.4.6_query_routing.md)

**6-retrieval-ranking/**

- [4.5.1 Retrieval & ranking → Hybrid search (BM25 + dense, fused)](apply/6-retrieval-ranking/04.5.1_hybrid_search.md)
- [4.5.2 Retrieval & ranking → Reranking (cross-encoder / late interaction)](apply/6-retrieval-ranking/04.5.2_reranking.md)
- [4.5.3 Retrieval & ranking → MMR / diversity](apply/6-retrieval-ranking/04.5.3_mmr_diversity.md)
- [4.5.4 Retrieval & ranking → Score thresholding & abstention](apply/6-retrieval-ranking/04.5.4_score_thresholding_abstention.md)
- [4.5.5 Retrieval & ranking → Recursive / agentic retrieval (bounded follow-up searches)](apply/6-retrieval-ranking/04.5.5_recursive_agentic_retrieval.md)

**7-advanced-rag/**

- [4.6.1 Advanced RAG → Agentic RAG (retrieval as a tool)](apply/7-advanced-rag/04.6.1_agentic_rag.md)
- [4.6.2 Advanced RAG → Corrective RAG (CRAG): grade, then rewrite or fall back](apply/7-advanced-rag/04.6.2_corrective_rag.md)
- [4.6.3 Advanced RAG → Self-RAG (reflection tokens: retrieve? supported? useful?)](apply/7-advanced-rag/04.6.3_self_rag.md)
- [4.6.4 Advanced RAG → GraphRAG / knowledge graph](apply/7-advanced-rag/04.6.4_graphrag.md)
- [4.6.5 Advanced RAG → Multimodal RAG (page images, ColPali, VLM descriptions)](apply/7-advanced-rag/04.6.5_multimodal_rag.md)
- [4.6.6 Advanced RAG → Structured + unstructured hybrid (SQL for numbers, vectors for narrative)](apply/7-advanced-rag/04.6.6_structured_unstructured_hybrid.md)

**8-generation-grounding/**

- [4.7.1 Generation & grounding → Grounded prompting](apply/8-generation-grounding/04.7.1_grounded_prompting.md)
- [4.7.2 Generation & grounding → Citations (span-level, validated)](apply/8-generation-grounding/04.7.2_citations.md)
- [4.7.3 Generation & grounding → Faithfulness check (is each claim entailed?)](apply/8-generation-grounding/04.7.3_faithfulness_check.md)
- [4.7.4 Generation & grounding → Conflict handling (when sources disagree)](apply/8-generation-grounding/04.7.4_conflict_handling.md)
- [4.7.5 Generation & grounding → Refusal / no-answer path](apply/8-generation-grounding/04.7.5_refusal_no_answer_path.md)

---

Previous: [Stage 5: Prompt and context engineering](../05-prompt-and-context-engineering/README.md) · Next: [Stage 7: Tools, agents and memory](../07-agents-tools-memory/README.md)
