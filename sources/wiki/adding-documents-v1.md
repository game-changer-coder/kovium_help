---
type: "documentation"
title: "Adding documents in one request (v1)"
description: "Track AR. The contract for POST /sources/add, the standing token budget (GET and POST /ops/budget), the one question a user is asked about it, and the short report. Lower-level intake (/sources/process, /sources/process_book..."
status: "stable"
reading_status: "read"
product: "second brain"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing second brain"
doc_path: "docs/adding_documents_v1.md"
doc_sha256: "e292fc723c29d900c0eb50c6c26861316851998767c07c64d427cd6fdfe822b8"
---

# Adding documents in one request (v1)

Track AR. The contract for `POST /sources/add`, the standing token budget
(`GET` and `POST /ops/budget`), the one question a user is asked about it, and
the short report. Lower-level intake (`/sources/process`, `/sources/process_book`,
`/sources/process_folder`, `POST /sources/process_videos`) is described in
[[folder-intake-v1|folder_intake_v1.md]] and
[[ingestion-spec-v1|ingestion_spec_v1.md]].

Track AG3 (first-run plan, D4, D5) adds, on top of the same request: what the
person's own files need, read from their structure; the plan put as plain
lines and questions in order; the doors over MCP (`plan_adding`,
`start_adding`, `adding_status`); `GET /sources/add/status`; the caps of the
agent's access policy, enforced at approval; and the approval enforced on
every route that spends ("The approval on every spending route", below).

## The rule

The caller agent and the knowledge brain system settle the work between
themselves. Whatever the system recommends, the caller carries out; whatever
failed is tried again by the system before anyone is told. A user is asked
about two things only: the collection's standing token budget, set once and
raised only by them, and their own originals: deleting or overwriting one (no
route of this version does either) and, since Track AZ4, whether a copy of
each original from outside the collection is kept in it or the original is
pointed to where it lies (asked once, below). Every other code is settled by
the system or the caller ([[next-steps-v1|next_steps_v1.md]],
`settled_by`).

## The flow

1. **Plan** (free, no agent call): `POST /sources/add` with `paths`, `urls`,
   or both. `dry_run` is the default.
2. **Within the budget**, the plan asks nothing about it, and `next` is the run
   call with its `approval`. The caller makes it.
3. **Beyond it**, the budget question is the last of the plan's `asks`, put to
   the user word for word, recommended answer first. On yes, the caller makes
   `on.yes`; on no, `on.no` (none, when no budget is set). Unanswered, nothing
   runs. Since Track AW4 the question is given once: the plan's `question`
   field is always null, and a stop at the budget after a run gives its
   question as one of `asks` only.
4. **Run**: `POST /sources/add` with `dry_run: false` and the plan's
   `approval`. It is a background job (`job_id`; follow it with
   `GET /brain_keeper/jobs/{job_id}`) unless `wait: true`.
5. **Report**: the job's result holds `report.lines`, at most six plain lines,
   which the caller relays as they are.

## `POST /sources/add`

