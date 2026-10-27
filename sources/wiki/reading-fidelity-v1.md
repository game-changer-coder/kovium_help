---
type: "documentation"
title: "Reading fidelity v1 (Track L)"
description: "Found on 2026-10-09 by the first external caller, which read a source's full text outside the brain and compared it with the source's note. The note was right on the method and wrong or incomplete on numbers: a value from a table garbled..."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing Kovium"
doc_path: "docs/reading_fidelity_v1.md"
doc_sha256: "9fce3cc536eb5ffc11e0331a62c5422786b9d74255017de38475b626f8f979ef"
---

# Reading fidelity v1 (Track L)

Found on 2026-10-09 by the first external caller, which read a source's full
text outside the brain and compared it with the source's note. The note was
right on the method and wrong or incomplete on numbers: a value from a table
garbled by two-column extraction was given to the wrong row; two phrasings of
one result ("3x faster", "70% faster") were recorded as a contradiction
although they agree; a cost the source defines from several components was
summarised as one. The caller had to leave the brain to read the source, and
its corrections had nowhere to go.

Every check is deterministic, costs no tokens, and is the same in every
language and every field: tables in a lab report, a financial filing and a
recipe book are garbled the same way. One step is not free and is the only one
that calls a model: since Track AS the brain keeper reads a statement-level
finding against its source before a person sees it, within the collection's
standing token budget (*A finding is read before a person sees it*, below).

Code: `second_brain/fidelity/`. Settings: `config/reading_fidelity.json`, with
a workspace's own `<workspace>/config/reading_fidelity.json` taking precedence.

## L1. Reading a source in full: `GET /review/source`

`line_start` + `line_end` returns that range of the source's text form, up to a
per-call cap, with:

| field | meaning |
|---|---|
| `line_start`, `line_end` | the lines returned (1-based, inclusive) |
| `snippet` | the text of those lines |
| `total_lines` | how many lines the text form has |
| `next_line_start` | where the next page starts; `null` at the end |
| `truncated` | the cap cut the range short (`requested` says what was asked) |
| `max_lines_per_call`, `max_chars_per_call` | the caps (`source_page_max_lines`, default 500; `source_page_max_chars`, default 200000) |
| `served_from` | `original`, or `converted` / `templated` for an original that is not text (`source_path` then names the original) |

A caller reads a whole source by asking from `next_line_start` until it is
null. A range ending before it starts is 422. Everything else is unchanged from
Track I3/J-R1: confined to the workspace, its declared source roots, its
registered sources and the notes its graph ingested (403 otherwise, before the
file is looked at); a binary original with no text form is 415; every call is
audited as `maintenance_source_evidence`, now with `line_end`, `total_lines` and
`served_from` in its details.

**Backward compatible.** A call with `line_start` alone keeps the old window
(two lines either side) and with neither keeps the first eight lines; both gain
`total_lines` and `next_line_start`.

## L2. Values from a damaged layout are marked, not stated

### What counts as layout lost (`fidelity/layout.py`)

Structural signals in a source's text form, never words:

| region | signal | read in |
|---|---|---|
| `shredded_table` | pipe rows with no separator row, or mostly empty cells, or an unnamed header | any text |
| `ragged_table` | a pipe table whose rows do not have the header's number of cells | any text |
| `flattened_table` | three or more consecutive lines (`flattened_table_min_lines`) each with two or more numbers, numbers a third of its tokens, no sentence-final punctuation | a converted text form only |
| `cells_one_per_line` | a run of short lines, at least half of them a bare number | a converted text form only |

The last two are read only where the source was not text to begin with: in a
note someone typed, lines of numbers are just lines of numbers. A well-formed
pipe table is the one shape whose values can be put in a row and column, and is
reported as clean.

### The checks a reader applies (`fidelity/checks.py`, `tables.py`, `units.py`)

For a note's content sections (bookkeeping sections such as `## Provenance`,
`## Links`, `## Citations` are skipped):

