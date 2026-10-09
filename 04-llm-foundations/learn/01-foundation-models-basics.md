# Lesson 01 — Foundation Models Basics

## Why this lesson exists

You can call a chat API in ten minutes and feel productive. Then a prompt that worked yesterday fails today, a long document silently truncates, or the bill jumps because someone pastes entire PDFs into every request. This lesson builds the mental model underneath those APIs so later lessons on prompting, RAG, and evaluation make sense.

By the end you should be able to explain, in plain language: how next-token prediction becomes useful chat behavior; what tokens and context windows constrain; how embeddings differ from generative models; how to choose model size/speed; and what typical failure modes look like.

## What is a foundation model?

A **foundation model** is a large model trained on broad data that can be adapted to many tasks. **LLMs** (large language models) are foundation models specialized in language—and increasingly in multimodal inputs such as images or audio.

Instead of training a classifier from scratch for every task, you often:

1. Choose a base model (hosted API or open weights)
2. Adapt with prompts, retrieval, tools, or fine-tuning
3. Evaluate on *your* domain examples

**Why this pattern won for many products:** training a capable language model from scratch costs enormous data and compute. Adapting a shared base is cheaper and faster. The tradeoff is that you inherit the base model's strengths *and* blind spots (stale knowledge, uneven skills, safety quirks). Engineering discipline—prompts, RAG, tools, eval—is how you turn a general model into a reliable feature.

### Generative vs discriminative (quick contrast)

| Kind | Rough job | Example |
|------|-----------|---------|
| Discriminative | Map input → label/score | Spam classifier, ranking model |
| Generative (LM) | Model a distribution over sequences; sample or complete text | Chat assistants, summarizers, code helpers |

An LLM can *act like* a classifier if you prompt it carefully and constrain outputs—but under the hood it is still predicting likely next tokens, not running a dedicated classification head unless you add one (fine-tuning / custom heads). That distinction matters when you care about calibrated probabilities or ultra-stable labels.

## How next-token prediction becomes chat behavior

### The training game

At the core, many LLMs are trained to predict the **next token** given previous tokens. Over huge corpora, that simple objective forces the model to internalize grammar, facts that appear often, reasoning patterns that show up in text, and stylistic conventions.

Intuition:

```text
Context: "The capital of France is"
Model assigns high probability to tokens like " Paris"
```

That is not a database lookup. It is a statistical continuation. Sometimes the continuation is correct and useful; sometimes it is fluent nonsense (**hallucination**).

### From completion to chat

Raw “complete this text” models are awkward as products. Providers wrap them with:

1. **Special formatting** so dialogue turns are marked (system / user / assistant)
2. **Instruction tuning** so the model follows requests more reliably
3. Often **preference / alignment training** so answers better match what humans rate as helpful and safe

When you call a chat API, you are not talking to a person. You are feeding a carefully formatted token sequence into a next-token engine that has been steered toward assistant-like behavior.

**Why you should care:** small changes in message roles, wording, or examples can change the continuation a lot. Prompting is not magic—it is steering a probabilistic sequence model.

### Sampling vs “the answer”

At each step the model produces a distribution over the next token. Decoding strategies (greedy, temperature sampling, top_p) pick from that distribution. So:

- There is often no single “the” answer—especially for open tasks
- Lower temperature and stricter decoding make outputs more repeatable (good for extraction)
- Higher temperature increases diversity (good for brainstorming, riskier for facts)

Lesson 02 covers prompting styles; this lesson only needs you to remember: **chat is sampled continuation under constraints**.

## Tokens

Models do not read characters the way humans do. Text is broken into **tokens** (often subword pieces).

### Why tokenization exists

A fixed vocabulary of subwords balances:

- Coverage of rare words via pieces
- Manageable vocabulary size
- Reasonable sequence lengths

Quirk examples you will meet in practice:

