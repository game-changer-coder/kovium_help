---
type: "documentation"
title: "Equations read where the caller needs them v1 (Track N)"
description: "How an equation a converter found and could not read is kept, how a caller gets exactly the ones its question needs read (by the local formula model or by vision), and how an answer says which equations and figures its evidence could not..."
status: "stable"
reading_status: "read"
product: "second brain"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing second brain"
doc_path: "docs/equations_v1.md"
doc_sha256: "64b1b322f46ee6f713a8376a1e3d23d151a946b3db847cf0285189bfd5617c20"
---

# Equations read where the caller needs them v1 (Track N)

How an equation a converter found and could not read is kept, how a caller
gets exactly the ones its question needs read (by the local formula model or
by vision), and how an answer says which equations and figures its evidence
could not show. Plan: `docs/knowledge_layer_improvement_plan_v1.md`, Track N.
Builds on `docs/document_conversion_v1.md` (Track M). Equations written in any
source's text (LaTeX, a page's MathML) and equations that arrive as images in
a source that is not a PDF use the same catalog and routes, read from the text
or by vision: `docs/images_v1.md` (Track T3, T4).

## Why

docling finds every equation on a page with its layout model, but reads one
into LaTeX only with its formula model, which on a CPU takes minutes per
equation (measured: about 27 minutes for the 11 equations of an 11-page
paper). So it is off by default, and every equation stays a marker,
`[equation not recovered (p. N); see the source page]`. Reading them all
for every document is not affordable on a library, and is not needed: a
question that needs the model of a paper, the reaction of a protocol or the
pricing formula of a filing needs a few equations, and only the question
knows which. The brain cannot know in advance; it can keep each equation
ready to be read, say when evidence holds one, and read exactly those on
request.

## Read at night, on this computer (Track AG5, D10)

Reading during conversion (`formula_enrichment`) stays off, because it is slow
on a CPU; what is off by design must not stay off because nobody knew it
existed. So the nightly run (`scripts/nightly_maintenance.py`) reads the unread
equations of the notes that hold some, with the local formula model, for no
tokens: for each note, `POST /sources/equations` as a dry run first (the
estimate), then for real with `method: local`, `access_mode: maintenance_write`
and `wait`, within `equations_minutes` a night (`config/maintenance.json`, 60
by default, `0` switches it off). A note begun is finished; one that will not
fit in what is left of the night waits for the next; one whose estimate is
longer than the whole limit is left for a request made on purpose. What each
attempt left is remembered (`maintenance/equations_nightly.json`: the notes'
unread count and hash), so a note that could not be read is not planned again
until it changes. `method: local` is the one real run of this route that needs
no approval of a plan where `approval_required` is on: it spends no tokens, and
`method: local` never chooses vision. Any other method still needs its
approval. The code is `second_brain/maintenance/equations_pass.py`.

## N1. Unread equations are regions with identities

The docling worker (`second_brain/conversion/docling_worker.py`) records, for
every formula it finds, its page, its box on the page (top-left origin, in
the page's points), its order on the page and whether it was read. With
`equation_regions` (on by default in `config/conversion.json`), each one not
read is cropped from its page into an image: pypdfium2 (a docling dependency)
draws one page at a time at three times 72 dpi, so a long document is never
held in memory as page images. The pipeline keeps them beside the figures:

```
sources/converted_markdown/<kind>/<stem>.md
sources/converted_markdown/<kind>/equations/<id>/eq-<n>.png
sources/converted_markdown/<kind>/equations/<id>/equations.json
```

`<id>` is the figures' short id of the copy (twelve hex characters, never its
name). `n` is the equation's number: its place among every formula docling
found in the document, in reading order. The catalog (`equations.json`)
records for each: `number`, `page`, `order_on_page`, `bbox`, `page_size`,
`decoded_at_conversion`, the crop `file`, its `sha256`, `width`, `height`,
`sentence_before` and `sentence_after` (the text between it and its
neighbouring equations, within its section) and, once read, its `reading`.

