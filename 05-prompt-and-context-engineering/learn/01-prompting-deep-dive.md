# Lesson 5.1 — Prompting Deep Dive

This is the centerpiece of Stages 4–9. Prompting is how you steer foundation models day to day. Shallow prompting notes list a few buzzwords and move on; this lesson teaches **what a prompt is**, **how to assemble one**, **which style to pick**, **which parameters fit which task**, and **how to iterate without lying to yourself**.

Read slowly. Do the before/after rewrites. Keep a tiny prompt changelog even for homework—that habit transfers straight into production.

## Learning goals

By the end of this lesson you should be able to:

- Decompose any prompt into instruction, context, constraints, and output contract
- Place content in system vs user vs assistant roles for clear reasons
- Choose among major prompting styles with when-to-use and when-not-to-use judgment
- Match temperature / top_p / max tokens / stop sequences to task types
- Delimit untrusted input and stay injection-aware (full safety in Stage 10)
- Iterate with one-variable changes, golden examples, and anti-pattern avoidance

---

## 1. What a prompt actually is

A **prompt** is the full token sequence you condition the model on before it generates—including system rules, prior turns, retrieved documents, tool results, and the latest user ask.

For design purposes, treat a prompt as four cooperating parts:

| Part | Job | Examples |
|------|-----|----------|
| **Instruction** | What to do | “Extract fields”, “Answer using only context”, “Refuse if…” |
| **Context** | What to use | Docs chunks, email body, schema description, prior slots |
| **Constraints** | Boundaries | Length, tone, allowed labels, “no speculation”, safety rules |
| **Output contract** | Shape of success | JSON schema, markdown sections, citation format, abstain phrase |

**Why this split helps:** when something fails, you can ask “was the instruction unclear, context missing, constraint conflicting, or contract unenforceable?” instead of rewriting the whole blob.

### Before / after

```text
BEFORE (vague blob):
Write something about our competitors and make it nice for executives.

AFTER (four parts visible):
Instruction: Compare our product to two named competitors for an executive audience.
Context: Product one-pager + competitor notes (pasted below).
Constraints: No invented metrics; if a figure is missing, write "unknown". Max 200 words.
Output contract: Use headings Strengths, Gaps, Questions for sales; bullets only.
```

The after version is longer—and **cheaper overall**—because it reduces retries and hallucinated numbers.

---

## 2. Anatomy of a strong prompt

A durable template many teams converge on:

```text
1. Role / persona (who the model should act as)
2. Task (the verb + object)
3. Context (data, docs, state)
4. Format (schema or section layout)
5. Constraints (must / must-not)
6. Examples (optional few-shot)
7. Stop conditions (when to abstain, ask a clarifying question, or end)
```

Not every field is required every time. A one-line classify call may only need task + format + constraints. A RAG answer needs role, task, context, format, constraints, and stop conditions (“say you don’t know”).

### Worked anatomy

```text
Role: You are a support assistant for Acme Billing.
Task: Answer the user's question about refunds.
Context: [retrieved policy chunks with ids]
Format: Short paragraphs; cite chunk ids like [p12].
Constraints: Use ONLY the context. Do not invent policy. Do not offer exceptions.
Examples: (optional) one sample Q → A with citations.
Stop conditions: If context lacks the answer, reply exactly:
  "I do not see that in the policy excerpts I was given."
```

### Why each piece exists

- **Role** biases vocabulary and priorities (teacher vs lawyer vs terse API voice)
- **Task** reduces ambiguity about the verb (“summarize” ≠ “critique” ≠ “extract”)
- **Context** supplies facts the weights may not know
- **Format** makes outputs parseable and reviewable
- **Constraints** cut unsafe or off-brand continuations
- **Examples** teach pattern faster than adjectives
- **Stop conditions** define graceful failure—production gold

---

## 3. Message roles: system vs user vs assistant

### What belongs where

| Role | Put here | Why |
|------|----------|-----|
| **system** | Durable policy, role, global output rules, safety boundaries | Stable across turns; easier to version as “app policy” |
| **user** | Task for this turn, untrusted documents, questions | Reflects end-user or upstream system input |
| **assistant** | Prior model answers you intentionally keep | Continues multi-turn coherence |