- Spaces and punctuation can attach to tokens in surprising ways
- Numbers may split oddly (`2026` vs digit-by-digit depending on tokenizer)
- Code and non-English text can use more tokens than similar-looking English

### Why tokens matter to engineers

| Concern | Why tokens matter |
|---------|-------------------|
| Cost | APIs are usually priced per input/output token |
| Context limits | Windows are measured in tokens, not pages |
| Latency | More tokens to process → more compute |
| Truncation bugs | Over-limit inputs get cut—often silently from the middle or end depending on client code |
| Evaluation | “Same paragraph” ≠ same token count across models |

Rough English rule of thumb historically used: ~0.75 words per token. **Only an estimate.** Measure with the provider’s tokenizer when cost or limits matter.

### Mini practice

Paste the same 200-word paragraph into two different tokenizers (or two model families’ token counters). Note the counts. Ask: would a hard 4k context budget fit a 3,000-word policy doc plus instructions for both?

## Context windows

The **context window** is the maximum token budget the model can consider for a request (input messages, and often coupled with limits on output length).

### What fits in the window

```text
[system instructions] + [conversation history] + [retrieved docs] + [user question] + [room for answer]
```

Everything competes for the same budget. That is why RAG exists: you cannot always paste the whole company wiki.

### Long context is not free

Newer models advertise large windows (tens or hundreds of thousands of tokens). Still:

- Cost scales with tokens processed
- Latency often grows with input size
- Models can pay uneven attention across very long contexts (“lost in the middle” style failures—important facts buried mid-prompt get missed)
- Stuffing more text can *hurt* answer quality if noise dilutes the signal

**Engineering habit:** retrieve less, better; summarize history; keep system prompts tight; measure quality vs length, not length alone.

## Embeddings vs generative models

An **embedding** is a vector representation of text (or other media) such that similar meanings tend to land nearby in vector space.

### What embeddings are for

- Semantic search / RAG retrieval
- Clustering and near-duplicate detection
- Ranking and some recommendation features
- Guarding pipelines (for example, similarity to known-bad prompts)—advanced use

### How they differ from chat models

| | Embedding model | Generative / chat model |
|--|-----------------|-------------------------|
| Output | Fixed-size vector | Token sequence (text) |
| Typical use | Similarity, search | Answer, write, reason, tool calls |
| Training focus | Represent meaning for comparison | Predict / generate text |
| You usually | Compare with cosine/dot product | Sample or constrain decoding |

You often use **both**: embed to find relevant chunks, then generate an answer conditioned on those chunks (Lesson 03).

**Why separate models?** Embedding models are optimized and priced for representation. Chat models are optimized for generation. Using a chat model as a makeshift embedder (for example, “rate similarity 1–10”) is usually worse and more expensive than a real embedding API or local embedding model.

## Chat APIs: messages you will send

Modern LLMs are commonly accessed via a **messages** API:

```text
system: who you are / hard rules / output format
user: the request (and often untrusted content)
assistant: prior model replies (for multi-turn)
```

Conceptual client sketch (provider APIs differ—check current docs):

```python
# Pseudocode — adapt to your SDK
response = client.chat.completions.create(
    model="your-model-name",
    messages=[
        {
            "role": "system",
            "content": (
                "You are a careful assistant for internal docs Q&A. "
                "If unsure, say you do not know. "
                "Answer in short paragraphs."
            ),
        },
        {"role": "user", "content": "Summarize this in 3 bullets:\n" + text},
    ],
    temperature=0.2,
    max_tokens=400,
)
```

### What belongs where (preview)

Lesson 02 goes deep on roles. Seed intuition now:

- **System:** durable policy, role, output contract—not one-off user questions
- **User:** the task and data for this turn
- **Assistant:** previous model outputs you intentionally keep in history

Do not put secrets (API keys, private tokens) in any message you would not log. Assume prompts may be logged for debugging.

## Temperature and related decoding knobs

