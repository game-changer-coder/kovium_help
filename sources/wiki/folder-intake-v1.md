---
type: "documentation"
title: "Folder intake v1"
description: "POST /sources/process_folder feeds a whole folder of sources into a workspace: triage first, with no agent call, then processing in date order under a token budget. Implemented in second_brain/source_pipeline/folder_intake.py."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing Kovium"
doc_path: "docs/folder_intake_v1.md"
doc_sha256: "9779676d720d7d8a5d0d1a27a39a9b6ca5c125dbb4867e75416d26bc26ee4332"
---

# Folder intake v1

`POST /sources/process_folder` feeds a whole folder of sources into a workspace:
triage first, with no agent call, then processing in date order under a token
budget. Implemented in `second_brain/source_pipeline/folder_intake.py`.

## Why

Standing up the first real instance meant finding by hand everything a folder of
real files holds besides ordinary documents: the same file under two names, the
same work downloaded twice from different places, a dissertation, a slide deck,
and a 636-page PDF whose pages were images. Each one either wasted an extraction
or failed in a way that took time to diagnose, and nothing said in advance what
a folder would cost.

## The request

```json
{"path": "library:papers", "kind": "paper", "dry_run": true,
 "token_budget": 2000000, "recursive": false}
```

| field | default | meaning |
|---|---|---|
| `path` | — | a folder, or a declared source root (`library:papers`) |
| `kind` / `template_id` | — | what kind of document the folder's files are, as for `/sources/process` |
| `dry_run` | `true` | return the plan and its estimated cost; no agent call |
| `token_budget` | none | stop before a document whose estimated cost would pass it |
| `recursive` | `false` | include subfolders |
| `reprocess` | `false` | process files the workspace already holds |
| `include_near_duplicates` | `false` | take in a file that is another version of a known work, and link it to that work with `version_of` (Track AJ); without it such a file is held back |
| `force_book_splits` | `false` | accept a book split the detector was not confident about |
| `auto_ingest` | `true` | as for `/sources/process` |
| `wait` | `false` | a real run is backgrounded in the long-job lane unless asked otherwise; a dry run never is |

A real run is logged to the governance audit. A dry run reads files only.

The response's `folder` is spelled the way a note spells a source: `root:relative`
under a declared source root, relative to the workspace or the checkout, else the
folder's name alone. It never echoes an absolute path the caller sent, which
would name the machine and whose folder it is, and it is written with forward
slashes on every operating system.

## Triage

Every file is hashed and read for its shape, with no agent:

| action | when | |
|---|---|---|
| `process` | an ordinary document, or an image (`image: true`, `estimated_tokens: 0`; Track T) | processed with its kind |
| `embedded` | an image a document of the same folder shows (`![…](…)`, `<img>`, `![[…]]`) | taken in with that document, not on its own |
| `book` | more than `long_document_pages` pages, or more than `long_document_chars` characters | processed through the book route, chapter by chapter |
| `duplicate` | the same bytes as another file in the folder | the shortest name is kept: a copy usually gains something, and the rule does not depend on any language's word for "copy" |
| `already_ingested` | the same bytes as a source a note in the workspace records (its `sha256`) | skipped unless `reprocess` |
| `near_duplicate` | another version of a note the workspace holds or of a file planned earlier, on the evidence below | held back with a reason naming the other version and the evidence, unless `include_near_duplicates` (then `process`, with `version_of`) |
| `slides` | a slide format, or a PDF of at least `slides_min_pages` pages whose median page holds under `slides_max_median_chars_per_page` characters | reported; no note type reads slides |
| `no_text_layer` | a PDF whose median page, or at least `max_near_empty_page_share` of whose pages, hold under `min_text_chars_per_page` characters; or whose text is under `min_letter_share` letters | refused: the pages are images, or the text layer is encoding debris; run OCR first. Track AG5: when the collection reads scanned pages itself (`ocr` is `auto` or true, docling and a text-recognition program are installed) the file is `process` (or `book`) with `scan: true` instead, read with text recognition |
| `not_recordable` | under `reference` retention, a file outside the workspace and every source root | refused before it costs anything, as the pipeline would refuse it |
| `unsupported`, `empty`, `unreadable` | a format intake does not convert; an empty file; a file that cannot be read | reported |