| check | meaning | severity |
|---|---|---|
| `value_from_layout_lost_text` | a value the note states occurs in the source only inside a layout-lost region | warning |
| `value_in_another_row` | a table cell whose value the source's own clean table prints in another row, never beside this row's label (which the source does have) | warning |
| `row_sum` | a column that is, in most rows, the sum of a set of other columns, and is not in this row. Recognised by arithmetic, never by its header; at least two rows and two thirds of the complete rows must agree before a column counts as a total | warning |
| `column_sum` | the same for a total row, in at least two columns | warning |
| `unit_header` | a column header's unit does not fit the quantity the header names (`Energy (mW)`) | warning |
| `unit_cell` | a cell's unit does not fit its header's unit or quantity | warning |
| `unit_mixed` | one column's cells carry units of different dimensions | info |
| `unit_quantity_mismatch` | a value whose unit does not fit the quantity word nearest before it ("an energy of 5 mW per bit"); see *Which quantity a value belongs to* | warning |

**Three things that are not a measurement (Track AS).** The first rule for
attributing a value still produced false alarms on a real collection: 7 of 11
accuracy findings, checked by hand, were not inconsistencies. The three generic
patterns behind them are now part of the reading, each decided by structure
where structure exists and by a word list, per language and per install, where
it does not:

- **A condition is not the quantity (AS1).** In "15 % of the energy at 100 MHz",
  100 MHz is the frequency the energy was measured at, not the energy. A value
  standing right after a condition word (`condition_words`: `at`, `with`, `on`,
  `under`, `per`, `for`, `during`, `while`, `when`, and their translations) is
  skipped when that word follows the quantity word within `condition_reach`
  words (default 1: "energy at", "energy saved at"). "Energy use stayed at 4 mW"
  has two words between, and 4 mW is the level the energy stayed at, so it is
  still judged: a word list cannot tell a verb from a noun in every language, so
  the reach is the setting, and a doubtful case is left to the brain keeper's
  reading (below) rather than to a guess. A percentage (a measured value with no
  unit word) ends the stretch a quantity word reaches, as a measured value with
  a unit does: "energy saving of 15 % at 100 MHz" attributes nothing to 100 MHz.
- **A number inside a name is not a measurement (AS2).** `Board-10W`,
  `Core-9900` and `Net-56` name things. A number joined to a word by a hyphen
  (or an en dash, or a minus sign) is part of a name; one written against
  letters with no hyphen (`Board10W`, `H2O`) always was. A range is not a name:
  `10 ms-20 ms` and `3x-5x` stay two values, because the word before the hyphen
  is the unit of a value just before it. A unit that opens a compound word
  (`10W-class`) names a kind of thing and is not read as a unit.
- **A label is not a figure, a method is not a conflict (AS3).** `Lemma 4`,
  `Theorem 2`, `Table 3`, `Figure 7`, `Section 3.2.1`, `Equation (12)` and
  `Algorithm 2` number a label: the number is left out of conflict figures, out
  of the values traced into a source, and out of the unit check, and a statement
  whose only numbers are labels raises nothing (`label_cues`). "Proof by
  contradiction" and "contradiction argument" name a method: a conflict cue
  inside one of `method_phrases` does not make the statement claim a conflict;
  a real conflict stated beside it still does.

**Lines the platform wrote are not statements.** The `## Source equations`
catalogue, and any line carrying the platform's own mark for an equation it could
not read or did read (`[equation not recovered ...]`, an equation reading's
provenance, a `formula-not-decoded` placeholder), hold the source's own words
around an equation, not a statement the note makes. They are recognised by the
platform's marks and section, never by an English phrase, and are not read in any
language.

**Units in plain words.** A sentence says what a unit measures in words a person
uses ("data size", "energy per time"), never `bit^1`. A unit that measures
something no quantity word of the install names (a memory size, a count) is
almost never a slip between quantities the install does name; after a condition
word it is a configuration ("the runtime on a node with 128 GB"), so it is
skipped however many words stand between the quantity and the condition word.

Values worth tracing are decimals and numbers of two digits or more; page,
section and reference numbers (`p. 12`, `§3.2`, `[12]`) are not values. Numbers
keep their precision (`3` covers 2.5-3.5; a round `70` may be a rounding to the
nearest ten); a decimal comma is read as one, and `1,234` keeps both readings.
Whole numbers in a sum are added exactly; decimals carry the rounding their last
digit allows.

