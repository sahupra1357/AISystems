# Solutioning AI Systems at Scale

A production architecture guide for AI-enabled systems, with emphasis on the decisions that change under high traffic, large knowledge bases, many tenants, long-running workflows, strict reliability targets, and cost pressure.

> **Revision:** 2026-09-07. The previous version is preserved as `AI_Systems.20260907T003545-0400.md.bak`.

---

## 0. What “at scale” means

Scale is not only requests per second. An AI system can hit a scaling limit along several independent dimensions:

| Dimension | Questions to quantify |
|---|---|
| Traffic | Average and peak requests/second, burst shape, concurrent streams, geographic distribution |
| Tokens | Input/output tokens per request, tokens/second, long-context percentage, batch volume |
| Data | Documents, pages, chunks, embedding count, daily change rate, retention, index replicas |
| Tenancy | Tenant count, users per tenant, largest-tenant share, isolation and residency requirements |
| Workflows | Steps/run, tool calls, parallel branches, duration, approval waits, retry frequency |
| Reliability | Availability SLO, p95/p99 latency, recovery time, recovery point, degraded-mode expectations |
| Safety | Data sensitivity, action impact, abuse rate, approval requirements, audit depth |
| Economics | Cost/request, cost/successful task, monthly ceiling, human-review cost, provider commitments |

The first architecture deliverable should be a workload model with current, launch, and 12–18 month estimates. Without it, “scalable” is an untestable claim.

### Critical principles

1. **Keep the online path short and bounded.** Unbounded agent loops, retrieval fan-out, and retries make latency and cost impossible to operate.
2. **Separate synchronous interaction from asynchronous work.** Queue long ingestion, batch inference, deep analysis, and approval-paused workflows.
3. **Keep state outside the model.** Context windows are request inputs, not durable stores or workflow engines.
4. **Put deterministic controls around probabilistic behavior.** Authentication, authorization, policy, limits, schemas, and state transitions belong in code.
5. **Design every side effect for retries.** Idempotency and postcondition checks prevent duplicate messages, charges, deployments, and records.
6. **Filter permissions during retrieval.** Post-filtering or asking the model to respect ACLs is both a leak risk and a scale problem.
7. **Isolate workloads and tenants.** Interactive traffic must not compete with ingestion, evaluation, or one noisy customer.
8. **Version every behavioral dependency.** Model, prompt, tool schema, retrieval config, index, policy, and feature flags must be traceable per response.
9. **Make overload behavior explicit.** Backpressure, admission control, degradation, and load shedding are part of the design.
10. **Optimize cost per successful task.** A cheap request that fails and is retried is more expensive than a correctly routed request.

---

## 1. Solutioning inputs and non-negotiable decisions

Before selecting services or frameworks, capture a one-page solution brief.

### 1.1 Workload and outcome

- Primary user journeys and whether each is search, generation, extraction, classification, conversation, analytics, or action-taking.
- Quality target and existing baseline for each journey.
- Peak traffic, token distribution, data volume/change rate, and expected growth.
- Latency budget split across gateway, retrieval, reranking, model, tools, validation, and network.
- Availability, freshness, RPO/RTO, and maximum acceptable queue age.
- Cost ceiling per request, successful task, tenant, and month.

### 1.2 Risk and data boundary

- Data classification, tenant isolation, residency, retention, and provider-processing restrictions.
- Whether the system advises, drafts, or executes; which actions require approval.
- Required evidence: source citations, tool receipts, audit history, reproducible traces.
- Failure posture: abstain, degrade to search, queue for later, switch model/provider, or escalate to a person.

### 1.3 Architectural choices to make explicitly

| Decision | Typical options | Scale consequence |
|---|---|---|
| Model hosting | Managed API, dedicated endpoint, self-hosted | Operational burden, quota control, latency, unit cost, data boundary |
| Interaction | Synchronous, streaming, asynchronous job | User experience, timeout risk, queueing, cancellation, recovery |
| Orchestration | Fixed workflow, bounded agent, free-form agent | Predictability, evaluability, latency, failure surface |
| Knowledge | Prompt-only, RAG, SQL, graph, fine-tuning | Freshness, accuracy, operating cost, update path |
| Tenant isolation | Shared rows, namespaces/partitions, dedicated deployment | Cost vs blast radius, noisy neighbors, residency |
| Retrieval store | Search engine, vector database, relational extension, managed KB | Filtering, hybrid search, operations, migration, cost |
| Deployment | Single region, active-passive, multi-region | Availability, consistency, residency, spend |

