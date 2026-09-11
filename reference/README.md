# AI Engineer Reference — Scenario-by-Scenario (Production at Scale)

This folder breaks `../AI_System.md` into one file per **scenario** (e.g. *Chunking → Fixed-size + overlap*).
Every file follows the same shape so it can later become a short video:

1. **What it is** — the idea in plain words
2. **How to implement** — concrete steps
3. **Small example** — runnable-sized code
4. **Pros / Cons**
5. **When to use / When NOT to use**
6. **Production at scale** — the part tutorials skip: throughput, cost, multi-tenancy, failure modes, observability
7. **Prod checklist** + **Metrics to watch**
8. **Video takeaway** — the 30-second version

File numbers map to the section numbers in `AI_System.md` (4.1.1 = section 4.1, first bullet).

---

## 0. Ingestion & Parsing — `00_ingestion_parsing/`
| File | Scenario |
|---|---|
| 00.1.1_connectors.md | Source connectors (S3, SharePoint, Confluence, DBs, crawl) |
| 00.1.2_change_detection.md | Change detection: size/mtime → SHA-256 ladder |
| 00.1.3_deduplication.md | Exact + near-duplicate removal (MinHash/SimHash) |
| 00.1.4_document_versioning.md | doc_id + version, soft-delete stale chunks |
| 00.1.5_tenant_acl_capture_at_ingest.md | Tenant / ACL capture at ingest time |
| 00.2.0_per_page_routing.md | Inspect & classify each page, route to the right parser |
| 00.2.1_digital_document_docling.md | Digital PDF → native text-layer parser |
| 00.2.2_scanned_page_ocr.md | Scanned page → OCR |
| 00.2.3_complex_layout_ocr.md | Multi-column / forms → layout model |
| 00.2.4_formula_heavy_mineru.md | Formula-heavy → LaTeX extraction |
| 00.2.5_charts_images_vlm_captioning.md | Charts / diagrams → VLM captioning |
| 00.3.1_reading_order.md | Preserve reading order |
| 00.3.2_tables.md | Tables as HTML/Markdown, never flattened |
| 00.3.3_headings_hierarchy.md | Headings → breadcrumb metadata |
| 00.3.4_page_numbers_bounding_boxes.md | Page numbers + bounding boxes for citations |
| 00.3.5_boilerplate_stripping.md | Strip headers / footers / legal boilerplate |
| 00.4.1_validation_golden_sample.md | Test-time validation (CER/WER, table-cell, layout) |
| 00.4.2_validation_production_heuristics.md | Per-page prod heuristics without ground truth |
| 00.5.1_retry_ladder.md | Bounded parser retry ladder + human queue |

## 1. Foundations — `01_foundations/`
| File | Scenario |
|---|---|
| 01.1_model_selection.md | Choosing models by capability / cost / latency / context |
| 01.2_routing_escalation.md | Cheap-first routing with escalation |
| 01.3_prompting_vs_rag_vs_finetuning.md | Which lever for which problem |
| 01.4_build_vs_buy.md | Managed retrieval vs owning the pipeline |

## 2. Prompt Engineering — `02_prompt_engineering/`
| File | Scenario |
|---|---|
| 02.1_prompt_structure.md | System → task → constraints → examples → format; stable-first |
| 02.2_delimiters_data_vs_instructions.md | XML/markdown delimiters; retrieved text is data |
| 02.3_zero_few_shot.md | Zero-shot and static few-shot |
| 02.4_chain_of_thought.md | CoT: when it pays and when it doesn't |
| 02.5_self_consistency.md | Sample n, majority vote |
| 02.6_decomposition_negative_examples_prefill.md | Decomposition, "do not…", prefilling |
| 02.7_dynamic_few_shot_selection.md | Retrieve the k most similar solved cases |
| 02.8_structured_outputs_tool_use.md | JSON schema, function calling, validate + retry |
| 02.9_prompt_versioning_evaluation.md | Prompts are code: version, diff, eval, log the ID |

## 3. Context Engineering — `03_context_engineering/`
| File | Scenario |
|---|---|
| 03.1_what_is_context_engineering.md | Managing everything in the window |
| 03.2_compaction_summarization.md | Rolling summary + last N turns |
| 03.3_lost_in_the_middle.md | Position matters: first or last, never buried |
| 03.4_per_source_truncation_budgets.md | Token caps per tool result / document |
| 03.5_prompt_caching.md | Cache static prefixes |
| 03.6_context_isolation.md | Subagents get their own window |
| 03.7_context_failure_modes.md | Poisoning, distraction, confusion, clash |