Units reduce to dimensions over energy, time, information, length, mass,
voltage and current, with SI prefixes. A per-item denominator (`/bit`, `/B`) is
set aside: energy per bit is still energy, so `J/bit` fits "energy" and `W/bit`
does not. A unit outside this table is never judged.

**Which quantity a value belongs to (L-R1).** A value is attributed to one
quantity word: the nearest one *before* it, within eight words, with no clause
or bracket punctuation (`;`, `:`, brackets, a sentence end, a dash between
spaces) and no other measured value between them. A comma does not break the
stretch, so a list keeps its head ("frequencies 33, 100 and 266 MHz"). Only when
that word's dimension and the unit's disagree is the value flagged. A clause
naming several quantities, a quantity after the value, or none near it proves
nothing and is left alone. The first rule judged a unit against every quantity
its clause named and was wrong four times out of four on a real collection
("near the 350 ms limit and power near the budget"; "energy ... (running at 33
MHz)"; "frequencies ... MHz with power ..."); those shapes are now regression
tests, with precision 1.0 on them and on the true mismatches. Missing a
mismatch is chosen over a false alarm.

**The word lists, and why.** No structure says which physical quantity a word
names, that a statement calls two figures inconsistent, that a word introduces a
condition, that a word turns a number into a label, or that a phrase names a
method. `quantities` maps words to units, `conflict_cues` lists stems that claim
a conflict, `condition_words`, `label_cues` and `method_phrases` list the
others; all ship with a multilingual default (English, German, French, Spanish,
Italian, Portuguese, Dutch) and are meant to be extended by a workspace writing
in another language. Every other check is structural. A workspace's own list
replaces the shipped one, so it lists every word it wants.

### At extraction (`fidelity/marking.py`)

`POST /sources/process` checks the note it has just written against the
source's text form, before concept links and the graph are built from it:

- a statement whose value was found only in layout-lost text ends with
  `[unverified: layout lost]`;
- a table cell gets `[unverified]` after its value;
- a table value the source prints in another row gets
  `[unverified: source row differs]`;
- the frontmatter records `fidelity_unverified_values` and
  `fidelity_layout_lost_blocks`;
- the result carries `fidelity` (`checked`, `layout_lost_blocks`,
  `unverified_values_marked`, `findings`, `review_items`, and `judging` when
  statement-level findings were read against the source, Track AS4), and a
  `fidelity:` review item is opened for whatever else the checks find.

This is part of writing the note: the marks are in the Markdown, visible, and
the note stays the source of truth. A note maintained by its owner is never
marked. Marking is idempotent. A check that fails never costs the note
(`fidelity.reason: fidelity_check_failed: ...`).

The extraction instructions say the same to the agent: the academic-paper and
note-normalization prompts and the `extract_academic_paper`,
`extract_template_draft` and `normalize_note` skills tell it never to assign a
value to a row it cannot pair unambiguously, to end a value from damaged layout
with `[unverified: layout lost]`, not to rebuild a damaged table as a clean one,
to check totals and units, and to keep a quantity's components. The wiki-shape
prompts and skills (`shape_templated_wiki`, `shape_template_body`) keep a mark
where it is.

### The fidelity pass over existing notes: `POST /maintenance/fidelity`

```json
{"agent_id": "...", "workspace_id": "...", "dry_run": true, "notes": ["optional", "keys"]}
```

Every source note (never a concept note, a map or an idea) is checked against
its source's own words: the original when it is text, else its converted copy.
The templated draft is never used, since it is the extraction being checked. A
note with no source text gets only the note-internal checks (sums, units,
conflicts).

