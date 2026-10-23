---
type: "documentation"
title: "L3 Note Format Contract v1"
description: "The L3 note is the human-readable artifact and the canonical semantic unit. Everything else — graph nodes, maps, indexes — is derived from it and must stay rebuildable from it."
status: "stable"
reading_status: "read"
product: "second brain"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing second brain"
doc_path: "docs/note_format_contract_v1.md"
doc_sha256: "82cff5209fbb95239b05a1f13e81c60015b7c85450f1274b283a88fdcb3c9853"
---

# L3 Note Format Contract v1

The L3 note is the human-readable artifact and the canonical semantic unit.
Everything else — graph nodes, maps, indexes — is derived from it and must stay
rebuildable from it.

Three layers. The Literature vault was the design inspiration; second_brain is
a separate, self-contained project and shares no files or formats with it.

| layer | what | where |
| --- | --- | --- |
| L1 | original source | `sources/raw/` |
| L2 | converted Markdown | `sources/converted_markdown/` |
| L3 | **the note** | `sources/templated_markdown/`, normalized into `sources/wiki/` |
| derived | graph nodes, note maps | the store, `sources/wiki/maps/` |

## Why sections are split into three kinds

A second brain ingests more than papers — meeting notes, transcripts, blog
posts. Those share very little *content* structure with a paper, but a reader
browsing the knowledge base needs the same entry points regardless of what they
opened. So the contract has:

- **Mandatory sections** — present in *every* note type, whatever the source.
  These are what a reader browsing the corpus uses, and what cross-type
  retrieval can rely on existing.
- **Type-specific sections** — meaningful only for one source type. A paper has
  Experiments; a meeting note does not.
- **Derived sections** — written by the pipeline, never by hand.
- **The owner's section** — `## My notes`, the opposite kind: written by the
  owner, never by the pipeline (Track AH).

The same split applies to frontmatter.

## The organising distinction: browsing vs reading

Two sections can look like duplicates and are not. The rule:

- **Mandatory sections are written for someone browsing the knowledge base**,
  who has not read the source. They must stand on their own: no undefined
  jargon, no "the proposed method", no section references.
- **Type-specific sections are written for someone reading this source.**
  In-context language, domain terms, and `§`/page anchors are all correct there.

Applied to the pairs that overlap:

| pair | mandatory (`Claims`, `Evidence`) | type-specific (`Key results`, `Experiments`) |
| --- | --- | --- |
| scope | general, standalone | this paper, in its own terms |
| audience | anyone browsing | someone reading the paper |
| jargon | avoided or explained | expected |
| anchors | not required | `(§III-A)`, `(p. 7)` required |
| example | "Raising a task's priority point never worsens another task's response time." | "Observation 1 (§III-A): raising τᵢ's relative PP by X > 0 increases its own WCRT by at most X." |

`Claims` is what you would tell a colleague in one sentence. `Key results` is
what you would cite.

## Mandatory sections — every note type

| section | contains |
| --- | --- |
| `## TL;DR` | one to three sentences, plain language: what this is and why it matters |
| `## Concepts` | the fine-grained concept list; grows without bound, drives the graph |
| `## Key Concepts To Understand` | short glossary for a reader new to the area |
| `## Claims` | standalone statements, understandable without the source |
| `## Evidence` | accessible support for those claims |
| `## Questions` | what this note leaves open |
| `## Relation to other work` | `[[wikilink]] — relationship`, for navigation |
| `## Provenance` | source, extracted text, ingest date |

## Type-specific sections — `type: paper`

| section | contains |
| --- | --- |
| `## Problem` | what breaks without this work; the setting and its assumptions |
| `## Approach` | the mechanism in prose — the narrative bullets cannot carry |
| `## Methods` | one bullet per method: what it does and what it assumes |
| `## Experiments` | setup, baseline, outcome |
| `## Key results` | the claim with its number and where it came from |
| `## Conclusions` | what the authors themselves conclude — their claim, not your reading |
| `## Assumptions & limits` | assumptions, setting, what the result does not cover |
| `## Quotes` | verbatim, with `(p. N)` or `(§X)` |
| `## Structured Extracts` | compact machine-readable extracts (formulae, parameter sets). **Tables from the source are preserved here automatically** — see below. |

Any note type, not only a paper, may also carry `## Source tables`: the
source's own tables, written by the pipeline after extraction, never by an
agent, each row of a table of data read as an `Extract` with its column names
and the table's caption (`docs/source_tables_v1.md`). Like `## Figures` and
`## Source equations` it is written by code; it is not edited by hand, and a
re-process or re-read writes it again. So is `## Source entries`, on any note
type whose source holds a list of terms with their definitions: one item per
entry, `- **Term**: definition` as the source writes it, each read as a
`Definition` with its term (`docs/reference_lists_v1.md`).

## Type-specific sections — the other built-in types

second_brain is not a paper tool, and these five are first-class, not
afterthoughts. Each is defined once as a `TypeProfile` in
`second_brain/notes/contract.py`; the template file, the pipeline's rendering
and this table all follow from it.

None of them repeats a mandatory section. `Claims` and `Evidence` are already
the standalone, browsable versions — a type earns a section only where it adds
something those cannot carry, and duplicating would split one piece of content
across two headings.

### `type: meeting-note`

| section | contains |
| --- | --- |
| `## Context` | why the meeting happened and what surrounds it |
| `## Discussion` | what was argued, in the room's own terms |
| `## Decisions` | what the meeting **actually decided** — not what was floated |
| `## Action items` | who does what by when, where stated |
| `## Risks / blockers` | what could go wrong, or what is blocking progress |

*Special rule:* a spoken idea is not a settled fact. Anything floated but not
decided belongs in `Discussion` or `Claims`.

### `type: chat-transcript`

| section | contains |
| --- | --- |
| `## Context` | what the conversation was for |
| `## Discussion` | the reasoning trail |
| `## Decisions` | what was settled |
| `## Follow-up items` | what still needs doing |
| `## Disagreements / uncertainty` | where participants disagreed or were unsure |
| `## Referenced sources` | sources mentioned in the thread |

*Special rule:* preserve speaker identity wherever it changes ownership,
confidence, or authority — "someone suggested" and "the maintainer decided"
are different facts.

### `type: white-paper`

| section | contains |
| --- | --- |
| `## Problem framing` | the problem the document says it addresses |
| `## Proposed approach` | the mechanism it proposes, in prose |
| `## Recommendations` | what it recommends the reader do |
| `## Assumptions & limits` | what it assumes and does not cover |
| `## Risk and bias notes` | who published this, and what they gain if the reader agrees |

*Special rule:* keep marketing claims, empirical evidence and recommendations
separate. An unsupported claim should read as visibly lower-trust than a
measured one.

### `type: blog-post`

| section | contains |
| --- | --- |
| `## Context` | what prompted the post |
| `## Main argument` | the central line of reasoning |
| `## Technical details` | mechanism, configuration, implementation specifics |
| `## Assumptions & limits` | what it assumes and does not cover |
| `## Applicability notes` | when this does and does not transfer elsewhere |

*Special rule:* separate explanation, opinion and evidence. An example is not
evidence unless it supports a technical claim.

### `type: webpage`

| section | contains |
| --- | --- |
| `## Context` | what this page is, whose site it is, what it exists to do |
| `## Reference details` | a specific fact the page is consulted for: a value, an option, a limit |
| `## Procedures` | a step the page instructs the reader to perform, in order |
| `## Volatility notes` | how likely the page is to change, and what would make this note wrong |

*Special rule:* the source can be edited in place with nothing announcing it, so
a webpage note is only ever true as of `retrieved_at`. That field is
load-bearing, not bookkeeping. Navigation, cookie banners and marketing copy are
not content — record what the page tells a reader, not what it displays.

Use `blog-post` instead when the page is an authored argument. A documentation
page has no thesis, and forcing one out of it invents a claim it never made.

### `type: documentation`

Track AG1: a page of documentation kept as a file (a manual, a how-to, a user
guide, a technical reference, a specification).

| section | contains |
| --- | --- |
| `## Purpose` | what the thing the page describes is for, and who the page is written for |
| `## How it works` | how the thing works, as the page explains it |
| `## How to use it` | a step the page tells the reader to take, in order, with exact names and values (may be empty) |
| `## Settings and options` | a choice the thing offers, what it changes and its default (may be empty) |
| `## Known limits` | what the page says the thing does not do, or where it stops (a limitation) |

Fields: `product`, `product_version`, `doc_kind`, `audience`, `url`,
`last_updated`. *Special rule:* like `webpage` it argues nothing; unlike it, it
is versioned with what it describes rather than edited in place under a link,
so its last section is where the thing stops, not how soon the page may change.
Use `webpage` for a page of the open web.

### `type: voice-note`

| section | contains |
| --- | --- |
| `## Context` | where and why it was recorded, and what prompted it |
| `## Unfinished thoughts` | half-formed ideas, kept as fragments rather than promoted |
| `## Action items` | something the speaker said they need to do |
| `## Transcript quality notes` | where the transcript is unreliable, and how that limits the note |

*Special rule:* a voice note is a **capture, not a conclusion** — it defaults to
`status: draft` and is promoted by a human deciding it is settled. `Claims` holds
only what the speaker plainly meant as a statement; "I think maybe we should look
at whether…" is an unfinished thought. Without that section every fragment gets
forced into `Claims` and the note asserts far more than the speaker did.

### `type: decision-record`

| section | contains |
| --- | --- |
| `## Context` | what forced the decision, and why then |
| `## Options considered` | an option on the table, and why it was not chosen |
| `## Decision` | the decision itself, stated once — a bullet, so it reaches the graph |
| `## Consequences` | what is now committed to, **including the costs** |
| `## Revisit when` | the condition that should reopen it, or "no known trigger" |

*Special rule:* overlaps `meeting-note` deliberately. A decision that outlives
the meeting it came from needs its own identity, because it is looked up later
by people who were not in the room. `Options considered` exists because a
decision recorded without its alternatives is undocumented — the next person
re-proposes the rejected option and nothing tells them it was rejected.
`Revisit when` exists because a decision has a lifetime, and recording what
should reopen it is the difference between a record and a minute.

### `type: reference-list`

