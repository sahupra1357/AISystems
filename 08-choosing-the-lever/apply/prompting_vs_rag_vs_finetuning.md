# Foundations → Prompting vs RAG vs fine-tuning

## Video takeaway

"Three levers, three problems. The model doesn't *know* something — your contracts, this week's prices — that's retrieval: RAG. The model knows but says it wrong — wrong format, wrong tone, wrong procedure — that's prompting. Prompting works but the prompt is huge, slow, and still flaky at scale — that's fine-tuning, for style and format, never for facts. Facts in weights go stale and can't be deleted. Diagnose the failure first, pull the matching lever, and re-run the same eval so you know which lever actually moved the number."

## What it is

| Lever | Changes | Cost to iterate | Typical use |
|---|---|---|---|
| **Prompting** | Behaviour: tone, format, procedure, constraints, examples | Minutes; zero training | Output format, persona, step procedures, few-shot for a task pattern |
| **RAG** | Knowledge: fresh, private, or too large for weights | Hours–days to build a pipeline; ongoing ingestion | Internal documents (contracts, policies, tickets, wikis); anything that changes |
| **Fine-tuning** | Style, format, narrow-domain behaviour; latency/cost via distillation | Days–weeks; data + training + eval | House formats (SOAP notes), consistent extraction on a niche domain, small-model distillation |

The decision tree:

```
Is the model wrong because it lacks facts it couldn't have?        → RAG (knowledge)
Is it wrong because it has the facts but misformats/misbehaves?    → Prompting (behaviour)
Does prompting work but the prompt is huge/brittle/slow/expensive? → Fine-tuning (bake it in)
Do you need a smaller/faster model to do what the big one does?    → Distillation (fine-tuning)
Does the knowledge change weekly?                                  → Never fine-tune for it; RAG
```

They stack: a fine-tuned small model for format, RAG for facts, a prompt for the task instructions.

## How to implement

