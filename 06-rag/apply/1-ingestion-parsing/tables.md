# Structure extraction → Tables (emit as HTML/Markdown, never flattened)

## Video takeaway

"Flatten a table to text and 'EMEA Q3 revenue' becomes a guess among forty numbers. Keep the grid: Markdown or HTML, header row repeated in every chunk, caption and section attached, cell values stored as data. Split big tables into row groups, never mid-row. Merge tables that continue across pages. Validate — if the Total row doesn't add up, the parser got a cell wrong, and that page goes to the retry ladder. And if the question is 'sum across regions', don't ask the LLM to do arithmetic; route it to code."

## What it is

"What was the EMEA revenue in Q3?" is a lookup: row = EMEA, column = Q3. A flattened table (`Region Q1 Q2 Q3 EMEA 3.8 4.0 4.2 APAC …`) makes the LLM guess which number belongs to which cell; a 30-column table makes guessing hopeless. Structure-preserving output keeps the grid:

```markdown
| Region | Q1 | Q2 | Q3 |
|---|---|---|---|
| EMEA | 3.8 | 4.0 | 4.2 |
| APAC | 2.1 | 2.4 | 2.6 |
```

or HTML with `rowspan`/`colspan` for merged cells. LLMs read both well; Markdown is compact, HTML is faithful for complex tables.

Sources of table structure: Docling's TableFormer (digital and OCR'd), PP-Structure's table model, MinerU, camelot/pdfplumber for ruled digital tables, native DOCX/XLSX/HTML tables (already structured — don't lose it).

## How to implement

1. **Get the cell grid** from the parser (`TableItem` in Docling: `export_to_dataframe()`, `export_to_html()`); for Office/HTML sources, read the native table.
2. **Normalise**: detect header rows (bold, first row, repeated on continuation pages), fill merged cells (propagate the value down/right or mark spans in HTML), strip footnote markers into a notes field, standardise numeric formats (keep original text too).
3. **Serialize twice**: `markdown` for indexing/LLM (compact) and `html` for display and for tables with spans; store the `cells` grid (JSON) for programmatic use and evals.
4. **Caption + context**: attach the caption ("Table 3: Revenue by region, $M") and the section breadcrumb; prepend to the chunk text.
5. **Chunking rule** ([layout / structure-aware](../2-chunking/layout_structure_aware.md)): a table is atomic. If it exceeds `max_tokens`, split into row groups of N rows, **repeat the header row** in every group, and give every group the caption + breadcrumb + `table_id` + `row_range`.
6. **Table summary chunk**: one LLM-generated sentence per table ("Quarterly revenue by region, FY2024, in $M; EMEA largest") indexed separately so "which table has X" queries hit ([contextual retrieval](../2-chunking/contextual_retrieval.md)).
7. **Validate**: rectangular grid (consistent column count), header present, numeric columns numeric, row/column totals reconcile when a "Total" row exists ([production per-page heuristics](validation_production_heuristics.md)).

## Small example

```python
import pandas as pd

def table_to_chunks(df: pd.DataFrame, caption: str, breadcrumb: str, table_id: str,
                    max_rows: int = 25) -> list[dict]:
    header_md = "| " + " | ".join(map(str, df.columns)) + " |\n|" + "---|" * len(df.columns)
    chunks = []
    for start in range(0, len(df), max_rows):
        part = df.iloc[start:start + max_rows]
        rows_md = "\n".join("| " + " | ".join(map(str, r)) + " |" for r in part.itertuples(index=False))
        text = f"[{breadcrumb}] {caption} (rows {start + 1}-{start + len(part)} of {len(df)})\n{header_md}\n{rows_md}"
        chunks.append({"type": "table", "table_id": table_id, "row_range": [start, start + len(part)],
                       "text": text, "html": part.to_html(index=False), "n_cols": len(df.columns)})
    return chunks

def validate_table(df: pd.DataFrame) -> list[str]:
    issues = []
    if df.shape[1] < 2: issues.append("too_few_columns")
    if df.columns.duplicated().any(): issues.append("duplicate_headers")
    if any(str(c).startswith("Unnamed") for c in df.columns): issues.append("missing_header")
    num = df.apply(pd.to_numeric, errors="coerce")
    for c in df.columns:
        frac = num[c].notna().mean()
        if 0.3 < frac < 0.8: issues.append(f"mixed_numeric:{c}")     # OCR noise or merged cells
    if "total" in " ".join(map(str, df.iloc[:, 0])).lower():
        body, total = num.iloc[:-1], num.iloc[-1]
        for c in num.columns[1:]:
            if pd.notna(total[c]) and abs(body[c].sum() - total[c]) > 0.01 * max(abs(total[c]), 1):
                issues.append(f"total_mismatch:{c}")
    return issues
```

## Pros

