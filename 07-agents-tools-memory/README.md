# Stage 7: Tools, agents and memory

Letting models act safely, with bounded cost and a clear stop point.

Summary, rules and context: [AI Engineering Guide, Stage 7](../AI_ENGINEERING_GUIDE.md#stage-7-tools-agents-and-memory).

## Learn

Curriculum lessons, in reading order.

- [Lesson 7.1 — Tools, Agents, and Structured Output](learn/01-tools-agents-and-structured-output.md)

## Apply

Field-manual scenarios: implementation steps, an example, trade-offs, production notes, a checklist and metrics.

**1-agents/**

- [Agents → Sandboxing (code execution and shell tools)](apply/1-agents/sandboxing.md)
- [Agents → Multi-agent communication (MCP, A2A, shared blackboard)](apply/1-agents/multi_agent_communication_mcp.md)
- [Agents → Cost & step budgets](apply/1-agents/cost_step_budgets.md)
- [Agents → Single agent loop](apply/1-agents/single_agent_loop.md)
- [Agents → Task decomposition ("define down")](apply/1-agents/task_decomposition.md)
- [Agents → Orchestrator / subagent](apply/1-agents/orchestrator_subagent.md)
- [Agents → Fan-out / fan-in](apply/1-agents/fan_out_fan_in.md)
- [Agents → Loop-back / reflection (critic, verifier, replanning)](apply/1-agents/loop_back_reflection.md)
- [Agents → Planning styles (ReAct, plan-then-execute, search)](apply/1-agents/planning_styles.md)
- [Agents → Tool design](apply/1-agents/tool_design.md)
- [Agents → Human-in-the-loop (approval gates and resume)](apply/1-agents/human_in_the_loop.md)
- [Agents → State & durability (checkpointing and idempotency)](apply/1-agents/state_durability.md)

**2-memory/**

- [Memory → Types (short-term, working, episodic, semantic, procedural)](apply/2-memory/memory_types.md)
- [Memory → What to write (the extraction step)](apply/2-memory/what_to_write.md)
- [Memory → When to retrieve (relevance + recency, with a cap)](apply/2-memory/when_to_retrieve.md)
- [Memory → How to forget (TTL, decay, deletion, user controls)](apply/2-memory/how_to_forget.md)
- [Memory → Conflict resolution (supersede, don't accumulate)](apply/2-memory/conflict_resolution.md)
- [Memory → Security (memory as an injection surface)](apply/2-memory/memory_security.md)

---

Previous: [Stage 6: Retrieval-augmented generation](../06-rag/README.md) · Next: [Stage 8: Choosing the lever](../08-choosing-the-lever/README.md)
