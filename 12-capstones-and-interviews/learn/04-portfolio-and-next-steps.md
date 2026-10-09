# Portfolio Packaging and Next Steps

You built skills and (ideally) a capstone. This lesson is about **showing the work** and continuing to learn without drowning.

## What a strong AI engineering portfolio entry includes

| Piece | Purpose |
|-------|---------|
| Clear problem statement | Shows product sense |
| Architecture diagram | Shows system thinking |
| How to run | Shows empathy for other engineers |
| Metrics + evaluation | Shows honesty |
| Limitations | Shows seniority |
| Short demo (GIF, asciinema, or screenshots) | Shows it works |
| Your role / decisions | Shows ownership |

Avoid walls of notebooks with no narrative.

## README outline (copy and adapt)

```markdown
# Project title

One-sentence value prop.

## Demo
Screenshot or short terminal session.

## Problem
Who hurts if this fails?

## Approach
Bullets + diagram.

## Results
Table of metrics vs baseline.

## How to run
Prereqs, install, commands.

## Limitations
Known failure modes.

## License / data rights
```

## Talking about LLMs without hype

Good:

- “Retrieval hit@5 is 0.72 on our 25-question golden set; failures cluster in multi-section policies.”

Bad:

- “Uses advanced AI to revolutionize documentation with 99% accuracy.”

Prefer measured claims you can defend.

## GitHub hygiene

- `.gitignore` for `.venv/`, keys, large artifacts
- Never commit API keys; use `.env.example`
- Small sample data in-repo; scripts for full downloads
- Tag a release of the MVP for a frozen demo

## Interview / stakeholder narrative (5 minutes)

1. Problem and user (30s)
2. Constraints (data, latency, cost) (30s)
3. Architecture (90s)
4. One hard bug you fixed (60s)
5. Metrics and what you would do next (60s)

Rehearse aloud.

## What to learn next (reading list style)

No fake URLs—seek the official or well-known resources by name:

### Core references

- **scikit-learn** user guide (pipelines, model selection, metrics)
- **PyTorch** tutorials and docs (tensors, training, torchvision models)
- **Hugging Face** course and documentation (transformers, datasets, PEFT concepts)
- Original **Attention Is All You Need** paper (Transformer foundations)
- Familiar survey material on **RAG** from major provider architecture blogs and academic surveys (read critically; verify against your evals)

### Engineering depth

- FastAPI documentation
- Twelve-Factor App ideas applied to model services
- Your cloud provider’s guidance on secrets management and observability
- OWASP materials relevant to LLM apps / prompt injection discussions (evolving space—verify dates)

### Practice habits

- Rebuild one paper or blog idea *small* and evaluate it yourself
- Keep a personal golden-set per project
- Write incident-style postmortems for your own demos when they fail

### Optional specialization paths

| Path | Next focus |
|------|------------|
| Classical ML / decision science | Causal inference caution, calibration, uplift basics |
| Deep learning systems | Mixed precision, distributed training awareness, compilation |
| LLM apps | Evaluation harnesses, hybrid search, prompt/version CI |
| MLOps | Feature stores, advanced monitoring, platform tooling |
| Safety | Red-teaming practice, policy design with legal/compliance partners |

## Maintaining momentum

After the capstone:

1. Schedule a weekly 90-minute block for deliberate practice
2. Alternate *build* weeks and *read* weeks
3. Revisit Part 1–2 when tabular problems appear—do not force deep learning everywhere
4. Prefer finishing unfinished projects over starting new ones

## Closing

AI engineering is less about memorizing every new model name and more about **data honesty, evaluation, and reliable delivery**. If you carried those habits through Parts 1–6, you have the foundation to keep growing as tools change.

Return to earlier parts whenever a concept feels shaky—the path is designed for revisiting.