| section | contains |
| --- | --- |
| `## About this list` | each paragraph the source writes about the list itself (an introduction, a note on usage), as written |
| `## Source entries` | one item per entry, `- **Term**: definition`, written by code |

*Special rule:* a list a reader looks things up in (a glossary, an
abbreviation list, a lookup list, a code table) is **built without an agent**:
an entry is kept exactly as written and found by its term, because a summary
of a list loses the entries a reader comes for. Track AE,
`docs/reference_lists_v1.md`.

### `type: ai-conversation`

| section | contains |
| --- | --- |
| `## What was asked` | the actual goal of the session, not the first prompt |
| `## What was established` | a conclusion the session **checked** — name what checked it |
| `## What was asserted` | what the model stated that nothing verified |
| `## Corrections` | where the human pushed back and the model was wrong |
| `## Dead ends` | what was tried and abandoned, so it is not retried |
| `## Artifacts produced` | files, commits or decisions that outlived the session |

*Special rule:* this is **not** a `chat-transcript`. A human thread is people
disagreeing; an AI session is a machine producing fluent, confident output whose
reliability varies wildly *within one conversation* — a verified command result
and an invented API arrive in the same assured tone. Something belongs in
`established` only if the session actually checked it: a command run, a file
read, a test passing, a citation followed. Everything else is `asserted` — not
wrong, unverified. Collapsing the two launders model output into the knowledge
base as established fact.

`Corrections` is the highest-value section here and the first thing a naive
summary destroys.

*How the split is enforced.* The type declares an **evidence split** on its
profile: which section holds checked claims, which holds stated ones, and which
phrases show an item names its check (`confirmed`, `measured`, `ran the`,
`grep`, … — matched at the start of a word, so `grep` finds "grepping" and
`verified` does not find "unverified"). Two things read it:

- **Lint** warns `claim-without-evidence` for an item under `## What was
  established` that contains none of the phrases. A warning, never an error:
  the phrases are lexical, and miss real wordings in both directions — "not
  verified" contains `verified`.
- **`POST /maintenance/evidence_split_audit`** sends only those flagged items
  to the brain-keeper (skill `audit_evidence_split`), which rules each
  `verified`, `asserted` or `uncertain`. Report-only by default. With
  `"apply": true`, items ruled `asserted` move to `## What was asserted` and
  nothing else in the note changes; the note is re-ingested so the graph
  agrees. Nothing moves without a ruling, an `uncertain` ruling moves nothing,
  and a note edited while the audit ran is not written. With no brain-keeper it
  returns the lint findings with `adjudicated: false`.

The split is not specific to AI sessions or to English. Any type may declare
one in `config/note_types.json`, and a workspace overrides it field by field:

```json
{"types": {
  "ai-conversation": {"evidence_split": {"evidence_markers": ["bestätigt", "geprüft", "gemessen"]}},
  "lab-notebook": {
    "sections": ["Observed", "Hypothesised"],
    "evidence_split": {"verified_section": "Observed",
                       "asserted_section": "Hypothesised",
                       "evidence_markers": ["measured", "photographed"]}}
}}
```

`"marker_match": "substring"` drops the word-start rule, for scripts written
without spaces between words. `"evidence_split": false` switches the check
off for a type. A split declaring no phrases is never linted — every item would
fire — and the audit then sends all of its verified items to the agent.

### `type: book-chapter`

| section | contains |
| --- | --- |
| `## Place in the book` | which chapter it is, what it follows, what it sets up |
| `## Main argument` | the through-line, **in the order the chapter builds it** |
| `## Prerequisites` | what it assumes the reader already knows or has read |
| `## Worked examples` | an example it teaches through, and what it is meant to show |
| `## Exercises & practice` | what it asks the reader to do; empty for most non-textbook chapters |

*Special rule:* a chapter is **not a standalone document** — the only type here
that is not. It sits in an argument that began before it and assumes the reader
did the earlier part, so `Place in the book` and `Prerequisites` are what make
its claims usable at all. Write `Place in the book` even when the chapter number
is all that is known. Order is content in `Main argument`: a chapter teaches by
sequence, and a shuffled list loses the argument.

A whole book is **split into chapters first** — one note per chapter, plus a
parent index note. One note cannot represent a book: measured on a real
14-chapter book, a single note retained 12 concepts and 12 claims out of roughly
1,600 extracted items.

### `type: youtube-video`

| section | contains |
| --- | --- |
| `## Context` | what the talk is and who gave it |
| `## Topic segments` | the segments and what each covers |
| `## Timestamped anchors` | `(MM:SS)` for important claims and demonstrations |
| `## Transcript quality notes` | where the transcript is unreliable, and how that limits the claims |

*Special rule:* timestamp important claims. Where the transcript is noisy, keep
the claim conservative and say so rather than guessing at the words.

## Any type, declared by the install: sections, roles, identities (Track U, 2026-10-11)

A type an install declares in `config/note_types.json` is a type like the
shipped ones. Besides `sections`, `frontmatter`, `facets`, `map_branches`,
`map_labels`, `link_fields` and `evidence_split`, each type may declare, as
objects merged over the built-in's:

```json
{"types": {
  "recipe": {
    "sections": ["Ingredients", "Steps", "What can go wrong"],
    "section_nodes": {"ingredients": "Ingredient", "steps": "Step", "what can go wrong": "Pitfall"},
    "section_roles": {"what can go wrong": "limitation"},
    "identity_fields": {"recipe_id": "identifier", "cook": "person", "cooked_on": "date"}
  }
}}
```

- `section_nodes`: heading (case-folded) → the graph node type its bullets
  become. Before this a declared type's sections produced nothing.
- `section_roles`: heading → `answer` (the default for every mapped section),
  `question`, `limitation` or `context`. **Answer weighting reads roles**,
  never node-type names: every `answer` and `limitation` section ranks with a
  paper's methods (boost 4.0), a `question` with questions (3.0), a `context`
  section at 1.0. **The limitation and gap views read roles too**: the shipped
  limitation-like sections are a paper's, white paper's and blog post's
  `Assumptions & limits`, a meeting's `Risks / blockers`, a white paper's
  `Risk and bias notes`, a thread's `Disagreements / uncertainty` and a web
  page's `Volatility notes` (not a transcript's quality notes, which share the
  `Caveat` node type: the section decides).
- `identity_fields`: field → `identifier`, `date`, `person` or `organisation`:
  what the document is known by. Owner notes are linked by them (Track U1), a
  question or `scope: {"identifier": …}` names a source by an `identifier`
  value (U3), folder intake recognises a file's kind by them (U4). Every type
  also has `id`, `identifier`, `doi` (identifiers) and `date`. An unknown
  value role is dropped, never half-applied. The same fields recognise
  another version of a work when documents are added (Track AJ): an
  `identifier` names the work, a `person` or `organisation` corroborates a
  title ([[folder-intake-v1|folder_intake_v1.md]], "Versions of one work").
- `work_fields` (Track AJ2): a list of fields whose different values make two
  notes of the type two different works, whatever else they share. Shipped:
  `meeting_date` (meeting notes), `date_range` (chat transcripts),
  `recorded_at` (voice notes), `decided_on` (decision records),
  `chapter_number` (book chapters); a paper declares none, as a preprint and
  its published form differ in their year. `"work_fields": ["schedule"]` makes
  a contract's annex another work than the contract it shares a reference with.
