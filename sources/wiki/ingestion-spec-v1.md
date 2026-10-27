---
type: "documentation"
title: "Ingestion Specification V1"
description: "Markdown is the only supported source-of-truth format for V1. All future source types must first be converted into Markdown by an external pre-ingestion layer."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing Kovium"
doc_path: "docs/ingestion_spec_v1.md"
doc_sha256: "c63c96d741c2506bceccb8650c6874a1ad21af097fd65e567c2650917eed1b48"
---

# Ingestion Specification V1

## Source Policy
Markdown is the only supported source-of-truth format for V1.
All future source types must first be converted into Markdown by an external pre-ingestion layer.

## Input Expectations
Markdown files may include:
- headings
- lists
- timestamps
- tags
- links
- structured sections

## Supported Source Areas
Recommended initial folders under `data/sources/`:
- `notes/`
- `chats/`
- `papers/`
- `summaries/`

## Parsing Rules
- preserve file path
- preserve title if detectable
- preserve heading hierarchy
- preserve bullet structure
- preserve timestamps where present
- preserve tags
- normalize whitespace and section boundaries

## Extraction Philosophy
Conservative and explicit only.

This means:
- explicit mentions produce nodes
- explicit relations produce edges
- ambiguous implications should remain unlinked unless promoted later by high-confidence rules or manual review

## Provenance Requirements
Every extracted node and edge must store:
- source file path
- section or line/segment reference if available
- supporting snippet
- extraction timestamp
- extractor version
- confidence

## Canonical Intermediate Format
Each parsed Markdown document should produce a structured JSON record with:
- document metadata
- extracted sections
- extracted entities
- extracted candidate relations
- provenance payload

## Note Identity
A note's graph id comes from **where it lives**, not what it says:
`note:<first 12 hex of sha256(workspace-relative path)>`.

This was previously the *content* hash. Editing a note then gave it a new id, so
`ON CONFLICT(id) DO UPDATE` could not match it: the old node was orphaned rather
than updated and the note appeared twice. Edge ids embed the note id, so every
edge duplicated with it. One live workspace held six Note nodes for four files;
another held six for three.

The `note:<12 hex>` shape is part of the contract — callers such as the canvas
service select note cards by the `note:` prefix.

The content hash is still recorded in `Node.metadata['hash']`, and is still how
staleness is detected. It simply is not the identity.

## What Reaches the Graph From the Frontmatter
Ingestion parses the frontmatter block and puts on the `Note` node:

- `type` — **always present**, as `"unknown"` when the note declares none.
  Omitting it would hide un-typed notes from a type query; recording it is how
  the gap becomes findable and gets fixed.
- `facets` — the closed vocabulary, normalised so a scalar and a list both
  arrive as a list. Frontmatter is read first, so it wins over a legacy
  `## Facets` section. **Which facets exist is read from `config/facets.json`**
  through `second_brain/notes/facets.py`, and a workspace may declare its own,
  so this is deliberately not a list here — a documented list is a copy, and
  ingestion walking a stale one drops a workspace's facet values silently.
- `status`, `reading_status`, `description`, `concepts` — what an agent needs to
  decide whether a note is worth opening, without opening it
  (`INDEXED_FRONTMATTER`).
- `source_status` — present only when the note's original is gone: `missing`
  (found by `POST /sources/reconcile`) or `retained_without_source` (kept by its
  owner). See `docs/source_renames_v1.md`.

### Where the original is: `raw_source_path`, `source`, `sha256`

A note records its original by path and by the hash of its bytes. The path is
where the source is today, and changes when an owner renames or moves it;
`POST /sources/reconcile` repairs it by the hash, keeping each earlier path in
`previous_source_paths` (a block list, one `- path` per line) and in the source
registry (`maintenance/source_registry.json`, rebuildable from these fields).
A note whose original is gone carries `source_status` and
`source_missing_since`. Re-processing a renamed original repairs the chain
first, so it rewrites the same note rather than writing a second one.

The frontmatter block is **skipped** when walking body lines. It is metadata,
not prose; walking it read `type: paper` as a content line and minted nodes out
of it.

### Values marked unverified (Track L2, 2026-10-09)

`POST /sources/process` checks the note it has just written against the
source's text form before concept links and the graph are built from it, and
marks a value it found only in text whose layout was lost in conversion:
`[unverified: layout lost]` at the end of a statement, `[unverified]` after a
table cell's value, `[unverified: source row differs]` for a table value the
source prints in another row. The frontmatter gains `fidelity_unverified_values`
and `fidelity_layout_lost_blocks`. Ingestion keeps the mark in the node's label
and provenance snippet, which is how the governed answer marks the item
`unverified`. An owner-maintained note is never marked. The marking is
deterministic; the statement-level findings the check then opens a review item
for are read against the source by the brain keeper first, within the standing
token budget (Track AS4), and the reading is counted in the note's cost. See
`docs/reading_fidelity_v1.md`.

Facets previously reached the graph only from a `## Facets` *section*, which the
current template does not emit — so in a live workspace one note in six carried
any facets at all.

## Tables — preserved deterministically, then read as facts

A table is the densest factual content a note carries: a config reference, a
comparison, a problem paired with its result. It used to be dropped twice over.

**At normalization.** The extraction contract the agent answers is a JSON
object of string lists with a hard "No Markdown" rule, so a table cannot
survive it structurally, and `## Structured Extracts` was rendered as an empty
heading with no variable behind it. Measured across every workspace: **0 of 40
normalized notes contained a table**, while their sources did — on one
14-chapter book, 1,913 rows across 234 tables reached the agent and none came
back.

Tables are now copied from the source into `## Structured Extracts` by
`second_brain/notes/tables.py`, before the agent's output is rendered around
them. This is **deterministic**: a table is already structured, so reading it
needs no judgment, costs no tokens, and cannot be hallucinated. A source with no
tables leaves the section empty exactly as before.

A source taken in through `/sources/process` is normalized from its templated
note, which the agent wrote and which therefore held no table: its tables now
reach the notes through `## Source tables`, copied from the converted copy
after extraction, captions read and tables a converter merged kept as text
without row facts (Track Y, `docs/source_tables_v1.md`). A list of terms with
their definitions (a glossary, an abbreviation list, a lookup list) reaches
them the same way, through `## Source entries`: each entry a `Definition`
labelled `Term: definition`, with a `DEFINES` edge to `concept:<term>`
(Track AE, `docs/reference_lists_v1.md`).

A table is only recognised with its separator row (`|---|---|`). Without that
requirement there is no way to tell a table from prose containing a pipe, and
guessing wrong mints nodes out of sentences.

**PDF-converted sources are the reason a quality guard exists.**
`oc-markitdown` renders a two-column academic PDF by shredding its *prose* into
pipe-delimited fragments — a paper's title arrives as a 15-column "header" of
half-words. These are syntactically valid Markdown tables and are not tables at
all. Measured across 24 converted papers: **442 such tables, none genuine.**

The two populations separate cleanly, with no overlap:

| | columns | empty-cell ratio |
|---|---|---|
| PDF-converted (mangled layout) | 11–17 | ≥ 0.42 |
| native Markdown (real tables) | 2–8 | 0.00 |

So `looks_like_a_data_table` requires a header that is at least half named and
a body less than 30% empty. The guard is deliberately biased towards rejection:
a false negative loses a table and leaves the previous behaviour, with the
content still reaching the agent through the note body, while a false positive
puts sentence fragments into the graph as facts and needs a re-ingest to undo.
Verified: 442 mangled tables rejected, and all 1,417 real table facts in a
native-Markdown book kept.

**At conversion.** `oc-markitdown` also loses the spaces *between words* inside
table cells and captions, so `Training Cost (FLOPs)` arrives as
`TrainingCost(FLOPs)`. `second_brain/conversion/table_repair.py` restores them,
running beside the maths repair and for the same reason: once corrupted text is
normalized and ingested, the wreckage is in the graph as node labels and only a
re-conversion gets it out. Deterministic — no model, no tokens. Measured on the
converted corpus: 120 cells and 10 captions repaired across 18 papers.

Spacing is restored only where the text still carries a signal that a boundary
was there — a case transition, or a bracket. `Transformerachieves` is left
alone, because nothing in it says where the seam was and recovering that needs
dictionary segmentation. Under-repair is the deliberate failure direction: a
wrongly split token changes a value or a name, while an unsplit one is merely
ugly. For the same reason the case rule requires a real word on each side of
the seam, so a model name like `ConvS2S` is not turned into `Conv S2S`, and the
repair runs **only inside tables and captions** — never across body prose,
where every ordinary CamelCase identifier would be at risk.

**At batching.** A batch that opens mid-table shows the agent
`| row 300 | value 300 |` with no column names, so whatever it extracts is
guesswork. `split_into_batches` therefore treats a contiguous pipe block as a
unit: a table that fits in one batch is pushed **whole** into the next, and a
table larger than the budget — which no choice of cut can keep whole — is split
with its header **repeated** at the top of each continuation, and reported as
`table_split_across_batches`.

The repeated header goes into the batch *text* only; `start` and `end` still
name the exact source range. Coverage is therefore measured from the offsets and
never by concatenating batch text, which would count the repeated headers and
understate coverage.

**At ingestion.** Each data row becomes one node, typed by its section like a
bullet, and labelled with its own header:

```
| Model | BLEU | Cost |        ->  Transformer — BLEU 28.4, Cost 3.3e18
| Transformer | 28.4 | 3.3e18 |
```

Pairing every value with its column name is the point: a bare `28.4` in a
ranked result is meaningless, and the header is what makes the number a fact.
The header row and separator produce nothing, and a blank line ends a table so
two tables in one section never share a header — the second read through the
first's columns would mislabel every fact in it.

Each row keeps its own line number, so a retrieved table fact has the same
provenance a bullet does.

## Maths Repair — before anything else touches the text
`oc-markitdown` flattens **both halves** of a MathML formula into the Markdown:
the presentation rendering *and* its LaTeX annotation, side by side and unfenced.

```
the rate is R u , e up = B log 2 ( 1 + P | h u,e |² σ² ) , R_{u,e}^{\mathrm{up}}=B\log_{2}\left(...\right),
```

It corrupts prose as well as display equations, because inline symbols double
the same way — "task m m is offloaded", "user u u", "bandwidth B B". Measured on
one shipped note: **21 doubled single-letter tokens, 140 stray LaTeX commands**,
and the wreckage had already reached the graph as a node label.

`oc-markitdown` is an external CLI outside this repo and cannot be changed, so
`second_brain/conversion/math_repair.py` repairs the output. Deterministic — no
model, no tokens, no network.

The rule is **keep the LaTeX, drop the rendering**: the LaTeX carries the
structure and can be rendered again, the spaced-out unicode cannot be recovered
into anything. Repaired maths is fenced (`$…$` inline, `$$…$$` for display) so
it survives into the note and renders in Obsidian.

It runs **immediately after conversion, before normalization**. Once corrupted
text is normalized and ingested, the wreckage is in the graph as node labels and
only a re-conversion clears it.

Safety properties, all tested:
- **Idempotent.** Re-running changes nothing, including across the three lines a
  display fence occupies.
- **Prose is never damaged.** The backward scan that removes a rendered half
  stops at the first ordinary word, and repeated English words (`a a`,
  `that that`) are left alone.
- **Failure is non-fatal.** An unreadable or unwritable file yields no repair
  rather than a failed conversion — unrepaired text beats no text.
- **Audited, not silent.** The count is recorded in the conversion metadata as
  `math_doubling_repaired` and surfaced as a warning.

A bullet that is mostly formula no longer becomes a node (`is_math_heavy`). A
formula is not a claim, and an equation as a node label is hundreds of
characters of LaTeX polluting every ranked result it appears in.

