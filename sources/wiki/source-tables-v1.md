---
type: "documentation"
title: "Source tables v1: a source's tables as facts (Track Y)"
description: "How the tables in a source (an inventory, a mapping, a price list, a parameter list, a comparison) reach its notes, the graph and an answer, whatever the note type and whatever the subject. Plan..."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing Kovium"
doc_path: "docs/source_tables_v1.md"
doc_sha256: "8a3bf475a8d903086dfcaf79146b660449ca63a589acca8bf7895282701cc88a"
---

# Source tables v1: a source's tables as facts (Track Y)

How the tables in a source (an inventory, a mapping, a price list, a parameter
list, a comparison) reach its notes, the graph and an answer, whatever the note
type and whatever the subject. Plan: `docs/knowledge_layer_improvement_plan_v1.md`,
Track Y. Code: `second_brain/notes/source_tables.py`, `second_brain/notes/tables.py`.

## Why

The extraction agent answers with lists of sentences under a "no Markdown"
rule, and the templated note, the wiki note and the graph are built from that
answer. A table therefore reached the converted copy and stopped: an inventory
question on a real instance found prose about the table's subject and none of
its rows. Tables were already copied by code into `## Structured Extracts` on
the normalizer's own paths (`docs/ingestion_spec_v1.md`, "Tables"), but a
source taken in through `/sources/process` is normalized from its templated
note, which held no table to copy.

## What happens

1. After the note is extracted, shaped and normalized, the pipeline reads the
   **converted copy** and writes its tables into a `## Source tables` section
   of the templated note and the wiki note, before `## Provenance`. No model is
   called and nothing an agent wrote is involved: a value in the note is the
   value in the copy. It runs for every note type (it is a mandatory mapping
   in `notes/contract.py`, `source tables` -> `Extract`), like `## Figures` and
   `## Source equations` are written for every type.
2. **Ingestion** reads each row of a table it can trust as an `Extract` node,
   its label the row with its column names and the table's title:
   `Battery capacity — Value [60, 100] Wh (TABLE II: PARAMETER SETTINGS)`. The
   node points at the row's line in the note, so a row found in an answer
   keeps the same provenance a bullet has.
3. `normalize_file` **sets the section aside** before any normalization path
   reads a note and puts it back unchanged after, so no agent is sent it, no
   shaping pass can drop it, and no path copies it twice.
4. **`/sources/reread`** (convert only, or with re-extraction) writes the new
   copy's tables into the notes and re-ingests them; the re-read's snapshot
   holds the old section, so a rollback restores it.

```
## Source tables

**TABLE II: Node configuration.**

| Node | Board | Memory |
|---|---|---|
| Logger | Board A | 1 GB |
```

## Which tables, and their titles

A table is what `extract_tables` accepts as a table of data: a header, a
separator row and data rows, a header at least half named and a body less than
30% empty. The pipe-delimited prose fragments some PDF converters emit are
rejected here as everywhere else.

**The title** is the table's caption when one is written **above** it, the
convention of every style seen. A caption is recognised by shape, never by a
word, so it holds in any language: a word, then an arabic number of one to
three digits or an upper-case roman numeral, then the end of the line,
punctuation or a title (`Table 3: …`, `TABLE IV`, `Tabla 2.`, `表 2 …`), but
not a sentence (`Step 2 of the process`, `In 2020 we`). A label alone on its
line with the title on the next (`TABLE II` / `PARAMETER SETTINGS`), as
converters often write a two-column paper's captions, is joined. The line just
below a table is never taken: it is as likely to be the next figure's caption.
A wholly bold line above a table is its title too. A table with no caption is
titled by the section heading it sits under (`Storage room inventory`), and one
with neither has no title: every word of the section is the source's own, so
a note written with no agent still states only what its source says.

## Tables a converter mangled

Measured on three real two-column conference papers (6 pages, 6 pages, 8
pages; 2 tables each) through docling: all six tables were found and all six
captions read. Four came through clean (61 row facts). Two, both borderless,
came through with **their rows merged into one**: every key of the first
column in one cell, every value of the second in the next, so nothing says
which value belongs to which key.

Such a table (`collapsed_rows`: at most three data rows, a row whose key cell
holds at least four words and three times its column name's, beside another
cell as full) is **copied as text**, so it can still be read and searched, with
a line saying the converter merged its rows and where to read it, and it gives
**no row facts**. The rule is structural and biased to the safe side: a table
wrongly flagged loses its row facts and keeps its text; one wrongly trusted
would state values beside the wrong keys.

What cannot be recovered here, and is not attempted:

- a table the **basic extractor** read: its text has no column boundaries
  (`docs/reading_fidelity_v1.md` reports such regions as lost layout); select
  docling for the workspace (`docs/document_conversion_v1.md`) and re-read;
- a table that is a **picture** in a scan;
- the PDF tables of `oc-markitdown`, which on the papers measured emitted no
  table at all.

## Limits

`config/ingestion.json` (a workspace's own `config/ingestion.json` wins):

```json
{"tables": {"max_rows_per_note": 200}}
```

At most `max_rows_per_note` rows, across all of a note's tables, are copied.
Past it, the note says how many rows it holds and on which lines of the
converted copy the rest are (`This note holds the first 1 of 3 rows; the rest
are in the converted copy (…), lines 18-22.`), and only the rows copied become
facts.

## What a caller sees

- `POST /sources/process` adds `tables`: `{tables, rows_total, rows_copied,
  collapsed: [table numbers], truncated: [table numbers]}`, when the source
  has a table of data (or `{error}` if copying failed; the note is filed
  either way).
- `POST /sources/reread` adds the same `tables` block, with `reingested`.
- An answer's knowledge items include the rows as `Extract` items, ranked like
  any other.

## Limitations

- A caption below a table is not read, and a figure caption directly above a
  table is taken as its title.
- Word counts decide a merged table, so a script written without spaces
  between words is never flagged as merged.
- A question whose answer sits only in a merged table gets no row fact for
  it; the table's text is in the note for a caller that reads it.
- A row's label repeats its table's title, so a long caption makes every row's
  label long.
