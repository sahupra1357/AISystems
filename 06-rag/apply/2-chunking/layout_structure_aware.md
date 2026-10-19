# Chunking → Layout / structure-aware

## What it is

Structure-aware chunking uses the **elements** emitted by a structure-preserving parser (Docling, Unstructured, MinerU, a Markdown/HTML parser) rather than raw text. Each element has a type — `title`, `section_header`, `paragraph`, `list_item`, `table`, `figure`, `caption`, `code` — and a position in the heading hierarchy. The chunker walks the element tree and assembles chunks that:

- never cross a section boundary,
- keep a table (or a table plus its caption) intact as one unit,
- keep list items with their introducing sentence,
- prepend the heading breadcrumb (`Doc › Chapter 3 › 3.2 Termination`) so each chunk is self-describing.

The chunk is now a *semantic unit the author intended*, not an arbitrary token window.

## How to implement

1. **Get structured output from the parser.** Docling `DoclingDocument`, Unstructured `Element` list, or Markdown AST. If your parser gives you flat text, stop: fix parsing first ([OCR / digitization](../../../AI_System.md#ocr--digitization), [structure extraction](../../../AI_System.md#structure-extraction-what-the-parser-must-preserve)). This chunker cannot invent structure.
2. **Build the heading path** for each element by walking the hierarchy (`h1 > h2 > h3`). Store it as `breadcrumb` metadata ([headings hierarchy](../1-ingestion-parsing/headings_hierarchy.md)).
3. **Define atomic elements** that must never be split: tables, code blocks, figures + captions, single list items. If an atomic element exceeds `max_tokens`, handle it specially (table → row-group chunks with the header row repeated; code → split on function boundaries).
4. **Pack sibling elements** within the same section greedily until `max_tokens`. Start a new chunk on every section header at or above a chosen level (e.g. h3).
5. **Prepend the breadcrumb** to the chunk text at embed time (or store separately and prepend at query time — but embedding it is what improves retrieval).
6. **Serialize tables as Markdown or HTML**, not prose ([tables](../1-ingestion-parsing/tables.md)). Store the raw table structure alongside for exact rendering later.
7. **Attach page and bbox** from the parser to enable citations ([page numbers + bounding boxes](../1-ingestion-parsing/page_numbers_bounding_boxes.md)).

## Small example

```python
from docling.document_converter import DocumentConverter
from docling.chunking import HybridChunker   # structure-aware + token-aware

conv = DocumentConverter()
doc = conv.convert("contract.pdf").document

chunker = HybridChunker(max_tokens=512, merge_peers=True)
for i, ch in enumerate(chunker.chunk(doc)):
    headings = " › ".join(ch.meta.headings or [])
    pages = sorted({p.page_no for it in ch.meta.doc_items for p in it.prov})
    text = f"[{headings}]\n{ch.text}" if headings else ch.text
    print(i, pages, headings, repr(ch.text[:60]))
    # persist: {"chunk_index": i, "breadcrumb": headings, "pages": pages,
    #           "text": text, "element_types": [it.label for it in ch.meta.doc_items]}
```

Hand-rolled equivalent on a Markdown AST: iterate nodes, maintain a heading stack, flush a chunk on each `h2`/`h3`, keep `table` and `code` nodes atomic, prefix each chunk with `" › ".join(stack)`.

## Pros

- **Chunks are meaningful units.** A clause, a procedure step, a table with its caption. Retrieval precision and answer quality both jump.
- **Breadcrumb metadata** makes bare numbers and pronouns retrievable ("the limit is 5" → "Policy › Leave › Sick leave: the limit is 5").
- **Tables survive.** The single biggest fix for "table questions always wrong" ([common failure modes](../../../AI_System.md#common-failure-modes-and-the-fix)).
- **Citations become precise**: chunk ↔ section ↔ page ↔ bbox.
- **Filtering by section** ("only search the Definitions section") becomes possible.

## Cons

- **Completely dependent on parser quality.** Missed headings or wrong reading order produce wrong breadcrumbs and merged sections. Garbage in, structured garbage out.
- **Highly variable chunk sizes.** A one-line section and a ten-page section are both "one section". Needs a token cap and an oversize strategy.
- **More engineering.** Per-format element handling, atomic-element rules, table row-group splitting.
- **Format-specific.** A chunker tuned for Docling PDF output does not apply to HTML from a crawler; each source needs a mapping to a common element schema.

## When to use

- **Long structured documents**: contracts, policies, manuals, regulations, financial filings, technical docs, wikis with real headings.
- **Anything with tables** that users ask questions about.
- When you already run a structure-preserving parser (which you should, per [ingestion & Parsing](../../../AI_System.md#ingestion--parsing)).

## When NOT to use

- **Flat text with no structure** (transcripts, chat logs, OCR of unstructured letters): there is nothing to be aware of; use recursive ([recursive character splitting](recursive_character_splitting.md)).
- **When parser structure is unreliable** for a source and you cannot fix it: structure-aware chunking will amplify parser errors. Fall back to recursive for that source and flag it.
- Tiny documents: one doc = one chunk.

## Production at scale

**Common element schema.** You will have five parsers for five source types. Define one internal element model (`type, text, heading_path, page, bbox, table_html, parent_id`) and write one adapter per parser. The chunker operates only on the common model. This is what lets you swap Docling for MinerU on formula pages ([formula-heavy page](../1-ingestion-parsing/formula_heavy_mineru.md)) without touching the chunker.

**Oversize sections.** Enforce `max_tokens`. When a section exceeds it, split *within* the section using recursive splitting ([recursive character splitting](recursive_character_splitting.md)) but keep the *same breadcrumb* on every sub-chunk and set `parent_section_id` so parent–child retrieval ([parent–child](parent_child_small_to_big.md)) can return the whole section later.

**Tables at scale.** A 400-row table is not one chunk. Split into row groups of N rows, repeat the header row in every group, keep caption + breadcrumb on each, and store `table_id` + `row_range`. Also index a short LLM/VLM-generated **table summary** as its own chunk so "which table covers X" queries hit ([tables](../1-ingestion-parsing/tables.md), [contextual retrieval](contextual_retrieval.md)).

**Heading level cut-off is tunable per corpus.** Cut on h2 for short docs, h3/h4 for long manuals. Make it config, version it, evaluate it.

**Parser upgrades change structure.** A Docling version bump can change how headings are detected → different chunks → different IDs → re-index. Treat parser version like embedding model version ([versioning & migration](../3-embeddings/embedding_versioning.md)): store `parser_version` on chunks and run the parsing eval ([test-time validation against a golden sample](../1-ingestion-parsing/validation_golden_sample.md)) before rolling out.

**Metadata size.** Breadcrumbs, page lists, bboxes, and table HTML add up. Keep the vector store payload lean (IDs + breadcrumb + page) and keep bulky fields (table HTML, bboxes) in a document store keyed by `chunk_id`.

**Multi-tenant + structure filters.** Section-level filters are a second filter dimension next to `tenant_id`. Make sure the store does native filtered-ANN on both ([filtered search](../4-indexing/filtered_search.md)) or the tenant filter degrades.

**Observability.** Log element-type distribution per source (paragraph / table / heading counts). A source whose heading count drops to zero after a deploy has a parser regression and is silently falling back to one giant chunk per document.

## Prod checklist

- [ ] Structure-preserving parser in place; parsing eval passes ([validation](../../../AI_System.md#validation))
- [ ] Common element schema + adapter per parser
- [ ] Breadcrumb prepended at embed time and stored as metadata
- [ ] Atomic elements (tables, code, figure+caption) never split; oversize strategy defined
- [ ] Table row-group splitting with header repetition
- [ ] `max_tokens` cap; oversize sections sub-chunked with `parent_section_id`
- [ ] `parser_version` + `chunking_config_version` on every chunk
- [ ] Page + bbox on every chunk for citations
- [ ] Element-type distribution dashboard per source

## Metrics to watch

| Metric | Why |
|---|---|
| Table-question accuracy on golden set | The reason you did this |
| % chunks with empty breadcrumb | Parser missing headings |
| Chunk token histogram | Oversize sections, one-line fragments |
| Chunks per doc, by source | Regression detection |

## Related

- [OCR / digitization](../../../AI_System.md#ocr--digitization) OCR / per-page routing — where structure comes from
- [Tables](../1-ingestion-parsing/tables.md), [Headings hierarchy](../1-ingestion-parsing/headings_hierarchy.md), [Bounding boxes](../1-ingestion-parsing/page_numbers_bounding_boxes.md) — the metadata this chunker consumes
- [Parent–child](parent_child_small_to_big.md) Parent–child — return the whole section after retrieving a sub-chunk
- [Contextual retrieval](contextual_retrieval.md) — LLM header as an alternative to / on top of breadcrumbs
- [Chunk metadata](chunk_metadata.md)

## Video takeaway (30 seconds)

"Structure-aware chunking splits where the author split: headings, sections, tables. Every chunk carries a breadcrumb like 'Contract › Termination › Notice period', so 'the limit is 30 days' is actually findable. It is the fix for table questions. The catch: it's only as good as your parser. In production, define one internal element schema, keep tables atomic with header rows repeated, and treat a parser upgrade like an embedding-model change: it re-chunks the world."