- **Table questions become answerable.** The [common failure modes](../../../AI_System.md#common-failure-modes-and-the-fix) fix for "table questions always wrong".
- **Header repetition** makes every row group self-describing.
- **Cell grid** enables exact-value lookup, SQL-style routing ([structured + unstructured hybrid](../7-advanced-rag/structured_unstructured_hybrid.md)), and evals on table-cell accuracy.
- HTML preserves merged cells; Markdown is cheap in tokens.

## Cons

- **Table structure models fail** on borderless, nested, or rotated tables; OCR'd tables carry cell-level errors.
- **Wide tables** (30+ columns) don't fit chunk budgets even as one row group; need column splitting or a per-row "key: value" transformation.
- **Token cost**: Markdown tables are verbose; a large table can be thousands of tokens.
- **Embedding quality of tables is mediocre** — the summary chunk and captions carry the retrieval load.
- Header detection on multi-level headers (grouped columns) is error-prone.

## When to use

- Any corpus with tables that users query: financial statements, price lists, spec sheets, clinical data, schedules, policy matrices.
- Always for native Office/HTML tables — it costs nothing to keep the structure.

## When NOT to use (the full treatment)

- Decorative or layout tables (HTML tables used for page layout): detect (one row, no header, mostly text) and treat as paragraphs.
- When the same data is available as a structured source (database, CSV): route table questions to SQL ([structured + unstructured hybrid](../7-advanced-rag/structured_unstructured_hybrid.md)) and index only the summary.

## Production at scale

**Two representations, one truth.** Store `cells` JSON as truth; derive Markdown/HTML on write. When the LLM answers from a table chunk, the citation should point at the table element and, ideally, the cell (row/col) — store `table_id` and row/col indices in chunk metadata ([citations](../8-generation-grounding/citations.md)).

**Wide tables.** Options: (a) row groups with header repeated (default); (b) **row-as-record** transformation: each row becomes "Region: EMEA; Q1: 3.8; Q2: 4.0; Q3: 4.2" — embeds better, filters well, costs more chunks; (c) column groups with the key column repeated. Pick per table shape (`n_cols > 12` → row-as-record).

**Multi-page tables.** Detect continuation (same column x-positions, "continued", header repeated) at reassembly ([reading order](reading_order.md)) and merge under one `table_id`. Without this, a 10-page table becomes 10 unrelated tables with 9 missing headers.

**Numeric normalisation.** Keep the original cell text (`"$4.2M"`, `"(1,234)"`) and a parsed numeric value (`4200000`, `-1234`) in `cells`. The parsed value drives validation and structured routing; the original drives display.

**Validation as a gate.** Failing tables (non-rectangular, mismatched totals, missing header) go to the retry ladder: second table model → VLM transcription ("transcribe this table as Markdown exactly") → human review for high-value sources. Log the failure reason; borderless tables from one template failing consistently is a signal to add a template rule.

**Table summary chunks** are an LLM cost per table; cache by cell-grid hash; batch backfill; version the prompt.

**Evals.** Golden set of table questions with expected cell values; metric: exact-value accuracy; break down by table type (ruled/borderless/OCR'd/native). Plus table-cell accuracy on the parsing golden sample ([test-time validation against a golden sample](validation_golden_sample.md): TEDS score for structure).

**Structured routing.** Many "RAG failures" on tables are aggregation questions ("total revenue across regions"). Detect and route to code execution or SQL over the extracted cells ([structured + unstructured hybrid](../7-advanced-rag/structured_unstructured_hybrid.md)) instead of expecting the LLM to add 40 numbers.

## Prod checklist

- [ ] Cell grid stored as truth; Markdown for index, HTML for display/spans
- [ ] Header detection incl. multi-level; header repeated in every row group
- [ ] Caption + breadcrumb prepended; table summary chunk generated and cached
- [ ] Wide-table strategy (row-as-record beyond N columns)
- [ ] Multi-page continuation merged under one `table_id`
- [ ] Original + parsed numeric values in cells
- [ ] Validation (rectangular, header, totals) → retry ladder
- [ ] Golden table-question set; TEDS on parsing golden sample
- [ ] Aggregation questions routed to SQL/code, not to the LLM's arithmetic

## Metrics to watch

| Metric | Why |
|---|---|
| Table-cell accuracy / TEDS on golden sample per parser version | Structure quality |
| Table-question exact-value accuracy | End-to-end |
| Table validation failure rate by source/type | Parser fit |
| Tokens per table chunk | Cost |

## Related

- [Digital document](digital_document_docling.md) / [complex layout](complex_layout_ocr.md) — where table structure is produced
- [Reading order](reading_order.md) — tables as floating elements; continuation
- [Layout-aware chunking](../2-chunking/layout_structure_aware.md) — atomic tables, row groups
- [Structured + unstructured hybrid](../7-advanced-rag/structured_unstructured_hybrid.md) — SQL for aggregates
- [Citations](../8-generation-grounding/citations.md) — cell-level provenance
