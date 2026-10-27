---
type: "documentation"
title: "Kovium Onboarding (Claude Code)"
description: "Create, configure and verify a Kovium instance from Claude Code, for a collection on any subject. Use once when you are given a Kovium instance to set up or operate, or whenever an existing instance stops answering correctly and needs re-verifying from the ground up."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "skill"
audience: "the AI program that works with Kovium"
doc_path: "skills/claude-code/kovium-onboarding/SKILL.md"
doc_sha256: "58b52d555f227f01b0ee795c3e990527e11092f6b79c97e8ee1b92386dbddfbe"
---

# Kovium Onboarding (Claude Code)

## Purpose

Bring one Kovium instance from nothing to verified-working, and prove it
works before any real knowledge goes into it.

Run this **once** per instance; afterwards use `kovium-operations`. If
answers later lack provenance or notes arrive half-empty, return to **Step 6**,
the diagnostic.

**A person new to Kovium needs none of this.** Over MCP, `welcome`
starts them on the example and makes or opens their own collection with
nothing typed (`GET /welcome`, `POST /collections` over HTTP); `help` answers
how anything works. This skill sets up and checks an instance.

## What you are onboarding

Kovium turns Markdown, and documents converted to it, into a knowledge
graph and answers from it with citations to the exact line of the exact note.
An instance can hold anything, in any language; its configuration, never the
platform, says what. You reach it over a loopback HTTP API (default
`http://127.0.0.1:8000`), never by reading its database or editing its derived
files.

## Step 1 — Confirm the service is up

```bash
curl -s http://127.0.0.1:8000/health
```

Expect `{"status":"ok", ...}`. If this fails the service is not running; start
it from the checkout root with `python scripts/run_kovium.py`, which works
on every platform (`./scripts/run_kovium.sh` on Linux and macOS does the
same). Live route documentation is at `/docs`.

`curl` as written needs nothing special on any shell; the one Git Bash case
is in `reference/configuration.md`.

## Step 2 — Fix your coordinates, and never drop them

Every instance is a **scoped workspace**. You have two identifiers:

- `agent_id` — who is asking
- `workspace_id` — which instance is being asked

**Send both on every single call.** This is the most important line in this
skill. A call that omits `workspace_id` is refused with `422`; one naming a
workspace this instance does not know is refused with `404`. Nothing is served
against a default, so a typo fails loudly instead of quietly creating a second,
empty brain. Where its access keys are on, also send `Authorization: Bearer <key>`
(`docs/access_tokens_v1.md`).

Record both where this project keeps its own notes, so a later session cannot
guess wrong.

## Step 3 — Register the workspace

A workspace is brought into existence deliberately, by this call or by
`POST /collections` (what the setup tools call):

```bash
curl -s -X POST http://127.0.0.1:8000/ops/workspaces \
  -H 'Content-Type: application/json' \
  -d '{"workspace_id":"<workspace_id>","agent_id":"<agent_id>"}'
```

Expect `201`. A `409` means it is already registered — carry on. By default
it lands under `sandboxes/<workspace_id>/` in the checkout. To keep the notes
elsewhere (a vault, a synced folder), add an absolute `"root_path"`. A relative one, or one nested inside another
workspace, is refused. `GET /ops/status` with both ids then shows the layout:

```
sources/raw/  sources/converted_markdown/  sources/templated_markdown/  sources/wiki/
data/staging/ (the SQLite graph)   data/backups/   data/exports/
maintenance/   logs/   config/
```

`sources/wiki/` holds the notes; `config/` holds this instance's settings.
Everything under `data/` is **derived** and rebuildable — never hand-edit it,
never read it directly. Paths you send are resolved by the **service**: a
relative path resolves against the checkout root
(`sandboxes/<workspace_id>/sources/raw/note.md`).

## Step 4 — Say what the instance holds

Ask the operator what the collection is and write it into the workspace's
`config/`, not into the platform: each file holds only the keys that differ
from the install's defaults, and is read when the workspace is first used in
a running service, so **write it before the first call, or restart**.

