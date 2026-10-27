---
type: "documentation"
title: "Workstream B Governance And Routing Policy V1"
description: "This document is the explicit closeout policy for Workstream B."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing Kovium"
doc_path: "docs/workstream_b_governance_and_routing_policy_v1.md"
doc_sha256: "1ac42dc79b2fca9f4e86b3a664bfee3bdc4e22a8806f17912070dd8eb61394a4"
---

# Workstream B Governance And Routing Policy V1

## Purpose
This document is the explicit closeout policy for Workstream B.

It defines:
- the supported default-routing behavior after the workspace migration
- which knowledge-platform actions are read-only versus write-capable
- which write-capable actions are currently auditable
- what legacy shared-root behavior is intentionally retained

---

## Final default-routing policy

> **Superseded in part, 2026-09-28 and 2026-10-01.** Requests are now refused
> unless they name a registered `workspace_id` (`422` when none is named, `404`
> when it is not registered); an `agent_id` alone no longer resolves to a
> workspace. The behaviour below survives only when an install opts out with
> `KOVIUM_ALLOW_UNSCOPED=1`. In that mode the legacy default workspace is
> built the first time an unscoped call needs it; a strict install never builds
> it at all, and asking the registry for it raises. `GET /agent/knowledge/contract`
> needs no workspace in either mode, because it describes the API rather than
> any one collection.

### Scoped requests
- A request with explicit `workspace_id` resolves to that workspace sandbox.
- A request with `agent_id` and no `workspace_id` resolves to that agent's default workspace sandbox.
- Migrated routes must not silently fall back to another agent's workspace.

### Unscoped requests
- A request with neither `agent_id` nor `workspace_id` resolves to the legacy default workspace.
- This legacy default is retained only for compatibility with unscoped local/operator workflows.
- It is not the recommended path for agent-facing knowledge use.

### Practical meaning after the August 8, 2026 migration closeout
- agent-knowledge routes are workspace-aware
- maintenance routes are workspace-aware
- retrieval routes are workspace-aware
- operational routes that were migrated in the Workstream B slices are workspace-aware
- legacy graph/wiki/search/path/gap/UI routes are now workspace-aware when scoped
- the browser UI now propagates scoped identity when `agent_id` and optional `workspace_id` are supplied in the page URL

---

## Supported access policy

### Read-only agent-facing knowledge
The agent-knowledge contract remains read-only by default and currently supports only:
- `access_mode=read_only`

This applies to:
- `POST /agent/knowledge/answer`
- `POST /agent/knowledge/task_context`
- `POST /agent/knowledge/read_next`
- `GET /agent/knowledge/contract`
- read-oriented graph/wiki/query surfaces when used as supporting inspection routes

The current project intentionally does **not** expose direct agent write-back into normalized wiki knowledge through the agent-knowledge contract.

### Governed write-capable action
The currently supported governed write-capable memory-maintenance action is:
- `POST /maintenance/refresh` with `access_mode=maintenance_write`

This remains narrow by design:
- it is explicit
- it is auditable
- it operates inside the resolved workspace
- it does not grant broad direct overwrite capability to the agent-knowledge contract

### Routed operational write-like workflows
The following workspace-routed workflows write artifacts inside the resolved workspace and now emit governance audit records:
- `POST /ingest/file`
- `POST /ingest/sample`
- `POST /normalize/file`

These are treated as:
- operator/developer or setup/import workflows
- auditable write-like actions
- not part of the agent-knowledge read contract
- not evidence of broad agent write-back permission

They are intentionally narrower than a general-purpose write-memory API.

---

## Audit policy

The current governance audit trail must record:
- caller identity
- effective scope
- resolved policy
- action name
- outcome
- action-specific details

As of August 8, 2026 this covers the supported Workstream B surfaces for:
- governed agent-knowledge reads
- maintenance inspect and registry reads
- maintenance refresh writes
- governance-audit history reads
- scoped ingest/import actions
- scoped normalization actions

Audit history remains inspectable through:
- `GET /ops/governance_audit`

### Governed reads (Track I4, 2026-10-09)

The first external caller's trace showed that only answers were on the record.
Every governed read surface now appends one event, with metadata only (never
content): `details.route`, the caller, the scope, what was asked for and how
much came back. `policy.access_mode` is `read_only`.

| route | `action` | `surface` | details |
| --- | --- | --- | --- |
| `GET /concepts` | `concepts_list` | `concepts` | `concept_count` |
| `GET /concepts/{key}` | `concept_read` | `concepts` | `key`, `resolved_key`, `source_count`; outcome `not_found` for an unknown key |
| `GET /wiki/page` | `wiki_page_read` | `wiki` | `path` (workspace-relative), `node_id`; outcome `denied` (403) or `not_found` (404) |
| `GET /review/source` | `maintenance_source_evidence` | (maintenance) | `path` (workspace-relative), `line_start`, `line_end`, `total_lines`, `served_from` (Track L1) |
| `POST /query/resolve_sections` | `resolve_sections` | `retrieval` | `query`, `limit`, `section_count`, `full_note_count` |
| `POST /query/gaps` | `gaps_read` | `gaps` | `categories`, `gap_count` |
| `POST /query/gaps/topic` | `topic_gaps_read` | `gaps` | `topic`, `facets`, `categories`, `scope_notes`, `gap_count` |
| `POST /query/gaps/limitation_clusters`, `/unanswered_questions`, `/weak_method_validation`, `/thesis_bridges` | `<route>_read` | `gaps` | `gap_count` |
| `POST /query/gaps/interpret` | `gaps_interpret` | `gaps` | `mode` (`caller_handoff`, `post_back`, `brain_keeper`), `topic`, `categories`, `gap_count`; `interpretations_posted` (a count, never the text) for a post-back; `billed_tokens` when the keeper ran; outcome `rejected` for a post-back that failed validation |

Track L (reading fidelity) adds, all with `surface: maintenance`:
`maintenance_fidelity_pass` (`dry_run`, `notes_checked`, `notes_with_findings`,
`counts`, `review_items`; outcome `denied` for a writing run without
`maintenance_write`), `maintenance_correction_filed` (`read_only` allowed,
`policy.resolution_basis: proposal_only`; outcome `duplicate` for a correction
already decided), `maintenance_correction_applied`,
`maintenance_correction_dismissed` and `maintenance_correction_rollback`.

`/health` and `/docs` are not audited. `GET /ops/governance_audit` filters by
`action`, `outcome`, `surface` and `caller_agent_id` (one caller's events);
filters apply to the whole log before `recent_limit`, so a busy caller cannot
push a quiet one's events out of view. `audit_path` is workspace-relative.

---

## Intentionally retained legacy shared-root behavior

The following behavior is intentionally retained after Workstream B closeout:
- truly unscoped calls still resolve to the legacy default workspace
- local compatibility and bootstrap workflows may continue to use that default when no scoped identity is provided

The following behavior is intentionally **not** retained:
- scoped requests reading or writing through the old shared root when a workspace sandbox should apply
- agent-scoped migrated routes silently bypassing the per-agent default workspace

This means the legacy shared root is now:
- a compatibility fallback
- not the normal path for scoped agent work
- not the routing target for migrated agent-scoped requests

---

## Why Workstream B is considered complete after this policy

Workstream B acceptance criteria are now satisfied because:
- agent-knowledge schemas are versioned
- scope rules are implemented and enforced
- supported write actions are auditable
- contract tests cover the supported agent-knowledge modes

The remaining path to project `100%` now belongs to other workstreams:
- Workstream A trust expansion
- Workstream D operations and recovery closeout
- Workstream C human review and correction workflow
- Workstream E performance and storage evidence