PDF pages and their text come from `pdftotext`, whose page breaks give a page
count and the text per page in one pass. Without it, pages are counted from the
file's own page objects and the plan marks the figures `chars_estimated`.

### Versions of one work (Track AJ)

A file is compared with what is already known: first the workspace's own notes,
then every file planned before it, including files the workspace already holds
(registered before anything is planned, since their other version may sort
first). Implemented in `second_brain/source_pipeline/versions.py`.

**The signals are each note type's own** (`identity_fields` and `work_fields`
in `notes/contract.py` and `config/note_types.json`, see
[[note-format-contract-v1|note_format_contract_v1.md]]):

- **an identifier the document states about itself** (by default a DOI, an
  arXiv-style archive id or an ISBN: `identifier_patterns`, a name → pattern
  map a collection extends with its own): in a field the type declares as an
  `identifier` (`identifier_fields` too); in the file's name
  (`file_name_identifier_patterns`, by default an archive id saved as
  `2402.16904v2.pdf`; used for nothing else); on a PDF's first page, in its
  header or footer (the first twelve and last six sentences, at most three
  words before the identifier: `arXiv:2402.16904v1 [cs.LG] …`, `DOI: …`); in a
  text's title block (its title and the lines before the first running text),
  on a line that is the identifier and at most one label word. Never in a
  table row, a list item, a quotation, a reference entry, a Markdown link or a
  sentence: an identifier a document only **cites** ("see arXiv:…", "ported
  from arXiv:…", a table of sources) never makes it a version of the work it
  names (fix round, R1: a folder index and a baseline's notes, both citing one
  paper, were linked as versions of each other). A workspace note's come from
  its frontmatter and from its converted source read the same way (a PDF's
  copy as a first page).
- **the people**: the values of the fields the type declares as a `person` or
  an `organisation` (a paper's authors, a contract's parties, a meeting's
  attendees), found in the other's fields or, two words or more, in its opening;
  for a document whose type names no person, the names of its **author block**,
  read by shape from the start of each line before the first running text:
  names of two or three capitalised words, separated by commas, semicolons,
  ampersands or one lower-case joining word, the reading stopping where
  something that is not a name begins (an acronym, a number, a single word, an
  e-mail address, a link). So a line that runs the authors into their
  affiliation and e-mail (`Ada Quell and Bo Varga Kiln Institute, Town, Land
  E-mail: …`) gives its names (fix round 2): the name after the joining word is
  the last, its first two words when the affiliation runs into it; a mark after
  a name (`Quell1`, `Varga*`) ends it, and once names are marked an unmarked run
  is the affiliation. Not read as a person (R4): a line with a second joining
  word or opening with one (a venue, a title in title case), a word the
  document writes in lower case elsewhere (`Conference`), a lone three-word
  line (a group). The block is read as a whole, up to the first heading after
  the title (chores after Track AM: a paper setting each author on a line of
  their own, after the first one's affiliation, gave its first author only):
  an e-mail or link line and a line opening with an affiliation mark or a
  number (`1 Kiln Institute`) are skipped; a line opening with a joining word
  (`and Bo Varga`) continues the list when a line before it gave names; names
  followed by a one-word place with no joining word and no marks (`Kiln
  Institute, Kiln Town, Land`) are an affiliation or an address, skipped; a
  lone three-word name with an initial (`Bo C. Varga`) is a person; and a line
  holding an e-mail address is left out of the document's lower-case words,
  since a grouped address (`{ada.quell, bo.varga}@…`) spells the names in lower
  case. When the block gives fewer than two names (a two-column first page
  can set an author after the abstract: C-R3), the names of the document's
  own reference line are read instead, a line of the shape `<names> . <year>
  . <title> …` (an ACM-style "Reference format"), only when its title is the
  document's own (its title, or its first page's opening lines) and every
  part of its names is a name of two to four words; a citation of another
  work never counts. Names read from documents are compared with the other
  document's names, never looked for in its text; two readings are one
  person when the shorter's words, two at least, are all in the longer (a
  block keeps a long name's first two words when its affiliation runs into
  it, a reference line keeps all of them). An organisation
  counts only when no more than `version_boilerplate_works` known works carry
  it, and is stated as "the same organisation", never as people.