- `versions_by_opening` (chores after Track AM): `true` or `false`, whether two
  documents of the type whose people cannot be read may be taken for versions
  of one work on a nearly identical opening alone. Absent: inherited (`true`
  for a type the install declares from nothing). Shipped `false` for the types
  whose documents are commonly made from one template, where the opening is the
  template rather than the work: meeting notes (an agenda's preamble),
  web pages (a site's header), decision records (an ADR form), AI
  conversations (sessions opened by one prompt) and video transcripts (a
  channel's spoken intro); `true` for papers, white papers, blog posts, book
  chapters and the rest, whose openings are their own. An install whose
  contracts or forms come from one template sets it `false` on that type.

**Notes written by hand (native notes, U2)**: a note in the wiki, at the top
level or in a folder of its own, that the pipeline did not write (no
`template_id`, `converted_markdown_path`, `raw_source_path`,
`templated_source_path`, no `## Normalization Provenance`). Ingestion reads it
under its own headings: a heading that stands for one of its type's mapped
headings (the same words of four letters or more, compared by their first
five: `## Risks` for `Risks / blockers`, `## Decisions taken` for `Decisions`)
gives that node type; any other heading of the author's own gives a
`Statement`. A note that uses none of the contract's mapped headings is read
whole, its opening included. Under those headings, and under its type's own
mapped sections, each paragraph sentence and each numbered or `*` item counts,
not only `- ` bullets; the contract's shared sections (concepts, claims,
evidence, questions) stay one bullet each. A note that follows the contract
keeps its prose sections (TL;DR, Context, …) as prose. Its concepts, when it
states none, are derived (`notes/native.py`): the agent's, for these bytes,
after `POST /notes/native/extract`, otherwise the extractive fallback's
(`concepts_basis` on the Note node); its identity terms are read from itself.
Nothing is written into the note.

## What does not apply, what may be empty, what is cited (Track AO)

Lint used to treat three honest answers as defects, and each pushed someone to
invent something. They are declared per note and per type, for any kind of
material; nothing here knows what a paper is.

**A facet can be said not to apply to a note (AO1).** `facets_not_applicable:
[locus]` in the frontmatter says the facet's question does not arise for this
note: a meeting with nobody to decide anything, a recipe with no source. It is
not the same as an empty facet, which means no registered value fits. Lint
counts a marked facet as reviewed (`not_applicable.facets` in the report) and
nothing writes a value into it. A mark on a facet that holds a value is a
warning (`not-applicable-has-value`), and so is one naming a facet the note's
type does not carry (`not-applicable-unknown`). Who writes the mark: the brain
keeper agent, in the answer that assigns the note's facets (at intake, as
`not_applicable`; and in `POST /maintenance/refacet`), and the owner by hand.
**Never a caller agent**: facets belong to second brain alone (Track AQ), and no
route lets a caller write the key. The system only accepts a mark for a facet
the note's type carries and the workspace has registered values for, and drops
it when the same answer gave the facet a value. A long document read in parts
keeps a mark only when every part said it. The facet review does not ask again
about a note whose every facet holds a value or is marked, and the nightly run
does not count it as pending. The key sits with the platform's own bookkeeping
(`PLATFORM_FRONTMATTER`), so no answer shows it as evidence.

**An empty section can be right (AO2).** `sections_not_applicable:
[Experiments]` says the source has nothing for that heading: a stated value,
neither missing nor empty (`not_applicable.sections`). A section marked so and
holding content is a warning (`not-applicable-has-content`). Beside it, a type
declares which of its sections may be empty (`optional_sections`) and where its
notes cite (`citation_sections`, `citation_fields`), in `config/note_types.json`:

```json
{"types": {"recipe": {
  "optional_sections": {"Source": true, "Steps": {"when": {"kind": ["no-cook"]}}},
  "citation_sections": ["Sources"], "citation_fields": ["inspired_by"]
}}}
```

`true` makes the section optional for every note of the type; `when` narrows it
to the notes whose frontmatter field holds one of the values (compared without
case); `false` takes a shipped declaration away; a list of headings is `true`
for each. Anything else is ignored, not half-applied. For such a note an empty
or absent section is a **warning** (`section-optional-empty`), never the
`map-branch-empty` error or a `section-missing`, and the map leaves the branch
out as it does for any empty one. The shipped types declare what is commonly
absent in their kind of document: a meeting's `Decisions`, `Action items` and
`Risks / blockers`, a thread's `Decisions`, `Follow-up items`,
`Disagreements / uncertainty` and `Referenced sources`, a paper's
`Experiments`, a white paper's `Recommendations`, a voice note's `Action items`,
a decision record's `Revisit when`, an AI session's `Corrections` and
`Dead ends`, a chapter's `Worked examples` and `Prerequisites`, a web page's
`Procedures`, an image's `Where it came from` and a reference list's
`About this list`. A section a type does not declare optional stays an error
when empty, unless the note marks it. Mandatory sections can be marked too.

**A citation is not a broken link (AO3).** A link to a work the collection does
not hold, where a note cites, is counted in `cited_not_held`, once per work
however many notes cite it, apart from the problems. Where a note cites is the
type's declaration: `Relation to other work` for every type (it is mandatory),
plus `citation_sections` and the frontmatter fields in `citation_fields` (a
paper's `cites`). A work is held when a note's file name, or any identifier
its type names (a citekey, a DOI, a URL), matches the link without regard to
case. When the work is added the link resolves and the count falls. A link
anywhere else to nothing is still an `unresolved-link`.

Written down once, the lint findings: `section-optional-empty`,
`not-applicable-has-value`, `not-applicable-has-content` and
`not-applicable-unknown` are warnings; the report adds `not_applicable`
(`facets`, `sections`) and `cited_not_held` (`work`, `cited_by`, `count`).
`facet-missing` now says that second brain assigns a value or marks the facet,
and that nothing asks for a value to be invented (Track AQ6).

## Broken characters (Track AT)

Characters a converter lost (the replacement character, a noncharacter, a
private-use code, a control code, a `(cid:N)` token: the list is in
`docs/document_conversion_v1.md`, "Broken characters") are counted per note.
`broken-characters` is a **warning** on a note that holds any, with the count
and, when some of its `## sections` are too broken to embed, how many are kept
out of the text embedded for search by meaning: `N broken character(s) from
conversion; M section(s) kept out of search by meaning`. A section is kept out
when broken characters are at least `max_share_in_search` (0.03) of its
non-space characters (`config/broken_characters.json`, per-workspace override).
The report adds `broken_characters` `{notes, characters, sections_left_out}`,
returned by `GET /maintenance/lint` and the nightly summary like every other
count (a filter never changes it). The note itself is not changed by lint: lost
letter pairs are repaired at conversion and, for notes already held, by the
nightly run, when the same document spells the word intact; the rest stays
counted.

## The owner's section — `## My notes` (Track AH)

A note made from a person's document is written again whenever its source is
processed again, and someone reading it who wants a line beside it ("this
contradicts the other paper", "ask about this") needs a place that survives
that. Every note the pipeline writes therefore carries **one section that is
the owner's**, after the note's content and before the sections the pipeline
keeps for itself:

```markdown
## Provenance
- ...

## My notes
<!-- Yours. Second brain never rewrites this section. -->

## Links
```

- **Declared once.** The heading and the hint line are
  `OWNER_SECTION_HEADING` and `OWNER_SECTION_HINT` in `notes/contract.py`;
  concept notes use the same two constants. An install renames either in
  `config/note_types.json` under `owner_section`
  (`{"heading": "...", "hint": "...", "former": ["My notes"]}`); a note that
  still carries a former name keeps its words under it. Nothing names a
  subject or a language. In its own words it is the one section no template,
  agent or pass is allowed to fill.
- **Written by code alone**, empty, by every writer of a wiki note
  (`WikiNormalizationService.normalize_file`, which every route that makes a
  note passes): the hint line and nothing else. All fourteen templates in
  `format-templates/` carry it (as a form to be filled in), before
  `## Provenance`; whatever an agent or a template put under that heading is
  dropped, never the owner's.
- **Carried forward byte for byte.** `notes/owner_section.carry` takes the
  section from the note being replaced and puts it into the new text. It is
  the one helper the writers share: processing a source again, re-reading it,
  the maintenance refresh and its scheduled job, a governed mutation bundle,
  and `/normalize/file`. The writers that edit a note in place do it around
  the section (`owner_section.outside`): the `## Links` redraw, the
  concept-link rewrite of `## Concepts`, a rename's links (a link the owner
  wrote there is left as they wrote it, and the rename names the note under
  `links_left_in_owner_notes`), the book parts' index, a path repair, the
  glyph repair, the item-level forget (never a line of it), and a table of
  source tables (placed before it). A rollback of a re-read, a correction or a
  source action puts the earlier note back and keeps the section as it is now.
  A restored backup or a restored withdrawal returns the files as they were.
- **A section runs from its `##` heading to the next section the pipeline
  knows**: a `#` heading, or a `##` heading named like a mandatory, derived or
  code-written section (`TL;DR`, `Claims`, `Provenance`, `Links`, `Source
  tables`, ...; `owner_section.ends_at`). Any other heading the owner writes
  inside it, at any level (`## Questions for Marta`), belongs to it, so a person
  who structures their own words never loses the half after a heading. Lines in
  a fenced code block are never headings.
- **A note whose source is removed** keeps the owner's words: before the note
  goes (`POST /review/action` with `decision: remove` on a `source-missing:`
  item), the section is copied as written to
  `maintenance/kept_owner_notes/<note>--<time>.md`, and an open review item
  `owner-notes-kept:<id>` (kind `owner_notes_kept`) names the note and the
  file. A note whose section holds only the hint has nothing to keep. A
  withdrawn note (Track AB4) is moved aside whole, so its section comes back
  with it.
- **Not the source's, never evidence.** Ingestion reads no line of it (no
  claim, concept, relation, table row or equation), for any note that is not
  hand-written; the Note node keeps the words (`metadata.owner_notes`, at most
  4,000 characters) so the note can be found by them. The concept layer reads
  and hashes a source note without it (typing there never makes a source
  pending again, and no agent is shown it). Normalization is not shown the
  previous note's section. The accuracy checker (Track AS) and the drift check
  of the maintenance refresh leave it out; lint reads the note with it
  emptied: never reported as missing, never as empty, and nothing inside it is
  linted. A note written by hand is the owner's whole, and is read as it
  always was.
- **Findable, labelled.** A question using the owner's words finds the note
  (`score_breakdown.found_through: owner_notes`), and an answer lists the words
  apart from the evidence, as the owner's: `from_your_work[]` entries with
  `kind: my_notes`, `maintained_by: owner`, a `label`, a bounded `text`, and
  `read` for the note (`docs/agent_memory_contract_v1.md`). A read of the note
  shows them under their own block, "the owner's own words beside this
  source (not what it says)", and as `owner_notes`; never among the sections
  of the source. The hint line alone is nothing: not embedded, shown nowhere.
  Search by meaning embeds the section's words like any other section's.
- **Notes that already exist** get the empty section only when the owner asks:
  `POST /maintenance/owner_sections` (`docs/api_spec_v1.md`), a dry run by
  default (a real run must say `access_mode: maintenance_write`, else 403
  `maintenance_write_required`), a backup first, idempotent, only notes the pipeline wrote, with
  the exact count of what it changed. Any later rewrite of a note adds it too.

## Derived sections — written by the pipeline

`## Links`, `## Source References`, `## Citations`,
`## Normalization Provenance`. Never hand-edited; regenerated on re-ingest.
None of them is embedded for search by meaning (`retrieval/embeddings.py`,
`sections_of`), and ingestion does not read them.

**`## Links` lists the note's connections** (Track AF, `notes/connections.py`),
written by code from the graph once the note is ingested, with no agent and
no tokens:

- each note it is connected to, both directions, named by the relation as
  read from this note: `- builds on [[file-stem|Title]]`, `- rested on by
  [[...]]`, `- drawn on by [[...]]`; a declared relation is named by the
  install's own vocabulary, an incoming one by its inverse;
- up to five notes sharing at least two of its concepts, most first:
  `- shares 3 concepts with [[file-stem|Title]]`;
- never a note `## Concepts` already links, and never a target with no file.

A note with none says so in one line. When a note is ingested (through
`/sources/process` or `/ingest/file`), the notes it lists are refreshed too,
since theirs may now name it. The rewrite keeps the concept layer's absorbed
state, so no document is read again for it. `scripts/repair_note_links.py`
rewrites existing notes once (a dry run by default; `--apply` keeps a
snapshot). A link in these sections that opens no file is a maintenance
finding, `unresolved_derived_links`.

`## Related Notes` and `## Backlinks` are no longer written: the first only
repeated the note's first concepts as plain text, and the second was never
computed. Both fold into `## Links`, and the repair removes them; their
headings stay in `DERIVED_SECTIONS`, so an older note that still carries them
is read the same way.

## `## Relations` — optional, every type

Where a note says **how** it relates to another note, as against merely
mentioning the same ideas. One bullet per relation:

```markdown
## Relations
- requires [[canal-survey-method]] :: "the survey must be complete and closed first"
- contradicts [[dredging-interval-dissent]]
- builds_on [[silt-load-measurements]] :: "reasons from the measured trend"
```

**The first token is an identifier, not a word.** It must be a relation the
install declares in `config/relations.json` (or a workspace's own copy of it),
read through `second_brain/notes/relations.py` — the single home for relation
names. Nothing parses prose for relation words, so the same line works in any
language, with the evidence quoted in the note's own. A token the install does
not declare produces **no edge** and a `unknown-relation` lint error: a guessed
relation is indistinguishable from an asserted one downstream, so it is worse
than a missing one.

**The evidence after `::` is optional** and is the sentence in this note that
justifies the relation. Surrounding quote marks of any kind are punctuation of
the syntax and are stripped.

**Either end may write the relation.** Each relation has an inverse name, and
writing the inverse stores the same single edge pointing the other way:
`- required_by [[a]]` in note B is stored as `a -requires-> B`, owned by B.
This is what lets an old note gain connections without its own file being
touched, and what stops the order notes arrive in deciding the graph. A
symmetric relation such as `contradicts` reads the same both ways.

**An existing note is never rewritten to receive a relation.** Relations are
read from both ends — `GET /query/relations` reports incoming ones under their
inverse name — so there is nothing to copy into the other note. Writing into a
note whose text does not say it would be invisible write-back, and would
collide with human edits.

**A target that is not ingested yet is allowed.** The edge is honest — this
note names something we have not read — and it resolves by itself when the
target lands, the same rule `supersedes` follows. Lint reports it as
`unresolved-link`, a warning.

**The same relation may also be written as a frontmatter key**, which is how
the pipeline links the pieces of a split document: `part_of: overview-note`,
`follows: previous-section`. Every relation name and every inverse name is
accepted as a key. This is the one place the two spellings mean exactly the
same thing.

`supersedes` and `superseded_by` are **not** relations in this sense. They are
document lifecycle — this note replaced that one — and stay in
`LINK_FRONTMATTER` so that following a `requires` chain never wanders into
replacement history.

## File names (Track AK1)

A name the platform gives a file is plain ASCII, spelled the way people write
it, so it is easy to use in code, on a command line and in a link. This covers
a templated draft and its wiki note, a converted copy, a book's chapter files
and folder, a paper's section files, a fetched video's transcript, an idea, an
owner-work note, a contribution and a concept note (`concept_notes_v1.md`).

- **A letter is spelled out, never dropped.** `Übersicht der Prüfungen`
  becomes `uebersicht-der-pruefungen`, not `bersicht-der-pr-fungen`. The
  shipped spellings: `ä` `ae`, `ö` `oe`, `ü` `ue`, `ß` `ss`, `æ` `ae`, `œ`
  `oe`, `ø` `oe`, `þ` `th`, `ð` `d`, `ı` `i`.
- **Every other Latin letter loses its accents** by Unicode decomposition (`é`
  `e`, `ñ` `n`, `ç` `c`); a Latin letter that does not decompose (`ł`, `đ`,
  `ħ`) is its base letter, read from its Unicode name; a digit of any script
  is its value.
- **A script with no Latin spelling is never guessed at** (Greek, Cyrillic,
  Chinese, Arabic, ...). A name that would lose such a letter is no name at
  all, so a title is never cut down to its Latin part: a note's name falls
  back to its source's own file stem, through the same rule, then to a short
  name from the source's hash (`note-<10 hex digits>`). The title, in its own
  script, stays in the note's `title:` and heading.
- **One table, in one place**: `second_brain/notes/ascii_names.py`
  (`DEFAULT_SPELLINGS`), with the install's `config/naming.json` laid over it
  entry by entry:

  ```json
  {"spellings": {"ö": "o", "α": "alpha"}}
  ```

  A key is one character (any case); a value is lower-case ASCII letters and
  digits, or empty to leave the character out. A bad entry is skipped and the
  rest still apply. The table is the install's, not a workspace's. Nothing in
  code assumes a language.
- **A name kept from a file's own stem** (a Markdown source normalized
  directly) has only its non-ASCII letters spelled; its ASCII characters,
  case and separators are left as they are.
- **Existing files keep their names.** A note's identity is the hash of its
  path, so every step that names a file first looks for the file an earlier
  run gave the same source, under the old rule (which dropped every letter
  outside `a-z0-9`), and keeps it. `POST /rename/ascii` (Track AP6,
  `docs/renames_v1.md`) brings existing names to this rule, dry run first,
  moving every key with each name; `scripts/report_ascii_names.py` lists,
  free and read-only, what it would rename and what to. Any one note is
  renamed by `POST /rename` (`kind: note`), its draft with it.

## Frontmatter

**It must be readable by a standard YAML reader**, not only by second_brain's
own parser. That parser is deliberately small — one `key: value` per line,
`[a, b]` for lists — and it is *forgiving*: it splits on the first colon and
keeps the rest. So the pipeline wrote `title: Chapter 2 — Agents: Identity and
Memory` unquoted, read it back perfectly, and produced a note no YAML
implementation could parse. Six of ten notes in one live workspace were broken
this way, and nothing inside the project noticed, because everything inside the
project used the forgiving reader.

Quoting is decided by the **value**, in one place —
`second_brain/notes/frontmatter_io.py::render_line` — and never per field. A
value is quoted when it contains `: `, ends with a colon, begins with a YAML
indicator character, carries a ` #`, has surrounding whitespace, or reads as a
bare `true`/`no`/`null`. Otherwise it is left bare, because frontmatter is read
by people as well as programs. Lists and mappings are marked `FlowValue` by
whatever rendered them rather than detected from the string, so a title that
genuinely starts with `[` stays a title.

**Line breaks inside a value** (AB4 fix rounds, 2026-10-17). Every character
`str.splitlines` ends a line at (newline, CR, VT, FF, the file, group and
record separators, NEL, U+2028, U+2029) is written as a code-point escape,
`\u` and four hex digits (`\u2028`), inside a quoted value, so no value can
start a frontmatter line of its own. The reader undoes exactly those escapes
and no other: a backslash written by hand stays as written, so
`"C:\new\folder"` and `"D:\archive\Umbrella"` read back literally, and so do
`\n` and a doubled backslash. The reader also ends a line at a newline only.
A value written by an earlier version with `\n` for a newline reads back as
those two characters, as it always did with this reader.

`tests/system/test_notes_are_readable_yaml.py` enforces this against PyYAML — a
real implementation, deliberately not our own reader, since checking our output
with our own parser is exactly what missed the bug.

**Mandatory — every note type**

```yaml
type:            paper | meeting-note | blog-post | chat-transcript | white-paper
                 | youtube-video | book-chapter | ai-conversation | voice-note
                 | decision-record | webpage | documentation | image | reference-list
title:           "..."
source:          path to the raw source
sha256:          hash of the raw bytes, never typed by hand
description:     one or two sentences, written by the agent    # OKF-recommended
status:          draft | stable | deprecated        # OKF lifecycle
reading_status:  skimmed | read | verified          # ours
generated:       {by: <actor>, at: <iso8601>}       # OKF: who produced it
cluster:         one broad grouping
concepts:        [fine-grained, free-growing]
topic:           [1-4, closed vocabulary]
import_date:     YYYY-MM-DD
```

**Why `status` and `reading_status` are two fields.** They answer different
questions — *is this concept current?* versus *how carefully has anyone read
it?* — and they used to be conflated under `status`, holding our reading
values. That collided with OKF v0.2, which reserves `status` for the lifecycle
with its own value space. Same key, incompatible values: a consumer that
defaults an unrecognised lifecycle to `stable` would present an unreviewed note
as settled. `status` now means what OKF says it means; the reading state moved
to the `reading_status` extension. `stable` is the default because a note the
pipeline has produced *is* ready to use — how well-read it is, is the other
field's job. See `okf_alignment_plan_v1.md`.

Both value spaces are enforced: an out-of-range value in either field is a
`frontmatter-invalid-value` lint error, which is stricter than the treatment of
a *missing* value, because a wrong lifecycle is actively misleading where an
absent one is merely incomplete.

**`description` is written by the agent, alongside the TL;DR.** OKF recommends
a one-line summary in the frontmatter so a tool can survey a corpus without
opening every file — and that is a *third* register, distinct from the two the
note body already has:

| field | read by someone who has… | length |
| --- | --- | --- |
| `description` | …not opened the note at all — it is one line in a catalog of hundreds | 1–2 sentences, hard |
| `## TL;DR` | …opened the note, and can afford some context | 1–3 sentences |
| type-specific sections | …the source itself in front of them | as long as the content needs |

Because the registers differ, the extraction prompts ask for `description` as
its own field rather than truncating the TL;DR into it — a catalog line that is
the first N sentences of something written for a different purpose reads as
truncation, trailing caveats and all.

A mechanical summary of the TL;DR remains the **fallback** for when the agent
supplies none, keeping the project's standing rule that every agent path
degrades deterministically. Content that is not prose (a corrupted note whose
`## Summary` holds only `---`) yields no description at all: an absent one is
an honest gap, a garbled one is worse than nothing. Being OKF-*recommended*
rather than required, absent is a lint **warning**.

**Verification is an event list, not a status.** Written when a human resolves
a review (`resolve_review`, the one action in the review vocabulary that means a
person looked and accepted — every other one marks a *problem*, and marking a
problem is not verification). Until 2026-09-16 this whole vocabulary existed and
nothing ever wrote an entry, so the tier below was derived from a permanently
empty list. When a person actually checks
a note, that is recorded as an OKF `verified` entry naming the actor and the
time — `[{by: human:<id>, at: <iso8601>}]` — from which OKF *derives* a trust
tier rather than having anyone declare one: no key at all is `unverified`,
non-human actors only is `machine-confirmed`, any `human:` actor is
`human-reviewed`. Actors follow OKF's convention, which the brain-keeper's own
identity already fits with no translation (`claude_code` running
`claude-sonnet-5` → `claude_code/claude-sonnet-5`); the deterministic fallback
is `process:deterministic-fallback`.

`generated` records the other half — who *produced* the note — in the same
`{by, at}` shape, through the same renderer, so the two trust-family keys can
never drift into different formats for the same kind of fact. It is filled from
the runtime identity the brain-keeper already reports; where extraction fell
back to deterministic heuristics the actor is `process:heuristic-normalization`,
because claiming an agent produced it would misstate provenance in exactly the
field a consumer reads to judge the note.

Notes reviewed before governance auditing existed carry **no** `verified` key
rather than a fabricated actor — `unverified` is the honest derivation, and
inventing `human:unknown` would manufacture a trust signal.

**Frontmatter that names another note becomes an edge.** `supersedes` and
`superseded_by` resolve at ingest to `SUPERSEDES` / `SUPERSEDED_BY` edges
(`LINK_FRONTMATTER`), so a replaced decision points at its replacement and a
query can follow the chain. The edge is emitted whether or not the target has
been ingested: a dangling edge honestly says the replacement is not here yet,
and resolves itself when it lands. `part_of` and `follows` work the same way
(`PART_OF` / `FOLLOWS`): a section note of a long paper split into several
notes names the paper's overview note and the section before it; a book
chapter names the chapter before it.

**Dates a note records about itself are checked.** `review_by` (on
`decision-record`, the type that declares one) and OKF `stale_after` (any type)
are compared as ISO strings by lint, which warns when one has passed and reports
a malformed one as malformed — a date nobody can read is the same as no date.
Both are warnings: an expired decision is a prompt to look, not a broken note.
Together with `status: deprecated` these are why there is no `validity` facet.

**Which keys are OKF's and which are ours** is recorded explicitly in
`second_brain/notes/contract.py` as `OKF_FRONTMATTER` and
`EXTENSION_FRONTMATTER`, rather than left implicit in whatever happens not to
be an OKF key. Our extensions are deliberately **un-namespaced**: the spec's
extension language anticipates bare producer keys and requires consumers to
preserve them, and prefixing every field would make notes worse to read as a
hedge against a collision that has not happened. If OKF ever claims one of
these names, that tuple is the single place a rename starts.

**Type-specific — `type: paper`**

```yaml
citekey:         <surname><year><distinctive-title-word>   # see "Citekeys" below
authors:         [...]
year:            YYYY
venue:           "..."
kind:            journal | conference | workshop | preprint | thesis | book
cites:           [citekeys of referenced work]

# ...plus this type's facets. Which ones, and how many values each takes, are
# declared in config/facets.json rather than here — a paper carries all of
# them by default, and a workspace may add, narrow or remove them. Run
# `GET /ops/facets` to see what an install actually resolves.
```

**Type-specific — the other built-in types**

```yaml
# meeting-note
meeting_date:      YYYY-MM-DD
attendees:         [...]
owner:             who ran it / owns the outcome
project:
decision_scope:    what this meeting was entitled to decide
recording_url:
transcript_source: where the text came from

# chat-transcript
participants:      [...]
channel_or_thread:
platform:          Slack | Telegram | Discord | ...
date_range:
export_source:     how the transcript was obtained

# white-paper
publisher:
organization:
year:              YYYY
publication_date:
url:
document_kind:     white-paper | report | standard-draft | position-paper | technical-report | unknown
audience:          who it was written for

# blog-post
author:
publisher:
publication_date:
url:
site:
retrieved_at:      when it was fetched — a page can change under you
post_kind:         tutorial | opinion | announcement | analysis | case-study | unknown

# webpage
url:               what was asked for
final_url:         where it actually resolved, after redirects
site:              the host it came from
retrieved_at:      when it was fetched — the page can change under you
http_status:
content_type:
author:
publisher:
publication_date:
last_modified:
page_kind:         documentation | reference | product | news | faq | changelog | unknown

# youtube-video
speaker:
channel:
publication_date:
url:
duration:
transcript_source:
transcript_quality: clean | auto-generated | partial | noisy | unknown
```

**The facet set is declared in `config/facets.json`** — which facets exist, how
many values each takes, and what each means. Every module, script and prompt
reads it through `second_brain/notes/facets.py` rather than carrying its own
list; `tests/unit/test_facet_spec.py` fails the build if a second list appears
in code. The listings elsewhere in this document are descriptive, and that file
is what the system actually reads.

**A workspace declares its own** in `sandboxes/<id>/config/facets.json`, layered
over the shipped seven. A facet it does not mention is inherited, `"name": {...}`
adds it or overrides a default's cardinality and definition, and `"name": null`
removes it — removal is stated rather than inferred from absence, because a
workspace file lists only what it changes. The seven that ship are an academic
paper's facets; a collection of legal filings or field records needs its own,
and choosing among seven it did not pick is not the same thing.

Two rules the loader enforces:

- **A facet may not be named after an existing note field.** Facets and ordinary
  frontmatter share one namespace, so a facet called `author` would collide with
  the field recording who wrote the document. Refused, with the reason, and
  reported by `GET /ops/facets`. `model` is grandfathered: it is a facet *and*
  `ai-conversation`'s key for which AI produced a transcript, and the notes
  relying on that are already written.
- **A facet a note still uses cannot be removed.** `POST /maintenance/facets/remove`
  checks the **graph** — what retrieval actually reads — and refuses while any
  note holds a value, naming the notes so there is a way forward. Removing it
  regardless would leave the values in the frontmatter and unreachable from
  search, with nothing reporting it. A removal forced through by editing the file
  directly still shows up, as `orphaned` in `GET /ops/facets`.

**Five facets ship**, and none of them assumes the document argues a case:

| facet | max | captures |
|---|---|---|
| `topic` | 4 | what the document is about |
| `about` | 3 | the named things it concerns — a party, project, place, piece of equipment |
| `function` | 2 | what you *do* with it: look it up, follow it, it binds you, it records, it plans |
| `authority` | 1 | how much weight the source itself carries |
| `locus` | 1 | which part or stage of a larger whole it concerns |

`topic`, `about` and `function` are **mandatory for every ingestion type**: every
document is about a subject, concerns something named, and is used for something.
`authority` and `locus` are per-type, because each is uniformly one value for
some types and so discriminates nothing there — the install *is* the source of
its own meeting notes, so their `authority` is always firsthand.

An **unknown** type still carries `topic` alone. A type nobody has described
cannot be held to facets nobody chose for it, which is the rule that caught a
`blog_post`/`blog-post` spelling slip silently narrowing every type.

**Five facets were retired** in the 2026-09-16 redesign: `contribution`, `model`,
`guarantee`, `technique` and `evaluation`. They were not academic by flavour but
by *assumption* — two assumed the document argues from stated premises, one that
its claims were tested, one that its author declares a confidence level. An
invoice, a wiring schematic and a recipe do none of those, so on any non-research
corpus they sat permanently empty. `docs/facet_set_redesign_v1.md` records why
each went, and why a proposed `validity` facet was dropped as already answered by
`status`, `supersedes`/`superseded_by`, `review_by` and OKF `stale_after`.

Which facets a type is asked for, and judged on, is that type's own `facets` —
declared on its `TypeProfile` beside its sections and frontmatter, and
overridable per workspace in `config/note_types.json`. A type declares them there
and *only* there: listing them in the type's `frontmatter` as well made one
missing facet report twice, once as a `frontmatter-missing` warning and once as a
`facet-missing` error.

They used to be global, and that was wrong in both directions at once. Every
extraction prompt asked for two facets only, so an instance that registered
values for the others could never have them filled; and lint judged every note
against the *whole* registered vocabulary whatever its type, so the moment such
an instance registered them, every meeting note beside the papers became a
permanent `facet-missing` error for lacking something a meeting note does not
have.

**A workspace that places them differently says so.**
`sandboxes/hello_brain/config/note_types.json` declares `topic, about, function,
locus` for its `note`, `meeting-note` and `decision-record` types and seeds values for them — as a
worked example, not a default. Declaring a type that overrides a built-in
inherits that built-in's facets unless it names its own, so adjusting a type's
sections does not silently drop its facets; a declared facet the schema does not
know is ignored, because the rest of the pipeline could not validate it.

Facets are for *finding*; `concepts` is for *connecting*. They are deliberately
not merged: `concepts` grows without bound, which is what makes it useless as an
index — and it is where an install's named parties and projects used to land
before `about` existed.

**How they get filled.** The workspace's registered vocabulary
(`config/facet_vocabulary.json` — the only format second_brain reads) is passed
to the extraction step, which may use only values that appear in it. Anything
else is discarded before it reaches the note, because a vocabulary that accepts
anything is not closed and stops working as an index.

A workspace that has registered no vocabulary is the domain-agnostic default,
not a failure: the facets stay empty, an empty vocabulary permits nothing
rather than everything, and lint reports `topic` as a warning instead of an
error. Once a vocabulary exists, an empty facet becomes a reported gap and an
unregistered value becomes an error.

**Pipeline-written**

`source_type`, `raw_source_path`, `converted_markdown_path`, `template_id`,
`template_version`, `agent_scope`.

**The header: what a caller is given (Track AB6).** A note's *header* is its
frontmatter less the platform's own bookkeeping, and the governed routes hand
it over with the note: the read opens each note with it, a note's entity card
carries it, and an answer names the fields its question names
(`docs/agent_memory_contract_v1.md`). A meeting's attendees, a paper's
authors, a contract's signatories and a recipe's servings are often written
there and nowhere else; before this, no route passed the frontmatter on, and a
caller reading a meeting note through the brain was told nothing of who
attended.

Everything is shown except the keys in `PLATFORM_FRONTMATTER` and those under
`PLATFORM_FRONTMATTER_PREFIXES` (`second_brain/notes/contract.py`). The list is
the platform's own vocabulary, the same for every install, never a corpus's:

| kind | keys |
| --- | --- |
| OKF keys the platform reports in fields of its own (a card's `note_type`, `title`, `summary`; trust signals) | `type`, `title`, `description`, `status`, `reading_status`, `generated`, `verified` |
| where the note came from: identities, hashes, recorded paths | `source`, `sha256`, `source_status`, `source_missing_since`, `source_type`, `raw_source_path`, `converted_markdown_path`, `templated_source_path`, `previous_source_paths`, `text_path`, `rests_on_files` |
| what a fetch recorded about the response (web intake; the page's own `url` and `site` stay) | `final_url`, `http_status`, `content_type`, `retrieved_at` |
| a facet review's date (`POST /maintenance/refacet`), and what a review or the owner said does not apply (Track AO) | `facets_reviewed`, `facets_not_applicable`, `sections_not_applicable` |
| the intake pipeline's bookkeeping | `agent_scope`, `import_date`, `topic_tags`, `identity_terms`, `concepts`, `preprocess_mode`, `selection_mode`, `selected_sections`, `tail`, and every key beginning `template_`, `templated_`, `conversion_`, `fidelity_` or `fallback_` |
| a concept page's upkeep | `concept`, `synthesis`, `definition_basis`, `sources`, `source_count`, `first_seen`, `last_seen`, `revision`, `created`, `updated`, `maintained_by`, `keep_separate`, `redirect_to`, `merged_on`, `merge_id` |
| an owner's registration, a gap proposal's acceptance | `registered`, `heading_parts`, `default_part`, `gap_proposal`, `accepted_on` |
| an accepted contribution's times and conversation (Track AB4) | `proposed_at`, `accepted_at`, `conversation` |
| on the platform's own `agent-contribution` type only (`PLATFORM_FRONTMATTER_BY_TYPE`): the review item's id | `contribution` |

`contribution` is bookkeeping only on a contribution: on any other note it is
a retired facet whose values describe the document, and older notes hold them.
`unknown`, the placeholder the shipped templates write for a field the source
did not give, counts as empty, like `[]` and `-`.

A key outside the list is shown whatever it is called, so a field an install
declares for its own type, or that an owner simply wrote, is never lost. Two
consequences are deliberate: an owner's key that happens to share one of these
names (a hand-written `created:`) is treated as bookkeeping too, and a key the
pipeline starts writing must be added to the list or it is shown. Two tests
hold it: one reads every shipped template's frontmatter, the other every key a
platform writer renders in the package's source (`render_line` calls, a
writer's `front` list, the dicts written through `write_fields` or
`edit_frontmatter_text`; `tests/unit/test_platform_frontmatter_writers.py`),
and each fails on a key that is neither on the list nor a content key the
contract names (AB6 fix round: the first missed a contribution's review item id
and a facet review's date, which reached callers). The type's declared fields come
first (its own, its identifying fields, its facets), the rest in the note's
order. Values are rendered compactly (a list as its items joined by commas,
white space collapsed), each cut at 200 characters, the whole header bounded
(1,200 characters and 24 fields by default; the read gives it at most a sixth
of a note's budget), and what did not fit is named, never dropped in silence.
A mapping (an editor's `- name: Ann` with `affil: X` below) is shown as its
pairs, items joined by semicolons. Each field carries the lines of the note it
was read from, so it is cited as `<note>#frontmatter:<lines>`.

The values of the fields a type declares as **identifying** it
(`identity_fields`: a person, an organisation, a date, an identifier; the
built-in's or the install's own in `config/note_types.json`) are found by
ranking with the note's title, so a question naming one finds the note. They
are recorded on the note's graph node at ingestion (`header_values`), and, for
a note ingested before that, read from its Markdown when the answer runs (the
note index it already keeps, remembered while the graph is unchanged), so an
existing install needs no re-ingestion; nothing is written. They weigh no word
in the collection's statistics, so a question naming none ranks exactly as
before; a word they hold is in the collection for question coverage. Only
those fields: an undeclared field's meaning is unknown, and every value
matching every question would be noise. A field an install wants findable is
declared there.

**Block forms are read** (Track AB6). The reader also takes the block forms
a person writes in an editor: a list as `- item` lines under an empty `key:`;
an item written `- by: someone` with `at: then` indented below it as a
mapping; one level of `name: value` lines indented under an empty `key:` as a
mapping; and a text as `key: |` or `key: >` with its lines indented below.
Keys may hold a hyphen after their first character and letters of any script
(`effective-date`, `größe`). Before, all of these read as an empty value and
such a key was not read at all. Anything deeper (a list inside an item, an
indented comment) is not read, and the key is marked as not read whole
(`frontmatter_io.read_entries`, `complete`).

**A writer never replaces what the reader did not read** (AB6 fix round, R68).
`write_fields` refuses, with the reason, to rewrite a key marked as not read
whole, and leaves the file as it was: a person edits such a key by hand. A key
read whole and written as a block is replaced with its block's lines. The
review found the case this guards: a `verified` list written as a block of
mappings was read as text, so recording a new verification wrote a list
holding only the new event and erased the earlier ones; the mappings are read
now, and the earlier events are kept. The writer still writes the flow forms.

## The canvas each type gets

One note's canvas is read with `GET /view/note_map?path=<note or map>`
(Track AK4); `POST /view/canvas` is the whole collection's.

A canvas is a reading aid, so its *shape* carries the meaning. Each type is
re-read for a different reason, and a single generic grid would serve none of
them well, so each ships its own builder. `scripts/note_maps.py` dispatches on
`type:`; a type an instance invents falls back to `build_generic_maps.py`, which
lays out the mandatory sections every note has.

| `type:` | builder | shape | the question it answers |
|---|---|---|---|
| `paper` | `build_paper_maps.py` | citation map | what did it show, and what cites it |
| `meeting-note` | `build_meeting_maps.py` | outcome board | what did we decide, who owes what, what is in the way |
| `youtube-video` | `build_video_maps.py` | timeline | what happens when, and where do I scrub to |
| `book-chapter` | `build_chapter_maps.py` | learning path | what must I know first, and where does this sit |
| `ai-conversation` | `build_conversation_maps.py` | verification ledger | which of this was actually checked |
| `voice-note` | `build_voice_maps.py` | capture board | what did I say, and what must I do |
| `decision-record` | `build_decision_maps.py` | decision and its lifetime | what did we decide, why, and when to revisit |
| `white-paper` | `build_whitepaper_maps.py` | argument + trust ledger | which claims are actually backed |
| `chat-transcript` | `build_chat_maps.py` | diverge/converge funnel | what settled, and what is still contested |
| `blog-post` | `build_blog_maps.py` | argument + applicability column | does this apply to my case |
| `webpage` | `build_webpage_maps.py` | ordered steps under a volatility band | how do I do it, and is this still true |
| `documentation` | `build_documentation_maps.py` | steps or workings, choices beside, limits above | how do I use it, and where does it stop |
| `reference-list` | `build_reference_maps.py` | index of entries | what does this term mean, as the source puts it |

### The paper map

- **Centre:** the title, every author in `authors:` order, year and venue (an
  unknown year is left out), and the citekey.
- **Right:** one branch per `map_branches` entry of the paper profile: Methods,
  Experiments, Key results, Conclusions, **Limitations** (from
  `## Assumptions & limits`) and **Research questions** (from `## Questions`).
  Only the labels differ from the headings. The headings are the contract and
  `## Questions` stays mandatory for every type.
- **Left:** Concepts, Cites, Cited by, Same author. A card for another paper in
  the collection links that paper's **note file by its exact file stem**, shown
  under its title (`[[note-stem|Title]]`), because Obsidian resolves links by
  file name and a citekey is not a file name. When another note in the vault
  shares that name, the link is the note's path inside the vault instead
  (`[[concepts/widget-store|widget-store]]`), so a click cannot open the wrong
  file. A note at the top of the wiki needs no folder, since Obsidian matches a
  path before a name. A cited work outside the
  collection is written as its citekey in plain text, not as a link that would
  create an empty note. A `cites:` entry may name a citekey or a note's file
  name; both reach the same note.
- **Concept** cards are shown under the concept's name, not its key: the
  concept note's `title:` when it exists, else the label the paper gives the
  concept in its `## Concepts` section, else the key read as words
  (`[[computation-offloading|Computation offloading]]`).
- **Same author** cards name the shared author or authors, on the card and as
  the edge label: `Bob Sample — [[note-stem|Other paper]]`, most shared authors
  first. Matching is exact after Unicode normalisation, case folding and
  whitespace collapsing. `T.-O. Okafor` and `Tomas Okafor` do **not** match.
  That is deliberate: a wrong "same author" link is worse than a missing one.

**Relabelling a branch.** Any type may give a branch a name other than its
heading with `map_labels` in `config/note_types.json`, merged over the
built-in labels:

```json
{"types": {"paper": {"map_labels": {"Assumptions & limits": "Grenzen"}}}}
```

**Where maps live, and which vault opens them.** Maps are written to
`<workspace>/sources/wiki/maps/<note-stem>.canvas`, and the Obsidian vault to
open is **`sources/wiki/`**. It holds the maps and every note they link, and
nothing that competes with them. Do not open the workspace root as the vault.
There, each pipeline note shares its file name with its templated draft under
`sources/templated_markdown/`, so a click could open the draft, and the
concept-history snapshots and converted sources would crowd its search and
graph. Note scans read `*.md`, so the `.canvas` files in the wiki are invisible
to them. `scripts/note_maps.py` owns the location (`maps_dir()`).

Maps were written to `data/exports/maps/` before 2026-10-09. Mapping a note
removes its copy there, and a full build of a workspace's maps (`note_maps.py
--workspace-root <ws> --wiki-dir <ws>/sources/wiki`, with no `--out-dir`, which
now defaults to the wiki) removes them all. An old copy whose note's map now
exists in the wiki is reported as *moved* (`note_map.moved_maps`, and
"moved to sources/wiki/maps/…" on the command line). Only one whose note is
gone is reported as *removed*.

**A map goes when its note goes.** `sources/wiki/maps/.note_maps.json` records
which note drew each canvas, relative to the workspace. Every map build through
`scripts/note_maps.py` (each ingest, and the CLI after a full build) removes a
canvas whose recorded note no longer exists. A canvas with no record (drawn
before the record existed) is removed when no `<stem>.md` exists anywhere under
`sources/wiki/`. A note outside the workspace is recorded as `null` and its map
is never removed, because its note cannot be checked from the workspace. A
note that no longer yields any branch loses its old map. The ingest result
reports removals as `note_map.removed_maps`.

Three layout decisions are load-bearing rather than decorative:

- **The white paper splits claims by whether the note carries evidence for
  them**, into *Backed by something in the note* and *Asserted — nothing
  attached*, and colours the edges feeding the recommendations accordingly. A
  recommendation resting on unbacked claims is then visible without tracing
  anything by hand. The pairing heuristic is deliberately weak and explainable
  (two shared significant words); it judges whether the note attached anything
  at all, never whether the evidence is any good.
- **A video's anchors hang under the segment their timestamp falls inside**,
  ordered by the clock rather than by the order they were written down. A
  timestamp is only useful next to what it points at.
- **A transcript's disagreements sit beside its decisions, not below them.**
  "We argued about this and never resolved it" is as much an outcome as a
  decision, and it is the part that gets lost when a thread is skimmed later.

Colour means the same thing in every builder — the legend lives in
`MEANING` in `scripts/canvas_kit.py`, so a reader learns it once rather than per
type. Shared layout mechanics (cards, groups, edges, text fitting) live in
`canvas_kit.py` and the read/write/CLI plumbing in `canvas_writer.py`, which
keeps each builder about shape and nothing else.

One contract-valid example note per type lives in
`tests/fixtures/note_types/`. They are invented, per the self-containment rule.

## The heading contract

Each builder reads the headings its type declares, so renaming, reordering or
flattening a heading costs whatever the layout built from it. For `paper`, the
bullets under **Methods**, **Experiments**, **Key results**, **Conclusions**,
**Assumptions & limits** (drawn as *Limitations*) and **Questions** (drawn as
*Research questions*) become the branches of its map. One idea per bullet.

`scripts/lint_notes.py` enforces this contract: missing mandatory sections,
missing mandatory frontmatter, unregistered facet values, and broken
`[[wikilinks]]` are errors.

A note with **no frontmatter block at all** is reported once, as
`frontmatter-block-missing`, and the per-key and per-facet checks are skipped
for it. They would all fail for the same single reason, and reporting each one
separately buries the fact that explains them under a dozen restatements of it:
three legacy notes produced 48 errors between them, of which 42 said the same
thing. Sections, wikilinks and placeholders do not depend on frontmatter, so
they are still checked. A block that is present but parses to no keys is
`frontmatter-block-empty` — a different fix from an absent one, so it is not
folded in.

A note with nowhere to put facets needs the block added, which is a
note-contract repair rather than a facet decision — `POST
/maintenance/refacet` reports it and stops there by design.

### Adding a missing frontmatter block

`POST /maintenance/repair_frontmatter` is that repair, and it closes the loop:
run it before refacet and a note that used to be refused with "note has no
frontmatter block" becomes reachable in the same maintenance pass.

Everything it proposes is mechanical — derived only from the note's own text
or a file that provably exists on disk, never invented:

- `title` — the note's own first `# H1` heading; the filename if it has none.
- `type` — whichever registered `TypeProfile` (built-in or from this
  workspace's `config/note_types.json`) has the most of its type-specific
  sections already present in the note's body. Evidenced by the note's own
  headings, never a guess with nothing behind it; falls back to the generic
  type on a tie or no match at all.
- `concepts` — the bullets already sitting under the note's own `## Concepts`
  section, if it has one.
- `source` / `sha256` — filled only when the wiki note's filename correlates
  to a real file under `sources/converted_markdown/`, using the pipeline's own
  `<stem>__templated.md` naming convention (`SourcePipelineService`). Two
  candidates matching the same stripped stem is treated as no correlation
  rather than a guess between them. With nothing to correlate, both stay
  empty — an honest gap, not a fabricated path or a hash of nothing.
- `status` — `skimmed`, the same default the pipeline itself writes for any
  freshly processed note.
- The seven closed facets are left empty. Assigning them is refacet's job, and
  it can only reach the note once this has given it somewhere to write.

No brain-keeper call: nothing here needs judgment beyond what the note's own
text and the files already on disk answer. Proposes by default; `apply: true`
writes, and emits a `repair_frontmatter_apply` governance event. A note whose
block is present but parses to no keys (`frontmatter-block-empty`) is repaired
by replacing that block in place, not by prepending a second one.

### Adding a stub for a missing mandatory section

`POST /maintenance/repair_sections` fixes the sibling failure: a mandatory
section (`## TL;DR`, `## Relation to other work`, ...) the note does not have
at all — `section-missing`, an error. It inserts `## Heading` with nothing
underneath and nothing else, which turns that error into `section-empty` — a
warning, an honest lesser problem, "needs content" rather than "entirely
absent." It never writes a sentence of prose; filling a stub with real content
is a content decision, left to a human or the brain-keeper's own extraction
skills, not this repair.

Placement is mechanical too: a stub goes right before whichever mandatory or
derived section (in the fixed order `MANDATORY_SECTIONS` then
`DERIVED_SECTIONS` are declared in `second_brain/notes/contract.py`) is the
first one already present in the note, so it lands where a reader expects it
rather than merely existing somewhere in the file. With nothing later in that
order present either, it goes at the end.

Type-specific sections (`## Methods`, `## Experiments`, ...) are out of scope
by design — the note contract's own position is that a meeting note is not
missing an `## Experiments` section, it simply does not have one, and stubbing
one would mean guessing at what belongs to a note's type.

No brain-keeper call. Proposes by default; `apply: true` writes and emits a
`repair_sections_apply` governance event.

### Filling in a note's type-specific frontmatter fields

`POST /maintenance/repair_metadata` is the one repair in this family that
genuinely needs judgment. A paper's `authors`/`year`/`venue`/`kind`, a meeting
note's `attendees`/`meeting_date`, a white paper's `publisher` — these are
facts stated somewhere in the source, not structure a heading count can
infer, so this route calls the brain-keeper's `extract_type_metadata` skill
rather than staying pure Python like the two repairs above it.

It is deliberately general, not paper-specific: for **any** registered type —
built-in or declared in a workspace's own `config/note_types.json` — it reads
that type's `TypeProfile.frontmatter` and asks about whatever is declared
there, minus two exclusions:

- the seven closed facets (`topic` and the six type-specific ones) — that
  vocabulary is closed and validated, which stays `POST
  /maintenance/refacet`'s job
- `citekey` — never asked of the agent. It is computed deterministically from
  `authors` + `year` + `title` (`second_brain/notes/citekey.py`), the exact
  same function and collision rule fresh extraction already uses, so a
  citekey means the same thing whichever path produced it. A note needing
  only `citekey` — everything else already known — costs no agent turn at
  all; that stays pure Python.

#### Citekeys

`<surname><year><word>`: the first author's surname and the title word folded
to ASCII (`Müller` → `mueller`, `Étude` → `etude`), the year's digits, and the
first title word that is not a stopword.

- **A part that is not known is left out**, never spelled `unknown`: no year
  gives `doecomputation`, no author gives `2024computation`.
- **A collision takes the next meaningful title word**, so a second
  "Energy …" paper by the same first author in the same year becomes
  `gao2025efficient` rather than `gao2025energya`. A letter suffix is the last
  resort, used only when every title word is already taken.
- **Nothing readable** (no author, no year, and a title with no Latin-script
  word) gives `ref` plus 8 hex digits of a hash of the title, authors and year,
  which is stable for that paper and distinct from others.
- Keys already written are not changed automatically. A citekey is what other
  notes' `cites:` point at, so silently re-deriving it would break those links.
- **Re-keying old placeholders is opt-in:** `scripts/rekey_citekeys.py
  --workspace-root <ws>` prints a plan, and `--apply` writes it. It touches
  only keys the old rule wrote as placeholders: exactly that rule's key for the
  note, containing `unknown`, or followed by one collision letter when the key
  it collided with exists. A key chosen by hand never matches. For each
  re-keyed paper it rewrites:
  - `citekey:` in the wiki note and its templated draft, and the key wherever
    their bodies restate it as a whole word (the draft's `- Citation key:`
    line, which re-normalising would otherwise carry back);
  - every `cites:` entry and every `[[old-key]]` link naming it, anywhere
    under `sources/wiki/` and `sources/templated_markdown/`;
  - a line under the note's `## Provenance`;
  - a JSON line in `maintenance/citekey_rekeys.jsonl`.

  Rebuild the graph and the maps afterwards.

The agent is given the note's own body and, when the pipeline's
`<stem>__templated.md` naming convention correlates the note to a real
`sources/converted_markdown/` file, that file's text too — the closer source,
preferred when both are given. Told explicitly never to invent: a field truly
not stated in either text comes back empty, an honest gap rather than a false
fact sitting in a knowledge base as if verified.

Extra discipline on the write side, matching every other agent-fed maintenance
write in this project: only a field this note's type actually declares is
eligible, and only if it is currently empty — an extraction naming an
undeclared field or a note that was never sent is discarded before it reaches
the note. Proposes by default; `apply: true` writes, and emits a
`repair_metadata_apply` governance event.

**Not wired into nightly maintenance**, unlike the two repairs above it, and
that is a deliberate boundary rather than an oversight. Refacet's "still
unfaceted" state converges as a vocabulary gets applied, but a note whose
metadata is genuinely unrecoverable — no correlated source, a corrupted body —
would stay a repair candidate forever, and asking about it again every single
night is exactly the standing bill "maintenance must never cost more than
operation" exists to prevent. Wiring this in safely needs the same kind of
repeat-suppression fingerprint vocabulary curation already has; that has not
been built.

### Exporting as an OKF bundle

`POST /ops/export/okf` writes this workspace's notes as an Open Knowledge
Format v0.2 bundle under `data/exports/okf/`. It is a **derived copy** — the
canonical notes are never modified — for the reason that decides this whole
design: three of the four changes an OKF reader needs would make our notes
worse, and rewriting `[[wikilinks]]` would break Obsidian navigation outright.

What the copy changes:

| | canonical note | exported concept |
| --- | --- | --- |
| links | `[[note-stem]]` — Obsidian traverses these | `Title` — OKF traverses these |
| paths | `/home/…/paper.pdf` | `file:///home/…/paper.pdf` |
| tags | `topic` + `concepts`, kept separate | `tags`, their union |
| bundle | — | a root `index.md` carrying `okf_version` |

A wikilink pointing at a note not in the bundle is flattened to plain text
rather than emitted as a dead path. The spec forbids *rejecting* a bundle over
a broken link, but knowingly writing one is still worse than not linking — a
reader cannot tell a typo from a note that has not been ingested yet.

`topic` and `concepts` both survive as extensions. OKF has one untyped `tags`
list; our two lists do different jobs (finding vs. connecting), and collapsing
them in the canonical note would lose a distinction that is load-bearing here
and simply absent from the spec.

Every extension key is copied through verbatim — the producer half of the
spec's requirement that consumers preserve unknown keys. A note with no `type`
is skipped and reported rather than exported, since that is OKF's one hard
requirement for a concept document; giving it one is
`POST /maintenance/repair_frontmatter`'s job, not the exporter's.

Like every derived artifact here, the bundle is rebuildable at any time and a
re-export replaces a stale one.

### Asking about the whole corpus

`GET /maintenance/lint` runs the linter across a workspace and returns the
findings. The per-note lint that runs at ingest reports only the note being
ingested, so a note that predates a rule is never re-checked and nothing
surfaces it; this route answers *"what is wrong with this knowledge base"*
rather than *"what was wrong with the thing I just handed you"*.

**Summary-first, because the reader is usually an agent.** Running the linter
costs nothing — it is pure Python and calls no agent — but *reading* it is not
free when the consumer has a context window, and a corpus-wide dump grows with
the vault while the signal in it does not. So the default response is counts
only (~500 bytes regardless of corpus size). Detail arrives when the caller has
said what it is looking for: narrowing by `kind` or `note` turns it on, and
`detail=true|false` overrides either way. `severity` deliberately does not:
halving a report is not saying what you are looking for.

Counts (`by_kind`, `error_count`, `notes_with_problems`) always cover
everything found; `worst_notes` is the 20 worst. The route is read-only: it
reports, and repair stays a deliberate action.

**It says what it left out (Track AO4).** `listed` and `not_listed` count the
findings in `problems` and the ones that are not; `detail_note` is one plain
line saying how to ask for the rest (`detail=true`, `kind=`, `note=`,
`severity=error`). The list is errors first, so a `limit` that cuts it cuts
warnings, never the errors behind them; `limit` is held between 1 and 1000
(zero or less used to give an empty list, or all but the last few). **A filter
narrows the list, never the counts**: `error_count`, `warning_count`, `ok`,
`by_kind`, `worst_notes` and `cited_not_held` describe the whole collection
whatever `severity`, `kind` or `note` say, `matching` is the view's own size,
and `detail_note` says plainly when a filter is applied. A request that names no
workspace is refused (422 `workspace_not_named`); a named workspace with no
notes checked nothing and carries the warning `lint_found_no_notes`, since an
empty list there is not a clean bill of health. The script
(`scripts/lint_notes.py`), the route, the intake result's lint and the nightly
summary all read through the one `lint_notes` function and give the same counts
on the same notes; the script reads the workspace's vocabulary when given none. The answer also carries `cited_not_held` (`works`, `links`,
`most_cited`; with detail, `list`, and `kind=cited_not_held` lists them, a
`note=` filter keeps the works that note cites) and `not_applicable` (see
"What does not apply, what may be empty, what is cited"). Over MCP the route is
not offered (`narrower_read`).

**Which vocabulary it uses.** `config/facet_vocabulary.json`, and nothing else.
second_brain is self-contained: it does not read, or parse the shape of, any
file belonging to another project, so a vocabulary from elsewhere is converted
into this format once rather than carried as a second runtime format forever.
A file that exists but registers nothing counts as absent, since workspace
bootstrap writes an empty `facet_vocabulary.json` and reporting that path would
claim a vocabulary is in use while every facet check is still being skipped.
When none is found the response carries a `no_vocabulary_registered` warning,
because a clean report then means less than it looks like.

**The nightly job reports lint counts too**, and never acts on them. Fixing a
lint finding means writing prose into someone's notes — a content decision, not
an additive one like registering a value or filling an empty facet — so it is
surfaced in the log and left there.

## Keeping the vocabulary healthy

The idea came from the Literature vault; the implementation is second_brain's
own, split along the project's standing line.

**Deterministic** (`scripts/reindex_vocabulary.py`, `POST /maintenance/vocabulary`):
usage counts per value, values a note used that nothing registered,
registered-but-unused values, cardinality violations, and near-duplicate pairs.

Near-duplicates are found by testing *shapes*, not string similarity — one
value containing another, the same tokens reordered, one token set a subset of
the other, or a difference only in plural. A similarity ratio is the wrong
test: it scores `uniprocessor`/`multiprocessor` at 0.85, a false positive since
they are opposites sharing a suffix, while scoring the genuine risk
`energy`/`energy-management` at only 0.52.

**Judgement** (`curate_vocabulary` skill, `curate: true`): whether a flagged
pair is really the same concept, what new value the corpus needs and the
one-line definition it must carry, and which unused values are dead weight
rather than an axis the corpus has not reached yet.

**Applying** (`POST /maintenance/vocabulary/apply`): the only route that
writes, and it writes only what the caller passes in. A value without a
definition is rejected, because a value with no definition is exactly the drift
the vocabulary exists to prevent. Merges and retirements are not accepted here
at all — they change the meaning of notes already using a value, so they stay a
reviewed human edit.

Proposing is never applying: a closed vocabulary anything can extend on its own
is no longer closed, which is precisely what makes it usable as an index.

### Running it nightly without a standing bill

Maintenance is scheduled **per second-brain instance, not per machine**. Each
workspace has its own brain-keeper, its own corpus and its own owner, so
whether it runs — and when, and how large a nightly batch it may take — lives
in that workspace's own `config/maintenance.json`, next to its vocabulary:

```json
{ "enabled": true, "time": "02:00", "refacet_batch": 5, "machine": "<this computer>",
  "ai_tokens_per_night": 0, "equations_minutes": 60 }
```

`ai_tokens_per_night` (Track AG5, D10) is the most the nightly steps that call
the AI program may use in one night; `0` holds them until the owner sets one,
and the free steps run whatever it is. A schedule without the key (written
before it existed) has no limit and its steps run as they did.
`equations_minutes` is the time the night may spend reading unread equations
on this computer (`0` for none). A collection made through `create_collection`
is written with `enabled: true`, `machine` this computer, `ai_tokens_per_night: 0`
and `equations_minutes: 60`, and this computer's scheduler is asked to hold the
job; see `docs/collections_v1.md`.

That config is the source of truth; the crontab is derived from it.

```
scripts/maintenance_cron.py enable  kronnos --at 02:00
scripts/maintenance_cron.py disable kronnos
scripts/maintenance_cron.py status          # configured vs actually installed
scripts/maintenance_cron.py sync            # crontab := what the configs say
```

Each enabled workspace gets one crontab line, with its own `flock` (a shared
lock would let one slow instance suppress every other) and a
`# second-brain-maintenance:<id>` marker. **Only marked lines are ever read,
rewritten or removed** — the crontab belongs to the user and usually holds
unrelated jobs. `disable` keeps the time and batch so re-enabling restores the
same schedule rather than silently reverting to the default.

A workspace with no config is **off** (an older collection is reported by
`check_setup`, and its owner turns the tidy on with `change_setting`): absent configuration must never mean a
neighbour's schedule sweeps it along. The runner checks the flag as well as the
scheduler, because a cron line can outlive the config that created it and a
workspace whose owner turned maintenance off must not keep spending turns.

The rule the runner itself is built around is that **maintenance must never
cost more than operation**, so it is designed not to call the agent:

1. Python's analysis runs first and is free.
2. Nothing flagged → it exits having called nothing.
3. Something flagged → it checks whether *this exact set of flags* was already
   judged. Re-proposing a merge nobody applied buys the same answer at the same
   price, every night, forever, so the flags are fingerprinted in
   `maintenance/vocabulary_state.json` and an unchanged set is skipped. The
   fingerprint covers only unregistered values and near-duplicate pairs, not
   usage counts — ingesting a note that uses registered values raises no
   question and must not look like a reason to re-run the curator.
4. Only the notes the flags actually name are sent. In maintain mode the
   curator gets the notes that used an unregistered value, not a blanket
   sample; near-duplicates and dead values are judged from the vocabulary's own
   definitions and need no note text at all.

**Dead values alone never buy a turn.** A retirement cannot be applied
automatically, so proposing one produces the same unactionable suggestion at
full price indefinitely. They stay in the analysis for a human to read.

**Refaceting is capped** at that workspace's `refacet_batch` per night, so a
backlog drains over several nights rather than in one very large turn. The cap
is what makes the worst case of any single run predictable, and it is per
instance because a workspace that ingests constantly has different needs from
one that sees a paper a month.

Free steps that follow the same rule (a request only when something changed,
nothing spent) include putting back the letter pairs a converter lost in the
notes the pipeline built (Track AT3, `docs/document_conversion_v1.md`), and
re-ingesting the wiki notes that changed.

Before any of that, the job runs frontmatter repair, then section-stub repair,
on every note that needs one, unconditionally — neither has anything to do
with the vocabulary, and refacet can only reach a note once frontmatter repair
has given it somewhere to write. Both are purely mechanical, so unlike
everything below them, they run even when the workspace has no vocabulary yet
and would otherwise report `needs_bootstrap` and stop.

The job may add a missing frontmatter block, stub a missing mandatory section,
register new vocabulary *values*, and facet notes that had none — all
additive, none changes what an existing note means. It will never bootstrap a
vocabulary (the one expensive call, and a first-time judgement about what the
workspace is *for*) and never merge or retire a value. Both are reported and
left for a person. `--dry-run` reports what would happen using Python alone
and makes no agent call, so checking the job is wired up correctly is free.

### Bootstrapping, and the notes that predate the vocabulary

A workspace's first vocabulary has no usage to reason from, so the analysis
report is empty and looks identical to a healthy one. `mode: bootstrap` is
passed to tell the two apart, along with a wider sample of the corpus — on a
cold start the notes themselves are the only signal.

Registering values does not retroactively facet the notes ingested before them,
so immediately after a bootstrap every value reads as *registered but unused*
and faceted retrieval still returns nothing. `POST /maintenance/refacet` closes
that loop: it selects notes whose facets are empty, hands them to the
`assign_facets` skill along with the vocabulary's definitions and each facet's
cardinality, and validates what comes back through the same closed-vocabulary
rules — an unregistered value is discarded rather than written.

It proposes by default; `apply: true` writes the facet keys of the note's
frontmatter in place and emits a `refacet_apply` governance event. Notes that
already carry facets are skipped unless `include_assigned: true`, since
re-deciding them costs an agent turn and risks overwriting a human correction.
An empty assignment is never written: a facet the agent could not place stays
as it was, because a missing facet is recoverable and an erased one is not.

Every note it declined to write comes back in `not_written` with the reason. A
note with no frontmatter block at all is reported rather than repaired — facets
live in frontmatter, so that note fails the contract above and the fix belongs
to the linter, not here. Reporting the refusal matters: silently writing
nothing is indistinguishable from having nothing to write.
