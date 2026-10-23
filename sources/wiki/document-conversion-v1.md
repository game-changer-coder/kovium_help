---
type: "documentation"
title: "Document conversion v1: equations, tables and figures (Track M)"
description: "How a PDF becomes the converted copy every note is built from, which program does it, what each one recovers, how figures are read by vision when the owner asks, and how a source read badly is read again. Plan..."
status: "stable"
reading_status: "read"
product: "second brain"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing second brain"
doc_path: "docs/document_conversion_v1.md"
doc_sha256: "eb311375325b8e0f4b80038124601225addcdd837fc8ddc843f19bddd7beaaa9"
---

# Document conversion v1: equations, tables and figures (Track M)

How a PDF becomes the converted copy every note is built from, which program
does it, what each one recovers, how figures are read by vision when the owner
asks, and how a source read badly is read again. Plan:
`docs/knowledge_layer_improvement_plan_v1.md`, Track M. Images as sources, the
images any document embeds (Markdown, HTML, Word) and equations written in any
source's text use the same catalogs and routes: `docs/images_v1.md` (Track T).

## Why

The basic text extractor (`pdftotext`) reads a page's text layer and nothing
else. On a two-column page the columns interleave line by line; an equation's
symbols and limits come apart and splice into the prose; a table flattens into
runs of numbers (the cause of Track L's misattributed table value); a figure
vanishes, leaving only its caption. This is a property of the tool, not of the
subject: a lab report, a financial filing or a manual loses the same things.

## M1. The converter is configuration, from a stated list

`config/conversion.json` (the install's default; a workspace's own
`sandboxes/<id>/config/conversion.json` wins; a call can override for itself):

```json
{"pdf_converter": "auto", "formula_enrichment": false, "table_structure": true,
 "figures": true, "figure_image_scale": 2.0, "ocr": "auto", "html_trim": true, "timeout_seconds": 3600,
 "docling_python": null}
```

One source can ask for more than its workspace: `POST /sources/process` takes
`pdf_converter` and `formula_enrichment` for that call only, as
`POST /sources/reread` does. That is how the slow formula model is spent on
the one paper whose equations matter, rather than on every PDF.

| `pdf_converter` | What happens to a PDF |
|---|---|
| `"auto"` (the default, Track AG5, D10) | docling whenever it is installed here, else the basic extractor; it spends no tokens, so it turns itself on as soon as what it needs is present. `check_setup` reports the better PDF reader as `works` once it is installed |
| `null` | the basic extractor, on purpose; the result and the note say no converter was selected and that equations, tables and figures are not recovered. In a collection's own file it is the owner's choice and stays one |
| `"docling"` / `"markitdown"`, installed | that converter |
| a supported one, not installed or failing on this file | the basic extractor, with `fallback_reason` and a warning (`docling_unavailable_using_basic_extractor`, `docling_failed_using_basic_extractor`, `oc_markitdown_failed_using_internal_pdf_markdown_fallback`) |
| anything else | refused, naming the list (`ConverterRefused`: 400 on `/sources/process`, 422 on `/sources/reread`, `pdf_converter_not_supported` in the health check) |

**`ocr`** (Track AG5, D10): `"auto"` (the default) reads text out of the pages of
a PDF **only when its pages hold no text** (the test triage uses for
`no_text_layer`), and only when docling is the reader that runs and a
text-recognition program is installed where it runs
(`converters.ocr_engine_status`: rapidocr, easyocr, tesserocr, pytesseract or
`tesseract`); `true` reads every PDF this way, `false` never does. The worker is
never handed the word `auto`: `_convert_pdf` decides it for the file. Triage
(`folder_intake.classify`) then takes such a scan in (`scan: true` on its row,
its length estimated from its pages) instead of refusing it as
`no_text_layer`, and the plan names the need `scans`. It uses no tokens and is
slow. `SECOND_BRAIN_NO_AUTO_READERS` makes both `auto` values mean off in one
process (the test suite's setting, so an installed docling decides nothing).

**Web pages.** An HTML source (a fetched URL or a saved page) is converted
from its main content, not the whole page (`html_trim`, on by default,
`second_brain/conversion/html_trim.py`). Measured on a live instance, one
181 KB page was read in 13 parts for about 177k tokens, most of it the site's
own menus, sidebars and footers. The rules read the page's structure, never
its words: its one `main` (or `role="main"`), else its one `article`, else its
body; inside, `nav`, `form`, scripts and the navigation, banner and search
roles are left out, and an `aside` or `footer` only when at least half its
text is links (a sidebar, a site footer), so a page's footnotes or a note in
an `aside` stay. With the whole body kept, a top-level `header` goes too. A
trim that would keep less than a fifth of the page's visible text and less
than 500 characters is not made: the page is converted whole and the result
says why. Measured on five public pages (an encyclopedia article, two
documentation sites, a reference manual, a blog post), the converted Markdown
was 6% to 90% smaller, with every article's text, title and ending kept. The
result reports `conversion.html_trim`, the note's `## Provenance` says how
much of the page was converted, and the original page is kept as it was
fetched. `"html_trim": false` converts every page whole.

Non-PDF formats (Office, HTML, EPUB) go to markitdown as before: through the
`oc-markitdown` CLI (`conversion.converter_command` and
`converter_allows_any_path` in `sys_config.json` still say where it is and what
it may read), or, when that cannot be started, the `markitdown` Python package
if installed (`python -m markitdown`, in a subprocess).

**Default is the basic extractor, on purpose.** Before Track M a PDF went to
`oc-markitdown` first whenever it was found. An install that relied on that
selects `"pdf_converter": "markitdown"`. Nothing is selected for an install
because a program happens to be present on the machine; the choice is the
workspace's and the result says what it was.

### Capability profiles

Declared per converter (`second_brain/conversion/converters.py::PROFILES`),
reported by the health check and in every result:

| | equations as LaTeX | tables | figures | reading order |
|---|---|---|---|---|
| docling | with `formula_enrichment` (slow) | Markdown tables | image files with caption and page | layout model, column by column |
| markitdown | no | Office formats only; a PDF table flattens | no | the PDF's text-layer order |
| basic | no | no | no (caption text only) | page layout; columns interleave |

### What every result and note records

- `conversion.converter`: `name`, `requested`, `selected_by` (`workspace
  setting` / `none selected`), `implementation`, `version`, `options`,
  `capabilities`, `fallback_reason`, `duration_seconds`, and for docling
  `recovered`.
- `conversion.metrics` (`second_brain/conversion/metrics.py`), counted from the
  converted Markdown: `latex_equations` (display and inline), `formulas_not_decoded`,
  `markdown_tables`, `table_rows`, `figures`, `figures_interpreted`, `chars`.
- The templated note's frontmatter: `conversion_converter`,
  `conversion_converter_version`, `conversion_options`; its `## Provenance`:
  `- Conversion: docling 2.x; options: …; recovered 1 equation(s) as LaTeX, 1
  table(s) (3 rows), 1 figure(s)`, or for the basic extractor why it ran and
  what it cannot recover. The wiki note's normalization provenance carries the
  same three keys.

## M2. docling

docling is an optional extra (`pip install -e ".[docling]"`); it brings PyTorch
and, on first use, downloads its local layout, table and (with the formula
option) formula models. It is detected without importing it
(`importlib.util.find_spec`, version from its metadata), or, when
`docling_python` names another interpreter, by asking that interpreter once
(cached five minutes).

**A subprocess of its own.** `second_brain/conversion/docling_worker.py` is run
with the interpreter that has docling (`python docling_worker.py --input …
--output-dir … --options '<json>'`), not imported into the server: docling's
models hold memory the server should not keep between conversions, a crash or
a hang must not take the server down, and `timeout_seconds` must be able to stop
it (on Windows too: `subprocess.run(timeout=…)`; the default, 3600 s, is
generous because the formula model is minutes a page on a CPU). docling reads a
copy of the original under a short name (`source.pdf`) in a short scratch
folder, so a long original path does not become a longer path inside docling.
A job that will be converted by docling (`/sources/process` on a PDF with
docling effective, and every `/sources/reread`) runs in the brain keeper's long
lane, beside books and folders, so it never holds a slot ordinary jobs need.
Every result reports `pages` and `seconds_per_page`. The worker imports nothing from
second_brain, strips its own folder from the import path, runs with the
billing guard's stripped environment, and writes into a scratch folder:
`document.md`, `fig-<n>.png` and `manifest.json` (version, options, pages,
figures with caption, page, label and size, tables with rows and columns,
formulas found and read as LaTeX). The library call rather than the `docling`
CLI because the library gives each figure's caption and page and each table's
shape as data.

**Billing.** The worker switches off, by name, every option that could call a
remote model (`REMOTE_OFF`: `enable_remote_services`, `do_picture_description`,
`do_picture_classification`, `do_code_enrichment`); a conversion runs local
models only and calls no model API. `tests/system/test_billing_safety.py`
checks the worker statically.

**Figures** are kept beside the converted copy, one folder per source:

```
sources/converted_markdown/<kind>/<stem>.md
sources/converted_markdown/<kind>/figures/<id>/fig-<n>.png
sources/converted_markdown/<kind>/figures/<id>/figures.json
```

`<id>` is twelve hex characters derived from the converted copy's name, never
the name itself, and the files are written by the pipeline from docling's
document object, not by docling's own image export. docling's export names a
figure after the PDF plus a 64-character hash in a nested `_artifacts` folder,
and on a real paper in a deep folder that passed Windows' 260-character path
limit and failed. A figure's path is now a fixed few characters longer than
its converted copy's folder, whatever the source is called.

`figures.json` (the catalog) records each figure's number, page, caption, the
document's own label for it (`"Fig. 3"`, read from the caption), the image file,
its SHA-256 and size, and its interpretation once read. In the converted copy
each figure is one marked block, rewritten in place from the catalog:

```
<!-- figure:1 sha256=… -->
### Figure 1 (p. 2)

![Figure 1](figures/<id>/fig-1.png)

**Caption:** Figure 1: …the document's own words…

**Interpretation (from the image):** … _(read from the image by claude_code, …; weaker evidence than the text)_
<!-- /figure:1 -->
```

The caption docling also prints beside the picture is not repeated.

**An equation found and not read is said, never dropped.** docling leaves
`<!-- formula-not-decoded -->` for an equation it located and did not read
(always, with `formula_enrichment` off): invisible in a rendered page and to an
extraction. The pipeline writes `[equation not recovered (p. N, eq. K); see
the source page]` in its place, with the page and the equation's number in the
copy's equation catalog, counts it (`formulas_not_decoded`,
`equations_not_recovered_pages`), and the note's `- Conversion:` line says how
many and where. Each one is kept as a region (its box and a crop, beside the
figures) that can be read later, alone, by the formula model on its page or by
vision, and the answer says when its evidence holds one: Track N,
`docs/equations_v1.md`. The form without a number, written before Track N,
still parses.

**Tables and the fidelity checks.** A docling table is a well-formed pipe
table, which Track L's layout analysis reads as clean structure: exactly the
"source's own clean table" its row checks compare a note against. The figure
blocks the pipeline writes are skipped by that analysis (their hash, page and
file name are not values a note could take from a table).

## M3. Figures read by vision, opt-in

### The skill

`interpret_figure` (`second_brain/brain_keeper/skill_definitions/`, `vision:
true`) gets one figure: the image (an attached file), its caption, page and
label, the document's title and opening, and the passages that mention it
(found by the document's own word for "figure", taken from its caption, so no
language is assumed) or else the section it sits in. It returns `kind`,
`shows` (what is plotted or drawn, and the visible trend or structure, in the
document's terms; a value only when it can be read from the image), `argument`
(what the document uses it to show, or "unclear"), `not_shown`,
`consistent_with_caption` (when false the difference goes into `unclear`; the
caption is never contradicted), `unclear` and `confidence`. Cacheable: the same
image with the same context is never paid for twice.

### What each backend is given

The only job that hands an agent a file.

- **Claude Code**: `--tools Read` (every other job keeps `--tools ''`), the
  image copied into the call's fresh, empty working folder, nothing else
  changed: `--strict-mcp-config`, `--setting-sources ''`,
  `--no-session-persistence`, no allow list, no extra folder, no permission
  bypass. In print mode nobody can approve a read outside the working folder.
- **Codex**: the image attached to the prompt (`--image=<path>`), no tool;
  `view_image` stays disabled with the rest.
- **OpenClaw** (`infer model run` takes a text prompt only) and any backend
  that cannot take an image: refused before it starts, `vision_unavailable`
  with the reason; the job moves on to the next backend that can. The
  deterministic tier never answers a vision job.

`BrainKeeperService.vision_status()` reports, per backend in try order,
whether it can read an image and why not; it is the health check's
`figure_vision` and every dry run's `vision`.

### During ingestion

`config/ingestion.json` (workspace override as usual), off by default:

```json
{"vision": {"enabled": true, "max_figures_per_source": 6, "token_budget": 30000,
            "opening_chars": 1500, "referencing_chars": 4000}}
```

When on, `/sources/process` (and folder intake, which calls it) reads a
docling-converted PDF's figures after the note has been extracted from the
text alone and before it is ingested, at most `max_figures_per_source`, and
stops before a figure whose estimate would pass `token_budget` given what has
been billed so far. The result's `figures` lists what was read, with
`billed_tokens`. A folder dry run states the vision allowance apart
(`vision.upper_bound_tokens`), outside `estimated_tokens`.

### After ingestion, on request

```json
POST /sources/figures
{"note": "sources/wiki/<note>.md", "agent_id": "…", "workspace_id": "…"}
```

A dry run (the default) lists every figure with its `status` and
`estimated_tokens`, `would_read`, the total, and `vision_available` with its
reason, and calls nothing: a caller agent shows it to its user and asks. Then
`"dry_run": false, "access_mode": "maintenance_write"` reads them (audited as
`source_figures_interpret`). `figures: [n]` picks some; `reinterpret: true`
reads again a figure already read.

**What it reads first.** A figure the document's text refers to, then one with
a caption, then the rest, in document order within each. An image the document
prints no caption for (in practice a logo, a portrait, a decoration) is not
read: it is listed in `uncaptioned_skipped` with status `uncaptioned_skipped`,
unless `vision.interpret_uncaptioned` is true or it is asked for by number.

**What a real run returns.** Every figure's final `status`: `interpreted`
(with its `interpretation` and `billed_tokens`), `reused` (answered from the
answer cache, 0 tokens), `failed` (with `reason`), `skipped_budget`,
`over_figure_limit`, `over_token_budget`, `already_interpreted`, `no_image`,
`uncaptioned_skipped`, `not_requested`. `stopped` is `budget_reached` (or
`vision_unavailable`) when the run ended early, and `remaining` lists the
figures left for a later run. The budget is checked before each call, from
what has been billed so far plus that figure's estimate: a figure is read
whole or not at all.

**Estimates** are this install's own once it has read enough figures: the
median billed tokens of its `interpret_figure` calls in `logs/agent_calls.jsonl`
(made, not replayed, succeeded, usage reported; at least 3), as folder intake
calibrates extraction. `estimate_basis` says which. Until then a formula: the
image at one token per 750 pixels after scaling to 1,568 px on the long side
(capped at 1,600), the prompt (about 2,000 tokens of a sealed call's own,
plus the text at four characters a token) counted twice, because a backend
that opens the image with a file-read tool takes a second turn, and 600 for
the answer. Measured live on 2026-10-09: 8,811-9,450 billed per figure, where
the first, one-turn formula said about 4,500.

### Where a reading lands

- under its figure in the converted copy (above);
- as a figure item in the note's `## Figures` (templated and wiki notes; a
  note with `maintained_by: owner` is never written, and is reported in
  `notes_skipped`):
  `- Figure 1 (p. 2): <caption> — Interpretation (from the image): … [read from
  the image by codex, confidence medium]`; a figure not read is listed by its
  caption;
- in the graph as a Figure node per figure of that document (never merged
  with another document's "Figure 1"), `metadata.derived_from` `caption` or
  `image`, confidence 0.85 for a caption and 0.6 for a reading, below text;
- in the governed answer: an image-derived item carries `image_derived: true`,
  `derived_from: "image"` and `confidence.image_derived`, with the warning
  `image_derived_statements`.

A reading is kept in the catalog by the image's hash, so converting the source
again carries it to the same image for free and never to a changed one.

## M4. Reading a source again

```json
POST /sources/reread
{"note": "sources/wiki/<note>.md", "pdf_converter": "docling",
 "formula_enrichment": true, "vision": true, "agent_id": "…", "workspace_id": "…"}
```

- **Dry run (default)**: the chosen converter on the original, into a scratch
  folder (local, no tokens); `before` (the current copy's converter, metrics,
  figures and Track L fidelity findings), `preview`, `comparison` (`{metric:
  {before, after, change}}`), `estimate` (`reextract_tokens` at this install's
  measured token rate, `vision_tokens` for the preview's figures within the
  budget, `total_tokens`) and `vision_available`.
- **What it costs.** The dry run's `estimate` adds `concept_sync_tokens`
  (when concept notes are synthesised by the agent on ingest) and prices
  vision with the same plan a figure pass makes (`vision_figures`,
  `vision_basis`). A real run returns `billed`: every call the workspace's log
  records during it, as `extraction`, `vision`, `concept_sync`, `other` and
  `by_operation`, so the estimate and the bill can be set side by side.
- **Review items it answered.** After a real run, an open correction
  (`correction:`) whose statement is no longer in the note is closed as
  `superseded`, with whether the new note states what it proposed
  (`covered`) and, if not, the words it is missing, so the owner can file it
  again against the new text; an open `fidelity:` item is closed when the
  checks find nothing in the note now (`findings_gone`). Both are listed in
  `review_items`; anything else stays open.
- **Real run** (`"dry_run": false, "access_mode": "maintenance_write"`,
  backgrounded unless `"wait": true`; audited as `source_reread`): a rollback
  snapshot of the converted copy, its figures and the notes
  (`maintenance/rollback_snapshots/reread/<id>/`), then the normal pipeline
  (`reextract: true`) or only a re-conversion and the figure items
  (`reextract: false`, no extraction tokens). Returns `after`, `comparison`,
  `fidelity: {before, after}`, `billed_tokens`, `snapshot_id`.
- `POST /sources/reread/rollback {"snapshot_id": …}` restores those files,
  removes figures the re-read added, and re-ingests.
- A note its owner maintains is never re-extracted (refused in the dry run,
  409 for a real run); `reextract: false` is allowed.

**One converted copy per source, whatever the original is called now
(M-R1).** A converted copy is named after the original's file name when the
source is first taken in. Converting a source the workspace already has (a
re-read, a reprocess) writes into the copy its notes record
(`converted_markdown_path`, found through the source's identity: its recorded
spelling, a spelling it had before a rename, its bytes), never into a new name
derived from the original's current name. Found live: an original renamed
after intake was re-read into a second copy, and the notes, the figures and
the comparison went on reading the old one. Only when the notes record no copy
inside the workspace is a new one written, and then every note is pointed at
it (`repointed`: `converted_markdown_path` and the `Converted markdown
reference` line; an owner-maintained note is skipped and reported), under the
same snapshot. A copy no note records that sits at the derived name (left by
the defect) is removed and listed in `orphaned_copies_removed`; the rollback
restores it. `after` and `fidelity.after` are measured from the copy the run
wrote (`after.converted_markdown`), `after.converter` names who wrote it and
`after.note_extracted_from` the converter whose text the note's claims still
come from (unchanged by a convert-only re-read). `POST /sources/figures` and
`/sources/reread` accept the note, the original under any of its names, or
the converted copy itself.

## Books (Track O)

`POST /sources/process_book` converts a book once, splits it into chapters
and writes one note per chapter. Track O makes each chapter know where in the
book it is, gives it its own figures, and states the cost before converting.
Plan: `docs/knowledge_layer_improvement_plan_v1.md`, Track O.

**Track AL.** Two things here serve the chapter-finding ladder
(`docs/ingestion_spec_v1.md`, "The chapter-finding ladder"):
- **The PDF's outline is read without converting it**
  (`second_brain/conversion/pdf_outline.py`): pypdfium2 or pypdf where this
  interpreter has them, pypdfium2 in docling's interpreter (`docling_python`),
  else a small reader of the file itself, which follows plain objects,
  compressed object streams and named destinations, needs nothing installed,
  and does not read an encrypted file. Pages are the file's own order, the
  numbering the page markers below use. The dry run reads it too, so the
  split a PDF with bookmarks would get is shown before any conversion.
- **A part's converted copy keeps the part's name** (AL8): a part file the
  platform wrote (under `sources/raw/book_chapters/` or
  `sources/raw/paper_sections/`) is converted to a copy of the same name,
  `<document-key>-<NN>-<title>.md`, not the underscore spelling other copies
  get; a copy an earlier run named that way keeps its name.

### O1. Chapters keep their pages and their book

The book's converted copy carries one marker at the start of every page,
`<!-- page:N -->`, whichever converter made it
(`second_brain/conversion/pages.py`):

- **docling**: the worker, asked for `page_markers` (only the book route asks),
  exports the Markdown one page at a time. docling's own page-break
  placeholder is one fixed string that does not name the page. A paragraph
  docling joins across a page break is one item, exported on its first page;
  the page it runs onto is marked `<!-- page:N continued -->`. Found with the
  real docling on the synthetic book: without it, a chapter ending in such a
  paragraph was reported a page short (`2` instead of `2-3`);
- **the basic extractor**: `pdftotext` without `-nopgbrk`, so a form feed ends
  every page, a blank one included; the form feeds become markers;
- **markitdown** on a PDF: its form feeds, when it writes them;
- **a book that is not a PDF** (Markdown, EPUB, Office): no pages. The result
  says so (`pages: {known: false, reason: "the book arrived as .md, which has
  no pages"}`) and its chapters record `source_pages: unknown`.

The markers stay in the book's converted copy and are taken out of the text
before it is split; each chapter's pages are read from where its text starts
and ends. Page numbers are the PDF's own order (1 is the file's first page),
the numbering docling's figure and equation markers already use, not the
numbers printed on the pages. Every chapter note (templated and wiki) records:

```yaml
book_source: sources/raw/ponds.pdf          # the book's original, as every source is recorded
source_pages: 4-5                           # or `unknown`
book_converted_markdown: sources/converted_markdown/<kind>/ponds.md   # workspace-relative
```

The chapter's own file under `sources/raw/book_chapters/` carries the same
keys, and its `source:` is now the book's recorded spelling (it was the
machine path of the book).