Document the choice, assumption, threshold that would change it, and rollback/migration path.

---

## 2. Scalable reference architecture

```text
Clients
   │
   ▼
Edge / API gateway
authn • rate/size limits • request ID • regional routing
   │
   ▼
AI application / orchestrator  ─────────────► Durable workflow + queues
stateless request handling                    long jobs • retries • approvals
   │              │                                      │
   │              ├────────► Tool gateway ────────────────┤
   │              │          authz • schemas • idempotency│
   │              │                                      │
   │              └────────► Retrieval service            │
   │                         query • ACL filter • rerank   │
   ▼                                                        ▼
Model gateway ───────────────────────────────────────► Worker pools
routing • quotas • cache • timeout • fallback          workload-isolated
   │
   ▼
Managed or self-hosted models

Data plane                         Control plane
──────────                         ─────────────
source connectors                  prompts/model routes/policies
parse/chunk/embed pipeline         config/feature flags/experiments
object + relational storage       secrets/IAM/deployment metadata
vector + lexical indexes          approvals/kill switches

Cross-cutting: traces • metrics • cost • audit • evaluations • alerts
```

### 2.1 Component boundaries

- **API/edge** owns authentication, admission control, payload limits, regional routing, and request identity.
- **Application/orchestrator** owns the task contract, context assembly, workflow state transitions, and response semantics.
- **Model gateway** owns provider abstraction, model routing, quotas, retry/fallback policy, usage normalization, and model-call telemetry.
- **Retrieval service** owns query transformation, tenant/ACL filters, hybrid retrieval, reranking, and citation-ready provenance.
- **Tool gateway** owns per-user authorization, schema validation, secrets brokering, idempotency, and action receipts.
- **Workflow runtime and queues** own durable execution, timers, retries, dead letters, cancellation, and resume-after-approval.
- **Control plane** owns versioned configuration and promotion; it should not sit on the per-token data path.

Stateless compute should be horizontally scalable. Durable state—conversation metadata, jobs, approvals, tool status, artifacts, usage, and checkpoints—must live in databases or object storage.

---

## 3. Capacity planning and performance budgets

### 3.1 Calculate the limiting resources

- **Concurrent requests ≈ arrival rate × average service time.** At 40 requests/second and 5 seconds average service time, plan for roughly 200 in-flight requests before burst headroom.
- **Token demand = requests/second × average tokens/request.** Separate input, cached input, and output tokens because providers price and rate-limit them differently.
- **Workflow amplification = user requests × average model/tool calls per request.** A 5-step agent turns 100 user RPS into as many as 500 downstream calls before retries.
- **Vector footprint ≈ vectors × dimensions × bytes/value**, then add metadata, graph/index overhead, replicas, and temporary space for migration. The index may require multiples of raw vector size.
- **Ingestion rate must exceed change rate.** Track documents/minute and source-to-searchable lag, including parsing, embedding quotas, and index commit time.

Use measured distributions rather than averages alone. Long prompts, large tenants, and tool timeouts dominate p99 behavior.

### 3.2 Allocate latency budgets

Example for an interactive request:

| Stage | Budget concern |
|---|---|
| Edge/auth | Stable and small; avoid remote authorization fan-out |
| Query/retrieval | Parallelize independent searches; bound top-k and query expansion |
| Reranking | Bound candidate count; use smaller model or skip on easy queries |
| Model | Track time-to-first-token separately from total generation time |
| Tools | Apply per-tool deadline shorter than the request deadline |
| Validation | Prefer deterministic streaming-safe checks; bound model-based verification |

Propagate one end-to-end deadline. Each component should consume the remaining budget rather than starting its own full timeout.

### 3.3 Load and stress testing

Test complete journeys with realistic token lengths, retrieval filters, streaming connections, tool latency, and tenant skew. Include:

- Cold starts, cache misses, and autoscaling delay.
- Provider quota exhaustion and rate-limit storms.
- One tenant generating a disproportionate load.
- Large-context and high-output requests at the same time.
- Ingestion/backfill while interactive traffic is at peak.
- Queue growth, retries, dead letters, and recovery after dependency restoration.

Define the saturation point and the degradation sequence before launch.

