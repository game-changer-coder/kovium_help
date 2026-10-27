---
type: "documentation"
title: "Renaming anything, and every reference follows"
description: "Roadmap: Workstream A (provenance that stays true), Workstream C (human review of every unsafe write) and Workstream D (recovery: every action reversible)."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing Kovium"
doc_path: "docs/renames_v1.md"
doc_sha256: "7f312627cbdc62104dc6f1058da0d14a96f03d257a96854e5833b16a05e6ad39"
---

# Renaming anything, and every reference follows

**Status: built 2026-10-22 (Track AP of `docs/knowledge_layer_improvement_plan_v1.md`).**
Roadmap: Workstream A (provenance that stays true), Workstream C (human review
of every unsafe write) and Workstream D (recovery: every action reversible).

## The problem

A name may need to change: two documents met on one name, a title came out
badly, a file named `raw__templated.md`, or the ASCII rule of Track AK for
every name the platform gave before it. A note's identity is the hash of its
path, and that identity is inside every edge id, every node made from the
note's lines, its passage vectors' key, its read history and the change log.
Renaming a note by hand orphaned all of them, and its map, its links and its
concept record with them. Track J (`docs/source_renames_v1.md`) already
carried a renamed **original** through; nothing carried a note, a concept, a
draft or a converted copy.

So the platform owns renaming: one route renames anything, a nightly pass
carries a rename made by hand down the whole chain, and one log records every
rename of every kind. Nothing here reads what a document is about: names are
paths, the same for every note type, for both retention modes, and spelled
the same on every host. Nothing here calls a model or the embedding server.

## The inventory: every place a name lives (AP1)

Written before the engine, in `second_brain/renames/inventory.py`, and held
to the code by tests (below). What a rename does with each place:

**A note** (a wiki note made by the pipeline, a note written by hand, a book's
chapter note):

| place | the key | what a rename does |
|---|---|---|
| the note file | `sources/wiki/<stem>.md` (or a folder of its own) | moved |
| its templated draft | `sources/templated_markdown/<folder>/<stem>.md` | moved with it, to the same stem |
| the note's own source lines | `- raw_source:` and `- source::` naming its draft | rewritten |
| the Note node | `note:<hash of its path>`, its metadata path, its provenance | moved |
| nodes made from the note's lines | `figure:<hash>-*`, `equation:<hash>-*` | moved |
| edges | ids (`edge:<type>:<from>:<to>`), from, to, owner | moved; an edge another note stated keeps that note as its owner |
| passage vectors | the note's key (and the older checkout-relative key) | moved, never re-embedded |
| access history | `note_access` | moved (merged if the new id had some) |
| the change log | `note_changes` | one `renamed` row with the old id and path |
| staging snapshots | `data/staging/*.json`: document id, paths, ids inside edge ids | rewritten |
| links in other notes | `[[stem]]` in every form, Markdown links, `## Links`, `## Relations`, the note-link frontmatter fields | rewritten, the words around each link kept; never in a note its owner maintains (listed instead) |
| concept notes listing it | `sources:` and `## Sources` | rewritten |
| the concept layer state | `maintenance/concepts_state.json`, by stem | moved, its absorbed hash carried, so no concept pass is owed and no token spent |
| its map and the maps index | `sources/wiki/maps/<stem>.canvas`, `.note_maps.json` | redrawn under the new name, the old one removed |
| the source registry | `maintenance/source_registry.json` `notes`, `templated` | rewritten |
| review items, fidelity records and other maintenance records | any path or node id they hold | rewritten |
| the brain keeper agent's answer cache | `brain_keeper/cache/<key>.json`, the key of an input that named it | re-keyed from the job that asked, so the answer is not paid for again |
| a caller's saved cursor and references | the change log | the `renamed` row tells the caller (AP5) |
| the rename log | `maintenance/rename_log.jsonl` | one line per rename |

The note-link frontmatter fields are `supersedes`, `superseded_by`,
`sources`, `rests_on` and every relation token of the install's vocabulary
(`config/relations.json`), and any value written as a `[[link]]`. A concept
list (`concepts:`) is never a note link.

**A concept note:**

| place | the key | what a rename does |
|---|---|---|
| the concept note file | `sources/wiki/concepts/<key>.md` and its `concept:` field | moved, a redirect stub left at the old key for links from outside the collection |
| sources' concept lists | `concepts:` and `## Concepts` links | relinked (`relink_concept`, the merge machinery) |
| other concept notes | related, broader, narrower, links in their prose | relinked, except in a note its owner maintains, which the stub serves |
| links anywhere else | `[[key]]` in any note | relinked |
| the concept node | `concept:<key>` | moved |
| the concept note's own Note node and vectors | as for a note | moved |
| the concept layer state | `sources[*].concepts`, `owner_listed`, `compared` | moved |
| open review items | `keep`, `absorb`, `concept`, paths | rewritten |
| maps | canvases drawing it | redrawn |

