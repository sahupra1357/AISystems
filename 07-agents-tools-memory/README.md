# Stage 7: Tools, agents and memory

Letting models act safely, with bounded cost and a clear stop point.

Summary, rules and context: [AI Engineering Guide, section 9](../AI_ENGINEERING_GUIDE.md#9-stage-7-tools-agents-and-memory).

## Learn

Curriculum lessons, in reading order.

- [Lesson 04 — Tools, Agents, and Structured Output](learn/04-tools-agents-and-structured-output.md)

## Apply

Field-manual scenarios: implementation steps, an example, trade-offs, production notes, a checklist and metrics.

**1-agents/**

- [5.10 Agents → Sandboxing (code execution and shell tools)](apply/1-agents/05.10_sandboxing.md)
- [5.11 Agents → Multi-agent communication (MCP, A2A, shared blackboard)](apply/1-agents/05.11_multi_agent_communication_mcp.md)
- [5.12 Agents → Cost & step budgets](apply/1-agents/05.12_cost_step_budgets.md)
- [5.1 Agents → Single agent loop](apply/1-agents/05.1_single_agent_loop.md)
- [5.2 Agents → Task decomposition ("define down")](apply/1-agents/05.2_task_decomposition.md)
- [5.3 Agents → Orchestrator / subagent](apply/1-agents/05.3_orchestrator_subagent.md)
- [5.4 Agents → Fan-out / fan-in](apply/1-agents/05.4_fan_out_fan_in.md)
- [5.5 Agents → Loop-back / reflection (critic, verifier, replanning)](apply/1-agents/05.5_loop_back_reflection.md)
- [5.6 Agents → Planning styles (ReAct, plan-then-execute, search)](apply/1-agents/05.6_planning_styles.md)
- [5.7 Agents → Tool design](apply/1-agents/05.7_tool_design.md)
- [5.8 Agents → Human-in-the-loop (approval gates and resume)](apply/1-agents/05.8_human_in_the_loop.md)
- [5.9 Agents → State & durability (checkpointing and idempotency)](apply/1-agents/05.9_state_durability.md)

**2-memory/**

- [6.1 Memory → Types (short-term, working, episodic, semantic, procedural)](apply/2-memory/06.1_memory_types.md)
- [6.2 Memory → What to write (the extraction step)](apply/2-memory/06.2_what_to_write.md)
- [6.3 Memory → When to retrieve (relevance + recency, with a cap)](apply/2-memory/06.3_when_to_retrieve.md)
- [6.4 Memory → How to forget (TTL, decay, deletion, user controls)](apply/2-memory/06.4_how_to_forget.md)
- [6.5 Memory → Conflict resolution (supersede, don't accumulate)](apply/2-memory/06.5_conflict_resolution.md)
- [6.6 Memory → Security (memory as an injection surface)](apply/2-memory/06.6_memory_security.md)

---

Previous: [Stage 6: Retrieval-augmented generation](../06-rag/README.md) · Next: [Stage 8: Choosing the lever](../08-choosing-the-lever/README.md)
