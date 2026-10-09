# Stage 10: Production engineering

Decision framework, MLOps, serving, cost and latency, GPUs, monitoring, safety, reliability and the data flywheel.

Summary, rules and context: [AI Engineering Guide, section 12](../AI_ENGINEERING_GUIDE.md#12-stage-10-production-engineering).

## Learn

Curriculum lessons, in reading order.

- [Part 5 — Production AI Systems: Overview](learn/00-overview.md)
- [Lesson 01 — Production Mindset and Decision Framework](learn/01-production-mindset-and-decision-framework.md)
- [Lesson 02 — Data Pipelines and MLOps](learn/02-data-pipelines-and-mlops.md)
- [Lesson 03 — Serving Architectures](learn/03-serving-architectures.md)
- [Lesson 04 — Monitoring, Drift, and Cost](learn/04-monitoring-drift-and-cost.md)
- [Lesson 05 — Safety, Privacy, and Human-in-the-Loop](learn/05-safety-privacy-and-hitl.md)
- [Lesson 06 — Reliability and Incident Response](learn/06-reliability-and-incident-response.md)
- [Part 5 — Exercises and “Ready for Part 6” Checklist](learn/07-exercises-and-checklist.md)

## Apply

Field-manual scenarios: implementation steps, an example, trade-offs, production notes, a checklist and metrics.

**1-serving-cost-latency/**

- [9.1 Production → Cost levers](apply/1-serving-cost-latency/09.1_cost_levers.md)
- [9.2 Production → Latency levers](apply/1-serving-cost-latency/09.2_latency_levers.md)
- [9.3 Production → Self-hosted serving (vLLM/TGI, quantisation, GPU sizing)](apply/1-serving-cost-latency/09.3_self_hosted_serving.md)
- [9.4 Production → Capacity & limits](apply/1-serving-cost-latency/09.4_capacity_limits.md)
- [9.5 Production → Versioning & rollout](apply/1-serving-cost-latency/09.5_versioning_rollout.md)
- [9.6 Production → Environments](apply/1-serving-cost-latency/09.6_environments.md)

**2-gpu/**

- [GPU resizing for production AI applications](apply/2-gpu/GPUResizing.md)

**2-gpu/details/**

- [01 — Mental model: what GPU resize actually is](apply/2-gpu/details/01_mental_model.md)
- [02 — Decision tree: whether to resize, and which axis](apply/2-gpu/details/02_decision_tree.md)
- [03 — Four axes: S, H, V, W](apply/2-gpu/details/03_axes_h_v_w_s.md)
- [04 — Three resources and the sizing math](apply/2-gpu/details/04_resources_and_sizing.md)
- [05 — Workload pools: never resize one fleet](apply/2-gpu/details/05_workload_pools.md)
- [06 — Considerations behind every resize](apply/2-gpu/details/06_considerations.md)
- [07 — Cloud and Kubernetes mechanics](apply/2-gpu/details/07_cloud_kubernetes.md)
- [08 — Serving knobs that must change with the GPU](apply/2-gpu/details/08_serving_knobs.md)
- [09 — Trade-off matrices](apply/2-gpu/details/09_trade_offs.md)
- [10 — Production cutover playbook](apply/2-gpu/details/10_cutover_playbook.md)
- [11 — Examples (sizing, axis choice, K8s)](apply/2-gpu/details/11_examples.md)
- [12 — Worked patterns for a prod AI app](apply/2-gpu/details/12_worked_patterns.md)
- [13 — Finer details that bite in production](apply/2-gpu/details/13_gotchas.md)
- [14 — Prod checklist and metrics](apply/2-gpu/details/14_checklist_metrics.md)

**3-deployment/**

- [16.1 Deployment → AWS: Fargate, Lambda, Bedrock](apply/3-deployment/16.1_aws_bedrock_fargate_lambda.md)

**4-observability-reliability/**

- [8.6 Testing & reliability → Observability & tracing](apply/4-observability-reliability/08.6_observability_tracing.md)
- [8.7 Testing & reliability → Failure handling](apply/4-observability-reliability/08.7_failure_handling.md)

**5-guardrails/**

- [8.2 Testing & reliability → Input guardrails](apply/5-guardrails/08.2_input_guardrails.md)
- [8.3 Testing & reliability → Output guardrails](apply/5-guardrails/08.3_output_guardrails.md)
- [8.4 Testing & reliability → Prompt-injection defence](apply/5-guardrails/08.4_prompt_injection_defense.md)
- [8.5 Testing & reliability → Excessive agency (least privilege)](apply/5-guardrails/08.5_excessive_agency.md)

**6-security-privacy-governance/**

- [10.1 Security → Authentication & authorization (permission-filtered retrieval)](apply/6-security-privacy-governance/10.1_authn_authz_permission_filtered_retrieval.md)
- [10.2 Security → Data residency & retention](apply/6-security-privacy-governance/10.2_data_residency_retention.md)
- [10.3 Security → PII (detect, mask, reversible mapping)](apply/6-security-privacy-governance/10.3_pii.md)
- [10.4 Security → Auditability](apply/6-security-privacy-governance/10.4_auditability.md)
- [10.5 Security → Model & supply-chain hygiene](apply/6-security-privacy-governance/10.5_supply_chain_hygiene.md)
- [10.6 Security → Threat checklist (OWASP LLM Top 10)](apply/6-security-privacy-governance/10.6_owasp_llm_top10.md)

**7-metrics-failure-modes/**

- [14.1 Metrics cheat sheet](apply/7-metrics-failure-modes/14.1_metrics_cheat_sheet.md)
- [15.1 Common failure modes (and the fix)](apply/7-metrics-failure-modes/15.1_common_failure_modes.md)

**8-data-flywheel/**

- [12.1 Data flywheel → The compounding loop](apply/8-data-flywheel/12.1_data_flywheel.md)
- [12.2 Data flywheel → Feedback capture & triage](apply/8-data-flywheel/12.2_feedback_capture_triage.md)

---

Previous: [Stage 9: Evaluation](../09-evaluation/README.md) · Next: [Stage 11: Architecture patterns](../11-architecture-patterns/README.md)
