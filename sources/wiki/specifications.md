---
type: "documentation"
title: "Kovium — the specification and operator's guide"
description: "This is the first document to read if you are going to run a Kovium instance, whatever it holds. It says what the platform does, how material moves through it, what the brain keeper is and why it exists, what each feature does and costs..."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "specification"
audience: "people and agents running Kovium"
doc_path: "specifications.md"
doc_sha256: "7c19711947be9d9ed48e69661d44ab0ca56fc34d2262dfd68604c6fc1609ca05"
---

# Kovium — the specification and operator's guide

This is the first document to read if you are going to *run* a Kovium
instance, whatever it holds. It says what the platform does, how material moves
through it, what the brain keeper is and why it exists, what each feature does
and costs, what you can configure and what is fixed, and what to do day to
day.

It describes the platform, not any one installation of it. One instance holds
research papers; another holds meeting minutes, a recipe box, customer
records, a lab notebook or a journal in a language nobody here reads. Every
rule below is stated so that it holds for all of them, and the examples are
drawn from unrelated fields on purpose.

It is deliberately not a developer document. If you are changing the platform
rather than operating it, read `project_guidelines.md` and `CLAUDE.md`. The
exact contracts behind each section are listed in `docs/README.md`; when a
contract and this guide disagree, the contract is the more precise statement.

Throughout, **Configurable** marks a behaviour an instance sets in its own
`config/` (most of it from the conversation, section 15) and **Fixed** marks
one the platform decides for every instance.

---

## 1. What this is

Kovium turns your written material into a knowledge base you can
actually ask questions of.

You give it sources: Markdown notes, PDFs, Word files, web pages, books,
images, transcripts, meeting minutes. It reads each one, rewrites it into a
consistent note shape, pulls out the ideas, claims, evidence and open
questions it contains, and links those to the same things found in your other
sources. You then ask a question in plain language and get an answer that
cites the exact note and the exact lines it came from.

Three things are true of it throughout, and they are the reason to trust it:

**Your Markdown is the real record.** Everything else — the graph, the search
index, the maintenance state — is derived from your notes and can be thrown
away and rebuilt. The system never becomes the only place your knowledge
lives.

**Every answer traces back.** Nothing is asserted without a pointer to where it
came from. If an answer looks wrong you can follow it back to the source line
in seconds.

**It runs on a subscription you already pay for.** There is no API key
anywhere in this project, and it is built so that there cannot be one. Model
work is handed to an agent command-line tool you already have installed, and
everything that spends that subscription says what it will cost before it
runs. See sections 11 and 14.

## 2. What it is not

It is not a cloud notes app, and it is not a chatbot with your files attached.
It does not sync, it does not host anything, and it does not answer from
memory. It runs on your machine, reads files you point it at, and every claim
it makes is anchored to a line in a file you can open.

It also does not silently rewrite your notes. Corrections go through a review
queue you can inspect. The few things it will change on its own are listed in
section 20, and they are all additive. Notes you write yourself, notes you
mark as yours (`maintained_by: owner`), and the `## My notes` section every
note carries for your own words, are never rewritten by any pass.

## 3. Five minutes from nothing

The first run is a conversation, not a list of commands. After one install
step, each step below is a sentence the person says to an AI program they
already use, and their agent makes the calls. The contracts are
`docs/install_and_caller_agent_v1.md` (the install step),
`docs/welcome_v1.md` (the welcome, the example, the tour and help),
`docs/collections_v1.md` (making and opening a collection) and
`docs/adding_documents_v1.md` (adding).

1. **One install step**, the same on Linux, macOS and Windows:
   `python scripts/setup.py`. It checks this Python, makes a private
   environment in the checkout and installs Kovium with its MCP extra,
   looks for the AI programs it knows (Claude Code, Codex, Cursor, OpenClaw),
   checks their sign-in where the program can say it (Claude Code's with
   `claude auth status`, a key that charges per use refused), says in plain words
   which optional helpers are missing (the better PDF reader, the reader for
   Word files and web pages, search by meaning), and connects every program
   it found in **welcome mode**: a connection with no collection yet, under
   the program's own agent id. Claude Code gets the plugin this checkout
   ships, a small folder of its own (`skills/claude-code/`: its connection,
   the four operator skills and a `/kovium:start` command; Track AY1:
   Claude Code copies a plugin's folder at install), and never a second, plain
   connection beside it; the others get a connection in their own format. A
   second run changes nothing. No service is left running and no token is
   spent.
2. **Welcome.** The person says "help me start with Kovium". A
   connection in welcome mode offers only the welcome's tools: `welcome` (what
   to do next), `help` (a question about the software itself), `try_example`
   and `tour` (the example collection), `get_manual` (the full manual), and
   `create_collection`, `open_collection` and `check_setup`.
3. **The example and its tour.** The example (`sandboxes/hello_brain/`) is a
   few short pages about Kovium itself, with their notes already built
   (of three kinds), one original read where it lies, and one page waiting in
   an inbox. Opening it makes a working copy, reads the graph, concept pages
   and maps derived from its notes at release (`derived/`), or builds them by
   code in a few seconds when its notes or the code changed since, and
   answers a suggested question with
   its sources: no tokens. The tour (Track BA) is eight steps, each with
   something to do and a question back to the person, every file given with
   its full place on this computer at run time: how to look (Obsidian, a text
   editor, or described in the chat), one document at every stop of its way,
   where files live with one note kept as a copy beside one pointed to, a
   note, a map, links, and a ready concept page beside a waiting one with a
   practice night that fills it from a prepared answer (free, said to be a
   practice, with what a real night costs). Its waiting
   page is added either free (read by code, as a note written by hand) or by
   the AI, which is a plan with its cost first and runs only with that plan's
   approval. The last step offers the full manual, once its address is set
   (section 12.5). Every collection, the example's working copy included,
   gets Obsidian view settings (Track BE, `docs/obsidian_view_v1.md`): the
   housekeeping hidden, the graph kept to the notes with arrows and without
   links to missing notes, and a colour per kind of note; free, merged into
   Obsidian's own files, a change made in Obsidian kept.
4. **A collection of one's own.** "Make a collection for ...":
   `create_collection` with a name and a sentence of what it will hold. It
   first asks where the collection goes: the folder the person works in now,
   a folder they name, or the default place (Track AZ1). The collection is
   registered, its maker is named its manager, the connection is
   bound to it in the same session and at every later start, the free nightly
   tidy is switched on for this computer (section 20), and asking before
   anything is spent is switched on (section 14). An existing collection, from
   another computer too, is opened by its folder (`open_collection`), and what
   did not travel with it is said plainly.
5. **Adding, with one approval.** "Add the files in this folder":
   `plan_adding` makes a free plan (what will be added, what is skipped and
   why, what the files need, read from their structure, and the estimated
   cost). The person is asked only what cannot wait (today: the better PDF
   reader, when their PDFs need it) and the one budget question. On yes,
   `start_adding` runs with the plan's approval, `adding_status` follows it,
   and the report is at most six plain lines (section 4.1).
6. **Asking.** A question in plain words; every answer cites the note and the
   lines it rests on (section 12). The collection can also get a caller agent
   of its own in the person's project, answering in the style they choose
   (`make_agent`, section 10).

Measured by the scripted newcomer test (a stand-in reader and no person, on
one machine): no command typed after the install step, one question before
the first example answer, at most five before the first answer from the
person's own document, and no token spent before an approval. On a simulated
clean machine, about 1.6 minutes from the install step to the first example
answer with its source.

**Over HTTP**, for scripts and for people who prefer it, the same routes run
in a service. From the repository root:

```bash
./scripts/run_kovium.sh        # Linux and macOS
python scripts/run_kovium.py   # any platform, Windows included
```

That creates a virtual environment if there isn't one, installs the package,
and starts the service on `http://127.0.0.1:8000`. Two things to open:

- `http://127.0.0.1:8000/` — the graph UI
- `http://127.0.0.1:8000/docs` — the live, clickable API reference

The welcome works the same way, with no registration first:

```bash
# Open the example: registered on first use, its graph, concept pages and maps built by code.
curl -X POST http://127.0.0.1:8000/welcome/example   -H 'Content-Type: application/json' -d '{}'

# Ask it something: the answer lists where each part comes from.
curl -X POST http://127.0.0.1:8000/welcome/example   -H 'Content-Type: application/json' -d '{"question":"What does a note hold?"}'

# The tour, step by step; a question about the software itself.
curl -X POST http://127.0.0.1:8000/welcome/tour -H 'Content-Type: application/json' -d '{}'
curl 'http://127.0.0.1:8000/help?question=What%20happens%20at%20night'

# A collection of your own, then a free plan for a folder of yours.
curl -X POST http://127.0.0.1:8000/collections -H 'Content-Type: application/json' \
  -d '{"agent_id":"<you>","name":"<a name>","holds":"<a sentence of what it holds>"}'
curl -X POST http://127.0.0.1:8000/sources/add -H 'Content-Type: application/json' \
  -d '{"agent_id":"<you>","workspace_id":"<its id>","paths":["<a folder>"]}'
```

Only the example's notes are committed; its graph, maps and concept pages are
derived, and opening it builds them. Nothing the welcome does spends a token,
except letting the AI read the tour's waiting page, which runs only with its
plan's approval.

`./scripts/seed_sample.sh` registers a fresh workspace of sample notes
(`sample`, or `$WORKSPACE_ID`) for testing the service. Section 26 is the
checklist for an instance of your own.

## 4. How information flows

Material moves through a fixed sequence of stages. Each stage is derived from
the one before it and can be rebuilt from it. They are never merged, because
being able to go back a step is what makes a bad import recoverable.

```
  1. raw source            the original file, exactly as you provided it
        │
  2. converted Markdown    documents and images turned into text, with figure
        │                  and equation catalogs beside it
  3. templated Markdown    poured into the shared template for its kind
        │
  4. wiki note             the finished note, in the standard note shape
        │
  5. graph                 ideas, claims, evidence, questions and their links
        │
  6. maintained layer      staleness, duplicates, drift, review history
        │
  7. answers               retrieval and the governed question-answering API
```

Stages 1 to 4 live as files on disk under your workspace, one directory per
stage, so you can look at any intermediate result. Stage 5 onwards lives in a
small local database that is rebuildable from stage 4.

**There are four ways material enters, and they are not interchangeable:**

| what | how | becomes |
|---|---|---|
| a source from outside: a document, a PDF, a web page, an image, a video, a book, a folder or playlist of them | `POST /sources/add` (any mix, in one request; section 4.1), or the lower-level `/sources/process`, `/sources/process_book`, `/sources/process_folder`, `/sources/process_videos` | a note of the collection (stages 1–5) |
| Markdown you are happy to treat as the source of truth | `POST /normalize/file`, then `POST /ingest/file` | a note of the collection |
| a note written directly into the wiki by hand | ingested as it is (`POST /ingest/file`, or a graph rebuild) | a note of the collection, read under its own headings (section 5) |
| the owner's own work about the collection | `POST /owner-work` registers it where it lies | a tier of its own, never evidence (section 6) |

Ideas (section 6) are a fifth thing: notes about what might be worth doing,
captured by `POST /ideas` or written into `sources/wiki/ideas/`.

Do not chain the stages by hand for ordinary imports. The combined routes exist
so that the provenance stays attached at every hop.

**The kinds of source.** Fourteen intake templates ship: academic paper, white
paper, book chapter, blog post, web page, documentation (a manual or help
page kept as a file), meeting note, decision record, chat transcript, AI
conversation, voice note, video transcript, image, and reference list (a
glossary, an abbreviation list, a lookup list, a code table). Each decides
what sections its notes get. An image and a reference list are built from the
source's own words with no agent: no tokens. `GET /format-templates` lists
them with their other names (`GET /format-templates/resolve?kind=` shows what
a word maps to). **Fixed:** the intake templates are the platform's; an
instance cannot add a fifteenth without a code change. **Configurable:** the
kinds of note an instance keeps (section 5), which is how a recipe, an account
record or a lab entry gets its own sections, roles and identifying fields.

### 4.1 Adding documents in one request

**The rule** (Track AR). The caller agent and the platform settle the work
between them. Whatever the platform recommends, the caller carries out;
whatever failed is tried again before anyone is told. A person is asked two
things only: the collection's standing token budget, and deleting or
overwriting their own originals (no route does that today, so in practice only
the budget). Contract: `docs/adding_documents_v1.md`.

**The request.** `POST /sources/add` takes files, folders (at any depth) and
links together (`paths`, `urls`). Over MCP its doors are `plan_adding`,
`start_adding` and `adding_status`, for an agent the collection lets add
documents (section 10). Text from the conversation is kept as a note by
`POST /sources/note` (`add_note`, Track AY4): a plan first, then on the
person's `confirm` the text is written word for word as a note written by
hand in the collection's notes and taken in by code, with no tokens.

1. **The plan** is the default (`dry_run`), free, with no agent call. It says
   in a sentence or two what will be added, what it costs and what is skipped
   and why; it carries an `approval` code, the estimate as a range (section
   4.6), and `asks`: the questions still open, one at a time, recommended
   answer first. A question comes only from what cannot wait: what the files
   need that cannot be added later without reading them again (today: the
   better PDF reader, for PDFs laid out in columns, tables or formulas), and
   the budget question when the addition would pass the standing budget
   (section 14). What the files need is read from their structure (columns,
   tables, display formulas, scans, pictures, file types), never from their
   subject; the other needs are only said.
2. **The run** (`dry_run: false`) needs the plan's `approval`, and spends
   within the standing budget or the one-time budget the person agreed to.
   One approval is one run: starting it twice returns the first run. It is a
   background job; `GET /sources/add/status` says how it is going.
3. **What the run does by itself**: triage, as for a folder (section 4.6);
   each file's kind (from the request's `kind`, the install's `default_kind`,
   else the kind the collection holds most of; with none of these the caller,
   never the person, chooses one from the plan's `kinds`); books split by the
   ladder of section 4.5; other versions of a known work taken in and linked
   (`version_of`; `versions: separate` or `skip` on the request instead);
   files processed oldest first; every failure tried once more before it is
   reported (a passing failure after a pause, a PDF with the next reader, a
   partial note made again when an agent is on); passage vectors that failed
   indexed again; one concept pass over every new note.
4. **The report** is at most six plain lines, which the caller relays as they
   are: what was added, what was skipped or failed and why, what it cost, and
   anything left for later (concepts the pass could not finish are listed
   with what finishing them would cost, and are finished by the next pass or
   the nightly run).

