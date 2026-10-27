---
type: "documentation"
title: "The documents in `docs/`"
description: "Two kinds of document live here, and they must not be confused:"
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "overview"
audience: "someone meeting Kovium for the first time"
doc_path: "docs/README.md"
doc_sha256: "6428e91542cd9df42bd5c197f97dd7e746b064556ee117a49863da2f009a697f"
---

# The documents in `docs/`

Two kinds of document live here, and they must not be confused:

- **Current contracts** say what the platform does now. They are kept true:
  `tests/system/test_documentation_matches_routes.py` checks that every route
  the app serves is named in one of them (or in `specifications.md`), that
  every route they tell a reader to call exists, and that the request fields
  and settings added since Track M are explained in them. A document listed
  under "Current contracts" below is part of that check.
- **Plans, reports and records** say what was intended, measured or decided
  at a date. They are history: true about their day, not necessarily about
  today. Nothing here is deleted, because the reasoning in them is how the
  platform got the way it is; when one disagrees with a current contract, the
  contract wins.

Start with [[specifications|`../specifications.md`]] (what an instance
does and how to run one), [[readme|`../README.md`]] (what the platform
is) and [[architecture|`../architecture.md`]]
(how the code is organised). Every change is logged, newest first, in
`../IMPLEMENTATION_JOURNEY.md`.

## Current contracts