### 3.4 Autoscaling signals

CPU alone is usually a poor signal for AI workloads. Scale each pool on the resource that actually queues:

- API/streaming services: in-flight requests, active streams, event-loop/connection saturation, and p95 latency.
- Workflow workers: queue age, runnable jobs, service time, and oldest-deadline risk—not depth alone.
- Model servers: queued tokens, active sequences, batch utilization, KV-cache pressure, and time-to-first-token.
- Ingestion workers: source lag, pages/chunks awaiting each stage, embedding quota, and index write throughput.

Keep warm headroom for bursty interactive traffic. Scale down slowly enough to avoid cancelling streams, discarding useful model cache, or oscillating between cold starts and overload.

---

## 4. Model and inference layer at scale

### 4.1 Use a model gateway

Applications should call internal capabilities such as `fast-extract`, `reasoning`, `vision`, or `embedding`, not provider-specific model IDs. The gateway maps capabilities to pinned versions by environment and enforces:

- Authentication, per-tenant quotas, token/output limits, and concurrency limits.
- Deadlines, retry rules, circuit breakers, and fallbacks by error class.
- Request normalization, structured-output validation, usage/cost accounting, and redaction.
- Model/prompt/config version attached to every trace.
- Routing experiments, canaries, and emergency model disablement.

Do not blindly retry safety refusals, invalid requests, or deterministic schema failures on another provider. Fallbacks are for defined availability/capacity failures and must be evaluated for behavioral compatibility.

### 4.2 Routing hierarchy

1. Route by required capability: text, vision, context length, tools, structured output, language.
2. Enforce data-region and provider eligibility.
3. Choose the smallest model that meets the measured quality threshold.
4. Escalate only on explicit signals such as task class, retrieval uncertainty, validation failure, or bounded self-check.
5. Record route and escalation rate. A router that escalates most traffic adds complexity without saving cost.

### 4.3 Managed vs self-hosted inference

| Managed API | Self-hosted/dedicated inference |
|---|---|
| Faster delivery, broad model access, elastic capacity | Greater control over weights, locality, batching, and steady-state cost |
| Provider quotas and variable availability | GPU capacity planning and on-call ownership |
| Less infrastructure work | Must manage serving engine, quantization, upgrades, fragmentation, autoscaling |

For self-hosting, size weights, KV cache, runtime overhead, sequence length, batch/concurrency, and replica headroom together. Benchmark the actual prompt/output distribution; headline tokens/second is not an application capacity number.

### 4.4 Inference efficiency

- Cache stable prompt prefixes when supported; measure cached-token hit rate.
- Use semantic response caching only for low-risk, permission-safe, freshness-tolerant requests. Include tenant, policy, corpus/index version, and model/prompt version in the cache key.
- Coalesce identical in-flight work and add expiry jitter to prevent cache stampedes after a popular entry expires.
- Batch embeddings and offline generation; do not delay interactive work to improve batch utilization.
- Cap output tokens and stop generation as soon as the schema/task is complete.
- Stream for perceived latency, but define how mid-stream validation, cancellation, and errors work.
- For self-hosting, evaluate continuous batching, prefix caching, quantization, tensor parallelism, and speculative decoding against quality and tail latency.

---

## 5. Data ingestion and index lifecycle at scale

### 5.1 Scalable ingestion pipeline

```text
discover/change event
   → fetch raw object once
   → hash + classify + ACL/tenant capture
   → parse/OCR/layout extraction
   → quality validation/quarantine
   → chunk + metadata
   → batch embed
   → write lexical/vector indexes
   → publish searchable version
```

Critical properties:

- **Incremental:** use source version/etag plus a content hash; do not reprocess unchanged data.
- **Idempotent:** the same event can run twice without duplicate chunks or inconsistent indexes.
- **Partitionable:** shard work by tenant/source/document while rate-limiting large producers.
- **Resumable:** checkpoint stages and retry only failed items; use dead-letter queues with replay tools.
- **Observable:** record stage latency, failure reason, retry count, and source-to-index freshness.
- **Reproducible:** store parser, chunker, embedding model, schema, and index versions.
- **Secure:** capture ACLs before indexing and quarantine content that fails policy or malware checks.

Route difficult pages to expensive OCR/vision only after cheap classification and validation. Parser quality still caps retrieval quality, but every escalation needs a cost and throughput budget.