The lower-level routes below are what it is built on; they stay for a caller
that needs one step at a time, under the same approval (section 14).

### 4.2 Converting documents

**Configurable**, per workspace, in `config/conversion.json`, and for one
source per call:

| `pdf_converter` | what a PDF gets |
|---|---|
| `"auto"` (the default) | docling whenever it is installed here, else the basic extractor; it spends no tokens, so it turns itself on (Track AG5) |
| `null` | the basic extractor (`pdftotext`) on purpose: two columns interleave, equations come apart, tables flatten into runs of numbers, figures are lost; the result and the note say so |
| `"docling"` | reading order by a layout model, tables as Markdown tables, every figure exported with its caption and page, every equation kept as a region; with `formula_enrichment`, equations read into LaTeX while converting (minutes a page on a CPU) |
| `"markitdown"` | the PDF's text-layer order; no figures, no equations |

docling and markitdown are optional extras (`pip install -e ".[docling]"`,
`pip install -e ".[markitdown]"`). docling runs in a subprocess of its own,
with its remote services switched off, so a conversion never calls a model
API. A selected converter that is not installed, or fails on a file, falls back
to the basic extractor and says why (`fallback_reason`); a name outside the
list is refused. A collection's own `null` or `"docling"` is the owner's
choice and stays one; "Better PDF reader" in `change_setting` sets it. Every
result's `conversion.converter` and every note's `conversion_converter` say
what ran. `POST /sources/process` takes `pdf_converter` and
`formula_enrichment` for that source only.

**Scanned pages** (`ocr`, `"auto"` by default): a PDF whose pages hold no text
is read by text recognition when docling and a text-recognition program are
installed, and taken in; without them it is refused at triage. `true` reads
every PDF this way, `false` none.

Word files, HTML pages and EPUB go to markitdown (the `oc-markitdown` CLI, or
the `markitdown` Python package); without one they are refused. Markdown and
plain text are copied as they are. `GET /ops/self_check` reports
`pdf_converters` (what a PDF gets here and why), `document_converter` and
`pdf_reader`.

**A web page is converted from its main content** (`html_trim`, on). The
page's one `main`, else its one `article`, else its body, without its `nav`,
forms, scripts, the navigation, banner and search roles, and any `aside` or
`footer` that is mostly links; an `aside` of text (footnotes, a note) stays.
The page's structure decides, never its words, so it holds for any site and
language. A trim that would keep too little converts the page whole. The
original page is kept as fetched; the result's `conversion.html_trim` and the
note's `## Provenance` say how much of the page was converted.

**A source with no readable text is refused** before any agent call: 422
`{reason: no_readable_text, message}` (a page drawn by script in the browser,
a scan without a text layer). Its original and converted copy are kept; the
message says what to do (save the page's text or print it to PDF; OCR a
scan).

**Letters a converter lost are put back** (Track AT). A PDF that sets the
letter pairs ff, fi or fl as one glyph can lose them ("di?erent"). Each such
word is matched against the document's own intact words, and repaired only
when exactly one candidate is a word the same document spells intact; no
dictionary and no language is assumed. What cannot be repaired is counted
(lint's `broken_characters`), and a section too broken is kept out of search
by meaning. Notes taken in before are repaired by the nightly run, for free.
**Configurable:** `config/broken_characters.json`.

### 4.3 Figures and images, in every kind of source

Every picture the platform keeps is catalogued the same way, beside the
converted copy (`figures/<id>/figures.json` and the image files), whatever it
came from:

- **a PDF converted by docling**: each figure with its caption, page and label;
- **an image taken in as a source** (PNG, JPEG, GIF, WebP, judged by its bytes):
  the `image` template, one figure, a caption from the request's `caption` or
  the file's own description, title or comment (GPS is never read), and the
  file's facts. Built without an agent: an agent asked to extract claims from a
  caption would be paid to invent them. Its `## What it shows` stays empty
  until vision reads the picture;
- **pictures a Markdown note, a web page or a Word file shows**: each kept with
  the caption the document gives it (a `<figcaption>`, a caption paragraph, an
  italic line, a title, alt text). Remote images are fetched only for a page
  taken in by URL by default (`images.fetch_remote`: `fetched_pages`, `always`,
  `never`), because fetching an image from a local file can tell its server the
  file was opened.

A format no vision backend reads (BMP, TIFF, HEIC, AVIF, SVG, ICO), an image
over 25 MB, and a binary that is neither a document nor text are refused with
415 **before anything is copied**. Image metadata is scanned for hidden
instructions like any fetched text.

**Reading a picture is vision, and vision is opt-in.** A figure is listed by
its caption in the note's `## Figures` and becomes a Figure node; what the
picture *shows* is read by the brain keeper only when asked:

- during intake, only when `vision.enabled` is on in `config/ingestion.json`
  (off by default), within `max_figures_per_source` and `token_budget`;
- after intake, on request: `POST /sources/figures` (a dry run first, listing
  every figure, its `estimated_tokens` and whether vision is available here; a
  real run needs `maintenance_write`).

A reading is written under its figure and in the note, marked "read from the
image" and trusted below text (confidence 0.6; an answer item resting on it
carries `image_derived: true`). It is kept by the image's hash, so the same
picture is never paid for twice. Images a document gives no caption for
are skipped by a figure pass unless the kind of source reads them (an image
source, a note that is mostly pictures) or `vision.interpret_uncaptioned`
says so, but they are kept: each figure records the headings it sits under
(`heading_path`). **Decorations** are told by the document's shape, never by
a file name or alt text: an image too small to carry content, the same image
on more than one page (the same bytes, or the same size at the same place),
the same bytes used three times in a document without pages, or an image in
the page's header or footer band (`decoration: true`, with
`decoration_reason`). A decoration is not listed in the note's `## Figures`
nor offered in an answer, and is read only when asked for by number. **Fixed:** vision needs a backend that can be given an image: Claude
Code (with its file-read tool alone, confined to a fresh folder) or Codex
(the image attached). OpenClaw takes text only and reports
`vision_unavailable`.

### 4.4 Equations

An equation reaches a note one of three ways, and each says which:

- **written in the source** (LaTeX in Markdown or a web page, MathML, an
  image whose alt text is LaTeX): read from the text, free, listed as
  `$$…$$ _(eq. 3, read from the text)_`, confidence 0.9;
- **found by docling and not read** (with `formula_enrichment` off, every
  equation): kept as a numbered region with its page, box and a crop image, and
  written `[equation not recovered (p. 4, eq. 7); see the source page]` in the
  copy, never dropped;
- **arriving as a picture** in a document that is not a PDF: kept as a region
  with its crop, the same marker without a page.

Each note built from the copy gets `## Source equations`, listing every
equation with the headings it sits under and the words before and after it,
so a question finds an equation by its context. An unread equation is read
**only on request**, and only the ones a question needs: `POST
/sources/equations` (by `equation_ids`, `pages` or `section`; a dry run first)
with one of two methods:

| method | cost | reads |
|---|---|---|
| `local` | free; docling's formula model on the selected PDF pages only, minutes a page on a CPU | PDFs only |
| `vision` | tokens; several crops a call through the `read_equations` skill | any source with crops |

`auto` chooses vision when a backend can read images and the selection fits the
budget, else local. A reading is written in place of the marker with its
origin, re-ingested (never re-extracted), kept by the crop's hash, and undone
by `POST /sources/equations/rollback`. **Configurable:** `equation_regions` and
the `equations` block (`max_per_call`, `token_budget`, `context_chars`) in
`config/conversion.json`.

### 4.5 Long documents and books

**Length never costs you content.** Nothing is trimmed for being too long. A
document bigger than one reading pass is read in consecutive pieces and the
results merged, and each piece is handed a short map of the whole document —
its title, its headings and its opening — so a section is never read as though
it were the entire work. The finished note records how much of the source was
covered (`source_coverage`), and anything short of all of it names the exact
ranges that were missed.

One note keeps at most sixty items per section, because past that a note stops
being something anyone reads. So a document long enough to fill that is
**split into several notes** rather than squeezed into one:

- **A very long document** processed as a paper (above `paper_split_chars`,
  300,000 characters by default) becomes an **overview note** plus **one note
  per section**, linked by `PART_OF` and `FOLLOWS`. The split is decided before
  any model call, from the document's own headings, and **refused rather than
  guessed**: too few sections, one section holding most of it, or pieces that
  do not add back up to the original, and you get the single note. A trailing
  reference list is recognised by its shape and set aside.
- **A book** goes through `POST /sources/process_book` and becomes an index
  note plus one note per chapter. Each chapter note records its pages
  (`source_pages: 41-78`; `unknown` for a book that arrived without pages, such
  as EPUB or Markdown), its `book_source`, and only its own figures and
  equations. A chapter opened by a bare `Chapter 4` line is named by the line
  after it. The parts are found by a ladder, each step only when the one
  before found no confident split: the PDF's own bookmarks, the contents page
  (its heading in any language the install declares, or its shape), the
  chapter headings in any script, and last the brain keeper agent from a
  compact view of the book, within the standing token budget. With no budget
  or no agent, only the free steps run, and a book none of them splits is
  read whole, in parts; the result's `split.line` says so plainly. The owner
  can say how: `chapters: [{title, pages}]` skips the ladder, and a dry run
  shows the split first. Each part is named `<document-key>-<NN>-<title>` on
  every layer, so a book's parts sit together and in reading order;
  `POST /sources/book_parts/attach` joins chapters split by hand to their book,
  and `POST /rename/parts` brings older names to the rule. A split still unsure
  is refused with its reasons; `force: true` accepts a split you inspected.
  The words are the install's (`config/books.json`).

**A book states its cost before it is converted.** `"dry_run": true` returns
its page count, the converter it would get and the estimated conversion time
(this workspace's measured seconds a page once docling has run here, a stated
default before), converting nothing. **Configurable:**
`docling_book_page_ceiling` in `config/conversion.json` (150 by default; 0
switches it off): a book docling would read with more pages is refused at once
with its estimate, unless the call passes `force_long_conversion: true`. A
book whose converted copy is current (the same bytes, converted by the
converter the workspace uses now) is read from that copy: nothing is converted
again and the ceiling does not apply. The dry run also states the book's tokens
chapter by chapter, and the parts it would find for free. A
book's figures are read by vision during processing only when both
`vision.enabled` and `vision.books.enabled` are on, within
`max_figures_per_book` and `token_budget_per_book`; after it,
`/sources/figures` and `/sources/equations` take a book with `chapter` or
`pages`.

### 4.6 A folder, versions of one work, and estimates

`POST /sources/process_folder` takes a whole folder (or a `root:path`) one
step at a time; `POST /sources/add` plans its folders the same way. It is a
dry run by default: every file is hashed and read for its shape, with no agent
call, and the plan says what each will do — process it, take it in with the
document that embeds it (`embedded`), split it as a book, skip it as a
duplicate or an earlier intake, take it in as another version of a known work,
or refuse it (slides, a file outside every declared root, an unsupported
format; a scan without a text layer unless the collection reads scanned
pages, section 4.2). Files are processed oldest first, by the strongest date
evidence each carries, because each is read against the concepts the ones
before it established. Contract: `docs/folder_intake_v1.md`.

**Each file gets its own kind** (`kind_per_file`, on by default): from its
frontmatter, its file type (an image is `image`), a cheap look at its
headings and fields against each kind's declared sections, or its shape (a
file at least 60% of whose lines are entries, a term and its definition, five
or more, is a `reference-list`); the request's `kind` is the default. An
identifier alone never overrides the asked kind: many kinds carry one. Every
row says `kind`, `kind_basis` and `kind_evidence`. A kind built without an
agent (an image, a reference list) is estimated at 0 tokens.