The marker carries the number:

```
[equation not recovered (p. 4, eq. 7); see the source page]
```

The older forms, `(p. 4)` and no page at all, still parse everywhere a marker
is read (metrics, notes, answers, `/review/source`). A source without pages
(Track T3) names the equation alone: `[equation not recovered (eq. 7); see the
source page]`. A reading written in the source itself is
`$$…$$ _(eq. 3, read from the text)_` (`derived_from: text`, confidence 0.9).

**In the note.** Each templated and wiki note built from the copy gets a
`## Source equations` section, rendered from the catalog without a model
call, before `## Provenance`:

```
- Equation 4 (p. 2): [equation not recovered (p. 2, eq. 4); see the source page] — before: The spread of the n readings x_i around their mean mu is the variance sigma squared, computed as: — after: The suspended load P decays from its initial value…
- Equation 6 (p. 5; the source's (1)): $$…$$ _(eq. 6, read from the image by …; check against the source)_ — section: III. SYSTEM MODEL > D. An Optimization Problem — before: … — after: …
```

**The headings an equation sits under** (N-R8) are recorded with it
(`heading_path`: the nearest heading and those above it, recomputed for every
region at each write-back, so a `locate_only` refreshes a stale one, N-R17)
and shown in the item
(`— section:`) and the node. A document's outline is its strongest structural
signal: "D. An Optimization Problem" within "III. SYSTEM MODEL" says what an
equation is part of whatever words surround it. A converter does not always
keep the outline's levels (docling writes every heading `##`), so
`second_brain/equations/headings.py` reads them from the `#` marks when the
document's own headings use more than one level (at least two below the top;
the pipeline's figure blocks, with their `### Figure N` headings, are not the
document's: found live, N-R13), else from the numbering the headings show
(Roman numerals, capital letters, small letters, Arabic numbers with a level
per dotted part; the scheme met first is the top level; a lone `C.` or `V.`
is a numeral only where it continues the Roman sequence), else gives the
nearest heading alone. `section` selections on `POST /sources/equations` use
the same outline, so "III. SYSTEM MODEL" holds its "A.", "B.", … even in a
flattened copy.