### 5.2 Index lifecycle

- Write a new version before switching reads; never mutate the only serving index during a full rebuild.
- Support aliases or version routing for blue/green indexes, validation, rollback, and dual-read comparison.
- Propagate source changes, permission changes, and deletions to all derived chunks, vectors, caches, replicas, and required logs.
- Separate tenant namespace/partition strategy from physical sharding so large tenants can be moved without changing the application contract.
- Plan index compaction, tombstone cleanup, replica rebuild, backup/restore, and regional recovery.
- Benchmark filtered retrieval with realistic ACL selectivity; post-filtering approximate-nearest-neighbor results can return too few or unauthorized candidates.

### 5.3 Storage sizing and partitioning

Partition using tenant, region, time, or corpus only when it matches access patterns. Too many small partitions waste resources; one shared global partition increases blast radius and noisy-neighbor risk. Track:

- Vectors and metadata bytes per document and tenant.
- Index overhead, replica factor, growth, and rebuild duration.
- Read/write QPS and filtered-query tail latency.
- Largest tenant as a percentage of total storage and traffic.
- Backup size, restore throughput, and re-embedding time.

---

## 6. Retrieval and context at scale

### 6.1 Retrieval pipeline

1. Normalize the query and resolve conversation references.
2. Extract tenant, user identity, hard metadata filters, and time scope.
3. Route to vector search, lexical search, SQL/analytics, graph, or a combination.
4. Run bounded query expansion only for queries shown to need it.
5. Retrieve with ACL filters pushed into the store.
6. Fuse lexical and vector results, deduplicate, and rerank a bounded candidate set.
7. Apply relevance/no-answer thresholds calibrated on real queries.
8. Return provenance-rich chunks to a deterministic context builder.

Hybrid retrieval is usually the production default: lexical search handles names, IDs, error codes, and exact terms; dense retrieval handles paraphrase and concept similarity.

### 6.2 Chunking that survives scale

- Preserve section, table, page, source, time, tenant, and ACL metadata on every chunk.
- Start with structure-aware chunking; use parent-child retrieval when small search units need larger generation context.
- Dedupe exact and near-duplicate content so top-k is not consumed by copies.
- Version chunking and embedding configurations. Embedding changes normally require a controlled re-index.
- Evaluate chunking by answer-bearing recall and end-to-end outcomes, not aesthetic chunk size.

### 6.3 Context builder

Make context assembly a standalone, tested component. It should:

- Reserve output capacity before allocating input tokens.
- Budget instructions, history, retrieval, memory, and tool results separately.
- Attach provenance, timestamp, trust level, and sensitivity to each item.
- Dedupe, prioritize, order, and truncate deterministically.
- Keep trusted policy separate from untrusted user, document, memory, and tool content.
- Emit telemetry for included and dropped items.

More context is not automatically better. It increases latency/cost and can reduce answer quality through distraction or contradiction.

---

## 7. Agent and tool orchestration at scale

### 7.1 Prefer workflows until the path is genuinely unknown

- Use deterministic workflows for known business processes, with model steps for interpretation or generation.
- Use a bounded agent when tool choice or step order cannot be enumerated in advance.
- Set maximum steps, parallel branches, tool calls, tokens, dollars, and wall-clock time in the runtime.
- Keep the orchestrator responsible for state and termination; do not ask the model to enforce its own budget.

### 7.2 Durable execution

Any workflow that may exceed an HTTP timeout, wait for approval, retry after outage, or run in parallel needs durable orchestration:

- Persist inputs, current state, step outputs, attempt count, deadlines, and model/tool versions.
- Resume from the last committed step after a crash.
- Use per-step idempotency keys and store action status before retrying.
- Model partial completion explicitly; define compensation when atomic rollback is impossible.
- Support cancellation, timeouts, dead letters, manual repair, and replay.
- Use optimistic versioning, locks, or compare-and-swap when agents mutate shared resources.

### 7.3 Tool gateway

Tools should have narrow, versioned schemas and explicit read/write risk. The gateway must:

- Authorize the authenticated user against the exact resource and action.
- Broker short-lived credentials without exposing secrets to the model.
- Validate arguments, destinations, size, and policy independently of model text.
- Separate plan/preview from commit for consequential actions.
- Return structured errors, affected resources, receipts, and postcondition evidence.
- Default-deny network and filesystem scope; isolate code execution.

