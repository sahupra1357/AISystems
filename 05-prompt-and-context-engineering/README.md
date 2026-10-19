# Stage 5: Prompt and context engineering

Reliable behaviour from what you put in the model's window.

Summary, rules and context: [AI Engineering Guide, Stage 5](../AI_ENGINEERING_GUIDE.md#stage-5-prompt-and-context-engineering).

## Learn

Curriculum lessons, in reading order.

- [Lesson 5.1 — Prompting Deep Dive](learn/01-prompting-deep-dive.md)

## Apply

Field-manual scenarios: implementation steps, an example, trade-offs, production notes, a checklist and metrics.

**1-prompt-engineering/**

- [Prompt engineering → Prompt structure (stable-first, ordered sections)](apply/1-prompt-engineering/prompt_structure.md)
- [Prompt engineering → Delimiters: instructions vs data](apply/1-prompt-engineering/delimiters_data_vs_instructions.md)
- [Prompt engineering → Zero-shot and few-shot prompting](apply/1-prompt-engineering/zero_few_shot.md)
- [Prompt engineering → Chain-of-thought (when it pays, when it doesn't)](apply/1-prompt-engineering/chain_of_thought.md)
- [Prompt engineering → Self-consistency (sample n, majority vote)](apply/1-prompt-engineering/self_consistency.md)
- [Prompt engineering → Decomposition, negative examples, and prefilling](apply/1-prompt-engineering/decomposition_negative_examples_prefill.md)
- [Prompt engineering → Dynamic few-shot selection (retrieve the k most similar solved cases)](apply/1-prompt-engineering/dynamic_few_shot_selection.md)
- [Prompt engineering → Structured outputs & tool use (schema, validate, retry)](apply/1-prompt-engineering/structured_outputs_tool_use.md)
- [Prompt engineering → Prompt versioning & evaluation (prompts are code)](apply/1-prompt-engineering/prompt_versioning_evaluation.md)

**2-context-engineering/**

- [Context engineering → What it is (managing everything in the window)](apply/2-context-engineering/what_is_context_engineering.md)
- [Context engineering → Compaction / summarisation of long history](apply/2-context-engineering/compaction_summarization.md)
- [Context engineering → "Lost in the middle" (position matters)](apply/2-context-engineering/lost_in_the_middle.md)
- [Context engineering → Per-source truncation budgets](apply/2-context-engineering/per_source_truncation_budgets.md)
- [Context engineering → Prompt caching of static prefixes](apply/2-context-engineering/prompt_caching.md)
- [Context engineering → Context isolation (subagents get their own window)](apply/2-context-engineering/context_isolation.md)
- [Context engineering → Failure modes: poisoning, distraction, confusion, clash](apply/2-context-engineering/context_failure_modes.md)

---

Previous: [Stage 4: Foundation models and LLMs](../04-llm-foundations/README.md) · Next: [Stage 6: Retrieval-augmented generation](../06-rag/README.md)