- **Kinds of note.** The shipped kinds (`GET /format-templates`: papers,
  white papers, book chapters, web pages, blog posts, meeting notes, decision
  records, documentation, chat and AI-conversation transcripts, voice notes, videos, images)
  cover intake. A kind of note the owner writes that is not among them (a
  recipe, an account record, a lab entry) is declared in `config/note_types.json`
  with its sections, which of them are answers, questions, limitations or
  context, and the fields that identify one.
- **Facets.** Which facets each kind is asked for, and later the values
  (`config/facet_vocabulary.json`). Start without values unless the owner
  already has a classification scheme.
- **Where the files live.** `config/sources.json`: `copy` into the workspace
  (the default) or `reference` with named `source_roots`.
- **The owner's own notes**, in another language: heading patterns in
  `config/owner_work.json`.

The settings, with examples: `reference/configuration.md`.

## Step 5 — Configure the brain keeper agent, and what it may spend

Judgment work — extraction, shaping, faceting, reading pictures — is delegated
to the *brain keeper agent*: one sealed, one-off call per job to an agent CLI on
an existing paid subscription (Claude Code first, then Codex, then OpenClaw, by
default). It has no tools and no memory; nothing is registered or left running.
**By default there is nothing to write.** To change it for this workspace
only, write just the keys that differ to `config/brain_keeper.json`:

```json
{
  "backend": "claude_code",
  "fallback": ["codex", "openclaw"],
  "model_tier": "balanced"
}
```

`model_tier` is `strong`, `balanced` or `fast`. Check what is ready with
`.venv/bin/python scripts/brain_keeper_check.py` (`.venv\Scripts\python` on
Windows). **There is no identity to register** on any backend; the `claude`
CLI signed in with a Claude subscription is the whole requirement for Claude
Code.

Then decide, with the operator, what the collection needs turned on. Each is
off or minimal by default because it costs time or subscription tokens:

| what | setting | default |
|---|---|---|
| a PDF converter that keeps columns, tables, figures and equations | `pdf_converter` in `config/conversion.json` (`docling` needs `pip install -e ".[docling]"`) | the basic extractor |
| Word files, web pages, EPUB | the `markitdown` extra or the `oc-markitdown` CLI | refused without one |
| reading pictures during intake | `vision.enabled` in `config/ingestion.json` (a backend that takes images: Claude Code or Codex) | off; on request only |
| a book's pictures, and how long a book may take | `vision.books`; `docling_book_page_ceiling` | off; 150 pages |
| the instance's tail | `config/tails.json` `enabled`, once a tail is drafted, approved and measured | off |
| meaning-based search | `config/embeddings.json`, with a local model server | on; says when unavailable |

## Step 6 — Verify before trusting

Run all four. Each catches a different failure.

**6a. Is the backend actually usable?**

```bash
curl -s "http://127.0.0.1:8000/brain_keeper/skills?agent_id=<agent_id>&workspace_id=<workspace_id>"
```

In `status`, require:
- `resolved_backend` is `claude_code`, **not** `deterministic`
- `backend_readiness.claude_code.available` is `true`

If unavailable, `backend_readiness.claude_code.reason` names the problem — most
often the CLI not being on PATH. Read the reason; do not guess.

**6b. Can this instance bill anyone?**

```bash
curl -s "http://127.0.0.1:8000/ops/billing_safety?agent_id=<agent_id>&workspace_id=<workspace_id>"
```

Require `safe: true` and `billable_env_vars_present: []`. The platform must
never make a metered per-token API call — all model work runs through the CLI on
an existing subscription. If this is not clean, **stop and report** rather than
working around it. An exported `ANTHROPIC_API_KEY` in your environment is the
usual cause, and the platform strips it from subprocesses by design.

**6c. Is the instance internally healthy, and what can it read?**

```bash
curl -s "http://127.0.0.1:8000/ops/self_check?agent_id=<agent_id>&workspace_id=<workspace_id>"
```

Read `pdf_converters` (`effective_for_pdf`), `document_converter` and
`figure_vision`: they must say what Step 5 decided.

**6d. One real note, end to end.** The only test that proves judgment actually
reaches the brain-keeper. Write a short Markdown file into
`sandboxes/<workspace_id>/sources/raw/` containing a few factual statements and
at least one open question, then:

```bash
# 1. normalize: raw Markdown -> wiki-form note
curl -s -X POST http://127.0.0.1:8000/normalize/file \
  -H 'Content-Type: application/json' \
  -d '{"path":"sources/raw/<file>.md","agent_id":"<agent_id>","workspace_id":"<workspace_id>"}'

# 2. ingest the normalized note into the graph (paths are workspace-relative,
#    as every result returns them)
curl -s -X POST http://127.0.0.1:8000/ingest/file \
  -H 'Content-Type: application/json' \
  -d '{"path":"<normalized_path from step 1>","agent_id":"<agent_id>","workspace_id":"<workspace_id>"}'

# 3. ask about it
curl -s -X POST http://127.0.0.1:8000/agent/knowledge/answer \
  -H 'Content-Type: application/json' \
  -d '{"agent_id":"<agent_id>","workspace_id":"<workspace_id>","query":"<something the note answers>"}'
```

## Connected over MCP

The install step (`python scripts/setup.py`) installs the extra and connects
each agent program found, in welcome mode: Claude Code as a plugin with these
skills; Codex, OpenClaw by connection (by hand: `docs/mcp_server_v1.md`). It reads; `request_tools` widens the set
within the workspace's `config/mcp.json` (`max_surface`). With `allow_proposals`, `remember` and `forget` only propose:
writes are the owner's. Without `--workspace` it serves the welcome
(`welcome`, `help`, `get_manual`, `try_example`, `tour`) and `create_collection` or
`open_collection`, which bind it; `join_collection` joins one agent alone. An agent the owner lets manage it
(`set_access`) also gets `connect_agent`, `change_setting` and `make_agent` (the
collection's own agent, beside the project's); `check_setup` says what works.
Where a collection asks for access keys, call `sign_in` first with `token_file`, your key file.

## Acceptance criteria

Onboarding is complete only when **all** of these hold:

| check | required value |
|---|---|
| `resolved_backend` | `claude_code`, not `deterministic` |
| `billing_safety.safe` | `true`, with no billable env vars |
| normalized note's `## Normalization Provenance` | `mode: brain_keeper` (a note from `/sources/process` names its template and has `template_runtime_mode: brain_keeper`) |
| same section | `source_coverage: 100.00% (<n> of <n> chars)` |
| `/ingest/file` | non-zero `node_count` and `edge_count` |
| `/agent/knowledge/answer` | `knowledge_items` whose nodes carry `source_refs` with real line numbers |
| `/ops/self_check` | the converters and vision the operator chose, available |

Then **read the normalized note** — use your Read tool; this is the one place
where reading a file directly is the right move, because you are auditing the
output rather than consuming knowledge. Mandatory sections must be populated and
the content must be true to the source: claims separated from open questions,
qualifiers preserved. If claims and questions are mixed, or a qualifier like
"unless X" has been dropped, extraction is not working properly even with every
status light green. Status checks cannot catch this — only reading can.

## Failure modes, and what each means

| symptom | cause | fix |
|---|---|---|
| `mode: heuristic`, `fallback_reason: brain_keeper_unavailable` | backend unreachable | read `backend_readiness.claude_code.reason` |
| `resolved_backend: deterministic` | no usable backend; usually `claude` not on PATH | fix PATH, restart the service |
| `422` or `404` on every call | `workspace_id` omitted, or not registered | Step 2 and Step 3 |
| `source_coverage` below 100% | part of the input never reached the model | do not accept the note; re-run, then report |
| `billing_safety` not clean | a metered key exported in the environment | unset it; never pass it through |
| a note is on disk but answers never find it | written into `sources/wiki/` and never ingested | `POST /ingest/file` on it, or rebuild the graph |
| a setting seems ignored | written after the workspace was first used | restart the service |

## Onboarding the collection

Only once acceptance passes. Add sources **a few at a time**, checking quality
before scaling up — a systematic extraction problem found after 200 documents
costs far more than one found after 3. For a folder, use
`kovium-intake`; for everything after, `kovium-operations`.