## Broken Characters Repair — letters a converter lost (Track AT)
A PDF that sets a letter pair (ff, fi, fl, ffi, ffl) as one glyph can lose it in
conversion ("di?erent" for "different"). `second_brain/conversion/glyphs.py`
runs on every converted copy right after conversion, again before anything
reads it: the Unicode ligature characters are spelled out, and a word with a
lost glyph is repaired only when exactly one candidate pair makes a word that the
same document holds intact. No dictionary, no language assumption, no model, no
tokens. What it cannot repair stays as written and is counted:
`conversion.warnings` carries `broken_glyphs_repaired:N` and
`broken_characters_left:N`, `conversion.metadata.broken_glyphs` the counts and
the listed words. Lint counts what is left per note, and a `## section` with a
high share of broken characters is not embedded (the vector text; the words stay
in the note). The nightly run repairs notes already held and re-ingests the wiki
notes it changed. Contract and settings: `docs/document_conversion_v1.md`,
"Broken characters".

## Books — split into chapters before anything else
**A whole book becomes one note per chapter, plus a parent index note.** One
note cannot represent a book: measured on a real 14-chapter book (653,275
chars), everything is read but a single note retains 12 concepts and 12 claims
out of roughly 1,600 extracted items. Input coverage was never the same thing as
representation.

`second_brain/normalization/chapter_split.py` proposes the boundaries;
`scripts/split_book.py` writes the files. Deterministic — no model.

### The chapter-finding ladder (Track AL)

Books differ by language, publisher and kind, so no single rule finds every
book's parts. `second_brain/normalization/book_ladder.py` climbs a ladder:
each step runs only when the one before gave no confident split, and every
proposal, whoever made it, passes the same checks in code.

1. **`owner`**: the parts the owner said, passed by the caller on the user's
   word as `chapters: [{title, pages}]` (pages `41-78`, or `41` for where a
   part starts; no pages for a book that has none). It skips the ladder. It
   is an option a caller may send, never a question the platform asks.
2. **`outline`**: the PDF's own bookmarks, read without converting it
   (`second_brain/conversion/pdf_outline.py`: pypdfium2 or pypdf where
   installed, pypdfium2 in docling's interpreter, else a small reader of the
   file itself that needs nothing installed, following plain and compressed
   object streams and named destinations). Free and exact. Its top level is
   taken, unless that level has fewer than four parts and the level below is
   confident with chapter-sized parts: then the top level only groups chapters
   (`Part I`, `Part II`), and a part's own page joins its first chapter.
3. **`contents`**: the contents page. Its heading is any of the install's
   declared headings (`contents_headings` in `config/books.json`, several
   languages shipped); with none, it is found by its shape: at least
   `contents_by_shape_min_entries` short lines near the start ending in rising
   page numbers, a list of figures or tables (lines opening with a caption
   label) excepted. Everything below about contents lists then applies.
4. **`headings`**: the body's openings, a label and an ordinal (the label any
   word of any script, the ordinal digits of any script, a roman numeral or a
   declared `ordinal_words` word), else a bare ordinal. A numbered caption or
   statement is never a part: `not_chapter_labels` (`Algorithm`, `Listing`,
   `Theorem`, `Lemma`, `Definition`, `Proof` and the rest, in several
   languages) is the install's list. Boundaries the code flags are ruled on by
   `adjudicate_chapters`, within the budget below.
5. **`agent`**: the brain keeper agent proposes the parts (`propose_chapters`)
   from a compact view, never the book: the contents page if there is one,
   every heading-like line with its page, and the first line of each page
   (each block of 80 lines in a book without pages), cut to
   `agent_view_max_chars`. Its answer is a list of `{title, start, end}` and
   `confident`; an unsure answer, and an answer that fails a check, is refused
   whole and never half used.

**The checks**: at least two parts; in page order; none past the document's
end; their declared ranges overlapping nowhere and leaving no pages out
between them or after the last (two pages of slack at the end); each title
found where it is said to start (on its page, or the page either side, as a
line, a label line and the title under it, or a title wrapped over lines;
compared on letters and digits of any script without case, and, for a title
of six letters and digits or more, without spaces too, so a converter's
`## 3. SystemModel` or a hyphen-broken `Sys- tem Model` is the title `3.
System Model`, always as a whole line, never inside a sentence; on a page a
heading line is preferred to a running header or a line-start mention:
chores after Track AM, C-R1);
no part holding more than `MAX_SHARE_OF_BOOK` of the body; the parts
reassembling into the text byte for byte. The owner's word is the pages: a
title of theirs not printed on its page starts the part at the page's top
(`owner_title_not_on_page`), and a large part of theirs is reported, not
refused; a split of theirs that fails any other check is refused
(`book_owner_split_refused`) rather than replaced by another step's.

**Matter around the chapters is not a thin chapter** (fix round, AL-R1).
With a numbered sequence of chapters, an entry before the first numbered
one, after the last, or with no number of its own (an abstract, an appendix
heading over its appendices, a glossary, a list of acronyms, lists of figures
and tables, a bibliography) is matter around them, told by position and
numbering, never by its words. The size checks (`thin_chapters`, the median
of `chapters_look_like_sections`) read the numbered chapters only, so a
contents split whose numbered chapters all place, cover the text and pass
the checks is confident however short its back matter. A numbered chapter of
one line is still thin.

**An older converted copy, and the PDF's own text layer** (chores after
Track AM). A converted copy can lose what a step reads: on a real 160-page
thesis an older docling copy had its contents page run together, the contents
step found nothing, and the headings gave three parts covering pages 17 to
160, one of them 92 pages long. The same PDF's text layer (`pdftotext`) gave
the right 14 parts. So for a PDF whose converted copy is not the text layer
already, when the copy's contents page gives no confident split, the text
layer's contents page is read next (free, once), and when the copy's headings
give none either, the text layer's headings; each before the agent is asked.
The text layer only says which parts there are and on which pages: the parts
are placed on the converted copy by their titles on those pages, and checked
as every proposal is, so the notes are always made from the copy. A
contents page comes before the copy's headings because it is the document's
own statement of its parts. `split.basis` is `text_layer` when it decided,
and `tried` lists each look at the text layer (`basis: text_layer`).

**Nothing is left out in silence.** Each part runs to the next and the last
to the end, so only the text before the first part can be in no part (a title
page, an abstract, the contents). The split reports it in `left_out`: `[{pages,
chars, why}]`, the pages it spans (null when the copy has no page markers),
its size, and why (it comes before the first part and is kept in the book's
index file, not in any part note); empty when the first part starts the text.
The dry run's `split.left_out` says the same before anything is converted.

**Within the budget, or not at all.** The agent step (and adjudication)
runs within the standing token budget (`config/budget.json`), or, when
adding documents, within what the run has left; no separate question. With
no budget set, no agent available, `agent_split: false`, or an estimate past
what is left, only the free steps run. A book none of them splits is
refused as before (`needs_review` when an unsure split exists,
`no_chapter_structure_detected` otherwise), and adding documents reads it
whole, in parts, the safe default. The book route itself reads nothing whole:
its `split.line` says nothing was split and what settles it, and `split.next`
(`settled_by: caller`) is the recommended call, adding it as a document
(`POST /sources/add`), or `chapters` when the user named them, or `force`
after inspecting the split. A split the owner gave that fails says so as
theirs (`book_owner_split_refused`), never that the parts were unclear. The result's `split` says which step
decided (`step`), what each step found (`tried`), what the agent cost
(`agent`: `asked`, `estimate_tokens`, `view_chars`, `book_chars`,
`billed_tokens`), why it was read whole (`whole_because`: `no_budget`,
`no_agent`, `agent_split_off`, `over_budget`, `agent_unsure`,
`agent_answer_refused`, `agent_answer_malformed`, `agent_unavailable`), and
one plain line (`line`) a caller can relay. The book result's `cost` holds
what finding the parts cost.

**A dry run shows the split first.** `dry_run: true` converts nothing and,
under `split`, shows the split the book would get and which step would make
it, with each part's name, wherever that is known for free (`basis`): a book
that is text already, a copy converted before, or a PDF's text layer read
quickly by the basic extractor (`pdftotext`, which triage already reads;
never docling), over which the free steps run (fix round, AL-R4); with no
text at all, the owner's split and the PDF's outline are shown unchecked. The
owner's split always passes every check that needs no text first (at least
two parts, order, overlap, pages within the page count, no pages left out
where ends are given): one that fails is `ok: false`, `split.refused`,
warning `book_owner_split_refused` with its `detail`, as the run would refuse
it (AL-R3). It never asks the agent; `split.agent` says whether the ladder
would reach it (`would_ask`), whether it may, and its estimate.

