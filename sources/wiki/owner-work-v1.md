---
type: "documentation"
title: "The owner's own work v1 (Track K)"
description: "Built 2026-10-09 for Track K of docs/knowledge_layer_improvement_plan_v1.md. Code: second_brain/owner_work/. Routes: docs/api_spec_v1.md section 7b."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing Kovium"
doc_path: "docs/owner_work_v1.md"
doc_sha256: "ac87db55ad2e57af906fc3c72ee49cc47e3aa6e8b8b63728f6d6bf109576a9ec"
---

# The owner's own work v1 (Track K)

Built 2026-10-09 for Track K of `docs/knowledge_layer_improvement_plan_v1.md`.
Code: `second_brain/owner_work/`. Routes: `docs/api_spec_v1.md` section 7b.

## Why

A collection holds more than its sources. Its owner writes about their *own*
contribution (a design, a method, a draft chapter, a draft argument) and about
*other people's* work (how a published method works, how they adapted it, what
they think of it). Before this track either kind could only enter as a source,
so it was cited "from the collection" beside peer-reviewed work, its concepts
fed the concept notes' literature synthesis, its comparisons became claims
attributed to it, and a note that differed from the paper it describes looked
like two sources disagreeing. Nothing recorded that it was unfinished. A
lawyer's draft argument, a team's design document and a reader's notes on a
book are the same case.

## Two kinds, one note type

`type: owner_work`, with:

| field | meaning |
|---|---|
| `kind` | `design` (the owner's own contribution) or `commentary` (notes on someone else's work) |
| `status` | `draft` or `stable` |
| `author` | `owner` |
| `about` | for a commentary, the work it discusses: a note key when the collection holds it, else citation text (`Author et al., Venue 2023`) |
| `concepts` | concepts it is about (resolved through the alias index at registration) |
| `compares_with` | works it compares itself with (note keys or citation text); omitted, the citations in its comparison part |
| `updated` | when the owner's text last changed |
| `maintained_by` | always `owner`: the brain links and reports, never rewrites |

The parts each kind keeps apart:

- **commentary**: *what the source says* (`source_says`), *how we adapt or use
  it* (`adaptation`), *our assessment* (`assessment`), plus `comparison` and
  `example`;
- **design**: `problem`, `approach`, `method`, *assumptions and limits*
  (`limits`), `comparison` with related work, `open_questions`, plus `example`.

`example` is an illustration the owner made up. Its numbers are theirs, so it
is never compared with a source.

## Recognised by its own headings

An owner's notes already have a structure, and it is theirs. A note is read
from its own headings; the template's are never demanded. `GET
/owner-work/template?kind=` returns a blank one for whoever wants it.

- Numbering and emphasis are not part of a heading: `## 3. One decision, step
  by step` is read as `One decision, step by step`.
- The first `#` heading is the title. Sections are the shallowest heading level
  after it; deeper headings belong to the section they sit in. A `#` inside a
  fenced code block is code.
- A heading maps to a part by, in order: the registration's own mapping (the
  request's `headings`, `{"<heading as written>": "<part>" | "body"}`), the
  template's heading, the workspace's patterns, then the shipped ones.