**The names a source defines for itself** (N-R9,
`second_brain/notes/identity_terms.py`): a bracketed short name its opening
defines (`… genetic algorithm (LGSTO)`), written mostly in capitals, not a
Roman numeral or a number, used at least three times in the source, and
rare across the workspace's other sources: found in at most one of them, each
other source counted once (its converted copy, or a note with a source of its
own and no converted copy: a Markdown source, a webpage, the owner's own
work). A note the brain derived (concept notes, maps, ideas, proposals) is
never another source, and neither is any text citing this one by its recorded
path, its notes' names or its citekey as a citation (N-R16, found live: eight
concept notes built from the paper made its own name look common). A
source also brackets the baselines it compares against ("… (PSO)"), which
many sources name; the rarity rule keeps those out (N-R12, found live), and a
term no longer held is removed from the note. At most five. Chosen over "signature terms" (frequent here, rare elsewhere)
because a defined name is the document's own declaration, needs no collection
statistics that move as sources arrive, and pulls in no topic words that are
not names. Recorded additively as `identity_terms` in the frontmatter of the
notes built from the source (the pipeline, for every converted source; a
recovery or `locate_only` for older notes; never an owner's note); ingestion
appends them to the note's node label when its title does not say them, and
to every equation item's identity. Nothing else in a note changes.

A distinctive heading, so it never replaces a section an extraction or an
owner wrote. The words before and after each equation are the source's own
and name its symbols; the converter's HTML entities (`&lt;`) and Markdown
escapes (`m\_i`) are undone in them (N-R3). Once an equation is read from
its image, the document's own label for it (`(1)`, `(2.3)`), printed beside it
in the crop, is recorded as `printed_label` and shown, so "Equation 1 is
subject to …" in the text can be resolved to a region (N-R4). The local
formula model gives no label; it stays absent.

Ingestion makes one `Equation` node per item (`equation:<document>-<n>`;
`metadata`: `equation_number`, `page`, `printed_label`, `heading_path`,
`latex`, `read`, `derived_from`: `unread`, `image` or `formula_model`;
confidence 0.85, 0.6 and 0.8). **Its label is what a question finds it by**
(N-R1): the region, the document's title, citekey, up to three aliases and its
`identity_terms`, the headings it sits under, and the source's words before
and after it, then the LaTeX once read; never the marker. Ranking
gives `Equation` a type weight of 3.0 (the level of a question: on a tie in
words, statements come first, so a question that matches nothing is not
answered with formulas). A marker an extraction carried anywhere else in a
note is numbered and read the same way. A note its owner maintains
(`maintained_by: owner`) is never written. A note written before these
changes is brought up to date, at no cost, by a `locate_only` run on its
source (which, on a copy that already has regions, only rewrites the
section and re-ingests).

**A copy without regions says so**, in the dry run's
`regions.how_to_get`, with the cheapest way to get them:

| the copy | what it says |
|---|---|
| made by the basic extractor or markitdown (no markers) | a free local re-read with docling: `POST /sources/reread` `{pdf_converter: docling, reextract: false}` re-converts the copy and keeps its regions; `reextract: true` also carries them into the note, at the stated extraction estimate |
| made by docling before Track N (markers with a page, no catalog) | a layout-only pass (`locate_only: true`): local, free, no formula model, no tables, no figures, no extraction; its estimated seconds |
| made by docling, no marker | nothing to read |

**The layout pass** (`locate`) runs the docling worker over the original with
the layout model alone, crops each formula, writes the catalog, and gives the
copy's old markers their numbers page by page: in order when a page has as
many markers as regions (the same layout model on the same file), else by the
words before each marker against each region's sentence, a number only when
one region clearly matches best; the rest keep the old form and are reported
in `markers_not_numbered`. A real reading run on such a copy makes the pass
first by itself.

## N2. Reading only the equations asked for

```json
POST /sources/equations
{"note": "sources/wiki/<note>.md", "equation_ids": [4, 5], "method": "auto",
 "agent_id": "...", "workspace_id": "..."}
```

`note` or `source` (the original under any of its names, or the converted
copy) names the source. The selection is one or more of `equation_ids`,
`pages` (page numbers of the original) and `section` (a heading of the
converted copy or of its notes, matched whole or as its end, so `Estimation`
finds `2 Estimation`); none of them is every unread equation. **A selection
always resolves to region numbers and page numbers of the original**, so a
book (Track O3) can add a chapter selection that resolves to pages the same
way, without a change to this contract. An unknown number is listed in
`selection.unknown_ids` (warning `unknown_equation_ids`); a selection of
unknown numbers only is refused (422, `unknown_equation_ids` with
`known_ids`); a section in neither the copy nor its notes is refused (422,
`section_not_found`).

### The dry run (the default, allowed `read_only`)

Reads nothing, converts nothing, calls nothing, writes nothing; audited as
`source_equations_plan`. Returns:

- `regions`: `located`, `converter`, `unread`, `read`, `markers_in_copy`,
  and `how_to_get` when not located;
- `equations`: each region with its `status` (`would_read`, `already_read`
  with its reading, `not_requested`, `read_at_conversion`, `not_in_copy`),
  page, `printed_label`, the text before it and whether it has a crop;
  `would_read` and `would_read_count`. On a copy whose regions are not
  located yet there are no numbers: each marker is listed by page with status
  `would_read_after_numbering`, `would_read` is empty, `would_read_count`
  counts them, `numbering` says the layout pass gives the numbers first, and
  `methods.vision` gives `batch_sizes` instead of `batches` (N-R6);
- `selection.resolved_pages`;
- `methods.local`: `available` (docling installed, the original a PDF and
  present) and why not, `pages`, `formulas_on_pages`, `estimated_seconds`,
  `tokens: 0`, `basis`;
- `methods.vision`: `available`, `backend`, `model` and why not, `batches`
  (equation numbers per call), `calls`, `batch_tokens`, `estimated_tokens`,
  `over_budget`, `no_image`, `within_budget`, `basis`;
- `method`: `requested`, `chosen` and `why`. **auto** chooses vision when a
  backend can read an image and the whole selection fits the budget, else
  local when docling is available, else nothing, and says why.

### The real run (`"dry_run": false, "access_mode": "maintenance_write"`)

403 without `maintenance_write` (audited `denied`). Backgrounded unless
`"wait": true` (docling work in the long lane); audited as
`source_equations_read`. Keeps a rollback snapshot of the converted copy, its
equation regions and the notes, then:

- **local**: the docling worker with its formula model on, tables and
  figures off, converts **only the pages** the selected equations are on
  (`--pages`; a run of consecutive pages is one conversion, in one process);
  each formula it reads is matched to a catalogued region by page and box
  overlap (intersection over union of at least 0.3), or by order on the page
  when no box overlaps and the counts agree. The formula model reads every
  formula on those pages; only the selected ones are written.
- **vision**: the `read_equations` brain-keeper skill reads up to
  `max_per_call` crops per call (a call's own overhead dominates one small
  crop), each with the text before and after it and its crop's hash in the
  input (so the answer cache never replays a reading for a changed image).
  Same backends and confinement as `interpret_figure`: Claude Code with
  `--tools Read` alone in a fresh folder holding only the crops, Codex with
  each crop attached (`--image=`), OpenClaw and any text-only backend
  `vision_unavailable` before it starts. The budget is checked before each
  call from what has been billed so far; a call is made whole or not at all.
  An answer for an equation not asked for, or `unreadable`, is dropped.

A reading is written with its spacing normalized, whichever reader produced
it (N-R7): the formula model writes every token apart (`o t _ { j }`), a
vision reading as a person would (`ot_j`); in math mode a space means nothing
except after a command name, so spaces next to braces, scripts and brackets
and between single symbols go, a single-character script loses its braces,
`=` and a binary `+` or `-` get one space each side except inside a script
(`_{j=1}`), a term after a script is set apart (`x_{ij} a_i`), and the words
inside `\text{…}` are kept as written. The raw reading stays in the catalog.

Every response, dry run or real, carries `spent_tokens` (what was billed: 0
for a dry run, a reading reused from an earlier identical call, or a local
reading) and `spend_note`, a sentence a caller can repeat to its user ("11
reading(s) reused from earlier identical calls; nothing billed (the estimate
was 19,500)"). `estimated_tokens` is only ever the estimate (N-R15, found when
a caller reported the estimate as spent).

Returns `method_used`, `read` (number, method, LaTeX), `failures` (with
reasons), `billed_tokens`, `seconds` or `calls`, `pages_converted`,
`stopped` and `remaining` when the budget ran out, `snapshot_id`, each
equation's final `status` (`read`, `failed`, `skipped_budget`, …), and
`notes`: `converted_markdown_written`, `notes_written`, `notes_skipped`
(owner-maintained), `markers_not_numbered`, `reingested`.

**Where a reading lands.** In place of the marker, in the converted copy, in
the note's `## Source equations` and wherever else a note carried the marker:

```
$$\sigma^2 = \frac{1}{n} \sum_{i=1}^{n} (x_i - \mu)^2$$ _(eq. 4, read from the image by claude_code, <model>; check against the source)_
$$P(t) = P_0 e^{-\lambda t}$$ _(eq. 5, read by the local formula model)_
```

The catalog records the reading's method, backend, model, confidence and
what was unclear, its share of the call's billed tokens (and the call's
total), seconds, when and by whom. The note is re-ingested, never
re-extracted, so the next answer sees the equation. A reading from an image
is trusted below text, as a figure's is: its `Equation` node has confidence
0.6, and in the governed answer an item resting on it carries
`image_derived: true` with the warning `image_derived_statements`. A
reading by the local formula model is the model docling uses with
`formula_enrichment`, written with its origin and trusted as such (0.8).

