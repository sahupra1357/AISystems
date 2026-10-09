# Part 5 — Production AI Systems: Overview

A model that works in a notebook is not yet a product. **Production AI** means data pipelines you can trust, deployments you can roll back, monitors that notice silent failures, costs you can explain, safety controls that respect users, and **explicit decisions** about quality gates—including when humans must review.

This part connects classical ML and LLM features (Parts 2–4) to engineering habits used on real teams. Every lesson centers **tradeoffs** (options → costs → how to choose) and how **evaluation** plus **human-in-the-loop (HITL)** gate releases—not only “what tool to install.”

## Goals for Part 5

By the end of Part 5, you should be able to:

- Apply a decision framework: problem → constraints → options → tradeoffs → eval plan → HITL plan → ship criteria → rollback
- Version data, track experiments, and promote models/prompts/indexes through a registry with eval gates
- Choose and sketch serving paths: batch, sync API, async queue, streaming—and FastAPI + Docker basics
- Design monitoring for quality, drift, latency, and cost; wire human review queues for low-confidence or thumbs-down traffic
- Implement layered safety/privacy: PII handling, prompt-injection defenses, guardrails, and risk-tiered HITL
- Operate reliability patterns: timeouts, retries, circuit breakers, fallbacks, graceful degradation, runbooks, postmortems
- Write a production design doc (capstone) that a teammate could implement
- Pass the Ready-for-Part-6 checklist

## Prerequisites

Complete **Parts 1–4** (or equivalent experience):

- You can train/evaluate a classical model and call an LLM API
- You understand RAG failure modes and golden-set evaluation (Part 4 lesson 06)
- You can sketch an orchestrator / retriever / LLM / tools shape
- Comfortable with Git, virtualenvs, reading logs, and basic HTTP concepts

```bash
python -m pip install fastapi uvicorn pydantic  # for serving sketches
# Later as needed: joblib, mlflow or wandb client, redis, etc.
```

## Mental model: production AI is a loop

```text
     ┌──────────────┐
     │  Data & labels│──► versioned snapshots / corpora
     └──────┬───────┘
            ▼
     ┌──────────────┐
     │ Train / adapt │──► model, prompt, index, tools config
     └──────┬───────┘
            ▼
     ┌──────────────┐
     │    Serve      │──► batch / API / queue / stream
     └──────┬───────┘
            ▼
     ┌──────────────┐
     │   Observe     │──► metrics, traces, cost, drift, feedback
     └──────┬───────┘
            ▼
     ┌──────────────┐
     │   Improve     │──► eval gates + HITL → promote or rollback
     └──────┬───────┘
            └──────────────► (back to data / adapt)
```

Part 4 built the *feature* (prompt, RAG, tools, eval). Part 5 hardens the *outer ring*: pipelines, serving, monitors, safety, reliability—and makes **eval + HITL first-class release controls**, not afterthoughts.

## Explicit teaching contract

For every major decision in this part you will see:

1. **Options** — at least 2–3 real alternatives
2. **Tradeoffs** — cost, latency, quality, complexity, risk, team skill
3. **How to choose** — heuristics and a default path for a small team
4. **Eval + HITL** — when automated metrics suffice, when humans must review, sample rates, escalation

There is rarely one “correct” stack. There *is* a correct habit: write down options, pick with eyes open, and prove the pick with eval and human verification before you promote.

## Suggested time: 4–6 weeks part-time

This is deeper than a tool tour. Budget time for design writing, small prototypes, and the capstone design doc.

| Week | Focus |
|------|--------|
| 1 | Overview + **01** Production mindset & decision framework (tradeoffs, risk tiers, worked examples) |
| 2 | **02** Data pipelines & MLOps (versioning, features, registry, eval gates) |
| 3 | **03** Serving architectures (batch/API/queue, caching, routing, rollouts) |
| 4 | **04** Monitoring, drift & cost (logs, quality, SLOs, review queues) |
| 5 | **05** Safety, privacy & HITL (PII, injection, guardrails, HITL rates) |
| 6 | **06** Reliability & incidents + **07** Exercises, design-doc capstone, Ready-for-Part-6 checklist |

Experienced backend engineers may compress weeks 3–4; still do the decision tables and HITL plans—those catch most AI-specific production bugs. If Part 4 evaluation felt thin, revisit Part 4 lesson 06 in parallel with week 1–2.

## Order to study

1. **[01-production-mindset-and-decision-framework.md](01-production-mindset-and-decision-framework.md)**
2. **[02-data-pipelines-and-mlops.md](02-data-pipelines-and-mlops.md)**
3. **[03-serving-architectures.md](03-serving-architectures.md)**
4. **[04-monitoring-drift-and-cost.md](04-monitoring-drift-and-cost.md)**
5. **[05-safety-privacy-and-hitl.md](05-safety-privacy-and-hitl.md)**
6. **[06-reliability-and-incident-response.md](06-reliability-and-incident-response.md)**
7. **[07-exercises-and-checklist.md](07-exercises-and-checklist.md)**

## Success criteria before Part 6

You are ready for Capstones when you can:

- Walk a feature through the decision framework and produce a short tradeoff table
- Explain how you would version training data (or a RAG corpus) and register a model/prompt/index
- Sketch serving for sync and async paths; say when you would add a worker queue
- List metrics and alerts for quality, latency, cost, and drift—and when humans review samples
- Describe PII handling, prompt-injection mitigations, and a risk-tiered HITL policy
- Design timeouts, retries, fallbacks, and a rollback for prompts/indexes/models
- Deliver the Part 5 production design-doc capstone (options, tradeoffs, eval, HITL, rollout, rollback)

Part 6 asks you to *use* these habits end-to-end on a portfolio project.