Run fan-out branches only when independent. Bound concurrency and define deterministic fan-in, duplicate handling, conflict resolution, and partial-failure behavior.

---

## 8. Multi-tenancy, security, and isolation

### 8.1 Isolation model

| Model | Use when | Main tradeoff |
|---|---|---|
| Shared compute and indexes with mandatory tenant filters | Many small tenants, common region/policy | Lowest cost; strongest need for filter enforcement and noisy-neighbor control |
| Shared compute with tenant namespaces/partitions | Medium tenants or easier migration/deletion | More isolation with moderate operational overhead |
| Dedicated index or deployment | Large/regulatory tenants, custom keys/region/SLO | Highest isolation and cost; more fleet management |

Design for tenant promotion: a large tenant should move from shared to dedicated infrastructure without changing public APIs or document IDs.

### 8.2 Controls that are critical at scale

- Authenticate at the edge and propagate a signed identity/tenant context; do not accept tenant IDs generated by the model.
- Enforce authorization in retrieval and tools, not after generation.
- Apply per-tenant request, token, storage, ingestion, workflow, and tool quotas.
- Use workload identity, short-lived credentials, managed secrets, encryption, and key separation where required.
- Redact sensitive data before it enters third-party models, logs, traces, evaluation sets, or support tools.
- Default-deny egress; block internal/metadata address ranges and protect URL-fetching tools against SSRF.
- Encode rendered output and never pass model text directly into shell, SQL, template, or policy execution.
- Provide kill switches by model, tool, tenant, workflow, and region.
- Audit who requested the task, what evidence was read, which model/tool versions ran, and what action changed.

Keep legal/governance documentation proportional to system risk, but data retention, deletion, residency, provider terms, and action accountability must be decided before production.

---

## 9. Reliability and failure engineering

### 9.1 Failure controls

- **Deadlines:** every remote call has a deadline derived from the parent request/workflow deadline.
- **Retries:** exponential backoff with jitter, retry budget, idempotency, and error classification.
- **Circuit breakers:** stop adding traffic to an unhealthy provider or tool.
- **Bulkheads:** isolate tenants, regions, providers, interactive/batch pools, and critical/non-critical tools.
- **Backpressure:** bound queues and concurrency; reject or defer work before memory and connection pools collapse.
- **Load shedding:** drop optional query expansion, verification, or low-priority batch work before core journeys.
- **Graceful degradation:** return search results without synthesis, use a smaller model, queue the task, or provide partial artifacts when safe.
- **Redundancy:** test fallbacks; provider compatibility cannot be assumed from matching API shapes.

### 9.2 SLOs and recovery

Define SLOs per journey, not one global number:

| Area | Example measure |
|---|---|
| Availability | Successful valid responses / eligible requests |
| Latency | p50/p95/p99 time-to-first-token and completion time |
| Quality | Task success or grounded-answer rate on sampled traffic |
| Freshness | Source update to searchable index |
| Safety | Unauthorized disclosure/action and policy-escape rate |
| Workflow | Completion before deadline; queue age; dead-letter rate |

Set RPO/RTO for relational state, artifacts, configuration, and indexes. Run restore and index-rebuild drills; “re-creatable” is not useful if recreation exceeds the recovery objective.

### 9.3 Failure tests

Inject provider timeouts, 429s, malformed structured output, partial tool success, stale ACLs, vector-store outage, queue backlog, expired credentials, region loss, and cancellation during side effects. Verify the user-visible result, state consistency, alerts, and recovery—not merely that an exception was logged.

---

## 10. Observability and evaluation

### 10.1 Trace one request end to end

Every trace should connect:

- User/tenant and request ID, with sensitive content redacted.
- Application, prompt, model route, policy, tool schema, retrieval, and index versions.
- Model latency, tokens, cache status, finish reason, retry/fallback, and cost.
- Query rewrites, filters, retrieved chunk IDs/scores, reranked order, and context inclusion.
- Agent steps, tool arguments after redaction, results, approvals, idempotency keys, and state transitions.
- Final status, validation/safety outcome, user feedback, and business outcome when available.

Control trace volume with sampling, but retain full traces for errors, safety events, expensive outliers, and canary traffic. Observability pipelines need their own access controls, retention, and cost limits.

### 10.2 Evaluation layers

