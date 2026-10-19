# Deployment → AWS: Fargate, Lambda, Bedrock

## Video takeaway

"Three services, three jobs. Fargate hosts your application — the API, the streaming responses, the retrieval service, the agent runtime — anything that needs to stay warm and hold connections. Lambda runs short stateless work: one invocation per document for parsing, connector polls, webhooks, scheduled cleanup jobs. Bedrock runs the model. The traps are predictable: Lambda's fifteen-minute limit kills large OCR jobs, so split per page or escalate to Fargate; cold starts hurt on the request path, so keep streaming on Fargate; and Bedrock quotas are regional, so capacity planning is per region. For durable agent runs with human approval waits, Step Functions gives you checkpointing and the pause for free."

## What it is

Three compute models with different shapes, and AI workloads stress exactly the dimensions where they differ:

| | Fargate (ECS/EKS) | Lambda | Bedrock |
|---|---|---|---|
| **Role** | Long-running services: API, retrieval, agent runtime, workers | Short stateless functions: connectors, per-document parsing, webhooks, event handlers | Model inference and managed agents/knowledge bases |
| **Max duration** | Unbounded | 15 minutes | Per-request |
| **Cold start** | None (warm tasks) | 100 ms–several seconds | N/A |
| **Streaming** | Native (SSE/WebSocket) | Response streaming supported, with caveats | Streaming API |
| **State** | In-memory caches, warm connections, loaded models | Stateless; /tmp only | Stateless |
| **Concurrency** | Task count × per-task concurrency | Massive automatic fan-out | Provider quotas |
| **Cost shape** | Per vCPU/GB-hour, always on | Per-request + GB-second, zero idle | Per token (or provisioned throughput) |

The mapping for a RAG/agent system:

```
Fargate:  API + streaming responses, retrieval service, reranker, agent runtime,
          long-running orchestration, anything holding warm connections or caches
Lambda:   connector polls, per-document parse jobs, S3 event handlers, webhooks
          (approval callbacks), scheduled retention/GC jobs, small transforms
Bedrock:  model inference (Claude and others), embeddings (Titan/Cohere),
          Guardrails, Knowledge Bases, Agents — the managed-retrieval option (build vs buy)
Also:     Step Functions for durable agent workflows (state & durability); SQS between ingest stages;
          OpenSearch Serverless or Aurora pgvector for the index (vector store choice)
```

## How to implement

1. **Put the request path on Fargate**: streaming needs a persistent connection, and the retrieval service benefits from warm connections and in-process caches.
2. **Use Lambda for the ingestion fan-out**: one invocation per document (or per page), triggered by S3 events or a queue — this is where Lambda's automatic concurrency is genuinely valuable.
3. **Respect the 15-minute limit**: a large PDF with OCR can exceed it. Split per page, or move heavy parsing to Fargate/Batch tasks ([per-page classification and routing](../../../06-rag/apply/1-ingestion-parsing/per_page_routing.md)).
4. **Bedrock for inference** with cross-region inference profiles for capacity, model IDs pinned ([versioning & rollout](../1-serving-cost-latency/versioning_rollout.md)), and Guardrails if you want managed input/output filtering ([input guardrails](../5-guardrails/input_guardrails.md), [output guardrails](../5-guardrails/output_guardrails.md)).
5. **Step Functions for durable agent runs** — it gives checkpointing, retries, and human-approval waits as primitives ([state & durability](../../../07-agents-tools-memory/apply/1-agents/state_durability.md), [human-in-the-loop](../../../07-agents-tools-memory/apply/1-agents/human_in_the_loop.md)).
6. **Queues between ingest stages** (SQS with DLQs) so fetch, parse, embed, and index scale independently and failures are visible ([connectors](../../../06-rag/apply/1-ingestion-parsing/connectors.md)).
7. **IAM as the least-privilege boundary**: per-task roles, per-tenant scoped policies, no long-lived keys ([excessive agency](../5-guardrails/excessive_agency.md), [authentication & authorization](../6-security-privacy-governance/authn_authz_permission_filtered_retrieval.md)).

