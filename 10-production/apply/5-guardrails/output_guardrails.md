# Testing & reliability → Output guardrails

## Video takeaway

"Output guardrails are the last thing between a bad answer and a user. Run the free checks on everything: is the schema valid, do the cited IDs actually exist, is there an API key in the text, and — the one people forget — is this output about to be rendered as HTML or interpolated into SQL, because that's an injection vector no matter how well-behaved the model was. Then tier the expensive checks by risk: faithfulness and toxicity on external-facing answers, deterministic only internally. Define the action per violation — redact, regenerate once, or block with an honest message — and treat rising violation rates as a symptom to fix upstream, not just a block that worked."

## What it is

Output guardrails inspect the generated response before delivery:

| Check | Catches | Cost | Action |
|---|---|---|---|
| **Schema validation** | Malformed structured output | Free | Retry with error fed back ([structured outputs & tool use](../../../05-prompt-and-context-engineering/apply/1-prompt-engineering/structured_outputs_tool_use.md)) |
| **Citation validation** | Fabricated citations | Free | Drop claim / regenerate ([citations](../../../06-rag/apply/8-generation-grounding/citations.md)) |
| **Faithfulness** | Ungrounded claims | Low (NLI) | Remove, regenerate, or abstain ([faithfulness check](../../../06-rag/apply/8-generation-grounding/faithfulness_check.md)) |
| **PII leakage** | Personal data from context reaching the wrong user | Low | Redact or block |
| **Banned claims / policy** | Promises, guarantees, legal/medical/financial advice the product must not give | Low (patterns + classifier) | Block or rewrite |
| **Toxicity/tone** | Offensive or off-brand output | Low | Regenerate |
| **Secret leakage** | Keys/credentials from retrieved content or tool output | Free (regex) | Redact, alert |
| **Insecure output handling** | Model output rendered as HTML/SQL/shell downstream | Free (encoding) | Escape/parameterise ([threat checklist](../6-security-privacy-governance/owasp_llm_top10.md)) |

Two categories with very different economics: **deterministic checks** (schema, citation IDs, secrets, encoding) are free and should run on 100% of output; **model-based checks** (toxicity, policy claims, faithfulness) cost latency and should be tiered by risk.

The most under-appreciated one is **insecure output handling** — model output interpolated into HTML, SQL, or a shell command is an injection vector regardless of how well-behaved the model is (OWASP LLM Top 10, [threat checklist](../6-security-privacy-governance/owasp_llm_top10.md)).

## How to implement

1. **Deterministic first, on everything**: schema, citation IDs against the retrieved set, secret patterns, and output encoding for the destination context.
2. **Faithfulness gate** with a defined action policy: contradicted claims never ship ([faithfulness check](../../../06-rag/apply/8-generation-grounding/faithfulness_check.md)).
3. **PII leakage check**: scan output for entities *not present in the user's own input or permitted context* — this catches context bleed and cross-tenant leakage.
4. **Banned-claim checks** as a domain-specific list: guarantees ("this will definitely"), unauthorised advice, competitor mentions, pricing commitments. Patterns plus a small classifier.
5. **Tiered by risk**: full checks on external-facing and high-stakes outputs; deterministic-only on internal, low-stakes paths.
6. **Actions defined per check**: redact, regenerate once, block with a safe fallback, or escalate to human ([human-in-the-loop](../../../07-agents-tools-memory/apply/1-agents/human_in_the_loop.md)). Never deliver silently modified output without a note where it matters.
7. **Log everything**, including what was redacted or blocked, with the trace ID ([observability & tracing](../4-observability-reliability/observability_tracing.md)).

## Small example