| Layer | Critical metrics |
|---|---|
| Parsing | Text/table/layout accuracy, empty-page rate |
| Retrieval | Recall@k, nDCG/MRR, ACL correctness, no-answer accuracy |
| Generation | Task correctness, groundedness, citation accuracy, schema validity |
| Agent | Task success, valid trajectory, action correctness, steps/cost/time |
| System | End-to-end success, p95/p99 latency, availability, cost/success |
| Business | Completion, time saved, escalation, correction, adoption |

Use real, versioned test cases split by tenant shape, language, task, difficulty, risk, modality, and failure mode. Include adversarial, no-answer, long-context, tool-failure, and permission-boundary cases.

### 10.3 Release gates

- Compare against the current production baseline using the same dataset and traffic shape.
- Set quality/safety floors and latency/cost regression ceilings before testing.
- Run deterministic component tests, offline evaluations, integration tests with simulated tools, then shadow/canary traffic.
- Version judge models and rubrics; calibrate model judges against human labels.
- Turn every production incident and material user correction into a regression case.

---

## 11. Cost and unit economics

Track **cost per successful task**, not only cost per model call:

```text
model + embedding + reranking + search + compute + storage + egress
+ retries + failed workflows + observability + human review
──────────────────────────────────────────────────────────────────
                         successful tasks
```

### Highest-impact cost levers

1. Route simple work to smaller models and avoid retrieval/model calls when deterministic code can answer.
2. Reduce unnecessary context and output tokens; rerank broadly, then send only high-value evidence.
3. Cache stable prefixes and safe repeated results with correct tenant/version/freshness keys.
4. Batch embeddings and offline jobs; use incremental ingestion.
5. Bound agent steps, fan-out, retries, and verification passes.
6. Separate interactive and batch capacity; use reserved/dedicated capacity only after utilization is predictable.
7. Detect expensive failures and retry loops through per-trace cost attribution.

Maintain budgets and alerts per tenant, model capability, workflow, environment, and feature. Showback/chargeback becomes necessary when shared platforms serve multiple teams.

---

## 12. Deployment, migration, and change management

### 12.1 Independently deployable artifacts

Treat these as separate versioned releases:

- Application and workflow code.
- Prompts and structured-output schemas.
- Model routes and inference configuration.
- Tool definitions and authorization policy.
- Parser, chunker, embedding, and retrieval configuration.
- Index/data version.
- Safety policy and feature flags.

A single trace must identify the exact combination. Roll back each artifact independently when possible.

### 12.2 Delivery path

```text
unit/component tests
 → offline evaluation
 → integration + fault tests
 → load/security tests
 → shadow traffic
 → small canary
 → staged tenant/region rollout
 → full production
```

Use infrastructure as code for networks, identity, compute, queues, data stores, indexes, dashboards, and alerts. Keep development, staging, and production data/indexes separate.

For schema, embedding, or index migrations, use backfill + dual-write/dual-read comparison + atomic alias/routing switch. Keep the old version until rollback risk has passed.

### 12.3 AWS mapping (optional implementation)

| Capability | Common AWS option |
|---|---|
| Model inference | Amazon Bedrock or dedicated/self-hosted inference |
| Stateless API/workers | ECS on Fargate; Lambda for bounded event-driven tasks |
| Durable workflow | Step Functions |
| Queues/events | SQS with DLQ; EventBridge for routing/schedules |
| Artifacts | S3 |
| Durable state | Aurora/RDS or DynamoDB by access pattern |
| Search | OpenSearch, Bedrock Knowledge Bases, or another vector/search service |
| Entry point | API Gateway for Lambda-centric APIs or ALB for ECS services |
| Identity/secrets/encryption | IAM, Secrets Manager, KMS |
| Operations | CloudWatch, CloudTrail, and OpenTelemetry-compatible tracing |

A common shape is **API Gateway/ALB → stateless Fargate application → Bedrock/model gateway**, with **SQS/Step Functions and separate workers** for long-running work. Service choice must follow the workload model, not the other way around.

### 12.4 Regional strategy

- Route stateless requests to a healthy eligible region, but pin an active conversation or workflow when its durable state is regional.
- Decide which data is replicated: prompts/config, relational state, artifacts, indexes, caches, and audit records have different consistency and residency needs.
- Pre-provision provider/model quotas and required indexes in the recovery region; DNS failover alone does not create AI capacity.
- Define behavior when the latest index or workflow checkpoint has not replicated: serve stale-but-labeled data, queue, switch to search-only, or refuse according to risk.
- Test regional failover with real authentication, secrets, model routes, queues, and data dependencies, then measure achieved RPO/RTO.