1. **Diagnose the failure** with examples from logs/golden set. Label each: `missing_knowledge` / `wrong_behaviour` / `inconsistent_style` / `too_slow_or_costly`.
2. **Prompting first** for behaviour: structure ([prompt structure](../../05-prompt-and-context-engineering/apply/1-prompt-engineering/prompt_structure.md)), delimiters ([delimiters: instructions vs data](../../05-prompt-and-context-engineering/apply/1-prompt-engineering/delimiters_data_vs_instructions.md)), few-shot ([zero-shot and few-shot prompting](../../05-prompt-and-context-engineering/apply/1-prompt-engineering/zero_few_shot.md)/02.7), structured output ([structured outputs & tool use](../../05-prompt-and-context-engineering/apply/1-prompt-engineering/structured_outputs_tool_use.md)). Measure on the golden set ([prompt evaluation](../../09-evaluation/apply/prompt_evaluation.md)).
3. **RAG** for knowledge: build the pipeline ([ingestion & Parsing](../../AI_System.md#ingestion--parsing), [retrieval / RAG pipeline](../../AI_System.md#retrieval--rag-pipeline)). Measure retrieval and generation separately ([retrieval evals](../../09-evaluation/apply/retrieval_evals.md)/07.5).
4. **Fine-tune** only when: the behaviour is stable, prompting works but doesn't scale (prompt too long → cost/latency; few-shot still inconsistent), or you need to distil to a smaller model ([fine-tuning](../../AI_System.md#fine-tuning-when-you-get-there)). Keep RAG for facts.
5. **Re-evaluate after each lever** on the same golden set so the gain is attributable.

## Small example — the same failure, three fixes

```python
# Failure: model produces clinical notes in inconsistent structure and sometimes invents medication doses.

# 1) Behaviour → prompting (format + procedure)
SYSTEM = """You write SOAP notes. Output exactly four sections with these headers: Subjective, Objective, Assessment, Plan.
Never state a medication dose that is not present in the transcript; write 'dose not stated' instead."""

# 2) Knowledge → RAG (the facts it can't know: this patient's history, formulary)
ctx = retrieve(f"patient {pid} history and current medications", tenant=clinic_id, k=8)
prompt = f"<transcript>{transcript}</transcript>\n<records>{ctx}</records>\nWrite the SOAP note from the transcript; use records only to resolve references."

# 3) Style at scale → fine-tuning (when 2k prompt tokens of format rules per call is too slow/expensive,
#    or few-shot still yields 8% format violations). Train a LoRA on (transcript, ideal SOAP note) pairs;
#    keep RAG for patient facts; the system prompt shrinks to one line.
#    See SFT, LoRA.
```

## Pros (of choosing correctly)

- **Prompting**: fastest feedback loop; no infra; reversible.
- **RAG**: fresh, auditable (citations), per-tenant, permission-aware; knowledge updates are ingestion, not retraining.
- **Fine-tuning**: consistent style/format; shorter prompts; enables small fast models; can encode domain conventions prompting can't express.

## Cons (of choosing wrongly)

- **Fine-tuning for knowledge**: stale the day training ends; no citations; leaks across tenants; can't delete a fact (GDPR); hallucination doesn't go away.
- **Prompting for knowledge**: stuffing docs into the prompt scales badly (cost, "lost in the middle", ["Lost in the middle"](../../05-prompt-and-context-engineering/apply/2-context-engineering/lost_in_the_middle.md)).
- **RAG for behaviour**: retrieval doesn't fix tone or format; it just adds context.
- **Fine-tuning too early**: the task changes, you retrain; evaluation becomes harder ([when *not* to fine-tune](fine-tuning/when_not_to_fine_tune.md)).

## When to use each

- **Prompting**: always first; for format, tone, procedure, refusal behaviour, tool-use instructions.
- **RAG**: the org has documents; answers must cite; facts change; per-user/tenant knowledge.
- **Fine-tuning**: stable narrow task at high volume; house style; distillation for cost/latency; domain vocabulary the base model mishandles (medical, legal shorthand) — with RAG still supplying the facts.

## When NOT to use each

- **Prompting**: when the prompt is 5k tokens of rules and still fails 10% of the time — you've hit the ceiling; fine-tune.
- **RAG**: for behaviour problems; for tiny static knowledge that fits comfortably in a cached system prompt ([prompt caching of static prefixes](../../05-prompt-and-context-engineering/apply/2-context-engineering/prompt_caching.md)).
- **Fine-tuning**: for changing knowledge; before the task is stable; when a better prompt fixes it; when you can't build an eval ([when *not* to fine-tune](fine-tuning/when_not_to_fine_tune.md)).

## Production at scale

**Attribute every fix to a lever.** Keep the golden set constant and run it after each change; the diff tells you which lever moved which metric. Without this, teams fine-tune to fix what was a retrieval bug.

**Prompting at scale** means prompt versioning, caching of the static prefix ([prompt caching of static prefixes](../../05-prompt-and-context-engineering/apply/2-context-engineering/prompt_caching.md)), and evals in CI ([prompt versioning & evaluation](../../05-prompt-and-context-engineering/apply/1-prompt-engineering/prompt_versioning_evaluation.md)). A "quick prompt tweak" in production without an eval run is the most common regression source.

**RAG at scale** is the bulk of this reference ([ingestion & Parsing](../../AI_System.md#ingestion--parsing), [retrieval / RAG pipeline](../../AI_System.md#retrieval--rag-pipeline)): ingestion cost, ACLs, versioning, hybrid + rerank, and the evaluation split between retrieval and generation.

**Fine-tuning at scale** is an MLOps problem: dataset versioning, held-out sets, training reproducibility, adapter registry, and the same eval suite as prompts ([fine-tuning ops](fine-tuning/fine_tuning_ops.md)). Budget for periodic retrains as the base model is deprecated — a fine-tune is tied to a base model version.

**Combine deliberately.** A common production stack: fine-tuned small model (format/style, cheap) + RAG (facts, citations) + short prompt (task) + reasoning-tier escalation ([routing & escalation](../../04-llm-foundations/apply/routing_escalation.md)) for hard cases. Each part has its own eval and its own owner.

**Compliance angles.** RAG lets you delete a document and it's gone; fine-tuned weights can't forget. For PII/GDPR-sensitive knowledge, RAG is the only defensible choice ([data residency & retention](../../10-production/apply/6-security-privacy-governance/data_residency_retention.md), [PII](../../10-production/apply/6-security-privacy-governance/pii.md)).

**Cost angles.** Prompt tokens are paid every call; fine-tuning is paid once per training run. At high volume, moving 3k tokens of instructions from the prompt into weights is a real saving — *if* the behaviour is stable enough to justify retraining cadence.

## Prod checklist

- [ ] Failures labelled by type (knowledge / behaviour / style / cost)
- [ ] Prompting exhausted and measured before fine-tuning
- [ ] RAG for all changing/private/citable knowledge; never fine-tuned in
- [ ] Fine-tuning gated on task stability + eval availability + measurable prompt ceiling
- [ ] Same golden set run after each lever change; gains attributed
- [ ] Fine-tune ↔ base-model version dependency tracked; retrain plan
- [ ] Deletion/compliance requirements considered (RAG for erasable knowledge)

## Metrics to watch

| Metric | Why |
|---|---|
| Failure-type distribution from triage | Which lever to pull |
| Golden-set delta per lever change | Attribution |
| Prompt tokens per call (static portion) | Fine-tuning ROI signal |
| Format-violation rate | Prompting ceiling |

## Related

- [Prompt engineering](../../05-prompt-and-context-engineering/apply/1-prompt-engineering/) Prompt engineering, [Prompt caching](../../05-prompt-and-context-engineering/apply/2-context-engineering/prompt_caching.md)
- [Retrieval / RAG pipeline](../../AI_System.md#retrieval--rag-pipeline) RAG pipeline
- [Fine-tuning](fine-tuning/) Fine-tuning, [When not to](fine-tuning/when_not_to_fine_tune.md)
- [Prompt evaluation](../../09-evaluation/apply/prompt_evaluation.md), [Feedback triage](../../10-production/apply/8-data-flywheel/feedback_capture_triage.md)
