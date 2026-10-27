---
type: "documentation"
title: "Workstream C Human Review Workflow V1"
description: "Describe the in-product human review and correction workflow that closes Workstream C."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing Kovium"
doc_path: "docs/workstream_c_human_review_workflow_v1.md"
doc_sha256: "a631721048b1c65de79f02606093928b8d519a4251375c766b1af7a3f3015e40"
---

# Workstream C Human Review Workflow V1

## Purpose
Describe the in-product human review and correction workflow that closes Workstream C.

## Main workflow
The main UI shell now supports the normal review loop without ad hoc scripts:

1. inspect a graph node
2. view its provenance directly in the main details panel
3. open source evidence lines from provenance or citation references
4. review wiki content and backlinks alongside the source evidence
5. inspect the workspace review queue for duplicate, drift, stale, or annotated items
6. record a correction action or request re-normalization from the same shell
7. inspect the review history for the selected source or node

This keeps graph, wiki, provenance, and correction paths in one operator flow.

## Backend surfaces
The human-review workflow is backed by these API surfaces:

- `POST /review/queue`
- `GET /review/history`
- `GET /review/source` (`path` relative to the workspace, in either
  separator, as answers and concept notes spell it; or absolute inside the
  workspace; or `<root>:<path>` under a declared source root. Anything outside
  the workspace, its declared roots, its registered sources and the notes its
  graph ingested is refused with 403 before the file is looked at; a missing
  file is 404 and names the workspace-relative path only. The returned `path`
  is workspace-relative. Track I3, 2026-10-09.)
  With `line_start` and `line_end` it returns that range, capped per call,
  with `total_lines` and `next_line_start`, so a source is read in full in
  pages (Track L1).
- `POST /maintenance/fidelity` (Track L2/L3): the fidelity pass; the checks are
  deterministic, and a real run (Track AS4) has the brain keeper read each
  statement-level finding against its source first, within the standing budget;
  it opens `fidelity:<note>:<hash>` review items, each finding in one plain
  sentence, with the false alarms kept out
- `POST /maintenance/fallback_notes` (Track P4): finds notes a deterministic
  fallback wrote; a real run opens `fallback-note:<note>` review items that
  offer re-extraction, and never changes a note
- `POST /maintenance/corrections` (Track L4): a caller files a correction to one
  statement as a `correction:<note>:<hash>` review item; filing is a proposal,
  allowed `read_only`
- `POST /review/action`

Correction actions supported:

- `mark_noisy`
- `mark_wrong`
- `mark_duplicate`
- `mark_drift`
- `request_renormalization`
- `resolve_review`

## Reading-fidelity review items (Track L, 2026-10-09)

Two kinds of item join the queue; see `docs/reading_fidelity_v1.md`.

- **`fidelity:<note>:<hash>`** (`kind: fidelity`), from `POST
  /maintenance/fidelity` and from intake: values in a note that its source's
  text does not support cleanly (read from layout-lost text, printed in another
  row), totals that do not add up, units that do not fit, conflicts the
  arithmetic does not bear out. A value after a condition word, a number inside
  a name or a label and a method's name are not raised (Track AS), and a
  finding the brain keeper judged a false alarm, with a quote, is kept out of
  the item (listed in the pass's report); one it could not judge, or could not
  be asked about (no standing budget, no agent), says so in its sentence.
  Nothing in the note changes. Resolve = acknowledged; dismiss = those findings are not raised again for the note.
  It offers re-extraction (`request_renormalization`, which costs tokens and is
  the owner's choice) and filing a correction; an owner-maintained note is
  offered neither.
- **`correction:<note>:<hash>`** (`kind: correction`), from `POST
  /maintenance/corrections`: one statement, as the note writes it, the source's
  own text and lines (checked against the source when given), and the proposed
  fix, with everyone who filed it. Resolve = patch exactly that statement, with
  an attributed line in the note's `## Provenance`, a rollback snapshot (`POST
  /maintenance/corrections/rollback`) and a re-ingest; refused (409, item left
  open) when the note no longer says what was filed or is `maintained_by:
  owner`. Dismiss = not applied, and filing it again returns that decision.

## Agent proposals (Track AB4, 2026-10-17)

- **`agent-proposal:remember:<key>`** (`kind: remember`) and
  **`agent-proposal:forget:<key>`** (`kind: forget`), from `POST
  /agent/knowledge/propose` (or the MCP tools `remember` and `forget`), only
  where the workspace's `config/mcp.json` allows proposals. Resolve = write an
  attributed note of the collection under `sources/wiki/contributed/` and
  ingest it, or withdraw a note or a statement (moved aside or taken out, never
  deleted; `POST /maintenance/withdrawals/restore` puts it back). Dismiss = not
  applied, and proposing it again returns that decision. The contract is
  `docs/agent_memory_contract_v1.md`.
- **Who decides.** An agent listed in the item's `proposed_by` cannot resolve
  or dismiss it (409). That stops only an agent deciding under the id it
  proposed with: **the deciding `agent_id` is not authenticated** (nothing on
  either surface is, until the authentication track exists), so a caller that
  reaches the HTTP service can send another id, and the note then records
  `verified: human:<that id>`. Treat `human-reviewed` on a contributed note as
  only as strong as the service's loopback boundary.

## Safety model
- Review reads remain `read_only`.
- Review actions require `maintenance_write`.
- Review actions do not directly overwrite canonical Markdown.
  The one exception is an accepted `correction:` item, which patches exactly
  the statement it names, attributed in the note's own provenance and
  reversible from its snapshot; never in an owner-maintained note. And an
  accepted agent proposal, which writes one attributed note or withdraws one
  (never deleting it), each reversible.
- Re-normalization uses the existing maintenance refresh path and remains rebuildable from source notes.
- Every review action is recorded in durable history.

## Durable evidence
The workflow keeps durable local evidence in:

- `maintenance/review_annotations.json`
- `logs/review_history.jsonl`

This preserves versioned human review history for later inspection.

## Provenance-first debugging
The main shell now exposes provenance directly through:

- node details provenance list
- wiki citation cards
- source evidence panel with path and line window

This supports:

- graph to page
- page to evidence
- evidence line inspection without leaving the product

## Duplicate, drift, and noise review
The review queue is built from the normal maintenance path and merges:

- registry-backed source records
- duplicate candidates
- semantic drift candidates
- stale and refresh-eligible records
- human review annotations

That makes duplicate, drift, and noise review part of normal maintenance rather than a separate script-only path.

## Verification
The closeout behavior is covered by:

```bash
cd <repo>
.venv/bin/pytest tests/unit/test_maintenance_service.py tests/integration/test_maintenance_api.py tests/integration/test_review_api.py tests/system/test_ui_shell.py tests/system/test_wiki_ui_shell.py tests/system/test_wiki_citation_ui_shell.py
.venv/bin/pytest tests/integration/test_graph_view_api.py tests/integration/test_wiki_page_api.py tests/integration/test_wiki_citation_api.py tests/integration/test_wiki_resolution_api.py tests/system/test_api_endpoints.py tests/system/test_graph_canvas_ui.py tests/system/test_graph_ui_shell.py tests/system/test_persistent_ui_flow.py
```
