---
type: "documentation"
title: "Kovium Architecture"
description: "How the platform is built: the layers material moves through, the modules that implement each, how storage, judgment, retrieval and maintenance fit together, and the rules that hold across all of them. It describes the code as it is. What..."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "overview"
audience: "someone meeting Kovium for the first time"
doc_path: "architecture.md"
doc_sha256: "0a22f71ab6e3c5c3f1c2c6ecb0eff920bcf010346a9736e60b7ae7728d9ae0a5"
---

# Kovium Architecture

How the platform is built: the layers material moves through, the modules that
implement each, how storage, judgment, retrieval and maintenance fit together,
and the rules that hold across all of them. It describes the code as it is.
What an instance does, from an operator's point of view, is in
`specifications.md`; every subpackage is listed in
`docs/implementation_overview.md`; the contracts are indexed in
`docs/README.md`.

This document replaces the original design proposal (August 2026), which was
written for a single research use and chose Neo4j and a React UI. Both choices
changed: SQLite is the store (`docs/persistence_decision_v1.md`,
`docs/workstream_e_performance_and_storage_decision_v1.md`) and the served UI is
a static page. The proposal's reasoning is in the repository history.

---

## 1. What the platform is for

A local-first Kovium instance for any collection: research papers, meeting
minutes, recipes, customer records, a lab notebook, a journal, in any language.
It keeps human-readable Markdown as the record, derives a knowledge graph from
it, answers questions with citations to exact lines, finds what is missing, and
keeps itself in order, for a human through a UI and for agents through a
governed HTTP contract.

One repository serves every instance. Nothing in the code names a subject: the
note types an instance keeps, its facets, its source folders, its converters,
its tail and its language are the instance's configuration. Behaviour that
cannot be universal is made configurable with a default that is safe for an
install nobody here knows.

## 2. Principles the code holds to

- **Markdown is the source of truth.** The graph, the vectors and the
  maintained state are derived and rebuildable (`scripts/rebuild_graph.py`,
  `scripts/rebuild_vectors.py`).
- **Provenance is first-class.** Every node and edge records the note, section
  and lines it came from, the stage and converter that produced it, and how much
  of the source was read.
- **Layers are never collapsed.** Each stage writes its own files, so a bad
  import is recoverable one step back.
- **Judgment is delegated, never billed.** Model work goes to the brain keeper
  on the owner's existing subscription; no metered API is reachable from the
  code, and every path that spends states its cost first.
- **Fallbacks only know their input.** With no agent, every task still
  completes, with a warning, by selecting spans of its input; nothing is
  composed.
- **Unsafe write-back is never invisible.** Corrections, merges, removals and
  approvals go through review items, snapshots and rollbacks; notes the owner
  keeps are never rewritten.
- **Agents read through a governed contract**: read-only by default, scoped,
  audited, versioned.
- **Self-contained.** No path outside the checkout or a registered workspace is
  read or written down (`second_brain/paths.py`).

## 3. The layers

```text
 raw source (sources/raw/, or a declared source root)
     │  conversion/ (pdftotext, docling worker, markitdown, web fetch), figures/, equations/
     ▼
 converted Markdown (sources/converted_markdown/) + figures/<id>/ + equations/<id>/ catalogs
     │  format_templates/, source_pipeline/ (brain keeper: extract, shape)
     ▼
 templated Markdown (sources/templated_markdown/)
     │  normalization/ (brain keeper: shape to wiki; extractive fallback)
     ▼
 wiki note (sources/wiki/)  ◄── native notes written by hand; own-work/ registrations;
     │                          ideas/; concepts/ and maps/ written by the brain
     │  ingestion/ (nodes, edges, provenance, section passages)
     ▼
 graph (data/staging/*.db: SQLite)  +  passage vectors (optional, local model server)
     │  maintenance/, fidelity/, concepts/, analytics/, tails/
     ▼
 maintained / review layer (maintenance/: work items, snapshots, state)
     │  retrieval/, agent_memory/
     ▼
 retrieval and the governed knowledge-answer layer (/agent/knowledge/*)
```

The preferred entry for anything from outside is `POST /sources/process`
(`source_pipeline/service.py`), which chains conversion, template resolution,
the shaping passes, normalization and, unless told otherwise, ingestion.
`/sources/process_book` and `/sources/process_folder` wrap it for books
(`source_pipeline/books.py`, `normalization/chapter_split.py`) and folders
(`source_pipeline/folder_intake.py`, `file_kinds.py`). `/normalize/file` and
`/ingest/file` are the lower-level steps for Markdown that is already a source
of truth.

## 4. Components

### 4.1 The service and its scoping

`api/app.py::create_app()` builds the FastAPI application and wires every
service; `tails/api.py` registers the tail routes. Every request carries
`agent_id` and `workspace_id`, resolved by `workspaces/service.py`
(`WorkspaceRegistryService`) to one workspace root with its own layout, store,
retrieval service, brain keeper state and configuration. An unregistered
workspace is a 404 and a missing one a 422; nothing is served against a
default. `paths.py` owns every path written down (`portable_path`,
`recorded_spelling`: relative, forward slashes) and `source_roots.py` the named
roots (`library:<path>`).

