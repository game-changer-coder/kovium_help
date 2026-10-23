---
type: "documentation"
title: "Implementation overview"
description: "A map of the code: what each part of the repository is, so a reader can find where a behaviour lives. How the parts fit together is in second_brain_architecture.md; what the platform does is in specifications.md; the contracts are indexed..."
status: "stable"
reading_status: "read"
product: "second brain"
product_version: "0.1.0"
doc_kind: "guide"
audience: "a person setting up or using second brain"
doc_path: "docs/implementation_overview.md"
doc_sha256: "0723bd9f8d97a04cfaff82ccb4c63553c641078b0a3fd9afe37b41d4ba2f9e19"
---

# Implementation overview

A map of the code: what each part of the repository is, so a reader can find
where a behaviour lives. How the parts fit together is in
`second_brain_architecture.md`; what the platform does is in
`specifications.md`; the contracts are indexed in `docs/README.md`.

This file once held the first version's plan (a research-only tool, four
modules, Markdown only). That plan is history; this is the current map.

## The repository

| path | what it holds |
|---|---|
| `second_brain/` | the Python package: the service and every module below |
| `config/` | the install's default settings; a workspace overrides any of them in its own `config/` |
| `format-templates/` | the fourteen intake templates, as Markdown (`documentation` since Track AG1) |
| `prompts/` | every model prompt, as a file |
| `schema/` | the graph ontology, the graph schema (written before the SQLite decision) and the gap categories |
| `scripts/` | operator scripts: start, seed, rebuild, migrate, maintain, evaluate, measure |
| `skills/` | the operator skills shipped to caller agents, per framework |
| `services/ui/` | the UI served at `/` (a static page and script) and a React scaffold |
| `sandboxes/` | workspaces; only `hello_brain`, the example collection the welcome opens (Track AG1), is committed |
| `tests/` | `unit/`, `integration/`, `system/`, `contracts/`, and synthetic `fixtures/` with their generators |
| `eval/` | committed evaluation inputs and results: retrieval, normalization, and the workstream A and E evidence |
| `docs/` | contracts, guides, plans and records (`docs/README.md` sorts them) |

## The package, `second_brain/`

| subpackage | responsibility |
|---|---|
| `api/` | `app.py::create_app()`: the FastAPI application, request models and every route but the tails' |
| `workspaces/` | the workspace registry: one root, store, retrieval service and configuration per `workspace_id` |
| `paths.py`, `source_roots.py`, `file_stamps.py` | how a path is written down (relative, forward slashes), named source roots, file stamps for caches |
| `source_pipeline/` | `POST /sources/process` and its book and folder forms; per-file kinds; retention; re-reading; the source registry and reconcile |
| `renames/` | renaming anything with every reference and key following: the inventory, the engine (snapshot, rollback, recovery), link rewriting, records, renames made by hand, the ASCII pass, the rename log |
| `conversion/` | PDF converters (docling worker, markitdown, basic), web fetch, page markers, maths and table repair, conversion metrics and timings |
| `figures/` | the figure catalog for every kind of source: docling figures, images as sources, images inside Markdown, HTML and Word; vision on request |
| `equations/` | equation regions and catalogs, equations from text and MathML, the outline they sit under, reading on request, unread regions in answers |
| `format_templates/` | the intake template registry and resolution |
| `normalization/` | Markdown to wiki note: batched reading, section and chapter splits, the extractive fallback, regression checks |
| `notes/` | the note contract and type profiles, facets and vocabulary, relations, native notes, identity terms, citekeys, tables, OKF export |
| `ingestion/` | a wiki note to graph nodes, edges, provenance and passages |
| `storage/` | the SQLite graph store (active) and a Neo4j store (not used) |
| `graph/` | the in-memory store and graph queries |
| `retrieval/` | ranking, question decomposition, semantic expansion, passage embeddings, the retrieval benchmark |
| `agent_memory/`, `agent_knowledge/` | the governed contract: answer, read, task context, read next; focus; compact views; conversations; synthesis |
| `concepts/` | concept notes: sync, hierarchy, merges, curation, comparison, current names |
| `ideas/` | idea notes: capture, listing, linking, review |
| `owner_work/` | the owner's registered work: kinds, headings, what it is about, comparison with sources, review |
| `analytics/`, `interpretation/`, `community/` | gaps, shared limitations, topic scopes, the proactive gap filler, the gap interpreter, clusters |
| `fidelity/` | reading fidelity: layout-lost values, checks, corrections, paging a source |
| `maintenance/` | lint, the review queue and work items, repairs, refresh, inspection, fallback-note detection, the schedule |
| `indexing/` | the facet vocabulary's lifecycle: candidates, promotion, import |
| `tails/` | instance tails: schema and screen, builder, glossary equivalence, selection, maintenance, routes |
| `brain_keeper/` | the agent backends, skills and their definitions, jobs and lanes, the answer cache, sessions, cooldowns, the billing guard |
| `metrics/` | every agent call and its tokens, and their sums |
| `security/` | the instruction scan over fetched text, and quarantine |
| `ops/` | backup, export, restore, integrity, schema status, capability checks |
| `canvas/` | the graph as a canvas for the UI |
| `prompts/` | resolving prompt files |
| `catalogue/`, `plain_words.py` | the option catalogue (plain words for every option a person can meet, four lines each) and the one plain-words checker shared with the code texts (Track AG0); the welcome's texts beside it (`welcome_v1.json`, Track AG1) and the collection tools' (`setup_v1.json`, Track AG2) |
| `collection_setup/` | collections and who may do what (Track AG2): making and opening a collection, what did not travel, the access policy per agent, `change_setting`, the check of what works, adding a collection's connection to an agent program (`docs/collections_v1.md`) |
| `welcome/` | the first five minutes (Track AG1): the welcome's state, the example collection opened and built by code, its tour and waiting page, help from the user guide and the catalogue (`docs/welcome_v1.md`) |
| `providers/` | the older model-runtime abstraction: a deterministic gap runtime and a local-server normalization runtime nothing constructs today; no metered provider |
| `performance/` | the storage and concurrency evaluation behind the storage decision |
| `cli/` | the sample seeding and single-source commands the scripts call |
| `models/` | shared dataclasses (`Node`, `Edge`, `GapRecord`, provenance and maintained-state types) |

## The guard tests worth knowing

| test | keeps true |
|---|---|
| `tests/system/test_billing_safety.py` | no metered API is reachable from the code |
| `tests/contracts/test_option_catalogue.py` | every option has its four lines and a "later" line, every text passes the plain-words rule, every setting has an entry or a declared reason, the user guide names every entry |
| `tests/system/test_self_contained_repository.py` | no user-naming path or credential in any tracked file |
| `tests/system/test_prompts_are_domain_neutral.py` | shipped prompts, skills and templates carry no one collection's subject |
| `tests/system/test_fallbacks_only_know_their_input.py` | every fallback copies its input and writes nothing of its own |
| `tests/system/test_no_model_residency.py` | no model server is asked to keep a model loaded |
| `tests/system/test_instance_tails_stay_in_instances.py` | no instance tail is committed |
| `tests/system/test_operator_skills.py` | the skills name only routes, fields, settings and scripts that exist, agree across frameworks and keep their budgets |
| `tests/system/test_documentation_matches_routes.py` | every route is documented, documents name only routes that exist, and the entry documents name no instance's subject |