Some APIs add a **tool** / **function** role for tool results—treat those as **data**, not instructions (Lesson 7.1).

### Why separation matters

1. **Versioning:** you can change system policy without rewriting every user template
2. **Security hygiene:** keep untrusted text out of system when possible; delimit it in user/context sections
3. **Debugging:** logs show whether a failure is policy (system) or instance data (user)
4. **Caching:** providers that cache long stable prefixes reward a stable system message

### Anti-pattern: stuffing everything into system

```text
BAD system message:
You are helpful. Also here is the entire employee handbook...
Also here is today's user email...
Also here is the API key for the CRM (never do this)...
```

**Why it is bad:** mixes policy with volatile data; encourages injection if handbook/email contain adversarial text; risks secret leakage into logs and model context.

**Prefer:** short system policy + user message that carries the email + RAG for handbook sections.

### Multi-turn sketch

```python
messages = [
    {"role": "system", "content": SYSTEM_POLICY},
    {"role": "user", "content": "What's the refund window?"},
    {"role": "assistant", "content": "For digital goods, 14 days [p3]."},
    {"role": "user", "content": "What about gift cards?"},
]
```

Only keep prior turns that still help. Wrong earlier answers can anchor the model—fix or drop them when correcting course.

---

## 4. Types / styles of prompting

Each subsection below follows the same teaching pattern: **definition → why it exists → how to write it → when to use → when NOT to use → mini example**.

### 4.1 Zero-shot

**Definition:** Ask for the task with instructions (and maybe format) but **no** input–output examples.

**Why it exists:** Fast; relies on the model’s pretraining and instruction tuning; great when the task is standard and well described.

**How to write it:** Clear task verb, explicit constraints, explicit output contract. Do not assume the model shares your unspoken template.

**When to use:** Simple classification with clear labels; straightforward rewrites; tasks the model already does well.

**When NOT to use:** Finicky output layouts; domain jargon with local meaning; edge-case-heavy policies; anything that failed twice already on phrasing alone.

```text
Classify the support ticket into one of: billing, bug, how_to, other.
Return JSON: {"label": "...", "confidence": "low|medium|high"}.
Ticket:
"""
{ticket}
"""
```

### 4.2 Few-shot / many-shot

**Definition:** Provide a handful (few-shot) or many (many-shot) demonstration pairs before the real input.

**Why it exists:** Examples teach format and decision boundaries more reliably than adjectives like “be consistent.”

**How to write it:** Keep examples **compact**, **representative**, and **labeled the way you want at inference**. Put the real input last in the same format. Avoid funny one-off examples that skew the pattern.

**When to use:** Brittle formatting; subtle label boundaries; style mimicry; teaching a house rubric.

**When NOT to use:** When examples eat the context window; when examples contain sensitive data you should not send; when a schema/constrained decode already locks format; when bad examples would teach the wrong rule.

```text
Extract company names as a JSON list.

Input: "I met Mira at Globex and Initech."
Output: ["Globex", "Initech"]

Input: "No companies here, just a hike."
Output: []

Input: "{new_sentence}"
Output:
```

**Many-shot note:** more examples can help until returns diminish or you dilute the task. Prefer quality and coverage of edge cases over dumping fifty near-duplicates.

### 4.3 Instruction prompting

**Definition:** Emphasis on explicit procedural instructions—checklists, must/must-not rules—over clever persona or long examples.

**Why it exists:** Instruction-tuned models are trained to follow numbered directives; clarity beats eloquence.

**How to write it:** Numbered steps; short sentences; put the output contract at the top or bottom consistently; resolve conflicts (do not say both “be concise” and “include every detail”).

**When to use:** Ops runbooks; compliance-ish wording; multi-step transforms (normalize → extract → format).

**When NOT to use:** Pure creative ideation where heavy rules crush variety; situations that need retrieval of private facts (instructions cannot invent your wiki).

