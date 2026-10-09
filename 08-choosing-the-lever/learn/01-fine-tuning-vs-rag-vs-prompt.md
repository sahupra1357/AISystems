# Lesson 8.1 — Fine-Tuning vs RAG vs Prompt

When should you change **instructions**, supply **context**, or change **weights**? Teams waste months fine-tuning when a better prompt + RAG would ship in a week—or they refuse to fine-tune when style consistency will never stabilize with examples alone.

This lesson gives a practical decision framework, concrete scenarios, LoRA intuition, data and cost realities, and maintenance burdens so you can choose deliberately.

## Learning goals

- Compare prompting, RAG/tools, and fine-tuning on capability, freshness, cost, and ops load
- Walk a decision flowchart with scenario practice
- Explain what fine-tuning teaches (and what it does not)
- Describe LoRA / PEFT at an engineer’s level of intuition
- Estimate data needs and maintenance traps
- Combine methods (common in production) without double-counting effort

---

## 1. Three levers (and a fourth)

| Lever | What you change | Best at | Weak at |
|-------|-----------------|---------|---------|
| **Prompting** | Instructions, examples, parameters | Format, tone light-touch, using general skills | Private/fresh facts; ultra-stable quirks |
| **RAG / tools** | Context and actions at request time | Private docs, citations, live state | Inherent skill gaps; very stable micro-format sometimes |
| **Fine-tuning** | Model weights (often adapters) | Consistent style/format/behavior; domain phrasing | Changing facts; cheap iteration speed |
| **Product / UI** | Buttons, forms, constraints | Irreversible actions, clarity | Not everything should be a chat |

Always ask whether a **non-LLM** control is better before any of the three.

---

## 2. Decision flowchart

```text
Need private, proprietary, or frequently changing facts?
  YES → RAG and/or tools (do not bake into weights alone)
  NO  → continue

Need live side effects (refund, ticket, DB write)?
  YES → tools (+ HITL as needed); prompts only orchestrate

Is the task mostly format/tone/labeling the base model already can do with clear instructions?
  YES → strong prompting + structured output + eval
  MAYBE → add few-shot; measure golden set

Still failing golden set after prompt+RAG/tools (behavior/style/skill)?
  AND you have clean training pairs + eval + budget?
    YES → consider fine-tune (often LoRA) WHILE keeping RAG for facts
    NO  → gather data / simplify product scope

Facts change after fine-tune?
  YES → you still need RAG/tools for those facts
```

**Golden rule:** Fine-tuning is not a substitute for a knowledge base.

---

## 3. Detailed comparison

| Dimension | Prompt | RAG/Tools | Fine-tune |
|-----------|--------|-----------|-----------|
| Time to first prototype | Hours | Days | Weeks+ |
| Ops burden | Low | Medium (index, ACLs) | Higher (train, deploy, regress) |
| Freshness of facts | Weak | Strong | Weak alone |
| Behavioral consistency | Medium | Medium | Strong if data good |
| Cost at inference | Baseline | +retrieval | Adapter may match base; training is extra |
| Reversibility | Easy revert templates | Reindex / rollback index | Must version adapters + eval |
| Privacy | Prompt logs | Doc ACLs critical | Data in training sets / weights risk |

---

## 4. What fine-tuning actually teaches

Fine-tuning adjusts weights so the model more often behaves like your training examples. It can:

- Adopt a house tone or rigid XML/JSON layout
- Improve domain phrasing and abbreviations
- Specialize classification-like behavior in generative wrappers
- Reduce how often you need long few-shot prompts (saving tokens)

It cannot magically:

- Stay current with a changing wiki better than RAG
- Encode authorization policy safely
- Guarantee zero hallucinations
- Replace evaluation

If you bake secrets into weights, you create privacy and compliance headaches—treat training data with care.

---

## 5. LoRA and PEFT intuition

Full fine-tuning updates all parameters—expensive and heavy to store per variant.

**LoRA (Low-Rank Adaptation)** freezes base weights and learns small low-rank adapter matrices injected into layers.

