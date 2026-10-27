---
type: "documentation"
title: "Sources renamed, moved or deleted"
description: "Roadmap: Workstream A (provenance that stays true), Workstream C (human review of every unsafe write) and Workstream D (recovery: every action reversible)."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing Kovium"
doc_path: "docs/source_renames_v1.md"
doc_sha256: "2199baf05bcdb88f267e213631667bfa4262a6d11790f4b09050434403f3621a"
---

# Sources renamed, moved or deleted

**Status: built 2026-10-09 (Track J of `docs/knowledge_layer_improvement_plan_v1.md`).**
Roadmap: Workstream A (provenance that stays true), Workstream C (human review
of every unsafe write) and Workstream D (recovery: every action reversible).

## The problem

Every note records the original it was made from by path: `raw_source_path:`
and `source:` in the wiki note and in its templated draft. Renaming an original
is ordinary housekeeping: a download arrives as `3722567.3727843 (1).pdf` and
its owner names it after what it is, a photo is filed under a year, a scan moves
into a client folder. After that, re-processing, `/review/source`, staleness
checks and re-derivation could no longer find the file the knowledge came from,
and re-processing the renamed file wrote a *second* note for the same document,
because ownership is decided from the recorded path. A deleted original left its
knowledge with nothing behind it, and nothing said so.

Nothing here is specific to one kind of document. A recipe photo, a scanned
contract, a recording and a paper are one case: a file whose bytes some notes
record.

## The source registry (J1)

`maintenance/source_registry.json`, per workspace. One entry per source, keyed
by the first 12 hex digits of the content hash the notes record (`sha256:`),
or, for a note recording no hash, by a hash of its recorded path:

| field | meaning |
| --- | --- |
| `sha256` | the bytes the notes were made from |
| `path` | where the source is now, spelled as a note spells it |
| `recorded_paths` | every spelling its notes record today |
| `previous_paths` | `[{path, moved_at, to, kind, basis}]`: every earlier path and when it moved |
| `size`, `mtime_ns` | the file's stamp when last seen |
| `notes`, `templated`, `converted` | what was built from it (workspace-relative) |
| `owner_maintained_notes` | its notes marked `maintained_by: owner` |
| `status` | `present`, `changed` (bytes differ from the notes'), `missing`, `retained_without_source`, `rename_pending`, `removed` |

It is **derived** and rebuildable from the notes: `source_registry.rebuild_entries`
reads only their frontmatter (`raw_source_path`, `source`, `sha256`,
`previous_source_paths`, `source_status`). What only the registry holds is the
date of each earlier move and the record of a removal; both survive a rebuild
whenever the file does. `GET /sources/registry` returns it (rebuilt from the
notes, and not saved, when no reconcile has written one yet).

Paths are spelled the way a note spells them (`source_roots.record_source`):
`<root>:<path>` under a declared source root, else relative to the checkout or
the workspace, always with forward slashes (`paths.recorded_spelling`). The
same on every host, and a case-only rename stays visible as a different
spelling on a case-insensitive file system.

A second file, `maintenance/source_hashes.json`, caches each file's stamp and
hash, so a library is hashed once. It records nothing about the knowledge:
any pass writes it (a dry run and a lazy lookup included), and deleting it
costs one slower pass.

## Noticing a rename or a move (J2)

### `POST /sources/reconcile {agent_id, workspace_id, dry_run: true, access_mode}`

Walks `sources/raw/` and every declared source root and compares each recorded
spelling with the files there:

- **present**: the exact spelling is in the listing. Its file is hashed only
  when its stamp (size, modification time in nanoseconds) is new, and a hash
  that differs from the notes' is reported as `changed` (edited in place; the
  staleness machinery's business, not a rename).
- **case-only rename**: the listing holds the same spelling apart from case,
  and the old spelling still opens (a case-insensitive file system). On a
  case-sensitive one the old spelling simply fails, and the bytes match below.
- **gone**: nothing at the recorded spelling. Untracked files (no note records
  them) are hashed, only when some source is gone, and only those of a gone
  source's known size, and matched by hash:
  - **exact**: one file has the bytes → `renamed`, `moved`, `moved_and_renamed`
    or `case_renamed`. Two files with the bytes are *ambiguous* and proposed,
    never guessed.
  - **probable**: no file has the bytes, but a new one carries the same
    persistent identifier, the same title in its own metadata or on its first
    page (folder intake's near-duplicate identity, `docs/folder_intake_v1.md`),
    or failing those the same file name. Always a proposal. Only untracked
    files of the same kind (extension) are read, newest first, at most
    `probable_rename_max_candidates` (default 200) per source per pass: reading
    an identity can mean extracting a PDF's text, and a library holds
    thousands of files no note records. A capped search is reported in
    `probable_search_truncated`.
  - **none**: missing (J4).
- **cannot be judged**, so nothing is written: a recorded `<root>:` path whose
  root is no longer declared (`root_undeclared`) or whose folder is not there
  (`location_unavailable`: an unplugged drive or unmounted share must not put
  every source on it up for removal; the same if the workspace's whole raw
  folder is gone), and a source recorded before named roots as a bare file
  name (`name_only`), which is still matched by its bytes when they turn up.
  Reported with `action: unverifiable`.

`dry_run: true` (the default) reports what it would do (`would_repair`,
`would_propose`, `would_mark_missing`) and writes nothing at all (Track AW1: the hashes it reads are kept in memory for the pass;
the governance audit log still records the call, as it records every call).
A run that writes needs `access_mode: maintenance_write` (403 otherwise, and
the refusal is audited). Every pass is audited (`surface: sources`, `action:
source_reconcile`) and reports `billed_tokens: 0`, `agent_calls: 0`: nothing
here calls a model.

The report: `counts` (per state, plus `untracked_files`), `scanned` (`files`,
`hashed`, `hashed_bytes`, `reused_hashes`), `sources` (each source that is not
plainly present, with its findings and `action`), `repairs`, `proposals`,
`missing`, `returned`, `closed_items`, `snapshot_id`, `seconds`.

**Measured** on a synthetic tree (300 files of 256 KB in 10 folders, one note
each, Windows host): first pass 10.8 s hashing all 78.6 MB; second pass 0.27 s,
0 files hashed; after 10 renames 0.29 s, 10 files hashed (the 10 new names);
the nightly gate 0.08 s.

### Lazily, wherever a path is used

`source_registry.locate(workspace_root, recorded)` is asked before a route
fails on a recorded source that is gone: first the registry (a path a repair
moved away from resolves to where it went), then the bytes the notes record for
that path, looked for under the source locations with the same incremental
hashing. Nothing about the knowledge is written. Used by:

- `GET /review/source`: serves the file where it is now; the answer gains
  `located: {recorded_path, path, basis, repaired}`. What is found is confined
  exactly as the path asked for was. An original that is not text (a PDF, a
  photo, a recording; J-R1) is never decoded: the route serves the text the
  workspace made of it, the converted copy its notes record first and the
  templated draft second, with `served_from: converted | templated` (and
  `original` for a text file), `path` the file served and `source_path` the
  original as a note spells it. With no text form it answers **415**
  `{reason: source_not_text, message, path}`, never a server error.
- `POST /sources/process` (and `process_book`): a recorded path that is gone is
  processed where its bytes are.
- `GET /ops/sources`: a new status, `moved`, with `resolved_path` where it is.
- the maintenance staleness check (`inspect_sources`): a moved source is read
  where it is, still keyed by the path its record knows, so it is not mistaken
  for a new source and re-derived.

A graph rebuild never reads the originals, so it needs nothing.

**Processing a renamed file** repairs the chain first. Its notes record the old
name, so the ownership check would not recognise them and a second note would
be written. When the file's bytes are what notes record under a name that no
longer exists, it is a rename: under `rename_repair: auto` the notes are
repaired (governed, audited, snapshotted) and processing goes on and rewrites
the same notes (`source_rename` in the result); under `propose` it is refused
(400) before any agent call, naming the `source-rename:` item that decides it.

No file watcher: reconcile plus the lazy lookup work on every platform without
one, and a watcher would be one more process to keep alive for no gain the
nightly pass does not give.

## Repairing the chain (J3)

For an exact match under `rename_repair: auto` (the default), and for any
accepted proposal:

- the wiki note and its templated draft get the new path in `raw_source_path`
  and `source` (only where they held the old one: a `source:` web address is
  never touched) and in the note's own `- Raw source reference:` line (since
  Track AP), the old path appended to `previous_source_paths` (a block
  list, so a name with a comma survives), and any `source_status` cleared;
- the wiki notes are re-ingested (the graph's provenance points at the note,
  which ingestion keeps in step; `source_status` reaches the Note node's
  metadata) and their maps redrawn;
- the concept layer's record of the note is carried forward
  (`ConceptService._rewrite_source`), so a path repair never makes the next
  concept sync re-read the document through the agent;
- the registry gets the new path and a `previous_paths` entry with when it moved.

Converted copies carry no reference to the original (their text only), and
their own recorded path does not change, so nothing in them needs repair.

**Governed.** Every write is snapshotted first, under
`maintenance/rollback_snapshots/sources/<id>/` (copies stored flat with short
names, so Windows' 260-character path limit is not reached, and a manifest of
workspace-relative paths plus the registry as it was).
`POST /sources/reconcile/rollback {snapshot_id}` puts every file back,
re-ingests and redraws. Repairs are audited (`source_rename_repair`) and
recorded in the review history.

**Policy**, per workspace in `config/sources.json`:

- `rename_repair: auto | propose`. `auto` (the default) repairs exact matches;
  `propose` turns them into review items. A value this code does not know is
  reported in `policy_problems` and read as `propose`, the choice that writes
  nothing without a person. Probable matches are always proposed.
- `reconcile_nightly: true | false` (default `true`).
- `probable_rename_max_candidates` (default 200): see above.

**Owner-maintained notes** (`maintained_by: owner`) are never rewritten by a
pass: a source with one is proposed (`why_proposed: owner_maintained_note`),
and the owner's own decision is what rewrites it.

### `source-rename:<id>` review items

`{match: exact | probable, why_proposed: policy | owner_maintained_note |
ambiguous | probable_match, from, to, candidates, evidence, notes}`.
`POST /review/action` with `resolve_work_item` repairs the notes to `to` (an
exact match is refused, 409, if `to` no longer holds the bytes; an accepted
probable match records the source as `changed`, because the notes were made
from the earlier bytes). `dismiss_work_item` rejects that file: it is never
proposed for the source again, and the next pass treats the source as missing
unless another file matches.

## A source that is gone (J4)

A recorded source with no file and no match:

- its notes (wiki and templated, never an owner's) gain `source_status:
  missing` and `source_missing_since: <date>`; the registry carries the status
  for every note, an owner's included;
- a review item `source-missing:<id>` names the source, its notes, templated
  and converted copies, the concept notes listing it, and its maps;
- the governed answer marks evidence from those notes (contract v2, additive;
  `docs/agent_memory_contract_v1.md`).

The owner decides, with `POST /review/action`:

- **Keep**: `resolve_work_item` with `decision: keep` (or `dismiss_work_item`).
  The notes become `source_status: retained_without_source`; the source is not
  asked about again, and the answer still reports it source-less.
- **Remove**: `resolve_work_item` with `decision: remove`. One audited action
  (`source_remove`): an `/ops/backup` archive first (`backup_archive` in the
  result), then a snapshot, then exactly that source's knowledge goes: its wiki
  notes and the graph documents they own (nodes, edges, passage vectors, read
  history), its templated drafts, its converted copies (a bare-name record is
  removed only when exactly one file has that name), its entries in concept
  notes, its maps, and the concept layer's record of it. Concept notes:
  - assembled without an agent: re-listed from their remaining sources; one
    with no source left and no owner's notes is removed;
  - written by an agent: the source's entry is removed and the prose kept,
    since rewriting it needs the agent; listed in
    `concepts_left_for_concept_sync`, never spent on here (`billed_tokens: 0`);
  - maintained by their owner: untouched, listed in `concepts_owner_maintained`.

  A note the owner maintains is never removed (`kept_owner_maintained_notes`).
  **Ideas** (`docs/ideas_v1.md`) are never built from a source, so none is
  removed or rewritten: an idea resting on the removed note keeps its file
  and graph node, and `GET /ideas/{key}` reports that `rests_on` entry
  `present: false`. A concept note an idea is about is kept, re-listed, even
  when no source is left. The registry never reads an idea note as a source.
  `/ops/restore` with the backup brings it all back.
- Resolving without a decision is refused (409); so is removing a source whose
  file is present again.

**A source that reappears** (restored, moved back, under any name) is matched
by its bytes on the next pass: its path is repaired if it changed, its notes
lose `source_status`, and an open item about it is closed.

## Nightly

`scripts/nightly_maintenance.py` calls `POST /sources/reconcile` (with its own
`dry_run`) before the vocabulary steps, gated by `reconcile_needed`, which
reads file names only: a collection whose every recorded source is where its
notes say makes no request at all. Off with `reconcile_nightly: false`, and
skipped silently then. Reports `0 token(s)`.

## Renaming on purpose, and everything else (Track AP)

An original is renamed on purpose with `POST /rename` (`kind: original`), which
moves the file in its folder and runs this repair, with the same snapshot,
audit and history, and records it in the one rename log of every kind
(`GET /rename/log`). Notes, concept notes, drafts and converted copies, and
renames of them made by hand, are carried through by the same engine
(`docs/renames_v1.md`); this document stays the contract for originals.

## Not in this change

- A note recording no `sha256` can be matched only by identity (a probable
  rename), never exactly.
- A source deleted without a reconcile is not known to be gone until the next
  pass (nightly, or on demand); until then the answer does not mark it.
- Re-processing rewrites a note from scratch, so its `previous_source_paths`
  is then held only in the registry.
- Removal leaves the vocabulary-candidate ledger's entries for the source, and
  links to the removed notes from other notes' prose.
- The changed-in-place case (same path, new bytes) is reported, not acted on.