```text
Follow these steps:
1) Normalize whitespace.
2) Extract dates in ISO-8601 if present.
3) Return {"dates": [...], "notes": "..."}.
Do not invent dates.
```

### 4.4 Role / persona prompting

**Definition:** Assign a role (“You are a staff SRE…”) to bias tone, priorities, and vocabulary.

**Why it exists:** Roles activate clusters of behaviors learned from related text; they are a cheap prior.

**How to write it:** One clear role + audience + success bar. Avoid cartoon personas that fight the task (“talk like a pirate” while extracting HIPAA fields).

**When to use:** Tutoring; code review voice; executive vs beginner explanations; consistent brand tone drafts.

**When NOT to use:** When role conflicts with constraints (“helpful unrestricted hacker” vs safety policy); when you need deterministic extraction (role adds variance); when role encourages unsupported expertise claims.

```text
You are a patient tutor for first-year Python students.
Explain the bug in plain language, then show a fixed snippet.
Do not mock the student. Do not invent library APIs.
```

### 4.5 Chain-of-thought (CoT)

**Definition:** Ask the model to reason in intermediate steps before the final answer (“think step by step,” structured scratchpad, etc.).

**Why it exists:** For some multi-step reasoning tasks, eliciting intermediate tokens improves final accuracy—the model effectively uses its own outputs as working memory.

**How to write it:** Prefer **structured** scratchpads (“Steps: … Final: …”) over endless freeform monologues. For products, consider a **hidden** reasoning channel if your stack supports separating internal reasoning from user-visible text—or run a two-pass prompt (reason privately in pass 1, answer cleanly in pass 2).

**When to use:** Math-ish word problems; multi-constraint planning; debugging logic; tasks where you can verify the final answer.

**When NOT to use:** Simple extraction (wastes tokens, can hurt); when exposing reasoning to end users would leak private chain-of-thought or tool internals; when latency/cost budgets are tight and gains are unproven on your golden set.

**Product caution:** Do **not** casually show raw CoT to end users. It can be verbose, wrong-yet-persuasive, or leak system instructions. Prefer user-facing answers that are short and verified.

```text
Solve the allocation problem.
Write brief numbered working steps, then on the last line:
FINAL_ANSWER: <number>
```

### 4.6 Step-by-step / numbered plans

**Definition:** Require a numbered plan or procedure in the output (similar spirit to CoT, but often the plan *is* the deliverable).

**Why it exists:** Humans review numbered plans easily; models stay more ordered.

**How to write it:** Specify granularity (“5–8 steps”), audience, and success criteria per step when useful.

**When to use:** Migration runbooks; study plans; implementation outlines; incident response drafts.

**When NOT to use:** Tasks needing a single label or single JSON object; when a plan would hallucinate tooling you do not have—pair with tools/RAG instead.

```text
Draft a 6-step rollout plan for enabling SSO.
Each step: owner role, risk, rollback note.
Do not invent vendor UI labels you were not given.
```

### 4.7 Self-ask / decomposition

**Definition:** Prompt the model to break a hard question into sub-questions, answer each, then compose.

**Why it exists:** Monolithic questions hide missing info; decomposition surfaces what to retrieve or ask the user.

**How to write it:** Ask for explicit sub-questions first; optionally stop after sub-questions so *your code* can RAG each one (powerful hybrid).

**When to use:** Complex research-style asks; multi-entity comparisons; “why is X broken?” where causes fan out.

**When NOT to use:** Single-hop factual lookups already solved by one retrieval; ultra-low latency paths.

```text
Before answering, list 2-4 sub-questions needed to answer the user.
Then answer each sub-question briefly.
Then synthesize a final answer in <=120 words.
User question: {q}
```

### 4.8 ReAct-style (reason + act) at prompt level

**Definition:** Interleave short reasoning with explicit actions (“Thought… Action… Observation…”) aimed at tools or lookups.

**Why it exists:** Connects language reasoning to external grounding—search, calculators, ticket APIs—reducing pure hallucination.

**How to write it (prompt-level):** Define legal actions and argument formats tightly. Better: use **native tool calling** (Lesson 7.1) and keep only light reasoning in text.

