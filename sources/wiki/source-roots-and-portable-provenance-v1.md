---
type: "documentation"
title: "Source roots and portable provenance"
description: "that stays true) and Workstream D (a workspace that can be moved, backed up and restored as one thing)."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing Kovium"
doc_path: "docs/source_roots_and_portable_provenance_v1.md"
doc_sha256: "abfb57b9b7a1bab0627a327e40c4284dea354fcd3311ade4f613574ae2e1d216"
---

# Source roots and portable provenance

**Status: written and built 2026-10-06.** Roadmap: Workstream A (provenance
that stays true) and Workstream D (a workspace that can be moved, backed up and
restored as one thing).

## The problem

Two rules pulled against each other, and the code resolved it by throwing
information away.

- **Self-containment:** nothing written down may name a home directory.
- **Asked at the first addition (Track AZ4):** a collection made from the
  conversation starts at `source_retention: ask` (read as `copy`); its first
  addition from outside asks whether to keep a copy of each original or to
  point to it, and pointing names each folder as a root, after its own name
  ([[adding-documents-v1|adding_documents_v1.md]]).
- **Your own library:** with `source_retention: reference` a workspace reads
  originals where they lie — a 50 GB library is not copied in.

A path outside the checkout went through `portable_path`, which keeps only the
file name. So a note made from `/data/library/papers/offloading.pdf` recorded
`source: offloading.pdf`. Retrieval did not care (the note's text is in the
workspace), but nothing could go back to the original: not to see whether it
changed, not to re-process it, not to tell two `chapter.md` files apart.

Separately, the graph database stored every provenance reference as an absolute
path (`/home/<someone>/second_brain/sandboxes/w/sources/wiki/x.md`). Moving the
checkout, restoring a backup on another machine, or cloning a workspace broke
every reference — and the database is exactly what an export copies.

## The mechanism: named source roots

A workspace declares the folders it reads from outside itself, by name, in its
own `sandboxes/<id>/config/sources.json`:

```json
{
  "source_retention": "reference",
  "source_roots": { "library": "/data/library", "fieldnotes": "D:/Research/Notes" }
}
```

A source under a declared root is recorded as `library:papers/offloading.pdf`.
The machine-specific part lives in one line of one config file; notes carry
none of it. Move the library, change the line, and every note resolves again.

- **Only the workspace's own file.** `config/sources.json` at the checkout root
  is tracked; a home path declared there would be committed. Roots are per
  install by nature, so they are read from the workspace config only.
- **Names:** lower-case, 2–32 characters, `[a-z][a-z0-9_-]+`. Two characters at
  least so a Windows drive letter (`C:`) is never read as a root; URL schemes
  (`http`, `https`, `file`, …) are reserved. A root is an absolute path, or (Track AG2) a path relative to the collection
for a folder inside it (`"library": "library"`), which travels with the folder
to any machine; a relative path that leaves the collection, or a full path
written for another system (`C:/...` read on Linux), is refused and reported.
  Invalid entries are reported, never guessed at.
- **Most specific root wins** when roots nest.

### How a source path is written into a note

In order: under a declared root → `name:relative`; inside the checkout →
checkout-relative (unchanged); inside the workspace → workspace-relative.
Anywhere else there is no way to write it down without a home path.

**Under `reference`, an undeclared external source is refused at intake**, with
a message naming the file to edit — before any agent call, so the refusal is
free. Under `copy` (the default) the original is copied into `sources/raw/`
first, so this never arises. Refusing was chosen over recording the bare name
because the bare name is the silent failure this replaces.

`POST /sources/process` also accepts `library:papers/offloading.pdf` as a path.

**The converted copy is spelled by the same rule (2026-10-25).** A note's
`converted_markdown_path` and its `- Converted markdown reference:` line used to
go through `portable_path`, which keeps only a file name outside the checkout,
so a collection living elsewhere recorded `a_page.md`. They now use the
original's rule, so the copy is checkout-relative inside the checkout and
collection-relative outside it; a re-read that repoints notes spells it the
same way.