**Never paid for twice.** A region with a reading is `already_read` and not
planned again unless `"reread": true`. A reading is kept by its crop's hash,
so converting the source again carries it to the same equation for free (and
never to a changed one); it is written into the copy only after the note has
been extracted, so an extraction never reads a model's reading of an image
as the source's text.

`POST /sources/equations/rollback {"snapshot_id": ...}` restores the copy,
its regions and the notes, removes crops a layout pass added, and re-ingests.
`/sources/reread` snapshots and restores the equation regions too.

### Estimates

Stated as estimates, with their `basis`. Each run is recorded in the
workspace's `logs/equation_reads.jsonl` (method, equations, pages, seconds,
billed and estimated), and once this install has run enough of them the
formula is multiplied by the median ratio of measured to estimated: three
vision calls, one local run, one layout pass.

- **vision**, per call: the prompt (a sealed call's own context, about 2,000
  tokens, plus the instructions and the text around each crop at four
  characters a token) read once per turn; one turn to answer plus one per
  four images opened with a file-read tool; each crop at one token per 750
  pixels (scaled to 1,568 px on its long side); about 120 tokens of answer
  per equation.
- **local**: 20 s to start, 12 s a page for the layout, 150 s an equation
  for the formula model (the conservative end of the measurements below).
- **layout pass**: 20 s plus 12 s a page of the original.