```python
SECRET_PATTERNS = re.compile(r"(sk-[A-Za-z0-9]{20,}|AKIA[0-9A-Z]{16}|password\s*[:=]\s*\S+|Bearer\s+[A-Za-z0-9._-]{20,})", re.I)
BANNED_CLAIMS = [
    (re.compile(r"\b(guarantee|guaranteed|will definitely|100%\s+(safe|certain|accurate))\b", re.I), "absolute_guarantee"),
    (re.compile(r"\byou should (invest|sue|take|stop taking)\b", re.I), "unauthorised_advice"),
    (re.compile(r"\b(refund|credit) (will|shall) be (issued|processed)\b", re.I), "financial_commitment"),
]

class OutputVerdict(BaseModel):
    deliver: bool
    text: str
    redactions: list[str] = []
    violations: list[str] = []
    action: str = "deliver"          # deliver | redacted | regenerate | block | escalate

def output_guardrails(output: dict, retrieved: list[dict], caller, policy, risk: str = "high") -> OutputVerdict:
    text = output.get("answer", "")
    violations, redactions = [], []

    # --- deterministic, always ---
    if not output.get("schema_valid", True):
        return OutputVerdict(deliver=False, text=text, violations=["schema"], action="regenerate")

    valid_ids = {r["chunk_id"] for r in retrieved}
    bad_cites = [c for cl in output.get("claims", []) for c in cl.get("citations", []) if c not in valid_ids]
    if bad_cites:
        violations.append("invalid_citation")                              # Citations

    if SECRET_PATTERNS.search(text):
        text = SECRET_PATTERNS.sub("<REDACTED>", text)
        redactions.append("secret"); alert_secret_in_output(caller, output.get("trace_id"))

    # --- PII leakage: entities in the output that the caller shouldn't be seeing ---
    out_entities = {(f.entity_type, text[f.start:f.end]) for f in analyzer.analyze(text=text, language="en",
                                                                                   entities=policy["pii_entities"])}
    permitted = permitted_entities(caller, retrieved)                       # from context the caller may read
    leaked = [e for e in out_entities if e[1] not in permitted]
    if leaked:
        text = redact_entities(text, [e[1] for e in leaked])
        redactions.append("pii"); violations.append("pii_leak")
        alert_pii_leak(caller, [e[0] for e in leaked])                      # entity TYPES only in logs (PII)

    # --- banned claims ---
    for pattern, label in BANNED_CLAIMS:
        if pattern.search(text):
            violations.append(f"banned_claim:{label}")

    # --- model-based, risk-tiered ---
    if risk == "high":
        faith = faithfulness_gate(output, retrieved)                        # Faithfulness check
        if faith.get("contradicted"):
            return OutputVerdict(deliver=False, text=text, violations=violations + ["contradicted_claim"],
                                 action="regenerate")
        if toxicity_score(text) > 0.5:
            return OutputVerdict(deliver=False, text=text, violations=violations + ["toxicity"], action="regenerate")

    if any(v.startswith("banned_claim") for v in violations):
        return OutputVerdict(deliver=False, text=text, violations=violations, action="block")

    return OutputVerdict(deliver=True, text=text, redactions=redactions, violations=violations,
                         action="redacted" if redactions else "deliver")

# --- insecure output handling: encode for the destination, always ---
def render_for_html(text: str) -> str:  return html.escape(text)            # never innerHTML raw model output
def use_in_sql(value: str, conn):       return conn.execute("SELECT ... WHERE x = %s", (value,))   # parameterised
def use_in_shell(value: str):           return shlex.quote(value)           # never string-interpolate

def deliver(output, retrieved, caller, policy, risk):
    v = output_guardrails(output, retrieved, caller, policy, risk)
    log_output_guardrail(caller, v.action, v.violations, v.redactions)
    if v.action == "regenerate":
        return regenerate_once(output, v.violations) or safe_fallback(v.violations)
    if v.action == "block":
        return safe_fallback(v.violations)                                  # specific, honest message
    return v.text
```

## Pros

- **Last line of defence** — catches what prompting, grounding, and retrieval controls missed.
- **Deterministic checks are free** and catch the most damaging failures (fabricated citations, leaked secrets).
- **PII leakage detection catches context bleed** that no upstream control sees.
- **Output encoding prevents a whole OWASP category** at zero cost.
- Violation telemetry is a direct measure of upstream quality problems.

## Cons

- **Latency on every response**, and it's at the worst moment — the user is waiting.
- **Streaming conflicts**: guardrails on a streamed response either delay the stream or require post-hoc correction (visible to the user).
- **False positives block legitimate output**, which is highly visible.
- **Model-based checks are imperfect** and add cost; toxicity classifiers over-flag legitimate domain language (medical, legal, security).
- Regeneration doubles cost for affected responses.

## When to use