**A copied or moved collection reads its checkout spellings as its own.** A
collection inside a checkout records `sandboxes/<id>/sources/...`. Copied to
another install or moved to a folder of its own, that spelling named nothing in
the checkout and, read against the collection, `<collection>/sandboxes/<id>/...`:
a re-processed source's plain text copy was written into that nested folder,
and its notes, no longer recognised as its own, got a second note beside them.
`source_roots.relative_readings` now tries, in order, the part after
`sandboxes/<id>/` inside the collection, the collection, then the checkout;
`resolve_source` and both ownership keys use it, and when nothing exists the
collection's own reading is the one returned. For a collection still in the
checkout the first and last readings are the same file.

### Ownership keys

Both `SourcePipelineService._source_key` and the normalization service's
resolve a recorded source to an absolute path to decide whether an existing
file is "ours". Both now resolve `name:` forms through the workspace's roots.
Without that, re-processing a referenced source would not recognise its own
templated draft and would write a second one with a second identity.

## Portable graph provenance

`SQLiteGraphStore` stores each provenance `source_path` in a portable form and
resolves it as it reads — one boundary, so every consumer above it (retrieval,
`/wiki/page`, the governed answer, `read_next`) sees exactly the absolute path
it saw before. **The agent contract does not change.**

- Inside the workspace → workspace-relative (`sources/wiki/x.md`).
- Under a declared root → `name:relative`.
- Anywhere else → kept as it is. The database is per-machine runtime state and
  losing a pointer there is worse than keeping it.

A relative path in the database always means relative to the workspace — never
to the checkout, because both have `config/`, `data/` and `logs/` and the guess
would be ambiguous. The workspace root is passed in by the registry, or inferred
from the documented layout (`<root>/data/staging/<db>`); with neither, nothing is
rewritten, which is exactly the old behaviour. Rows written before this keep
their absolute paths and still load.

A Note node's own `metadata.path` — the graph's pointer to the note itself —
is stored the same way, and `note_id_for_source_path` matches both the portable
form and the absolute one older rows hold. The per-document staging snapshot
(`data/staging/<hash>.json`) uses the same form throughout.

**Cost of resolving on read.** The store resolves its workspace root once
(again only if `workspace_root` is reassigned), not once per path, and loads
the declared roots at most once per read, and only when that read meets a
`name:relative` path. Resolving a path is a file-system call, slow on Windows:
before this, one governed answer resolved the same root about 20,000 times on a
60-note workspace and close to 50,000 on a real one, most of its time. The
absolute-form check and the join are memoised (bounded) as pure functions of
their strings. The paths returned are unchanged.

## Tracking the original: `GET /ops/sources`

Per note: the recorded source, where it resolves to, and a status —

| status | meaning |
| --- | --- |
| `present` | found where the note says |
| `changed` | found, and its bytes no longer match the note's `sha256` (with `verify=true`) |
| `missing` | the note says where it is; nothing is there |
| `root_undeclared` | recorded as `name:…` but this workspace declares no such root |
| `name_only` | recorded before this change as a bare file name; cannot be located |
| `unrecorded` | the note names no source |
| `moved` | not where the note says, but its bytes were found under the source locations; `resolved_path` is where (Track J; `POST /sources/reconcile` repairs the note — `docs/source_renames_v1.md`) |

Hashing is opt-in (`verify=true`): reading every original of a large library is
the expensive part, and presence is usually the question.

**What the report checked (Track L5, 2026-10-09).** The report's `verified`
field has only ever meant "the bytes were re-hashed in this call". A caller read
`verified: false` as "the registry is unreliable", which it never meant. It is
kept for compatibility, and the report now says plainly what it looked at:
`bytes_rehashed` (the same boolean, named for what it is), `checked`
(`recorded_path_resolves`, `file_present`, and with `verify=true`
`content_hash_matches_note`) and `verification_note`, one sentence. Without
`verify`, a file edited since intake still reads `present`.

## Not in this change

- The older `/maintenance/inspect` and `/maintenance/refresh` still take paths
  from the caller; they were not rewired to start from notes.
- Job files, logs and the maintenance registry still hold absolute paths. They
  are per-machine runtime state, not published, and not read across machines.
- A note recorded as a bare name before this change is reported (`name_only`),
  not repaired: which folder it came from is genuinely unknown.