| field | meaning |
|---|---|
| `paths` | files and folders, relative to the workspace or `root:path` for a declared source root. A folder is read at any depth; files starting with `.` and files in hidden folders are left out |
| `urls` | links: a web page, a video or a playlist |
| `kind` | the kind a file gets unless its own look says otherwise. Without it: the install's `default_kind` (`config/intake.json`), else the kind the collection already holds most notes of (pieces of a work and kinds built without an agent are never a default). With none of these, the plan is not ready and says `adding_kind_unknown`: the caller sends `kind` |
| `agent_id`, `workspace_id` | as on every route |
| `dry_run` | `true` (the default): the plan. `false`: the run, which needs `approval` |
| `approval` | the code a plan gave: `<issued>.<budget>.<digest>`. It binds the files (their bytes), their actions and kinds, the request and the budget, and is valid for `adding_approval_valid_hours` |
| `token_budget` | a one-time budget the user said yes to (a plan's question, or "add the rest" after a run stopped), used in place of the standing one for this plan |
| `versions` | what becomes of another version of a work already known (Track AJ): `link` (the default) takes it in and links the two; `separate` takes it in unlinked; `skip` holds it back. A request option the caller sends, never a question; it is part of what the approval binds |
| `originals` | Track AZ4: `keep` (a copy of each original from outside the collection goes into its folder for originals, `sources/raw/`) or `point` (each stays where it is and is recorded through a named folder); the person's answer to the plan's question, or what they said before ("keep copies"). It becomes the collection's way (`sources.json` `source_retention`, written at the run through the settings path) and is part of what the approval binds |
| `wait` | run in the request rather than as a background job |
| `keep_as_limit` | Track AG3: the person's yes to the budget question also keeps the approved budget as the standing one (what `POST /ops/budget` sets), written and audited in the same call; only with a valid `approval` |
| `access_mode` | `manage` when the call comes from the adding tools over MCP: the agent must hold a manage entry that may `add` ([[collections-v1|collections_v1.md]]) |

Refusals: `adding_nothing_named` (422), `adding_approval_required` (422: no
approval, or one no plan gave), `adding_plan_changed` (409: a file, a kind,
the request or the budget changed since the plan), `adding_approval_expired`
(409), `request_refused` (an unknown `kind`, or a `versions` other than the
three). Each carries its fix, settled by
the caller: plan again.

### The plan

| key | meaning |
|---|---|
| `summary` | one or two plain sentences: what will be added, what it costs, what is skipped and why |
| `question` | always null since Track AW4: the budget question is the last of `asks` (given once) |
| `ready` | whether it can run: something to add, and every file has a kind |
| `approval`, `approval_within_limit` | the run's approval; the second runs within the standing budget when the user says no to more |
| `budget`, `standing_budget` | what the run may spend; the collection's standing budget per addition |
| `estimated_tokens`, `estimated_minutes`, `estimate_basis` | the cost (the planning figure, the expected one), the conversion time of long documents where it can be estimated, and the basis (this install's measured rate or the stated default) |
| `estimated_range`, `estimate_groups`, `planning_figure`, `may_pass_limit`, `also_possible` | Track AM: `{low, expected, high}` summed over the rows; how many documents were estimated from a length band, a note type, all runs or the default; `expected`; whether the standing (or one-time) budget lies between the expected and the high end; and costs a run may add that the estimate leaves out, each `{what, up_to_tokens, text}` (checking flagged figures against their source, up to its share of the standing budget) |
| `default_kind` | `{kind, basis}`: `request`, `setting`, `collection`, or null |
| `next` | the run call, when there is no budget question |
| `files`, `counts` | one row per file or link, as a folder plan's rows, with `action`, `kind`, `kind_basis`, `estimated_tokens` (a book's includes `estimated_split_tokens`, the agent's look at its parts at most); a version also carries `version_of` (`of`, `note` or `file`, `evidence`, `link`), and a near miss `similar_to` (with `why_not`) |
| `not_found`, `refused` | names that are no file or folder, and names or links that may not be read |
| `warnings` | `adding_budget_needed`, `adding_kind_unknown`, `adding_path_not_found`, `adding_nothing_to_add` |
| `say` | Track AG3: the plan in plain lines a person is shown: the summary, each need that is only said, and this agent's cap when it limits the run (`adding.agent_limit`) |
| `needs` | Track AG3, D5: what the files need that is not there yet, each `{signal, option, name, files, files_total, said, lines, later, asked}` (below) |
| `originals` | Track AZ4: `{way, outside, counts, folders_to_name, asked}`: `keep`, `point` or null (not decided), how many files come from outside the collection, how many are `copied`, `pointed` or already `in_collection`, the folders the run names, and whether the question is asked. Each row of `files` says its own `original`, and a pointed one its `named_folder` |
| `asks` | the questions, in the order they are put, one at a time, recommended answer first: the originals question (`about: originals`, below), each need that cannot wait (`about: need`), then the budget question (`about: budget`). Each is `{about, text, recommended, answers, on}`; on yes the caller takes `on.yes`, on no the next question. A need's `on.yes` is `{install, switch_on, then}`: how the agent gets the program (for the agent, never shown), the choice to switch on (`change_setting`), and the plan to make again. The budget's `on.yes` is the run with `approval` (and `keep_as_limit` when no budget was set); its `on.no`, when there is one, the run with `approval_within_limit` |
| `start` | the run, when nothing is left to ask (the same call as `next`, in the call-hint form `{route, body}` every tool result takes) |
| `kinds` | with `adding_kind_unknown`: every kind a file can be given, `{kind, title, what}`, so the caller settles it (AR3) with no other call |
| `agent_limit` | `{tokens}` when the agent's cap (below) is under what the plan would spend: the run stops there cleanly |

**The question.** With no standing budget it says what the addition uses and
asks whether additions may use up to the proposed figure each time without
asking; yes sets the standing budget (`POST /ops/budget`) and runs. With a
standing budget too small for this addition it asks to allow the proposed
figure this once; yes runs with `approval`, no runs with
`approval_within_limit` and stops cleanly at the limit. The proposed figure is
the estimate times `adding_budget_headroom`, rounded up to two significant
digits, or the high end of the range when that is more (Track AM): the
question reads "will probably use about X tokens ... and could use up to Y". A
standing budget between the expected figure and the high end asks nothing, and
the summary says it could pass the limit and what happens then. Each row's
`estimated_range` is `{low, expected, high, basis, group, concept}`; see
[[folder-intake-v1|folder_intake_v1.md]], "Estimates that learn from this
install".

**A first guess, said so; a short note, offered free (Track BA, fix round
R5).** A newcomer's first plan for a five-line file said "about 70 thousand
tokens" from the stated default, with nothing saying it was a guess. Now:

- When every document's estimate stands on the stated default (no measured
  run in this collection, `estimate_groups` all `default`), `say` adds, second:
  "This figure is a first guess, since nothing has been measured in this
  collection yet. The real cost can be much lower or higher."
- A plain text file (`.md`, `.markdown`, `.txt`) of at most
  `adding_short_note_chars` characters (`config/intake.json`, 3000 by default; 0
  offers none) is offered the free way too: `say` adds "1 of these is a short
  note in plain text. You can add it as a note you wrote instead, read by code
  word for word, with no AI and no tokens." and `free_way` lists, for each, `{file,
  title, call}`, the call being `POST /sources/note` (`add_note`) with its title
  (the header's, else its first heading, else its name) and its text (its header
  left out). That route shows its own plan with a `confirm` code first and writes
  only on the person's yes. The paid plan is unchanged beside it.
- Not done here: the default's fixed part per document (`tokens_per_document`,
  30,000, plus the concept upkeep default, 40,000) is what a short file mostly
  pays in the estimate. It stands for the real overhead of a reading (prompts,
  two passes), measured at 23,310 tokens for the example's one short page, so
  scaling it down for small files is a change to the estimator's model (Track
  AM), left for its own decision.

### Keep a copy of each original, or point to it (Track AZ4)

What becomes of an original added from outside the collection is the
collection's choice, the catalogue's "Keeping copies of your originals":

- **keep** (`source_retention: copy`, recommended): a copy goes into the
  collection's folder for originals, so the collection holds everything and
  can be moved or backed up as one folder;
- **point** (`reference`): the original stays where it is. Its folder is
  recorded as a named folder (the catalogue's "Named folders for your
  originals", `sources.json` `source_roots`): one already named is used, and
  a folder not named yet is named at the run after its own name in ASCII
  (`Field Notes 2024` is `field-notes-2024`, `-2` when taken). A note then
  records `field-notes-2024:meadow.md`, never a path naming the user.

A collection made from the conversation starts undecided (`source_retention:
ask`, which every other reader takes as `copy`). Its first plan that brings
files from outside asks once, for the whole batch, first among `asks`:
"2 files are outside the collection. Keep a copy of each in the collection,
or leave them where they are and point to them? Keeping copies is
recommended: the collection then holds everything, so it can be moved or
backed up as one folder." with the answers `keep` and `point`, each the plan
made again with `originals`. While it waits the plan has no `approval`, no
`start` and no budget question (`adding_originals_choice`, `ask_user`,
settled by the user). The run with `originals` writes the answer as the
collection's way through `change_setting`'s path (its preview, then its
apply), names the folders to point to the same way, and only then adds; the
result's `originals` says what it wrote. Later plans ask nothing and say
which way it goes ("2 originals from outside are copied into the
collection's folder for originals."). A collection made before keeps its
behaviour: it is never asked, and copies as it did. Files already inside the
collection ask nothing either.

Decided per collection, never per file: a collection where some notes can be
refreshed and others not, with nothing recording which, is worse than either
uniform answer (`retention.py`). A caller sends `originals` again to change
the way for later additions; files already added keep the way they were
saved. Under `point`, a file outside every named folder is no longer refused
at plan time (`not_recordable`) on this route: its folder is named at the run.

### What the files need (Track AG3, D5)

The plan looks at the files it queued, by their structure only
(`second_brain/source_pipeline/needs.py`), and names each option of the
catalogue they call for that is not already in effect here:

| signal | read from | option |
|---|---|---|
| `pdf_layout` | a PDF whose laid-out first pages hold two columns of text side by side, rows of a table (three fields or more apart, one a number), or display formulas | the better PDF reader, unless it is installed and chosen |
| `equations` | display formulas: a number in brackets set apart at a line's end, mathematical symbols, glyphs the plain reader cannot name | equations in PDFs, unless on |
| `other_formats` | Word files, web pages, e-books | the document reader, unless installed |
| `scans` | files the plan skips for having no text | reading scanned pages, unless on |
| `pictures` | pictures, documents that embed them, PDFs holding image objects | reading pictures, unless on |
| `long_documents` | documents the plan reads part by part | splitting long documents, when off |
| `search_program` | search by meaning not answering on this computer | search by meaning |

Each need is the catalogue's own words: its signal sentence, its four lines
and its `later` line. The better PDF reader counts as there only when it is
installed and chosen for the collection, the same definition `check_setup`
uses (`needs.reader_state`). A skip always says its reason: past three kinds of
reason, the rest follow in further short sentences ("Also ..."), never "for
other reasons". A need whose option cannot wait without reading the
documents again (`later.possible` other than `yes`, and `ask_before_first_run`:
today only the better PDF reader) is a question before the first run (rule 7,
S7); every other need is said in `say`, never asked. What a yes to such a need
does is said as it will be on this computer: "installed first" only where a
line installs it, else "switched on for this collection" (Track AW4). The
advice about scans is given only for a scan the switch would add (a text
recognition engine and the better reader are here); a scan skipped whatever is
switched on has its skip reason and nothing more (Track AW4). Every sentence
of the plan and the report, and the line about concepts left for the AI, is
the catalogue's (`plan.*`, `skip.*`, `run.*`, `report.*`, `left.*` in
`second_brain/catalogue/setup_v1.json`), not written in code. A PDF is looked at with
`pdftotext -layout`; without it, only its image objects are seen. Nothing is
read from a subject, a field or a language.

### What the run does by itself

- **Triage**, as a folder: exact duplicates are processed once, a file already
  in is skipped, a scan with no text, a slide deck and an unreadable format are
  skipped with their reason. Files named together are planned together, so a
  copy in one folder and its original in another are found.
- **Versions of one work** (Track AJ): a file that is another version of a
  work the collection holds, or of a file earlier in the same addition (a
  draft and its final form, a preprint and its published form, two editions),
  is taken in, and its note is linked to the other's with `version_of`, the
  relation `config/relations.json` declares, right after it is made. A link
  is reported `linked` only when its edge reaches the other note in the graph:
  that note is ingested first when its Markdown is in the collection but not
  yet in the graph (free), and a link that would still point at nothing is not
  kept, with its `reason` (fix round, R3). Which
  files are versions, and on what evidence, is in
  [[folder-intake-v1|folder_intake_v1.md]] ("Versions of one work"): strong
  evidence only, stated on the row; a near miss (a title alone, a similar title
  by the same people with no passage in common) is taken in as its own work and
  says why. Nothing is merged, and no one is asked: `versions: skip` holds
  versions back, `versions: separate` takes them in unlinked. Later, the owner
  removes a version through review (a withdrawal of its note, Track AB4), or
  the link alone by deleting the `version_of` line of the note (the Markdown
  is the source of truth; the next ingest drops the edge). The plan's summary
  and the report each say it in one plain sentence.
- **Order**: oldest first, by the date each file carries.
- **Kinds**: each file its own (frontmatter, file type, content, shape), the
  default otherwise. A link to a video is a video; any other link a web page.
  A video with no caption track is taken in from its description and chapters,
  and its note says that is all it holds.
- **Long documents** go to the book route with its conversion time accepted
  (it is part of the plan the budget covers). Their parts are found by the
  chapter-finding ladder (Track AL, [[ingestion-spec-v1|ingestion_spec_v1.md]],
  "The chapter-finding ladder"): the PDF's outline, the contents page in any
  declared language, the headings, and last the brain keeper agent from a
  compact view, within what the run has left of its budget (a book row's
  `estimated_split_tokens`, part of its `estimated_tokens`, is that look at
  most). No one is asked how to split. A book none of the steps splits with
  confidence is read whole, in parts, rather than split on invented
  boundaries (`adding_book_read_whole`); its row says why (`whole_because`)
  and carries the book route's own plain line (`split_line`), and the
  report's first line says it in words ("read whole, as its chapters were
  unclear and no assistant was available to look"). Each part is named
  `<document-key>-<NN>-<title>`, so a document's parts sit together and in
  reading order.
- **Retries** (AR8), each within the budget: a failure the same call would
  not repeat (a crash, a timeout, a video site that did not answer) is tried once more after
  `adding_retry_backoff_seconds`; a PDF that still fails, or converted to no
  text, is tried once with the next PDF reader (docling, then markitdown, then
  the basic extractor; the basic extractor's failure tries markitdown). A
  refusal the same call would repeat (no readable text from a non-PDF, an
  unsupported file, a quarantined source) is not repeated. When the agent is
  on, a note that came out partial (an extractive fallback, a degraded
  extraction, parts not read, coverage below 100 %) is made again once.
  `retries` lists every document tried more than once, with each try.
- **One concept pass** over every note the run wrote, after the last. A concept
  call that failed is tried once more inside the pass, within what the documents
  left of the budget, before the concept falls back (Track AN1). With no AI program
  set up nothing is attempted and no failure is counted. A concept page still written
  without the AI waits for it and is never reported as done: the report has
  one line saying which concepts, why, and what trying again would cost, and the
  run warns `adding_concepts_left_for_agent` (`concept_pass.left_for_agent`;
  `docs/concept_notes_v1.md`, "A pass that finishes what it starts").
- **Passage vectors** that failed during the run are indexed once more at the
  end (`vectors`).
- **The budget**: a document is not started when its estimate would pass what
  is left; the run stops cleanly, says what is left, and carries
  `adding_stopped_at_budget` with the one question to go on, whose yes plans
  the rest with a one-time `token_budget`.

### The run's result

| key | meaning |
|---|---|
| `report` | `{lines, text}`: at most six lines, no code, field, route or path; a file is named by its name, a link by its site |
| `question` | null, or the question to go on after a stop |
| `complete` | false when it stopped at the budget |
| `budget`, `billed_tokens` | what it could spend and what it spent, concept pass included |
| `added` | per document: `file`, `action`, `kind`, `ok`, `note`, `billed_tokens`, `estimated_tokens`, `attempts`, `read_whole` (with `whole_because`), `split_line` (a long document's one plain line on how its parts were found), `chapters`, `partial` (what is still missing, if anything), `version_of` for a version (`of`, `note`, `evidence`, `linked`, and `reason` when it could not be linked), `error` and `reason` when it failed |
| `versions` | every version of this run, as `added` states it, with its `file`; warnings `adding_versions_linked` and `adding_version_not_linked` |
| `retries`, `stopped_at_budget`, `vectors`, `warnings` | as above |
| `concept_pass` | the pass's report: `agent_calls`, `agent_failures` (real attempts that failed), `retries`, `retries_recovered`, `billed_tokens` (every attempt, a failed one included), `agent_available`, `sources_finished`, and `left_for_agent` (`count`, `concepts`, `why`, `retry_estimate_tokens`, `line`) |

An example report:

```
Added 12 documents. 1 long one was read whole, as its chapters were unclear.
1 of them is another version of a document you have; both are kept and linked.
Skipped 2: one already in and a scan with no text.
Could not add 1, even after trying again: ledger.docx.
Used about 1.2 million tokens. Nothing needs you.
```

With every kind of news at once (added, versions, skipped, failed, partial),
the cost and the last line share one line, so the report stays at six.

## How an addition is going: `GET /sources/add/status`

A real run answers at once with its `job_id`, `say` (the catalogue's
`adding.started`) and `next`: the status call. `GET /sources/add/status`
takes `job_id` (omitted: the latest addition of this collection) and `wait`
(seconds to wait for it to finish, bounded as a job wait is). It answers
`state`:

- `running`: `progress` (`done`, `total`, `current`, for a job this process
  started) and `say` ("Adding is under way: 3 of 12 documents so far."), with
  `next`, the same call to wait again. The caller waits; it never starts the
  addition again (Track AI);
- `done`: the run's `report`, its lines in `say` word for word, `complete`,
  `budget`, `billed_tokens`, and, when it stopped at its budget, AR's question
  to go on as one of `asks` (its yes plans the rest);
- `failed`: `say` (`adding.failed`) and the job's `error`;
- `none`: nothing was added to this collection yet.

Refusals: `adding_job_not_found` (404) for a job that is no addition of this
collection. Free and read only.

## A note from the conversation: `POST /sources/note`, `add_note` (Track AY4)

Text the person gives in the conversation (something they wrote, pasted or
said) is kept as a note, not added as a document: `title` and `text` as they
gave them, in any language, `kind` only when they name one. Without `confirm`
the answer is the plan, free and writing nothing: `say` (what will be added,
how many words, and that taking it in uses no tokens), `note.file` (the file
name, the title in ASCII per Track AK, or `note-<hash>` for a title with no
Latin spelling, unique across the collection's notes) and a `confirm` code
bound to the title, the text, the kind and that name. With the code, the note
is written as a new file in the collection's notes folder (never over another
file), word for word under the title as a heading, with `title`, `type` (the
kind's note type, when one was named), `created`, `added_from: conversation`
and `added_by` in its frontmatter and none of the fields the source pipeline
writes, so it is a note written by hand (`second_brain/notes/native.py`): it is
ingested by code (its headings, its concepts by the free extractive reading,
its links), `tokens: 0`. A code for other text, or whose name was taken since,
is refused `confirm_code_stale`; an empty title or text `request_refused`.

Why not the plan of an addition: a note the person wrote is its own original,
and Markdown is the source of truth. This request's pipeline would convert it
(nothing to convert) and have the brain keeper agent rewrite it into a
template, spending tokens to change the person's own words.
`second_brain/source_pipeline/note_from_text.py`.

## Over MCP: `plan_adding`, `start_adding`, `adding_status` (Track AG3)

| tool | route | does |
|---|---|---|
| `plan_adding` | `POST /sources/add`, `dry_run: true` | the plan: `say`, `needs`, `asks`, `start`; free |
| `start_adding` | `POST /sources/add`, `dry_run: false` | the run, only with the plan's `approval` and the same `paths`, `urls`, `kind`; in the background |
| `adding_status` | `GET /sources/add/status` | progress, and at the end the report word for word |

Every view of a plan over MCP carries what the agent needs to act (fix round
R1): the budget question as the last of `asks`, with its text and answers and
its `on.yes` and `on.no` as tool calls (AR's two HTTP calls of a yes with no
standing budget are one `start_adding` with `keep_as_limit`, since no tool sets
the budget alone), the plan's `next` as the `start_adding` call, and `start`. The
`adding_kind_unknown` notice's fix is `plan_adding` again with `kind` to fill,
from the plan's `kinds`: the agent chooses it from the files, never the person
(AR3). One approval is one run (fix round R3): a second `start_adding` with
an approval whose run exists starts nothing and returns that run (`already`,
its `job_id`, `say` "already started" or "already done", and the status call);
the collection's `logs/approved_spending.jsonl` records each approval's run. A
started run's compact result keeps `job_id`, `status`, `say` and `next`, not
the job's bookkeeping. A need's yes (fix round R5) carries `say`: how the
program gets installed (the agent runs the one line in `install` where it may,
else shows it to paste; `install` is null when it is here already, and the yes
only switches it on) and that nothing is added meanwhile.

Each needs the `manage` grant (a local connection whose agent the collection's
policy names at `manage`) and is called with `access_mode: manage`; the route
then also needs that entry to allow `add` (`access_not_granted` otherwise, in
words saying who can allow it). A reading connection lists none of them and is
refused each, by name (S12). `paths` take a folder as the person named or
dragged it, in full: its files are copied into the collection (or pointed to,
when it keeps references), never moved. The calls a plan names come back with
the names the caller sent, as it sent them, so a folder outside the collection
is planned and run as one request.

### The caps of an agent's access, enforced at approval (D4a)

A manage entry's `per_run_tokens` and `per_day_tokens`
([[collections-v1|collections_v1.md]]) limit what can be approved for the
agent the call names. The plan takes the smaller of the per-run cap and what
the per-day cap leaves today (`approvals.agent_cap`; today's approved runs are
`logs/approved_spending.jsonl` of the collection): it never proposes more, the
user's own figure is cut to it, a plan that needs more runs up to it and stops
cleanly, and `say` says so. A run whose approval allows more than the cap now
(the owner lowered it after the plan) is refused `adding_over_agent_limit`
(403), said in words. Each approved run is recorded with its budget, so the
day's cap holds across sessions. The approval of each plan is still needed
every time: a cap limits what can be approved, it never replaces the approval.

## The approval on every spending route (Track AG3, D4, S8)

Every route that spends tokens gives an `approval` with its dry run, and takes
it back in the same field on its real run: `POST /sources/add`,
`POST /sources/process` (its dry run, `dry_run: true`, is new: what would be
read, its estimate, its approval, nothing read), `POST /sources/process_book`
(`dry_run: true`), `POST /sources/process_folder`,
`POST /sources/process_videos`, `POST /sources/figures`,
`POST /sources/equations` and `POST /sources/reread` (each `dry_run: true`, the
default for the last five). Track AW3 brought the last two that still spent
without one under the same approval and setting: `POST /concepts/sync` (a
`dry_run` lists the sources it would read; a collection whose concepts are
written without the AI, `synthesis: deterministic`, spends nothing and needs no
approval) and `POST /query/gaps/interpret` with `auto: true` (a `dry_run` states
how many gaps the keeper would read; handing the prompt back or posting an
interpretation back spends nothing and needs none). Both default to a real run,
as they always did, so a collection with the setting off behaves as before.

- **The code** is AR's, `<issued>.<budget>.<digest>`, stateless
  (`second_brain/source_pipeline/approvals.py`). Its digest binds the route,
  the collection, every argument that changes what is done (not how the call
  is made: `dry_run`, `wait`, `approval`, `access_mode` and the caller's ids),
  the bytes of the file the call names (`path`, `source`, `book`) or the names,
  sizes and times of change of the files of a folder, and the budget the call
  names (`token_budget`). `POST /sources/add` binds its own plan (files, kinds,
  request, budget) and, since AG3, the route and the PDF reader the collection
  chose.
- **The setting.** A real run without a matching approval is refused when the
  collection's `config/intake.json` has `approval_required: true`: written on
  by `POST /collections` (a collection made from the conversation; decided by
  the owner: on for new installs), off by default for every collection made
  before, which keeps today's behaviour until its owner turns it on
  (`change_setting`, "Asking before anything is spent"). `check_setup` says
  which, once, as one of its items, and for a collection where it is off (every
  one made before it existed, registered over HTTP included) it offers to turn it
  on in words (Track AW3); it is never turned on silently, since that would
  refuse the calls of every program already using the collection. `POST /sources/add` needs its approval
  whatever the setting, as it always has.
- **Refusals**, before anything is read or spent, audited (`spending_refused`):
  `spending_approval_required` (422: none, or not a code),
  `spending_approval_mismatch` (409: made for another route, other arguments,
  a file that changed since, or another budget), `spending_approval_expired`
  (409: older than `adding_approval_valid_hours`). Each says the next step: the
  same call as a dry run, its plan shown, then the real call with its approval.
- **What it cannot do.** The platform has no authentication: the code proves a
  plan was made for exactly this call, not that a person read it.

## The standing budget: `GET` and `POST /ops/budget`

`config/budget.json` of the workspace:

```json
{"tokens_per_run": 2000000, "set_at": "2026-10-21T10:00:00+00:00", "set_by": "<agent id>"}
```

| key | meaning |
|---|---|
| `tokens_per_run` | the most one addition may spend without a question; null (or no file) means none is set, and every addition with a cost asks |
| `set_at`, `set_by` | when, and through which agent, the user's answer was recorded |

`POST /ops/budget` takes `tokens_per_run` (0 or more, or null to remove it),
`agent_id` and `workspace_id`, writes the file and records the change in the
governance audit. It is the user's answer, relayed by the caller agent: the
platform has no authentication, so over HTTP any program on the machine could
call it, as it could any route (the same limit as the first-run plan's access
policy states).

## Settings (`config/intake.json`)

| key | default | meaning |
|---|---|---|
| `default_kind` | null | the kind a file gets when the request names none (a template id, note type or alias); null goes by what the collection holds most of |
| `link_chars_estimate` | 40000 | a link's length before it is fetched, for its estimate |
| `adding_budget_headroom` | 1.25 | the proposed budget is the estimate times this, rounded up |
| `adding_approval_valid_hours` | 24 | how long a plan's approval stays valid |
| `adding_retry_backoff_seconds` | 2.0 | the wait before a failed document is tried again |
| `adding_short_note_chars` | 3000 | a plain text file up to this many characters is offered the free way too, as a note the person wrote (Track BA R5); 0 offers none |
| `approval_required` | false | Track AG3: a real run of any spending route needs the approval its dry run gave; a collection made from the conversation writes it true |

The triage thresholds (`long_document_pages`, `identifier_patterns`, and the
rest) are those of [[folder-intake-v1|folder_intake_v1.md]].

## Limits of this version

- **Versions** (Track AJ) are recognised from what a file's first pages,
  frontmatter and name say; a PDF's authors are seen only on its first pages,
  and a format read only after conversion (Word, HTML) has no opening text to
  compare. Making a note again from its source does not keep a `version_of`
  link written into it; the link is written once, when the version is added.
- **Unlinking** has no review action of its own yet: the owner deletes the
  `version_of` line, or withdraws the note through review.
- **ASCII names** (Track AK1) apply to an addition's notes and files, as to
  every note the pipeline names; names written before Track AK are brought to
  the rule by the rename pass of Track AP (AP6).
- **Lint** of each note is in each result's detail but does not cause a retry
  (Track AO).
- **Estimates** are the folder plan's (Track AM: ranges from this install's
  own history, by note type and length band). A link's cost is estimated
  before it is fetched, from a stated length, so its range comes from the
  runs of its note type and is only as good as that guess. A book's chapters
  are assumed in the plan row (one to about 25 pages); the book route's dry
  run shows them from the real split. Video batches (`POST /sources/process_videos`)
  still use the plain fixed-plus-per-character line.
- **The approval** on the older spending routes is enforced only where the
  collection's setting asks for it (off for a collection made before AG3, until
  its owner turns it on). The guard binds a plan's call; it proves nothing
  about who read it.
- **Needs** are read from a PDF's first pages laid out by `pdftotext`; without
  that tool only image objects are seen. Columns and tables are recognised by
  the gaps and numbers of laid-out lines, so a page laid out in another way can
  be missed, and a page of aligned prose can be taken for a table. An e-book or
  a Word file's pictures are counted only where intake counts them.
- **Progress** is counted in the process that started the run; a status read
  from another process says the run is under way without the count.
- **The caps** bind the agent a call names: over MCP that is the connection's
  agent; over HTTP any program can name any agent (no authentication).
- **Over MCP** only `POST /sources/add` is offered; the lower-level intake
  routes stay the owner's operations, under the same enforced approval.