| Setting | Why it exists | Typical use |
|---------|---------------|-------------|
| **Temperature → 0** | Sharpens toward high-probability tokens; more deterministic | Extraction, classification-like tasks, code that must parse |
| **Higher temperature** | Flattens distribution; more diverse | Brainstorming, creative drafts |
| **top_p** (nucleus) | Sample only from smallest set of tokens whose cumulative probability ≥ p | Fine control on diversity; often leave default unless tuning |
| **max tokens** | Cap output length / cost | Always set deliberately in products |
| **stop sequences** | Halt generation when a marker appears | Delimited formats, turning control back to your code |

**How to choose (starter map):**

| Task type | Temperature starting point | Notes |
|-----------|----------------------------|-------|
| JSON extraction | 0–0.2 | Validate schema in code |
| Classification labels | 0–0.2 | Prefer constrained outputs |
| RAG factual answer | 0–0.3 | Ground in context; cite |
| Tutoring explanation | 0.3–0.7 | Clarity over novelty |
| Brainstorming names | 0.7–1.0 | Then filter with a second pass |

Change **one** knob at a time when debugging. Lesson 02 expands this into task playbooks.

## Cost and latency intuition

### What drives cost

- Input tokens (system + history + retrieved context + user)
- Output tokens (often priced higher per token than input on many APIs)
- Model tier (larger / more capable models cost more)
- Retries, tool loops, and multi-step agents multiply spend

### What drives latency

- Model size and provider load
- Input length (prefill)
- Output length (decode is sequential)
- Tool calls and network round trips
- Your own validation / repair loops

### Practical habits

1. Prefer the **smallest model that passes your golden set** for that task
2. Keep retrieved context lean (rerank, top-k carefully)
3. Cap `max_tokens` to what the UI actually needs
4. Cache stable system prefixes when the provider supports prompt caching
5. Measure on *your* traffic shapes—not blog benchmarks alone

```python
# Tiny cost sketch (illustrative numbers only — replace with your price sheet)
def estimate_cost(input_tokens, output_tokens, in_price_per_mtok, out_price_per_mtok):
    return (
        input_tokens / 1_000_000 * in_price_per_mtok
        + output_tokens / 1_000_000 * out_price_per_mtok
    )
```

## Choosing model size and speed

Think in **product requirements**, not prestige.

| Need | Lean toward |
|------|-------------|
| High volume, simple classify/extract | Smaller / faster model, strict schema |
| Hard reasoning, messy instructions | Larger model, still with eval |
| Private docs answers | Medium+ model **plus RAG**, not “biggest model alone” |
| Offline / air-gapped | Open weights you can run; accept hardware limits |
| Burst creative drafts | Larger or mid model at higher temperature, human edit |

**Routing pattern:** many teams send easy traffic to a cheap model and escalate hard cases to a stronger one—based on classifiers, confidence heuristics, or user tiers. Routing only works if you evaluate both paths.

## Failure modes you should expect

| Failure | What you see | Why it happens | First responses |
|---------|--------------|----------------|-----------------|
| Hallucination | Fluent wrong facts | Model predicts plausible tokens, not truth | RAG/tools; ask to abstain; cite; eval |
| Stale knowledge | Outdated APIs/policies | Training cutoff / no live retrieval | RAG, tools, dated system notes |
| Brittleness | Tiny prompt tweak flips behavior | Sensitivity to phrasing and examples | Few-shot; golden regressions; templates |
| Truncation | Answers cut off mid-thought | Hit max tokens or context | Raise carefully; shorten prompts; summarize |
| Format drift | Almost-JSON, missing fields | Generative freedom | JSON mode/schema; validate; repair limitedly |
| Over-long context noise | Misses the key sentence | Dilution / attention limits | Better retrieval; structure; reorder |
| Prompt injection | User/doc content steers the model | Untrusted text in context | Delimiters; don’t treat prompts as security; Part 5 |
| Cost blowups | Bill spike | Huge contexts, loops, wrong model tier | Caps, caching, routing, monitoring |