**A chapter's name** (O-R3). A typeset book (a LaTeX thesis, most
textbooks) prints `Chapter 1` on one line and the chapter's name on the next,
and every chapter used to be titled `Chapter 1`…`Chapter 5`. When a chapter's
opening is only a label and an ordinal (`Chapter 4`, `Part III`, `Kapitel 2`,
a bare `4`; read by shape, so any language's chapter word), its name is the
first short line after it that is not a sentence; a chapter that goes
straight into prose keeps its numbered title. The name is the chapter's
`title`, its file name (`thesis-ch01-introduction.md`) and what `/sources/figures`
shows in `book`, read from the chapter file rather than from the note's
title, which the deterministic tier takes from the file name. The index note lists each chapter with `(pp.
4-5)`. Markers inside a chapter keep the book's page numbers, because the
book is converted once: `[equation not recovered (p. 6); see the source
page]`, `### Figure 1 (p. 5)`.

`second_brain/source_pipeline/books.py` reads this back: `find_book(context,
note=… | source=…)` finds a book from its original (any spelling a chapter
records) or from one of its chapter notes, with that chapter; `select(book,
chapter=…, pages=…)` returns the chapters and the page set a selection names
(a chapter's own pages when only a chapter is given). Figures and equations
(O3) both select through it.

### O3. Equations of book chapters

Track N's regions of a book are one catalog beside the book's copy; each
chapter note lists only its own in `## Source equations`, and `POST
/sources/equations` takes a book with `chapter` or `pages`, as
`/sources/figures` does. An answer's `unread_regions` lists a chapter's
unread equations with the book as their source. The contract is in
`docs/equations_v1.md`, "Books (Track O3)".

### O2. Figures of book chapters

The book's figure catalog (`figures/<id>/figures.json` beside the book's
converted copy, numbered through the whole book) is assigned to chapters: a
figure whose block sits in a chapter's text is that chapter's; one the text
does not place goes to the chapter whose pages hold its page; one in
neither (front or back matter) is `unassigned`. Each chapter note's `##
Figures` lists its own figures only, so the Figure nodes ingestion builds
hang off that chapter's note.

During `process_book` a book's figures are read by vision only under its own
switch, in `config/ingestion.json`:

```json
{"vision": {"enabled": true,
            "books": {"enabled": false, "max_figures_per_book": 20,
                      "token_budget_per_book": 150000}}}
```

Off by default, and read only when both `vision.enabled` and
`vision.books.enabled` are on: a book multiplies the per-figure cost by its
length, and one source's allowance spent per chapter would be that allowance
times the number of chapters. The ceilings bound the whole book, all chapters
together, checked before each figure as for a single source. With it off the
chapter notes still list their figures by caption, and the result's
`figures.vision.reason` says which switch is off. The health check's
`figure_vision.books` reports it.

After ingestion, `POST /sources/figures` reads a book's figures on request:

```json
{"source": "sources/raw/ponds.pdf", "chapter": 2}
{"source": "sources/raw/ponds.pdf", "pages": "41-78"}
{"note": "sources/wiki/<a chapter note>.md"}
```

A chapter note alone selects its chapter; `chapter` and `pages` together keep
what is in the chapter and on those pages; neither is the whole book. The
dry run lists only the selection's figures (`outside_selection` counts the
rest) with the same estimates, `book` reports every chapter's figures and the
selection, and the budget defaults to the book's ceilings rather than one
source's. A real run (`maintenance_write`, audited as before) writes each
reading under its figure in the book's converted copy and as a figure item in
the chapter note that holds it, then re-ingests the chapter notes it changed.
Refusals: an unknown chapter is a 404 that names the chapters; a malformed
page selection, a page selection on a book whose pages are not known, and
`chapter` or `pages` on a source that is not a book are 422.

A figure that belongs to no chapter is listed in `unassigned` and, with its
reason, in `unassigned_detail` (O-R6): `before the first chapter (front
matter)`, `after the last chapter (back matter)`, or a page no chapter holds.
It is read only by a whole-book request or a page selection that holds its
page; the book's index note is not a note the graph holds, so it does not
list them.

**A book is named as every result spells it** (O-R2): `sources/raw/thesis.pdf`
is the workspace's file, for `/sources/process_book`, `/sources/process` and
`/sources/figures`, not a path under whatever folder the server started in
(found live: the dry run spelled the source that way and the real run then
returned 404 for it). A declared root's `<root>:<path>` is accepted as before.

**Figure labels with chapter numbering** (O-R4). A caption's label is read as
a word and its number, the number chapter-numbered or not: `Figure 2.1`,
`Fig. 2-1`, `Abbildung 3.2`, `Fig. 3`. It used to stop at the first dot
(`Figure 2.1` became `Figure 2`, and a pass looked for mentions of Figure 2).
The catalog reads the label from the caption when it installs figures, so a
manifest from an older worker is right too; mentions of `Figure 2.1` match
`2.1`, `2-1` and `2–1` in the text, never `2.10`.

### O4. A book states its cost before it is converted

`POST /sources/process_book {"dry_run": true}` answers at once and converts
nothing: `pages` with `pages_basis`, `converter.effective` (what a PDF gets
here), and `estimate`: `seconds`, `minutes`, `seconds_per_page` and its
`basis`.

**Counting pages** (O-R1), the first that answers: `pdfinfo`; pypdfium2 in
the server's interpreter (docling depends on it, so it is there whenever
docling is); pypdf; pypdfium2 in docling's own interpreter when
`docling_python` names one; the pages an earlier conversion of the same bytes
recorded in this workspace (the timing log keeps each source's SHA-256);
the file's page objects. Found live: an 82-page thesis keeps its page objects
in compressed streams, the only counter then was `pdfinfo`, which that install
does not have, and the 50-page ceiling never applied, not even on a second
dry run after docling had recorded 82 pages for it. When the pages still
cannot be counted, docling would read the book and a ceiling is set, the
call is refused (`reason: docling_book_pages_unknown`, warning
`book_pages_unknown_for_docling_ceiling`) unless `force_long_conversion`:
a ceiling that cannot be checked is not passed silently.

**What the time covers** (O-R5). `estimate.seconds` is the converter alone,
and `estimate.covers` says so. Measured live: docling took 210 s of a
1,031 s run on an 82-page book; splitting, reading every chapter and ingesting
took the rest. Every `process_book` run now records its whole duration too
(`kind: book` in the same log), and `estimate.whole_run` gives the whole run
at that measured rate, or None with `whole_run_note` until one has been
measured. A real run returns `duration_seconds`. The converter's rate is this workspace's own: every docling conversion appends
its pages and wall seconds to `logs/conversion_timings.jsonl`
(`second_brain/conversion/timing.py`), read back as total seconds over total
pages, so a long document, where the worker's start-up is spread thin, counts
for more than a one-page note. The formula model is measured apart. Until
docling has run on the workspace the basis says it is a default (11 s a page,
160 with the formula model, from the 2026-10-09 CPU measurement). The dry
run also states the vision allowance a book would spend
(`vision.upper_bound_tokens`, 0 when `vision.books` is off).

`docling_book_page_ceiling` in `config/conversion.json` (150; 0 switches it
off): a book docling would read with more pages is refused, before anything
is backgrounded, with `reason: over_docling_page_ceiling`, the page count
and estimate in `detail`, unless the call passes `force_long_conversion:
true`. That is separate from `force`, which accepts an uncertain chapter
split: accepting a split says nothing about accepting hours of conversion.
The ceiling applies only when docling would run.

**A current converted copy is read, not converted again** (chores after Track
AM, C-R2). A real 160-page thesis already converted was refused at the
ceiling, and with `force_long_conversion` converted again by docling (about
574 s). The book route now reads the copy at the book's converted path, with
its pages marked, when it is current: the workspace's conversion log records
these very bytes converted by the converter the workspace uses now
(`basis: recorded`), or, when the log holds no conversion of these bytes (a
copy made before the log recorded them), the copy is newer than the source
(`basis: newer_than_source`). Then nothing is converted and the ceiling does
not apply; the dry run's plan says which copy, under `converted_copy` (`path`,
`basis`, `converter`, `line`), and the run's `conversion.method` is
`reused_converted_copy`. A conversion of these bytes recorded by another
converter means the workspace changed converter, and the book is converted
again; so is a source whose bytes changed. To convert a book again anyway,
delete its converted copy.

When the estimate passes
docling's `timeout_seconds`, the plan warns `estimate_exceeds_docling_timeout`
(docling would be stopped and the basic extractor used).

Measured (2026-10-10, CPU, docling 2.137.0, models already downloaded): the
seven-page synthetic book (`tests/fixtures/make_book_pdf.py`) through the real
worker with page markers, 19.1 s wall, 2.73 s a page; chapters 2-3, 4-5 and
6-7, the figure on page 5, the equation marker on page 6.

## Broken characters (Track AT)

A PDF that sets a pair of letters (ff, fi, fl, ffi, ffl) as one glyph can lose
it in conversion: "different" arrives as "di?erent" and "offloading" as
"o?loading", where the "?" is a character that stands for nothing. Measured in
one real collection: 120 words in one converted copy, and across the collection
19 converted copies holding 13,664 broken characters and 8 wiki notes holding
234 (mostly the replacement character in maths and tables). A word with a lost
glyph fails every search for it, and it reached notes through figure captions.

**What counts as a broken character.** The replacement character (U+FFFD), the
noncharacters (U+FDD0 to U+FDEF and U+FFFE/U+FFFF of every plane), control codes
other than tab, newline, return and form feed, a `(cid:N)` token, and a
private-use code (a font glyph with no Unicode meaning) **when it sits against a
letter**, as a lost "fi" does in "nal": on its own, a private-use code is a
bullet or an icon, not a lost letter, and is not counted. One each.

**AT1. Repaired at conversion, with no tokens and no dictionary**
(`second_brain/conversion/glyphs.py`, run on whatever copy any converter or the
basic extractor left, and on a book's text; `config/broken_characters.json`,
where `repair` switches it off).
1. The real ligature characters (U+FB00 to U+FB06) are spelled out as the
   letters they stand for. Only those seven characters are normalised
   (compatibility normalisation of each alone): a superscript, a full-width
   digit or a circled number elsewhere in the text is left as it is.
2. A word holding lost glyphs is matched against **the document's own intact
   words**. Each candidate pair (`pairs`: ff, fi, fl, ffi, ffl by default) is
   put where the glyph was; the word is repaired **only when exactly one
   candidate is a word that occurs, intact, in the same document**. No
   candidate (no intact twin) or more than one (ambiguous): the word is left as
   the converter wrote it and counted. A word with more lost glyphs than
   `max_per_word` (2) is left too. The pieces on either side of a lost glyph
   are never taken for words. Candidates are matched without regard to case;
   **the case comes from the broken word's own context**: a word in capitals is
   repaired in capitals, and a word whose remaining letters are lower case is
   repaired in lower case. A glyph that opens the word makes a capital only
   where the word itself starts a sentence, a line, a heading or a list item,
   or where every intact occurrence of it in the document is capitalised
   *inside a sentence of ordinary prose* (a proper name). A capital in a title,
   a heading or a reference entry says nothing either: a word in a run of at
   least three capitalised words (short lower-case words may sit between them,
   by length), or in a line nearly all of whose longer words are capitalised, is
   capitalised because the line is. A capital that opens a sentence in the
   document says nothing about the word elsewhere ("First, we begin" does not
   make "on the First page"). Text a repair never touches (code, maths, links,
   comments, HTML, a `## My notes` section) supplies no word either.
3. There is no dictionary and no language assumption: a German or Spanish
   document repairs from itself, a document in a script no pair belongs to is
   left untouched (its broken characters are counted), and a document that never
   spells a word intact repairs nothing. An install whose documents lose other
   pairs adds them to `pairs`; a pair only ever repairs a word the document
   itself holds.

What a repair never reaches: fenced code, inline code, maths (`$..$`, `$$..$$`),
links and bare addresses, HTML tags and comments (their broken characters are
counted, not repaired); a Markdown file the owner wrote (`direct_copy`,
`text_passthrough`: not converter output). It never loses text: a character
that stood for nothing is replaced by letters, a word is never dropped or
reordered, the raw source is never touched, and repairing a repaired text
changes nothing.

**Figure captions are repaired with the text.** A copy's figures are catalogued
beside it (`figures.json`), a note's `## Figures` section is rendered from that
catalog, and the copy's own `**Caption:**` lines are rewritten from it: this is
how lost pairs reached notes through captions. The repair therefore also covers
each figure's `caption` and `heading_path` in the catalog
(`second_brain/conversion/glyph_catalogs.py`), from the words of the document
they belong to; the metadata counts them as `figure_captions_repaired`.

**What changed is traceable.** The conversion result carries the warnings
`broken_glyphs_repaired` (`:N`, the words repaired plus the ligature characters
spelled out) and `broken_characters_left` (`:N`, what remains) in
`conversion.warnings`, and
`conversion.metadata.broken_glyphs`: the counts (`ligatures_spelled`,
`glyph_words_repaired`, `glyph_words_ambiguous`, `glyph_words_without_twin`,
`glyph_words_too_broken`, `broken_before`, `broken_after`) and the first
`max_listed_changes` (50) repaired words, each `{line, before, after}`
(`changes_truncated` says when the list was cut; the counts are complete). A
repair that could not be saved warns `glyph_repair_could_not_be_written` and
the copy keeps the converter's text.

**AT2. The rest is counted and kept out of search.**
- Lint counts the broken characters of each note: a `broken-characters` warning
  with the count (and, when some, the sections kept out of search), and the
  whole collection's `broken_characters` `{notes, characters, sections_left_out}`
  in `GET /maintenance/lint` and the nightly summary (`docs/note_format_contract_v1.md`).
- A `## section` whose share of broken characters (of its non-space characters)
  reaches `max_share_in_search` (0.03) is left out of the text that is embedded
  for search by meaning (`sections_of`, used by indexing and by the nightly
  catch-up alike, so a note of nothing but such sections is not waiting for
  vectors). Its words are still in the note and in lexical search.

**AT3. Existing notes are repaired at night, for free**
(`second_brain/maintenance/glyph_pass.py`, a step of
`scripts/nightly_maintenance.py`). The same repair runs on the converted
copies, the templated notes and the wiki notes the pipeline built, in that
order. **One vocabulary per document:** a document is its converted copy, the
templated note(s) made from it and the wiki note(s) made from those, joined
through every way a note records its copy: `converted_markdown_path`,
`book_converted_markdown` (a book's chapter notes record the book's copy, not
`converted_markdown_path`), the `- Converted markdown reference:` line of its body,
in any spelling the notes use (workspace-relative, under a declared source root,
a bare file name), its draft by `templated_source_path` or its `- raw_source:`
line, and a section's or a chapter's `part_of` parent. All of them, with the figure catalog
beside the copy, are repaired from the words of all of them together, read
before any file changed, so they agree: a caption the copy never spells intact
is repaired from the note that does, and a note rebuilt from the catalog does not
bring the broken caption back. The wiki notes it changed are re-ingested through
`POST /ingest/file` (the graph and the vectors read the repaired words); one
that could not be re-ingested is remembered and tried again the next night.
Never changed: a note the owner wrote (a native note), a note with
`maintained_by: owner`, a `## My notes` section, and a note's frontmatter apart
from `title` and `description`. Every changed file is listed, with its repaired
words, in `maintenance/glyph_repairs.jsonl` of the workspace. No agent, no
model, no token; and gated by a check that reads files only: a collection with
nothing to repair, or whose leftovers were looked at and have not changed since
(the pass remembers each file's state, with its document's, in
`maintenance/glyph_state.json`), makes no request. A night that changed nothing
and has nothing to re-ingest says nothing; when it speaks, the broken characters
"left" are always those of every converted copy and note it may change, counted
after the repair (before it, on a dry run: "now held"), never only those of the
files it looked at that night.

Limits, stated: a lost glyph that stood for something other than a listed pair
in a word that happens to have a listed candidate in the document is repaired to
that candidate (the document holding the word is the only evidence there is);
words the document never spells intact stay broken (the original PDF, read again
with another converter, may do better: `POST /sources/reread`); a scrambled table
(rows the converter mixed up) is a limit of the converter, flagged where the
layout check can see it, and not repaired here.

## Health check

`GET /ops/self_check` and `scripts/ops_status.py`: `pdf_converters` (supported,
selected, `effective_for_pdf` and why, every converter's availability,
version, options and capability profile, how to select one) and
`figure_vision` (available, backend, each backend's reason, whether ingestion
reads figures, its budget). Warnings: `pdf_converter_not_supported`,
`pdf_converter_unavailable`, `figure_vision_enabled_but_unavailable`.

## Measured (2026-10-09, CPU, docling 2.137.0)

A real 11-page two-column paper (measured by the orchestrator with the docling
CLI, on a short path): **118 s without the formula model**, 51 Markdown table
rows with the results table exact (every row and column matches, and row
arithmetic holds where the basic extractor had flattened it and caused a
misattributed value), 14 figures exported, 11 equations found and not read.
**With the formula model: about 29.5 minutes**, the formula model dominating
(downloaded first). Hence: off by default, per source when wanted, a generous
timeout, the long lane, and `seconds_per_page` in every result.

A synthetic one-page two-column PDF (`tests/fixtures/make_document_fidelity_pdf.py`):

- basic extractor: columns interleaved, table as runs of numbers, equation
  split over four lines, no figure; 0 tables, 0 equations, 0 figures.
- docling, formula option off: 23.6 s wall for the worker on first use
  (model weights downloaded), 11.2 s in docling; 37 s through the pipeline in
  the test suite. 1 Markdown table, 3 rows matching the source, 1 figure
  (362x245 px, caption and page), the equation found and not decoded.
- docling, formula option on: 102 s wall on first use (formula model
  downloaded), 87 s in docling; 150 s through the pipeline under a loaded
  machine. The equation read as `L = \sum_{i=1}^{n} \frac{m_i}{M}`.

## Limitations

- Books (Track O): a page is the PDF's own order, not the number printed on it. A chapter's
  pages are read from where its text falls between page markers: when docling
  joins a paragraph across a page break the paragraph is placed on its first
  page, so inside a chapter an item's page is the page it starts on. A
  folder run (`process_folder`) does not state a book's conversion time in
  its dry run; the ceiling still refuses such a book when the run reaches it.
  `/sources/reread` does not take a book.
- The vision allowance is not part of a folder run's `token_budget`, which
  counts extraction only; each source's vision spend is bounded by
  `vision.token_budget`.
- Claude Code's confinement of `Read` to the working folder rests on its
  print-mode permission default; the test suite checks the command against
  stand-in CLIs, never a live one. `scripts/probe_vision_confinement.py --run`
  is the live check, on purpose and at the cost of one call: a figure job whose
  caption and referencing text tell the agent to read a secret file outside
  its folder must return nothing of it (it prints any permission denials the
  CLI reports). Until it has been run on an install, treat the confinement as
  stated, not proven.
- Of a PDF, only docling exports figures; a markitdown or basic copy of a PDF
  has none to read. Every other source's images (an image as a source, the
  images a Markdown, HTML or Word file embeds) are catalogued in the same
  format since Track T (`docs/images_v1.md`); `vision.interpret_uncaptioned`
  is decided per template there.
- Equations by vision and the formula model on selected pages only are
  built (Track N, `docs/equations_v1.md`): each unread equation is a region
  with a crop, read on request by `POST /sources/equations` with a dry run
  first, and listed in answers as `unread_regions`.