The stub is a `concept_redirect` note (`Renamed to [[new|label]] on <date>`),
like a merge's, dated as the concept layer dates its notes and a merge's stub. Ingestion now follows a stub: a note naming the concept the
old way lands on the node the renamed note has.

**An original:**

| place | the key | what a rename does |
|---|---|---|
| the original file | `sources/raw/...`, `<root>:...`, or any folder of the workspace a note records it in (`inbox/a.md`) | moved, in its folder |
| its notes' recorded source | `raw_source_path`, `source`, `previous_source_paths`, the `- Raw source reference:` line, under every spelling the notes use (`inbox/a.md` and, once a root is declared over the folder, `inbox:a.md`), each kept in its own form | Track J's repair |
| the source registry | `path`, `recorded_paths`, `previous_paths` | Track J's repair |

**A converted copy:**

| place | the key | what a rename does |
|---|---|---|
| the converted copy | `sources/converted_markdown/...` | moved |
| its notes | `converted_markdown_path` and `- Converted markdown reference:` | rewritten (a bare file name stays a bare name) |
| the source registry | `converted` | rewritten |

**The store's tables** (`STORE_TABLES`): `nodes`, `edges`, `passage_vectors`
and `note_access` are moved; `note_changes` is history (a `renamed` row is
added, earlier rows keep the name they had); `store_meta` holds no name.