### 4.2 Conversion: every source to text, with its pictures and equations

- `conversion/converters.py`: the PDF converter as configuration (`docling`,
  `markitdown`, or the basic extractor), its capability profile, fallback and
  refusal; markitdown for Office, HTML and EPUB.
- `conversion/docling_worker.py`: docling in a subprocess of its own, with its
  remote options switched off, writing the copy, figure images, equation crops
  and a manifest; page markers for books (`conversion/pages.py`), measured
  seconds a page (`conversion/timing.py`).
- `conversion/web_fetch.py`: URL intake and remote images, open web only.
- `conversion/math_repair.py`, `table_repair.py`, `metrics.py`: repairs of how
  text arrived, and what a copy recovered.
- `figures/`: one catalog format for every picture (`catalog.py`), images as
  sources (`images.py`: format by magic number, captions from metadata, GPS
  never read), images inside Markdown, HTML and Word (`embedded.py`,
  `docx_media.py`), vision on request (`service.py`), book chapters
  (`books.py`).
- `equations/`: unread equations as regions with crops (`catalog.py`),
  equations written in text and MathML (`text.py`, `mathml.py`), the outline
  they sit under (`headings.py`), reading on request by the local formula model
  or vision (`service.py`), what an answer's evidence holds unread
  (`unread.py`), book chapters (`books.py`).
- `source_pipeline/reread.py`: reading a source again with a comparison and a
  snapshot; `reconcile.py`, `source_registry.py`, `tracking.py`: originals
  renamed, moved or deleted; `retention.py`: copy or reference.