Why teams like it:

- Far fewer trainable parameters → cheaper training
- Small artifact files to store and swap
- Multiple adapters can specialize one base model (brand A vs brand B)
- Easier to disable/rollback an adapter than to re-cook a full model

**PEFT** = parameter-efficient fine-tuning family (LoRA is the headline example). Follow current docs for your stack (training frameworks and provider fine-tune offerings change).

```text
Base model (frozen)
   +
LoRA adapters (trained)
   =
Specialized behavior at inference
```

### Inference note

You still prompt the adapted model. Fine-tune + RAG + tools is a normal production combo: adapters for voice/format, RAG for facts, tools for live state.

---

## 6. Data needed for fine-tuning

Quality beats quantity. Prefer:

- Clean input → desired output pairs aligned with production distribution
- Coverage of edge cases and **refusals** / abstains
- De-duplication and PII scrubbing
- Held-out evaluation pairs **never** used in training
- Versioned dataset cards: source, license, date, known gaps

### Rough intuition (not a promise)

| Task | Data sketch |
|------|-------------|
| Mild tone shift | Hundreds of good pairs may help |
| Strict format + domain | Low thousands often discussed in practice |
| Hard new skill | More data + stronger base model; prove need with baselines |

Always beat a **prompt+RAG baseline** on the same held-out set before celebrating.

### Negative / refusal data

If the model should decline medical advice or legal commitments, include those examples. Otherwise fine-tuning on only “helpful answers” can make it over-eager.

---

## 7. Cost and maintenance

### Cost buckets

1. **Data engineering** (often the real cost)
2. **Training compute**
3. **Eval labor**
4. **Inference** (adapter overhead usually small; worse if you force a larger base)
5. **Ongoing refresh** when product voice or taxonomy changes

### Maintenance traps

- Base model vendor deprecates the version your adapter was trained on
- Taxonomy changes → silent label drift
- Team edits prompts *and* adapters without regression gates → mystery soup
- RAG corpus updates while users expect fine-tune to “know” new pages

**Operational habit:** one owner for adapter versions; changelog; golden set gate before promote.

---

## 8. Scenario workshop

For each scenario, choose prompt / RAG / tools / fine-tune (or combo). Justify in 3–5 sentences.

### Scenario A — HR policy Q&A

Employees ask about leave, benefits, hybrid work. Policies update quarterly.

**Strong answer:** RAG (+ authz) + grounded prompts. Fine-tune optional only for tone. Tools if integrating HRIS balances.

### Scenario B — Brand voice for marketing drafts

Company wants a quirky but consistent voice across many campaigns; facts come from briefs pasted by humans.

**Strong answer:** Start few-shot + critique pass. If still inconsistent at scale, fine-tune on accepted drafts. Keep humans for factual claims.

### Scenario C — Support ticket classification into 20 labels

Years of labeled tickets exist; latency sensitive.

**Strong answer:** Classifier (classical or fine-tuned generative classifier / dedicated model). Prompts alone may work at low volume; at high volume, specialized model often wins. RAG optional for explanations.

### Scenario D — “Refund this order” in chat

**Strong answer:** Tools + HITL + policy RAG. Do not fine-tune “to learn refunds.” Never trust narration.

### Scenario E — Code assistant knowing your private monorepo APIs

**Strong answer:** RAG over code/docs + tools (search, open file). Fine-tune rarely replaces retrieval for large changing codebases.

### Scenario F — Extract fields from idiosyncratic PDFs into strict schema

**Strong answer:** Schema-first prompting + validation; chunk/layout parsing. Fine-tune if layout family is stable and prompt few-shot fails golden set.

### Scenario G — Multi-tenant SaaS knowledge bases

**Strong answer:** RAG with hard tenant filters. Fine-tune shared model on format OK; never train on Tenant A data to serve Tenant B.

---

## 9. Combining methods (typical production)

```text
Router (prompt or small model)
  ├─ FAQ-ish → RAG answer
  ├─ Account state → tools
  └─ Draft marketing → fine-tuned style model + human edit
```