---

## 13. Critical design review checklist

### Workload and capacity

- [ ] Launch and growth estimates exist for RPS, concurrency, tokens, data, tenants, workflow amplification, and regional traffic.
- [ ] End-to-end latency budgets, quotas, saturation point, burst headroom, and load-shedding order are documented and tested.
- [ ] Interactive, ingestion, evaluation, and batch workloads have separate queues/pools or enforced priorities.

### Architecture and data

- [ ] Synchronous vs asynchronous boundaries and durable state owners are explicit.
- [ ] Model, retrieval, and tool access use stable internal contracts with versioned schemas.
- [ ] Ingestion is incremental, idempotent, resumable, observable, and faster than the expected source change rate.
- [ ] Index rollout, rollback, re-embedding, deletion propagation, backup, and restore have tested procedures.
- [ ] Tenant/ACL filtering happens inside retrieval; large tenants can be isolated without API redesign.

### Agents and actions

- [ ] Agent loops, fan-out, retries, tokens, time, and spend are runtime-bounded.
- [ ] Long or approval-paused work uses durable orchestration, cancellation, and replay.
- [ ] Side effects have authorization, preview/approval where needed, idempotency, postcondition checks, and compensation.

### Reliability and security

- [ ] Deadlines, retry classification, circuit breakers, bulkheads, backpressure, degradation, and kill switches are implemented.
- [ ] Provider/tool/index outage and partial-success scenarios have been fault-tested.
- [ ] Secrets are brokered, egress is restricted, logs are redacted, and tenant quotas prevent noisy neighbors.
- [ ] SLOs, burn-rate alerts, RPO/RTO, runbooks, ownership, and recovery drills are in place.

### Evaluation and economics

- [ ] Component and end-to-end evaluations cover important slices, permission boundaries, no-answer cases, and adversarial failures.
- [ ] Every response is reproducible from traced model/prompt/policy/tool/index versions.
- [ ] Canary and rollback gates cover quality, safety, p95/p99 latency, and cost per successful task.
- [ ] Cost is attributed by tenant/workflow/model and includes retries, infrastructure, observability, and human review.

---

## 14. At-scale failure patterns

| Symptom | Likely scale cause | Architectural response |
|---|---|---|
| p50 is good but p99 is unusable | Long-context/tool outliers; shared pool contention | Per-stage deadlines, workload isolation, output limits, tail-aware routing |
| Provider 429s cause a retry storm | Unbounded retries and no local admission control | Token/concurrency quotas, exponential backoff, retry budget, circuit breaker |
| Interactive traffic slows during re-index | Batch and online workloads share resources | Separate queues/pools, priorities, reserved interactive capacity |
| Retrieval returns nothing after ACL filtering | ANN post-filtering or wrong partition strategy | Native pre-filtered search, oversampling, tenant-aware partitioning |
| One tenant degrades everyone | No quota/bulkhead; skewed partition | Tenant rate/storage limits, fair scheduling, move tenant to dedicated resources |
| Model costs grow faster than users | Agent/tool-call amplification and retries | Cost per task tracing, bounded steps/fan-out, smaller-model routing |
| Retry repeats a real-world action | Missing idempotency and persisted status | Idempotency key, commit record, postcondition check, compensation |
| New embedding improves tests but breaks production | Test corpus not representative; unsafe in-place re-index | Slice evals, dual index, shadow reads, atomic switch and rollback |
| Failover model changes behavior | API-compatible but semantically different provider | Evaluate fallback route, normalize contracts, restrict when safety-critical |
| Deleted or restricted data remains answerable | Derived indexes/caches not in deletion workflow | End-to-end lineage, tombstones, cache invalidation, deletion SLA tests |
| System cannot explain a bad response | Missing versioned traces or context provenance | End-to-end trace IDs and immutable release/index metadata |
| Queue never recovers after outage | Retry arrival rate exceeds worker capacity | Backoff, admission control, priority draining, autoscaling, controlled replay |

The critical test of an at-scale design is not whether it works under normal load. It is whether its latency, cost, data boundaries, and state remain controlled during bursts, dependency failures, migrations, retries, and uneven tenant growth.
