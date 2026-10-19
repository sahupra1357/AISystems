# Prompt engineering → Zero-shot and few-shot prompting

## Video takeaway

![Zero-shot vs one-shot — the same review classified with no example, then with one example](demo/zero-few-shot-demo.gif)

*([full-resolution MP4 with audio](demo/zero-few-shot-demo.mp4) — 29s · [watch on YouTube](https://youtu.be/iBipBulxNg0))*

"Zero-shot: tell the model what to do. Few-shot: show it three or four worked examples and it copies the pattern — the exact format, how to handle 'not found', where the boundary is between two labels. Use real-looking examples from your failures, balance the labels, shuffle the order once because models over-predict the last example's label, and always include one 'no answer' case. Put them in the cached part of the prompt so they cost nothing per call. And still validate the output — examples improve the odds, they don't guarantee the format."

## What it is

**Zero-shot** — instruction only. Works well for tasks modern models already understand (summarise, translate, classify into common labels, answer from context). Cheapest in tokens; least control over format and edge cases.

**Few-shot** — instruction + examples of input → ideal output. The model infers the *pattern*: exact output format, level of detail, how to handle "not found", how to phrase refusals, label boundaries for a custom taxonomy. It is the fastest way to move behaviour without training. Examples are prompt real estate: 3 good examples beat 10 mediocre ones, and each costs tokens on every call (cacheable if static, [prompt caching of static prefixes](../2-context-engineering/prompt_caching.md)).

Rules of thumb:
- Format/structure → 1–3 examples.
- Custom classification taxonomy → one example per label, including hard/near-miss cases.
- Extraction with tricky boundaries → examples that show what *not* to extract.
- Include at least one **negative / abstain** example ("input has no answer → output: not found").

## How to implement

1. **Start zero-shot** with a clear task statement and output format ([prompt structure](prompt_structure.md)). Evaluate ([prompt evaluation](../../../09-evaluation/apply/prompt_evaluation.md)).
2. **Collect failures** from the golden set; each failure type becomes a candidate example.
3. **Write examples as real cases**, not toy ones; keep the input realistic in length and noise. Vary surface features (names, numbers, phrasing) so the model doesn't copy specifics.
4. **Balance labels** in classification examples; order them randomly (models show recency/position bias — the last example's label is over-predicted).
5. **Delimit examples** (`<example><input>…</input><output>…</output></example>`) in the stable block so they cache ([prompt structure](prompt_structure.md), [prompt caching of static prefixes](../2-context-engineering/prompt_caching.md)).
6. **Evaluate** each added example set on the golden set; stop when the marginal gain flattens. Watch token cost.
7. **Version** examples with the prompt ([prompt versioning & evaluation](prompt_versioning_evaluation.md)); they are part of the prompt's behaviour.

## Small example

```python
SYSTEM = """You classify internal support tickets into exactly one label: BILLING, ACCESS, BUG, HOW_TO, OTHER.
Output the label only.

<examples>
<example><input>I was charged twice this month for the Pro plan.</input><output>BILLING</output></example>
<example><input>Can't log in since the SSO change, keeps redirecting.</input><output>ACCESS</output></example>
<example><input>Export to CSV produces an empty file when filters are applied.</input><output>BUG</output></example>
<example><input>How do I share a dashboard with someone outside my team?</input><output>HOW_TO</output></example>
<example><input>The export takes forever — is that expected or a bug? Also how do I invite a contractor?</input><output>OTHER</output></example>
<example><input>Invoice shows the wrong company address.</input><output>BILLING</output></example>
</examples>
Notes: a ticket with two distinct requests is OTHER. A question about whether behaviour is expected is HOW_TO unless it reports an error."""

def classify(ticket: str) -> str:
    r = llm("intent_classifier", SYSTEM, [{"role": "user", "content": f"<input>{ticket}</input>"}], max_tokens=5)
    label = r.content[0].text.strip().upper()
    return label if label in {"BILLING", "ACCESS", "BUG", "HOW_TO", "OTHER"} else "OTHER"   # validate + default
```

## Pros

- **Zero-shot**: minimal tokens, fastest to write, no example bias.
- **Few-shot**: precise control of format and edge cases; big accuracy gains on custom taxonomies and extraction; no training needed; cached examples cost little per call.

## Cons

- **Zero-shot**: format drift, inconsistent edge-case handling, weaker on custom labels.
- **Few-shot**: examples bias the model (label order, surface features); tokens on every call (mitigated by caching); examples go stale as the task evolves; too many examples can *reduce* accuracy (over-anchoring); leaking example content into outputs.

## When to use

- **Zero-shot**: common tasks with strong models; when output is validated by schema anyway ([structured outputs & tool use](structured_outputs_tool_use.md)); latency/token-critical paths.
- **Few-shot**: custom labels, house formats, extraction with subtle boundaries, refusal phrasing, anything where the golden set shows format or boundary errors.

## When NOT to use

- Few-shot for **knowledge** — examples don't teach facts; use RAG ([prompting vs RAG vs fine-tuning](../../../08-choosing-the-lever/apply/prompting_vs_rag_vs_finetuning.md)).
- Few-shot when examples must be tenant-specific and there are thousands of tenants — use dynamic retrieval of examples ([dynamic few-shot selection](dynamic_few_shot_selection.md)) or fine-tuning ([SFT](../../../08-choosing-the-lever/apply/fine-tuning/sft.md)).
- Piling on examples to fix a fundamentally ambiguous task definition — fix the definition.

## Production at scale

**Examples are cached prompt tokens** — put them in the stable block ([prompt structure](prompt_structure.md)). Cost per call is then small; the cost that matters is on cache misses and in latency for very long example sets. Track stable-block tokens.

**Position and recency bias** are real in classification; randomise example order once (fixed seed, then frozen for caching) and verify on the golden set that no label is systematically over-predicted.

**Example provenance and privacy.** Examples drawn from real tickets/documents may contain PII or tenant data. Scrub and synthesise; never put one tenant's data in a prompt used for another tenant ([PII](../../../10-production/apply/6-security-privacy-governance/pii.md)).

**Validate outputs anyway.** Few-shot improves format adherence; it doesn't guarantee it. Schema validation + retry ([structured outputs & tool use](structured_outputs_tool_use.md)) remains mandatory.

**Model upgrades change how much few-shot helps.** Newer models often need fewer examples; re-tune on upgrade ([versioning & rollout](../../../10-production/apply/1-serving-cost-latency/versioning_rollout.md)) — dropping examples reduces tokens.

**Drift.** Taxonomies evolve; stale examples teach old labels. Tie examples to the taxonomy version; review examples in the same PR that changes labels.

**A/B example sets** with the eval harness; keep the set that wins on the golden set *and* on token cost.

## Prod checklist

- [ ] Zero-shot baseline evaluated before adding examples
- [ ] Examples derived from golden-set failures; realistic, varied, balanced; includes abstain/negative
- [ ] Examples in the cached stable block; order fixed after randomisation
- [ ] PII-scrubbed / synthetic examples; no cross-tenant data
- [ ] Output validation + fallback regardless of examples
- [ ] Examples versioned with the prompt and taxonomy
- [ ] Re-tuned on model upgrade

## Metrics to watch

| Metric | Why |
|---|---|
| Golden-set accuracy zero-shot vs few-shot (per label) | Value of examples |
| Per-label prediction skew vs truth | Position/recency bias |
| Stable-block tokens | Cost |
| Format-violation rate | Adherence |

## Related

- [Prompt structure](prompt_structure.md) — where examples live
- [Dynamic few-shot](dynamic_few_shot_selection.md) — retrieve examples per query
- [Structured outputs](structured_outputs_tool_use.md) — validation
- [Prompt evaluation](../../../09-evaluation/apply/prompt_evaluation.md)
- [SFT](../../../08-choosing-the-lever/apply/fine-tuning/sft.md) — when examples outgrow the prompt