**Versions of one work** (Track AJ). A draft and its final form, a preprint
and its published version, two editions: a file is another version of a work
the collection holds, or of another file of the same batch, only on strong
evidence it states about itself — an identifier it carries as its own (never
one it cites), or the same or a nearly identical title with the same people
or the same opening passage, or two shared people and a strongly similar
opening — and only when no field its note type declares as telling works
apart (`work_fields`: a meeting's date, say) differs. Anything less is a near
miss, taken in as its own work and listed with why (`similar_to`). A version
is taken in and linked with `version_of` on the new note only; the existing
note is never edited. `POST /sources/add` links by default;
`process_folder` holds versions back unless `include_near_duplicates`. The
evidence each note type declares is in `docs/folder_intake_v1.md`.

**Estimates that learn from this install** (Track AM). Every document carries
`estimated_tokens` and `estimated_range` (low, expected, high), read from this
install's own runs: of the document's note type and length band, else its
type, else all runs, else a stated default, with the basis said in words. The
plan sums them, the expected figure is the planning figure, and a run stops
cleanly **before** a document whose estimate would pass its budget. A book's
dry run states its tokens chapter by chapter and in total. The vision
allowance is stated apart. **Configurable:** `config/intake.json` (triage
thresholds, the identifier patterns a collection's files carry,
`kind_by_suffix`, the version and estimate settings).

### 4.7 Videos

A video link (YouTube watch, short-link, Shorts, embed or live) given to
`/sources/process` is the video, not its page: its public metadata (title,
channel, publication date, length, description) and **its own caption track**
(a preferred language, else its spoken language; captions written for it
before automatic ones). Never the video file, a signed-in session or a model,
so reading it costs no tokens. The raw source is the transcript in passages,
each starting with the time it was said, under the video's chapters; the note
records the channel, date, duration, where the transcript came from and how
reliable it is, and links each chapter to its moment. A video with no usable
transcript is refused (`no_transcript`) unless `description_only` is asked
for, and then the note says so; a video that is not public is refused with the
site's reason.

`POST /sources/process_videos` takes a list of video and playlist links like a
folder: a dry run by default, one row per video with its transcript's length
and source, its estimated cost and its action (`process`, `duplicate` by id or
by identical transcript, `already_ingested`, `no_transcript`, ...), then a
real run within a token budget with one concept pass after the last.
**Configurable:** `config/video.json` (languages, manual first, description
only, pacing, limits, and the site's endpoint settings, which change).
**Fixed:** every request goes through the one module that reaches the web.

### 4.8 An unchanged source is not paid for twice

The brain keeper keeps its answers (`answer_cache`, 30 days by default), keyed
on the skill, its definition, the input, the model tier and, when one is
carried, the instance's tail. Advice that changes with every ingest — the
collection's current concept names shown to extraction as `known_concepts` —
is left out of the key. So processing an unchanged source again reuses its
extraction: the result says `reused_extraction: true`, why, and the tokens the
first call billed (`extraction_reuse.billed_tokens_saved`), and the reused
answer's concept names are brought to the concept layer's current wording
(preferred names, merges, screened aliases) before anything is written.

`"refresh": true` on `/sources/process` reads the source again (extraction,
shaping, normalization) at full cost and stores the new answers;
`/sources/reread` with `reextract: true` does the same. What still re-extracts:
a changed text, template prompt, skill definition, tier, tail, or facet
vocabulary. Changing a prompt touches no note already written; the cost is paid
once, when a source is next processed (`docs/prompt_system_v1.md`).

### 4.9 When no agent reads a source

Every judgment task ends in a deterministic fallback that calls no model. The
fallbacks **select and never compose**: every content line of a fallback note
is a span of its input — its first sentences as the summary, a sentence with a
digit as evidence, a sentence ending in a question mark as a question, a
`**Term**: …` line as a key concept — and nothing else. Problems, limitations
and relations have no structural shape, so a fallback leaves them empty. The
note carries `fallback_extractor: extractive-v1` and
`fallback_found_nothing_for`, so a weaker note always says it is one.
**Fixed**, and guarded by a test that runs every fallback on sources from
unrelated fields and languages and fails on any line its input does not
contain. `POST /maintenance/fallback_notes` finds notes an older fallback wrote
with fixed sentences and offers re-extraction as a review item.

## 5. What a note looks like

Every note in the system, whatever it came from, has the same eight sections:

**TL;DR**, **Concepts**, **Key Concepts To Understand**, **Claims**,
**Evidence**, **Questions**, **Relation to other work**, **Provenance**.

And the same eight header keys: `type`, `title`, `source`, `sha256` (a
checksum of the original), `status`, `reading_status`, `concepts` and `topic`.

The point of the fixed shape is that anyone browsing the collection — and any
search across it — gets the same entry points no matter what kind of document
they landed on. A paper and a meeting note look different in their detail and
identical in their skeleton. **Fixed:** the mandatory sections and keys.

On top of that, each kind adds what only it needs. A paper gets Methods and
Results; a decision record Context, Decision, Consequences and Alternatives
considered; a meeting note Decisions, Action items and Risks; an image Caption,
What it shows and Where it came from; a reference list About this list and its
Source entries. A kind is not marked down for lacking
another kind's sections.

**What does not apply is said, never invented** (Track AO). A note can state
that a section or a facet does not apply to it (`sections_not_applicable`,
`facets_not_applicable`, written by the extraction or the facet review), and a
type declares which of its sections may be empty (`optional_sections`). Lint
counts these as answered, and nothing asks for a value to be made up.

**File names are plain ASCII, spelled the way people write them.** A name the
platform gives a file (a note, its draft, a converted copy, a chapter, a
concept note) spells a letter out rather than dropping it: `Übersicht der
Prüfungen` becomes `uebersicht-der-pruefungen` (`ä` `ae`, `ö` `oe`, `ü` `ue`,
`ß` `ss`, `æ` `ae`, `œ` `oe`, `ø` `oe`, `þ` `th`; every other Latin letter
loses its accents). A title in a script with no Latin spelling (Greek,
Cyrillic, Chinese, Arabic, ...) is never guessed at: the name falls back to
the source's own file name, then to a short name from a hash, and the title
stays, in its own script, in the note. **Configurable:** the install's
`config/naming.json` (`{"spellings": {"ö": "o"}}`) changes or adds a
spelling. **Fixed:** a file that already exists keeps its name; renaming it is
a separate operation with every reference following (`POST /rename`, and
`POST /rename/ascii` for names given before this rule; section 18).

**Kinds an instance declares.** **Configurable** in `config/note_types.json`:
a type's own sections, the graph node type each section's items become
(`section_nodes`), each section's **role** (`section_roles`: `answer`,
`question`, `limitation` or `context`), the fields that identify one of its
notes (`identity_fields`: `identifier`, `date`, `person`, `organisation`) and
its facets. Roles, never type names, decide ranking and the gap views: an
`answer` or `limitation` section ranks like a paper's methods, a `limitation`
section feeds the shared-limitation views. So a recipe's "What can go wrong" or
an account's "Commitments" join with no code change. Identity fields are what a
question, a caller's `scope: {"identifier": …}`, an owner's note and folder
triage recognise a note by.

**Notes written by hand.** A note in the wiki (at the top or in a folder of its
own) that the pipeline did not write is a **native note**, and counts as much
as any other: ingestion reads it under its own headings (a heading that stands
for one of its type's sections, `## Risks` for `Risks / blockers`, gives that
kind of item; any other gives a `Statement`), its paragraphs by sentence and
its numbered and `*` items, its own short names, and its images and formulas
(as Figure and Equation nodes). Its file is never changed. When it states no
concepts, its concepts are derived and kept beside it in
`maintenance/native_notes.json`: by the free extractive fallback until the
owner asks, then by the brain keeper (`POST /notes/native/extract`, a dry run
first, cached by the note's bytes). `GET /notes/native` lists each with its
`concepts_basis` (`stated`, `agent`, `extractive`).

**Tables from the source.** Every note, whatever its type, carries the
source's tables in `## Source tables`, copied from the converted copy by code,
never by an agent, so no value is altered and no tokens are spent. Each row of
a table of data is a fact (an `Extract`) labelled with its column names and its
table's title: the caption written above the table, read by its shape in any
language (`Table 3:`, `TABLE IV`, a label line with the title under it), else
the heading the table sits under. A table whose rows the converter merged into
one is kept as text, said to be unreliable, and gives no facts. The pipe
fragments some converters make of prose are not tables and are left out. At
most `tables.max_rows_per_note` rows are copied (200); the note says where the
rest are in the converted copy. The section is written again when the source
is processed or re-read, and no agent pass reads it.

**Glossaries and lookup lists from the source.** A list of terms with their
definitions is copied the same way, into `## Source entries`, one item per
entry as the source writes it (`- **Term**: definition`), each a `Definition`
with its term and a `DEFINES` link to the concept of that name. An entry is
recognised by layout, in any language: a definition list (Markdown `: `, HTML
`<dt>`/`<dd>`), or a term, one separator (`:`, a dash, `=`, a tab or wide
gap) and a definition, repeated through a run of lines; a definition may wrap
onto the lines under it. A run counts when it is long enough (two entries in a
reference list, `entries.min_entries_in_documents`, 5, in any other kind),
its terms are distinct, are not one word followed by numbers (`Step 1`,
`Step 2`) or numbers counting up (an algorithm's numbered lines), and it is
not wrapped prose with a colon here and there. A document that is not a list
is not split. At most `entries.max_entries_per_note` entries (500) are copied
and a definition is cut at `entries.max_definition_chars` (1,000), the note
saying so. A source taken in as a reference list whose lines are mostly not
entries is filed with its words kept as written and the warning
`reference_list_mostly_prose`. See `docs/reference_lists_v1.md`.

The **Provenance** section is the one to look at when you doubt something. It
records where the note came from, which converter and agent produced it, how
much of the original was covered, and every correction since.

**`## Links`** lists the note's connections, written by code from the graph
with no tokens: each note it is connected to, named by the relation as read
from this note, and up to five notes sharing at least two of its concepts.
`GET /view/note_map` draws one note's map.

**`## My notes` is the owner's** (Track AH). Every note the pipeline writes
ends its content with an empty section for the owner's own words, before
`## Links`. No template, agent or pass writes into it; it is carried through
every rewrite byte for byte (processing the source again, re-reading, a
refresh, a rename, a rollback), and a removed source's words are kept in a
file and named by a review item. It is never read as the source: no claim,
concept or relation comes from it, and the accuracy checker and lint leave it
alone. A question can find a note by its words, and an answer lists them apart
as the owner's (`from_your_work`, `kind: my_notes`). An install renames the
section under `owner_section` in `config/note_types.json`.
`POST /maintenance/owner_sections` adds it to notes made before it (a dry run
first; a real run needs `maintenance_write`).

## 6. Three tiers: the collection, the owner's work, and ideas

A brain holds three kinds of content, and an answer never confuses them.

**The collection** is every note of a source and every native note: the only
evidence. Its statements rank, are cited, and appear in `knowledge_items`.

**The owner's own work** is what the owner writes about their own work or
about the collection, registered where it lies (`POST /owner-work`, an access
mode of `maintenance_write`; the file is never modified, no agent is called).
Three kinds:

| kind | for | its parts |
|---|---|---|
| `notes` | a topic or something that happened: a meeting, a call, a visit, a day | what happened, thoughts, next steps, open questions |
| `design` | the owner's own contribution: a method, a design, a draft | problem, approach, method, assumptions and limits, comparison, open questions |
| `commentary` | someone else's work | what the source says, how the owner adapts it, their assessment |

It is read from the owner's own headings (shipped English patterns, an
instance's own in `config/owner_work.json`, or a per-note mapping). **What it is
about** is linked automatically: a stated `about` (a key, a link, a citation, a
title, an identifier), a commentary's first citation, or, for a note that names
nothing, the one source it shares most with — a date, the people and
organisations named, an identifier, the title. A link is made only when the
best candidate is clearly ahead and rests on a date, a name, an identifier or a
whole title; otherwise the candidates go to the owner as an `owner-work-link:`
review item, never guessed. A commentary that states a value differently from
the source note it discusses raises `owner-work-differs:` (either may be
wrong); one about a work the collection lacks raises `owner-work-unheld:`,
naming a file in the intake folders that looks like it. Owner work is never
evidence: ranking skips it, answers list it apart under `from_your_work`, and a
design's assumptions and limits are reported as the owner's own open problems.
Concept notes gain a `## Your work` section. The owner's words in a note's
`## My notes` (section 5) are listed the same way, never as evidence.

**Ideas** are what someone thinks might be worth doing: `type: idea` notes in
`sources/wiki/ideas/` with a `status` (`seed`, `exploring`, `parked`,
`dropped`, `done`) and an `origin` (`owner`, `agent`, `gap_filler`,
`conversation`). Captured by `POST /ideas` or written by hand; accepting a gap
proposal writes one. Never evidence: answers list them under `related_ideas`.
After every ingest a source that bears on an idea's "what would test it" raises
`idea-tested:`; `POST /ideas/review` raises stale and duplicate ideas. Nothing is
acted on.

## 7. Facets — the tags that make things findable

Each note carries a handful of tags drawn from a **closed** list. Closed means
a value that is not registered in the workspace's vocabulary file is discarded
before it ever reaches a note. That is what makes the tags usable as an index:
there is exactly one spelling of each.

**Facets belong to the system** (Track AQ). Only the platform assigns values,
proposes new ones and grows the vocabulary: extraction proposes a value where
none fits, and a proposed value is registered once enough documents ask for it
(the first vocabulary forms from the first documents' proposals; the nightly
run, or `POST /maintenance/vocabulary/promote` on request, registers the
rest). The routes that write facets or curate the vocabulary refuse a caller
agent (`facets_managed_by_system`); a caller reads them only.

Five facets ship. What they *mean* in your collection is yours to define.

| facet | max values | captures |
| --- | --- | --- |
| topic | 4 | what the document is about |
| about | 3 | the named things it concerns — a client, supplier, project, place, piece of equipment |
| function | 2 | what you *do* with it: look it up, follow it, it binds you, it records, it plans |
| authority | 1 | how much weight the source itself carries |
| locus | 1 | which part, layer or stage of things it concerns |

`topic`, `about` and `function` are asked of every kind of document, because
every document is about a subject, concerns something named, and is used for
something. `authority` and `locus` are per type.

**Configurable:** the set is declared in the install's `config/facets.json`,
and a workspace may declare its own in its `config/facets.json` (a facet may
not be named after an existing note field; removing one is refused while a note
still carries it: `POST /maintenance/facets/remove`, `GET /ops/facets`). Each
type says which facets it is asked for (`note_types.json`). The values live in
`config/facet_vocabulary.json`, each with a one-line definition (a value with
none is rejected) and optionally **other names it answers to**:

```json
"value-added-tax": {
  "definition": "A consumption tax charged on the value added at each stage of supply.",
  "aliases": ["VAT", "sales tax on value added"]
}
```

Aliases are ways of **asking** for a value, never second spellings to store. A
question using one finds the documents tagged with the registered value, and an
extractor answering with one has its answer rewritten to the registered
spelling. An alias that is already another value's real name, or is claimed by
two values, is refused rather than guessed.

Facets do more than filter. When you ask a question, the system maps it onto
the vocabulary (semantic expansion) and uses that to reach documents that share
no words with your question but share meaning. Renaming or removing a facet
does not rewrite existing notes, so run `scripts/rebuild_graph.py` afterwards.

## 8. Concept notes

One maintained Markdown note per concept the collection's notes name, in
`sources/wiki/concepts/`: what the idea is, how understanding of it developed
across sources (oldest first), where sources disagree and how they model it
differently, its open problems (the limitations sources state about it), broader
and narrower concepts, the owner's ideas and own work about it, and its sources,
each linked. A name the collection already knows under another spelling joins
the existing note; two notes for one idea are merged (reversibly,
`POST /concepts/merge`, `/concepts/unmerge`); a nightly curation pass proposes
what it is unsure of. An owner can mark a concept note as theirs
(`maintained_by: owner`): its prose is then never rewritten, and sources it has
not absorbed are listed instead.

A concept note is a synthesis: answers point to it, but cite the sources it
lists. **Configurable** in `config/concepts.json`: written by the agent
(`synthesis: agent`, one call per new document) or assembled without tokens
(`deterministic`), and the curation policy.

**What upkeep costs** (Track X). Each source taken in is read once against the
concept notes it names, and what that call carries does not grow with the
collection: the concept's summary and definition, the latest part of a long
story, the latest few sources, a capped list of the other concepts (those the
source names first). How the sources model an idea is compared in full when
first written; later sources are placed against that comparison, and the whole
comparison is redone only nightly or on request, once the concept's sources
have doubled. A batch (a folder, a book's chapters, a split paper's sections)
is one concept pass after its last piece. A single `/sources/process` can
leave its upkeep for later (`concepts: defer`); `POST /concepts/sync` or the
nightly runner pays it, and answers warn (`concept_layer_behind`) until then.
The cost a processing result reports includes the upkeep it caused
(`cost.concept_upkeep`), and every concept call in the log names the sources it
was made for.

## 9. Relations — how one note bears on another

Facets and concepts tell you which notes are *about* similar things. They do
not tell you that one note cannot be understood without another. Relations do,
and they are the thing you can actually follow: "what must I read before this?"
is a question about relations, not about similarity.

**The shipped set**, which any collection can fill — a lab, a law practice and
a kitchen each have prerequisites and contradictions:

| relation | read the other way | means |
| --- | --- | --- |
| requires | required_by | the source can't be understood or carried out without the target first |
| builds_on | built_on_by | the source extends, refines or corrects the target |
| contradicts | *(itself)* | the two make claims that can't both hold |
| example_of | has_example | the source is a concrete instance of the general case |
| part_of | has_part | the source is a component, section or stage of the target |
| follows | followed_by | the source comes directly after the target in an intended order |
| version_of | *(itself)* | the two are versions of one work (a draft and its final form, two editions); written by adding documents (Track AJ) |

**Configurable** in `config/relations.json`, which a workspace may override.
Keep the set small. In the note, in a `## Relations` section:

```markdown
## Relations
- requires [[canal-survey-method]] :: "the survey must be complete and closed first"
- contradicts [[dredging-interval-dissent]]
```

The first word is an identifier, not prose, so the same line works in any
language. A relation can also be a frontmatter key (`part_of: overview-note`).
Either end can write it and only one link is stored; an existing note is never
edited to receive one; a misspelt relation creates nothing and is reported by
lint and `GET /ops/relations`; pointing at a note not yet imported is fine.

Read them with `GET /query/relations?node_id=` (both directions; incoming links
named by their reverse reading, which is where backlinks come from),
`GET /query/relation_chain?node_id=&relation=requires` (the whole run, nearest
first; `direction=back` for what needs this note), and `POST /query/path` with
`both_directions: true`. **What is not here yet:** every relation today is one
a note states; relations inferred by reading two notes are designed but not
built (`docs/typed_relations_plan_v1.md`).

## 10. Workspaces and collections — one instance per collection

Everything is scoped. A workspace is one collection (the API says workspace,
a person is told collection): its own notes, its own configuration, its own
database, its own brain keeper state. They live side by side, by default under
`sandboxes/<workspace_id>/`, and do not see each other.

```
sandboxes/<your workspace>/
  config/        how this instance behaves          ← changed from the conversation, or by you
  sources/
    raw/                 originals as provided
    converted_markdown/  documents as text, with figure and equation catalogs
    templated_markdown/  filled into the template
    wiki/                the notes themselves       ← your real content
      concepts/ ideas/ own-work/ maps/              ← written by the brain, or registered
  data/          database, backups, exports         ← generated
  logs/                                             ← generated
  maintenance/   review queue, state, tails         ← generated
  brain_keeper/  agent job records, answer cache    ← generated
```

You are responsible for `config/` (most of it is changed through
`change_setting`, section 15) and `sources/wiki/`. Everything else is derived
and rebuildable; never hand-edit it.

**Making and opening a collection.** From the conversation (Track AG2,
`docs/collections_v1.md`): `create_collection` (`POST /collections`) with a
name in any script and a sentence of what it will hold; it first asks where
it goes (Track AZ1): the folder the caller's AI program works in (sent as
`working_folder`, never one inside the program's own folder), a folder the
person names in full, or where the install keeps collections, said to be
inside the program's own folder when it is (a folder already holding other
files gets a folder of its own inside it, so the person's files are never
mixed in). `open_collection`
(`POST /collections/open`) opens an existing collection's folder, from
another computer too, and says what did not travel: named folders for
originals that are not on this computer, search entries built by another
model, an AI program that is not here. Neither spends a token.

**One connection per collection.** Over MCP a connection serves one
collection. A connection made in welcome mode is bound to the collection it
makes or opens, in the same session, and starts bound next time; a further
collection gets a connection of its own. `connect_agent`
(`POST /collections/connect`) adds the collection's own connection to another
AI program (Claude Code, Codex, Cursor, OpenClaw), reading only until the
owner says otherwise.

**Who may do what** (the access policy). The `agents` table of the
collection's own `config/mcp.json`, read on every call and kept in the
folder, so it travels with it. Three levels, each including the one before:

| level | may |
|---|---|
| `read` | the read tools; every agent the policy does not name |
| `propose` | also file proposals for the owner's review (`remember`, `forget`; section 12) |
| `manage` | also the setup tools; an entry says what it `may` do: `add` documents, change `settings`, `grant` other agents, and the most one addition and one day may be approved to spend (`per_run_tokens`, `per_day_tokens`) |

The collection's maker is its manager, with every ability and no cap.
`set_access` (`POST /collections/access`) shows the change as one sentence,
then applies it with its `confirm` code. An agent never raises its own access;
changing another's needs `grant`. **Without access tokens the platform has no
authentication:** over MCP the connection says which agent it is and the
policy binds it; over HTTP any program on the computer can name any agent, so
the routes bind only an honest caller, and the guard over HTTP is the approval
of each plan (section 14). **Access tokens** (Track BB,
[[access-tokens-v1|`docs/access_tokens_v1.md`]]), per collection and
off until its owner turns them on (`require_token`, "Access keys for AI
programs"): a caller proves its agent by holding that agent's token for that
collection (`sign_in` over MCP, `Authorization: Bearer` over HTTP); only a
fingerprint is kept, beside the policy; issued by `scripts/access_token.py`,
by `issue_token` and by a join, taken back by `revoke_token`; the owner's web
page signs in by a one-time link the start script prints. On one computer
under one login an agent that can read any file can read another's key file:
tokens stop mistakes and callers from elsewhere, not a determined local agent
with a shell.

**A caller agent of its own** (Tracks AG4, AG7,
`docs/install_and_caller_agent_v1.md`). `make_agent`
(`POST /collections/agent`) adds a Kovium agent for this collection to
the person's project, beside their own agents, never over a file it did not
write: a Claude Code agent file, a Codex custom agent (used when asked for by
name), or a marked section of an OpenClaw workspace's `AGENTS.md`. Cursor gets
no caller agent yet. It answers in the collection's **style**
(`config/style.json`: simple or technical, short or detailed, a tone and
language in the person's words, anything else), changed with `change_setting`.
A style changes how things are said, never what: every answer still shows its
sources, says "the collection does not say" plainly, shows the plan and its
cost before anything spends, and relays every warning that changes what an
answer means. Once a style is chosen, a bound connection's instructions carry
it too, so the caller agent is an offer, never a requirement.

**Over HTTP, two identifiers go on every request:** `agent_id` (who is asking)
and `workspace_id` (which instance). A request that does not name a workspace
this instance recognises is refused rather than served:

| what you sent | what comes back |
|---|---|
| no `workspace_id` | `422 workspace_id is required` |
| a workspace that is not registered | `404 unknown workspace` |

Nothing is served against a default. Earlier versions guessed, and a mistyped
name quietly created a second, empty collection that passed every health check
and answered nothing. `GET /ops/workspaces` lists the registered ids and their
agents, never a folder, and answers with no workspace named.

A script can also register a workspace directly: `POST /ops/workspaces`
(`{"workspace_id": "<name>", "agent_id": "<you>"}`). A workspace registered
this way gets none of what `POST /collections` writes (no access policy, no
nightly tidy, no approval required) until its owner turns them on.
`DELETE /ops/workspaces/<name>` removes a workspace from the list **without
deleting its notes**.

**Keeping your notes somewhere else.** Give an absolute `root_path` at
registration (or a `folder` when making a collection) and the whole collection
lives there instead — an existing vault, a synced folder, a larger disk. The
path must be absolute and may not sit inside another workspace's root. A
note's identity is anchored to its workspace root, so the whole collection
can be moved and every note keeps its identity. Registration takes no other
field: one it does not know (`root` for `root_path`) is refused with 422
`request_field_unknown`, naming the field and the one meant, and nothing is
registered. On every route, a field that looks like a misspelling of one the
route takes is refused the same way, and any other unknown field is accepted
and named in the response's warnings (`request_fields_ignored`); see
`docs/api_spec_v1.md`, "API Style".

## 11. The brain keeper

Some of this work needs judgement: deciding what the claims in a document
actually are, which of two notes is a duplicate, what a figure shows, whether a
gap is real. Kovium does not do that work itself. It hands each such task
to the **brain keeper agent**: one sealed, one-off call per task to an agent
command-line tool already installed on your machine and running on a
subscription you already pay for. Supported, in the default order: Claude Code
(`claude`), Codex (`codex`, signed in with ChatGPT) and OpenClaw (`openclaw`).
The task travels inside the message; the agent gets no tools (a vision job
excepted: Claude Code gets its file-read tool alone, confined to a fresh folder
holding only the image), no memory and no saved session, and nothing is
registered with any backend or left running. A caller agent never talks to it
directly: everything it produces reaches a caller inside an ordinary HTTP
response, after the service has validated it.

It works this way for a blunt reason: **Kovium must never produce a
metered, per-token charge.** There is no path in the codebase to a paid model
API — no SDK, no raw HTTP, no flag that switches something into API-key mode.
Known billing credentials are stripped from the environment before any tool is
launched. If the only credential available were a metered API key, the correct
behaviour is to fail loudly, not to quietly spend your money. A test suite
enforces this, and you can check it live at `GET /ops/billing_safety`.

**What it is asked to do.** About twenty-seven defined skills
(`GET /brain_keeper/skills`): extracting a source into a note, shaping it,
normalizing a note, filling type fields, assigning facets, curating the
vocabulary, writing and merging concept notes, comparing how sources model a
concept, judging duplicates and drift, interpreting gaps, judging relevance,
writing an answer, labelling clusters, proposing a long document's parts from
a compact view of it and ruling on uncertain chapter boundaries, judging
whether an accuracy finding is real against its source lines, reading a
figure, reading equations from their crops, drafting the instance's tail,
auditing extraction quality.

**When it is missing.** Every task ends in a deterministic fallback (section
4.9). The request still succeeds — with an explicit warning and a plainly worse
result. That is also the failure mode to watch for: a fallback looks like a
working system. Before trusting a new instance, confirm the backend is live:

```bash
curl "http://127.0.0.1:8000/brain_keeper/skills?agent_id=<you>&workspace_id=<name>"
```

In the `status` block, `resolved_backend` must name a real backend and not
`deterministic`, and the readiness entry for your backend must say
`available: true`; if not, it names the reason. `scripts/brain_keeper_check.py`
says what is ready and how to fix what is not.

**When a backend runs out.** A job that hits a quota limit, a rate limit, a
failed sign-in or a model the plan does not serve moves to the next backend in
the list; the one that failed rests for a while (`cooldown_seconds`, or the
reset time it states), shared by every instance on the machine. Every attempt is
recorded on the job, and every call is logged with its billed tokens
(`logs/agent_calls.jsonl`, `GET /ops/metrics`).

**Conversations.** `POST /query/gaps/interpret` with `auto` and the
operator-only `POST /brain_keeper/jobs` (off unless `allow_direct_jobs`) can
continue a conversation with the brain keeper. A session belongs to one
instance and one caller agent; an instruction written into a job's input is
ignored by design.

## 12. Asking questions

There are two ways in, and they are for different callers.

**The exploratory routes** (`/search`, `/query/retrieve`, `/query/related`,
`/query/path`, the gap queries) are for a human poking around.

**The governed routes** (`/agent/knowledge/*`) are the stable contract, and the
one a downstream tool or agent should use: `contract` (what the interface
guarantees), `answer` (answer a question), `read` (read the relevant parts of
one source or several in one call), `task_context` (gather grounding for a
piece of work) and `read_next` (what to read next). What "governed" buys you:
read-only unless narrowed deliberately; explicit scope (who asks, about which
collection); a durable audit trail (`GET /ops/governance_audit`); stated
confidence and warnings; a versioned response shape (contract v2: every path in
an answer is workspace-relative).

**Over MCP.** An agent program that speaks MCP reaches the same routes
without HTTP: `kovium-mcp` serves over standard input and output,
calling the routes in process, so no service need be running. The install step
adds the connection (section 3); the contract is `docs/mcp_server_v1.md`.

- **Welcome mode.** Started without `--workspace`, a connection has no
  collection and offers only the welcome's tools; making or opening a
  collection binds it (section 10).
- **A bound connection** serves one collection and one agent (`--workspace`,
  `--agent`), refuses to start for an unregistered pair, and audits every call
  (`surface: mcp`). It reads only, unless the collection's access policy names
  its agent higher.
- **Results** come in a compact view by default (`view: "full"` gives the
  route's whole response). A tool outside the session's set is refused by name
  with what would allow it.

| tools | listed | does |
|---|---|---|
| `overview`, `recall`, `read`, `synthesize`, `entity`, `context_pack`, `delta`, `request_tools` | a bound connection (the default surface) | the governed reads; `synthesize` is the only one that calls a model |
| `welcome`, `help`, `check_setup` | every connection | what to do next, a question about the software, what works here |
| `try_example`, `tour` | welcome mode | the example collection and its tour |
| `create_collection`, `open_collection` | a connection holding the `manage` grant (welcome mode, or a managing agent) | make or open a collection |
| `connect_agent`, `set_access`, `change_setting`, `make_agent` | a bound managing agent | another program's connection, who may do what, one choice, the caller agent |
| `plan_adding`, `start_adding`, `adding_status` | a bound managing agent that may `add` | adding documents (section 4.1); `start_adding` spends, with its plan's approval |
| `add_note` | a bound managing agent that may `add` | text from the conversation kept as a note, shown first; free (Track AY4) |
| `get_manual` | a connection holding `manage` | fetch the full manual (section 12.5) |
| `read_next`, `review_queue`, `codes`, `contract`, `capabilities` | a session widened to the `wider` surface | the reading list, the review queue's open items, the codes, the contract, the connection's capabilities |
| `remember`, `forget` | the `write` surface, where proposals are allowed | proposals for the owner's review (below) |

The `manage` grant is held only over a local connection. How wide a session
may go is the collection's own ceiling (`max_surface` in `config/mcp.json`;
absent, the default set alone); `request_tools` moves a session within it and
tells the client its tools changed. The capabilities report is also a
resource, `kovium://capabilities`. The contract's "Wiring a harness"
gives the entry for Claude Code, Codex, OpenClaw and Cursor by hand, with each
one's tool timeout and what to set for `synthesize` (Codex and OpenClaw stop a
call after 60 s unless raised). An install outside the checkout
(`pip install '.[mcp]'`, not editable) is given the checkout at start
(`--root` or `--checkout`), since the prompts, intake templates and install
configuration live there. The four caller skills say which tools replace which
calls. One measured run of three questions on a synthetic collection cost a
caller 35% fewer tokens over MCP than over HTTP with the research skill.

**Proposals, decided by the owner** (Track AB4). `POST
/agent/knowledge/propose`, the one route taking `access_mode: propose`,
files a caller's proposal as a review item and writes nothing else, and
only where the workspace's owner allows proposals (`allow_proposals` in its
`config/mcp.json`, or an agent the access policy names at `propose` or
`manage`; over HTTP and MCP alike):
`remember` (a statement, what it rests on, an optional subject) or `forget`
(a note or an item, and why). It answers `queued_for_review`, or what was
decided on the same proposal before (`already_dismissed`, ...), never that it
was stored. The owner decides in review (`POST /review/action`): an accepted
`remember` becomes a note of the collection under `sources/wiki/contributed/`,
attributed to the agent and to the person who accepted it, with its
provenance, and is evidence from then on; an accepted `forget` moves the note
aside or takes the statement out, never deleting a file
(`POST /maintenance/withdrawals/restore` puts it back). Over MCP the same
route is two tools, `remember` and `forget`, on the `write` surface, listed
only when the workspace's `config/mcp.json` allows proposals (and a
`max_surface` of `write`) and the server was started with
`--allow-proposals`. The contract is in `docs/agent_memory_contract_v1.md`.

A retrieval result is not an answer, and the contract says so out loud: an
answer names the sections to read rather than presenting node labels as prose.

### 12.1 Answers in two or three calls

Every turn of a caller re-sends everything before it, so the number of calls
and the size of each response are the cost. The governed routes are built so a
caller answers in two calls, three when equations or figures must be read:

1. **`POST /agent/knowledge/answer` with `"view": "compact"`.** The compact view
   keeps ids, labels, a pointer and a short snippet per item, and every trust
   signal, at most half the size of the full view with the same ids. A long
   question is split into its own sentences and clauses and the results fused.
   **`focus`** says where the answer rests: `single_source`, `multi_source` or
   `unsure`, decided by the caller's `scope`, by the question naming a source
   (its citekey, an identifier, a short name it defines for itself, its title),
   or by the shares of the strong evidence — from structure only, never from
   words of any language. `focus.read.body` is the read call, as given.
2. **`POST /agent/knowledge/read`** with that body: per source, compact Markdown
   of the relevant parts of the source itself (whole when it fits a shared
   character budget, else the sections the question matches, each with its
   heading path, file and lines), read equations filled in, figures and images
   kept as references with their caption and any reading, what is not read yet
   with the call that would read it, the owner's work about it, ideas resting on
   it, and what was left out. Read-only, confined to the workspace, audited.
3. Only when the question needs them: read the unread equations or figures
   (section 12.3), then answer.

Two more governed reads need no question (Track AB0): `GET
/agent/knowledge/changes` lists the notes added, changed or removed since the
caller's cursor, at least once and never skipping, and `POST
/agent/knowledge/entity` returns one card for a name (a note or any item:
what it is, where it comes from, what it is connected to), `found: false`
with close names on a miss. Both are read-only, audited and free of model
calls. A third needs nothing at all (Track AB2): `GET
/agent/knowledge/overview` says what the collection holds for a caller that
does not know yet (its notes by kind and tier, its notes by title, the
concepts most of them name), so a newcomer can ask a good first question.

A caller states the source it means with `scope`: `{"note": …}`,
`{"citekey": …}` or `{"identifier": …}` (any value of a field the note's type
declares as an identifier: an account id, a URL, a DOI). The answer then keeps
to that source: only what rests on it is listed (`source_filter`), and a
source holding nothing of the question lists nothing and says so
(`scope_matched_nothing`); every item listed is located in that source, and
a note not ingested yet says so (`scope_not_ingested`). A collection's own name never names one of its
notes: a title head that other titles, or most of the notes, also hold is the
collection's subject (a project's name heading one meeting's title), not that
note's name.

### 12.2 How a question finds a note

Four routes, additive:

1. **Shared words.** The question and the note use the same terms. A word
   counts by how rare it is in this collection. The collection's function
   words (in nearly every note) count almost nothing, and a word far rarer
   than the question's other words is its **anchor**: items naming it rank
   ahead of items that only share ordinary words, so "what does <code> stand
   for" finds the note defining the code. All of it is counted from the
   collection, never from the wording, so it holds in any language
   (`config/retrieval.json`, `anchors`; `score_inputs.anchor_tier` marks such
   an item). Rare is not the same as the subject: a task's own phrasing
   ("write a paragraph comparing approaches") is rare in a collection of
   statements too, so a rare word stays the anchor only when it carries at
   least half the information of all the question's other words together
   (information: how few notes hold a word), with rare words the collection
   holds together (at least half the notes of the rarer one hold both, so
   not one note by chance) weighed as one (`anchors.min_information_share`). One code among
   ordinary words passes; one rare word among a task's several specific
   ones does not, and is ranked by its rarity like any word. An entry of a glossary or a lookup list whose whole term the
   question names ranks first, when that term is the question's most
   specific part (no other word of the question is in fewer notes): "what
   does UV stand for" returns the `UV` entry, while a question about how
   much water a zone needs does not lift an entry for `Water`. A term of two
   letters or fewer counts only in its own case (`anchors.term_tier`;
   `score_inputs.term_tier`). A question reaching a concept also reaches the
   list that defines it.
2. **Tags.** The question names a registered facet value or one of its aliases
   (section 7).
3. **Linked ideas.** The question names a concept, and notes carrying narrower
   forms of it are reached too.
4. **Meaning.** The question is compared against the text of each section, so a
   note can be found without sharing any word with what you typed. It needs a
   local model server; when there is none, the result says so plainly
   (`semantic_search`), rather than quietly returning less.

What ranks is decided by **section roles** (section 5), so every kind of
statement ranks: a paper's methods, a meeting's decisions, a recipe's steps
and a hand-written note's statements alike. Equations rank at the level of a
question. Ideas and the owner's work never rank; they are attached apart.

If a question returns nothing at all, that is worth trusting rather than
working around: the system declines rather than guessing.

### 12.3 What the answer could not see, and what it costs to see it

When the evidence holds an equation or a figure nobody has read, the answer,
`task_context`, the read and `/query/resolve_sections` carry `unread_regions`
and the warning `evidence_has_unread_regions`: per note, which equations
(matched to the question by the headings and words around them) and figures,
an `estimate` per method, and `recover` calls that work as given (a dry run in
`body`, the real run in `run_body`). The answer itself reads files and spends
nothing. A figure the document prints no caption for is offered only when it
sits where the question is: the headings over it or the text around it match
the question's words, weighed as an unread equation's are
(`uncaptioned_offered`, with each one's heading and the words that matched).

**Reading one is a spend, made only when the answer needs it.** The code's
action is `run`, the caller's to settle: the recovery call's dry run first,
then the real run with that dry run's approval (section 14). The local formula
model spends no tokens; vision does, and a caller keeps within a cap the user
stated, and otherwise shows the estimate before the run. Either way it reports
what was **spent** (`spent_tokens`, and `spend_note`, a sentence it can
repeat), never the estimate as if it were spent. The audit records which focus notes held
unread regions and whether a recovery followed, so an answer that skipped one
can be found. `scripts/measure_answer_steps.py` measures turns and tokens of
real questions through a headless caller (operator-run, spends only with
`--yes`).

### 12.4 Trust signals

| signal | means |
|---|---|
| `weak_context`, `question_not_covered`, low `coverage_ratio` | the collection does not properly cover the question; `absent_terms` names what it lacks |
| low `provenance_strength` | few source notes state the evidence (one note: 0.57; two: 0.78; three or more: 1.0): quote the source, or do not assert the point |
| `freshness` stale | the supporting notes are old |
| `source_missing` | the original behind this evidence is gone |
| `unverified: true` | a value the brain could not trace cleanly to its source (a table whose layout was lost) |
| `image_derived: true` | read from a picture: weaker than text |
| `semantic_search` not contributing | recall may be narrower than the collection |
| `related_ideas`, `from_your_work`, `concept_notes` | the owner's ideas, the owner's work, syntheses: never findings |

**Every warning says what to do next.** A response's `notices` hold one entry
per warning code: what it means, `fix.next` (an `action` and, where one
exists, the call to make, filled with the caller's own arguments) and
`fix.verify` (a read that shows it resolved), with `fix.consent` when the call
spends or writes. A refusal of a governed or intake route keeps its `detail`
and adds `error` in the same shape (`source_not_found`, `no_transcript`,
`maintenance_write_required`, ...). The actions: `run` the call, `ask_user`
before it (only for the standing budget or a person's own originals),
`tell_user` (the remedy is the owner's: a setting, an install, a source),
`wait` on the job, `retry` later, `none`, `report`. Every code also says who
settles it (`settled_by`: `system`, `caller` or `user`) and, where a person
may meet it, the words to show them (`fix.next.user_text`). Every code and its
fix is listed by `GET /agent/knowledge/codes` and in `docs/next_steps_v1.md`;
one table in code holds them, and the test suite fails on a code missing from
it.

### 12.5 Help, and the full manual

A question about the software itself ("how do I add a book?", "what does
search by meaning cost?") goes to `help` (`GET /help`, on every MCP
connection). It answers from the user guide and the option catalogue by
matching words, rarer words counting more: no agent and no tokens, with the
section or entry each passage comes from, or one plain sentence saying it does
not know. It matches words of one language.

**The full manual** (Track AG6) is Kovium's own current documentation
taken in as a collection of its own: one note written by hand per current
document, built by the maintainers at release with no tokens
(`scripts/build_help_collection.py`). It is fetched only on a yes, offered at
the end of the tour and by `help` (`get_manual`, `POST /welcome/manual`; free,
a `git` copy of the repository named in the install's `config/help.json`),
into a collection of its own under an agent of its own. From then on `help`
asks it through the governed answer, with the document each item comes from,
and the guide's passages beside. A copy that describes another version of the
platform is said, and `help` keeps to the guide shipped with this version
until the manual is fetched again. The shipped address is a placeholder until
the manual is published; while it stands, nothing is offered.

## 13. Instance tails

The instructions the repository ships (the **head**) are general, so they read
every new document blind: they do not know that this collection already calls
an idea by one name, that its owner writes in German, or that its documents
give weights in grams. A **tail** is a short, structured, owner-approved
description of one instance — its recurring concepts with the other names they
appear under, its facet values, note types, languages, writing systems,
conventions and what the owner's work concentrates on — appended to the head
for the jobs that benefit (extraction, normalization, cluster labels).

- **It lives only in the instance** (`maintenance/tails/`); the repository
  delivers the mechanism and never contains a tail.
- **It describes, never commands.** Every field has a cap; a value that
  commands, hides text, links elsewhere or addresses the model is refused.
- **Drafted, then approved.** `POST /tails/draft` builds the derived part from
  the instance's own data with no agent, and the descriptive part from the
  owner's words or one brain-keeper call within `max_tokens_per_refresh`; the
  draft is a review item, and nothing is used until the owner approves it
  (`POST /tails/approve`). A glossary variant is kept only when it is shown to
  be another name for the concept; the doubtful aliases go to the owner in one
  `concept-alias-doubts:` item, and concept notes are never edited.
- **Off by default.** **Configurable** in `config/tails.json`: `enabled`,
  `cadence` (`nightly`, `weekly`, `off`) for `POST /tails/maintain`, which
  measures drift and, past `drift_threshold`, proposes a refreshed draft (never
  an approval), `max_tokens_per_refresh`, `token_budget` (what a rendered tail
  may cost on every job), `glossary_size`.
- **What it costs.** Every job that takes a tail pays its tokens; the job
  records which tail it was given, and the note says so in its provenance.
  Approving a new version re-reads nothing: a source is extracted under it when
  it is next processed. `scripts/eval_tail.py` measures a tail before it is
  switched on.

## 14. Spending, consent and reporting

Every action that spends the subscription states its cost first, needs a yes
that a person gave once or a plan's approval, and reports what it billed.

**Who settles what** (Track AR). The agents settle the work between them; a
person is asked only the standing token budget and about deleting or
overwriting their own originals. Every warning and refusal code says who
settles it (`settled_by`: `system`, `caller` or `user`) and carries the words
a person may be shown (`fix.next.user_text`); see `docs/next_steps_v1.md`.

**The standing budget.** `config/budget.json` of the collection,
`tokens_per_run`: the most one addition may spend without a question, read
and set by `GET` and `POST /ops/budget` (audited). It is the one question:

- with no budget set, the first plan with a cost says what it will probably
  use and could use at most, and asks whether additions may use up to a
  proposed figure each time; yes sets the budget and runs (over MCP, one
  `start_adding` with `keep_as_limit`);
- with a budget too small for an addition, it asks to allow more this once;
  no runs within the limit and stops cleanly, and a stopped run carries the
  same question to go on;
- with a budget between the expected figure and the high end of the range,
  nothing is asked and the plan says it could pass the limit.

Several things spend within the standing budget with no separate question:
the brain keeper's look at a book's parts when the free steps found none
(section 4.5), its reading of the accuracy checker's findings (at most a tenth
of the budget a pass, section 17), the concept pass's one retry of a failed
call, and an addition's retries.

**The approval of a plan.** Every route that spends gives an `approval` code
with its dry run, and its real run takes it back: `POST /sources/add`,
`/sources/process`, `/sources/process_book`, `/sources/process_folder`,
`/sources/process_videos`, `/sources/figures`, `/sources/equations`,
`/sources/reread`, `POST /concepts/sync` (a collection that writes its concepts
without the AI spends nothing there and needs none) and `POST
/query/gaps/interpret` with `auto: true`. The code binds the route, the collection, every argument
that changes what is done, the bytes of the files named and the budget, and is
valid for 24 hours by default. `POST /sources/add` always needs it; the other routes
need it where the collection's `config/intake.json` says
`approval_required` (written on by `POST /collections`; off in a collection
registered before, until its owner turns it on with `change_setting`). A run
without a matching approval is refused before anything is read
(`spending_approval_required`, `spending_approval_mismatch`,
`spending_approval_expired`), with the dry run as its next step. Reading
equations with the local formula model (`method: local`) spends no tokens and
needs no approval. The code proves a plan was made for exactly this call, not
that a person read it.

**Caps per agent.** A manage entry's `per_run_tokens` and `per_day_tokens`
(section 10) limit what can be approved for that agent: a plan never proposes
more, and a run above the cap now is refused (`adding_over_agent_limit`). A cap
limits what can be approved; it never replaces the approval.

**At night.** The nightly steps that use the AI run only within
`ai_tokens_per_night` (section 20), chosen as a level in plain words (off,
light, medium, heavy; Track AX). The welcome asks once per install; a
collection made before the answer has `0`, which holds them, and one made
after starts at the answer.

| action | consent | estimate | report |
|---|---|---|---|
| adding (`/sources/add`) | the plan's approval, within the standing or a one-time budget | the plan: `estimated_tokens`, `estimated_range`, `also_possible` | the six-line report; per document `billed_tokens`, `cost` |
| lower-level intake (`/sources/process`, books, folders, videos) | the dry run's approval where required; else the request itself | the dry runs (a range per document, a book's tokens per chapter, time) | `billed_tokens`, `cost` (`extraction` and `concept_upkeep`, or upkeep `deferred` with who pays it), `extraction_reuse` |
| vision on figures and images | `vision.enabled`, or a real `POST /sources/figures` | its dry run | `billed_tokens` per figure |
| reading equations | a real `POST /sources/equations` | its dry run, per method | `spent_tokens`, `spend_note` |
| reading a source again | a real `POST /sources/reread` | its dry run (re-extraction, vision, concept sync) | `billed`, by operation |
| a book's parts found by the agent | the standing budget | the book's dry run (`agent.would_ask` and its estimate) | the book result's `split` and `cost` |
| the accuracy checker's reading of its findings | the standing budget (a share of it) | the fidelity dry run (`would_read`) | `judging`, in the pass and the review item |
| a native note's concepts | a real `POST /notes/native/extract` | its dry run | billed tokens |
| a tail draft | `POST /tails/draft` with an agent | capped by `max_tokens_per_refresh` | `billed_tokens` |
| query widening | a question not asked before | — | `expansion.billed_tokens`; `widen: cache_only` never spends |
| a written answer, a relevance judgment | `synthesize`, `judge_relevance` | — | in the response |
| concept synthesis, curation, gap filling | configuration, or the route | the intake estimate includes concept upkeep; `/gaps/fill` dry run | in the response; `/ops/metrics`; each concept call's `for_sources` in the log |
| the nightly AI steps | the level the person chose (`ai_tokens_per_night`, `ai_level`), asked once in the welcome | the level's most, said in brackets | the night's report, in words |

Every call is logged with its tokens (`logs/agent_calls.jsonl`), and
`GET /ops/metrics?group_by=operation` adds them up. Estimates are this
install's own once it has run the same work, and a stated default before;
every estimate says which (`estimate_basis`, `basis`). Two routes that spend
are not yet under the approval: `POST /concepts/sync` and
`POST /query/gaps/interpret` (section 24).

## 15. Configuring an instance

**From the conversation first.** Every choice a person can make is an entry of
the option catalogue (`second_brain/catalogue/options_v1.json`): what it does,
what it needs, what it costs, the recommended answer, whether it can be
changed later and at what cost, and how it ships. `check_setup`
(`GET /collections/setup`) says what works on this computer, what is off or
missing and what each costs, the nightly tidy's state, and which versions
differ from the ones that wrote the collection. `change_setting`
(`POST /collections/settings`) takes a choice by its plain name: it shows the
change as one sentence, then writes it into the collection's own file with its
`confirm` code, and the collection's services read it from the next call. It
reaches, among others, the better PDF reader, reading scanned pages, equations,
splitting long documents, search by meaning, the second check of numbers, how
concept pages are written, the collection's name, the words a collection's
documents use (contents headings, numbers written as words, measured
quantities), named folders for originals, asking before anything is spent, the
nightly tidy, its AI token limit and its equation minutes, and the four answer
style choices; `docs/collections_v1.md` has the table. A choice it does not
reach yet says how it is changed instead. The rules every text a person reads
follows are `docs/plain_language_rules_v1.md`.

**The files behind them.** Configuration lives in `config/` inside the
workspace. The repository's own `config/` holds the install's defaults; a
workspace's file holds only the keys that differ, merged key by key. A new
workspace works before you have written any of it. **A file edited by hand is
read when a workspace is first used in a running service, so write it before
the first call, or restart.**

| file | decides | where |
| --- | --- | --- |
| `brain_keeper.json` | which agent tool does the thinking, in what order, at what capability; the answer cache | both |
| `conversion.json` | the PDF converter, text recognition for scans, formula enrichment, equation regions and their reading, the book page ceiling, web page trimming | both |
| `ingestion.json` | vision during intake (per source and per book), uncaptioned images, remote images, the rows a note copies from its source's tables | both |
| `books.json` | how long documents are split into parts: contents headings, labels that are never parts, numbers written as words, the PDF outline, the agent's look | both |
| `broken_characters.json` | repairing letters a converter lost, and when a broken section is kept out of search | both |
| `naming.json` | how a letter is spelled in ASCII file names (section 5) | install |
| `video.json` | videos from their link: caption languages, manual first, description only, pacing, limits, the site's endpoint settings | both |
| `intake.json` | folder triage thresholds, identifier patterns, `kind_by_suffix`, the default kind, versions, estimates, `approval_required` | workspace |
| `budget.json` | the standing token budget per addition (section 14) | workspace |
| `mcp.json` | the collection's name and description, who may do what, the MCP surface ceiling and proposals (sections 10, 12) | workspace |
| `style.json` | how answers are worded for this collection (section 10) | workspace |
| `sources.json` | copy or reference originals, named source roots, rename repair | both |
| `note_types.json` | the kinds of note this workspace keeps: sections, roles, identity and work fields, optional sections, facets, the owner's section's name | workspace |
| `facets.json` | which facets exist and how many values each takes | both |
| `facet_vocabulary.json` | the closed set of tag values, with definitions and aliases | workspace |
| `concepts.json` | concept notes: agent or deterministic synthesis, curation, retries | both |
| `owner_work.json` | heading patterns for the owner's notes in another language; comparison thresholds | workspace |
| `ideas.json` | idea headings in another language; when a source tests an idea, and when an idea is stale or a duplicate | workspace |
| `reading_fidelity.json` | page sizes for reading a source in full; the fidelity checks' tolerances, quantities, conflict, condition, label and method words, and the settings for reading findings against their source | both |
| `tails.json` | the instance's tail: on or off, cadence, drift threshold, costs | both |
| `gap_filler.json` | the proactive gap filler: on or off, topics, budget | workspace |
| `embeddings.json` | passage embeddings: on or off, which local model, thresholds | both |
| `semantic_expansion.json` | how questions are mapped onto the vocabulary | both |
| `retrieval.json` | ranking settings, question decomposition, anchor words | both |
| `relations.json` | the relation set | both |
| `maintenance.json` | the nightly tidy: on or off, on which computer, when, its AI token limit, its equation minutes | workspace |
| `help.json` | where the full manual is fetched from, and its size (section 12.5) | install |
| `sys_config.json` | prompt files, the document converter's command and reach, long-document reading | both |
| `schema_version.json` | the storage version and the upgrade policy | both |

**What is fixed**, for every instance: the layered pipeline and the mandatory
note shape; the intake templates; the three tiers (collection, owner's work,
ideas) and that the latter two are never evidence; that nothing spends without
a dry run, a plan's approval or a stated setting; that a person is asked only
the budget and about their own originals; that no metered API is ever called
and no model is asked to stay resident; that the owner's files, notes marked
as theirs and every note's owner's section are never rewritten; that every
path written down is relative to the checkout or the workspace; that fallbacks
only copy their input; and the governed contract's read-only default and
audit.

### `brain_keeper.json`

```json
{
  "backend": "claude_code",
  "fallback": ["codex", "openclaw"],
  "fallback_on": ["quota_exhausted", "rate_limited", "auth_failed", "unavailable", "model_unavailable", "timeout", "context_exceeded"],
  "cooldown_seconds": {"quota_exhausted": 3600, "rate_limited": 120, "auth_failed": 1800, "model_unavailable": 3600},
  "model_tier": "balanced",
  "models": {},
  "max_concurrent_jobs": 2,
  "allow_direct_jobs": false,
  "answer_cache": {"enabled": true, "max_age_days": 30, "max_entries": 2000},
  "normalization_mode": "brain_keeper",
  "source_pipeline_mode": "brain_keeper"
}
```

- `backend` — tried first: `claude_code`, `codex`, `openclaw`, or `auto` (the
  first ready one). Then each entry of `fallback`. Backends that are not
  installed, or are signed in only with a metered key, are skipped. The
  deterministic tier is always last.
- `model_tier` — `strong`, `balanced` or `fast` for the model CLIs; pin exact
  models under `models`. OpenClaw runs whatever model it is configured with.
- `answer_cache` — section 4.8. Never for a job carrying conversation history.
- `normalization_mode`, `source_pipeline_mode` — `deterministic` skips the
  agent for that stage: offline, or in automated testing.
- No API key appears here or anywhere else. `KOVIUM_BRAIN_KEEPER_DISABLED=1`
  forces the no-model path globally.

### `conversion.json`, `ingestion.json`

```json
{"pdf_converter": "auto", "formula_enrichment": false, "table_structure": true, "figures": true,
 "figure_image_scale": 2.0, "ocr": "auto", "html_trim": true, "equation_regions": true, "timeout_seconds": 3600,
 "docling_python": null, "equations": {"max_per_call": 8, "token_budget": 40000, "context_chars": 400},
 "docling_book_page_ceiling": 150}
```

```json
{"vision": {"enabled": false, "max_figures_per_source": 6, "token_budget": 30000,
            "opening_chars": 1500, "referencing_chars": 4000, "interpret_uncaptioned": null,
            "books": {"enabled": false, "max_figures_per_book": 20, "token_budget_per_book": 150000}},
 "images": {"fetch_remote": "fetched_pages", "max_per_source": 40},
 "tables": {"max_rows_per_note": 200}}
```

Sections 4.2 to 4.5 and 5 say what each does. `html_trim` converts a web
page from its main content (false converts it whole); `tables` caps the rows a
note copies from its source's tables. `docling_python` names another
interpreter that has docling; `ocr` reads page images (for scans; `auto`
reads only the files whose pages hold no text, section 4.2);
`timeout_seconds` stops a conversion that runs too long, after which the basic
extractor is used. `interpret_uncaptioned` is `null` (each template's own
default), a boolean, or `{"<template_id>": true|false}`. A malformed value is
replaced by its default and reported. `GET /ops/self_check` reports what is in
effect.

### `sys_config.json`

The document converter's command and reach (`conversion.converter_command`,
`converter_allows_any_path`: by default each document is copied into a folder
the external `oc-markitdown` tool already accepts, rather than widening what it
may read), the prompt files, and how long documents are read
(`normalization.batch_budget`, `batch_concurrency`, `batch_attempts`,
`paper_single_pass_chars`, `paper_batch_budget`, `paper_split_chars` (0
switches splitting off), `paper_min_section_chars`, `known_concept_limit` (how
many existing concept names extraction is shown; 0 switches the hint off)). The
defaults are for a collection we know nothing about: a corpus of short memos
never reaches them, a corpus of long surveys will want them tuned.

### `note_types.json`

How a workspace stops being about someone else's subject matter (section 5).
Every type gets the mandatory sections and keys for free; list only what is
specific to it. A type that declares no facets is asked only for `topic`.

### `embeddings.json`

**On by default**, and it needs a locally running model server. With nothing
answering, the collection behaves exactly as without it, and says so: every
search result carries a `semantic_search` block (with a code: `disabled`,
`no_model`, `no_profile`, `endpoint_not_local`, `server_unreachable`,
`not_indexed`, `stale_format`, `over_cap`, `error`, or `no_match` when it ran
and found nothing close enough), a governed answer warns
`semantic_search_unavailable`, and `GET /ops/embeddings` reports the whole
installation, including passages stored by another model or an older format.
**Fixed:** a non-local address is refused; nothing is ever sent off the
machine; no model is asked to stay loaded. Changing `model` invalidates what is
stored; `scripts/rebuild_vectors.py --workspace-root <root> --purge` rebuilds
it. `max_scored_vectors` (7,000) is a deliberate ceiling past which the
signal is skipped rather than computed over a subset. **No passage is cut
short or lost for its length**, on any machine: one past the model profile's
`max_words` or `max_chars` is split at its paragraphs, lines or sentences and
embedded part by part, its vector the mean of the parts; one the server still
refuses as too long is halved until it fits; and the server is asked never to
trim a text in silence (servers differ, by version and by text, in whether
they trim or refuse). `scripts/rebuild_vectors.py` reports how many passages
took parts (`SPLIT`). Lower `min_similarity` for
a collection of short notes. **A note without vectors is caught up by
itself:** one taken in while the server was down is indexed by the nightly
run once the server answers, and `POST /ops/embeddings/reindex` (one note,
`missing: true` or `all: true`; free, the same code as the script) does it at
once.

### `sources.json`

```json
{ "source_retention": "reference",
  "source_roots": { "library": "/data/library", "fieldnotes": "D:/Records/Notes" } }
```

`copy` (the default) copies a source from outside the workspace into
`sources/raw/`; `reference` reads it where it lies, under a named root, recorded
as `library:papers/a.pdf`, which names no home directory and survives the
library moving (change the one line). A file outside the checkout, the
workspace and every declared root is **refused at intake** under `reference`.
Retrieval never depends on the original surviving; refreshing and re-reading
do. `GET /ops/sources` says where each original is and whether it is present,
missing or moved. Roots go in the workspace's own file, never the checkout's.
`rename_repair` (`auto` or `propose`) decides whether a renamed original is
repaired in its notes or proposed for review (section 18).

### `maintenance.json`

```json
{ "enabled": true, "machine": "<this computer>", "time": "02:00",
  "ai_tokens_per_night": 0, "ai_level": "off", "equations_minutes": 60, "refacet_batch": 5 }
```

What `POST /collections` writes. Without the file, or with `enabled: false`,
nothing runs at night; `check_setup` says so. `change_setting` turns the tidy
on here, sets the nightly AI level (`light`, `medium`, `heavy`, `off`, or a number of
tokens shown as custom) and the equation minutes;
`scripts/maintenance_cron.py enable <workspace_id> --at 02:00` does the first
from a terminal (`status`, `disable`). A schedule without
`ai_tokens_per_night` (written before the levels) is read as medium, the
catalogue's `level_when_unset`, computed on read and never rewritten (Track
AX). `ai_level` is the name shown back. Section 20 says what the night does.

## 16. A normal working day

**Adding something.** Ask the agent to add it, or `POST /sources/add` with
the file, folder or link: a free plan first, then the run with its approval
(section 4.1). Then read the note it produced — not the status codes, the
actual note. Sections should be full, claims should be separate from open
questions, and qualifiers should still be there. Status checks cannot catch a
note that is well-formed and wrong.

**Adding a lot of something.** Add a few first and read their notes, then the
rest. A systematic extraction problem found after two hundred documents costs
vastly more than the same problem found after three.

**Writing your own notes.** Write them in the wiki (native notes, counted as
the collection), keep them where you keep them and register them
(`POST /owner-work`, kept apart from the collection), or write a line under
any note's `## My notes`. Note an idea with `POST /ideas`.

**Asking.** The governed answer in compact view, then one read of what its
focus names (section 12.1). Follow the provenance on anything you intend to
rely on.

**Finding what is missing.** The gap queries: unanswered questions, weakly
validated methods, missing bridges, limitations several sources share
(`/query/gaps/limitation_clusters`), and the gaps under one topic
(`/query/gaps/topic`). `POST /query/gaps/interpret` adds the brain keeper's
reading. `/gaps/fill` (off until `config/gap_filler.json` enables it) watches
topics and only raises proposals.

**When something looks wrong.** Put it in the review queue rather than editing
the derived data (section 17). A name that is wrong is changed with
`POST /rename` (section 18), never by hand in the store.

## 17. Review and correction

The review layer exists so that fixing something is visible and undoable rather
than an invisible rewrite.

- `POST /review/queue` — what currently needs a human's attention: staleness,
  duplicates, drift, and work items (`correction:`, `fidelity:`,
  `source-missing:`, `source-rename:`, `name-rename:`, `tail-draft:`,
  `owner-work-link:`, `owner-notes-kept:`, `idea-tested:`, `gap-proposal:`,
  `concept-merge:`, `fallback-note:`, a caller's proposals, …)
- `GET /review/source` — the original evidence, paged by `line_start` and
  `line_end` through the whole source (its converted copy for a PDF or another
  original that is not text), with `next_line_start` until the end
- `POST /review/action` — record a decision; on a work item, resolve or dismiss
- `GET /review/history` — everything that has been decided, and by whom

**Reading fidelity.** A note can be right on the method and wrong on a number.
A value found only in layout-lost text is marked `[unverified: layout lost]` at
intake; `POST /maintenance/fidelity` (a dry run first, free) checks
existing notes against their sources — table values in another row, totals that
do not add up, units that do not fit their quantity, two figures called
inconsistent when the arithmetic says they agree — and raises one item per note.
A value after a condition word, a number inside a name or a label, and a method's
name are not raised (Track AS1 to AS3, word lists per language in
`config/reading_fidelity.json`); and in a real run, within the standing token
budget, the brain keeper reads each statement-level finding against its source
first, keeps the false alarms out of the queue and says in one plain sentence
which findings it could not judge (Track AS4).
`POST /maintenance/corrections` files a correction to one statement (allowed
read-only; it is a proposal); accepting it patches exactly that statement, with
an attributed provenance line and a snapshot
(`POST /maintenance/corrections/rollback`).

**Reading a source again.** `POST /sources/reread` re-converts a source with a
chosen converter: the dry run converts into a scratch folder for free and shows
the before/after comparison (equations, tables, rows, figures) and the estimate;
a real run keeps a snapshot (`POST /sources/reread/rollback`), re-processes the
source, and closes the corrections and fidelity items the new reading answered.

**Repairs that touch no meaning.** `repair_frontmatter` writes a header block
derived only from what the note and the files on disk already say,
`repair_sections` inserts an empty stub for a missing mandatory section, and
`repair_metadata` fills derivable fields. None of them writes a sentence of
prose. `POST /maintenance/extraction_audit` looks for text in the graph that is
unusable as knowledge, such as extraction damage.

## 18. Originals renamed, moved or deleted

Notes record their original by path and by the hash of its bytes.
`POST /sources/reconcile` (a dry run first; no model call, ever) matches every
recorded source to the files present: the same bytes under a new name or folder
are repaired in the notes (under `rename_repair: auto`) or proposed; a renamed
and edited file is always proposed; a source that is gone marks its notes
`source_status: missing` and opens a `source-missing:` item for the owner to
keep the knowledge or remove exactly that source's knowledge (after a backup).
Every write is snapshotted (`POST /sources/reconcile/rollback`);
`GET /sources/registry` shows each source's current and earlier paths. The
nightly run reconciles unless the workspace says not to. Answers mark evidence
whose original is gone with `source_missing`.

Anything else is renamed through one engine (`docs/renames_v1.md`):
`POST /rename` renames an original, a note with its draft, a concept (leaving
a redirect stub) or a converted copy, and moves every key and link that
recorded the old name in the same step: node and edge ids, passage vectors
(never re-embedded), read history, the concept layer's state, records and the
agent's answer cache; the change log lists one `renamed` entry. A dry run
first, a snapshot (`POST /rename/rollback`), an audit record and the rename
log (`GET /rename/log`). Renames made by hand in an editor are carried through
nightly by `POST /rename/reconcile`; `POST /rename/ascii` brings names given
before the ASCII rule to it, and `POST /rename/parts` a document's parts to
`<document-key>-<NN>-<title>`. No model call, no question to the user except
before replacing a file of theirs.

## 19. Keeping it healthy

**Lint, any time:**

```bash
curl "http://127.0.0.1:8000/maintenance/lint?agent_id=<you>&workspace_id=<name>"
```

This reads the notes on disk and reports what breaks the note contract: missing
sections, missing header fields, facet values that are not in the vocabulary.
It costs nothing and calls nothing.

The answer is counts first. Add `&detail=true` to list the findings (errors
first), or narrow with `&severity=error`, `&kind=<kind>` or `&note=<file>`; it
says how many it listed and how many it left out. Works your notes cite that
the collection does not hold are counted apart (`cited_not_held`), not as
broken links. What a note says does not apply to it, a facet or a section, is
counted as answered (`not_applicable`), and lint never asks for a value to be
invented. Characters a converter lost are counted per note
(`broken_characters`, and a `broken-characters` warning): the nightly run puts
back the letter pairs a PDF's glyphs lost ("di?erent" for "different") when the
same document spells the word intact, for free, and the sections too broken to
embed are kept out of search by meaning.

**Regenerating everything derived from your notes**, safe to run at any time:

```bash
.venv/bin/python scripts/rebuild_graph.py   --workspace-root sandboxes/<id>
.venv/bin/python scripts/rebuild_vectors.py --workspace-root sandboxes/<id>
```

Your Markdown is the source of truth and neither touches it. Indexing one
note, or only the notes missing their vectors, needs no command:
`POST /ops/embeddings/reindex` (free), and the nightly run does the latter by
itself. Connections
between ideas are withdrawn when the reason for them goes, and a phrase listing
several things is split into the things it lists.

## 20. What runs on its own

**The nightly tidy is on in every collection made from the conversation**
(Track AG5), on the computer that made it, and built around *not* spending
anything. `config/maintenance.json` names that computer (`machine`), and the
computer's own scheduler is asked to start the job (cron where `crontab`
exists, the Windows task scheduler where `schtasks` does). The job runs in
process, so no service need be left running; a computer with neither
scheduler keeps the schedule and says the job is not started yet. A collection
made before, or registered by a script, is reported by `check_setup` and not
switched on: its owner turns it on with `change_setting` ("Nightly tidy"), or
with `scripts/maintenance_cron.py enable <workspace_id>`. A collection shared
between computers is tidied by one of them; turning the tidy on at another
moves it, and says so first.

Python does the deterministic analysis for free. If nothing is flagged it
exits having called nothing; a set of flags already judged on an earlier night
is not sent again; only the notes a finding names are sent. On a collection
that has not changed, the steady state is zero agent calls.

**What it may do on its own, all of it free and additive:** write a header
block on a note that has none; insert an empty stub for a missing mandatory
section; repair a renamed or moved original's path (under
`rename_repair: auto`) and carry through renames made by hand; put back the
letter pairs a converter lost (section 19); index the passages of notes that
have no vectors, once the embedding server answers; read the unread equations
of PDFs with the local formula model, for up to `equations_minutes` a night
(60 by default); merge concept notes whose names are typographically
identical; register a label value once enough documents have proposed it
(the step `POST /maintenance/vocabulary/promote` runs on request); bring
concept notes up to date with sources whose upkeep was deferred, in one
pass; propose (never approve) a refreshed tail.