## Small example

```python
# ---- Lambda: per-document parse, triggered by S3 (short, stateless, fan-out) ----
def lambda_parse_handler(event, context):
    """One document per invocation. Guard the 15-minute ceiling explicitly."""
    for rec in event["Records"]:
        key = rec["s3"]["object"]["key"]
        meta = s3.head_object(Bucket=BUCKET, Key=key)
        n_pages = int(meta["Metadata"].get("pages", 0))
        if n_pages > 200 or meta["ContentLength"] > 100 * 1024 * 1024:
            ecs.run_task(cluster=CLUSTER, taskDefinition=HEAVY_PARSE_TASK,      # escalate to Fargate
                         overrides={"containerOverrides": [{"name": "parser",
                                    "environment": [{"name": "S3_KEY", "value": key}]}]})
            continue
        if context.get_remaining_time_in_millis() < 60_000:
            sqs.send_message(QueueUrl=PARSE_QUEUE, MessageBody=json.dumps({"key": key}))  # requeue, don't fail
            continue
        elements = parse_document(s3_stream(BUCKET, key))                        # Per-page classification and routing
        sqs.send_message(QueueUrl=CHUNK_QUEUE, MessageBody=json.dumps({"key": key,
                         "elements_ref": store_elements(elements)}))             # pass by reference, not payload

# ---- Fargate: the request path, streaming, warm caches ----
"""
ECS service:
  - task: api (2 vCPU, 4 GB), min 3 tasks, autoscale on ALB request count + p95 latency
  - ALB with sticky-free routing; SSE streaming through the ALB (idle timeout > max response time)
  - warm: embedding client, reranker client, vector store connections, prompt cache breakpoints (prompt caching of static prefixes)
  - IAM task role: bedrock:InvokeModel* on pinned model ARNs; read-only S3 on the document prefix
"""

def bedrock_generate(messages, system, model_id="anthropic.claude-sonnet-4-20250514-v1:0",
                     inference_profile="us.anthropic.claude-sonnet-4-20250514-v1:0"):
    resp = bedrock_runtime.converse_stream(
        modelId=inference_profile,                       # cross-region profile for capacity (capacity & limits)
        system=[{"text": system, "cachePoint": {"type": "default"}}],   # prompt caching (prompt caching of static prefixes)
        messages=messages,
        inferenceConfig={"maxTokens": 1500, "temperature": 0},
        guardrailConfig={"guardrailIdentifier": GUARDRAIL_ID, "guardrailVersion": "3"},  # Input guardrails/08.3
    )
    for event in resp["stream"]:
        if "contentBlockDelta" in event:
            yield event["contentBlockDelta"]["delta"]["text"]

# ---- Step Functions: durable agent runs with approval waits (human-in-the-loop, state & durability) ----
STATE_MACHINE = {
    "StartAt": "Plan",
    "States": {
        "Plan":    {"Type": "Task", "Resource": PLAN_LAMBDA, "Next": "FanOut"},
        "FanOut":  {"Type": "Map", "MaxConcurrency": 12, "ItemsPath": "$.subtasks",   # Fan-out / fan-in
                    "Iterator": {"StartAt": "Subagent",
                                 "States": {"Subagent": {"Type": "Task", "Resource": SUBAGENT_LAMBDA,
                                                         "Retry": [{"ErrorEquals": ["States.TaskFailed"],
                                                                    "IntervalSeconds": 2, "MaxAttempts": 3,
                                                                    "BackoffRate": 2.0}],
                                                         "Catch": [{"ErrorEquals": ["States.ALL"],
                                                                    "ResultPath": "$.error", "Next": "Failed"}],
                                                         "End": True},
                                            "Failed": {"Type": "Pass", "End": True}}},   # partial failure OK
                    "Next": "NeedsApproval"},
        "NeedsApproval": {"Type": "Choice",
                          "Choices": [{"Variable": "$.requires_approval", "BooleanEquals": True,
                                       "Next": "WaitForHuman"}], "Default": "Synthesize"},
        "WaitForHuman": {"Type": "Task",
                         "Resource": "arn:aws:states:::lambda:invoke.waitForTaskToken",   # durable pause
                         "Parameters": {"FunctionName": REQUEST_APPROVAL_LAMBDA,
                                        "Payload": {"taskToken.$": "$$.Task.Token", "input.$": "$"}},
                         "TimeoutSeconds": 86400, "Next": "Synthesize"},
        "Synthesize": {"Type": "Task", "Resource": SYNTH_LAMBDA, "End": True},
    },
}
```