## 4. Retrieval / RAG — `04_rag/`
### 4.1 Chunking — `04.1_chunking/`
| File | Scenario |
|---|---|
| 04.1.1_fixed_size_overlap.md | Fixed-size + overlap (baseline) |
| 04.1.2_recursive_character_splitting.md | Paragraph → sentence → word splitting |
| 04.1.3_layout_structure_aware.md | Split on headings / sections / tables |
| 04.1.4_semantic_chunking.md | Split where embedding similarity drops |
| 04.1.5_parent_child_small_to_big.md | Embed small, return parent (highest ROI) |
| 04.1.6_contextual_retrieval.md | LLM-generated context header per chunk |
| 04.1.7_late_chunking.md | Embed whole doc, pool per chunk |
| 04.1.8_chunk_metadata.md | Metadata on every chunk |
### 4.2 Embeddings — `04.2_embeddings/`
| File | Scenario |
|---|---|
| 04.2.1_embedding_model_choice.md | Domain, language, dimensions, Matryoshka |
| 04.2.2_symmetric_vs_asymmetric.md | query: / passage: prefixes |
| 04.2.3_fine_tuning_embeddings.md | Triplets from your own logs |
| 04.2.4_embedding_versioning.md | Model change = full re-index; dual-index |
### 4.3 Indexing — `04.3_indexing/`
| File | Scenario |
|---|---|
| 04.3.1_index_types_flat_hnsw_ivfpq.md | Flat vs HNSW vs IVF-PQ |
| 04.3.2_vector_store_choice.md | pgvector / Qdrant / Pinecone / OpenSearch |
| 04.3.3_filtered_search.md | Pre-filter vs post-filter |
### 4.4 Query Understanding — `04.4_query_understanding/`
| File | Scenario |
|---|---|
| 04.4.1_query_rewriting.md | Coreference resolution over chat history |
| 04.4.2_multi_query_rrf.md | Paraphrases + Reciprocal Rank Fusion |
| 04.4.3_hyde.md | Hypothetical document embeddings |
| 04.4.4_decomposition.md | Multi-hop → sub-questions |
| 04.4.5_self_query_metadata_filters.md | NL → hard filters |
| 04.4.6_query_routing.md | Vector / SQL / graph / web / no retrieval |
### 4.5 Retrieval & Ranking — `04.5_retrieval_ranking/`
| File | Scenario |
|---|---|
| 04.5.1_hybrid_search.md | BM25 + dense, fused |
| 04.5.2_reranking.md | Cross-encoder / ColBERT rerank |
| 04.5.3_mmr_diversity.md | Maximal Marginal Relevance |
| 04.5.4_score_thresholding_abstention.md | Floor score → "I don't have that" |
| 04.5.5_recursive_agentic_retrieval.md | Bounded follow-up searches |
### 4.6 Advanced RAG — `04.6_advanced_rag/`
| File | Scenario |
|---|---|
| 04.6.1_agentic_rag.md | Retrieval as a tool |
| 04.6.2_corrective_rag.md | Grade docs, rewrite or fall back |
| 04.6.3_self_rag.md | Reflection tokens |
| 04.6.4_graphrag.md | Entities + relations → global questions |
| 04.6.5_multimodal_rag.md | ColPali-style page images, VLM descriptions |
| 04.6.6_structured_unstructured_hybrid.md | SQL for aggregates, vectors for narrative |
### 4.7 Generation & Grounding — `04.7_generation_grounding/`
| File | Scenario |
|---|---|
| 04.7.1_grounded_prompting.md | Answer only from context, then verify |
| 04.7.2_citations.md | Span/chunk citations, validated |
| 04.7.3_faithfulness_check.md | NLI / small-LLM entailment pass |
| 04.7.4_conflict_handling.md | Surface disagreements with dates and sources |
| 04.7.5_refusal_no_answer_path.md | Designed and evaluated abstention |

## 5. Agents — `05_agents/`
| File | Scenario |
|---|---|
| 05.1_single_agent_loop.md | LLM → tool → observe → repeat; termination |
| 05.2_task_decomposition.md | "Define down" with contracts |
| 05.3_orchestrator_subagent.md | Plan holder + narrow subagents |
| 05.4_fan_out_fan_in.md | Parallel subtasks, partial failure, ordering |
| 05.5_loop_back_reflection.md | Critic / verifier with bounded retries |
| 05.6_planning_styles.md | ReAct vs plan-then-execute vs search |
| 05.7_tool_design.md | Fewer, well-described, idempotent tools |
| 05.8_human_in_the_loop.md | Approval gates and resume path |
| 05.9_state_durability.md | Checkpoint per step, idempotency keys |
| 05.10_sandboxing.md | Containerized code execution |
| 05.11_multi_agent_communication_mcp.md | MCP, A2A, blackboard |
| 05.12_cost_step_budgets.md | Runtime-enforced hard caps |