- **the title**: the file's own (a PDF's `/Title`, a text file's `title:` or
  first heading) or a first page carrying a known title (an opening line alone
  or joined with the next, as titles wrap). Equal, or nearly identical: at
  least `version_title_similarity` (0.8) alike word by word, differing in
  words with no digit (`Report 2021` and `Report 2022` are two works).
- **the opening passage**: the first `version_opening_words` (600) words,
  compared in runs of five; at least `version_min_shared_words` (40) in common.
  A run found in the openings of more than `version_boilerplate_works` (2)
  known works (a licence, a venue's first-page text, a template's preamble) is
  boilerplate and never counts.
- **the work fields**: fields whose different values make two works, whatever
  else they share (a meeting's `meeting_date`, a chapter's `chapter_number`, a
  decision's `decided_on`). A paper declares none: a preprint and its published
  form differ in their year.

**A version only on strong evidence**, stated on the row (`version_of`:
`of`, `note` or `file`, `evidence`, `link`):

- an identifier in common, carried by no other known work (an identifier
  several notes carry, a book's ISBN on each of its chapters, names a whole,
  not one of them; a work whose work field differs does not count);
- the same title, with the same people or the same opening passage;
- a nearly identical title, with the same opening passage;
- the people cannot be read on one side or both (unknown is not different,
  fix round 2) and the openings are nearly identical: at least
  `version_identical_opening` (0.8) of the shorter one's three-word runs in the
  other; only for a type whose `versions_by_opening` is true (the note type's
  declaration; false for the shipped types whose documents are commonly made
  from one template: meeting notes, web pages, decision records, AI
  conversations, video transcripts), and only when the titles are at least
  alike (half the shorter title's words in the other) or one is missing
  (chores after Track AM: two contracts or two forms from one template share
  their opening, and differ in what they are called). A pair this rule turns
  away is a near miss with its reason;
- two or more of the same people (`version_min_shared_people`) with a
  strongly similar opening, whatever the titles (R2: a preprint, its conference
  paper and its journal paper, each retitled and its abstract reworded). The
  opening is the first `version_similar_opening_chars` (1,500) characters of
  running text after the title block, compared by three-word runs: the share of
  the shorter one's runs found in the other must reach `version_similar_opening`
  (0.5). Measured on the tests' invented abstracts: one word in eight or nine
  changed, a sentence moved and one added keeps 61 to 67 % of the runs; one
  word in six, 50 %; one in five, 40 %; a different abstract sharing one
  sentence of background, 5 %. The default sits well above what different
  abstracts share and asks for an abstract still mostly the same: a false link
  is worse than a missed one. It is a setting of the install, measured on
  invented text, not yet on a real lineage.

An opening that reaches `version_similar_opening` without making a version
is always a near miss (`similar_to`), its `why_not` saying which: none of the
same people (people read on both sides and none shared are two works,
whatever the overlap), too few of them, or people unknown and the opening
under the strict threshold. Runs found in the openings of more than
`version_boilerplate_works` known works never count. And never when a work
field both carry differs. Anything less (a title alone,
a similar title by the same people with no passage in common, which is often
another work by the same people) is a **near miss**: the file is taken in as
its own work and the row says so (`similar_to`, with `why_not`). A type that
declares no fields has the generic signals: the title and the opening.

**A version of a set of versions (Track AW5).** Works found to be versions of
one another (a preprint, its conference paper and its journal paper, planned
in one batch or held as notes that say `version_of`) are one set. An identifier
they all carry then names that set, not a whole: a further version carrying it
is linked to the set (to its earliest work) instead of being reported as a near
miss of one of them. Identifiers carried by works that are not known to be
versions of each other (a book's chapters) stay ambiguous, and a near miss
alone never links, however many works of a set it resembles.

Words are compared after Unicode normalisation (NFKC: a ligature `ﬁ` a page
kept is `fi`; a decomposed accent is the composed one) and case folding, and a
script written without spaces (Chinese, Japanese) one character at a time, so
a title in any language is compared in its own script. A page's first line is
never itself taken as a title: it is often a running header many documents
share. A tie between two copies of one work keeps the shorter name, as for
byte duplicates.

**What becomes of a version**: held back on this route (`near_duplicate`)
unless `include_near_duplicates`, which takes it in and, right after its note
is made, links that note to its other version's with `version_of`, the
relation declared in `config/relations.json` (the run's outcome says so:
`version_of`, `linked`). `POST /sources/add` takes versions in and links them
by default ([[adding-documents-v1|adding_documents_v1.md]]). The low-level
route keeps its default because it is the step-by-step path, whose caller
reads the plan; whichever route takes a version in, it is linked the same way.

**A note's converted source in every spelling** (AJ1, 2026-10-22): a note
records its converted copy relative to the workspace, relative to the
checkout (a workspace under `sandboxes/`) or by bare file name (a workspace
outside the checkout). Only the first was tried before, so the opening of a
real note's source, and the identifiers on it, were never read.

## Order

Documents are processed oldest first, because each is read against the concepts
the ones before it established. The year a document carries for itself is
taken, strongest evidence first, from its frontmatter (`year`, `date`,
`meeting_date` and the rest), an identifier that states a date (an arXiv-style
`YYMM.NNNNN` id), a year standing alone in its file name, a `©` line on its first
pages, and last the PDF's creation date. Each row says which (`year_basis`). A
year in a file name must stand alone: `2001.05747v2` and `2480362.2480670` are
identifiers, so a digit run joined by `.` or `/` to more digits is not a year
(`smith2019energy` is). Of byte-identical copies, the kept one takes the
strongest date evidence any copy has. A document with none is `undated` and goes
last: when a file entered a folder is not when it was written.

## Cost

Every document planned for processing carries `estimated_tokens`: a fixed
part per document plus a part per character of the document,
`tokens_per_document + tokens_per_char × characters`. A document costs a fixed
amount whatever its length (the prompts, the shaping passes, the agent's own
context): measured on one install, about 34k tokens plus about 0.3 a character,
so a 5,000-character document costs most of what a 30,000-character one does.

Until this install has `min_history_runs` (1) past `process_source` runs in
its call log (`logs/agent_calls.jsonl`), both parts are stated defaults. After,
they are measured: each run's billed tokens against the length of the document
it read (`document_chars`, recorded on every call of the run). With runs of
clearly different sizes (the largest at least 1.5 times the smallest), both
parts are fitted from them; with one run, or runs of about one size, the
default rate per character is kept and the fixed part is measured. A run
logged before `document_chars` existed is sized by what its extraction calls
read, and the basis says how many such runs there were. The concept-layer
calls an ingest causes are left out: they do not depend on the document's
length.

**A short document's first guess (2026-10-25).** The two stated fixed parts
(`tokens_per_document`, 30,000, and the concept upkeep default
`concept_sync_tokens_per_document`, 40,000) describe a paper of tens of
thousands of characters, and put a five-line note at about 70 thousand tokens.
While nothing is measured, a document shorter than
`estimate_small_document_chars` (20,000) has each fixed part scaled in a
straight line from a floor at no length to the full part at that length: the
extraction floor `estimate_small_document_tokens` (23,000: the example
collection's four measured readings of 1,400 to 2,300 characters billed 21,929
to 26,986 tokens, fixed parts of about 21,000 to 26,000), the upkeep floor
`estimate_small_document_upkeep_tokens` (12,000: about one agent call, since a
short reading's two passes billed 23,310, and a short document's upkeep is one
`maintain_concepts` call; inferred, not measured). A 300-character note is now
about 36 thousand (23,255 reading, 12,420 upkeep), a 10,000-character one about
58 thousand, and from 20,000 characters nothing changes. The default range
stays wide, its basis says the figure was made smaller for a short document,
and the plan still calls it a first guess. Measured figures, once this install
has runs or measured upkeep, are never scaled. `0` turns the scaling off; a
fixed part set below its floor is kept as set.

### Estimates that learn from this install (Track AM)

Found on a real collection: short papers ran at about 0.6 times their
estimate, one paper at 1.7 times and a thesis at 2 times. One fit over all
runs cannot be right for every kind and length, so an estimate is a **range
with the basis it stands on**, read from the same call log. Nothing in it is
fixed to a kind of document; the types and the length bands are whatever this
install's own history holds.

- **The group.** A run records the note type it used (`template_id`, the
  template) beside `document_chars`. A document is estimated from, in order:
  the runs of its **note type and length band** (when the type has at least
  `estimate_min_group_runs` runs per band, cut into up to `estimate_max_bands`
  bands of equal counts from the sizes this install holds, never at fixed
  lengths); else the runs of its **note type** (at least
  `estimate_min_group_runs`); else **all runs**; else the **stated default**.
  Within a band the cost per character comes from the type's whole history
  (a narrow band cannot tell it) and the fixed part from the band.
- **The range.** What those runs actually reached against their own fit: the
  10th and 90th percentiles of billed over fitted from five runs, the
  extremes from three or four. With one or two runs, the stated
  `estimate_sparse_spread` (`[0.7, 1.5]`); with none, `estimate_default_spread`
  (`[0.5, 2.0]`), and the basis says it is a default and wide.
- **Left out.** A run with a failed call, a run that reused an earlier answer
  (it billed far less than a fresh one), a run whose usage was not fully
  reported, and, from `estimate_outlier_min_runs` (6) runs, one that billed
  more than `estimate_outlier_factor` (3) times or less than a third of its
  fitted figure. `estimate_basis` counts them ("left out: 1 failed, ...").
- **A synced log is read tolerantly.** NUL bytes and a byte order mark before
  a line (a sync artefact) are stripped, not the call lost; every reader of
  `logs/agent_calls.jsonl` uses the same reader. A line that still cannot be
  read is counted ("N lines of the call log could not be read" in
  `estimate_basis`), and a run it can be traced to (its `run_id` is still
  legible) is left out as "not fully recorded" rather than kept with its total
  understated.
- **Each row is estimated with the kind triage decided for it** (`kind`, the
  template id), so a plan of mixed files uses each type's and band's runs once
  runs log `template_id`. Runs logged before that have no type and serve only
  the "all runs" group.
- **Concept upkeep** has its own range (`plan.concept_upkeep` and each row's
  `estimated_range.concept`): the mean of this install's measured upkeep per
  source and its spread, and per note type once the type has enough sources,
  since a book's chapters did not cost what a short paper did. A retried
  concept call (Track AN) is billed and logged like any call, so the measured
  mean already holds retries; with no history the default does not.
- **Each row** has `estimated_tokens` (the planning figure, the expected one)
  and `estimated_range`: `{low, expected, high, basis, group, concept}`.
  `basis` is plain ("5 earlier runs of this kind of note, of 8,000 to
  12,000 characters"); `group` is `band`, `type`, `all` or `default`. The plan
  adds `estimated_range` (the rows summed) and `estimate_groups` (how many
  documents each kind of basis served). Summing the high ends of many
  documents is the cautious bound: errors in different documents partly
  cancel.
- **Which end is the planning figure: the expected one.** A run that reaches
  its budget stops cleanly before the next document and says what is left,
  so being a little low costs a resumed run; a figure at the high end would
  ask every addition a question. What the high end does is set the budget a
  question proposes (the larger of the estimate with headroom and the high
  end), so one heavy document does not stop the run it was approved for, and
  a plan whose standing budget lies between the two says so in its summary
  (`may_pass_limit`).
- **A book row** is the extraction of its chapters (estimated from the
  `book-chapter` runs, per chapter), their concept upkeep, and
  `estimated_split_tokens` (Track AL4): the chapter-finding agent's look, a
  **ceiling**. It is in `estimated_tokens` and the high end, and not in the
  low end, which assumes the free steps find the chapters. The book route's
  dry run states all of this chapter by chapter
  ([[ingestion-spec-v1|ingestion_spec_v1.md]]).
- **Stated apart, never added in:** Track AS's checking of flagged figures
  against their source spends from a share of the standing budget in its own
  pass; `also_possible` on an addition plan says how much, when a standing
  budget is set.

Before 2026-10-14 a measured estimate was a single rate, billed tokens per
character *sent*: it dropped the fixed part and divided by every pass's
characters, about one and a half times the document, so it put a short
document at a tenth of its cost.

When the concept layer reads each new document with an agent, each document's
estimate includes its concept upkeep (`estimated_concept_tokens`, Track X):
once `min_history_runs` sources have concept calls in the log that record the
sources they were made for, the mean of this install's measured upkeep per
source; until then `concept_sync_tokens_per_document`. `plan.concept_upkeep`
states the rate, its `basis` and which pass pays it. The plan says which basis
the extraction estimate used (`estimate_basis`). It is an estimate, not a
quote.

With a `token_budget`, a dry run lists `would_fit_budget`, and a real run stops
cleanly **before** a document whose estimate would pass the budget, reporting
`stopped_at_budget` with what was spent and what was not started. The budget
keeps each processed document's concept share in reserve for the concept pass
at the end. A document already started always finishes, so the real total can
pass the budget by at most one document's overrun.

## Running

Documents are processed one at a time, in plan order. A document that fails is
recorded with its error and the run goes on. Each outcome reports the note
written, its billed tokens (extraction, and any chapters or sections) and its
estimate, so estimates can be checked against reality. Running the same folder
again plans nothing new to process: those files are now `already_ingested`.

**One concept pass** (Track X, `docs/concept_notes_v1.md`). Every document's
concept upkeep is deferred, and one pass after the last document brings the
concept notes up to date over every note the run wrote, in date order:
`run.concept_pass`, whose cost is part of `run.billed_tokens`. It runs whether
or not the run stopped at its budget, so what was taken in is up to date. The
pass is told what is left of the run's budget: a concept call that failed is
retried once inside it only if that fits (Track AN1), and concepts still written
without the agent are listed in `run.concept_pass.left_for_agent`.
`concepts: defer` on the request leaves the pass for `POST /concepts/sync` or
the nightly runner; answers warn until it runs.

## A kind per file (Track U4, 2026-10-11)

One folder may hold meeting notes, a chat export, a decision record, a saved
web page and a photo. With `kind_per_file` (the default) each file's kind is
decided on its own, without an agent (`source_pipeline/file_kinds.py`), and
the request's `kind` or `template_id` is the default:

1. `frontmatter`: a text file's `type`, `kind` or `template` names a
   registered kind (by template id, note type, title or alias);
2. `file_type`: `kind_by_suffix`, used only when that kind is registered (the
   shipped rule maps image files to `image`, the kind Track T added, so a
   photo in a folder of documents is an image source, `image: true`, costing
   no extraction);
3. `content`: the file's headings (Markdown, text, the `<h1>`–`<h3>` of HTML),
   frontmatter keys and identifiers (DOI, arXiv, ISBN) scored against each
   kind's declared sections, frontmatter and own identifying fields (one point
   per heading sharing a word, per field, two per identifier the kind is known
   by); chosen when it scores 2 or more and leads both the next kind and the
   default, and never on identifiers alone (chores after Track AM, C-R5: a
   conference paper asked for as a paper was made a book chapter by its
   proceedings' ISBN; an identifier is carried by many kinds, so it counts
   only beside a heading or a field the kind declares);
4. `shape` (Track AE): a file at least 60% of whose text lines are entries (a
   term and its definition, in runs of three or more, five entries or more)
   is given the kind declaring `file_shape: entries` (`reference-list`),
   `kind_evidence` `shape:entries`, `entries:N`, `share_of_lines:S`; such a
   kind is not scored by headings in step 3 (`docs/reference_lists_v1.md`);
5. otherwise `folder_default`.

A row whose kind is built without an agent (`image`, `reference-list`) has
`estimated_tokens: 0` and `estimated_concept_tokens: 0`.

Each plan row carries `kind`, `kind_basis` and `kind_evidence`; the plan adds
`kinds`. The table is the template registry and the note contract, so a kind
registered later is decided for with no change here.

## Configuration

`<workspace>/config/intake.json`, only the keys that differ:

| key | default |
|---|---|
| `min_text_chars_per_page` | `25` |
| `max_near_empty_page_share` / `min_letter_share` | `0.5` / `0.4` |
| `identifier_patterns` | DOI, arXiv-style id, ISBN (name → regular expression, first group the id) |
| `identifier_fields` | `doi`, `arxiv`, `isbn`, `identifier`, `identifiers` |
| `slides_min_pages` / `slides_max_median_chars_per_page` | `5` / `600` |
| `long_document_pages` / `long_document_chars` | `100` / `400000` |
| `tokens_per_document` / `tokens_per_char` | `30000` / `0.5` |
| `concept_sync_tokens_per_document` | `40000` |
| `estimate_small_document_chars` / `estimate_small_document_tokens` / `estimate_small_document_upkeep_tokens` | `20000` / `23000` / `12000` (with nothing measured, a shorter document's fixed parts scale down to these floors; `0` chars turns it off) |
| `estimate_min_group_runs` / `estimate_max_bands` | `3` / `3` (runs a note type, and each length band of it, needs before it stands in for all runs / the most bands a type is cut into) |
| `estimate_default_spread` / `estimate_sparse_spread` | `[0.5, 2.0]` / `[0.7, 1.5]` (low and high as multiples of the expected figure, stated, for a default and for one or two runs) |
| `estimate_outlier_min_runs` / `estimate_outlier_factor` | `6` / `3.0` (from this many runs, one this many times above or below its fitted figure is left out) |
| `estimate_chars_per_page` | `3000` (a book known by pages only) |
| `min_history_runs` | `3` |
| `kind_by_suffix` | `{}` (a file type → a kind, over the shipped rule mapping image files to the `image` kind) |
| `file_name_identifier_patterns` | an archive id (name → regular expression over the file name, for versions only) |
| `version_opening_words` / `version_min_shared_words` | `600` / `40` |
| `version_boilerplate_works` / `version_title_similarity` | `2` / `0.8` |
| `version_min_shared_people` / `version_similar_opening_chars` / `version_similar_opening` | `2` / `1500` / `0.5` |
| `version_identical_opening` | `0.8` (enough alone when the people cannot be read on a side) |

## Limitations

- A version with no identifier and no title in common is not recognised: the
  opening passage and the people only ever corroborate a title. A version
  whose opening was rewritten, by other people, under a new title, is two works
  here.
- A PDF's people are found only on its first page: names of two or three
  words in a cased script, with no lower-case particle (`van der`), not printed
  in capitals, on lines the text extractor kept apart from the affiliations; a
  lone two-word line (a group, a place) can still be read as a name. A format
  read only after conversion (Word, HTML) has no opening text to compare before
  it is converted.
- A concept page is never another version: only source notes at the top of
  the wiki are compared, never a page whose type is a concept or its redirect.
- An identifier printed on every part of a whole saved apart (a book's ISBN
  on each chapter file) links the parts as versions unless the type declares a
  work field that tells them apart and the files carry it.
- The text-layer check is by counts and by the share of letters; a text layer
  of real-looking but wrong letters (a font mapped to the wrong glyphs) passes.
- Year detection reads metadata, the file name and a `©` line; a document dated
  only in prose is `undated`.
- The estimate scales with length only. A dense document and a sparse one of
  equal length are estimated alike.
- Formats other than PDF and plain text are sized from their byte count until
  converted.
- A per-file kind is chosen among the registered templates; an install's own
  note types (declared in `config/note_types.json`) are not templates and are
  never chosen. A PDF or DOCX is judged only by its identifiers.