- **A dry run** (the default) reports: per note `findings` (each with its
  `plain` sentence), `counts`, `layout` (the source's layout-lost regions) and
  `unverified_lines`; totals; `judging` (`state: dry_run`, `enabled`, `would_read`, `remembered`); `cost:
  {agent_calls: 0, billed_tokens: 0}`. It writes nothing and spends nothing.
- **A real run** needs `access_mode: maintenance_write` (403 otherwise, audited
  as denied). It writes one review item per note with findings and the report
  `maintenance/fidelity/report.json`, and never a note, whoever maintains it.
  Before it does, the brain keeper reads the statement-level findings against
  the source, within the standing budget (below); `cost` is then what that
  spent, and `judging` and `judged_false` say what came of it.

The review item `fidelity:<note>:<hash>` (`kind: fidelity`) lists the findings
(at most 50; `finding_count` says how many), names the note and its source, and
offers:

- `correct_one_statement`: file a correction (L4);
- `re_extract`: `POST /review/action` with `request_renormalization` and the
  `source_path`, which re-reads the source through the agent and **costs
  tokens**: offered, never triggered, and the owner's choice;
- for an owner-maintained note, only `owner_maintained`: nothing will change it.

Resolving acknowledges; dismissing is remembered per finding, so the same
findings are never raised again for that note. When a note's findings change,
the previous open item is resolved as superseded. Audited as
`maintenance_fidelity_pass`.

### In the governed answer

Additive to contract v2 (`docs/agent_memory_contract_v1.md`): an item whose
line carries an `[unverified: ...]` mark, or that the last fidelity pass listed
in `unverified_lines`, gets `unverified: true`, `unverified_reason`
(`marked_in_note`, `layout_lost`, `source_row_differs`), `confidence.unverified:
true`, and the provenance reference `unverified: true`; the answer gains the
warning `unverified_values` with a sentence in `warning_details`. Nothing is
dropped.

## L3. A numeric conflict is checked before it is reported

Whether two statements of one change agree (`fidelity/arithmetic.py`). Figures
are a number with `x`/`×`/`fold` (`ratio_suffixes`), with `%`, or with a unit.
Two figures agree if any reading makes them agree, each with the rounding its
own precision allows plus `tolerance_points` (default 2 percentage points):

| reading | relation |
|---|---|
| `reduction` | percentage = 1 - 1/ratio (time or size saved) |
| `increase` | percentage = ratio - 1 (a gain stated as a percentage) |
| `fraction` | percentage = 1/ratio ("takes 30% of the time") |
| `multiple` | percentage = ratio ("300% of") |
| `ratio` | ratio = the quotient of two measurements in one unit |

A ratio is a number with a sign (`3x`, `3×`) or a word (`3 times`, `3-fold`,
and in the shipped settings `fach`, `mal`, `fois`, `veces`, `volte`, `vezes`,
`keer`, `maal`), words in any case. A ratio below one is the same change
inverted (`0.3x the time`). `3x3` is not a ratio. A statement that claims a
conflict (`conflict_cues`) is judged:

- `conflict_not_borne_out` (warning): the figures agree under a named reading.
  "3x faster" and "70% less time" are one result (3x is 67% less time).
- `possible_inconsistency` (info): no reading reconciles them; both numbers are
  reported, since they may describe two different results.
- `conflict_unchecked` (info): the statement claims a conflict but its figures
  cannot be compared (no ratio, percentage or measurement pair); the numbers it
  states are listed. A claimed conflict is never passed over in silence (L-R2).

For a ratio against a percentage, each comparison lists every reading in
`readings` (`means`, the ratio it implies, whether it agrees), and the finding
names the ones that reconcile them in `reconciled_if`. "3 times faster" against
"70% faster" is `conflict_not_borne_out`, `reconciled_if: ["70% less time or
size"]`; read as a 70% gain it would be 1.7x, which is named as not agreeing.
The ambiguity of "70% faster" is the point: the note should say which it means,
not that the source contradicts itself.

The extraction instructions carry the same rule.

## A finding is read before a person sees it (Track AS4)

The free checks are rules, and rules misread. What the caller agent once did by
hand, checking every finding against the source, the brain keeper now does
first (skill `judge_fidelity_finding`, its instructions in
`prompts/prompt_judge_fidelity_finding.md`, through `try_skill`: the
subscription's agent CLI, never a metered API).

- **Which findings.** The statement-level ones, `judge_checks` (default
  `unit_quantity_mismatch`, `conflict_not_borne_out`, `possible_inconsistency`,
  `conflict_unchecked`). Table sums, row sources and layout loss are arithmetic
  and structure; there is nothing to read.
- **What it reads.** Only the lines of the source that hold the finding's
  figures, with two lines of context each, at most `judge_excerpt_chars`
  (default 2400) per finding, in batches of `judge_batch` (default 6). Never the
  whole source. A finding whose figures are in no line of the source, or a note
  with no source text, is not sent (`no_source`).
- **What it answers**, per finding: `real` (the source shows what the checker
  claims), `misstated` (the source is consistent, but the note does not say what
  it says: a figure or label that is not the source's), `false` (the checker
  misread **and** the note states what the source states: a setting taken for
  the quantity, a name or label taken for a measurement, a method taken for a
  conflict, one result stated two ways), or `unsure`, with a `quote` copied from
  the lines it was shown, `source_says` for `misstated`, and a one-sentence
  `reason`. The judge is told to compare the note's statement with the lines
  before it answers `false`.
- **What happens.**

| ending | what a person sees |
|---|---|
| `real` | the finding stays in the review item, its sentence saying the source confirms it |
| `misstated` | the finding stays in the review item as its own kind, never dropped: "The note says "X"; the source says "Y"." It is a real error in the note, whatever the checker thought it found |
| `false`, and the quote is in the lines shown | the finding is kept out of the queue; it is listed in the pass's `judged_false` (with the quote and the reason) and counted in `judging.false`; nothing is hidden |
| `false` with no quote, or one that is not in the lines shown; `unsure`; no answer for it | the finding reaches the review item *unjudged* (`unclear`), saying a person should look: a false alarm is believed only with the words that show it |
| no standing budget set | nothing is spent; every finding reaches the review item unjudged (`no_budget`) |
| no agent is ready | unjudged (`no_agent`); nothing is asked; the pass goes on and succeeds |
| the agent was asked and its reply could not be read (`unreadable`), or the call failed (`failed`) | the batch is tried once more, if what is left of the pass's share allows (Track AR8); if it fails again the findings are unjudged with that reason, in words that say the reply could not be read, not that no assistant was there. A backend that fails twice is left alone for the rest of the pass |
| the pass's share of the budget is used up | the rest are unjudged (`over_budget`) |

  Each finding carries `plain`, one sentence a person reads ("5 mW is given for
  energy, but that unit measures power; the source confirms it."), and
  `judgement` (`verdict`: `real`, `misstated`, `false` or `unjudged`; `why` for
  the last: `no_budget`, `switched_off`, `no_agent`, `unreadable`, `failed`,
  `unclear`, `over_budget`, `no_source`; `quote`, `reason`, `quote_found`,
  `source_says` for `misstated`, `source_lines`, `reused`). The
  review item adds `judging` (the note's counts) and `summary` (one sentence:
  "Read 3 of 4 accuracy findings against the source: 2 real, 1 false alarm kept
  out, 1 left unjudged (...)."). When every finding left on a note is judged
  false, an item an earlier pass opened for them is closed, saying why.
- **Within the standing budget, with no separate question.** It spends tokens,
  so it runs only when the collection has a standing token budget
  (`GET`/`POST /ops/budget`, Track AR2), and one pass may spend at most
  `judge_budget_share` of it (default 0.1). The user is asked nothing new: the
  budget is the one answer about spending. Before each call an estimate (the
  call's fixed cost, the characters sent, an allowance for the answer) is held
  against what is left, and a call that would pass it is not made. Every call
  made is counted in the pass's `cost`, in `judging.billed_tokens`, in its share
  of the budget and in the call log, whatever came of it: a failed call is still
  counted, an unreadable reply included, and its tokens are held against the
  retry. A backend that reports no usage is counted by the estimate.
- **Remembered.** A `real` or `false` verdict is stored per finding and per
  cited lines in `maintenance/fidelity/judgements.json`: the same finding is not
  paid for twice, a remembered verdict costs nothing even when the budget has
  since been taken away, and a re-read source (other lines) is judged again.
  Unjudged findings are not stored, so they are tried again within the next
  budget.
- **Only in a real run.** A dry run spends nothing. It reads the sources (free)
  and its `judging` says what a real run would do: `state: dry_run`; `enabled`
  and `budget_for_judging` as a real run reports them (judging asked for and
  not switched off; the share of the standing budget, `null` with none);
  `candidates` (the findings a real run judges: those not dismissed);
  `remembered` (those holding a verdict for the very same cited lines, reused
  free); `no_source` (those with no line of the source to show); `would_read`
  (the ones it would send to the brain keeper: the candidates less the other
  two, and 0 when `why_not` is `switched_off` or `no_budget`). `would_read` is
  still an upper bound: the share can run out before the last batch
  (Chores after Track AM; it counted remembered verdicts and said
  `enabled: false` with a budget set before).
  `POST /maintenance/fidelity`
  takes `judge: false` to keep a real pass entirely free, and
  `judge_findings: false` in `config/reading_fidelity.json` does the same for a
  collection. A pass's `cost` is the agent calls and billed tokens it made.
- **At intake.** `POST /sources/process` and `/sources/add` run the pass on the
  note they have just written. The reading is attributed to the intake's own run,
  so the note's `cost` counts it, a `fidelity.judging` line appears in the
  result (`state`, `real`, `false`, `unjudged`, `billed_tokens`, `line`), and the
  adding run's budget accounting sees the tokens.
- **It never blocks or fails a note.** Every failure ends in an unjudged
  finding. The note is never written by the pass.

## L4. Corrections: `POST /maintenance/corrections`

```json
{
  "agent_id": "...", "workspace_id": "...", "access_mode": "read_only",
  "note": "sources/wiki/<note>.md", "section": "Key results", "line": 10,
  "statement": "the statement exactly as the note writes it",
  "proposed": "the corrected statement",
  "source": {"path": "sources/raw/<file>", "line_start": 120, "line_end": 122,
             "text": "the source's own text"},
  "reason": "optional"
}
```

**Filing is a proposal, so `read_only` may file.** It writes one review item and
an audit event (`maintenance_correction_filed`, `policy.resolution_basis:
proposal_only`) and never the note (`note_written: false`). This is the only
write a `read_only` caller can make, and it is a deliberate decision: a reader
that finds an error and cannot say so leaves the brain wrong and reads the
source elsewhere next time. Nothing filed changes knowledge until a person with
`maintenance_write` accepts it.

Checked before it is filed (422 with a `reason` otherwise):

- `note` is a note in the wiki (403 outside it, 404 when absent);
- `statement` is in the note as written (whitespace aside) exactly once, at
  `line` or in `section`: `statement_not_in_note`, `statement_ambiguous`;
- `proposed` differs from it: `no_change`;
- `source.text`, when given, is in the source's text form within the lines given
  (whitespace and line-break hyphenation aside): `source_text_not_found`,
  `source_text_elsewhere` (in the source but at other lines, which it names),
  `source_lines_out_of_range`. The source is read through the same confined
  paging as L1, so a PDF is checked against its converted copy. With no
  `source.path`, the note's recorded source is used; with no `text`, the
  location alone is recorded (`source_verified: false`).

The item `correction:<note>:<hash>` (`kind: correction`) carries the note, its
`line` and `section`, `statement`, `proposed`, `why`, `source` (`path` as the
note records it, `served_path` the text read, `served_from`, the lines,
`verified`), `owner_maintained`, `filed_by` and `filings`. The same correction
(same note, statement and fix) filed again, by anyone, is the same item, with
the filer added; filed again after a decision, it returns that decision
(`already_decided`).

**Accepting** (`POST /review/action`, `resolve_work_item`, `maintenance_write`):

- re-reads the note; if it no longer says what was filed, 409 and nothing
  changes (the item stays open);
- replaces exactly that statement on that line, nothing else;
- appends an attributed line to the note's `## Provenance` (or `## Normalization
  Provenance`, or a new `## Provenance`): the item, the old and new text, who
  filed it, who accepted it, the source lines, and the snapshot;
- keeps the note as it was under
  `maintenance/rollback_snapshots/corrections/<id>/`; `POST
  /maintenance/corrections/rollback {snapshot_id, access_mode:
  maintenance_write}` puts it back and re-ingests it;
- re-ingests the note and redraws its map; audited as
  `maintenance_correction_applied`, recorded in the review history.

The result carries `correction` (`decision: applied`, `line`, `before`, `after`,
`snapshot_id`, `reingested`). A changed note is re-read by the next concept sync
as any edited note is.

**Dismissing** is remembered (`maintenance_correction_dismissed`). **A note its
owner maintains** (`maintained_by: owner`) is never patched: the item says so
in `resolve_means`, accepting it is 409, and the item stays open until it is
dismissed.

## L5. Plain status wording: `GET /ops/sources`

`verified` is kept for compatibility. It has only ever meant "the bytes were
re-hashed in this call", and `false` read as "the registry is unreliable". The
report now also says:

- `bytes_rehashed`: the same boolean, named for what it is;
- `checked`: `recorded_path_resolves`, `file_present`, and with `verify=true`
  `content_hash_matches_note`;
- `verification_note`: one sentence saying what was and was not checked.

## Settings: `config/reading_fidelity.json`

| key | default | meaning |
|---|---|---|
| `source_page_max_lines` | 500 | lines one `GET /review/source` page returns at most |
| `source_page_max_chars` | 200000 | characters one page returns at most (always at least one line) |
| `tolerance_points` | 2.0 | slack in percentage points on top of rounding, for L3 |
| `flattened_table_min_lines` | 3 | lines of numbers before a run is a flattened table |
| `ratio_suffixes` | `×`, `x`, `X`, `fold`, `-fold`, `times`, and the same word in six more languages | what makes a number a ratio (words in any case) |
| `conflict_cues` | multilingual stems | what makes a statement claim a conflict |
| `quantities` | multilingual words to units | which quantity a word names |
| `condition_words` | multilingual (`at`, `with`, `bei`, `avec`, `bajo`, `sotto`, `bij`, ...) | a value right after one, close after a quantity word, is a condition (AS1) |
| `condition_reach` | 1 | words allowed between a quantity word and a condition word for the condition to qualify it |
| `label_cues` | multilingual (`lemma`, `table`, `satz`, `tableau`, `tabla`, ...) | a word followed by a number makes the number a label (AS3) |
| `method_phrases` | multilingual (`proof by contradiction`, `durch widerspruch`, ...) | a conflict cue inside one of these names a method, not a conflict (AS3) |
| `judge_findings` | true | let the brain keeper read statement-level findings in a real pass (AS4) |
| `judge_checks` | the four statement-level checks | which findings are read |
| `judge_budget_share` | 0.1 | the most of the standing token budget one pass may spend reading |
| `judge_batch` | 6 | findings per call |
| `judge_excerpt_chars` | 2400 | characters of source lines shown per finding |

A malformed value falls back to its default and is reported in
`config_problems` of the pass; it never switches a check off.

## Limitations

- Layout loss is recognised from the shapes above. A converter that loses a
  table's layout in a way that still looks like prose is not caught; a value
  from it is then traced as `clean` if it occurs anywhere else, or not at all.
- `value_in_another_row` is judged only against a clean table in the source,
  and only when the source has a row with the note row's label.
- A value the extraction computed (a ratio it worked out) is not in the source
  and is not flagged: absence alone proves nothing.
- "A quantity defined from several components summarised as one" is in the
  extraction instructions only; no deterministic check finds it.
- `quantities`, `conflict_cues`, `condition_words`, `label_cues` and
  `method_phrases` are word lists. An install in a language the default does not
  cover extends them; until then the first two are quiet, never wrong, and the
  last three leave the false alarms they exist to prevent (which the brain
  keeper's reading then catches, when a budget is set).
- A condition is told from a quantity's own value by distance, not by grammar
  (`condition_reach`): "energy at 100 MHz" is skipped, "energy use stayed at
  4 mW" is not, and a sentence that puts three words between "energy" and "at"
  is judged though it is a condition. The reading catches those; without a budget
  a person does.
- The brain keeper's reading is as good as the lines it is shown. A figure that
  appears in no line of the source (a computed ratio, a figure spelled in words)
  is not read (`no_source`). A false alarm needs a quote found in those lines;
  the quote check is a check that the words are there, not that they support the
  verdict, so `judged_false` is listed in every report for anyone to audit.
- The reading is a share of the standing budget, not a reservation in it: an
  adding run's own estimate does not yet include it (Track AM owns estimates), so
  a run can end slightly above its estimate by the reading's cost, which the run's
  tally still counts.
- Re-extracting an existing note costs tokens and is the owner's call; the pass
  only offers it.