- `renames/`: renaming anything, every reference following (Track AP):
  `inventory.py` (every place a name lives), `engine.py` (plan, snapshot,
  apply, rollback, recovery; the store's `move_keys`), `text.py` (links),
  `records.py` (records, the concept state, the answer cache), `hand.py`
  (renames made by hand, nightly), `ascii_pass.py` (the ASCII rule for
  existing names), `log.py` (the rename log).

### 4.3 Shaping: templates, normalization, the note contract

- `format_templates/service.py`: the twelve intake templates (Markdown files in
  `format-templates/`), their other names, which raw kinds each takes, and
  which are built without an agent (`image`).
- `normalization/`: Markdown to wiki note through the brain keeper, read in
  contiguous batches with a map of the whole document (`batching.py`), long
  papers split into an overview and sections (`section_split.py`), books into
  chapters (`chapter_split.py`), and the extractive fallback (`extractive.py`).
- `notes/`: the note contract (`contract.py`: mandatory sections and keys, type
  profiles, section roles and identity fields an install declares), facets and
  vocabulary (`facets.py`, `vocabulary.py`), relations (`relations.py`), native
  notes (`native.py`, `native_extract.py`), the names a source defines for
  itself (`identity_terms.py`), citekeys, tables, OKF export.
- `prompts/` (the folder) holds every prompt as a file; brain-keeper skills keep
  their instructions in `brain_keeper/skill_definitions/*.json`.

### 4.4 The graph and its store

- `ingestion/service.py`: a wiki note to nodes and edges with provenance: a Note
  node, items per mapped section (node type by the type profile's
  `section_nodes`), concepts, Figure and Equation nodes, typed relations,
  `IDEA_ABOUT`/`RESTS_ON` for ideas, `WORK_ABOUT`/`COMMENTS_ON`/`POSITIONS_AGAINST`
  for the owner's work, and section passages for embeddings.
- `storage/sqlite_store.py`: the active `GraphStoreProtocol` implementation
  (schema version in `config/schema_version.json`; migrations are reviewed, not
  automatic). `storage/neo4j_store.py` exists and is not used.
- `graph/`: the in-memory store and the query service (subgraph, focus,
  related, path, relation chains, centrality, clusters).
- `community/detection.py`, `analytics/` (`gap_engine.py`, `limitations.py`,
  `topic_scope.py`, `gap_filler.py`), `interpretation/gap_interpreter.py`: gaps,
  shared limitations, gaps under a topic, the proactive filler that only
  proposes.

### 4.5 Retrieval and the governed answer

- `retrieval/service.py`: ranking by words, facets and their aliases, concept
  hierarchy and, when a local model server answers, passage embeddings
  (`embeddings.py`, `passage_index.py`); boosts by section role; long questions
  split and fused; semantic expansion onto the vocabulary (`semantic_expand.py`).
- `agent_memory/service.py`: the governed contract behind `/agent/knowledge/*`
  (`agent_knowledge/` is the public-name layer). `focus.py` decides where an
  answer rests; `reading.py` is the one-call read; `views.py` the compact view;
  `conversations.py` sends a caller only what is new; `synthesis.py` the
  optional written answer. `fidelity/answer.py`, `figures/answer.py` and
  `equations/unread.py` add the trust signals for unverified values,
  image-derived statements and unread regions.
- `ideas/` and `owner_work/`: the two tiers that are never evidence, listed
  apart (`related_ideas`, `from_your_work`), with their own review items;
  `owner_work/collection.py` links an owner's note to what it is about.
- `concepts/`: one maintained note per concept (`notes.py`, `service.py`),
  curation and merges (`curation.py`), current names for a reused extraction
  (`current_names.py`).

### 4.6 The brain keeper

`brain_keeper/`: every judgment task is a skill (`skills.py`,
`skill_definitions/`) run as a sealed one-off call to Claude Code, Codex or
OpenClaw (`runtime.py`), with a fallback order, cooldowns per failure
(`cooldowns.py`, `failures.py`), host-wide concurrency (`host_limits.py`), a
long lane for books, folders and docling work (`jobs.py`), an answer cache keyed
on skill, definition, input, tier and tail, with advisory inputs left out
(`cache.py`), sessions for the routes that continue a conversation
(`sessions.py`), and the billing guard that strips metered credentials before
any CLI starts (`billing_guard.py`). Vision jobs give Claude Code its file-read
tool alone in a fresh folder, or attach the image for Codex. `metrics/` counts
every call and its tokens.

### 4.7 Maintenance, review and safety

- `maintenance/service.py`: lint, review queue and work items, repairs that
  touch no meaning, refresh and inspection; `fallback_notes.py` finds notes an
  older fallback wrote; `schedule.py` and `scripts/nightly_maintenance.py` the
  nightly run.
- `fidelity/`: values from a lost layout marked at intake, the fidelity pass,
  checked conflicts, corrections with snapshots, a source read in full pages.
- `indexing/`: the facet vocabulary's lifecycle (candidates, promotion, import).
- `tails/`: the instance tail: schema and screen, the deterministic builder,
  equivalence tests for glossary variants, selection by job, maintenance on a
  cadence.
- `security/`: fetched text scanned for instructions addressed to a model, and
  quarantine.
- `ops/`: backup, export, restore and its rehearsal, integrity, schema status,
  capability checks (converters, vision, semantic search).

### 4.8 The UI

`/` serves a static page (`services/ui/templates/index.html`,
`services/ui/static/app.js`) drawing the graph as SVG, with focus, canvas and
side-panel routes behind it (`/view/*`) and the wiki and provenance views.
`services/ui/react_app/` is a React and Cytoscape scaffold, not the served UI.

## 5. Data model

The graph holds Note nodes, one per wiki note, and the items its sections
produce: claims, evidence, questions, methods, results, limitations, decisions,
action items and whatever node types an install's own types declare, plus
Concept, Figure, Equation and `Statement` nodes (the last for a native note's
own headings). Edges include `MENTIONS`, `SUPPORTS`, `USES`, `ADDRESSES`,
`EVALUATED_ON`, `OPEN_QUESTION_FOR`, `RELATES_TO`, `PART_OF`, `FOLLOWS`,
`BROADER_THAN`, the typed relations a note states (`REQUIRES`, `BUILDS_ON`,
`CONTRADICTS`, … from `config/relations.json`), and the tier edges above. Every
node and edge carries its provenance (note path relative to the workspace,
section, lines, extractor, confidence). `schema/ontology_v1.md`,
`schema/graph_schema_v1.md` and `docs/ingestion_spec_v1.md` are the detailed
references; the first two predate the SQLite decision and list the original
node labels.

Confidence follows where a statement came from: the source's text highest; an
equation read from the text 0.9; a caption 0.85; a formula-model reading 0.8;
a reading of a picture 0.6, and an answer item resting on one says
`image_derived`.

## 6. What an answer goes through

1. The question is scoped (workspace, facets, a caller's `scope`), widened onto
   the vocabulary (cache first, at most one brain-keeper call), split if long.
2. Retrieval ranks items by words, tags, linked concepts and meaning, weighted
   by section role; ideas and the owner's work are skipped.
3. The governed layer assembles the envelope: items with provenance,
   confidence and coverage of the question, warnings, `focus` with the read
   call, `unread_regions` with their recover calls and estimates, concept notes,
   `related_ideas`, `from_your_work`; compact or full; audited.
4. A caller reads the focus sources in one call (`/agent/knowledge/read`) and,
   only with consent, recovers unread equations or figures.

## 7. Reliability

- Backups, exports (including OKF), restores and restore rehearsals
  (`/ops/backup`, `/ops/restore/rehearsal`); integrity and schema status.
- Every write a maintenance route makes is snapshotted and can be rolled back:
  corrections, re-reads, equation readings, source reconciliation, concept
  merges.
- Rebuild strategy: the Markdown is enough. Rebuild the graph, then the
  vectors; the brain keeper's answer cache makes re-processing an unchanged
  source free.
- Logs: agent calls with tokens (`logs/agent_calls.jsonl`), conversion timings,
  equation reads, the governance audit.

## 8. Agent access

Agents use the governed routes and the shipped skills (`skills/`), never the
store or the files. An MCP wrapper over the governed routes is a possible later
step (`docs/gbrain_reference_v1.md` maps one); nothing depends on it.