| document | what it is the contract for |
|---|---|
| [[naming-v1|naming_v1.md]] | the name: Kovium, Kovium instance (km-instance), Kovium collection; the spelling in prose, identifiers, environment variables and markers; what keeps an old name and why (Track BG) |
| [[upgrading-to-kovium-v1|upgrading_to_kovium_v1.md]] | moving an install and its collections made before Kovium to the new names, by hand, step by step, for a separate agent (Track BG9) |
| [[api-spec-v1|api_spec_v1.md]] | the HTTP routes, grouped, with their request fields and refusals |
| [[next-steps-v1|next_steps_v1.md]] | every warning and refusal code with its next step as data: `notices`, a refusal's `error`, the actions, the registry and `GET /agent/knowledge/codes` |
| [[mcp-server-v1|mcp_server_v1.md]] | the MCP server over the governed routes: launch and fail-closed checks, the generated instructions, the read tools and their generated schemas, the surfaces and the workspace's ceiling (`config/mcp.json`), the write tools that only propose (`remember`, `forget`) and their grant, `request_tools` and the capabilities resource, the compact result view, what is not offered and why, scope from the connection, refusals and `fix.mcp`, audit, progress, standard output; wiring Claude Code, Codex, Cursor and OpenClaw, timeouts, the install outside a checkout, the harness behaviour observed and the token cost measured at the CLI |
| [[agent-memory-contract-v1|agent_memory_contract_v1.md]] | the governed `/agent/knowledge/*` routes: envelope, scope, access (`read_only`, and `propose` for proposals alone), proposals and the owner's decision on them, withdrawal and restore, confidence and warnings, `focus`, the one-call read, compact views, ideas and the owner's work in answers, consent to spend |
| [[note-format-contract-v1|note_format_contract_v1.md]] | the note: mandatory and per-type sections, frontmatter, section roles and identity fields an install declares, notes written by hand, relations, lint and repair |
| [[ingestion-spec-v1|ingestion_spec_v1.md]] | how a note becomes graph nodes, edges and provenance; tables, maths repair, books, long documents, URL intake, the extractive fallback |
| [[folder-intake-v1|folder_intake_v1.md]] | taking in a folder: triage, kinds per file, order, cost, budget |
| [[plain-language-rules-v1|plain_language_rules_v1.md]] | every text a person reads: the nine rules as applied, the option catalogue (`second_brain/catalogue/options_v1.json`, four lines and a "later" line for every option), the welcome's texts (`welcome_v1.json`), the collection tools' texts (`setup_v1.json`), where the words for each code live, the one checker (`second_brain/plain_words.py`) and the settings coverage test |
| [[welcome-v1|welcome_v1.md]] | the first five minutes: `GET /welcome`, `GET /help`, `POST /welcome/example`, `POST /welcome/tour`; the example collection (`sandboxes/hello_brain`, its manifest, how it is built and opened, the rebuilt-note guard), the tour and its sample added free or by the AI with an approval, help from the user guide and the catalogue or, once fetched, from the full manual (the help collection: its repository, the zero-token build `scripts/build_help_collection.py`, `config/help.json`, `POST /welcome/manual`, a copy of another version reported), the MCP server's welcome mode, the `documentation` kind and kind guessing |
| [[collections-v1|collections_v1.md]] | collections and who may do what: `POST /collections`, `/collections/open`, `/collections/connect`, `/collections/access`, `/collections/settings`, `GET /collections/setup`; making and opening a collection (from another computer too, with what did not travel), one connection per collection and its binding, the access policy per agent (`read`, `propose`, `manage`) in the collection's `config/mcp.json`, `change_setting` and the words an install adds, the check of what works, the MCP setup tools and their `manage` grant |
| [[obsidian-view-v1|obsidian_view_v1.md]] | the view in Obsidian (Track BE): the template `config/obsidian_view.json`, contributions in `config/obsidian_view.d/`, the owner's word, the housekeeping hidden from the registry's layout, colours by `type:` and the palette recorded in `config/obsidian_view_state.json`, merge never overwrite, nested vaults, when it runs (`create_collection`, the example, the nightly tasks, `scripts/obsidian_view.py`), the `obsidian_view` switch, what Obsidian does with it |
| [[access-tokens-v1|access_tokens_v1.md]] | access tokens (Track BB): a caller agent proves who it is; the token's format and its fingerprint kept beside the access policy, the `require_token` switch (`access_keys`), the gate on every route, `sign_in` over MCP and `Authorization: Bearer` over HTTP, `POST /collections/tokens`, `GET /collections/tokens`, `POST /collections/tokens/revoke`, `scripts/access_token.py`, the join's token, the owner's one-time web sign-in (`/ui/sign-in`, `/ui/sign-in/code`), the refusals and the honest limit |
| [[install-and-caller-agent-v1|install_and_caller_agent_v1.md]] | the one install step (`python scripts/setup.py`: the environment, the checks in words, each AI program connected in welcome mode, idempotent), the Claude Code plugin this checkout ships (`skills/claude-code/`, its own small folder: its skills, `/kovium:start`, its connection and `scripts/mcp_launch.py`, which finds the checkout through the record the step writes; what `claude plugin install` copies, measured), one connection per program, the caller agent added to a person's project (`POST /collections/agent`, `make_agent`) and the answer style kept per collection (`config/style.json`, its four choices, the four things no style drops) |
| [[adding-documents-v1|adding_documents_v1.md]] | adding files, folders and links in one request (`POST /sources/add`): the plan, its approval, the standing token budget (`/ops/budget`) and the one question about it, retries, the six-line report |
| [[document-conversion-v1|document_conversion_v1.md]] | the PDF converter as configuration, docling, figures read by vision on request, reading a source again, books with pages and a cost ceiling |
| [[images-v1|images_v1.md]] | images as sources, images inside Markdown, HTML and Word, equations written in text or MathML, the figure and equation routes without docling |
| [[equations-v1|equations_v1.md]] | equations as regions with ids, read on request by the local formula model or vision, `unread_regions` in answers |
| [[video-intake-v1|video_intake_v1.md]] | videos taken in from their link: metadata and the video's own caption track with times, chapters, refusals, many videos and playlists planned like a folder |
| [[source-tables-v1|source_tables_v1.md]] | a source's tables copied by code into `## Source tables`, each row a fact with its column names and caption, merged tables kept as text, the row cap |
| [[reference-lists-v1|reference_lists_v1.md]] | glossaries and lookup lists kept term by term: the `reference-list` kind, entries recognised by layout and copied by code into `## Source entries`, each a Definition found by its term |
| [[reading-fidelity-v1|reading_fidelity_v1.md]] | reading a source in full, values marked unverified, the fidelity pass, corrections |
| [[concept-notes-v1|concept_notes_v1.md]] | one maintained note per concept: names, hierarchy, merges, divergences, open problems, ideas, the owner's work, nightly curation |
| [[ideas-v1|ideas_v1.md]] | ideas as a note kind of their own, never evidence |
| [[owner-work-v1|owner_work_v1.md]] | the owner's own notes (`notes`, `design`, `commentary`), what they are about, never evidence |
| [[instance-tails-v1|instance_tails_v1.md]] | one shared head, an approved tail per instance |
| [[prompt-system-v1|prompt_system_v1.md]] | where prompts live, what changing one costs, extraction reuse |
| [[source-renames-v1|source_renames_v1.md]] | originals renamed, moved or deleted: the registry, reconcile, rollback |
| [[renames-v1|renames_v1.md]] | renaming anything (an original, a note, a concept, a converted copy) with every reference and key following: the inventory, the engine, `/rename`, rollback, the rename log, renames made by hand carried through nightly, the ASCII pass |
| [[source-roots-and-portable-provenance-v1|source_roots_and_portable_provenance_v1.md]] | named source roots and paths that survive a move |
| [[workstream-b-governance-and-routing-policy-v1|workstream_b_governance_and_routing_policy_v1.md]] | governance: read-only by default, scope, the audit trail |
| [[workstream-c-human-review-workflow-v1|workstream_c_human_review_workflow_v1.md]] | the review queue and correction workflow |
| [[workstream-d-operations-recovery-policy-v1|workstream_d_operations_recovery_policy_v1.md]] | backup, restore rehearsal, schema migrations, runbooks |
| [[agent-knowledge-contract-v1|agent_knowledge_contract_v1.md]] | a pointer: the public name of the agent contract |

## Guides

