---
type: "documentation"
title: "Reference lists v1: glossaries and lookup lists kept term by term (Track AE)"
description: "How a list a reader looks things up in (a glossary, an abbreviation list, a lookup list, a code table, a list of symbols) reaches its notes, the graph and an answer, whatever its subject and language. Plan..."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing Kovium"
doc_path: "docs/reference_lists_v1.md"
doc_sha256: "c2caceeca8d56e07ed0d5ad927ee5ca4579d4e20fcca0613d093ff5cafd3f116"
---

# Reference lists v1: glossaries and lookup lists kept term by term (Track AE)

How a list a reader looks things up in (a glossary, an abbreviation list, a
lookup list, a code table, a list of symbols) reaches its notes, the graph and
an answer, whatever its subject and language. Plan:
`docs/knowledge_layer_improvement_plan_v1.md`, Track AE. Code:
`second_brain/notes/source_entries.py`; the kind is `reference-list`
(`format-templates/reference-list.md`).

## Why

A glossary taken in through a document template was summarised away on a
second instance: the extraction agent wrote a few sentences about the list,
and the definitions a reader comes to a glossary for never reached the notes
or the graph. The same list written as a table reached them since Track Y
(`docs/source_tables_v1.md`); written as a definition list, as `Term:
definition` lines or in two columns, it did not, and there was no kind of note
for reference material.

## What counts as an entry

Recognised by layout alone, never by a word, so it holds in any language:

| layout | written as |
|---|---|
| `definition_list` | a term on its own line, then `: definition` on the next (Markdown); several `:` lines are one entry |
| `html_definition_list` | `<dt>term</dt><dd>definition</dd>`, read from the HTML original's tags, since a converter may flatten them |
| `colon` | `Term: definition`, `**Term**: definition`, `**Term:** definition`, a full-width `：` |
| `dash` | `Term — definition`, `Term – definition`, `Term - definition` (spaces around the dash) |
| `equals` | `Term = definition` |
| `columns` | a term and its definition in two columns: a tab, or three spaces or more |

A list marker (`-`, `*`, `1.`) before the term is ignored. A definition may
wrap onto the lines under it: any number indented, two at most flush with the
margin. A **term** is a name, not a sentence: at most `max_term_words` words
(8) and `max_term_chars` characters (80), not ending in sentence punctuation,
not starting with a bracket or a quote, holding no colon, not a web address.

A **run** is consecutive entries with **one** separator; a heading, a
different separator or a paragraph ends it. A run is a list only when:

- it is long enough: two entries in a reference list, `min_entries_in_documents`
  (5) in a document of any other kind;
- each term is defined once (at least 80% distinct): a transcript repeats its
  speakers;
- its terms do not all begin with one word (`Step 1:`, `Step 2:` is a
  sequence) and are not mostly numbers counting up (`1:`, `2:`, the numbered
  lines of an algorithm as a PDF converter writes them); a code table's
  numbers (`200`, `301`, `404`) still count;
- it carries at most 1.5 wrapped lines per entry: more is prose with a colon
  here and there (a two-column PDF flattened to text).

Never read as entries: frontmatter, fenced code, table rows (Track Y copies
tables), HTML comments, and the pipeline's own figure and equation blocks.

## What happens

1. **The kind.** `reference-list` (note type `reference-list`), found by
   `kind` (`glossary`, `abbreviations`, `acronyms`, `terminology`,
   `lookup list`, `code table`, `codebook`, `data dictionary`, …;
   `GET /format-templates`) or by **folder triage** from the file's shape
   (below). It is **built without an agent** (`deterministic`): its title,
   its TL;DR (the first paragraph the source writes about the list, else how
   many entries it holds) and `## About this list` (every paragraph that is
   not an entry, as written) come from the source; a folder plan estimates it
   at 0 tokens.