## 6. Memory — `06_memory/`
| File | Scenario |
|---|---|
| 06.1_memory_types.md | Short-term, working, episodic, semantic, procedural |
| 06.2_what_to_write.md | Extraction step decides memory-worthiness |
| 06.3_when_to_retrieve.md | Relevance + recency with a cap |
| 06.4_how_to_forget.md | TTL, decay, deletion, user controls |
| 06.5_conflict_resolution.md | Supersede, valid_from, prefer newest |
| 06.6_memory_security.md | Memory as injection surface |

## 7. Evaluation — `07_evaluation/`
| File | Scenario |
|---|---|
| 07.1_golden_dataset.md | 50–200 hand-labeled real cases |
| 07.2_prompt_evaluation.md | Variants vs golden set, regression tracking |
| 07.3_llm_as_judge.md | Calibrated grading, known biases |
| 07.4_retrieval_evals.md | recall@k, MRR, nDCG |
| 07.5_generation_evals.md | Faithfulness, relevance, citation accuracy |
| 07.6_agent_evals.md | Trajectory + cost |
| 07.7_component_vs_end_to_end.md | Test parts, then the whole |
| 07.8_statistical_hygiene.md | Temp 0, seeds, confidence intervals |
| 07.9_online_eval.md | Thumbs, edit distance, escalation rate |

## 8. Testing & Reliability — `08_testing_reliability/`
| File | Scenario |
|---|---|
| 08.1_testing_pyramid.md | Unit → eval → integration → red-team → CI |
| 08.2_input_guardrails.md | PII masking, topic/abuse classification |
| 08.3_output_guardrails.md | Leakage, toxicity, schema, banned claims |
| 08.4_prompt_injection_defense.md | Data ≠ instructions |
| 08.5_excessive_agency.md | Least-privilege credentials |
| 08.6_observability_tracing.md | Trace every call, log chunk IDs |
| 08.7_failure_handling.md | Timeouts, backoff, breakers, fallbacks |

## 9. Production, Cost & Latency — `09_production_cost_latency/`
| File | Scenario |
|---|---|
| 09.1_cost_levers.md | Caching, batching, routing, budgets, $/task |
| 09.2_latency_levers.md | Streaming, semantic cache, parallelism |
| 09.3_self_hosted_serving.md | vLLM/TGI, quantization, GPU sizing |
| 09.4_capacity_limits.md | Rate limits, throttling, backpressure |
| 09.5_versioning_rollout.md | Canary, A/B, rollback, pinned models |
| 09.6_environments.md | Separate dev/staging/prod indexes |

## 10. Security, Privacy & Governance — `10_security_privacy_governance/`
| File | Scenario |
|---|---|
| 10.1_authn_authz_permission_filtered_retrieval.md | Filter at query time with caller identity |
| 10.2_data_residency_retention.md | Where data lives, how long |
| 10.3_pii.md | Detect, mask, reversible mapping |
| 10.4_auditability.md | Who asked, what was retrieved, what was done |
| 10.5_supply_chain_hygiene.md | Pin versions, vet MCP servers |
| 10.6_owasp_llm_top10.md | Threat checklist |

## 11. Fine-tuning — `11_fine_tuning/`
| File | Scenario |
|---|---|
| 11.1_sft.md | Supervised fine-tuning |
| 11.2_peft_lora_qlora.md | Low-rank adapters |
| 11.3_preference_tuning_dpo_rlhf.md | Chosen / rejected pairs |
| 11.4_distillation.md | Big model teaches small model |
| 11.5_when_not_to_fine_tune.md | The anti-checklist |
| 11.6_fine_tuning_ops.md | Dataset versioning, held-out splits |

## 12–16
| File | Scenario |
|---|---|
| 12_data_flywheel/12.1_data_flywheel.md | The compounding loop |
| 12_data_flywheel/12.2_feedback_capture_triage.md | Explicit/implicit signals, weekly triage |
| 13_reference_architecture/13.1_reference_architecture.md | End-to-end offline + online paths |
| 14_metrics/14.1_metrics_cheat_sheet.md | Metric per stage |
| 15_failure_modes/15.1_common_failure_modes.md | Symptom → cause → fix |
| 16_deployment/16.1_aws_bedrock_fargate_lambda.md | Fargate / Lambda / Bedrock roles |