**What it says.** The platform version, the data layout and the search
program and model that built the collection's vectors are recorded in the
collection (`maintenance/versions.json`); when this computer's differ, the
night, `check_setup`, `open_collection` and `GET /ops/status` say so in plain
words. Vectors built by another search program can be rebuilt in a copy and
swapped in only when whole (`POST /ops/embeddings/rebuild`, free); a build
that fails changes nothing.

**The steps that use the AI program** (judging label words, merging concept
pages, faceting, hints, suggestions) run only within the nightly most
`ai_tokens_per_night` of the collection's `config/maintenance.json`, chosen as
a level (off; light 50,000; medium 150,000; heavy 400,000 tokens a night: the
option catalogue's numbers, Track AX). The welcome asks once per install;
`0` is off. A schedule written before the levels has no number and runs at
medium; the example collection never uses the AI at night. A step already begun is finished, and the next waits
for the next night. The free steps run either way.

**What it will never do on its own:** merge or retire a vocabulary value;
remove a source's knowledge; approve a tail; act on a proposal, a
correction, an owner-work link or an idea; spend on vision, reading equations
by vision, or re-reading.

## 21. Operations

| what you want | call |
| --- | --- |
| is it alive | `GET /health` |
| overall status summary, with what is off and what differs from the versions that wrote the collection | `GET /ops/status` |
| what works on this computer, what is off and what it costs, who may do what | `GET /collections/setup` (`check_setup`) |
| the standing token budget | `GET /ops/budget`, `POST /ops/budget` |
| broad readiness: converters, vision, semantic search | `GET /ops/self_check` |
| are embeddings working, and are they stale | `GET /ops/embeddings` |
| storage consistency | `GET /ops/integrity` |
| which workspaces exist | `GET /ops/workspaces` |
| can this instance bill anyone | `GET /ops/billing_safety` |
| what was billed, by operation | `GET /ops/metrics` |
| who asked what, on the governed routes | `GET /ops/governance_audit` |
| what got held back as unsafe | `GET /ops/quarantine` |
| where each original is | `GET /ops/sources` |
| take a backup | `POST /ops/backup` |
| restore from one | `POST /ops/restore` |
| prove a restore works, safely | `POST /ops/restore/rehearsal` |
| export the collection | `POST /ops/export`, `POST /ops/export/okf` |

The same checks from the command line: `scripts/ops_status.py`,
`scripts/ops_self_check.sh`, `scripts/ops_integrity_check.py`.
`restore/rehearsal` is worth a habit: an untested backup is not a backup.
Storage upgrades are deliberately manual: review before upgrade, no automatic
migration on start-up.

## 22. The safety rules built in

Enforced by the code and by tests that fail the build, not by convention:

**No metered billing** (section 11), checkable live at `/ops/billing_safety`.
The install step, the connections it adds and every program command it runs
strip metered credentials too.

**Nothing spends unasked.** Vision, equation reading by vision, re-reading,
agent reading of native notes, the nightly AI steps and the gap filler are off,
held at a zero limit or dry runs by default. Every spending route gives an
approval with its dry run, and a real run without it is refused where the
collection asks for it (always for `POST /sources/add`); a cap per agent
limits what can be approved (section 14).

**A person is asked about three things only:** the standing token budget;
their own originals (deleting or overwriting one, and, at a collection's first
addition from outside, keeping a copy of each or pointing to it, Track AZ4);
and where a new collection is kept (Track AZ1). Everything else is settled by the platform
or the caller, and every code says who settles it.

**Read only unless the owner says otherwise.** A governed caller reads by
default; an MCP connection reads unless the collection's access policy names
its agent higher (section 10), and an agent never raises its own access. The
setup tools need a local connection. Without access tokens there is no
authentication: over HTTP these rules bind an honest caller, and a plan's
approval is the guard; with a collection's tokens on (Track BB), each caller
proves its agent by its own token.

**A style never drops the essentials.** However a collection's answers are
worded, they keep their sources, say "the collection does not say" plainly,
show a plan and its cost before spending, and relay the warnings that change
what an answer means.

**Nothing outside its own tree.** The platform depends on no file, format or
path outside the repository and the workspaces it owns. No absolute path to
anyone's home directory is written down anywhere: not in notes, configuration
or any log a workspace writes, and not in a response (a collection's folder is
shown relative to the install or as `~/...`). A path is shown relative to the
workspace, as `root:relative` under a declared source root, else relative to
the checkout, with forward slashes on every system; what one route returns,
another accepts. The test suite fails a run that changes the checkout. Real
collections are per-machine state and are never committed; neither is an
instance's tail.

**Shipped text names no one's subject.** Prompts, skill instructions, format
templates, the option catalogue and fallbacks are checked for subject matter
that leaked from one collection; a fallback may only copy its input. Every text
a person reads passes one plain-words checker (`docs/plain_language_rules_v1.md`).

**Fetched content is data, never instructions.** A web page, a local file or
an image's metadata is scanned for text addressed at an AI reader; if found, the
source is held in quarantine rather than discarded (`GET /ops/quarantine`,
`POST /ops/quarantine/release`).

**The owner's words are never rewritten.** Native notes, notes marked
`maintained_by: owner` and every note's `## My notes` section survive every
pass, rewrite, rename and rollback.

**No model is asked to stay resident.** Holding one saves this project some
latency and takes memory from everything else on your machine.

## 23. The API, grouped

Full, live, clickable documentation is at `/docs` on the running service; the
contract for each group is in `docs/api_spec_v1.md`. This is the map; the MCP
tools and the routes behind them are in section 12's table.

**The welcome** — `/welcome` (the next step for a newcomer), `/help` (a question about
the software, from the user guide), `/welcome/example` (the example collection),
`/welcome/tour` (its tour and waiting page), `/welcome/manual` (fetch the full
manual on a yes); `docs/welcome_v1.md`

**Collections and who may do what** — `/collections` (make one), `/collections/open`
(open one's folder, from another computer too), `/collections/connect` (another agent
program), `/collections/access` (what each agent may do), `/collections/settings` (one
choice by its plain name), `/collections/agent` (the caller agent for a collection,
added to a person's project, in the style they choose), `/collections/setup` (what works
here); `docs/collections_v1.md`, `docs/install_and_caller_agent_v1.md`

**Access tokens** (Track BB) — `/collections/tokens` (issue one for an agent, or list
them), `/collections/tokens/revoke` (take one back), `/ui/sign-in/code` and `/ui/sign-in`
(the owner's web page signed in by a one-time link); `docs/access_tokens_v1.md`

**Bringing material in** — `/sources/add` (the plan, then the run with its approval),
`/sources/add/status` (how an addition is going), `/sources/process`, `/sources/process_book`,
`/sources/book_parts/attach`,
`/sources/process_folder`, `/sources/process_videos`, `/normalize/file`, `/ingest/file`, `/ingest/sample`,
`/format-templates`, `/format-templates/resolve`, `/format-templates/{id}`

**Pictures, equations and re-reading** — `/sources/figures`,
`/sources/equations`, `/sources/equations/rollback`, `/sources/reread`,
`/sources/reread/rollback`

**Originals** — `/sources/reconcile`, `/sources/reconcile/rollback`,
`/sources/registry`, `/ops/sources`

**Renaming** — `/rename`, `/rename/rollback`, `/rename/log`, `/rename/reconcile`,
`/rename/ascii`, `/rename/parts`

**Reading and searching** — `/search`, `/concept/search`, `/question/open`,
`/wiki/page`, `/node/{id}`, `/node/{id}/neighbors`, `/edge/{id}`,
`/provenance/{id}`, `/schema/summary`, `/version`

**Exploring the structure** — `/query/subgraph`, `/query/related`,
`/query/path`, `/query/centrality`, `/query/clusters`,
`/query/cross_chapter_bridges`, `/query/bridges/summary`, `/query/priority`,
`/query/facets`, `/query/facets/assign`, `/query/relations`,
`/query/relation_chain`

**Getting answers** — `/query/retrieve`, `/query/context_pack`,
`/query/read_next`, `/query/focused_bundle`, `/query/resolve_sections`

**The governed contract** — `/agent/knowledge/contract`,
`/agent/knowledge/answer`, `/agent/knowledge/read`,
`/agent/knowledge/task_context`, `/agent/knowledge/read_next`,
`/agent/knowledge/codes` (every warning and refusal code with its next step),
`/agent/knowledge/changes` (what changed since a cursor),
`/agent/knowledge/entity` (one card for a name),
`/agent/knowledge/overview` (what the collection holds),
`/agent/knowledge/capabilities` (what the workspace can do, what is not ready),
`/agent/knowledge/review_queue` (the review queue's open items),
`/agent/knowledge/propose` (a proposal for the owner's review),
`/maintenance/withdrawals/restore` (put back what an accepted `forget` moved
aside)

**Finding what is missing** — `/query/gaps`, `/query/gaps/interpret`,
`/query/gaps/unanswered_questions`, `/query/gaps/weak_method_validation`,
`/query/gaps/thesis_bridges`, `/query/gaps/limitation_clusters`,
`/query/gaps/topic`, `/gaps/fill`

**Concepts** — `/concepts`, `/concepts/{key}`, `/concepts/sync`,
`/concepts/hierarchy`, `/concepts/open_problems`, `/concepts/ideas`,
`/concepts/your_work`, `/concepts/consolidate`, `/concepts/merge`,
`/concepts/unmerge`, `/concepts/compare`, `/concepts/curate`

**Ideas, the owner's work, native notes** — `/ideas`, `/ideas/{key}`,
`/ideas/review`, `/owner-work`, `/owner-work/{key}`, `/owner-work/template`,
`/owner-work/open_problems`, `/owner-work/refresh`, `/owner-work/review`,
`/notes/native`, `/notes/native/extract`

**The instance's tail** — `/tails`, `/tails/preview`, `/tails/draft`,
`/tails/approve`, `/tails/retire`, `/tails/maintain`

**Maintenance** — `/maintenance/lint`, `/maintenance/inspect`,
`/maintenance/refresh`, `/maintenance/registry`, `/maintenance/refacet`,
`/maintenance/vocabulary`, `/maintenance/vocabulary/apply`,
`/maintenance/vocabulary/promote` (the system's own promotion, on request),
`/maintenance/facets/remove`, `/maintenance/extraction_audit`,
`/maintenance/repair_frontmatter`, `/maintenance/repair_sections`,
`/maintenance/repair_metadata`, `/maintenance/evidence_split_audit`,
`/maintenance/fidelity`, `/maintenance/fallback_notes`,
`/maintenance/corrections`, `/maintenance/corrections/rollback`,
`/maintenance/owner_sections`

**Review** — `/review/queue`, `/review/source`, `/review/action`,
`/review/history`

**The brain keeper agent** — `/brain_keeper/skills` (skills, and each
backend's readiness, fix and rest), `/brain_keeper/jobs` (operator-only, off by
default), `/brain_keeper/jobs/{id}`, `/brain_keeper/jobs/{id}/cancel`;
`/brain_keeper/provision` is deprecated and only lists leftovers

**Workspaces** — `GET /ops/workspaces` (the registered ids and their agents),
`POST /ops/workspaces` (register one from a script; `POST /collections` and
`POST /collections/open` register too), `DELETE /ops/workspaces/{id}` (without
deleting its notes)

**Operations** — the table in section 21, plus `/ops/self_check`,
`/ops/relations`, `/ops/facets`, `/ops/knowledge_events`,
`/ops/monitoring/normalization`, `/ops/semantic_expansion`, `/ops/schema`,
`/ops/schedule`, `/ops/schedule/run`, `/ops/budget` (the standing token budget per addition),
`/ops/embeddings/reindex` (index one note's or the missing notes' passage vectors, free),
`/ops/embeddings/rebuild` (every note's vectors again in a copy, swapped in at once, free)

**The UI** — `/` for the graph view, plus `/view/graph/focus`,
`/view/canvas`, `/view/note_map` (one note's map), `/view/sidebar/details/{id}` behind it.

Most responses share a shape: a status, the payload, a timestamp, any warnings
worth noticing with their `notices` (each warning's next step, section 12.4),
and provenance or evidence references where they apply. A refusal keeps
FastAPI's `detail`; on the governed and intake routes it adds `error`, the
same shape as a notice.

## 24. What to watch out for

Most of these are deliberate trades you should be able to recognise; the last
group are known gaps.

- **A silent fallback looks exactly like success.** If the brain keeper is
  unreachable, requests still succeed — with a warning and a much weaker
  result. Check `resolved_backend` (or `check_setup`) before trusting a new
  instance, and again after any change to the machine.
- **Without docling, PDFs get the basic extractor.** Two-column pages, tables,
  equations and figures suffer. `pdf_converter: auto` uses docling as soon as
  it is installed; re-read the sources that matter afterwards.
- **Word files and web pages need markitdown.** Without it they are refused.
- **Scans need docling and a text-recognition program.** Without them a PDF
  whose pages are images is refused at triage.
- **A web page built without `main`, `article` or `nav`** keeps its menus:
  the trim reads structure only. Set `html_trim: false` for a site it misreads.
- **A table the converter merged gives no facts.** Its text is in the note;
  read it there, or re-read the source with a better converter.
- **Vision needs a backend that takes images.** An install that runs OpenClaw
  alone catalogues pictures and equations and cannot read them.
- **The intake templates are fixed.** A kind of source with no template goes in
  under the closest one; a kind of note you write yourself is declared in
  `note_types.json`.
- **Answers are grounded support, not synthesis.** Follow the citations on
  anything you intend to rely on.
- **Renaming or removing a facet does not rewrite existing notes.** Rebuild the
  graph afterwards.
- **A split document takes a while**, and its section notes inherit its own
  heading wording. A book none of the free steps splits, with no budget or no
  agent, is read whole, in parts, and the report says so.
- **A version is found only on the evidence it states** on its first pages,
  its frontmatter or its name; a rewritten, retitled version is two works.
- **A new tail or prompt re-reads nothing by itself.** A source is extracted
  under it the next time it is processed, once.
- **A file edited by hand is read once per workspace per running service.**
  Restart after changing one; `change_setting` needs no restart.
- **Storage upgrades are manual by policy.**
- **No authentication.** Over HTTP any program on the computer can name any
  agent and set the budget; an approval proves a plan was made, not who read
  it.
- **The nightly tidy needs a scheduler.** A computer with neither cron nor the
  Windows task scheduler keeps the schedule and does not start the job.

**Gaps found by the build of AR to AG, Tracks AU, AV and AW**
(`docs/knowledge_layer_improvement_plan_v1.md`, all built 2026-10-23):

- **AU, answers that stay fast as a collection grows (built).** Measured on a
  synthetic collection of about 10,000 graph nodes: an answer took 43 to
  55 s (a long question 80 to 89 s), because it ranked the question four
  times and rebuilt whole-graph summaries each time. Now the work that does
  not depend on the question is kept while the graph is unchanged (the
  store's graph revision, `second_brain/graph/kept.py`) and one answer
  scores its question once: about 3 s (a long question about 11 s), with
  the same answers. A collection that ships what was derived from its notes
  (`derived/`: the example, the help collection) is opened by reading it
  instead of building it: 83 s became a few seconds. A test holds the
  budget (`tests/system/test_answer_time_budget.py`) and the retrieval
  benchmark reports times.
- **AV, tests that pass every time (built):** the suite runs in a seeded
  random order and a guard names a test that leaves state behind.
- **AW, small leftovers (built 2026-10-23):** a nightly dry run writes
  nothing; a maintenance refresh reads relative paths from the collection;
  `POST /concepts/sync` and `POST /query/gaps/interpret` ask for an approval
  like the other spending routes; the plan's words are fixed and AR's
  sentences are in the catalogue; a further version of a set of versions is
  linked; the nightly run reads in notes the graph lost; the install step
  checks Claude Code's sign-in and a moved checkout is found in the caller
  agent file and offered again.

Also open: the help collection's address in `config/help.json` is a
placeholder until it is published, so the full manual is not offered yet; only
Claude Code has a plugin (Codex and OpenClaw get a connection and the caller
agent, Cursor a connection only, and OpenClaw's skills are copied by hand); on
Windows a checkout in a deeply nested folder can fail the install on the path
length limit.

## 25. Where to go next

| you want | read |
| --- | --- |
| a guide for a person, not an agent | `docs/user_guide_v1.md` |
| the install step, the plugin, the caller agent and the answer style | `docs/install_and_caller_agent_v1.md` |
| the welcome, the example, the tour, help and the full manual | `docs/welcome_v1.md` |
| collections, who may do what, `change_setting`, `check_setup`, the nightly defaults | `docs/collections_v1.md` |
| adding documents, the budget, the approval on every spending route | `docs/adding_documents_v1.md` |
| the option catalogue and the rules every text a person reads follows | `docs/plain_language_rules_v1.md` |
| to operate an instance as an agent | `skills/` — the four for your agent tool (the Claude Code plugin installs them) |
| every document, sorted into current contracts and history | `docs/README.md` |
| the exact note format | `docs/note_format_contract_v1.md` |
| the governed answer contract in detail | `docs/agent_memory_contract_v1.md` |
| to connect an MCP harness to an instance | `docs/mcp_server_v1.md`, "Wiring a harness" |
| what each warning or refusal asks you to do | `docs/next_steps_v1.md` |
| the API contract | `docs/api_spec_v1.md` |
| converters, figures, equations, images, books | `docs/document_conversion_v1.md`, `docs/equations_v1.md`, `docs/images_v1.md`, `docs/ingestion_spec_v1.md` |
| folders, versions and estimates | `docs/folder_intake_v1.md` |
| renaming anything | `docs/renames_v1.md` |
| the owner's work, ideas, tails | `docs/owner_work_v1.md`, `docs/ideas_v1.md`, `docs/instance_tails_v1.md` |
| how the code is organised | `architecture.md`, `docs/implementation_overview.md` |
| the graph's domain model | `schema/ontology_v1.md`, `schema/graph_schema_v1.md` |
| how any piece got the way it is | `IMPLEMENTATION_JOURNEY.md` |
| to change the platform itself | `project_guidelines.md`, then `CLAUDE.md` |

If you are handing an instance to an agent to run, the skills under `skills/`
are the thing to install (the install step does it for Claude Code; for
OpenClaw they are copied by hand): onboarding (once), operations (every
interaction), research answering (questions) and intake (sources of any kind).
They carry the checks and the failure modes in the form an agent will actually
follow.

## 26. A checklist for a new instance

1. Run `python scripts/setup.py` and read what it says is missing.
2. In your AI program, say "help me start with Kovium"; open the example
   and follow its tour, so you have seen a working instance.
3. Make your own collection from the conversation (`create_collection`; over
   HTTP `POST /collections`). You are asked where it goes: the folder your
   AI program works in, a folder you name in full, or where the install keeps
   collections. A workspace registered by
   a script (`POST /ops/workspaces`) gets no access policy, nightly tidy or
   approval setting until you turn them on.
4. Ask for `check_setup`: confirm the AI program that does the work is ready
   (`resolved_backend` is a real backend, not `deterministic`), that
   `/ops/billing_safety` reports `safe: true`, and see what is off and what it
   costs: the better PDF reader, the reader for Word files and web pages,
   reading scanned pages, search by meaning (a local model server), reading
   pictures.
5. Say what it holds: the kinds of note you keep (`note_types.json`), where
   the originals live (named folders, through `change_setting`), and the
   words your documents use for a contents page or numbers, if they are in a
   language the install does not know.
6. Add one real document. Then read the note it produced, properly.
7. Ask a question and follow one citation back to its line in the source.
8. Answer the budget question when the first plan with a cost asks it, then
   add the rest: a few first, checking as you go. Register your own notes
   rather than processing them.
9. Decide the nightly AI limit (`change_setting`), or leave it at zero.
10. Decide who else may do what: `connect_agent` for another AI program,
    `set_access` for what it may do. Ask for a caller agent and an answer
    style if you want one.
11. Take a backup, then rehearse the restore.
12. Once the collection has a shape, draft a tail, show it to the owner, measure
    it, and only then switch it on.

At that point you know both what it does and how to tell when it is not doing
it.
