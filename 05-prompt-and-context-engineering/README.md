# Stage 5: Prompt and context engineering

Reliable behaviour from what you put in the model's window.

Summary, rules and context: [AI Engineering Guide, section 7](../AI_ENGINEERING_GUIDE.md#7-stage-5-prompt-and-context-engineering).

## Learn

Curriculum lessons, in reading order.

- [Lesson 5.1 — Prompting Deep Dive](learn/01-prompting-deep-dive.md)

## Apply

Field-manual scenarios: implementation steps, an example, trade-offs, production notes, a checklist and metrics.

**1-prompt-engineering/**

- [2.1 Prompt engineering → Prompt structure (stable-first, ordered sections)](apply/1-prompt-engineering/02.1_prompt_structure.md)
- [2.2 Prompt engineering → Delimiters: instructions vs data](apply/1-prompt-engineering/02.2_delimiters_data_vs_instructions.md)
- [2.3 Prompt engineering → Zero-shot and few-shot prompting](apply/1-prompt-engineering/02.3_zero_few_shot.md)
- [2.4 Prompt engineering → Chain-of-thought (when it pays, when it doesn't)](apply/1-prompt-engineering/02.4_chain_of_thought.md)
- [2.5 Prompt engineering → Self-consistency (sample n, majority vote)](apply/1-prompt-engineering/02.5_self_consistency.md)
- [2.6 Prompt engineering → Decomposition, negative examples, and prefilling](apply/1-prompt-engineering/02.6_decomposition_negative_examples_prefill.md)
- [2.7 Prompt engineering → Dynamic few-shot selection (retrieve the k most similar solved cases)](apply/1-prompt-engineering/02.7_dynamic_few_shot_selection.md)
- [2.8 Prompt engineering → Structured outputs & tool use (schema, validate, retry)](apply/1-prompt-engineering/02.8_structured_outputs_tool_use.md)
- [2.9 Prompt engineering → Prompt versioning & evaluation (prompts are code)](apply/1-prompt-engineering/02.9_prompt_versioning_evaluation.md)

**2-context-engineering/**

- [3.1 Context engineering → What it is (managing everything in the window)](apply/2-context-engineering/03.1_what_is_context_engineering.md)
- [3.2 Context engineering → Compaction / summarisation of long history](apply/2-context-engineering/03.2_compaction_summarization.md)
- [3.3 Context engineering → "Lost in the middle" (position matters)](apply/2-context-engineering/03.3_lost_in_the_middle.md)
- [3.4 Context engineering → Per-source truncation budgets](apply/2-context-engineering/03.4_per_source_truncation_budgets.md)
- [3.5 Context engineering → Prompt caching of static prefixes](apply/2-context-engineering/03.5_prompt_caching.md)
- [3.6 Context engineering → Context isolation (subagents get their own window)](apply/2-context-engineering/03.6_context_isolation.md)
- [3.7 Context engineering → Failure modes: poisoning, distraction, confusion, clash](apply/2-context-engineering/03.7_context_failure_modes.md)

---

Previous: [Stage 4: Foundation models and LLMs](../04-llm-foundations/README.md) · Next: [Stage 6: Retrieval-augmented generation](../06-rag/README.md)