Document the combo so on-call knows which lever to turn when quality drops.

---

## 10. Experiment plan before fine-tuning

1. Freeze a golden set (Lesson 9.1)
2. Record prompt-only baseline scores
3. Record prompt+RAG/tools baseline
4. Estimate data + training cost
5. Train adapter on train split only
6. Compare on held-out; include safety/refusal cases
7. Shadow deploy; watch online metrics
8. Keep ability to kill-switch back to baseline

If step 3 already meets the bar, **stop**—ship the simpler system.

---

## 11. Anti-patterns

| Anti-pattern | Why | Prefer |
|--------------|-----|--------|
| Fine-tune to memorize the wiki | Stale + privacy risk | RAG |
| Fine-tune before measuring prompt baselines | Unneeded complexity | Baselines first |
| One giant adapter for all tenants’ secrets | Leakage risk | Shared skills + per-tenant retrieval |
| No held-out set | You will overfit anecdotes | Gold + regression |
| Updating adapter and prompt simultaneously blindly | Confounds | One lever per experiment |

---

## 12. Practice

1. Fill a decision matrix for your workplace (or a fictional product) across five user journeys.
2. Write a one-page LoRA explainer for a backend engineer: what freezes, what trains, how you deploy.
3. List PII scrubbing steps before any fine-tune dataset export.
4. Given Scenario C, outline features for a non-LLM classifier vs LLM approach—when would you pick each?

---



## 13. Worked comparison table with fake-but-realistic numbers

Suppose a support-answer feature must hit ≥85% rubric pass on a 100-item gold set.

| System | Build time | Gold pass | $/1k answers | Notes |
|--------|------------|-----------|--------------|-------|
| Prompt only | 2 days | 61% | $4 | Hallucinates policy |
| Prompt + RAG | 1.5 weeks | 88% | $7 | Meets bar |
| RAG + LoRA tone | 4 weeks | 90% | $7.5 | Marginal gain |
| Fine-tune facts, no RAG | 5 weeks | 70% | $5 | Fails freshness test next month |

**Decision:** ship prompt+RAG; park LoRA unless brand tone becomes a KPI the 88% system fails.

Numbers are illustrative—your job is to fill a real table like this before committing to weights.

## 14. Prompt-only failure signatures that look like “need fine-tune”

| Signature | Often actually needs |
|-----------|----------------------|
| Wrong private facts | RAG/tools |
| Unstable JSON | Schema mode + validation + few-shot |
| Ignores one rule among twenty | Split prompts / conflict removal |
| Bad multi-hop | Decomposition + retrieval per hop |
| Wrong live order status | Tools |
| Slightly off tone | Few-shot or light fine-tune |

Misdiagnosis is expensive. Collect 20 failures, tag root causes, then choose the lever that matches the dominant tag.

## 15. Fine-tune data pipeline sketch

```text
Production logs / CRM exports
  → filter to consented, allowed data
  → strip PII
  → convert to {input, ideal_output} pairs
  → dedupe near-duplicates
  → split train / val / held-out gold (by time or hash)
  → human spot-check 5–10%
  → train adapter
  → eval on held-out + safety suite
  → registry: adapter version ↔ base model ↔ dataset version
```

If you cannot name the dataset version in an incident, you are not ready to fine-tune in production.

## 16. Evaluation gates specific to fine-tuning

Before promote:

- [ ] Beats prompt and prompt+RAG baselines on held-out
- [ ] No severe regression on refusal / safety cases
- [ ] Schema_pass_rate ≥ baseline
- [ ] Latency within SLO on target hardware
- [ ] Rollback path tested (traffic back to previous adapter or base)

## 17. Organizational checklist: “Should we fine-tune this quarter?”

1. Is there a metric owner and a gold set?
2. Did simpler levers fail with evidence?
3. Is training data legally usable?
4. Do we have someone who can retrain when the base model changes?
5. Is the expected lift worth the opportunity cost vs shipping RAG ACLs / eval CI?

If three or more answers are “no,” do not fine-tune yet.

## 18. Extended scenario answers (model solutions)