**History is never rewritten**: `logs/` (audit, review history, metrics),
`brain_keeper/jobs/` (each job's input as it was asked), every snapshot under
`maintenance/rollback_snapshots/`, concept merge and concept history
snapshots, the rename log, the hand-rename decisions, the curation and
citekey logs, the originals' hash cache, backups and exports.

**The tests that hold the inventory to the code:**
`tests/contracts/test_rename_contract.py` fails when the store gains a table
the inventory does not classify, or when this document stops naming a place;
`tests/integration/test_rename_api.py` renames a note in a workspace where
every layer has written (graph, vectors, access, change log, staging, concept
layer, maps, registry, records) and asserts that no row outside the change log
and no file outside history holds the old path, the old id or its hash, that
a link to the old name is left only in the note its owner maintains, and that
every file under `maintenance/` and `data/` is one the inventory classifies.
So the chain cannot grow a place that records a name without the engine
knowing of it.

## The engine (AP1)

`second_brain/renames/engine.py`. Given an object and a new name it plans
every change on a view of the workspace as the renames planned so far leave
it (so a batch that renames a note and a note linking it gets both right),
then applies them in one step:

1. **a dry run first**: the plan lists every file moved, every note
   rewritten, the store rows that move (`store_rows`), the records rewritten,
   the answers re-keyed and the links left in the owner's notes; it writes
   nothing;
2. **a snapshot** under `maintenance/rollback_snapshots/renames/<id>/`: every
   file about to change is copied first, and the manifest (moves, created
   files, the keys to move) is written before anything changes and kept
   current;
3. the files moved and rewritten, the records and the concept state written,
   the answer cache re-keyed;
4. the store's keys moved in one transaction (`SQLiteGraphStore.move_keys`);
5. the rewritten notes ingested again from their Markdown (no model), their
   maps redrawn, orphans pruned;
6. the rename log, the audit (`surface: renames`, `rename_applied`) and the
   review history.

**Names.** A new name is taken as given, in any script, once a file system on
every host would take it (no `< > : " / \ | ? *`, no reserved Windows name,
no trailing dot or space, at most 150 characters): `rename_name_invalid`
otherwise. Since Track AZ2 a note made by the pipeline is named without its
draft's `__templated` ending, and its draft keeps it: a rename of such a note
names the note as given (an ending sent is dropped) and its draft the same
name with the ending, which is what ties the two and what processing the
original again finds them by. A note renamed by hand keeps the name the
person gave it, with or without the ending. A concept's new name is made a
key (`concept_key`).

**Notes named the old way** (Track AZ2): `POST /rename/clean_names` `{dry_run:
true, access_mode, limit}` renames, in one pass and one snapshot, every
pipeline note whose name carries the `__templated` ending or was cut from its
title at 60 characters inside a word, to the note's rule
([[note-format-contract-v1|note_format_contract_v1.md]], "File names"):
`<title-cut-mid-phrase>__templated.md` becomes `<title cut after a whole word
at 80 characters>.md`, its draft `<that name>__templated.md`. Links, keys,
vectors and maps follow as in any rename; `POST /rename/rollback` undoes it.
Left: notes written by hand, notes the owner maintains, parts of a document
(Track AL8's names), and names chosen any other way (a name kept apart from
another document's, one the person gave). The answer's `say` is the
catalogue's (`clean_names.*`); the catalogue's "Clean names for notes you
already have" is this one-time step. No model call.

**A note and its draft** are paired by one rule, `notes/draft_pairing.py`,
which the ASCII report uses too: the note's own source line
(`- raw_source:`) names its draft; a draft two notes claim (a note copied by
hand carries the original's line) belongs only to the one named after it. A
rename never moves a draft for another claimant: that note is renamed alone
(`draft_left: draft_claimed_twice` in the plan), and naming such a draft as
the target is refused (409 `rename_draft_claimed_twice`).

**An original** is any file a note of the workspace records, wherever it is
in the workspace, or any file under `sources/raw/` or a declared source root.
A file no note records outside those is refused (400
`rename_original_not_recorded`): it is no original of the collection.

**Never overwritten.** A new name another file has is refused,
`rename_name_taken`, with a free name proposed (`detail.free_name`), which the
caller sends without asking anyone. The one exception is an original whose new
name is an untracked file of the user's: that is the user's question
(`rename_would_overwrite_original`, `asks: originals`); only a yes sends
`overwrite: true`, and the other file is then set aside in the snapshot, never
deleted. A file another source's notes record is never overwritten at all.

**Undone whole.** A step that fails part way is rolled back at once from its
snapshot and reported (`rename_failed_undone`); nothing of it remains. A step
cut off by a stopped process leaves its manifest `applying`: the next rename,
or the nightly pass, undoes it first (`rename_recovered_unfinished`). A dry
run lists such a snapshot in `unfinished`.

**Idempotent.** Renaming to the name it has answers `already_named` and does
nothing; the same rename sent twice does the work once.

**The owner's notes.** A note marked `maintained_by: owner` is never
rewritten by a rename: its links to the old name are listed
(`links_left_in_owner_notes`, warning `rename_links_left_in_owner_notes`, a
line the owner reads). A note written by hand without that mark is the
collection's and its links follow.

## The routes (AP2, AP4)

### `POST /rename {kind, target, new_name, dry_run: true, access_mode, overwrite: false, agent_id, workspace_id}`

`kind`: `original`, `note`, `concept` or `converted`. `target`: an original
by its path (`sources/raw/...` or `root:path`), a note by its path, its
draft's path or its file stem, a concept by its key, a converted copy by its
path. `new_name`: the new file name (a concept's new key). A dry run is the
default; a real run needs `access_mode: maintenance_write` (403
`maintenance_write_required` otherwise, audited). The answer: `renames` (per
object: `kind`, `from`, `to`, `files`, `notes_rewritten`, `created`,
`links_left_in_owner_notes`, `node_ids`), `counts` (`renames`,
`files_moved`, `notes_rewritten`, `records_rewritten`, `answers_rekeyed`,
`store_rows_moving`, `links_left_in_owner_notes`), `store_rows`, `records`,
`unfinished`, `summary` (one plain line), `billed_tokens: 0`,
`agent_calls: 0`, `embedding_calls: 0`; a real run adds `snapshot_id`,
`keys_moved`, `reingested`, `maps`, `originals` and `recovered`.

Refusals: 404 `rename_not_found`, 409 `rename_name_taken`, 409
`rename_would_overwrite_original`, 422 `rename_name_invalid`, 500
`rename_failed_undone`.

### `POST /rename/rollback {snapshot_id, force: false, access_mode, agent_id, workspace_id}`

Undoes one rename, or one pass of them, from its snapshot: files moved back
(a file renamed by hand and carried through goes back to the name it had),
copies restored, created files (a redirect stub) removed, the store's keys
moved back (the change log records `renamed` again), the notes ingested again
and their maps redrawn. Needs `maintenance_write`. Refused (409
`rename_changed_since`, listing the notes) when a note changed since the
rename, because undoing would discard that edit; `detail.later_renames` names
the later renames that touched those notes, newest first, to undo before this
one; `force: true` undoes anyway, but never discards what the owner wrote in a note's own
section (`## My notes`, Track AH) since the rename: the restored note keeps the section as it is now.
Undoing twice is 409 `rename_already_rolled_back`; an unknown id is 404
`rename_snapshot_not_found`.

### `GET /rename/log?kind=&name=&limit=` (AP4)

The rename log, newest first: `{at, kind, old, new, how, by, snapshot_id,
old_id, new_id}`. `kind` is `original`, `note`, `concept`, `converted` (or
`concept_node` for a graph node the ASCII pass moved); `how` is `route`,
`hand` (made by hand, carried through), `exact` or `probable` (a hand rename
decided in review), `ascii` (the ASCII pass) or `rolled_back`. `name`
matches an old or a new name. Read-only, audited. The log is append-only
(`maintenance/rename_log.jsonl`); it widens the source registry's
`previous_paths`, which still records an original's earlier paths.

Over MCP these routes are not offered (`owner`; the log `narrower_read`):
renaming reaches an MCP caller only through the first-run track's access
policy (`manage`, Track AG), which does not exist yet.

## Callers are told (AP5)

`GET /agent/knowledge/changes` and the MCP `delta` tool list a renamed note as
one entry with `change: renamed`, `note_id` (the new id), `path`,
`previous_note_id` and `previous_path`, so a caller's saved references to the
old id or path follow it (`docs/agent_memory_contract_v1.md`). A note whose
text the rename changed (its own source lines) also gets a `changed` entry. A
file moved behind the platform's back still reads as a removal and an
addition until the nightly pass carries it through. The store records the
two columns since schema v7 (`config/schema_version.json`; an idempotent
`ALTER TABLE`, nothing to rebuild).

## Renames made by hand (AP3)

### `POST /rename/reconcile {dry_run: true, access_mode, agent_id, workspace_id}`

`second_brain/renames/hand.py`. A note or concept note **missing at its known
path** (the store knows each note's path) while a file no note owns carries
its identity is a rename:

- **exact**: the same bytes (the content hash the store recorded) or the same
  source hash in its frontmatter (the source registry's): applied through the
  engine under the workspace's `rename_repair` policy (`config/sources.json`,
  `auto` by default, as for originals); under `propose`, or when the note is
  maintained by its owner, or when two files match, it is proposed instead;
- **probable**: the same title, different bytes: always proposed, never
  applied.

Also found: **a draft renamed alone** (its note follows it), **a converted
copy renamed alone** (exact when its bytes are the original's, as for a
Markdown original; otherwise the only unrecorded copy in its folder is
proposed), and **a map renamed alone** (drawn again under its note's name,
the stray copy removed). A platform note renamed by hand keeps the name the
person gave it; its draft follows with the `__templated` ending (Track AZ2).

A proposal is a review item `name-rename:<id>` (`kind: name_rename`,
`object`, `match`, `why_proposed`, `from`, `to`, `candidates`, `evidence`).
`POST /review/action` with `resolve_work_item` carries the rename through
(`rename_decision` in the answer); `dismiss_work_item` rejects that file for
that name, never proposed again (`maintenance/renames_state.json`).

A dry run reports `found` (`exact`, `probable`, `drafts`, `converted`, `maps`)
and the engine's `plan`; a real run needs `maintenance_write` and adds
`applied`, `proposals`, `maps_redrawn`, `snapshot_id` and `recovered`. One
snapshot covers every rename the pass carried through.

**Nightly.** `scripts/nightly_maintenance.py` calls it after the source
reconcile and before the vector catch-up (so a renamed note's vectors are
moved, not indexed again), with the same switch (`reconcile_nightly`), gated
by `hand.needed`, which reads the store read-only and file names: a
collection where every note the store knows is where it was makes no
request. The same pass catches a rename a synced folder brought as a deletion
and a creation.

## Existing names brought to the ASCII rule (AP6)

### `POST /rename/ascii {dry_run: true, access_mode, limit, agent_id, workspace_id}`

`second_brain/renames/ascii_pass.py`, from the report of Track AK
(`second_brain/notes/name_rule_report.py`, `scripts/report_ascii_names.py`):

- a platform-named note and its draft, to the name the report proposes (a
  letter spelled out, `bersicht-...` given its dropped letters back);
- a converted copy, to the name the pipeline gives it today, so processing
  its original again lands on it;
- a book's chapter files (`sources/raw/book_chapters/...`), folder and file
  each spelled out, through the original's rename, the empty old folder
  removed (processing the book again would otherwise write into it);
- a concept note, to its ASCII key, a stub left at the old key;
- the graph's concept nodes still named in a label's own script while their
  note's key is ASCII (the limitation Track AK left): moved to the key.

Two names that meet in ASCII are never merged: the second gets a short suffix
(a concept's from a hash of its own name, as `AliasIndex.key_for` does).
Notes written by hand and notes or concepts their owner maintains are never
renamed (`skipped`, `why: owner_maintained`), nor is a note whose source line
names another note's draft (`why: draft_claimed_twice`, and one plain line in
the summary). The report lists a pipeline-made note even when it has no draft
of its own (`draft_problem`), as far as it can tell the note is not
hand-written. The whole pass is one snapshot,
undone by one `POST /rename/rollback`; `limit` bounds one pass, the rest
waits for the next. The answer adds `notes`, `converted`, `chapters`,
`concepts`, `graph_nodes` and `skipped` to the engine's.

## A document's parts brought to their order (Track AL8)

### `POST /rename/parts {book?, dry_run: true, access_mode, agent_id, workspace_id}`

`second_brain/renames/parts_pass.py`. Every part of a multi-part document is
named `<document-key>-<NN>-<title>` (`docs/ingestion_spec_v1.md`, "Each
part's name"), so a document's parts sort together and in reading order. This
pass brings parts named before that rule to it: for each book the notes know
(`book_source`: a book the platform split, or one attached by hand through
`POST /sources/book_parts/attach`), or only the `book` named, in reading
order:

- each part's note and its draft (`notes`);
- its part file, when it is the platform's own, under
  `sources/raw/book_chapters/` (`part_files`), through the original's
  rename, so every note recording it follows (Track J's repair);
- its converted copy, when its part file is the platform's own
  (`converted`);
- the book's index file in the same folder, to `<key>-00-<word>`, its links
  to the parts rewritten (`index_files`).

A file the owner put in the collection is never renamed, nor is a note its
owner maintains (`skipped`). One snapshot for the pass, the index file
included, undone by one `POST /rename/rollback`. Run again, it renames
nothing.

Found while building it: an original renamed together with the notes that
record it, in one batch, failed at the original's repair, which looked for
the notes where they were before the batch moved them. The repair now reads
each one where the batch put it.

**A rollback leaves no node for a name it took away** (fix round, AL-R6).
Found on a real collection: notes the graph held under an older spelling of
their id, or did not hold at all, were read in afresh under their new names,
and those nodes (about a thousand) outlived the rollback. Every rename now
records the notes the graph held before (`graph_before` in the snapshot);
after a rollback has put the files back and the keys, a Note whose file is
gone is removed with every node made from its lines, a Note the graph did not
hold before for a file another Note already records is a second copy and is
removed too, and edges whose ends are gone are removed. A Note for a file on
disk the graph did not hold before stays (the rename filled a gap), and the
rollback's `graph` says so: `{stale_removed, copies_removed,
dangling_edges_removed, added_for_notes_on_disk, refreshed_from_disk, line}`.
This is the engine's, so every kind of rename gets it.

**Notes read again from their files are kept and said** (chores after Track
AM). A note whose graph copy was behind its file (edited by hand and not read
in since, or whose `## Links` was redrawn after it was read) is read again when
a rename rewrites its links. The rollback puts its file back and reads it in
again, so the graph holds the file's text, not the older copy: on a real copy,
6 notes, 38 nodes and 116 edges. That is kept, since the graph then matches
the disk; the rename records each note's content hash beside `graph_before`
(`graph_before_hashes`), and the rollback lists every note held before under
the same id whose hash changed in `refreshed_from_disk`. `line` says, in
plain words, how many notes were read again, how many notes on disk were
added, and how many links to things the undo took away were cleared (empty
when none of the three; the last two since fix C-R4). A snapshot written before this change has
no hashes and lists none.

## Who settles what

Every refusal is the system's or the caller's to settle (Track AR):
`rename_name_taken` (send the free name), `rename_name_invalid`,
`rename_not_found`, `rename_kind_unknown`, `rename_snapshot_not_found` and
`rename_changed_since` (leave the rename; force only when the edit may go)
are the caller's; `rename_recovered_unfinished` and
`rename_already_rolled_back` are the system's. The user is asked one thing,
and only for an original: `rename_would_overwrite_original`, the originals
question, with the free name as the recommended answer. Two codes reach a
person as a line in plain words: `rename_links_left_in_owner_notes` and
`rename_failed_undone`.

## Not in this change

- A note's id still follows its path; a rename is a move of keys, not a
  stable id. A file moved behind the platform's back reads as a removal and an
  addition in the change log until the nightly pass carries it through.
- Links to a renamed note from outside the collection (another vault, a
  bookmark) break; only a concept leaves a redirect stub.
- A stored agent answer is re-keyed only when its job record still holds the
  input and its key matches the skill's current definition.
- Over MCP nothing renames until the access policy of Track AG exists.
- A converted copy recorded by a bare file name shared by two copies is not
  rewritten (which one it means is unknown), as in Track J.
- Stale staging snapshots of other notes keep their own old text (a link's
  words inside an old snippet); their ids and paths are moved.
