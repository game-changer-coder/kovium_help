---
type: "documentation"
title: "Instance tails v1: one shared head, a tail per instance (Track R)"
description: "Every second brain is built from this repository by someone whose collection has nothing in common with anyone else's: papers, leases, recipes, lab notebooks, minutes in a language nobody here reads. The instructions the repository ships..."
status: "stable"
reading_status: "read"
product: "second brain"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing second brain"
doc_path: "docs/instance_tails_v1.md"
doc_sha256: "95cb009a029c7d32c32fb616af7d770eee05977d62dba2300b90dd6946ce1784"
---

# Instance tails v1: one shared head, a tail per instance (Track R)

Every second brain is built from this repository by someone whose collection
has nothing in common with anyone else's: papers, leases, recipes, lab
notebooks, minutes in a language nobody here reads. The instructions the
repository ships (the **head**) must therefore stay general. A general
instruction reads every new document blind, though: it does not know that the
collection already calls an idea by one name, that its owner writes in German,
or that its documents give weights in grams. Duplicates and inconsistent names
are then cleaned up afterwards, at a cost.

A **tail** is a short, structured, owner-approved description of one instance,
appended to the head for the jobs that benefit. The repository delivers the
mechanism; it never contains a tail.

## What the repository delivers, and what an instance holds

| | The head (in the repository) | The tail (in one instance) |
|---|---|---|
| What | general prompts and skill instructions; the tail schema and its screen (`second_brain/tails/schema.py`); the deterministic builder (`builder.py`); selection (`registry.py`); the `draft_tail` skill; the maintenance job (`service.py`); the `/tails` routes; the defaults in `config/tails.json`; the evaluation `scripts/eval_tail.py` | one or more tails, each a sequence of versions; the instance's own `config/tails.json` overrides; the maintenance state |
| Where | the checkout | `<instance>/maintenance/tails/<tail_id>/v<N>.json`, `<instance>/maintenance/tails/state.json`, `<instance>/config/tails.json` |
| Committed | yes | **never** (`.gitignore` ignores every `maintenance/tails/`; `tests/system/test_instance_tails_stay_in_instances.py` fails on any tracked tail) |
| Rebuildable | n/a | yes: the derived part from the instance's notes, concept layer and facet vocabulary with no agent; the descriptive part by drafting again. Only the owner's approval decisions are state |
| Names a subject | never (the domain-neutral guard scans it) | always: it is that instance's description |

## A tail is structured, short and safe

Fixed fields, each capped (`schema.CAPS`); anything else is refused.