**A dry run states its tokens (Track AM2).** Beside the conversion time
(`estimate`, seconds, unchanged) the dry run has `tokens`, from this install's
own call log (`folder_intake_v1.md`, "Estimates that learn from this
install"): `chapters` (each `{position, title, pages, chars, extraction,
concept_upkeep, estimated_tokens}`, every figure a `{low, expected, high}`),
`extraction` and `concept_upkeep` (the sums; `concept_upkeep.paid_by` is the one
pass after the last chapter, and it is zero with a basis saying why when the
concept layer is off or written without an agent), `split_agent`
(`estimated_tokens`, `in_the_total`, `note`: the chapter-finding step is a
ceiling, in the expected and high figures only when the free steps did not
find the chapters), `total`, `basis` and `group` (the chapter notes of this
install it stands on, or the stated default), `chapters_basis` (the real split
this dry run found, or an assumed count of about 25 pages each when no split
is known yet), `planning_figure: expected` and one plain `line` (the total and its range; what reading a chapter takes
on average; and, with upkeep, the range of one chapter's total). A book
whose length is not known yet answers `known: false`. The figures are
estimates, not a quote.

**Measured** (`scripts/eval_book_ladder.py`, on the synthetic documents of
`tests/fixtures/long_documents.py`: thesis-, report-, manual- and novel-like,
in English and German, with and without an outline and a contents page, 32
in all): with the free steps only, 30 split right (the outline decided 16,
the headings 12, the contents page 2) and 2 were read whole; with the agent,
all 32 split right, the agent asked for those 2 only (stood in by an answer
of the right parts, no model), its view 2,130 and 2,138 characters against
116,155 and 121,861 for the whole book, estimated at 3,833 and 3,835 tokens.
No synthetic document was split wrong by any step.

### Each part's name, on every layer (Track AL8)

Every part of a multi-part document is named
`<document-key>-<NN>-<part-title>`: the key is the document's own title cut
to `document_key_chars` at a word and spelled by Track AK1's rule (its file's
stem when the title has no Latin spelling); the part's title is cut at a word
too, and the book's own words it repeats (`Introduction — <book title>`, as
hand-split chapters were titled) are left out of its name; `NN` is the part's position in reading
order, two digits (three past 99), not the label its author used (a `Part II`
or `Appendix B` keeps its label in the title and heading); `00` is the index
note, `<key>-00-<index_part_name>`, so it sorts first. The same name is on
every layer the platform writes for that part: its part file under
`sources/raw/book_chapters/<book>/`, its converted copy, its draft
(`<name>__templated.md`), its wiki note and its map. Two books with one key
are told apart as any collision is (`-2`, `-3`, the folder too). A paper
split into sections is named the same way. Each part's notes record
`part_number` (its position) beside `chapter_number` (the author's number,
where its title has one), and its `title` and heading are the part's own
(`3. System Model`), the document's being `book_title` (AL-R5: they were the
file's slug, and titles feed answers and search); a book's parts are ordered by page, then by
`part_number`. Both are work fields of a book chapter (Track AJ), so two
parts of one book, sharing its ISBN and authors, are never versions of each
other. The owner's original is never renamed.

Parts named under the earlier rule (`<book>-chNN-<title>`, drafts named after
each chapter's own title) keep their names when the book is processed again,
and are brought to this one by `POST /rename/parts` (dry run first), through
the rename engine: every part's note and draft, and its part file, converted
copy and index file when those are the platform's own, in one snapshot.

### Chapters split by hand (Track AL6)

`POST /sources/book_parts/attach` `{book, parts: [{note, pages}], book_title?,
dry_run: true}` records chapter notes made by hand as parts of their book, in
the order given: on each note and its draft, `book_source`, `source_pages`
(`unknown` without pages), `part_number` and, where the note has none,
`book_title`; then the note is read into the graph again (no model). Notes
their owner maintains, and notes already part of another book, are left as
they are and listed (`skipped`). Pages that go back are refused
(`book_parts_out_of_order`); a note that is not the collection's is
`book_part_not_found`. The result's `rename_next` is the `POST /rename/parts`
dry run that brings their names to the rule above.

### Settings (`config/books.json`)

The install's file (shipped) writes the defaults out, and its lists are the
lists; a workspace's own `config/books.json` adds to a list entry by entry,
and an entry written `-word` takes that word out; numbers and switches
replace. A malformed value falls back and is named in `problems`.

| key | default | meaning |
|---|---|---|
| `contents_headings` | `contents`, `table of contents`, `index`, and the words of several other languages | the heading of a contents page, a whole line, compared without case |
| `not_chapter_labels` | `figure`, `table`, `algorithm`, `listing`, `theorem`, `lemma`, `definition`, `proof`, ... in several languages | a label before a number that is a caption or a numbered statement, never a part |
| `ordinal_words` | `one`, `two`, `three`, `four`, `five`, `six`, `seven`, `eight`, `nine`, `ten`, `eleven`, `twelve` | numbers written as words (`Stave One`); a workspace adds its language's |
| `read_outline` | true | read a PDF's bookmarks first |
| `contents_by_shape_min_entries` | 4 | the fewest lines with rising page numbers that make a contents page with no heading |
| `agent_split` | true | the agent may propose the split, within the budget |
| `agent_view_max_chars` | 60000 | the most its compact view holds |
| `index_part_name` | `contents` | the index note's name word |
| `document_key_chars`, `part_title_chars` | 40, 40 | how long the key and a part's title may be in a name |

Detection was derived from **six books with deliberately different
conventions**, not one: a British Academy report, an arXiv paper, an FAO
guidelines document, a textbook chapter, Dickens' *A Christmas Carol*, and a
Markdown-authored technical book.

**Converted PDFs contain no Markdown headings at all** — zero, across all five
PDFs. A rule built on heading level is not merely book-biased, it is
*format*-biased. So the detectors adapt: where headings exist they are the
evidence; where they do not, every line is a candidate.

Three detectors, strongest evidence first:

1. **Table of contents** — the book's own declaration of its chapters, and
   authoritative where present (two of five books had one). Bounded by where
   entries stop looking like entries, and matched back to the body forward-only
   on normalised text, so matches arrive in order by construction.
2. **Label + ordinal**, label-agnostic. *A Christmas Carol* uses **Stave**; the
   FAO document uses **Part**. Ordinals may be arabic, roman or spelled out.
3. **Bare ordinal** (`1. Introduction`) — stricter, because numbered list items
   and running page headers look identical to chapter headings.

Then shared validation:

- **Monotonicity** — chapter starts always ascend.
- **Shallowest heading level wins.** A book can carry two hierarchies at once;
  `### Stage 1..6` inside one chapter must not outvote `# Chapter 1..14`.
- **Distinct titles.** `2 CHAPTER1 • INTRODUCTION`, `4 CHAPTER1 • INTRODUCTION`
  is page furniture, and left alone it split one textbook chapter into eleven
  fake chapters while reporting itself confident.
- **Near-consecutive ordinals.** Page numbers climb in twos and fours; chapters
  climb in ones.
- **Scale as a hint.** A paper's numbered sections and one chapter's `##`
  headings are structurally identical to a book's chapters — only scale
  separates them, and only as a hint, so a small median is reported, never used
  to reject.
- **Byte-exact coverage.** Front matter plus every chapter must reassemble into
  the source. The Phase 5 guarantee, applied to splitting.

**`POST /sources/process_book`** does the whole thing in one action: split,
write the chapter files and index into the workspace, and process each as a
`book-chapter` note. A failing chapter does not lose the book — the files and
index survive and failures are reported per chapter.

**A book may arrive as a PDF or any other document format.** It is converted
first, as `/sources/process` converts a source, and the converted text is kept
beside every other converted source (`conversion` in the result says how).
A PDF goes to the workspace's selected PDF converter (`pdf_converter` in
`config/conversion.json`, Track M; `docs/document_conversion_v1.md`); with none
selected, or the selected one unavailable, its text is taken from `pdftotext`
with its layout, and not through the paper-oriented conversion: that one joins
lines into paragraphs, while chapter detection reads chapter openings as lines.
A refused converter name is `reason: converter_refused`. A PDF with
no text layer, or a format the converter rejects, is a stated refusal
(`ok: false`, `reason: no_text_layer` or `conversion_failed`, with `detail`),
like the route's other refusals, never a server error. Before this, the route
read its source as text, so a PDF arrived as its bytes and was refused as having
no chapters.

**Contents lists as real theses lay them out.** Converted PDFs of theses showed
what the detector got wrong, and each fix is about layout, not language:

- dot leaders, packed or spaced (`. . . .`), and roman page numbers are
  stripped from contents entries
- an entry whose page is roman is front matter, not a chapter
- only the top level of numbering is a chapter (`1 Introduction`, never
  `1.4 Outline`)
- a wrapped entry is joined before matching, in a list that gives page numbers
- a list that gives page numbers ends at its last entry that gives one, so the
  body's first heading is not swallowed by it
- a line with a dot leader is never a chapter opening, so a second contents page
  or a list of figures cannot match
- a body line must hold most of a title to match it: a lone page number `1` is
  a prefix of `1 introduction` too

- a chapter opening set as a label line with the title on the next line
  (`Chapter 3` / `System Model`) matches its entry; a numbered entry needs its
  own number in the body, so two chapters sharing a title stay apart and a
  running header (`12  Chapter 3. System Model`, a page number first) opens
  nothing
- parts numbered with roman labels above arabic chapters are a level up and are
  not chapters; a contents page's own header between its pages is not an entry
- an opening spread over a few lines is recognised: within at most six
  non-blank lines, lines holding only the chapter's own number and at most one
  word (in any order, any case, any language), then a line that is exactly the
  title (`1` / `1 CHAPTER` / `Introduction`); the window starts on a line
  holding the number, so a part's title above it is not pulled in
- an entry ending in a page number, even after a single space
  (`II. Middleware 23`, `Foreword 4`), is complete and never joined to the next
  line; a lone page number or the list's heading on a line of its own is a page
  header; and a paged list ends before the first entry whose page goes back
  (the body's `Chapter 1` after page 25)
- a contents list is bounded by where its entries stop, not by a line count: a
  thesis listing its sub-sub-sections runs to hundreds of lines (searched for
  within the first 3000 lines, read for up to 3000)
- an opening whose title wraps onto the next line (`6. Minimizing the Energy` /
  `Consumption`) is recognised: joined, it holds the chapter's number and ends
  with the entry's title

**A split is confident only when it is complete.** From a contents list, every
entry the list declares must be found in the body, in order; one that is not
is named in `toc_entries_not_found`. Whatever found the chapters, numbers that
skip (1, 2, 4, 5) mean a chapter was not found and its text sits inside its
neighbour: `chapter_numbers_skip`, never confident, unless the missing number is
one the brain keeper ruled was not a chapter. And the parse of the list is
itself checked: a numbered section in the list (`6.1`, `6.3.1`) whose chapter
is not among the parsed entries means an entry was lost while reading the list,
named in `toc_sections_without_chapter`, never confident. This check reads the
list again from its heading, independently of where the parse stopped, so it
does not share the parse's blind spot; it is bounded only by the list's own
structure (three non-entry lines in a row, or an entry whose page goes back, as
a list of figures after the contents does). A number counts as a lost chapter
only where it continues the chapters' sequence (a gap, or at most one number
past the last); `0.5 times ...` or `56.2 percent ...` is not a chapter. Measured live, a thesis was split
"confidently" into chapters 1, 2, 4 and 5 with 3 and 6 absorbed, which is the
outcome this route exists to refuse. A split the detector is still unsure of is
refused as before.

**The brain-keeper rules on what code could not settle.** `adjudicate_chapters`
receives only the flagged candidates and their excerpts, never the book — the
agent never scans to find chapters. A rejected boundary is **merged into the
chapter before it, never dropped**, so byte-exact reassembly survives
adjudication; `uncertain` leaves the boundary and the review flag, because the
agent being unable to tell is not permission to decide. Every failure path
degrades to the unadjudicated split.

**A book processed as one note now says so.** Splitting is opt-in, so a book
uploaded without saying it is one would otherwise become a single note
representing a few percent of itself, silently. A long, confidently chaptered
source now carries `WARNING unsplit_book: N chapters detected` on the note. The
check is conservative — a long paper with numbered sections is not nagged.

**When detection is not confident, no files are written and the source is
processed as a single note.** A wrong split is worse than none: it invents
boundaries that misattribute every claim inside them. Thin or ambiguous segments
become `review_candidates` with an excerpt, for the brain-keeper to rule on —
judgment on a handful of candidates, never a scan of the book.

Front matter before chapter one is attached to the **index**, not to chapter
one, so a chapter note is exactly its chapter.

## Converting a PDF — the converter is the workspace's choice (Track M)

Full contract: `docs/document_conversion_v1.md`. `pdf_converter` in the
workspace's `config/conversion.json` is `docling`, `markitdown` or null. Null
(the default) is the basic extractor, `pdftotext`: two columns interleave,
equations come apart, tables flatten and figures are lost, and the result
(`conversion.converter`, with `fallback_reason`) and the note (frontmatter
`conversion_converter`, a `- Conversion:` provenance line) say so. A selected
converter that is not installed or fails on the file falls back to the basic
extractor and says why; a name off the list is refused, naming the list.
Non-PDF formats go to markitdown as before. `conversion.metrics` counts what the
converted copy recovered: equations as LaTeX, Markdown tables and rows, figures.

With docling, each figure is an image file beside the converted copy
(`figures/<id>/fig-<n>.png`, a short id rather than the source name, catalogued in `figures.json` with caption and
page), a marked block in the converted copy, and a figure item in the note's
`## Figures` (never in an owner-maintained note), ingested as a Figure node
per document with confidence below text (0.85 caption, 0.6 a reading of the
image). Vision reads figures only when the owner switches it on
(`config/ingestion.json` `vision.enabled`, with per-source figure and token
budgets) or asks after ingestion (`POST /sources/figures`, dry run first). The
extraction never sees an interpretation: it reads the converted copy before any
is written. `POST /sources/reread` reads one source again with another
converter and returns a before/after comparison.

## Running the Pipeline — backgrounded, and paid for once

`POST /sources/process` and `POST /sources/process_book` both **background by
default** and return a `job_id` to poll at `GET /brain_keeper/jobs/{job_id}`.
`POST /sources/process_folder` triages a whole folder first and is a dry run by
default; its real run is backgrounded in the long-job lane. See
`docs/folder_intake_v1.md`.
The pipeline is several agent turns and takes minutes; blocking held an HTTP
connection open for all of it while occupying one of the 40 threadpool slots the
whole server shares. Observed live: a caller waited several minutes, gave up
client-side, and the work had already completed and persisted — so the run
looked failed while having succeeded. `wait: true` still blocks, for a caller
that genuinely wants it.

An obvious mistake is still caught before backgrounding: a missing source is a
404 on submission rather than a failed job discovered minutes later.

## When an extraction falls back, the note says why