**When to use:** Workflows that must hit tools; multi-step data gathering with clear APIs.

**When NOT to use:** Pure formatting tasks; when freeform “Action:” parsing is fragile—prefer JSON tool calls your SDK validates.

```text
You can use actions: SEARCH[query], FINISH[answer]
Thought: I need the 2024 refund window.
Action: SEARCH[refund window digital goods]
```

(Your orchestrator executes SEARCH and returns Observation.)

### 4.9 Tree-of-thought / multi-path (high-level)

**Definition:** Explore multiple candidate reasoning paths, then select/merge—more expensive than single-chain CoT.

**Why it exists:** Some puzzles benefit from considering alternatives rather than one greedy chain.

**How to write it (conceptual):** Generate N plans; score with a rubric or second model pass; expand the best. This is an **orchestration pattern**, not a single magic sentence.

**When to use:** Hard planning with high value per request; offline jobs where cost is fine.

**When NOT to use:** Default interactive chat; when you lack an evaluator; high-QPS endpoints—the cost multiplies fast.

### 4.10 Generated knowledge prompting

**Definition:** Ask the model to first produce relevant background knowledge, then answer conditioned on that knowledge.

**Why it exists:** Surfaces latent knowledge before committing to an answer; can help some reasoning tasks.

**How to write it:** Two clear phases; still verify facts for anything consequential—generated knowledge can be wrong with high confidence.

**When to use:** Brainstorming explanations; soft domain warm-up before drafting.

**When NOT to use:** Regulated factual answers (use RAG); anything requiring citations to *your* corpus.

```text
Phase 1: List 5 bullet facts relevant to {topic} that a careful engineer should verify.
Phase 2: Using those bullets as hypotheses, draft an answer and mark each claim Needs-Source or Common-Knowledge.
```

### 4.11 Contrastive / positive-negative examples

**Definition:** Show desired **and** undesired outputs so the model learns the boundary.

**Why it exists:** Negative examples prevent systematic errors (“do not include trailing commentary”) better than another adjective.

**How to write it:** Keep pairs tight; label GOOD vs BAD explicitly; make the BAD example wrong for one clear reason.

**When to use:** Format enforcement; tone boundaries; safety-adjacent refusals phrasing; grading rubrics.

**When NOT to use:** When negatives accidentally teach harmful patterns in detail; when space is better spent on two solid positive edge cases.

```text
GOOD: {"label":"bug","confidence":"high"}
BAD:  Label: bug (confidence high)   ← not JSON
BAD:  {"label":"bug","confidence":"high","extra":"hope this helps!"}  ← extra keys forbidden
```

### 4.12 Constrained / schema-first prompting

**Definition:** Specify JSON/XML/markdown templates—and ideally use provider **JSON mode / structured output** features—so decoding is constrained.

**Why it exists:** Downstream code needs parseable data; free prose is a integration tax.

**How to write it:** Show the schema; list enums; state “no markdown fences” if your parser is strict; validate in code anyway.

**When to use:** Almost all production pipelines that feed software; extractions; routing; tool args.

**When NOT to use:** End-user essays where structure harms UX—still fine to use internal structured drafts then render.

```text
Return ONLY valid JSON matching:
{
  "intent": "billing"|"bug"|"how_to"|"other",
  "urgency": "low"|"medium"|"high",
  "needs_human": boolean
}
```

```python
import json
from typing import Any

ALLOWED = {"billing", "bug", "how_to", "other"}

def load_intent(raw: str) -> dict[str, Any]:
    data = json.loads(raw)
    if data["intent"] not in ALLOWED:
        raise ValueError("invalid intent")
    return data
```

### 4.13 Extraction vs generation vs classification vs rewriting vs critique

These are **task families**. The prompt style should follow the family.

