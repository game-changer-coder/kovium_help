---
type: "documentation"
title: "Kovium"
description: "A local-first, provenance-backed, workspace-aware knowledge platform. It turns your written material — Markdown notes, PDFs, Word files, web pages, books, images, transcripts — into a structured, queryable knowledge graph with retrieval..."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "overview"
audience: "someone meeting Kovium for the first time"
doc_path: "README.md"
doc_sha256: "cfd686a7beafb22f3ae371def0cd3af7b32c6ead2f76e1cad709917b12d0addf"
---

# Kovium

A local-first, provenance-backed, workspace-aware knowledge platform. It turns
your written material — Markdown notes, PDFs, Word files, web pages, books,
images, transcripts — into a structured, queryable knowledge graph with
retrieval, gap analysis, agent-facing knowledge APIs, review and maintenance
workflows, and a UI.

It is a platform, not one collection. One instance holds research papers;
another holds meeting minutes, a recipe box, customer records, a lab notebook
or a journal, in any language. Nothing in the platform assumes the subject:
each instance says what it holds in its own configuration.

Markdown is always the canonical source of truth. Everything derived — wiki
pages, graph nodes and edges, maintained state — stays rebuildable from it.

**No metered billing.** Every piece of work that requires judgment is handed
to a *brain keeper*: a one-off call to an agent CLI (Claude Code, Codex or
OpenClaw) that runs on your own already-paid subscription. Kovium itself
never calls a paid model API and never holds an API key, and everything that
spends that subscription states its cost first. See [Billing rule](#billing-rule).

**New here?** Follow GETTING_STARTED.md: a step-by-step guide for Claude Code,
Codex and OpenClaw, from the download to your first answer.

## Install, in one step

```bash
python scripts/setup.py
```

The same command on Linux, macOS and Windows, with any Python 3.10 or later.
It makes the private environment (`.venv`), installs Kovium into it,
checks the AI programs on this computer and their sign-in, says in plain words
which optional helpers are missing and what each brings, and connects Kovium
to every AI program it finds: Claude Code as a plugin (the connection,
the skills and `/kovium:start`), Codex, Cursor and OpenClaw by a
connection each. It needs no administrator password, spends no token, and a
second run changes nothing. Then open your AI program and say **"help me
start with Kovium"**: nothing else is typed, and there is no server to
keep running. Once you have a collection, your program can add an agent of its
own for it, in the program's own format: a Claude Code agent file, a Codex
custom agent (ask for it by name), or a marked section of an OpenClaw
workspace's `AGENTS.md`, always beside what is already there. The contract:
`docs/install_and_caller_agent_v1.md`.

## Try it in five minutes, over HTTP

```bash
./scripts/run_kovium.sh
```

On Windows, or anywhere without bash: `python scripts/run_kovium.py`
(`--port`, `--skip-install`; `--help` for the rest).

This creates a `.venv` if one doesn't exist, installs the package, and starts
the API on `http://127.0.0.1:8000`. UI: `http://127.0.0.1:8000/?workspace_id=<id>` ·
interactive API docs: `http://127.0.0.1:8000/docs`.

A small example collection ships in the repository,
`sandboxes/hello_brain/`: a few short pages
about Kovium itself, their notes already built. Connected over MCP (see
`docs/mcp_server_v1.md`), say "help me start with Kovium" to your agent:
it calls `welcome`, opens the example, answers a suggested question with its
sources and offers a short tour, with no command typed and no tokens spent
(`docs/welcome_v1.md`). Then "make my own collection", or "open my collection
in this folder", makes or opens one and binds the conversation to it
(`docs/collections_v1.md`). "Make me a Kovium agent" adds that
collection's own agent to your project, beside your own agents, answering in
the style you choose (`docs/install_and_caller_agent_v1.md`). Over HTTP, with
no registration first:

```bash
# Open the example: its graph, concept pages and maps are built by code in seconds
curl -X POST http://127.0.0.1:8000/welcome/example -H 'Content-Type: application/json' -d '{}'

# Ask it a question; the answer lists where each part comes from
curl -X POST http://127.0.0.1:8000/welcome/example -H 'Content-Type: application/json' \
  -d '{"question":"What does a note hold?"}'

# A question about the software itself, answered from the user guide
curl 'http://127.0.0.1:8000/help?question=What%20uses%20my%20subscription'
```

`./scripts/seed_sample.sh` registers a workspace of sample notes (`sample`, or
`$WORKSPACE_ID`) and prints the link to open.

## Starting an instance of your own

1. **Register a workspace**: `POST /ops/workspaces` with a `workspace_id` (and
   an absolute `root_path` if the collection should live outside `sandboxes/`,
   in an existing vault or a synced folder). Every later call names it.
2. **Say what it holds**, in the workspace's own `config/`: the kinds of note
   you keep, with their sections, roles and identifying fields
   (`note_types.json`); the facets each is asked for and their values; whether
   originals are copied in or read where they lie (`sources.json`).
3. **Turn on what the collection needs** (each off or minimal by default,
   because it costs time or subscription tokens): a PDF converter
   (`conversion.json`), reading figures and images by vision (`ingestion.json`),
   a ceiling on how long a book may take, an instance tail (`tails.json`).
4. **Verify** that a real backend answers (`GET /brain_keeper/skills`), that
   nothing can bill (`GET /ops/billing_safety`) and that the converters you
   chose are in effect (`GET /ops/self_check`).
5. **Take in sources**: one at a time (`POST /sources/process`), a folder with
   a free dry run first (`POST /sources/process_folder`), books with their page
   count and time stated first (`POST /sources/process_book`). Register your own
   notes rather than processing them (`POST /owner-work`).

[[specifications|`specifications.md`]] is the full guide, and section 26 is
the checklist. To have an agent do this, install the skills under
`skills/`.

## Optional extras

```bash
pip install -e ".[docling]"      # PDFs with columns, tables, figures and equations (large: brings PyTorch)
pip install -e ".[markitdown]"   # Word files, web pages and EPUB
pip install -e ".[dev]"          # the test suite's own dependencies
```

Without docling, a PDF is read by the basic extractor (`pdftotext`), which loses
two-column order, tables, equations and figures, and every note says so.
Without markitdown (the Python package or the `oc-markitdown` CLI), Word files
and web pages are refused. Images need nothing extra. A local model server for
passage embeddings is optional and reported when absent. `GET /ops/self_check`
says what this install has.

## What it does

- **Takes in sources of any kind** through a layered pipeline: documents
  converted by a converter the workspace chooses; images as sources; figures
  and equations kept from every kind of document; books split into chapters
  with their pages; whole folders triaged before anything costs a token, each
  file with its own kind. An unchanged source is never paid for twice.
- **Reads notes you write yourself**: a note written into the wiki by hand
  counts as much as an imported one, read under its own headings. The owner's
  own work (notes on a meeting or a topic, designs, commentary) and ideas are
  kept as tiers of their own, linked to what they are about, and never cited as
  evidence.
- **Answers in two or three calls**: a governed answer says where it rests
  (`focus`), one call reads the relevant parts of those sources, and equations
  or figures the evidence holds unread are read only when the question needs
  them and the user agrees to the stated cost.
- **Retrieves by reading text**, not by handing back a bare node label as if it
  were an answer — a graph label is a title, not a prose response. Ranking is
  driven by the role of each section, so a meeting's decisions or a recipe's
  steps rank like a paper's methods.
- **Analyses gaps**: unanswered questions, weak validation, missing bridges,
  limitations several sources share, gaps under one topic.
- **Maintains itself**: concept notes kept up to date, duplicate and drift
  detection, renamed originals found again by their bytes, reading-fidelity
  checks, a closed facet vocabulary, an optional instance tail, and a nightly
  job that costs nothing on a collection that hasn't changed.
- **Scopes work by workspace.** Every instance is its own folder — its own
  notes, config, database and agent state — so one deployment serves very
  different collections without either shape distorting the other.
- **Exposes a governed contract** (`/agent/knowledge/*`) for downstream
  consumers: read-only by default, explicit scope, durable audit records.

## Architecture, in one picture

Content moves through distinct layers — never collapsed:

```
raw source → converted Markdown (+ figure and equation catalogs) → templated Markdown
    → normalized wiki → ingested graph (nodes/edges/provenance) → maintained/reviewed wiki
    → retrieval / governed knowledge-answer layer
```

Each layer is derived from the one before it and stays rebuildable from it.
[[architecture|`architecture.md`]] maps the layers
onto the `second_brain/` package; [[implementation-overview|`docs/implementation_overview.md`]]
lists every subpackage.

### The note contract

Every note — whatever it came from — carries eight mandatory sections and eight
mandatory frontmatter keys, so a reader browsing the corpus gets the same entry
points regardless of note type. Type-specific sections layer on top (a paper has
`## Methods`; a meeting note has `## Decisions`), and an instance declares its
own kinds. Full spec: [[note-format-contract-v1|`docs/note_format_contract_v1.md`]].

### The brain keeper agent

Judgment-requiring work — extraction, shaping, faceting, concept notes, gap
interpretation, reading a figure or an equation from its image — is handed to
the **brain keeper agent**, run on an agent CLI the user already pays for by
subscription: Claude Code first, then Codex (signed in with ChatGPT), then
OpenClaw, in the order `config/brain_keeper.json` gives. Each job is one sealed,
one-off call: the job travels in the message, the agent has no tools (a vision
job's confined file-read excepted), no memory and saves nothing, and the Python
service validates and applies whatever comes back. Nothing is registered with
any backend and nothing stays running.

When a backend runs out of quota, fails to sign in or does not serve the model,
the same job moves to the next one and that backend rests for a while.
`model_tier` (`strong` / `balanced` / `fast`) picks capability without pinning a
vendor model id. A backend signed in only with a metered API key is refused.
With no backend ready, work degrades to a deterministic fallback that only
copies sentences of its input, with an explicit warning — it never hard-fails
and never invents.

```bash
.venv/bin/python scripts/brain_keeper_check.py          # what is ready, and how to fix what is not
.venv/bin/python scripts/brain_keeper_check.py --live   # plus one tiny real job per ready backend
```

## Commands

```bash
# tests (tiered: unit / integration / system / contracts; the full suite takes ~12 minutes)
.venv/bin/pytest -q
.venv/bin/pytest -q tests/unit/test_retrieval_service.py::test_name
# in a random order fixed by a seed (files, and the tests inside each, are shuffled);
# `random` draws a seed and prints it, the same seed repeats the order
.venv/bin/pytest -q --shuffle-seed random      # or KOVIUM_TEST_SHUFFLE_SEED=1234
# Every test fails at teardown, by name, if it leaves an environment variable or the working
# folder changed, or a file in the checkout's config/, data/, sources/, logs/ or sandboxes/;
# caches of settings files are dropped before each test (tests/conftest.py).

# operational checks
.venv/bin/python scripts/ops_status.py
./scripts/ops_self_check.sh
.venv/bin/python scripts/ops_integrity_check.py

# evaluation (run when touching retrieval/normalization ranking)
.venv/bin/python scripts/eval_retrieval_benchmark.py
.venv/bin/python scripts/eval_normalization_regressions.py

# rebuild everything derived from the Markdown
.venv/bin/python scripts/rebuild_graph.py --workspace-root <workspace root>

# per-workspace nightly maintenance — on in every collection made by create_collection;
# for an older collection, turn it on with change_setting, or here
.venv/bin/python scripts/maintenance_cron.py enable  <workspace_id> --at 02:00
.venv/bin/python scripts/maintenance_cron.py status
.venv/bin/python scripts/maintenance_cron.py disable <workspace_id>
```

## Self-contained, and safe to publish

Kovium depends on nothing outside its own tree or a workspace assigned
to an instance. No dependency on another project's files or formats, no
private corpus, no credential, no machine-specific path. Real workspaces
(`sandboxes/*`, other than `hello_brain`) and instance tails are per-machine
runtime state and are never committed. Enforced by
`tests/system/test_self_contained_repository.py`
and written into `CLAUDE.md` as a standing rule.

## Billing rule

**Kovium must never cause a metered, per-token charge.** All model work
goes through the brain keeper, which runs on your own paid subscription.
Nothing in this project may create a cost beyond that. No calls to a paid
model API, no metered credentials passed to a subprocess, no direct HTTP to a
non-loopback model endpoint. Enforced by
`tests/system/test_billing_safety.py`
and checkable live via `GET /ops/billing_safety`. Full statement in
`CLAUDE.md`.

## Documentation map

| doc | covers |
| --- | --- |
| [[specifications|`specifications.md`]] | **start here**: what an instance does, every feature, what is configuration and what is fixed |
| `skills/` | **operating** an instance — load these into the agent that will run it |
| [[which-documents-are-current|`docs/README.md`]] | every document, sorted into current contracts, guides, open plans and history |
| [[api-spec-v1|`docs/api_spec_v1.md`]] | the HTTP routes and their contracts |
| [[agent-memory-contract-v1|`docs/agent_memory_contract_v1.md`]] | the governed `/agent/knowledge/*` contract |
| [[note-format-contract-v1|`docs/note_format_contract_v1.md`]] | the note contract, facet vocabulary, lint, nightly maintenance |
| [[architecture|`architecture.md`]] | how the code is organised, layer by layer |
| `CLAUDE.md`, `project_guidelines.md` | the rules for changing the platform |
| `IMPLEMENTATION_JOURNEY.md` | the persistent implementation history, newest first |

## License

MIT.