### Settings

`config/conversion.json` (workspace override as usual): `equation_regions`
(crop the equations at conversion), and

```json
"equations": {"max_per_call": 8, "token_budget": 40000, "context_chars": 400}
```

A call's `token_budget` replaces the setting for that call. A malformed value
is replaced by its default and reported (`config_problems`).

## N3. The answer says what it could not see, and how to get it

`/agent/knowledge/answer`, `/agent/knowledge/task_context` and
`/query/resolve_sections` return `unread_regions`, built from the evidence
they return (and `/agent/knowledge/read`, Track Q2, per source, from the text
it returned), two ways:

- **markers inside the evidence**: an unread equation (either form) in a
  returned section, an `Equation` item the evidence names itself (when its own
  words match the question), and figures without an interpretation (a
  `## Figures` item, a figure block of a copy);
- **an `Equation` item of the evidence** (N-R11), when the ranking matched
  its own label's words (lexical score 2.0 or more, more than a pair of
  ubiquitous words can make) or its region matches the question below;
- **the source behind each note of the evidence** (N-R1, N-R8): its unread
  equations are matched against the question by the headings they sit under
  (twice), the words before them (in full) and after them (at half weight: a
  sentence between two equations belongs to both, and introduces the second).
  Words are weighted by the source's own text (N-R10) and the workspace
  (N-R14): a word in half or more of the source's paragraphs, or in 85% or
  more of the workspace's notes (once it has ten), weighs nothing ("the",
  "for", and their like in any language); for context, others by the lower of
  their rarity across the source's paragraphs and the workspace's notes,
  fading to nothing as a word nears half the paragraphs, so two common words
  together never reach a match; for headings, by how few of the source's
  headings use them. No list of words is kept. A region or a heading matches
  at 1.5 or more.

One entry per note, the notes nearest the top of the evidence first, at most
four:

| field | |
|---|---|
| `note`, `source`, `sections` (`section`: the first) | where the evidence is, the original it was built from, the sections that held markers |
| `unread_equations` | how many of the source's equations are unread |
| `equation_ids`, `matches` | those held by the evidence first, then every equation of the best-matching section, then up to three single best matches (at most 20); each match with `number`, `page`, `printed_label`, `heading_path`, `score`, `in_evidence`, `matched_words`, and the words `before` and `after` it |
| `sections_matched` | up to three headings whose words match the question, each with `summary` ("III. SYSTEM MODEL: 9 unread equation(s) (1-9), pp. 4-5"), `equation_ids`, `pages`, `score`, `matched_words` and its own `recover` (by ids, or by `section` past 20) |
| `pages`, `equations_without_ids`, `regions_located` | |
| `figure_numbers`; `uncaptioned_figures` | figures with a caption that nobody has read; images the document prints no caption for are listed apart and not offered unless `vision.interpret_uncaptioned` is on (N-R2) or they sit where the question is (below); decorations are in neither |
| `uncaptioned_offered` | Track AD2: uncaptioned figures offered by their place, at most three a source: each with `number`, `page`, `heading_path`, `score`, `matched_words` and `why`. A figure qualifies when the headings over it and the text just before and after its block match the question with at least the weight an equation needs (`rank_regions`, `MIN_MATCH`); it is then in `figure_numbers`, `estimates.figures` and `recover_figures` like a captioned one |
| `estimates` | `equations`: `would_read`, `regions_located`, `local` and `vision` (availability and cost), `auto_would_use`, `why`, `how_to_get_regions`; `figures` |
| `recover`, `recover_all`, `recover_figures` | `route`, `body` (a dry run of exactly those equations, with the caller's ids: it works as given), `run_body` (`dry_run: false`, `maintenance_write`, `wait: true`), `then`; `recover_all` for every unread equation of the source |

A note none of whose unread equations matches the question gets a compact
entry: `unread_equations`, `equation_ids: []`, `why`, `recover_all`. The
warning `evidence_has_unread_regions` (with a sentence in `warning_details`)
is raised only when an entry has something to read. The answer reads files
and spends nothing: no agent call, no conversion, still `read_only`. It
probes, only for an entry with something to read, whether a backend can read
an image and whether docling is installed (both cached). The caller shows the
dry run's estimate to its user and asks; reading is the caller's decision,
never the answer's.

`GET /review/source` returns `unread_regions` (`equations` with number, page
and line; `figures`) for the lines it serves, so an agent reading a source can
pick regions itself.

## N4. The operator skills

`skills/*/second-brain-research-answering` (and the operations skills) teach
the loop: answer; see `unread_regions`; decide whether the question needs
them; dry-run the `recover` body; ask the user with the estimate (local:
minutes, free; vision: tokens); send `run_body`; answer again; say that an
equation read from an image is a reading, to check against the source. And
when not to: a question the text already answers.

## Books (Track O3)

A book (`POST /sources/process_book`) is converted once, so its unread
equations are one catalog beside the book's converted copy, numbered through
the whole book, and its chapters are separate notes
(`second_brain/equations/books.py`, through the chapter-to-page mapping in
`second_brain/source_pipeline/books.py`; see `docs/document_conversion_v1.md`,
"Books"):

- **Each chapter's own regions.** A region whose numbered marker sits in a
  chapter's text is that chapter's; one the text does not hold goes to the
  chapter whose pages hold its page; one in neither is unassigned, with the
  reason (front or back matter). Each chapter note's `## Source equations`
  lists only its own, so its Equation nodes hang off it. The `process_book`
  result's `equations` says which chapter got which.
- **`POST /sources/equations` takes a book**: `source` the book's original,
  or `note` a chapter note (which alone selects its own chapter), and
  `chapter` (a number or a chapter note's name), with `pages` narrowing it.
  The selection resolves to region numbers, as this contract promised, so
  everything above holds unchanged: local reading converts only the pages
  those regions are on, vision reads only their crops, a reading lands in the
  book's copy and in the chapter note that holds it, a rollback restores both.
  `equation_ids` and `section` narrow a chapter selection further. A chapter
  with no unread equation is a dry run with nothing to read
  (`method.why: "nothing to read: chapter 2 holds no unread equation…"`),
  never the whole book. An unknown chapter is a 404 naming the chapters;
  `chapter` on something that is not a book is a 422. A copy whose regions
  are not located yet is selected by the chapter's pages. `book` in every
  response lists each chapter's equations and the selection.
- **Answers.** `unread_regions` reads a chapter note's regions from the
  book's catalog, scoped to that chapter: its `source` is the book's
  original, its `equation_ids` are the chapter's own (book numbering), and
  its `recover` body (the chapter note and those ids) works as given.
- **Outline.** docling writes a typeset book's chapter opening as two `##`
  headings (`Chapter 3`, then its name) and its sections numbered (`3.1`,
  `3.1.1`); the outline reads the name as the top heading and the numbered
  sections below it, so an equation's path is `EDF Scheduler > 3.1
  Deadlines`. The bare `Chapter 3` heading is not in the path.

## Measured (2026-10-10, CPU, docling 2.137.0)

On the synthetic two-page PDF (`tests/fixtures/make_equations_pdf.py`):

- docling with regions, formula model off: 16.3 s for the worker (both
  pages), five regions found, each cropped (863 px wide, 69-109 px high).
- local reading of page 2 alone, formula model on: 127.8 s wall for the
  worker on that page (the formula estimate said 332 s); both equations read,
  one of them with a symbol misread (`\Box` for the mean): a reading is a
  reading, written with its origin. The same page in a bare docling library
  call took 110 s, the formula model most of it; with the layout alone both
  pages took 17 s.

On the first instance (Track M, 2026-10-09): an 11-page paper, 118 s without
the formula model, about 29.5 minutes with it, 11 equations not read.

By the formula, one crop alone is about 5,800 tokens and eight crops in one
call about 14,900. Live on the first instance (2026-10-10, the coordinator's
check on an 11-page paper): the layout pass took 50 s (estimate 152 s), 11
regions all numbered, crops clean; vision read all 11 in two calls (8 + 3),
16,235 billed against an estimate of 18,314, in 20 s, every reading matching
its crop; the formula model on one page took 179 s (estimate 332 s), 2 of 2
correct; a rollback restored copy, notes, catalog and graph.

## Limitations

- A note carries a marker only where its extraction copied one, and in its
  `## Source equations`; the extraction prompts are unchanged (changing them
  would invalidate cached extractions), so an equation is usually found
  through `## Source equations`, by the words before it.
- Numbering an older copy's page-only markers relies on the layout pass
  finding the same formulas as the conversion did; with a different docling
  version a page whose counts differ is matched by words, and what stays
  unmatched keeps the old form (reported).
- The local formula model reads every formula on the selected pages, and its
  time is per equation, not per page; the estimate's per-equation constant is
  conservative until this install has run it once.
- Books (Track O3, above): a chapter's equations are those its text holds or
  its pages hold; one in front or back matter is unassigned and is read only
  by a page selection or a whole-book request.
- `python scripts/probe_vision_confinement.py --run --skill read_equations`
  is the live confinement check (passed on the first instance, 2026-10-10,
  6,000 tokens). It runs the model the install configures for the backend
  (`--workspace-root` reads a workspace's, `--model` overrides), and a call
  that failed or was refused is INCONCLUSIVE with exit 3, never a PASS
  (N-R5; the figure probe had both problems too, fixed in the same script).
- Matching a question to an equation reads the headings and words around
  it, not its mathematics: a question phrased only in symbols finds nothing,
  and a document that introduces its equations after them is matched at half
  weight. Outline levels are read from marks or numbering; a document whose
  headings carry neither gives each equation its nearest heading only.
- `identity_terms` are names the source defines in brackets in its opening;
  a name it never brackets, or brackets only later, is not found.
- The local formula model and the layout pass read PDFs only. Any other
  source's equations are read from its text or, for one that arrived as an
  image, by vision; the dry run says so (`methods.local.offer: vision`), and
  `pages` on a source without pages is a 422 (Track T4, `docs/images_v1.md`).