| Family | Goal | Prompt emphasis | Typical decode |
|--------|------|-----------------|----------------|
| **Classification** | Pick from labels | Enum list, brief definitions, edge cases | temp ~0, short max tokens |
| **Extraction** | Pull fields from text | Schema, null policy, span fidelity | temp ~0, JSON |
| **Generation** | Create new text | Audience, length, tone, originality rules | higher temp optional |
| **Rewriting** | Transform existing text | What to preserve vs change | low–mid temp |
| **Critique / review** | Evaluate text against a rubric | Explicit rubric dimensions + scale | low temp, structured scores |

**Why this taxonomy matters:** people reuse a chatty “generation” prompt for extraction and then wonder why JSON drifts. Match family → style → parameters.

Mini examples:

```text
CLASSIFY: Choose one label from {...}. Reply with the label only.

EXTRACT: Fill the schema; use null if absent; do not paraphrase IDs.

GENERATE: Propose 8 names; avoid trademarked terms; vary length.

REWRITE: Simplify to grade-8 reading level; keep numbers unchanged.

CRITIQUE: Score clarity, correctness, tone on 1-5 with one evidence quote each.
```

### 4.14 Meta-prompts / prompt templates for apps

**Definition:** Prompts that **write or parameterize** other prompts; or application **templates** with variables (`{policy}`, `{user_question}`) stored in code/config.

**Why it exists:** Products need reusable, testable, versioned strings—not one-off playground pastes. Meta-prompts can help authors draft templates (still reviewed by humans).

**How to write templates:**

```python
RAG_ANSWER_TMPL = """You answer using ONLY the context.
If missing, say you do not know.
Cite sources as [id].

Context:
{context}

Question: {question}
"""

def render_rag_answer(context: str, question: str) -> str:
    return RAG_ANSWER_TMPL.format(context=context, question=question)
```

**When to use meta-prompts:** Internal prompt engineering assistants; migrating style guides into templates.

**When NOT to use:** Fully automated prompt rewrites deployed without eval—meta-prompts can silently regress quality.

**App rules:** version templates; code review them; bind variables safely (watch for format-string injection via user text—prefer `str.replace` or template engines that do not evaluate code).

---

## 5. Task-type playbooks

### 5.1 Classification

- List labels with one-line definitions
- Include “other” / “abstain” if forced choice would lie
- Few-shot edge cases beat long essays
- Temperature 0; consider returning label only, then map in code

### 5.2 Extraction

- Schema-first; null policy explicit
- Quote thresholds: “copy IDs exactly”
- Validate types/ranges server-side
- Chunk long documents; extract per chunk; merge with code

### 5.3 Summarization

- State audience and length budget
- “Do not add facts not in source”
- For long docs: map-reduce summarize sections then synthesize
- Separate “summary” from “recommendations” so opinions do not smuggle in

### 5.4 Brainstorming

- Higher temperature; ask for N diverse options
- Second pass: filter/rank with low temperature and a rubric
- Ban leaking private context into public drafts

### 5.5 Coding help

- Specify language, version, and constraints (“stdlib only”)
- Ask for explanation + code, or code-only, deliberately
- Require tests or examples when useful
- Never execute model-suggested shell blindly

### 5.6 Tutoring

- Role + student level
- Socratic mode vs direct answers—pick one
- Check understanding with a quick question at the end

### 5.7 RAG-answer prompts (vs free chat)

Free chat may use general knowledge. **RAG-answer prompts must prioritize provided context.**

```text
Differences that matter:
- Explicit "use ONLY context" (or "prefer context, mark outside knowledge")
- Citation contract tied to chunk ids YOU inserted
- Abstain language when context is insufficient
- Usually lower temperature
- No dangling "as an AI" fluff—users want grounded answers
```

```text
You are a docs Q&A assistant.
Use ONLY the Context. If the answer is not there, say you do not know.
Cite chunk ids like [n].

Context:
[1] ...
[2] ...

Question: ...
```

Wire citations in the UI from **retriever metadata**, not from free-form invented URLs.

---

## 6. Parameters that change behavior