- The shipped patterns are English regular expressions, per kind, tried in
  order (the first part that matches wins, so "the paper versus our
  implementation" is how we adapt it, not a comparison). An install writing in
  another language adds its own in `config/owner_work.json`:

```json
{"headings": {"commentary": {"source_says": ["\\bdie idee\\b"], "adaptation": ["unsere umsetzung"]},
              "design": {"limits": ["grenzen"]}}}
```

  They are tried before the shipped ones, never replace them. A pattern that is
  not a regular expression is matched as plain text and reported under
  `problems`.
- Adaptation phrases (`our application`, `for our ...`, `my own setup`) are
  each searched on their own anywhere in a heading, so a typo in another word
  does not defeat them.
- **A design's sections are placed by position** (K-R2). A design explains its
  method in sections named after what they do ("The score", "The debt
  counter"), which no pattern can know in any language. An unmapped section
  after the problem, approach or method and before the limits, comparison or
  open questions is the `method`, reported with `mapped_by: position` (the
  reply's `headings` and `mapped_by_position`). Language-free.
- **A heading nothing maps is kept as body**, listed under `body` and in the
  registration reply's `unmapped_headings`, never dropped, unless the request
  gives a `default_part` (a part of its kind, stored in the registration,
  `mapped_by: default`).

## Where the text lives

The owner's file is the source of truth and is **never modified**. It stays
where the owner keeps it: anywhere in the workspace, or under a declared source
root (`config/sources.json` `source_roots`, read as `<root>:<path>`; see
`docs/source_roots_and_portable_provenance_v1.md`).

Registering it writes a **registration note** into `sources/wiki/own-work/`
(one level below the wiki, like concept notes and ideas, so every per-note scan
of source notes leaves it alone):

```
---
type: owner_work
description: These notes explain Rotaflow (Okafor et al., GardenConf 2021), ...
kind: commentary
status: draft
author: owner
maintained_by: owner
about: Okafor et al., GardenConf 2021
concepts: [watering-schedule]
compares_with: []
heading_parts: []
source: notes:rotaflow-notes.md
sha256: <the bytes read>
updated: 2026-10-09
registered: 2026-10-09
---

# Rotaflow: how it works

_Written by the owner and kept at `notes:rotaflow-notes.md`. ..._
```

`title:` is written only when the owner states one; otherwise the text's own
first heading is the title, so renaming it there renames it here. Values that
may hold commas (`compares_with`, `heading_parts`) are block lists.

The graph is built from the two together (ingestion follows `source:`), so it
stays rebuildable from Markdown: `scripts/rebuild_graph.py` reads
`own-work/` too. A note written directly in `own-work/` with `type: owner_work`
frontmatter is its own text, ingested as it is.

The registration records `source:` and `sha256:` exactly as every note records
its original, so Track J's registry knows the file: a reconcile reports the
owner's edit as `changed` (with `owner_work_changed` naming
`POST /owner-work/refresh`), and a moved file is found by its bytes and
**proposed** for review, never repaired silently, because the registration is
owner-maintained. Folder triage reports a registered file as
`already_ingested`, so a folder holding the owner's notes never processes one
as a source.

## Intake: no tokens

`POST /owner-work` takes a path and what the owner says about it, and calls no
agent. Refusals: no `access_mode` (422); no `kind` and none in the file's own
frontmatter (400); an undeclared root or a file outside the workspace and every
root (400: a recorded path must not name whose machine it is on); no file
(404); not Markdown or plain text (415); a note of the collection under
`sources/wiki/` (409: it would count twice); a file already registered with
other fields and no `update: true` (409). Registering again with the same
fields is `unchanged` (200), or `refreshed` when the text changed.

What a commentary is about: `about` as given, else the file's own frontmatter,
else the **first citation in its opening** (`stated_by: first_citation`). It is
resolved against the collection's source notes (`second_brain/owner_work/
collection.py`): a note key or `[[link]]`, a citekey, an exact title, an
identifier, or a citation and a year.

- A citation's parts may come in any order (`Author et al., Venue 2023`,
  `Title, Author et al., 2024`). The author part is the one with `et al.`,
  else a short one joining names with `&` or `and`, else the first short one;
  a longer part before it is the cited title.
- The first author is matched **whole** (K-R1): their written name, as
  consecutive words, anywhere in one of the note's `authors`, case and accents
  folded, so `Ben Sada` fits `Abdelkarim Ben Sada` and `van der Berg` fits
  `Jan van der Berg`, while one word of a compound surname alone does not
  single a note out. A note whose `citekey` starts with that name's last word
  fits too (authors written surname first).
- The work's own name confirms or settles (`evidence`): the commentary's
  `Foo` (from `Foo: how it works`) or the cited title in the note's title; a
  one-word name or the cited title's acronym (`QRSS` from `Quiet Rotating
  Sprinkler Scheduling`) in the note's text; or most of the cited title's
  words in its title. Several authors' fits are narrowed by it, then by the
  venue. With no author fit, a single note with the year and the name is taken
  (`basis: name_and_year`).
- Several notes that still fit stay `ambiguous`, listed as `candidates`: never
  guessed. Resolution runs at every ingest, so a work added to the collection
later is linked then (an ingest of the new source re-links the owner notes that
name it).

`POST /owner-work/refresh {keys?, force?}` reads again each file whose bytes
changed (`GET /owner-work` lists them under `text_changed`), updating the graph,
`## Your work` and only the registration's `sha256` and `updated`.

## A tier of its own

- **Never evidence.** Ingestion gives an owner-work note its Note node and
  three edge types no other note writes: `WORK_ABOUT` to a concept,
  `COMMENTS_ON` to the source note a commentary discusses, `POSITIONS_AGAINST`
  to a work it compares itself with. No line becomes a node, so nothing of it
  is ever a claim, least of all a claim of the work it discusses. Ranking skips
  its notes, so it is never in `knowledge_items`, `read_next`,
  `/query/resolve_sections` or a topic scope.
- **Apart in the answer**: `from_your_work` (`docs/agent_memory_contract_v1.md`).
- **Concept notes** gain `## Your work` (`docs/concept_notes_v1.md`), after
  `## Ideas`. Concept sync reads the wiki's top level only and skips
  `type: owner_work`, so owner work never feeds Definition, Development,
  Divergences or how sources model a concept.
- **A design's assumptions and limits are the owner's own open problems**
  (`GET /owner-work/open_problems`), reported apart from the literature's: the
  limitation-clusters answer carries them under `owner_open_problems`, and
  owner work never joins a cluster or a gap theme as a source (it has no
  Limitation nodes).
- **A comparison is a link** (`POSITIONS_AGAINST`) to the compared work's note,
  never a claim made by that work.

## The owner's words beside a source (Track AH, 2026-10-22)

Not an owner-work note: a line or two a person writes in the `## My notes`
section of a note the pipeline wrote (`docs/note_format_contract_v1.md`, "The
owner's section"). It is the same tier as the owner's work and obeys the same
rule: **never evidence**.

- No line of it becomes a claim, a concept, a relation or a finding, and the
  concept layer never reads it.
- It is found, and shown apart: a question using its words finds the note; the
  governed answer lists it in `from_your_work` as `kind: my_notes`
  (`maintained_by: owner`, a label, the words, the call that reads the note),
  beside the owner-work notes and after them, bounded by the same
  `from_your_work_limit`. It is listed when the answer's evidence comes from
  that note, or when its words hold the question's (the same coverage as an
  owner-work note's `words`).
- A read of the note shows it under its own block and as `owner_notes`, never
  among the source's sections.
- It outlives its note: if the note is removed with its source, the words are
  kept in `maintenance/kept_owner_notes/` and a review item names them.

## Commentary that differs from its source (K3)

`POST /owner-work/review {keys?}` (and every registration, refresh, and ingest
of a source a commentary discusses) compares a commentary's `source_says` part
(`config/owner_work.json` `compare_parts`) with the source note it discusses.
Deterministic, no tokens, erring towards silence:

- **parameters**: a name bound to a value with `=`, `:=`, `≈` or `≡` (`θ =
  0.30`, `\lambda = 3`; Greek letters and their names are one name). The same
  name with no value in common is a difference.
- **values in prose**: a statement with a number, and a statement of the source
  note on the same point: the source's statement holds at least
  `differs_min_coverage` (0.6) of the owner's rarity-weighted words, numbers
  left out, and at least `differs_min_terms` (3) of them, by the strict word
  matching of the governed answer (Track I). When no statement on that point
  shares a value with the owner's, the best-matching one is reported.
- Values are numbers written any way: decimal point or comma, thousands
  separators either way, a percentage as a fraction too, compared to the
  precision of the less precise (`0.33` agrees with `0.333`).

One item per commentary and source, `owner-work-differs:<key>+<note>`, naming
both statements with their sections and lines (`differences`, at most
`differs_max_listed`), and saying either may be wrong: the owner's draft or the
extraction. Never acted on. Resolved or dismissed, it is not raised again for
the same differences; a different difference found later is a new item
(`...+<note>.2`). When the two agree again (the owner corrected a draft), the
open item is closed by the brain with a note saying why.

**A commentary on a work the collection does not hold** is a read-next
candidate: `GET /owner-work` lists it under `read_next_candidates`, and an
`owner-work-unheld:<key>` item says so. When a file in the intake folders
(`sources/raw/` and every declared root) looks like the cited work, the item
names it: the first author's name and the year, or the work's name and the
year, in its file name or on its first page (`pdftotext` for a PDF, cached per
file stamp in `maintenance/owner_work_intake_cache.json`), at most
`intake_scan_limit` files opened per look. A file a note already records is
held, not a candidate.

## Notes on any kind of source (Track U1, 2026-10-11)

The first instance held papers, so an owner's note found its source only by a
note key or an author-year citation, among the wiki's top-level notes. A
meeting, an email, a customer record, a recipe or a journal entry has no
citation, and an owner writing "thoughts after Tuesday's call" names none.

**A third kind, `notes`.** The owner's notes on a topic or on something that
happened (a meeting, a call, a visit, a day). Parts: `observations` (canonical
heading *What happened*), `thoughts` (*My thoughts*), `actions` (*Next
steps*), `open_questions`, `example`; English patterns ship (`what I
noticed`, `ideas`, `follow-ups`, …) and an install adds its own language under
`headings.notes` in `config/owner_work.json`. Chosen over stretching the other
two: a commentary's parts presume a work whose stated values can be compared
with the owner's (K3 runs on commentary only), a design's presume a method of
the owner's. A single idea is an idea note (Track H). `notes` are never
compared with their source; they are unheld or unconfirmed like a commentary.

**Every note of the collection is a candidate**, top-level or in a folder of
its own (`meetings/`, `2026/03/`); the brain's folders (concepts, maps, ideas)
and `own-work/` are not. A stem a top-level note already has is keyed by its
wiki path (`meetings/kickoff`). `[[stem]]` and `[[folder/stem]]` both resolve.

**Identifying fields come from the note contract**, never from names in the
resolver: each type declares `identity_fields` (field → `identifier`, `date`,
`person` or `organisation`). A meeting is known by `meeting_date`,
`attendees`, `owner`, `project`; a web page by `url`, `site`; a paper by
`citekey`, `authors`, `year`, `venue`; every type also by `id`, `identifier`,
`doi`, `date`. An install declares its own in `config/note_types.json`
(`"identity_fields": {"account_id": "identifier", "company": "organisation"}`).

**`about` resolves in this order**: note key or link; citekey; exact title;
any identifier field's value; an author-year citation (unchanged: a citation of
a work not held stays a read-next candidate); the work's name in its title;
then **signals** (`basis: signals`): what the text shares with one note: a
date (`2026-03-14`, `2026/3/14`, `14.3.2026`; month names are words of one
language and are not read), the names of its people and organisations (all
of a name in order, or a long name's first two words), its whole title in the
reference, most of its title words (rarity-weighted once the collection has
four notes or more), an identifier. A name that is also the note's title (an
account named for its company) counts once.

**A note that states no `about`** (kinds `commentary` and `notes`) is matched
the same way from itself (`infer_about`): dates in its title or its own
frontmatter count fully, dates in its text less; plus a concept both are
about and the source's rarer words of its title and description. Scored: date
2 (1 in the text), each name 1 (at most 2), title 1-2, whole title in title 2,
concept 1, wording 1, identifier 3. **Linked** (`basis: inferred`) only when
the best scores 3 or more, leads the next by 2, and rests on a date, a name,
an identifier or its whole title; **suggested** (`basis: suggested`, the best
five scoring 2 or more) otherwise, never linked. Recomputed on every read, so
a meeting added later is found; ingest routes re-link and re-review the notes
a new source could be about. Deterministic, no tokens.

**Review**: a suggested link raises `owner-work-link:<key>` listing the
candidates (with each one's evidence); a dismissed item is not raised again
for the same candidates. `owner-work-unheld:` now covers `notes` too.
`GET /owner-work` adds `about_unconfirmed`; summaries add `about_basis`;
registration reports `about.stated_by` (`given`, `first_citation`,
`inferred`, `suggested`, `not_stated`) and warns `about_uncertain`.

In answers, `from_your_work[].about` carries `note`, `title`, `path` and
`basis` for an inferred link too, and the reason reads "is about" for a note,
"comments on" for a commentary: the meeting appears with the owner's note, and
the owner's note with any answer whose evidence comes from the meeting.

## Configuration

`config/owner_work.json`, every key optional:

| key | default | |
|---|---|---|
| `headings` | `{}` | `{kind: {part: [pattern]}}`, tried before the shipped ones |
| `compare_parts` | `["source_says"]` | the commentary parts compared with the source note |
| `differs_min_coverage` | `0.6` | share of the owner's statement's words the source's must hold |
| `differs_min_terms` | `3` | and at least this many |
| `differs_max_listed` | `10` | differences in one item |
| `concept_sections` | `true` | write `## Your work` |
| `intake_scan_limit` | `400` | files opened per look in the intake folders |

## Limitations

- Heading recognition ships English patterns; another language needs its
  patterns declared (or a per-note `headings` mapping).
- K3 compares values only. A definition stated differently, a parameter written
  as prose ("a threshold of 0.3"), equivalent values in different units or
  forms ("3x faster" and "70% less time": Track L's numeric checks) are not
  judged; the last is reported and dismissed.
- The same symbol meaning different things in two places is reported as a
  difference.
- Citation parsing reads author-year forms; a numbered citation (`[12]`) names
  nothing resolvable.
- Inference reads dates written with digits only; "14 March" names nothing. Two
  sources sharing a date and the same people (two meetings on one day) are only
  suggested. Wording uses a source's title and description, not its body.
- The point values are fixed in code (`owner_work/collection.py`), not yet
  configurable per install.
- The intake-folder match is a hint from names and years on a first page, not
  an identity.
- When the owner's file is unreachable (a root not attached), the note keeps
  its registration and reports `text_status: missing`; its parts are empty
  until the file is back.