**Scenario A (HR):** RAG + grounded prompts; optional tone few-shot; HRIS tool for balances; quarterly reindex. Fine-tune low priority.

**Scenario B (brand voice):** few-shot library from approved posts → measure→ LoRA if still inconsistent; humans approve facts; no RAG of confidential roadmaps into public marketing model context.

**Scenario C (20 labels):** start with classical or encoder classifier on labels; if you must use an LLM, fine-tune or instruct with strict label enum + low temp; RAG only for “explain why.”

**Scenario D (refund):** tools + HITL + policy RAG; fine-tune irrelevant for side effects.

**Scenario E (monorepo):** code RAG + search tools; maybe fine-tune style of explanations later.

**Scenario F (PDF extract):** layout parsing + schema prompts; fine-tune if a stable document family dominates volume.

**Scenario G (multi-tenant):** hard ACL RAG; shared format adapter OK if trained without tenant secrets.

## 19. Practice: write the one-pager exec decision

Pick Scenario B or C. Write:

- Problem statement
- Baselines tried
- Recommendation (lever)
- Cost/risk
- Eval plan
- Kill criteria

Keep it to one page—executives fund clarity, not vibes.



## 20. Parameter-efficient alternatives and cousins (awareness)

Besides LoRA you may hear about:

- **QLoRA** — LoRA atop quantized base weights to reduce memory during training
- **Adapters / prefix tuning** — other PEFT variants with similar “small add-on” spirit
- **Provider fine-tune APIs** — hosted training on your JSONL; still need your eval

You do not need to memorize every acronym. You need to know: **prefer PEFT over full fine-tunes for large bases unless you have a rare reason**, and always keep a baseline.

## 21. Prompt distillation vs fine-tune

Sometimes a large model + fancy prompt generates training pairs for a smaller model (distillation-style data). That can cut latency/cost. It is still a fine-tune project with the same eval obligations—and it can amplify the teacher’s blind spots. Measure on gold; include refusals.

## 22. When the answer is “change the product”

Examples:

- Replace open chat with a form for refunds
- Show three FAQ cards before chat
- Require order id picker instead of free text

These often beat any model work on reliability. Record them as first-class options in your decision matrix.



## 23. Full decision worksheet (copy/paste)

Use this when a stakeholder says “we should fine-tune.”

```text
Problem:
User journey:
Success metric + gold set size:
Baseline A — prompt only — score/date:
Baseline B — prompt+RAG/tools — score/date:
Dominant failure tags (facts / format / tone / tools / safety):
Freshness requirements:
Privacy / tenancy constraints:
Data available for fine-tune (count, license, PII status):
Training budget + who maintains adapters:
Recommendation:
Kill / rollback criteria:
```

If Baseline B already meets the metric, the worksheet should end with “do not fine-tune.”

## 24. Mixing adapters with RAG: ownership

| Artifact | Owner | Cadence |
|----------|-------|---------|
| Prompt templates | Feature eng | Weekly as needed |
| Index / chunker | Search/platform | On corpus change |
| Adapter weights | ML eng | Rare; gated |
| Gold set | Shared | Continuous from incidents |

Incidents should name which artifact changed. “The model got worse” is not a root cause.

## 25. Mini case: taxonomy change

You fine-tuned a 20-label ticket classifier. Product adds 3 labels and renames 2.

**What breaks:** adapter still emits old labels; dashboards skew; routers misfire.

**Response:** update label enum in prompts immediately if still prompt-based; for adapters, collect new labeled data, retrain, dual-run old/new, gate on gold including new labels; never silently map unknown → `other` without measuring.


## Key vocabulary

prompt vs RAG vs fine-tune; PEFT; LoRA; adapter; training pair; held-out eval; baseline; kill switch; taxonomy drift; parametric vs non-parametric knowledge

## What is next

**[01-evaluation-for-llm-systems.md](../../09-evaluation/learn/01-evaluation-for-llm-systems.md)** — prove changes help: golden sets, rubrics, pairwise compares, regression gates, and online signals.