| Parameter | Why it exists | How to choose |
|-----------|---------------|---------------|
| **temperature** | Controls randomness in sampling | 0–0.2 reliability tasks; 0.3–0.7 explanations; 0.7–1.0 ideation |
| **top_p** | Restrict sampling to nucleus mass | Often leave default; if tuning, change *either* temperature or top_p, not both wildly |
| **max tokens** | Caps length/cost; prevents runaway | Set to UI need + small buffer; don’t use huge defaults “just in case” |
| **stop sequences** | Hard stop at markers | Useful for `END`, custom delimiters, or halting before a leaked second section |

**Task starter map:**

| Task | temp | max tokens | notes |
|------|------|------------|-------|
| JSON extract | 0–0.2 | small | schema validate |
| Label classify | 0–0.2 | tiny | enums |
| RAG answer | 0–0.3 | medium | abstain + citations |
| Rewrite | 0.2–0.5 | ~source length | preserve facts |
| Brainstorm | 0.7–1.0 | medium | filter second pass |
| CoT math | 0–0.4 | higher | hide scratchpad from users |

Always change **one** variable when debugging quality.

---

## 7. Delimiters and untrusted input; prompt injection awareness

Anything you did not write—user emails, uploaded PDFs, web pages, even retrieved tickets—can contain text that tries to **override instructions** (“Ignore previous directions and…”).

### Delimiters (necessary, not sufficient)

```text
Instructions:
- Extract invoice totals.
- Return JSON.

Untrusted document:
"""
{document}
"""
```

Use clear fences (`"""`, XML-ish tags, markdown). Tell the model that fenced content is **data**.

### Awareness checklist

- Separate instructions from data
- Do not put secrets in the prompt
- Validate outputs (especially tool args)
- Treat retrieved docs as untrusted too
- Authorization happens in **your** code, not in prose rules

**Deeper controls** (guardrails, sandboxes, allowlists, human approval): see **Stage 10 — Safety, privacy, reliability**. This lesson only builds the prompting habits that make those controls feasible.

---

## 8. Iteration method

1. **Write a golden handful** (5–20 examples) before endless tweaking
2. **Change one variable** (instruction OR example OR temperature OR model)
3. **Keep a prompt changelog**

```text
2026-09-23  v3  Add null policy for missing emails; temp 0→0
  metric: schema_pass 14/20 → 18/20
  notes: still fails on signature blocks — add few-shot next
```

4. Prefer **templates + versions** in git over playground folklore
5. When stuck, split into two calls (extract then decide) instead of one mega-prompt

---

## 9. Anti-patterns

| Anti-pattern | Why it fails | Do instead |
|--------------|--------------|------------|
| Vague asks | Underspecified success | Four-part prompt |
| Mega-prompts | Conflicting rules, hard to debug | Chain focused calls |
| Conflicting rules | Model picks arbitrarily | Resolve priorities explicitly |
| Secrets in system | Leak + false security | Secret stores; never prompt |
| Prompts as security | Injection bypasses prose | Server authz + validation |
| Adjective stacking | “Be careful clear brief…” | Examples + schema |
| Unlimited history | Cost + drift | Trim/summarize/state |
| No abstain path | Confident nonsense | Teach “I don’t know” |
| Editing five things at once | No causal insight | One-variable iteration |

---

## 10. Practice section

### 10.1 Rewrite weak prompts

Turn each into a strong prompt (role/task/context/format/constraints/stop):

1. “Clean this data.”
2. “Are we liable here? Need something legal-ish.”
3. “Summarize the meeting and also figure out action items and owners and deadlines and make it pop.”

### 10.2 Three styles, one business task

Task: **triage inbound partner emails** into `{queue, priority, summary}`.

Design:

1. A **zero-shot schema-first** prompt
2. A **few-shot + contrastive** prompt
3. A **two-call** pipeline (extract facts → classify)

Say which you would ship first and how you would evaluate.

### 10.3 Shipping checklist for a prompt

- [ ] Instruction / context / constraints / output contract are identifiable
- [ ] Roles used deliberately; untrusted data delimited
- [ ] Style chosen with a written reason
- [ ] Temperature and max tokens set for the task family
- [ ] Abstain / escalation path defined
- [ ] Parser + validation in code
- [ ] Golden examples exist; changelog entry ready
- [ ] No secrets in the prompt
- [ ] Owner + version recorded