An agent draft can fail in four places: no brain keeper, a job that could not be
submitted, a job that failed, or an answer refused for its shape. A paper then
falls back to the extractive fallback described below. Any other type falls
back to it too when **no agent is available at all** (the brain keeper is
switched off or absent for the workspace); when an agent is on and failed, a
non-paper type is still refused with an error naming the agent's reason, since
a retry is likely to give the note it should have. Each failure is recorded
with its stage and reason, and reaches:

- the templated note's frontmatter: `template_runtime_degraded_reason` (the
  agent's failure first, then the next tier's own error, so neither hides the
  other), `template_runtime_missing_fields` (list fields the agent's answer
  left out, filled empty), `template_runtime_part_failures` (parts of a long
  document whose read failed while others succeeded)
- its `## Provenance`, as `- Extraction: …` lines a reader sees
- the wiki note's normalization provenance, which carries the same three keys
- the error a non-paper type raises, which now names the agent's reason

A note built the strong way gains none of these, so their presence alone marks
a weaker note.

## What a fallback may write: only its input (Track P)

When no agent reads a source, a deterministic fallback builds the note. Until
2026-10-10 these fallbacks *wrote* content of their own, taken from the first
instance's corpus. Every fallback now **selects and never composes**: each
content line of a fallback note is a span of its input (after Markdown markup
is removed, and possibly cut short with `…`). The only text a fallback adds is
template scaffolding that names no subject (headings, the `- TODO` placeholder
of an empty section, provenance lines).

**What it selects** (`second_brain/normalization/extractive.py`,
`EXTRACTOR_ID = extractive-v1`). Structure only, never subject matter, so it
means the same in a recipe, a contract, a lab protocol or a paper, in any
language that ends sentences with punctuation:

| Shape in the input | Becomes |
| --- | --- |
| the first one or two sentences of the first prose paragraph | the summary / TL;DR |
| the first sentence of every later paragraph; a bulleted item that is a whole sentence | claims |
| a sentence with a digit in it | evidence |
| a sentence ending in `?` (`？`, or opening with `¿`) | questions |
| a numbered item | methods (steps) |
| `**Term**: …`, `**Term** — …`, `Term: …` in a list; `Long Form (LF)` whose initials spell the abbreviation | key concepts (term and its sentence) and concepts |
| `**bold**` phrases of up to six words | concepts |
| other bulleted items | structured extracts |
| block quotes | quotes |
| `concept: …`, `claim: …`, `method: …` lines (this project's own labelled-line format) | the section they name |

Headings are not made concepts: `Introduction` or `Ingredients` would become
one concept shared by every document with that heading. Problems, limitations,
results and relations to other work have no structural shape, so a fallback
leaves them empty.

**How the note says what it is.** The note carries `fallback_extractor:
extractive-v1` (frontmatter of an imported source's note; `- fallback_extractor:`
in a native note's normalization provenance) and `fallback_found_nothing_for`,
the kinds of content the source gave it nothing for. An imported source's
`## Provenance` adds `- Extraction: extractive-v1 fallback; the content is
copied from the source's own sentences, none was written; nothing found in
the source for: …`.

**Format repairs stay, stated generally.** Rejoining a small-caps heading that
PDF extraction split (`II. R ELATED W ORK`, only when every word has that
shape), recovering a paper's title from its first heading or a title block
rendered as a table, and dropping a publisher's boilerplate lines (DOI,
copyright, ACM reference format, ISSN) are kept: they repair how text arrived,
not what it says.

**A source's own wiki links become their words** (chores after Track AM,
investigated and kept). A Markdown source that writes `[[some-note]]` or
`[[some-note|its words]]` gives a note with `some-note` or `its words` and no
link. Where: the extractive fallback's markup removal (`extractive.clean`,
the deterministic path), and the template draft's and the normalizer's
`_strip_markdown_markup`, which clean titles and list items on both paths;
the converted copy keeps the source as written, links included, so nothing is
lost and the note's provenance names that copy. Why it is kept: the link
names a file in the author's own collection, by a name this collection does
not share. In a note here, `[[...]]` names a concept or a note of this
collection, and the sections the pipeline writes are checked for links that
open nothing (lint, `unresolved_links`). Carried in as written, such a link is
a broken link, or, when a note here happens to share the name (`[[introduction]]`,
a daily note's date), a wrong one, made silently; turned into a relation it
would be the same guess with an edge. A Markdown file whose links do belong to
this collection is the owner's own note: placed in the wiki it is kept as
written (a native note), its links working in an editor and its `## Relations`
bullets and relation fields read as typed edges. Test:
`tests/integration/test_source_wiki_links_api.py`.

**The guard.** `tests/system/test_fallbacks_only_know_their_input.py` runs
every deterministic fallback (the extractor, the heuristic normalizer, the
heuristic path for a legacy templated paper, the whole intake with no agent,
and the cluster-theme scorer) on synthetic sources from unrelated domains (a
recipe, a lease contract, a technical report, German meeting minutes,
Japanese notes) and fails on any content line the input does not contain. It
keeps no list of forbidden words, which would itself encode a corpus.

### What was removed (the inventory, P1)

Recorded so nobody puts it back. Described, not quoted. Counts are of string
literals in the bodies of the functions removed or rewritten, measured on the
code as it was before the change.

- **Source pipeline, academic fallback** (`_extract_academic_paper_draft_from_texts`
  and twelve helpers): a fixed summary sentence naming one control technique,
  one class of edge systems and one paper's method acronym (two variants,
  chosen by whether two subject words appeared); two fixed problem statements
  about that field, one written for *every* paper that matched no cue; two
  fixed method sentences; two fixed claims naming the acronym; two or three
  fixed questions written for every paper; two fixed evidence sentences
  (including a simulation count from one paper); a fixed "no limitations were
  extracted" sentence filed as a limitation; a seven-entry map from that
  field's terms to concept labels; section finders keyed to that paper's
  headings and to two joined-word extraction artifacts of it. 14 functions,
  82 string literals in their bodies.
- **Normalization, heuristic renderer**: concept selection weighted by six
  subject words; key concepts admitted only if they contained one of sixteen
  subject tokens and explained by ten fixed sentences about that field;
  method, claim and evidence detection by about fifty phrases from one
  advisor's notes and one field's vocabulary; summary selection that skipped
  nineteen phrases from one tool's transcripts and preferred seven of one
  author's phrasings; a "research review" detector keyed to one document's
  section names, with its own eight-entry concept list and evidence taken only
  from two sections named after that corpus's topics; an "academic paper"
  renderer with seven subject-specific concept cues, eight method cues and
  five claim cues from one paper, two fixed questions about that paper's
  subject, and a question invented about the title when none was found (also
  in the legacy templated-paper path, which also turned problem statements
  into questions). 32 functions, 314 string literals in their bodies (some of
  them ordinary English question words rather than one corpus's terms; the
  functions went whole).
- **Normalization, model-input compaction**: two recognisers for one tool's
  transcripts and one author's advisory notes (by speaker names, a tool's
  status lines and the author's phrasings) that dropped every line beginning
  with one of about twenty phrases from any note they matched, before the
  agent read it. 7 functions, 49 string literals.
- **Normalization, PDF line filter**: one paper's running header (two rules),
  one journal's ISSN, and four terms from one paper's classification block
  were dropped from every PDF. 7 rules; the publisher boilerplate around them
  stays.
- **Community detection**: cluster labels scored up for four subject words and
  down for five tool status phrases: 1 function, 11 string literals; now
  scored by a label's connections within its cluster.

Model prompts that carry examples from the same corpus are not fallbacks and
were left for a separate change; see the journal entry.

### Finding the notes the old fallbacks wrote (P4)

`POST /maintenance/fallback_notes` `{dry_run: true, access_mode, notes?,
include_current: false}` reads every source note, with no model call, and
reports:

- `legacy_fixed_content`: lines matching the **fingerprints** of the old fixed
  sentences (`second_brain/maintenance/legacy_fallback_fingerprints.json`:
  27 sentences as hashed five-word shingles of their constant parts; the
  sentences are not in the repository). A line matches when it holds four in
  five of a sentence's shingles, so a title spliced into the sentence does not
  hide it;
- `legacy_fallback`: a note whose provenance says a deterministic fallback
  built it (`template_runtime_mode: deterministic_fallback`, normalization
  `mode: heuristic`, or `templated_wiki_runtime_mode: skipped_heuristic`) and
  which has no `fallback_extractor` mark, so the old rules chose its content;
- `extractive_fallback`, only with `include_current: true`: notes the current
  fallback built (their sentences are the source's own; an install that runs
  without an agent on purpose would otherwise see every note).

A dry run writes nothing. A real run needs `access_mode: maintenance_write`
(403 otherwise) and opens one `fallback-note:<note>` review item per note,
listing the reasons and matched lines, and offering re-extraction
(`POST /sources/process` for an imported source, `request_renormalization`
for a native note), which costs tokens and is the owner's choice. An
owner-maintained note is reported and offered nothing. A resolved or dismissed
item is never reopened. No note is ever changed.

## Ingesting a URL

`POST /sources/process` takes **either** a `path` to a file already in the
workspace **or** a `url`:

```json
{"url": "https://example.com/docs/setup", "kind": "webpage"}
```

A URL passed in `path` is accepted too — an agent holding a link will use
whichever field it sees first, and rejecting it over which parameter it arrived
in helps nobody.

The page is retrieved into `sources/raw/` and then treated as an ordinary raw
source, so conversion, templating, provenance and ingest are all unchanged.
**Where the converter is.** Read from the install, never assumed: the
workspace's `conversion.converter_command` in `config/sys_config.json` when
set, else `oc-markitdown` on PATH, else its default install location. It used
to be only that last one, so an install that put it anywhere else never found
it. `GET /ops/self_check` says whether it is there and how to enable it.

`oc-markitdown` converts a local file and does not fetch, which is why this step
had to exist: before it, a link failed the file-existence check and came back as
`source not found`, naming the page as the problem when nothing had tried to get
it.

What the fetcher will not do:

- **Anything but `http`/`https`.** `file://` would turn the parameter into an
  arbitrary local-file read.
- **Follow a redirect off the web**, checked on the final URL, not just the
  first one.
- **Save a non-document.** An installer or a video is refused by content type.
- **Return an empty or error page.** A note built from a 404 reads exactly like
  a note built from the real thing, so the failure is loud instead.

It sends no credentials and identifies itself as `kovium/1.0`. This is not
a model call and cannot be billed; `tests/system/test_billing_safety.py` names
the module explicitly so that stays a recorded decision rather than an accident
of what the import scanner happens to check.

### Read-only, and only the open web

Enforced rather than intended:

- **GET and nothing else.** No form is submitted, no script is run, nothing
  changes on the far side. A page is a document to be read, not a service to
  interact with.
- **No identity, kept or presented.** The opener is built with no cookie jar and
  no authentication handler, so nothing is stored between fetches and a `401` is
  an answer rather than a prompt to send a credential.
- **Nowhere that is not the open web.** A host resolving to a private, loopback,
  link-local or reserved address is refused — checked on the first URL *and* the
  final one, since a redirect is the obvious way to send a fetcher somewhere it
  would have refused to go directly. Without this, `127.0.0.1:11434` (the local
  model server) and `169.254.169.254` (cloud metadata, whose purpose is handing
  out credentials) would both be reachable.

The address check resolves the name here and connects a moment later, so a
hostile DNS server could in principle answer differently each time. Closing that
properly means connecting to an already-resolved address, which `urllib` does
not expose. It raises the cost of the obvious attack and is documented as a
limit rather than assumed away.

## Fetched Content Is Data, Never Instructions

A page is written by someone else, can change after anyone reviewed it, and is
read by an agent whose own instructions are also text. The whole risk is that
the two get confused — that a sentence *in the page* is followed as if it came
from the operator. There are two moments where that can happen: the extraction
agent reading the page now, and some other agent reading the note built from it
months later.

Three layers, applied in that order.