- Deterministic checks: always, on everything.
- Model-based checks: external-facing, high-stakes, regulated, or anywhere the cost of a bad output exceeds the cost of checking.

## When NOT to use

- Heavy model-based checks on internal, low-stakes, latency-sensitive paths — deterministic only.
- Toxicity classifiers on domains where the vocabulary triggers them constantly (security, medical) without domain tuning.
- Guardrails as a substitute for grounding and retrieval quality: they're the last defence, not the first.

## Production at scale

**Streaming is the hard problem.** Options: (a) buffer and check before streaming (adds full latency); (b) stream and check in windows, cutting the stream on violation (visible but fast); (c) stream to a "draft" state and commit after checks. Most products choose (b) with deterministic checks running on windows and model-based checks on the complete response before the final commit. Decide deliberately; it's a UX/safety trade-off.

**PII leakage detection needs a notion of "permitted"** — entities present in the user's own input or in context they're authorised to read. Comparing against that set is what turns a noisy PII scan into a real cross-tenant-leakage detector ([authentication & authorization](../6-security-privacy-governance/authn_authz_permission_filtered_retrieval.md)).

**Banned claims are domain-specific and legally sourced.** Get the list from legal/compliance, version it, and test it ([prompt evaluation](../../../09-evaluation/apply/prompt_evaluation.md)). This is one of the few places where a regex list is genuinely the right tool.

**Insecure output handling is the quiet one.** Model output rendered as HTML enables XSS; interpolated into SQL enables injection; passed to a shell enables RCE. Encode at every boundary as a matter of course, and test it ([threat checklist](../6-security-privacy-governance/owasp_llm_top10.md)).

**Violation telemetry drives upstream fixes**: a rising invalid-citation rate means a prompt or model regression; rising PII redactions mean a retrieval-scope problem. Treat guardrail hits as symptoms to investigate, not just as blocks that worked ([feedback capture & triage](../8-data-flywheel/feedback_capture_triage.md)).

**Regenerate once, then fall back.** A second regeneration rarely helps and doubles latency again. Fall back to a safe, honest message naming the limitation ([refusal / no-answer path](../../../06-rag/apply/8-generation-grounding/refusal_no_answer_path.md)).

**Tier by destination**: output going to another system (a ticket, an email, a database) needs stricter validation than output shown to the requesting user, because the audience is different and the blast radius is wider.

**Never silently alter high-stakes output.** If you redact, say so. A user acting on a silently modified answer is a worse outcome than a visible block.

## Prod checklist

- [ ] Deterministic checks on 100% of output: schema, citation IDs, secret patterns, output encoding
- [ ] PII leakage check against entities the caller is permitted to see
- [ ] Versioned banned-claim list sourced from legal/compliance, with tests
- [ ] Model-based checks (faithfulness, toxicity) tiered by risk and destination
- [ ] Defined action per violation: redact / regenerate once / block / escalate
- [ ] Streaming strategy chosen and documented; final check before commit
- [ ] Output encoded for every destination (HTML, SQL, shell, downstream API)
- [ ] Redactions disclosed to the user where material
- [ ] All violations logged with trace ID; alerts on secret and PII leakage
- [ ] Violation trends reviewed as upstream quality signals

## Metrics to watch

| Metric | Why |
|---|---|
| Violation rate by check | Upstream quality signal |
| Invalid-citation and PII-leak rates | The two that matter most |
| Regeneration rate and cost | Guardrail expense |
| False-positive rate (review sample) | Blocked legitimate output |
| Guardrail p95 latency | User-visible cost |

## Related

- [Input guardrails](input_guardrails.md) — the mirror image
- [Citations](../../../06-rag/apply/8-generation-grounding/citations.md), [Faithfulness](../../../06-rag/apply/8-generation-grounding/faithfulness_check.md) — the checks reused here
- [PII](../6-security-privacy-governance/pii.md), [OWASP](../6-security-privacy-governance/owasp_llm_top10.md) — insecure output handling
- [Structured outputs](../../../05-prompt-and-context-engineering/apply/1-prompt-engineering/structured_outputs_tool_use.md) — schema validation and retry
- [Triage](../8-data-flywheel/feedback_capture_triage.md) — violations as symptoms