## Pros

- **Fargate**: no cold starts, native streaming, warm caches and connections, long-running work, straightforward local/dev parity.
- **Lambda**: massive automatic fan-out for ingestion, zero idle cost, native event triggers from S3/SQS/EventBridge.
- **Bedrock**: multiple model families behind one API, IAM-native auth, in-VPC access, regional processing for residency ([data residency & retention](../6-security-privacy-governance/data_residency_retention.md)), managed Guardrails and Knowledge Bases ([build vs buy](../../../04-llm-foundations/apply/build_vs_buy.md)).
- **Step Functions**: durable execution, retries, and human-approval waits as primitives rather than code you write ([state & durability](../../../07-agents-tools-memory/apply/1-agents/state_durability.md)).

## Cons

- **Lambda's 15-minute ceiling and cold starts** make it a poor fit for the interactive request path and for heavy OCR.
- **Lambda statelessness** means no warm model clients or in-process caches; connection setup per invocation adds latency.
- **Fargate always-on cost** for bursty ingestion workloads.
- **Bedrock model availability differs by region**, and quotas are per-region — capacity planning is regional ([capacity & limits](../1-serving-cost-latency/capacity_limits.md)).
- **Managed Knowledge Bases trade control**: chunking, ACLs, and retrieval debugging are limited ([build vs buy](../../../04-llm-foundations/apply/build_vs_buy.md)).
- Streaming through ALB/API Gateway needs careful timeout configuration.

## When to use each

- **Fargate**: the API and streaming layer, retrieval service, self-hosted reranker/embedder, agent runtime, long orchestration.
- **Lambda**: connector polling, per-document/per-page parse jobs, S3 and queue event handlers, webhooks (approval callbacks), scheduled retention and GC jobs.
- **Bedrock**: all model inference; Guardrails when managed filtering suffices; Knowledge Bases for a fast prototype ([build vs buy](../../../04-llm-foundations/apply/build_vs_buy.md)).
- **Step Functions**: any agent run that must survive restarts or wait for a human.

## When NOT to use

- **Lambda for the streaming request path** — cold starts and statelessness hurt exactly where users feel it.
- **Lambda for whole-document OCR** on large PDFs; split per page or use Fargate/Batch.
- **Bedrock Knowledge Bases** when you need per-chunk source ACLs, custom chunking, or retrieval-level debugging ([build vs buy](../../../04-llm-foundations/apply/build_vs_buy.md), [authentication & authorization](../6-security-privacy-governance/authn_authz_permission_filtered_retrieval.md)).
- **Fargate for spiky, low-duty-cycle ingestion** — you'll pay for idle capacity Lambda wouldn't charge for.

## Production at scale

**Split the ingestion pipeline across queues** (SQS between fetch → parse → chunk → embed → index) with DLQs at each stage. This gives independent scaling, visible failures, and natural retry semantics ([connectors](../../../06-rag/apply/1-ingestion-parsing/connectors.md), [failure handling](../4-observability-reliability/failure_handling.md)).

**Pass references, not payloads.** Lambda payload limits and SQS message size mean documents and parsed elements go to S3 with a reference in the message. This also keeps the trace store lean ([observability & tracing](../4-observability-reliability/observability_tracing.md)).