| document | for |
|---|---|
| ../GETTING_STARTED.md, with one page per AI program in [[claude-code|getting_started/]] ([[claude-code|Claude Code]], [[codex|Codex]], [[openclaw|OpenClaw]]) | a beginner, step by step: install once, connect the AI program, the welcome and the tour, a collection of one's own or joining one, readers and writers, the first document (Track BF; held to the code by `tests/contracts/test_getting_started_guide.py`) |
| [[user-guide-v1|user_guide_v1.md]] | a person, not an agent: what a collection is, adding documents, asking, what happens at night, what uses the subscription (a guide, not a contract) |
| [[first-instance-lifecycle-guide-v1|first_instance_lifecycle_guide_v1.md]] | someone setting up their first instance, from an empty folder to a restore rehearsal |
| [[test-me-first|TEST_ME_FIRST.md]], [[run-local|run_local.md]] | developers: starting the web page and the sample workspace over HTTP (a newcomer is sent to the getting started guide) |
| [[deployment-spec-v1|deployment_spec_v1.md]] | the local-only deployment scope |
| [[implementation-overview|implementation_overview.md]] | a map of the repository and of every subpackage, with the guard tests |

## Standing plans with open work

Read the relevant one before picking up an area; each says what is built and
what remains.

| document | open work |
|---|---|
| knowledge_layer_improvement_plan_v1.md | Tracks A to AT: built, their contracts listed above; Tracks AU to AW (answers that stay fast on a large collection, tests that pass every time, the small leftovers of the AR to AG build): planned, not started |
| first_run_experience_plan_v1.md | Track AG, the first five minutes: install, welcome, plain-language setup and adding, approval enforced (AG0 to AG7 built, the track is complete: the catalogue, the user guide and the style rules, the welcome, collections, adding, the install step, self-fixing defaults, the help collection, Codex and OpenClaw in their own formats) |
| mcp_server_plan_v1.md | Track AB, an MCP server over the governed routes: AB0 to AB5 built, the track complete (AB4: writes through review) |
| typed_relations_plan_v1.md | inferred relations (Phases 2 to 5) |
| retrieval_fact_recall_plan_v1.md, answer_token_efficiency_plan_v1.md | section-level semantic signal; the cost of escalation |
| book_ingestion_followup_v1.md | the index note's own type, front-matter bibliography, chapter-to-chapter links |
| followup_implementation_plan_v1.md | the maths repair's chemistry and non-English gaps |
| phase_5_concept_shortlist_v1.md | shortlisting concepts past the extraction hint's ceiling (designed, not started) |
| gbrain_reference_v1.md | a reference on an external memory system, and what to borrow next |
| brain_keeper_backends_migration_v1.md | the brain keeper's backends; its isolation statement and answer-cache contract are still in force |

## Plans, reports and records (history)

Roadmaps and plans: `production_grade_roadmap.md`,
`production_readiness_phase_1_implementation_plan_v1.md`,
`phase_1_phase_2_roadmap_v1.md`, `phase_1_execution_plan_v1.md`,
`phase_2_execution_plan_v1.md`, `phase_3_execution_plan_v1.md`,
`mvp_roadmap.md`, `mvp_completion_roadmap.md`, `mvp_finish_execution_v1.md`,
`formal_completion_plan_v1.md`, `final_phase_plan_v1.md`,
`priority_completion_plan_v1.md`, `coherent_next_phase_v1.md`,
`dependency_buildout_plan_v1.md`, `advanced_query_plan_v1.md`,
`runtime_ui_upgrade_plan_v1.md`, `persistence_plan_v1.md`,
`graph_index_and_long_text_plan_v1.md`, `living_wiki_discipline_implementation_plan_v1.md`,
`wiki_layer_merge_plan.md`, `wiki_maintenance_plan_v1.md`,
`note_type_integration_plan.md`, `okf_alignment_plan_v1.md`,
`facet_extensibility_plan_v1.md`, `facet_set_redesign_v1.md`,
`vocabulary_lifecycle_plan_v1.md`, `agent_conversations_plan_v1.md`,
`retrieval_ranking_upgrade_batch_1_plan.md`.

Tickets, results and status reports: `production_phase_1_ticket_*.md`,
`phase_2_*.md`, `phase_3_workstream_*_results_v1.md`,
`phase_1_real_note_eval*.md`, `MVP_READY_STATUS.md`,
`testing_ready_status_v1.md`, `DONE_STATE_VALIDATION.md`,
`live_normalization_validation_v1.md`, `brain_keeper_phase0_findings.md`,
`agent_knowledge_retrieval_repository_landscape_2026-08-09.md`.

Early design notes, superseded by the contracts above:
`ingestion_improvements_v1.md`, `gap_engine_improvements_v1.md`,
`gap_prompt_runtime_v1.md`, `hybrid_normalization_v1.md`,
`model_runtime_live_v1.md`, `normalization_runtime_answer_v1.md`,
`wiki_schema_template_v1.md`.

Decisions still in force, recorded at their date:
`persistence_decision_v1.md` (SQLite is the store),
`workstream_a_trust_limits_v1.md` (the stated trust limits),
`workstream_e_performance_and_storage_decision_v1.md` (the measured envelope).