**1. Scan.** Every source is checked for text addressed to an AI reader —
override phrasing, role reassignment, message markers, exfiltration and
credential-hunting instructions, tool-call shapes. Local files are scanned too:
the risk is not web-specific, since a page saved by hand and dropped into
`sources/raw/` reaches the same agent.

The discriminator is **delivery, not vocabulary**, and it is the part most naive
versions get wrong. A security article explaining prompt injection contains
"ignore all previous instructions" in plain sight, and it is a perfectly good
thing to ingest — refusing it would break the feature for the reader most likely
to want it. A real attack hides the same sentence where a human reviewer will not
see it. So:

| where the instruction is | verdict |
| --- | --- |
| hidden — HTML comment, `display:none`, `aria-hidden`, off-screen, inside `<script>`, behind zero-width characters | **blocked** |
| visible in the prose | **flagged**, recorded, ingested |

**2. Sanitize.** What is written to disk is the cleaned form: invisible and
control characters removed, and the text of hidden elements dropped. None of it
is content, so nothing a note needs is lost — the page `<title>` is deliberately
treated as visible, since it is what the note is named from. This runs even when
the scan is clean, so the text handed on is the text that was judged.

**3. Frame.** Source content reaches the agent wrapped in explicit markers, with
a preamble saying it is data to be catalogued and not instructions, whoever it
claims to be from. Every extraction prompt carries the matching rule. A document
cannot close the wrapper early — pre-existing markers in the text are
neutralised. This is the layer that keeps working when the scanner misses
something: the scanner catches what it has seen before, the frame applies to
every document including the ones that look clean.

### Quarantine

A blocked page is **held, not deleted**. Deleting would destroy the evidence and
leave the operator with a refusal and nothing to look at, and the scanner reads
patterns rather than intent, so it can be wrong.

The file is moved into `<workspace>/quarantine/<entry_id>/` with the scan that
condemned it, and recorded in an append-only ledger. It never lands in
`sources/raw/`, so no later run picks it up as though nothing had happened, and
it never becomes a note — which is what makes one control cover both moments
above.

- `GET /ops/quarantine` — what is held, and why. `?entry_id=` for the full scan.
- `POST /ops/quarantine/release` — override, deliberately. The file returns to
  `sources/raw/` and **nothing is re-ingested**; what happens next is a decision.
  The ledger keeps the block *and* the override, because that sequence is what an
  audit needs to see.

`/ops/status` surfaces a held page rather than letting it be forgotten.

### What this does not claim

Not a guarantee. A sufficiently novel phrasing in visible prose will pass, and
no scanner can recognise intent. This raises the cost of the easy attacks and
makes the rest visible in an audit trail; the durable defence is layer 3, which
does not depend on recognising anything.

The result carries a `retrieval` block — the URL asked for, where it actually
resolved, the HTTP status and the moment it was read — and those land in the
note's frontmatter. A web page is edited in place with nothing announcing it, so
`retrieved_at` is what bounds every claim the note makes.

**Poll with `wait`.** `GET /brain_keeper/jobs/{job_id}?wait=60` blocks until the
job settles rather than returning immediately. Without it the only way to follow
a job is a busy loop, and for an agent caller each poll is a whole turn
re-sending a growing context: measured on a live ingest, the work cost 180,319
tokens and the loop watching it cost about **1.03M — 5.7x the job itself**. One
waiting call replaces roughly ten polls.

The reply distinguishes the two outcomes explicitly, because both arrive with
status 200: `settled: true` means finished, while `settled: false` plus a
`job_still_running` warning means the wait expired and the caller should call
again. The wait is capped server-side so a connection is never held open
indefinitely.

This route and the job list are the only `async` routes in the API. Every
other endpoint is sync and runs in Starlette's bounded threadpool, where a
sleeping handler would hold a worker for the whole wait and a handful of
waiting callers could stall the server; an async handler holds no worker while
it sleeps.

**A status read never waits behind other work (Track AI1).** Each read borrows
a thread from a small pool of its own (`JOB_STATUS_THREADS`, 4), never from the
pool every synchronous route shares, so forty long requests in flight cannot
queue it. Each read also has a deadline (`JOB_STATUS_READ_DEADLINE_SECONDS`,
10 s). Past it, the reply is a 503 refusal, `job_status_unavailable`, with
`Retry-After: 5`. The job itself is unaffected, and the caller reads its state
again rather than resubmitting it. So a status request answers within `wait`
plus one read's deadline, whatever else the server is doing. Job records are
written whole: a temporary file beside the record, then one atomic replace. A
read can no longer meet a record half-written by the job (or by an MCP session
writing the same folder) and answer `job_not_found` for a job that is running.
On Windows a reader holding the record refuses the replace; the writer tries
again after a growing pause (5 ms up to 100 ms) for up to 10 seconds of elapsed
time, not a fixed number of tries (2026-10-25). Past that bound it raises
`JobRecordNotWritten`, naming the record: the last whole version stays on disk,
the temporary copy is removed, and the job's next save writes the new state.

**Lost a job id? List the jobs (Track AI2).** `GET /brain_keeper/jobs` returns
the workspace's most recently active jobs, newest first: `job_id`, `skill`,
`status`, `outcome`, `outcome_reason`, `agent_id` (who submitted it), `subject`
(the source a pipeline job processes, as the workspace records a path), and
`created_at`, `updated_at` and `duration_ms`. It is filtered by `status`,
`skill` and `submitted_by`, and bounded: `limit` defaults to 20 and is at most
200. At most 500 records are read for one listing; `truncated: true` and the
`job_list_truncated` warning say the scan stopped before the oldest record. An
instance keeps every job as provenance, and some records carry a whole pipeline
result, so reading them all would make the listing slower the longer the
instance lives. The `job_not_found` refusal names this list as its next call.

**Status says where a job is; `outcome` says what it came to (Track AI3).**
Every job reply carries `outcome` and `outcome_reason` beside `status`:

| `outcome` | meaning |
| --- | --- |
| `ok` | finished, and the result says nothing is wrong |
| `needs_review` | finished, and the result holds something back for the owner to decide, such as a book whose chapters were held for review |
| `failed` | did not finish, or finished with a result that says it failed |
| `cancelled` | stopped on request |
| `interrupted` | the server running it stopped before it finished (status `failed`); submit it again if it is still wanted |
| `null` | still pending or running |

It is read from conventions every result follows, never from one skill's field
names: `ok: false` with its `reason`, a `needs_review: true` flag, or a
`status` field naming a review or a failure. A skill added later is covered by
the same rule.

**A job whose server stopped is not `running` forever.** A job runs in memory,
in the process that created it, and each job records that process as its
`owner`: a token unique to it, the machine's name and the process id. While a
process has jobs in flight, it touches a small file of its own,
`brain_keeper/jobs/.owners/<token>`, every 30 seconds. It stops when it has
none, so a synced folder is not touched while idle. A job that says pending
or running is corrected to `failed`, with outcome `interrupted` and its
reason, the first time any reader sees it, in any of these cases:
- its owner ran on this machine and that process is gone (seen at once);
- its owner's heartbeat has been silent for 3 minutes, which covers a job
  from another machine sharing the folder;
- it was written before jobs recorded an owner, and has not moved for a day.

The correction is written back, so every later reader agrees, and a waiting
status read settles at once with the `job_interrupted` warning. A job just
created elsewhere is given the 3 minutes for its heartbeat to arrive.

A settled reply also carries the warning
`job_outcome_needs_review`, or `job_outcome_failed` when a job that
`succeeded` returned a failed result, so a caller reading only the status and
the warnings is no longer told a held-back result was a success.

**An unchanged source costs no agent calls.** Per-batch extractions are cached
under a key covering both the source text and the extraction prompt, and are
replayed on any run — not only a repair. Re-extracting a document that has not
changed is the most expensive thing this pipeline can do for no result: one
43.5 KB chapter measured at ~1.33M tokens, and a re-run paid that again to
produce the same note.

The prompt is part of the key because a cached payload is only valid for the
prompt that produced it; serving pre-edit answers under a post-edit prompt is
the worst kind of stale, since the note looks current. `refresh: true` on
`/normalize/file` ignores the cache, for when extraction *behaviour* changed in
a way neither hash can see. Since Track S it also skips the brain keeper's
answer cache, which used to answer the same batch with the same stored answer,
so a refresh did not actually reach the agent.

**An imported source is not extracted twice either (Track S).** `/sources/process`
reuses an extraction already given for the same text, template prompt, skill
definition, model tier, facet vocabulary and instance tail. The concept names
the extractor is shown (`known_concepts`) are advice and not part of that key,
so a collection that grew does not make every unchanged source an extraction
again; a reused answer's concept names are mapped onto the concept layer's
current wording before the note is written. The result says
`reused_extraction`, why, and the tokens saved; `refresh: true` reads the
source again. See `docs/prompt_system_v1.md`, "What changing a prompt costs".

A cache-served note is byte-identical to a freshly extracted one: the run
metadata is cached beside each payload, so a replayed batch still reports which
prompt and transport produced it. A cheaper note must not be a less accountable
one.

## The owner's own work — a Note node and its links, nothing else