**Cross-region inference profiles** for Bedrock spread load across regions and raise effective quota — but check that every region in the profile satisfies your residency commitments ([data residency & retention](../6-security-privacy-governance/data_residency_retention.md)).

**Provisioned throughput** for Bedrock is worth modelling once volume is steady: it trades per-token pricing for reserved capacity, which also removes throttling risk ([capacity & limits](../1-serving-cost-latency/capacity_limits.md)).

**IAM is the least-privilege mechanism** ([excessive agency](../5-guardrails/excessive_agency.md)): per-task roles scoped to specific model ARNs and specific S3 prefixes, per-tenant STS sessions for tenant-scoped storage access, and no long-lived credentials anywhere near the agent runtime.

**Step Functions for agent durability**: `waitForTaskToken` gives you a durable human-approval pause without building your own checkpoint store ([human-in-the-loop](../../../07-agents-tools-memory/apply/1-agents/human_in_the_loop.md), [state & durability](../../../07-agents-tools-memory/apply/1-agents/state_durability.md)), and Map state gives bounded-concurrency fan-out with partial-failure handling ([fan-out / fan-in](../../../07-agents-tools-memory/apply/1-agents/fan_out_fan_in.md)).

**Watch the ALB idle timeout** for streaming responses — a 60-second default will cut long generations. Set it above your maximum expected response duration.

**Cost attribution by tag**: tag tasks, functions, and Bedrock calls by tenant and feature so the cost dashboard can slice them ([cost levers](../1-serving-cost-latency/cost_levers.md)).

**Local development parity**: Fargate containers run locally; Lambda needs SAM or LocalStack; Bedrock needs either real calls or a recorded fixture layer. Plan for the last one, or your tests become slow and expensive ([the testing pyramid](../../../09-evaluation/apply/testing_pyramid.md)).

## Prod checklist

- [ ] Request path and streaming on Fargate with warm clients and connections
- [ ] Ingestion fan-out on Lambda, per document or per page, with the 15-minute ceiling guarded
- [ ] Heavy parsing escalated to Fargate/Batch tasks
- [ ] SQS between ingest stages with DLQs and alerting
- [ ] Payloads passed by S3 reference, not inline
- [ ] Bedrock model IDs pinned; cross-region inference profiles checked against residency
- [ ] Prompt caching (cachePoint) configured; cache hit rate monitored
- [ ] Step Functions for durable agent runs and approval waits
- [ ] Per-task IAM roles scoped to model ARNs and data prefixes; no long-lived keys
- [ ] ALB idle timeout above max response duration
- [ ] Cost tags by tenant and feature; Bedrock quota and throttle monitoring

## Metrics to watch

| Metric | Why |
|---|---|
| Lambda duration p95 vs timeout; requeue rate | The 15-minute ceiling |
| Cold-start rate on any request-path function | Latency |
| SQS queue depth and DLQ size per stage | Ingest health |
| Bedrock throttle rate and quota utilisation per region | Capacity ([capacity & limits](../1-serving-cost-latency/capacity_limits.md)) |
| Cost by service, tagged by tenant | Attribution ([cost levers](../1-serving-cost-latency/cost_levers.md)) |

## Related

- [Build vs buy](../../../04-llm-foundations/apply/build_vs_buy.md) — Bedrock Knowledge Bases as the managed option
- [Self-hosted serving](../1-serving-cost-latency/self_hosted_serving.md) — the alternative to Bedrock for open models
- [Capacity & limits](../1-serving-cost-latency/capacity_limits.md) — regional quotas and throttling
- [State & durability](../../../07-agents-tools-memory/apply/1-agents/state_durability.md) — Step Functions as the durable runtime
- [Excessive agency](../5-guardrails/excessive_agency.md), [AuthZ](../6-security-privacy-governance/authn_authz_permission_filtered_retrieval.md) — IAM as the boundary
- [Connectors](../../../06-rag/apply/1-ingestion-parsing/connectors.md), [Routing](../../../06-rag/apply/1-ingestion-parsing/per_page_routing.md) — the ingestion fan-out