| Field | What | Cap | Written by |
|---|---|---|---|
| `about` | what the collection is, two or three sentences | 480 chars | agent or owner |
| `languages` | languages the notes are written in | 6 × 40 | notes' `language`/`lang`/`languages`, else agent or owner |
| `scripts` | writing systems of the notes' letters | 6 × 40 | builder (Unicode) |
| `glossary` | recurring concepts, by preferred name, with the other names they appear under | 40 entries, 80 chars, 6 variants, 10 words a name | builder (concept layer: labels, and those aliases, merged names and marked translations that are name-equivalents; else the notes' `## Concepts`) |
| `facets` | registered facet values | 8 facets × 12 values | builder (`config/facet_vocabulary.json`) |
| `note_types` | note types (counts kept in the record for drift, never rendered) | 12 | builder |
| `conventions` | notation, units, naming that recur | 8 × 200 | agent or owner |
| `owner_focus` | what the owner's own work concentrates on | 320 chars | agent or owner |

**It describes; it never commands.** Every value is one line. Refused in every
field: hidden or control characters, anything the injection scanner calls
instruction-shaped (`security/injection.py`, visible or not), tags, role
labels, delimiters and fences, links and addresses, second-person address. In
the prose fields also refused: sentence-initial directives (`Always ...`,
`Use ...`, `Do not ...`) and obligations (`must`, `should`, ...). A name may
start like a verb (`Use case analysis`) but may not read as a command.

**Rendered, it fits a budget.** The tail is rendered under a header that says
it is reference information derived from the collection and approved by its
owner, and that the job's instructions win wherever it seems to disagree. The
rendering is estimated conservatively (four ASCII characters a token, every
other character one) and refused over `token_budget` (default 700, at most
2,000). The builder trims derived fields, least used first, to fit; it never
cuts prose.

**Only other names are variants (R-R1).** A concept note's `aliases:` can hold
things an earlier curation filed there that are not names for it: an
instance, a subtype, a method, another concept's abbreviation. A glossary that
calls them "other names" would teach extraction to merge distinct things, so a
variant is kept only when it is shown to be a name-equivalent, by tests that
are typographic and structural and hold in any language
(`second_brain/tails/equivalence.py`):

- **spelling**: the same letters once case, punctuation, hyphens, spacing and
  accents are set aside, or one letter apart in a name of six or more;
- **abbreviation**: the initials of the name's words (all, or those longer
  than three letters), or those initials in order with further letters found
  in order in the name (a compound's inner word), or the name's own trailing
  parenthetical; or the reverse when the name is the abbreviation;
- **word forms**: the same words in order, each pair sharing a start and
  differing by a short ending;
- **abbreviated run** (R-R6): an abbreviation of at least two of the name's
  leading or trailing words, the other words written out as in the name or
  left off; left off, the acronym needs three letters or more (`PCR` and
  `PCR test` for `Polymerase chain reaction test`);
- **inner words left off** (R-R6): the name's first and last word kept and up
  to two inner words dropped, a hyphenated compound counting as one word
  (`laminated dough` for `Laminated yeasted dough`);
- **contained**: the name with one leading or trailing word left off. A
  variant that adds a word to the name is never accepted: that is how a
  subtype reads (`Croissant laminated yeasted dough`);
- **translation**: listed under the concept note's `translations:`, never
  inferred.

Always dropped, whatever the test: a variant that is another concept's name or
key, and one several concepts list unless it is the equivalent of exactly one
of them (it stays with that one). Every drop is reported in the draft's
`dropped` with its reason. The draft also raises **one** review item,
`concept-alias-doubts:<tail_id>@v<N>` (R-R2, R-R5), listing the whole
concept layer's doubtful aliases: conflicts first (an alias that is another
concept's name, or is another name for another concept), then the aliases not
shown to be other names, with a compact table. The next draft supersedes it,
and also every per-concept `concept-alias-doubt:<concept>` item an earlier
version raised: each is closed with a note naming the item that replaced it.
Concept notes are never edited.

**No counts in the text (R-R3).** Counts change with every intake, and a
changed text is a changed fingerprint and new cache keys for no change in
meaning. The rendering names note types and concepts without counts and in
alphabetical order; the drafting agent is given orders, never counts, and is
told not to state any. Counts stay in the record and drive drift, which counts
membership (concepts, types, facet values), not how many documents hold them.

Every version has an id, a version number and a **fingerprint** (a hash of the
rendered text only, so the same content is the same fingerprint whatever its
id or version).

## How a tail is drafted, approved and used

1. **Draft** (`POST /tails/draft`, `maintenance_write`). The builder makes the
   derived part from the instance's data, with no agent: this alone is a useful
   tail for an instance with no agent. The descriptive part comes from, in
   order: fields the owner writes in the request (refused with 422 and the
   reasons if any fails the screen); the `draft_tail` skill on a digest of the
   instance (concept names and counts, facet values, note types, a sample of
   titles, owner-work kinds), only if an agent is ready and the call's
   estimate fits `max_tokens_per_refresh`; otherwise the approved version's
   own wording. An agent's sentence that fails the screen is dropped and
   reported, never passed on. The draft is saved as a new version and raised
   as the review item `tail-draft:<id>@v<N>`, carrying the rendered text.
2. **Approve** (`POST /tails/approve`, or resolve the review item through
   `POST /review/action`; dismissing it rejects the draft). The version is
   screened again (a file edited by hand is caught here), its fingerprint
   recomputed, the tail's earlier version superseded. Approving calls no agent
   and touches no note.
3. **Use.** Only while the instance's `config/tails.json` says `enabled: true`
   (off by default), only for skills that declare `accepts_tail`, only an
   approved version. Retire one with `POST /tails/retire`.

## Several tails, and the right one for each job

A tail's `applies_to` names the work it is for:

- `note_types` (`recipe`, `minutes`), `template_ids` (format templates),
- `source_paths`: workspace-relative folders (`sources/raw/contracts/`) or a
  declared source root (`library:`, `library:papers/`), spelled as the instance
  records sources,
- `facets`: `{facet: [values]}`, any listed value of each facet.

A tail matches when every key it sets matches; an empty `applies_to` is the
default and matches every job. Of the matching approved tails the most specific
wins: more keys set, then the more concrete key (source path, then template,
then note type, then facets), then the longer source path. Two approved tails
may not have the same `applies_to` (approval is refused with 409), so the
choice never depends on file order. Nothing matching means the head alone.
`GET /tails/preview` with a job's selectors shows the choice and why.

The selectors come from the work in progress: the source pipeline sets the
template, its note type and the original's location for every job of a source
(a book's chapters and a long paper's sections are scoped by the original's
location, not the folder their split-out files went to); the normalizer adds
the note's type, template and facets from its frontmatter. They travel in a
context variable and are carried into the batch pools.

## Which skills take a tail, and why

| Skill | Takes a tail | Why |
|---|---|---|
| `extract_template_draft`, `extract_academic_paper` | yes | they choose a source's concept names and facets: the preferred names and their variants are what stop a second spelling entering the collection, and languages and conventions shape what is extracted |
| `normalize_note` | yes | the same choice for notes written natively |
| `label_clusters` | yes | one call per labelling run; themes in the collection's own vocabulary and language |
| `shape_template_body`, `shape_templated_wiki` | no | they reshape a draft whose names were already chosen and must add nothing; a tail would be paid on every document for little |
| `curate_concepts`, `merge_concepts`, `classify_duplicates`, `consolidate_concept` | no | the glossary is derived from the concept layer itself; feeding it back into the judgement that maintains that layer is circular |
| `draft_tail` | no | circular |
| everything else (answers, gaps, figures, equations, audits) | no, for now | no measured benefit yet; `read_equations` is the next candidate (notation conventions) |

A skill opts in with `"accepts_tail": true` in its definition. The tail is
appended last to the job's system text, so the skill's own prefix stays
byte-identical and a tail is one more cacheable prefix.

## What a tail costs and what a new version re-reads

- Every job that takes a tail pays its rendered tokens (the budget caps it).
- The **job record** says which tail it was given: `tail: {tail_id, version,
  fingerprint}`, and its spec holds the exact text.
- The **note** says so in its provenance: `tail: <id>@v<N>` for a note
  normalized natively, `template_tail: <id>@v<N>` for an imported source (in
  the templated draft's frontmatter, carried to the wiki note).
- The tail's fingerprint is part of the **answer cache key** and of the
  **normalization batch cache's prompt hash**, only when a tail is carried:
  keys of jobs without one are unchanged. So approving a new version re-reads
  nothing by itself; a source is extracted under it when it is next processed
  (a refresh, a re-read, a re-run of intake), once, as with a prompt change
  (`docs/prompt_system_v1.md`). The concept names extraction is shown
  (`known_concepts`) are not in the key (Track S): only the tail and the
  material decide whether an unchanged source is read again.

## Maintenance on a configurable schedule

`config/tails.json` (install default in the checkout; an instance overrides
any key in its own):

| Key | Default | Meaning |
|---|---|---|
| `enabled` | `false` | jobs are given approved tails; the maintenance job runs |
| `cadence` | `weekly` | `nightly`, `weekly` or `off` for the maintenance job |
| `drift_threshold` | `0.2` | share of the basis that must have changed before a refresh is proposed |
| `max_tokens_per_refresh` | `8000` | the most one drafting call may cost, estimated before it is made; 0 means deterministic refreshes only |
| `token_budget` | `700` | the most a rendered tail may cost on every job |
| `glossary_size` | `30` | concepts the glossary is chosen from |

`POST /tails/maintain` (run by `scripts/nightly_maintenance.py` while
`enabled` and cadence is not `off`; skipped silently and without a request
otherwise) checks each tail on its cadence (nightly: 20 hours after the last
check; weekly: six and a half days). It compares the approved version's
**basis** (the concepts its glossary was chosen from, the note types, the
facet values) with the instance now: new concepts among the top ones, concepts
gone (renamed, merged), new note types, new facet values. Past the threshold
it drafts a refresh (deterministic, plus the agent within the cap) and raises
it for review. It never approves; while a draft awaits review it proposes
nothing more. An enabled instance with no tail at all is proposed a first
draft. A dry run (the route's default) measures and writes nothing.

## Measured before trusted

`scripts/eval_tail.py` is run by an operator, never by the suite, because it
spends the subscription: two calls per sampled source. It extracts the same
small sample with the head alone and with the tail (by default the approved
default tail, else the pending draft, else one built on the spot) and reports
concept reuse, naming consistency, duplicate proposals and billed tokens of
each, with each source's own concepts held out. `--dry-run` prints the plan,
the tail and an estimate and calls nothing. If the tail does not help, leave
`enabled` off.

```
python scripts/eval_tail.py --instance-root sandboxes/<id> --dry-run
python scripts/eval_tail.py --instance-root sandboxes/<id> --sample 3 --yes
```

## Limitations

- The directive and injection checks are English. Structure (one line, no
  markup, no links, caps), the header's framing, derivation from the
  instance's own data and the owner's approval are the defence in every
  language.
- Languages are taken from what notes declare; two languages in one script are
  not told apart without a declaration or an agent.
- The glossary favours concepts many sources name; a new instance with few
  notes gets a thin tail until it grows.
- The equivalence tests are structural. A true synonym with no shared letters
  (another word for the same thing) is dropped unless the owner lists it under
  `translations:`; a compound abbreviated by its inner word only, or a short
  form of a one-word name, may be missed. Dropping is the safe error: a
  missing variant costs a spelling, a wrong one merges two things.
- The glossary is chosen from the concepts most sources name; a concept
  moving across that boundary counts as drift.
- **Fixed in Track S (was R-R4).** The extraction answer cache used to be
  keyed on the job's exact input, and template and paper extraction send
  `known_concepts` (up to 200 concept labels from the graph, ranked by how
  many notes mention each). Every ingest that added a concept or changed that
  ranking changed the input, so processing an unchanged source again after
  the collection grew extracted it again, whatever the tail: two runs of one
  source with an ingest between them made two extraction calls instead of
  one. Both extraction skills now declare `known_concepts` advisory
  (`advisory_input_fields`), so it is left out of the key; the tail
  fingerprint stays in, and a new tail version still re-reads a source the
  next time it is processed. A reused answer's concept names are mapped onto
  the current concept layer before it is written, aliases screened by this
  track's equivalence test (`second_brain/concepts/current_names.py`). See
  `docs/prompt_system_v1.md`, "What changing a prompt costs".