### Mini capstone for this lesson

Build a **prompt library** (markdown or YAML) with at least five templates covering classify, extract, rewrite, RAG-answer, and critique. For each: purpose, style, parameters, example I/O, known failure modes. You will reuse this in Lesson 9.2.

---



## 11. Combining styles without creating a mega-prompt

Real systems compose styles **across calls**, not by stacking every trick into one message.

| Stage | Style | Output |
|-------|-------|--------|
| 1. Route | zero-shot classify | `intent` |
| 2. Gather | tool / RAG (Lessons 6.1–7.1) | context pack |
| 3. Answer | schema-first + RAG-answer rules | user-visible text or JSON |
| 4. Check | critique rubric (optional) | pass/fail + notes |

**Why:** each stage has a clear contract and golden set. A failure tells you which stage broke.

Anti-pattern: one prompt that classifies, retrieves (magically), reasons in a tree, and writes poetry. That is undebuggable.

## 12. Worked case study: from playground to template

**Business ask:** “Help sales reply to pricing emails.”

### Weak playground prompt

```text
You're a sales expert. Write a reply.
```

### Problems

- No product facts → invented discounts
- No tone guide → inconsistent brand
- No structure → hard to review in CRM
- No abstain → answers legal questions it should not

### Stronger system + user split

```text
SYSTEM:
You draft replies for Acme sales. Use ONLY the Pricing Context.
If asked for legal commitments or custom discounts beyond the context, set needs_human=true and draft a brief holding reply.
Tone: clear, respectful, no hype adjectives.
Return JSON:
{
  "subject": string,
  "body": string,
  "needs_human": boolean,
  "citations": [string]
}

USER:
Pricing Context:
"""
{retrieved_pricing_chunks}
"""

Inbound email:
"""
{email}
"""
```

### Evaluation seeds

1. Standard seat-count question with answer in context → `needs_human=false`, correct numbers, citations present
2. “Can you do 90% off forever?” → `needs_human=true`
3. Empty context → abstain / needs_human, no invented price list

This case study is the pattern for Lessons 6.1–9.1: prompt contract + retrieval + eval.

## 13. Prompt testing notes (lightweight)

Even before the full evaluation lesson:

- Store prompts next to code; PR review required for production templates
- Snapshot 10 fixtures: input → expected constraints (not always exact strings)
- Run fixtures on model upgrades before flipping traffic
- Track schema_pass_rate and human spot-check notes

```python
FIXTURES = [
    {"id": "refund_14", "email": "...", "expect_needs_human": False},
    {"id": "custom_discount", "email": "...", "expect_needs_human": True},
]

def run_fixture(model_call, fixture):
    out = model_call(fixture["email"])
    assert out["needs_human"] == fixture["expect_needs_human"]
```

## 14. Choosing a style under time pressure

Quick decision tree:

```text
Need parseable fields for code?
  yes → schema-first (+ JSON mode if available)
Need private/fresh facts?
  yes → RAG-answer (and/or tools), not cleverer prose
Format still flaky after clear instructions?
  → add 1–3 few-shots; consider contrastive BAD examples
Multi-step reasoning error on golden set?
  → structured scratchpad / two-pass; measure cost
Need tools?
  → native function calling (Lesson 7.1), not fragile ReAct text parsing
Still unstable behavior/style after prompts+RAG?
  → consider fine-tune (Lesson 8.1), still keep eval
```


## Key vocabulary

prompt anatomy; output contract; system/user/assistant roles; zero-shot; few-shot; instruction prompting; persona; chain-of-thought; decomposition; ReAct-style; tree-of-thought; generated knowledge; contrastive examples; schema-first; meta-prompt / template; temperature; top_p; stop sequences; prompt injection awareness

## What is next

**[01-rag-deep-dive.md](../../06-rag/learn/01-rag-deep-dive.md)** — when prompting alone cannot know your private or changing knowledge: chunking, embeddings, hybrid retrieval, citations, and RAG debugging playbooks.