## Structured output (preview)

You will go deeper in Lessons 02 and 04. Seed the habit now:

1. Ask for a schema (JSON / constrained fields)
2. Parse and **validate** in code
3. Treat parse failure as a failed attempt, not creative freedom

```python
import json

def parse_json_answer(raw: str) -> dict:
    data = json.loads(raw)  # wrap in try/except in real code
    required = {"title", "severity", "summary"}
    missing = required - data.keys()
    if missing:
        raise ValueError(f"missing keys: {missing}")
    return data
```

## Putting it together: a first reliable call

```python
SYSTEM = """You extract fields from short support emails.
Return ONLY valid JSON with keys: intent, urgency, product.
intent must be one of: billing, bug, how_to, other.
urgency must be one of: low, medium, high.
If the email lacks evidence for a field, use null for that field.
"""

def classify_email(client, model: str, email_body: str) -> dict:
    resp = client.chat.completions.create(
        model=model,
        messages=[
            {"role": "system", "content": SYSTEM},
            {"role": "user", "content": email_body},
        ],
        temperature=0,
        max_tokens=200,
    )
    raw = resp.choices[0].message.content
    return parse_json_answer(raw)
```

Notice what this already encodes from this lesson: tight system contract, low temperature, capped tokens, validation in code. Prompting styles in Lesson 02 will make the *language* of that system message even stronger.



## Multimodal and tool-ready models (awareness)

Many “chat” models today accept more than text: images, sometimes audio, and structured **tool schemas**. For this fundamentals path:

- Treat multimodal input as **another context channel** that still consumes budget and can carry untrusted content (for example, text inside a screenshot)
- Treat tool schemas as **contracts your server must enforce** (Lesson 04)—the model proposing a tool call is not the same as the action being safe
- Do not assume every model supports the same message roles, JSON mode, or vision features; read the model card for the deployment you use

## Provider APIs vs open weights

| Path | Why teams choose it | What you still owe |
|------|---------------------|-------------------|
| Hosted API | Speed to product, ops handled by vendor | Eval, prompts, data handling, egress/privacy review |
| Open weights (self-host or rented GPUs) | Control, customization, sometimes cost at scale | Serving, scaling, upgrades, security patching |
| Hybrid | Sensitive retrieval local + generation via API (or reverse) | Clear trust boundaries and logging |

Fundamentals stay the same: tokens in, tokens out, stochastic decoding, need for eval. Deployment choice changes ops and compliance more than it changes prompting theory.

## Conversation state and memory

Multi-turn chat means you resend (or summarize) history each request unless the product stores state server-side.

**Why this bites people:**

- History grows → cost and latency grow
- Old tool results and wrong turns poison later answers
- “Memory” features are usually **your** storage + retrieval, not magical model memory

Patterns:

1. **Sliding window:** keep last N turns
2. **Summarize older turns** into a compact system/developer note
3. **Structured state:** store slots (order_id, locale) in your DB; inject only what the turn needs
4. **RAG over past tickets** instead of raw infinite chat logs

```python
def trim_messages(messages, max_turns=8):
    """Keep system message + last max_turns user/assistant pairs (simplified)."""
    system = [m for m in messages if m["role"] == "system"][:1]
    rest = [m for m in messages if m["role"] != "system"]
    return system + rest[-(max_turns * 2) :]
```

## Determinism, seeds, and reproducibility

Even at temperature 0, hosted APIs may not guarantee bit-identical outputs across time (batching, backend changes, sparse MoE routing, etc.). For engineering:

- Aim for **stable enough** behavior on golden sets, not perfect replay
- Pin **model version** strings when the provider offers them
- Log raw prompts and outputs for failed cases
- Prefer schema validation over matching exact prose

## Safety-aware basics (before Part 5)

Even in fundamentals:

- Assume **user content and retrieved docs are untrusted**
- Never put API keys or credentials in prompts
- Do not treat “the model refused” as a security control for authorization
- Log thoughtfully: prompts may contain PII—apply retention and access rules early

Full injection, jailbreaks, and guardrails arrive in Part 5; Lesson 02 teaches delimiters and injection *awareness*.

## Debugging playbook: “the model is being weird”

Work top-down:

1. **Confirm the exact messages** sent (logging). Many bugs are wrong concatenation or missing system text in one code path.
2. **Check truncation** (context overflow, max_tokens).
3. **Simplify:** remove history and tools; reproduce with one user message.
4. **Lower temperature** for reliability tasks; see if format stabilizes.
5. **Swap model size** once—if a small model fails and a large one passes, you may have a capability gap, not a prompt typo.
6. **Add one example** (few-shot) if format is the issue.
7. **Add retrieval or a tool** if facts are the issue—do not keep yelling at the prompt to know private data.
8. **Write a golden case** so the bug cannot return unnoticed.

## Worked example: cost of a naive support bot

Suppose:

- System prompt: 800 tokens
- Average history: 1,200 tokens
- Retrieved chunks: 2,500 tokens
- User message: 150 tokens
- Answer: 400 tokens
- Price sheet (illustrative): $0.50 / 1M input tokens, $1.50 / 1M output tokens
- Traffic: 100,000 requests/day

```text
Input tokens/request  ≈ 800+1200+2500+150 = 4,650
Output tokens/request ≈ 400

Daily input  = 100e3 * 4650 = 4.65e8 tokens → ~$232.50
Daily output = 100e3 * 400  = 4.0e7  tokens → ~$60.00
Daily ≈ $292  (illustrative only)
```

Levers that matter: shrink retrieval with reranking, summarize history, shorter system text, smaller model for easy intents, cache repeated policy chunks when available. **Architecture and prompting are cost features**, not just quality features.

## Anti-patterns at the foundation layer

| Anti-pattern | Why it hurts | Prefer |
|--------------|--------------|--------|
| Pasting entire corpora into every prompt | Cost, latency, dilution | RAG / selective context |
| One mega-model for every intent | Spend and latency | Route by task difficulty |
| Ignoring tokenizer differences | Silent overflow | Measure tokens per model |
| Treating chat history as infinite memory | Drift and cost | Explicit state + trim/summarize |
| No output cap | Runaway bills and timeouts | Always set max tokens in products |
| Skipping validation because “it usually returns JSON” | Parse bombs in prod | Schema validate every time |

## Practice: design brief

Write a one-page design for an internal **“explain this error log”** helper:

1. Which model tier would you try first and why?
2. What goes in system vs user messages?
3. Temperature and max tokens starting points?
4. Do you need embeddings/RAG on runbooks on day one?
5. Three failure modes and how you would detect them in logs?

Keep the brief concrete enough that a teammate could implement a prototype in a day.


## Key vocabulary

- foundation model, LLM
- next-token prediction, decoding / sampling
- token, tokenizer, context window
- embedding, vector similarity
- system / user / assistant messages
- temperature, top_p, max tokens, stop sequences
- hallucination, prompt injection (awareness)
- structured output / schema validation

## Mini practice set

1. Explain to a backend engineer, in five sentences, why a chat assistant can invent a plausible package version number.
2. You have a 12k-token policy PDF and a model with an 8k context window. List three strategies (without naming a specific vendor product) to still answer questions about it.
3. For ticket classification at 50 requests/second, would you default to the largest chat model available? Why or why not?
4. Name two reasons embeddings and chat models are usually separate services.

## What is next

**[02-prompting-deep-dive.md](../../05-prompt-and-context-engineering/learn/02-prompting-deep-dive.md)** — the centerpiece of Part 4: what a prompt really is, anatomy of strong prompts, message roles, every major prompting style with when/why/how, task playbooks, parameters, iteration habits, and anti-patterns.