2. **`## Source entries`**, written by code after the note is filed, into the
   templated note and the wiki note, for **every** kind: all the entries of a
   reference list, and in a document of another kind (a paper's glossary, a
   manual's list of abbreviations) each run of at least
   `min_entries_in_documents`. One item per entry, its definition as written:

   ```
   ## Source entries

   - **Turbidity**: How cloudy the water looks to the meter, in turbidity units.
   - **UV**: ultraviolet lamp in the clear well
   ```

   `normalize_file` sets the section aside before any agent reads a note and
   puts it back after, as for `## Source tables`. Past
   `max_entries_per_note` (500) the note says how many and where the rest are;
   a definition longer than `max_definition_chars` (1,000) is cut, marked
   `…`, and the note says so. `/sources/reread` writes it again.
3. **Ingestion** reads each item as a `Definition` (the mandatory mapping
   `source entries` → `Definition` in `notes/contract.py`), its label
   `Term: definition`, with a `DEFINES` edge to `concept:<term>`: the id any
   note's mention of that name gives its concept. The edge is the record of
   the term (the node may also be stated by another document, such as the
   concept note the concept layer writes for the term, and a node's metadata
   is its last writer's).
4. **Found by the term.** In ranking, an entry whose whole term the question
   names ranks first, when the term is the question's most specific part: no
   other content word of the question is held by fewer notes of the collection
   (Track AA's measure). "What does UV stand for" returns the `UV` entry;
   "how much water does zone 2 need" does not lift an entry for `Water`. A
   term of two letters or fewer is named only in its own case (`a` in "what
   is a pump" is not the code `A`). `score_inputs.term_tier` marks it;
   `config/retrieval.json` `anchors.term_tier` turns it off.
5. **Linked to other notes' mentions.** A question that reaches a concept
   reaches the reference list that defines it too (the `DEFINES` edge counts
   as the list naming the concept), and `POST /query/related` on the concept
   shows the entry. The edge waits, unresolved, until a note names the term.

## Its map

`scripts/build_reference_maps.py` (`reference-index`): the title, the list's
own words about itself, and its entries as cards in the source's order, each
term with its definition as written, at most 48, then a card saying how many
more the note holds.

## Folder triage: a file shaped like a list

A file at least 60% of whose text lines are entries, in runs of three or more,
with five entries or more, is shaped like a list (`source_entries.shape`).
Folder triage gives it the kind that declares that shape (`file_shape:
entries` on the template, so triage names no kind): after its frontmatter,
its file type and its headings, before the folder's default, with
`kind_basis: shape` and `kind_evidence` (`shape:entries`, `entries:N`,
`share_of_lines:S`). A kind known by its shape is not guessed from headings:
its sections are written by the pipeline and their words are in any
document's headings.

## When a list is not one

A source given the `reference-list` kind whose lines are mostly not entries
(under 50%) is filed with its other words kept as written in `## About this
list` and the warning `reference_list_mostly_prose` (`docs/next_steps_v1.md`):
ask the owner whether it is a list; if not, process it again with its own
kind, which an agent reads.

## Configuration

`config/ingestion.json`, block `entries`, a workspace's own over the install's:

| key | default |
|---|---|
| `min_entries_in_documents` | `5` (0 turns entries off in documents of other kinds) |
| `max_entries_per_note` | `500` |
| `max_term_words` / `max_term_chars` | `8` / `80` |
| `max_definition_chars` | `1000` |

`config/retrieval.json`: `anchors.term_tier` (`true`).

## Measured

On invented lists (`tests/fixtures/make_glossaries.py`): an English
definition list, a Spanish `**Término**: definición` list with a wrapped
definition, an English abbreviation list in two columns, an HTML `<dl>`, and a
manual with a glossary at its end. Every entry is a Definition with its term,
copied as written; a question naming a term returns its entry first (`what
does UV stand for` did not without the term rule); a prose letter is not
split. On 251 real Markdown documents and converted papers on the build
machine, read at run time: after the numbered-line and wrapped-prose rules,
no converted paper holds a run, and folder triage over 955 real files gave
three the `reference-list` kind by shape (a tag index with counts and two
`Status: …` records).

## Limitations

- A glossary whose term is set apart only by bold type on a line of its own,
  with the definition in the next paragraph and no `:`, is not recognised.
- A short record of fields (`Status: open`, `Next step: …`) is shaped like a
  list and is triaged as one; its lines are kept as written, as entries.
- A bullet list of bold-led points (`**Point**: explanation`) of five or more
  in a document is copied as entries too: it is a list of that shape.
- Only the HTML original's `<dl>` is read from tags; a `.docx` glossary
  reaches entries only through its converted Markdown.
- A hand-written note of type `reference-list` gets no `## Source entries`:
  the section is written for sources taken in through the pipeline.
- A note taken in before this track gets its entries on its next re-process
  or re-read.