A `type: owner_work` note (Track K, `docs/owner_work_v1.md`) is the owner's
design or their commentary on someone else's work. Its text is read from where
the owner keeps it (the registration's `source:`, resolved like any recorded
original, `<root>:<path>` included), never changed, and parsed into its parts
from its own headings. Ingestion gives it a Note node (`epistemic_tier:
owner_work`, the parsed note under `owner_work`, the title of the owner's text)
and three edge types no other note writes:

- `WORK_ABOUT` to each concept it names;
- `COMMENTS_ON` to the source note a commentary discusses, when the collection
  holds it (resolved by key, citekey, title, identifier, or author and year);
- `POSITIONS_AGAINST` to each work it compares itself with.

The body is not walked: no line becomes a Claim, Question, Concept or
Limitation, so ranking, limitation clusters, gap themes and concept synthesis
never see it as a source. Resolution of what it discusses runs at every ingest,
so a work added to the collection later is linked when it arrives.

## Typed relations — how one note bears on another

**The fourth layer of cross-document linking.** Phrase-shape concept links,
facets and optional passage embeddings all improve *ranking*; none of them
produces a relation an agent can follow. A typed relation does, and it is what
makes "what must be understood before this?" answerable.

**The vocabulary has one home.** `config/relations.json`, read only through
`second_brain/notes/relations.py`, with a per-workspace override and a guard
test (`tests/unit/test_relation_spec.py`) that fails the build if a second list
of relation names appears in code. The shipped set is `requires`, `builds_on`,
`contradicts`, `example_of`, `part_of`, `follows`, `version_of` (Track AJ: two
versions of one work, written by adding documents on the new version's note);
each declares an inverse name or declares itself symmetric. A declared name becomes the edge type mechanically
(`requires` → `REQUIRES`), so a workspace that declares `blocks` gets `BLOCKS`
with no code change. Declarations are refused — and reported by
`GET /ops/relations` — when they collide with an edge type that already means
something else, when two relations claim one inverse, or when a symmetric
relation also names one.

`supersedes` / `superseded_by` are deliberately **not** relations. They are
document lifecycle and stay in `notes/contract.py::LINK_FRONTMATTER`, so
following a `requires` chain never wanders into replacement history.

**Two ways to write one**, meaning exactly the same thing:

```markdown
## Relations
- requires [[canal-survey-method]] :: "the survey must be complete and closed first"
```

```yaml
part_of: overview-note
follows: previous-section
```

The token is matched as an identifier, never as a word, so the syntax is
language-neutral and the evidence after `::` is quoted in the note's own
language. The frontmatter form is how the pipeline links the pieces of a split
document; every relation name and every inverse name is accepted as a key.

**Stored once, read from both ends.** Each relation has an inverse, and writing
the inverse stores the same single edge pointing the other way:
`- required_by [[a]]` in note B becomes `a -requires-> B`. Three consequences
that are the point of the design:

- an existing note gains connections **without its file being rewritten**,
  which would be invisible write-back and would collide with human edits;
  the one exception, since Track AF, is its `## Links` section, which the
  pipeline owns, nobody edits, and the graph is never read from: it is
  refreshed from the graph when a neighbour is ingested, and the ingest
  result names every note refreshed (`links.neighbours_refreshed`). Nothing
  else in the file changes, and the edge set does not depend on it;
- the **order documents arrive in does not change the edge set**, measured by
  `scripts/eval_relations.py`, which ingests the fixture in both orders and
  fails if the two differ;
- **backlinks are computed when read** rather than stored. The note's
  `## Backlinks` section was never computed — it only copied forward whatever
  the previous version held — and incoming relations are the honest version.

**Edge ownership (schema v5).** Re-ingest replaces the edges a note *owns*, not
the edges leaving it. `owner_id` names the note whose text asserted the
relation; NULL means `from_id`, which is every edge written before the column
existed, so no rebuild is needed. Without it a relation stated from the far end
would be deleted by the note that never asserted it and kept by the note that
should have replaced it. When both notes state the same relation, one edge
exists and the last ingested note owns it; whichever note still asserts it
re-writes it, so the edge survives until neither does.

**Refusals and reports.**

- A token the install does not declare produces **no edge**, a
  `unknown-relation` lint error, and an entry in `GET /ops/relations`. A
  guessed relation is indistinguishable from an asserted one downstream, so it
  is worse than a missing one. It is an error rather than a warning because,
  unlike a dangling link, it does not resolve by itself.
- A target that has not been ingested is **allowed**. The edge is honest and
  completes itself when the target lands, the rule `supersedes` already follows.
  Lint reports `unresolved-link`, a warning.
- `GET /ops/relations` reports the resolved vocabulary, counts by relation and
  by kind, dangling relations, refused declarations, misspelt tokens, and
  `legacy_pattern_edges`.

**Reading them.** `GET /query/relations?node_id=` (both directions, incoming
named by inverse), `GET /query/relation_chain?node_id=&relation=&direction=`
(the transitive walk, nearest first, generic over any declared relation), and
`POST /query/path` with `both_directions: true` (off by default, because a
relation is stored once and crossing edges backwards finds routes the route has
never reported).

**Still stated-only.** Every relation is one a note asserts; `kind` is always
`stated`. Inferring relations by reading pairs of notes needs a model and is
Phases 2–5 of `docs/typed_relations_plan_v1.md`. The `kind` field exists now so
that inferred relations will be tellable apart when they arrive.

The old English `RELATION_PATTERNS` path (`X -> uses Y`, matched with English
regexes) still produces edges and is **not** retired: 51 lines across 23 test
files use it, and removal is gated on `legacy_pattern_edges` being zero in real
workspaces rather than on the assumption that nothing emits them.

## Connecting documents to each other

Extraction produces one set of concepts per document, and nothing joins them.
Measured on a live six-chapter corpus: **255 concept nodes, 86% of them known to
exactly one note.** The graph had no path between documents, so a query returned
whichever document phrased an idea closest to the wording used and never learned
the others existed.

**Merging the concepts is the wrong fix.** `session store`, `session boundaries`
and `session manager` are different ideas, not spellings of one; collapsing them
destroys a distinction the extraction correctly found. Exact-form normalisation
was measured on that corpus and merged *zero* nodes.

They are **related** instead. `second_brain/graph/concept_links.py` emits a
`BROADER_THAN` edge whenever one concept's words are a strict subset of
another's: `gateway` is broader than `gateway routing`. That relation follows
from the shape of the phrases alone — no domain knowledge, no vocabulary, no
model, no tokens — which is what makes it work for any corpus in any field. It
is derived after a document lands, because the relation is *between* documents:
a concept only gains a broader form once another note contributes one.

A concept note's own `broader:` frontmatter (the hierarchy a person or the
agent declared, which word containment cannot see) is ingested as
`BROADER_THAN` edges too, owned by that note and carrying its provenance. The
derived pass withdraws only edges it derived (those with no provenance), so it
never removes a declared one, and it does not add a second edge for a pair a
note already states. Words are runs of letters and digits in any script, and
graph ids keep letters of any script too, so a concept named only in Greek or
Chinese has its own node rather than `concept:unknown`.

Two guards keep it honest. Stopwords are excluded, so `of` never becomes a
parent. And a concept linking to more than `MAX_NARROWER_PER_CONCEPT` others is
dropped as too generic — a single common word would otherwise become the parent
of everything containing it and drag unrelated documents into every answer.

**Retrieval walks those edges.** A concept matching the query is the entry
point; `BROADER_THAN` widens it to narrower forms, and every note mentioning any
of them is *reached*. Reached notes then get a place in the result set, because
scoring alone cannot deliver this: concept reach is a property of the note, so
every node in a reached note carries the same bonus and lexical match still
decides the order — the exact thing that was hiding whole chapters.

Two rules that took measurement to get right:

- **The best match always leads.** Breadth belongs in the places after it.
  Displacing the top result cost 2.4 points of top-1 accuracy on the benchmark
  for no gain: "what does the corpus say about X" is answered by the set, not by
  which item is first.
- **A note contributes its best item *about the query*, not its best item.**
  Filtered on the tokens that actually named a concept, because matching on
  query words like "what" let any node in a reached note pose as relevant — a
  live query for "session" pulled "what host setup best fits a given threat
  model" out of a chapter that does discuss sessions.

What this cannot do is link ideas sharing no words. `heartbeat` and `cron job`
are related and this will never say so; that needs meaning rather than
structure. The closed facet vocabulary (`CLOSED_FACETS`) is the designed answer
for that and is currently unpopulated in every corpus seen so far.

## Cost — every agent call is counted

`second_brain/metrics/` records **one JSONL line per agent call** in
`<workspace>/logs/agent_calls.jsonl`, and `GET /ops/metrics` aggregates them.

**Every brain-keeper call, wherever it comes from.** The brain keeper records
each skill call itself, where every skill passes, as an operation named after
the skill (`operation: interpret_gaps`, `curate_concepts`, ...), with the same
fields as every other record. Before this only `/sources/process` recorded its
calls, so gap interpretation and concept curation cost tokens nobody could see.
Plain prompt calls (query widening) are recorded too. A caller that records its
calls against its own operation, with the batch and source detail the keeper
does not have (normalization, `/sources/process`), wraps them in
`counted_by_caller()`, so no call is recorded twice. Each backend tried is its
own record, a failed one included; an answer served from the answer cache is
recorded as `reused`; the deterministic tier is not an agent call and is not
recorded.

This exists because the backends were already reporting cost and the platform
was discarding it. OpenClaw puts usage in `result.meta.agentMeta.usage`, in the
same object as the `sessionId` and `durationMs` that *were* being read; Claude
Code puts it in `usage`. So an investigation into where 1.33M tokens went had to
be run against agent session trajectories by hand.

Each record carries the run it belonged to, the operation, the source, the
skill, the model, the characters sent, the duration and the usage. Records are
kept **flat** on purpose: grouping is a read-time concern, so
`group_by=operation|skill|source|model|backend|run_id` all answer different
optimisation questions off history already collected, with no re-instrumentation.

The derived field to watch is `tokens_per_1k_chars`. It is what makes the
dominant waste legible: a call costs roughly the same in fixed context whatever
it carries, so many small calls show up as an enormous ratio. A measured trivial
call — "Reply with exactly: OK" — cost **31,352 tokens**.

Three rules it follows:

- **Unknown is never zero.** A backend reporting no usage produces
  `tokens_available: false`. "This cost nothing" and "we do not know what this
  cost" lead to opposite conclusions when hunting waste, and a total assembled
  from incomplete readings is reported as a floor via `tokens_complete: false`.
- **Metrics never fail the work they measure.** Every write is best-effort; a
  full disk must not cost an ingest its note. Records are written as they
  happen rather than buffered, so a run that crashes half-way still shows what
  it had already paid for.
- **Nothing is written outside a workspace.** A service built outside the
  layout counts calls in memory and persists none, exactly as the batch cache
  does — derived state has no business in the checkout.

A replayed batch is recorded with `reused: true` and excluded from
`agent_calls`, so a cache-served run is visibly cheap rather than looking like a
run that never happened.

## How much of a note is kept

Each section of a note keeps **one item per 1,000 characters of source, never
fewer than 8 and never more than 60** (`item_limit_for` in
`normalization/batching.py`).

A fixed limit was the largest cause of knowledge loss in this pipeline. Two were
stacked — a merge keeping 12, then a cleanup keeping 8 — and both were sized
when a document was read in one pass. Batching made the input several times
larger and neither moved. Measured on a 48,000-character chapter: the agent
found **53 distinct claims and the note kept 7**. There is now one limit, in one
place, and at that size it keeps 48.

The rate comes from what the agent actually produces: on that chapter, one claim
per 907 characters, one concept per 697, one piece of evidence per 1,659. One
per thousand sits in the middle, so the limit stops being a guillotine and
becomes a guard against runaway.

The floor keeps short notes exactly as they were. The ceiling exists because
past it a note stops being something an agent reads; anything that long should
be split into chapters instead.

**It scales on the source length, not on the batch count.** Tying it to batches
would mean a change to how the text is sliced silently changed how much of it
survives, and the batch-budget setting and this one would fight each other.

## Long Documents — batched, never truncated

**Length alone never sends a note to the fallback** (2026-10-03). Every
document type is read in parts when it is long: templated sources (web pages,
chats, meetings, transcripts...) at `normalization.batch_budget`, your own notes
the same, and — since this date — academic papers too. A paper is read in one
pass up to `normalization.paper_single_pass_chars` (default 120,000 characters)
and in parts of `normalization.paper_batch_budget` (default 60,000) beyond it;
before this, a paper always went in whole. Measured on a 144-page survey
(1.15M characters): one pass was accepted by Claude Code but cost 482,237
tokens and kept 5 claims, 5 pieces of evidence and 4 questions; a backend
with a smaller window refuses such a pass, which left only the deterministic
extractor. In 20 parts: 60 claims, 60 pieces of evidence, 53 questions,
826,619 tokens in all, every part about 21,000 tokens — inside every
backend's window. The fallback tiers run only when every part fails.

**Every part carries a map of the whole document** (`document_map`: title,
the top two heading levels, the opening 1,500 characters, wrapped as untrusted
text). A part read alone used to know only its index and heading; the skills
are told to extract everything the part supports, from the part only, and to
describe the whole document in the whole-document fields. Parts run
`normalization.batch_concurrency` at a time, and are merged in document order;
the merge now keeps every field a skill returns (a paper's `conclusions`,
`quotes`, `cites`, `authors`...), not only the template fields.

**What length still costs.** One note keeps at most 60 items per section, and
a document long enough to fill that is past the point where one note is worth
reading. Such a document is now split — see *Very Long Papers* below; the
survey above is the case that forced it. And when OpenClaw is the only
backend, a shaping pass whose input exceeds its 120,000-byte argument limit is
skipped: the note keeps the deterministic layout of the extracted draft rather
than the agent's polish.

**The batch budget is configurable** (`normalization.batch_budget` in a
workspace's `sys_config.json`, default 12,000 characters). It is a real
trade rather than free tuning: a bigger budget means fewer agent calls and
proportionally fewer tokens, but also fewer extraction passes over the same
text. Now that the item limit scales with the source instead of capping at 8,
those lost passes really do mean less reaches the note.

**The concept hint is configurable too** (`normalization.known_concept_limit`,
default 200). Extraction is shown the concepts the workspace already uses, most
widely shared first, so it reaches for existing wording instead of coining a
second spelling for an idea already in the corpus. How many fit is a judgement
about *this* install's corpus and its model's context window, so the shipped
number is only a default for an install we know nothing about. Set it to `0` to
switch the hint off entirely and get the prior behaviour. The hint is advisory
and never enforced on the way back in — concepts stay free-growing, so a
genuinely new idea still enters under its own name.

**Waiting facet candidates ride the same call** (Track AQ7;
`normalization.facet_candidate_limit`, default 20 per facet, and
`normalization.facet_candidate_chars`, default 6,000 characters for the whole
list, `0` for none). Beside `facet_vocabulary`, extraction is sent
`facet_candidates`: values other documents proposed that are not registered
yet, per facet of the note type, the best supported first, each with its
definition and how many documents asked. Where no registered value fits but
a candidate does, the agent proposes that candidate again, spelled the same,
which is a second vote. It is advisory to the answer cache
(`advisory_input_fields`), so re-reading an unchanged source still reuses its
extraction. A proposal of a value another document asked for is recorded with
`reuses_candidate: true`, and `facet_process` reports the share.

**"Does not apply" rides the same call** (Track AO1). Beside `facets` and
`proposed`, extraction returns `not_applicable`: facets of the note type whose
question does not arise for the document, which is a different answer from "no
registered value fits". It is accepted only for a facet the note's type carries
and the workspace has registered values for, and dropped when the same answer
gave the facet a value; a document read in parts keeps a facet only when every
part said so (`normalization/batching.py`). It is written into the note as
`facets_not_applicable: [...]`, and lint counts that facet as reviewed. Nothing
puts a value into the facet. The facet review (`POST /maintenance/refacet`,
skill `assign_facets`) returns it the same way. See
`docs/note_format_contract_v1.md`.

**The first vocabulary forms at the end of an intake** (Track AQ8). When the
vocabulary is still empty after a document's proposals are recorded, the
system registers the first values at once, as the nightly promotion would. It
takes the best-supported proposals, threshold one, at most ten. They are
written into the notes that asked, this one before it is ingested, and the
earlier notes are ingested again. It is free, and is reported as
`vocabulary_bootstrap` in the result.

Measured on one 48,000-character chapter, extrapolating from its observed
per-batch yields:

| budget | agent calls | claims extracted |
|---|---|---|
| 12,000 (default) | 5 | 53 |
| 20,000 | ~3 | ~45 |
| 30,000 | ~2 | ~40 |
| 60,000 | ~1 | ~20 |

Raise it for corpora where cost matters more than completeness; leave it for
corpora where the opposite is true.

**Batches are packed to the budget, not left at section size.** Splitting on
`##` decides *where* a document may be cut; it does not decide how much a pass
should hold. Adjacent sections are packed greedily up to the budget, because an
agent call costs the same fixed context whether it carries 2,000 characters or
12,000.

Measured on a live workspace before this: 134 extraction calls carrying 271,642
characters, a median of 1,962 each — 5.9x the calls the content needs. Across a
14-chapter book, 273 calls against 69, and at ~32k tokens of fixed context per
call roughly 6.5M tokens spent on repetition alone.

Packing is greedy and forward-only, so batches stay contiguous, ordered and
exhaustive. A packed batch spans several sections, so its recorded `heading` is
the first of them. The budget remains a budget rather than a cap: a trailing
fragment below `MIN_BATCH_CHARS` folds backwards instead of costing a whole
agent call, which can put the final batch slightly over.
**No ingested file may lose text.** A document larger than one extraction pass
is processed in full, in batches, and merged.

What this replaced was invisible rather than merely crude. Two paths disagreed:
`source_pipeline` sent the whole document uncapped, while
`WikiNormalizationService._prepare_text_for_model` returned `raw_text[:12000]`
with metadata saying only `{'mode': 'raw_trimmed'}` — no count, no flag. On the
largest document in the corpus that discarded **446,078 of 458,078 characters
(97.4%)** and produced a note that looked entirely normal. The number was ours,
hardcoded in the first commit with no recorded reasoning.

`second_brain/normalization/batching.py` now owns the policy, and both paths use
it. Deterministic — no model, no tokens.

1. **Outline first** (`document_outline`) — headings plus a figure, table and
   equation inventory. Free, and it says where the seams are.
2. **Split** (`split_into_batches`) — on `##` boundaries first, then paragraphs
   inside an oversized section, then sentences. Never mid-sentence: a claim cut
   in half reaches the graph as a confident fragment. Batches are **contiguous
   and exhaustive by construction**, so coverage is a property of the split
   rather than something to hope for. A span with nothing to extract — only a
   frontmatter block, headings and whitespace — is never a batch of its own: it
   joins the span after it, even past the budget. Sent alone, a 299-character
   metadata block produced "no transcript text was included", which reached the
   note's summary.
3. **Merge** (`merge_batch_payloads` / `merge_template_drafts`) — union, with a
   deliberately high bar for collapsing two statements into one.

   Two statements merge only when they are the **same statement**, or when one
   is a strictly more complete version of the other with nothing meaning-bearing
   added — and then the **fuller wording wins**, whichever batch it arrived in.

   A statement that adds a qualifier is a *different claim* and both are kept:
   `only`, `unless`, `except`, `not`, `may`, `less`, and the rest of
   `QUALIFIERS`. This matters more than it looks. The rule was previously
   substring containment in either direction, keeping whichever arrived first —
   and since the shorter, more absolute statement almost always arrives first,
   the bias ran consistently towards over-claiming:

   > "The cache is fast" + "The cache is fast **only when the working set fits in
   > memory**" → *"The cache is fast"*

   Measured on five such pairs, the old rule discarded the qualified version
   **5 times out of 5**; the current rule discards none. A redundant
   near-duplicate is a cosmetic cost; a lost qualification is a claim the corpus
   attributes to a source that never made it.

   Containment is compared at **token** granularity, never as raw substrings:
   `"the bound is 5"` is a substring of `"the bound is 50"` but not a token
   subsequence of it, and treating those as one claim discards a measurement.

   Tokens are **letters and digits in any script**. Until 2026-10-02 they were
   `a-z0-9` only, so a statement written in Cyrillic, Greek, Arabic, Devanagari
   or CJK normalised to nothing and was skipped as empty: every claim, concept
   and question of such a source was dropped here. `QUALIFIERS` is still an
   English list, so in another language a qualified statement and its bare
   version may be merged into one — the fuller, qualified wording is the one
   kept, so the merge errs towards the narrower claim, never the stronger one.

   **The merge carries every field a template prompt asks for**, and it runs on
   every draft, a single batch included — so a field missing from it never
   reaches the note at all. Lists are unioned as above; `key_concepts` are
   merged by name, first explanation kept, at most eight; whole-document strings
   (`title`, `description`, `tldr`, `approach`) come from the first batch, which
   opens the document. The shared `extract_template_draft` skill must likewise
   *allow* every such field: the agent matches its `output_schema` exactly.
   Until 2026-10-02 neither held for `key_concepts`, `tldr`,
   `relation_to_other_work`, `approach`, `key_results` or `experiments` (nor,
   in the merge, `description`), which is why `## Key Concepts To Understand`
   read `TODO` on pipeline-made notes of every type but papers.
   `tests/unit/test_template_draft_fields_survive.py` reads the prompts and
   fails if any field they ask for is forbidden or dropped.

4. **Dispatch** — batches of one document run concurrently, bounded by
   `normalization.batch_concurrency` (default 2). The pool is **local to the
   call**: the brain-keeper's executor is two workers per workspace with a
   one-worker long lane, and dispatching batches through it would consume
   exactly the slots that lane protects. Each batch gets its **own session**,
   since parallel agent turns are only safe on distinct session keys.

   Results are collected **by batch index, never by completion order** —
   otherwise the same document merges differently on a re-run. Measured on a
   41-batch document: **2.09 s sequential vs 0.60 s at concurrency 4 (3.5×)**,
   with the merged note identical either way.
4. **Account** (`coverage_for`) — anything short of 100% names the byte ranges
   that were missed. "3,000 characters were not processed" is not actionable;
   "characters 18,000–21,000 were not processed" is.

Every batch is told it is a fragment (`batch: {index, of, heading}`); an agent
that thinks it has the whole chapter summarises a section as one.

**A failed batch is retried before it becomes a hole.** Three attempts by
default (`normalization.batch_attempts`), with exponential backoff, because the
usual cause is a rate limit or a momentarily malformed answer and retrying
immediately asks the same question of the same exhausted budget. Validation
happens inside the retry, so an unusable shape is retried like any other
failure rather than being recorded as a fact about the source. Each attempt gets
a fresh session — reusing a failed attempt's session hands the agent its own bad
answer as context.

**A note with holes is repairable for the cost of the holes.** Successful
per-batch extractions are cached under `data/staging/normalization/`, keyed by a
hash of the prepared text and by each batch's byte range. `normalize_file(...,
repair=True)` replays them and calls the agent only for the ranges that failed:
measured, **15 agent calls on the first run, 1 on the repair**. A changed source
invalidates the cache entirely — cached extractions describe a document that no
longer exists, and splicing them into a new note would attribute old content to
new text. Repair is opt-in, so a plain re-run never silently replays stale
answers.

**A failed batch does not discard the ones that succeeded.** Its range is
recorded on the note so the hole is visible and the run can be repeated for that
range alone. Throwing away good extractions because one section failed is how a
long document becomes unprocessable.

Each batch payload is validated **before** merging: `merge_batch_payloads`
constructs a well-formed dict from whatever it is given, so merging first would
launder an invalid agent response into a valid-looking note.

The note records what it was built from:

```
- selection_mode: batched
- input_chars_original: 32344
- input_chars_sent_to_model: 32344
- batches: 8/8
- source_coverage: 100.00% (32,344 of 32,344 chars)
- batch 0: chars 0-10936 — Chapter 1 — <the chapter's first section>
```

A note built from eight passes over a chapter and one built from a single read
of its first 12,000 characters look identical on the page. They are not the same
note, and how much of the source was seen is the difference a reader most needs.

A note built from an imported source (`/sources/process`, the template path)
carries the same `source_coverage` line since 2026-10-09 (Track I7): the
extraction that drafted the templated source records
`template_source_coverage` in its frontmatter, and the wiki note's provenance
renders it beside `mode: <template>_template` and `template_runtime_mode`. On
such a note `input_chars_original` / `input_chars_sent_to_model` measure the
shaping pass over the templated draft, not the original. A draft no agent read
(`deterministic_fallback`) states no coverage; its mode and
`template_runtime_degraded_reason` say what happened instead.

`preprocess_mode` was renamed `raw_trimmed` → `raw`: the old name described a
truncation that no longer happens, and a mode name that lies about what was done
to the text is worse than none. Transcript and advisory sources keep their
*compaction* step — removing turn scaffolding is not the same as dropping the
tail — and their `[:9000]` caps are gone too.

## Very Long Papers — an overview note plus one note per section

**Reading a long paper in parts stopped it failing; it did not stop the note
filling up** (2026-10-03). The 144-page survey above, read in 20 parts, filled
Claims, Evidence *and* Concepts to the 60-item ceiling one note keeps, and the
parts had found more than that. The ceiling is right for one note — past it a
note stops being read and becomes a dump — so a document that long should not
be one note at all. It is split the way a book is: an **overview note** for
the whole paper and **one note per top-level section**, each with its own room.

**The decision is made before any agent call**, from the size and the paper's
own headings, and every condition must hold:

- the source is going through the `academic-paper` template and is not already
  a piece of something else (a section is never split again);
- the brain keeper agent is on — without one a section note would get only
  the extractive fallback's thin note, so the single note is the safer result;
- the converted Markdown is longer than `normalization.paper_split_chars`
  (default 300,000 characters; `0` switches splitting off entirely);
- the paper's own headings give a split worth trusting.

Otherwise the paper stays one note, read in parts as described above.

**Finding the sections is deterministic — no model, no tokens.**
`second_brain/normalization/section_split.py` tries three detectors in order
and keeps the first plausible result: the book detectors, then numbered
capitals, then Markdown heading levels. `segment_book` alone is not enough,
and not by oversight: it reports a paper's numbered sections as unconfident on
purpose, and a two-column PDF conversion leaves headings where no heading
parser expects them — at the end of a line, inside a table row, or with body
text running on from them.

**No split is better than a wrong one.** The split is refused, and the single
note written instead, when fewer than three sections are found, when one
section holds most of the document (the wrong level was found), when fewer
than two survive merging, or when the pieces do not reassemble into the source
byte for byte. That last check is made, not assumed, the same guarantee
batching and book splitting give.

Two shaping rules apply to the result:

- **Small sections merge into a neighbour** rather than becoming thin notes,
  below `normalization.paper_min_section_chars` (default 20,000).
- **A trailing reference list is set aside**, written beside the sections as
  `<stem>-references.md` so the pieces still add up to the whole paper, but not
  made into a note: a list of citations is not something to extract claims
  from. It is found by its *shape* — most lines carrying a year or a `[n]`
  marker — not by an English heading, so a paper in another language is
  treated the same. A section that merely cites a lot of years is not mistaken
  for it; the shape needs contrast with what comes before.

**What each note reads.** The overview note reads the paper's front matter,
first section and last section, capped at one part's budget so it is a single
call — a paper says what it is in its abstract and introduction and what it
found in its conclusion, and the sections between are read in full by their
own notes. Each section note reads its own section, in parts, as any long
document is.

**Every piece is given the outline of the whole** (`section_map`, the same
shape as `document_map` plus `this_part: {index, of, title}`). The outline
comes from the split itself rather than from Markdown headings, which a
converted PDF does not have. Each section note is also **named for itself**,
not for the paper: an agent shown the whole paper's outline sometimes titles
the piece after the paper, and every section would then compete for one note
name.

**The links are written once and read from both ends.** Each section note
carries `part_of: <overview>` and `follows: <previous section>` in its
frontmatter, which become `PART_OF` and `FOLLOWS` edges at ingest. The
overview note is never rewritten to receive them — neighbours include incoming
edges, so it shows its sections without its file being touched. Authors and
year are passed down from the overview's draft rather than re-extracted per
section.

**A failed section does not lose the others.** It is reported in the result
and the remaining sections still process. Running the same paper again repeats
only what failed, because answers already given come from the brain keeper
agent's answer cache.

`POST /sources/process` returns the overview's usual result with a `sections`
block added: `overview`, `split`, `section_dir`, `references_path`,
`sections_written`, `sections_processed`, `failures` and one entry per section.

### Settings

| setting | default | what it does |
|---|---|---|
| `normalization.paper_split_chars` | 300,000 | above this a paper is split; `0` switches splitting off |
| `normalization.paper_min_section_chars` | 20,000 | below this a section merges into a neighbour |
| `normalization.paper_batch_budget` | 60,000 | one part of a section, and the overview's cap |
| `normalization.paper_single_pass_chars` | 120,000 | below this a paper (or the overview) is read in one pass |

### Measured, live on the 144-page survey

The same source as above, run end to end through `/sources/process`:

| | one note, read in parts | split into section notes |
|---|---|---|
| notes | 1 | 1 overview + 10 sections |
| claims / evidence / questions / concepts | 60 / 60 / 53 / 60 (all at the ceiling) | 184 / 126 / 78 / 404 |
| agent calls | 22 | 43 |
| tokens | ~827k (extraction only) | ~900k (every call) |
| time | 6.5 min | 27 min |

No section failed and the deterministic extractor was never reached. All 10
sections point at the overview and 9 at the section before them. The reference
list was set aside and kept.

Two things about that run worth reading as *conditions*, not results: all but
the overview call ran on **Codex**, the main backend having apparently hit its
usage limit — which is the fallback working, and part of why it took 27
minutes — and the sections ran **one after another**, each read in parts in
parallel inside. The time is therefore an upper bound rather than the cost of
splitting.

### What this still does not solve

- **Two sections still filled the 60-item ceiling** — the two largest, at
  130,000 and 157,000 characters. A section that big wants splitting again,
  which is not implemented.
- **The overview note is thin** (4 claims, 3 pieces of evidence) by design: it
  reads only the opening, first and last sections. Whether it should instead
  be built from the section notes' own drafts is open.
- **Section titles are inherited verbatim**, so a converted PDF's mix of
  `5 POST-TRAINING` and `2 Overview` reaches the note names as-is, without the
  paper's own name in them.
- **Sections are processed sequentially.** Running them concurrently is the
  obvious next saving and is bounded by the same host-wide call cap as
  everything else.
- **The reference list is kept as a file, not a note**, so its citations do not
  reach the graph.

## Sections to Graph Node Types
Which `## section` a bullet sits under decides what kind of thing it becomes.
The mapping lives in `second_brain/notes/contract.py`
(`MANDATORY_SECTION_NODE_TYPES` and each `TypeProfile.section_nodes`), beside
the section lists it refers to, so the two cannot drift. Bulleted lines become
nodes, and so do **table rows** (see *Tables* below); headings match
case-insensitively.

**Mandatory — identical for every note type, always.** No type may override or
narrow these; that consistency is what lets the same question be asked of any
note whatever its type.

| section | node type |
|---|---|
| `Concepts` | `Concept` |
| `Key Concepts To Understand` | `Definition` |
| `Claims` | `Claim` |
| `Evidence` | `Evidence` |
| `Questions` | `Question` |
| `Structured Extracts` | `Extract` |

`TL;DR`, `Relation to other work` and `Provenance` are deliberately unmapped —
the first two are prose rather than one-idea-per-bullet, and the third is
bookkeeping.

`Figures` (Track M) is read apart from this table, for any type: each
`- Figure N (p. X): …` item is one `Figure` node *of that note*
(`figure:<note id>-<N>`, so one document's "Figure 1" is never another's), with
`metadata.derived_from` `caption` (confidence 0.85) or `image` (0.6, when the
item carries `Interpretation (from the image):`), below every text-derived node.
Other lines of the section make no node.

**Type-specific — what each type adds on top:**

| type | sections it contributes |
|---|---|
| `paper` | Methods→`Method`, Experiments→`Experiment`, Key results→`Result`, Conclusions→`Conclusion`, Assumptions & limits→`Limitation`, Problem→`Problem` |
| `meeting-note` | Decisions→`Decision`, Action items→`ActionItem`, Risks / blockers→`Risk` |
| `chat-transcript` | Decisions→`Decision`, Follow-up items→`FollowUp`, Disagreements / uncertainty→`Disagreement`, Referenced sources→`Source` |
| `white-paper` | Recommendations→`Recommendation`, Proposed approach→`Approach`, Assumptions & limits→`Limitation`, Risk and bias notes→`BiasNote` |
| `blog-post` | Main argument→`Argument`, Applicability notes→`Applicability`, Assumptions & limits→`Limitation` |
| `youtube-video` | Topic segments→`Segment`, Timestamped anchors→`Anchor`, Transcript quality notes→`Caveat` |
| `book-chapter` | Main argument→`Argument`, Prerequisites→`Prerequisite`, Worked examples→`Example`, Exercises & practice→`Exercise` |
| `ai-conversation` | What was established→`Established`, What was asserted→`Assertion`, Corrections→`Correction`, Dead ends→`DeadEnd`, Artifacts produced→`Artifact` |
| `voice-note` | Unfinished thoughts→`Fragment`, Action items→`ActionItem`, Transcript quality notes→`Caveat` |
| `decision-record` | Decision→`Decision`, Options considered→`Option`, Consequences→`Consequence`, Revisit when→`RevisitTrigger` |
| `webpage` | Reference details→`Reference`, Procedures→`Procedure`, Volatility notes→`Caveat` |

Before this, ingestion kept its own five-entry map shaped entirely around
papers, so **every type-specific section was dropped**: a meeting note's
decisions produced no nodes at all, and papers kept only `Methods`. Rebuilding
the current corpus with the contract-driven map took it from **739 to 964
nodes**.

`Evidence` was previously folded into `Claim`, collapsing the distinction the
note contract works hardest to maintain. It is its own type now (123 nodes in
the current corpus). `Definition` is new: that section reached the graph not at
all, so definitional context never ranked (162 nodes now).

**A note with no type, or an unrecognised one, gets the union of every known
type's sections**, not just the mandatory set. Dropping to the mandatory set
would be a regression rather than a purity win — a large share of any real
corpus predates the type contract, and their `## Methods` bullets would silently
stop reaching the graph. When the type is unknown the heading is the best
evidence available, and reading it beats discarding content. No two types may
map one heading to different node types, so the union is unambiguous; a test
enforces it.

## Access History — the one thing not derived from the Markdown
`last_read_at`, `read_count`, `first_read_at`, `last_retrieved_at` and
`retrieved_count` are merged onto every `Note` node when it is read. A note
nobody has read reports zeros rather than nothing, so a caller never has to tell
"unread" apart from "not tracked".

These live in their own `note_access` table, **not** in node metadata. Metadata
is derived from the Markdown and rebuilt from it wholesale, so a read count
stored there would be erased by the next rebuild. The row survives because the
note id is stable (see **Note Identity**) — it still matches the right note
afterwards. `scripts/rebuild_graph.py` preserves the history and drops rows only
for notes no longer on disk, after re-ingesting, where the live set is known.

Recording a read **never writes to the Markdown**. Doing so would churn git,
change the content hash, and re-trigger ingestion on every read.

Two counters, deliberately separate:
- **read** — the note's content was served to a consumer.
- **retrieved** — it appeared in a ranked result.

A note can rank for a hundred queries nobody acts on; merging the two would make
the signal measure retrieval noise rather than use. Recording happens at the
agent boundary (`AgentMemoryService`), not inside retrieval: one `answer()` call
ranks four times, so counting deeper would measure internal fan-out. Within one
request a note counts **once**, and never as both.

Known limit: identity is path-based, so **renaming a note starts its history
over**.

## Incremental Update Rule
Re-ingest only when the file content hash changes.

Re-ingest **replaces**, it does not accumulate. `store.replace_document()`
clears the edges leaving a note before writing the new ones, so a note edited to
drop a claim stops pointing at it. Without this the graph drifts away from the
Markdown it is supposed to be derived from.

Entity nodes are *shared* — `concept:scheduling` may be referenced by twenty
notes — so re-ingesting one note keeps every entity something else still points
at. What it does remove (since 2026-10-01) is an entity **this note pointed at
and nothing points at any more**: the store holds the whole workspace, so such a
node is referenced by no note anywhere. Leaving it was not harmless — a claim
edited out of a note stayed searchable under its old node type, and a claim moved
from `## What was established` to `## What was asserted` came back from search
as `Established`. Anything else with no edges, such as nodes left behind before
this rule existed, is still `prune_orphan_entities()`'s job, run by the rebuild
command below with the whole workspace in view. A rebuild is unaffected: it
clears every note's edges first, so no note has previous targets to remove.

## Rebuilding
`scripts/rebuild_graph.py` rebuilds a workspace's graph from its Markdown:

```bash
python scripts/rebuild_graph.py --workspace-root sandboxes/<id> --dry-run
python scripts/rebuild_graph.py --workspace-root sandboxes/<id>
```

It drops every Note node and its edges, re-ingests each note, then prunes
entities nothing references any more. Ingestion is deterministic Python with no
model calls, so a rebuild is free and can be run whenever the corpus and the
store might have drifted.

This is the command to run after any change to *what* gets ingested — otherwise
the store holds a mixture of old and new rules. It is also the repair for graphs
written before note ids were stable. `--dry-run` reports what would change
without writing.

## Error Handling
- malformed file should be logged and skipped, not crash pipeline
- extraction failure should still preserve parse record when possible
- partial ingestion should be resumable
