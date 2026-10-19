# Stage 10: Production engineering

Decision framework, MLOps, serving, cost and latency, GPUs, monitoring, safety, reliability and the data flywheel.

Summary, rules and context: [AI Engineering Guide, Stage 10](../AI_ENGINEERING_GUIDE.md#stage-10-production-engineering).

## Learn

Curriculum lessons, in reading order.

- [Stage 10 — Production AI Systems: Overview](learn/00-overview.md)
- [Lesson 10.1 — Production Mindset and Decision Framework](learn/01-production-mindset-and-decision-framework.md)
- [Lesson 10.2 — Data Pipelines and MLOps](learn/02-data-pipelines-and-mlops.md)
- [Lesson 10.3 — Serving Architectures](learn/03-serving-architectures.md)
- [Lesson 10.4 — Monitoring, Drift, and Cost](learn/04-monitoring-drift-and-cost.md)
- [Lesson 10.5 — Safety, Privacy, and Human-in-the-Loop](learn/05-safety-privacy-and-hitl.md)
- [Lesson 10.6 — Reliability and Incident Response](learn/06-reliability-and-incident-response.md)
- [Stage 10 — Exercises and “Ready for Stage 11” Checklist](learn/07-exercises-and-checklist.md)

## Apply

Field-manual scenarios: implementation steps, an example, trade-offs, production notes, a checklist and metrics.

**1-serving-cost-latency/**

- [Production → Cost levers](apply/1-serving-cost-latency/cost_levers.md)
- [Production → Latency levers](apply/1-serving-cost-latency/latency_levers.md)
- [Production → Self-hosted serving (vLLM/TGI, quantisation, GPU sizing)](apply/1-serving-cost-latency/self_hosted_serving.md)
- [Production → Capacity & limits](apply/1-serving-cost-latency/capacity_limits.md)
- [Production → Versioning & rollout](apply/1-serving-cost-latency/versioning_rollout.md)
- [Production → Environments](apply/1-serving-cost-latency/environments.md)

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

- [Deployment → AWS: Fargate, Lambda, Bedrock](apply/3-deployment/aws_bedrock_fargate_lambda.md)

**4-observability-reliability/**

- [Testing & reliability → Observability & tracing](apply/4-observability-reliability/observability_tracing.md)
- [Testing & reliability → Failure handling](apply/4-observability-reliability/failure_handling.md)

**5-guardrails/**

- [Testing & reliability → Input guardrails](apply/5-guardrails/input_guardrails.md)
- [Testing & reliability → Output guardrails](apply/5-guardrails/output_guardrails.md)
- [Testing & reliability → Prompt-injection defence](apply/5-guardrails/prompt_injection_defense.md)
- [Testing & reliability → Excessive agency (least privilege)](apply/5-guardrails/excessive_agency.md)

**6-security-privacy-governance/**

- [Security → Authentication & authorization (permission-filtered retrieval)](apply/6-security-privacy-governance/authn_authz_permission_filtered_retrieval.md)
- [Security → Data residency & retention](apply/6-security-privacy-governance/data_residency_retention.md)
- [Security → PII (detect, mask, reversible mapping)](apply/6-security-privacy-governance/pii.md)
- [Security → Auditability](apply/6-security-privacy-governance/auditability.md)
- [Security → Model & supply-chain hygiene](apply/6-security-privacy-governance/supply_chain_hygiene.md)
- [Security → Threat checklist (OWASP LLM Top 10)](apply/6-security-privacy-governance/owasp_llm_top10.md)

**7-metrics-failure-modes/**

- [Metrics cheat sheet](apply/7-metrics-failure-modes/metrics_cheat_sheet.md)
- [Common failure modes (and the fix)](apply/7-metrics-failure-modes/common_failure_modes.md)

**8-data-flywheel/**

- [Data flywheel → The compounding loop](apply/8-data-flywheel/data_flywheel.md)
- [Data flywheel → Feedback capture & triage](apply/8-data-flywheel/feedback_capture_triage.md)

---

Previous: [Stage 9: Evaluation](../09-evaluation/README.md) · Next: [Stage 11: Architecture patterns](../11-architecture-patterns/README.md)
