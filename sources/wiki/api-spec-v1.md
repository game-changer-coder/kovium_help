---
type: "documentation"
title: "API Specification V1"
description: "Provide stable interfaces for:"
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing Kovium"
doc_path: "docs/api_spec_v1.md"
doc_sha256: "ec4db36e39adf02a5365a7ee7cec222c1f94ad4dcdf3d3992efd940f99e68313"
---

# API Specification V1

## API Role
Provide stable interfaces for:
- the visual UI,
- caller agents, through the governed `/agent/knowledge/*` routes
  (`docs/agent_memory_contract_v1.md`),
- operators and maintenance scripts,
- future optional MCP wrapping.

The live route table, with every request field, is `/docs` on the running
service. This document groups the routes and states their contracts.
`tests/system/test_documentation_matches_routes.py` checks that every route the
app serves is named here or in another current contract (`docs/README.md`),
that every route documented here exists, and that the request fields added
since Track M are explained. Every request carries `agent_id` and
`workspace_id` (422 without the workspace, 404 for one not registered).

## API Style
- local HTTP JSON API
- loopback-only binding for V1
- private access
- **a request field the route does not take is never dropped in silence**
  (chores after Track AM; `api/next_steps_wiring.py`, checked on every JSON
  body against the route's request model, top-level fields only):
  - one that looks like a misspelling of a field the request did not send
    (the same letters in another spelling, `dry-run` for `dry_run`; a whole
    word of a longer field, `root` for `root_path`, `tokens` for
    `tokens_per_run`; or nearly the same word) is refused on any route: 422,
    `error.code: request_field_unknown`, `detail.fields: [{field,
    did_you_mean}]` and `detail.known_fields`; nothing is done, since the
    route's default would have acted in place of what the caller meant;
  - on a route whose model forbids extra fields, any unknown field is refused
    the same way: today `POST /ops/workspaces` (where a collection lives) and
    `POST /ops/budget` (what the user agreed to spend), where a 201 for
    `root` once registered a workspace under `sandboxes/`;
  - any other unknown field is accepted as before, so callers sending
    harmless extras keep working, and named in the response's `warnings` as
    `request_fields_ignored:<fields>` with its notice.

## Core Endpoint Groups

### 1. Health and metadata
- `GET /health`
- `GET /version`
- `GET /schema/summary`

### 2. Search
- `GET /search?q=`
- `GET /concept/search?q=`
- `GET /question/open`

### 3. Node and edge retrieval
- `GET /node/{id}`
- `GET /node/{id}/neighbors`
- `GET /edge/{id}`
- `GET /provenance/{entity_id}`

### 4. Subgraph and path queries
- `POST /query/subgraph`
- `POST /query/path`
- `POST /query/related`
- `POST /query/cross_chapter_bridges`

### 5. Gap queries
Categories and the record shape are in `schema/gap_analysis_spec.md`.
- `POST /query/gaps` `{categories?, min_sources?, min_similarity?,
  max_per_category?}` — every category, or the named ones, **ranked widest
  evidence first**: distinct sources (`metadata.sources`, set on every gap),
  then confidence, so a limitation several sources share outranks any one
  note's gap. Unvalidated methods are one gap per note here (`members` list
  them; the per-method listing is `/query/gaps/weak_method_validation`).
  `max_per_category` caps each category after ranking. `min_sources`
  (default 2) and `min_similarity` (default 0.25) apply to
  `limitation_cluster` only; out-of-range values are refused with 400
- `POST /query/gaps/unanswered_questions`
- `POST /query/gaps/weak_method_validation`
- `POST /query/gaps/thesis_bridges`
- `POST /query/gaps/limitation_clusters` `{min_sources?, min_similarity?}` —
  limitations several sources state, grouped, most widely shared first; each
  record lists its `members` (note, title, workspace-relative path, line,
  quote) and `metadata` (sources, cohesion, shared terms, concepts)
- `POST /query/gaps/topic` `{topic?, facets?, categories?, min_sources?,
  min_similarity?, semantic?, lexical?}` — the gaps under one topic or context.
  The `scope` is chosen by the workspace's own routes and returned with every
  note's `basis`: concepts the topic names as whole words (label or alias),
  most specific first, with everything narrower and never siblings; facet
  values it names, aliases included; nodes containing every word of the
  topic; and meaning, only when an embedding server answers. `scope_basis` says
  what each route contributed or why it did not. `facets` narrows the scope
  (alone, it is the scope). Detectors run inside it: a limitation cluster counts
  only in-scope sources. A topic reaching nothing returns an empty scope and
  the warning `topic_matched_nothing`, never the collection's gaps. No topic
  and no facets is a 400.
- `POST /gaps/fill` `{dry_run?=true, topics?, token_budget?}` — one pass of
  the proactive gap filler. For each topic configured in
  `config/gap_filler.json` (or given here): the scoped gaps, one brain-keeper
  interpretation, and one review-queue proposal (`gap-proposal:<id>`) per real
  gap, holding attributed evidence the collection may already have, cited works
  it does not hold (`read_next`), and the research direction. A dry run (the
  default) returns topics, gaps and `estimated_tokens` with no agent call and
  no write. A real run is refused with 409 until `enabled: true`, and stops at
  the token budget (`stopped: budget_reached`). `/review/action` on a
  `gap-proposal:` item: `resolve_work_item` writes an attributed idea note
  (`type: idea`, `origin: gap_filler`) under `sources/wiki/ideas/` and ingests
  it; `dismiss_work_item` is remembered, and that gap is never proposed again.
  A gap an idea already addresses is reported under the topic's
  `addressed_by_ideas` and not proposed (`docs/ideas_v1.md`)
- `POST /query/gaps/interpret` `{categories?, topic?, facets?, auto?,
  interpretation?, dry_run?, approval?}` — the caller-handoff payload (prompt from
  `prompts/prompt_interpret_gaps.md`, gaps, output schema), or with
  `auto: true` the brain keeper's interpretation. `categories` limits it, e.g.
  `["limitation_cluster"]` for research directions alone; with `auto: true`
  it spends, so `dry_run: true` gives its plan and `approval`, and a real run
  needs that approval where the collection asks for one (Track AW3,
  `docs/adding_documents_v1.md`); `topic`/`facets`
  interpret only that scope, and the topic frames the directions. Every
  interpretation carries `research_direction`, `gap_id`, `same_condition_as`,
  `rejected_members`, `theme_title` and `evidence_kept` (empty for other
  categories). The finalized result
  adds `themes` (limitation clusters the interpreter joined, with distinct
  sources and the strongest direction) and `separated` (rejected members, kept
  as single-source limitations). An interpretation posted back is finalized
  against the same gaps, so post back the `topic`/`facets`/`categories` it was
  asked with. When limitation clusters are interpreted, up to 15
  single-source limitations close to a shared cluster ride along as
  `Candidate (one source)` gaps (`metadata.candidate`), so the interpreter can
  join them to the condition they state. An unjoined candidate never becomes a
  theme. In `auto` mode the reply reports `billed_tokens` and
  `brain_keeper.model`/`usage`. With nothing to interpret, no agent is asked
  (`no_gaps_to_interpret`)

### 6. UI-oriented graph view queries
- `POST /view/graph/focus` — the subgraph around a node (`focus_node`, `depth`), with centrality and clusters, for the graph view
- `POST /view/canvas` — the whole graph as a canvas (nodes, edges, clusters), optionally written to a file (`write_to_file`)
- `GET /view/sidebar/details/{node_id}` — a node's details for the side panel
- `GET /view/note_map?path=` — one note's map (Track AK4): `path` is the note
  (`sources/wiki/<name>.md`, workspace-relative) or its map
  (`sources/wiki/maps/<name>.canvas`). Returns `note`, `map_path`, `canvas`
  (the JSON Canvas: `nodes`, `edges`), `node_count`, `edge_count`. The map is
  found through the maps index (`.note_maps.json`, which note drew each
  canvas), else by the note's file name. Read-only and audited like
  `/wiki/page`; 403 for a path outside the workspace or not a note or map,
  404 when the note has no map (maps are drawn when a note is taken in, for a
  note whose sections a map shows)

### 6a. Source intake
See `docs/ingestion_spec_v1.md` and `docs/folder_intake_v1.md`.

**Videos** (Track W, `docs/video_intake_v1.md`). `POST /sources/process` with a
YouTube link takes in the video (kind `youtube-video` unless named): its
metadata and own caption track, the raw source its timed transcript. Request
`description_only` (null follows `config/video.json`). The result adds `video`
(`video_id`, `title`, `channel`, `published`, `duration`, `transcript_source`,
`language`, `transcript_chars`, `truncated`, `chapters`). Refused with 422
`{reason, message}`: `no_transcript`, `video_not_public`,
`video_page_unreachable`, `video_player_unreachable`.
- `POST /sources/process_videos` `{urls, kind?, template_id?, dry_run: true,
  token_budget?, description_only?, auto_ingest, concepts, wait}` — video and
  playlist links planned like a folder. A dry run returns `plan.videos` (one
  row per video: `video_id`, `url`, `title`, `channel`, `duration`,
  `language`, `transcript_source`, `transcript_chars`, `estimated_tokens`,
  `action`: `process`, `duplicate`, `already_ingested`, `no_transcript`,
  `video_not_public`, ..., `reason`), `plan.estimated_tokens`,
  `plan.estimate_basis`, `plan.counts`, `plan.over_limit`, `refused_links`,
  and with a budget `would_fit_budget`. A real run (audited
  `process_videos`) adds `run` as a folder run does (`processed`,
  `stopped_at_budget`, `concept_pass`, `billed_tokens`); backgrounded unless
  `wait`.

**Paths in intake results** (Track Z). `/sources/process`, `/sources/process_book`,
`/sources/process_folder`, `/sources/reread` and `/normalize/file` return every
path relative to the workspace (`sources/wiki/a.md`), in forward slashes on
every system, never an absolute path naming the machine: `normalized_path`,
`templated_source_path`, `converted_markdown_path`, `raw_source_path`, a
book's `chapter_dir` and `index_path`, and paths inside messages. A path
outside the workspace is `root:relative` under a declared source root, else
relative to the checkout, else its name. `/ingest/file` and `/normalize/file`
take a workspace-relative path (or `root:relative`, or an absolute one), so
what one route returns another accepts; a path that names no file is 404.
The governance audit, the normalization log, the review history and every
other log a workspace writes spell paths the same way.
- `POST /sources/process` — one source (a file, a source-root reference, or a
  URL) to a note. Track S: an unchanged source is not extracted twice. The
  result carries `reused_extraction` (true when every part of the extraction
  came from the brain keeper's answer cache) and `extraction_reuse`: `why`,
  `billed_tokens_saved` (what the call that first gave the answer billed;
  null when unknown, 0 when nothing was reused), `cached_from` (that job),
  `answered_by`, for a document read in parts `parts` and `parts_reused`, and
  `names_mapped` (`{from, to, why}`: concept names of the reused answer
  brought to the concept layer's current wording, `why` one of `preferred
  name`, `merge`, `alias`). `refresh: true` reads the source again
  (extraction, shaping, normalization) with the hints the collection has now,
  at full cost, and stores the new answers. A paper split into sections passes
  `refresh` on to them; each section's result carries its own reuse. A brain
  keeper job record carries `answer_cache` (`status`: `reused`, `miss`,
  `refreshed` or `off`, with `reason`, `key_leaves_out`, and for a reuse
  `source` (`cache` or `earlier_key_format`), `cached_from`, `answered_by`,
  `cached_at`, `billed_tokens_saved`).
  Track X (`docs/concept_notes_v1.md`, "What upkeep costs, and when it is
  paid"): `concepts: sync` (default) or `defer`. The result's `cost` is every
  call the run caused: `cost.billed_tokens` is the sum the call log records
  for `cost.run_id`, split into `cost.extraction` and `cost.concept_upkeep`
  (`agent_calls`, `billed_tokens`, `chars_sent`, `deferred`). With `defer`, or
  a piece of a batch, `concepts` and `cost.concept_upkeep` say `deferred: true`
  and `paid_by` (the batch's pass, or `POST /concepts/sync` and the nightly
  runner), and the source is pending with `deferred: true` in `GET /concepts`.
  A split paper's overview and sections make one concept pass, in the
  overview's `concepts` and `cost`
- `POST /sources/process_book` — a book split into chapter notes; accepts a PDF
  or any document format, converting it first. Track O (see
  `docs/document_conversion_v1.md`, "Books"): `dry_run: true` returns the
  book's `pages` (`pages_basis`), the converter it would get and `estimate`
  (`seconds`, `seconds_per_page`, `basis`: this workspace's measured rate or
  a stated default), converting nothing; a book docling would read with more
  pages than `docling_book_page_ceiling` is refused at once (`ok: false`,
  `reason: over_docling_page_ceiling`, the estimate in `detail` and `plan`,
  warning `book_over_docling_page_ceiling`) unless `force_long_conversion:
  true`, except when a current converted copy of the same bytes is read
  instead (the plan's `converted_copy`: `path`, `basis`, `converter`, `line`;
  the run's `conversion.method: reused_converted_copy`, nothing converted);
  a book whose pages cannot be counted is refused the same way
  (`reason: docling_book_pages_unknown`, warning
  `book_pages_unknown_for_docling_ceiling`). `estimate.covers` says the
  seconds are the converter's alone; `estimate.whole_run` is the measured
  whole run (split, chapters, ingest), None until one has run here. A dry run
  also has `tokens` (Track AM2): `chapters` (each with `estimated_tokens`
  `{low, expected, high}`, its `extraction` and `concept_upkeep`), `total`,
  `extraction`, `concept_upkeep`, `split_agent`, `basis`, `group`,
  `chapters_basis`, `planning_figure` and `line`, from this install's own
  chapter history (`docs/ingestion_spec_v1.md`). `path`
  may be workspace-relative (`sources/raw/book.pdf`), as for
  `/sources/process`. Each chapter note records `book_source`, `source_pages`
  (`41-78`, or `unknown` for a book without pages) and
  `book_converted_markdown`; a chapter opened by a bare `Chapter N` line is
  named by the line after it. The result adds `pages` (`known`, `count`,
  `basis` or `reason`), `conversion_estimate`, `duration_seconds`, and
  `figures` (each chapter's figure numbers, `unassigned` with
  `unassigned_detail` saying why, and what `vision.books` read). Track X: the
  chapters' concept upkeep is one pass after the last chapter, reported as
  `concept_pass` (each chapter's `concepts` says `deferred` and who pays);
  `concepts: defer` leaves that pass for `POST /concepts/sync`; the dry run's
  `concept_upkeep` states `tokens_per_chapter`, its `basis` and `paid_by`.
  Track AL (`docs/ingestion_spec_v1.md`, "The chapter-finding ladder"): the
  parts are found by the ladder (owner, outline, contents, headings, agent);
  `chapters: [{title, pages}]` is the owner's own split, sent on the user's
  word, which skips the ladder and passes the same checks (`reason:
  book_owner_split_invalid` or `book_owner_split_refused`, with `detail`,
  when it cannot be used). The result's `split` is `{step, confident, line,
  tried, agent, parts, left_out, forced, basis?, whole_because?}` (`left_out`:
  the pages before the first part, in no part, with why; `basis: text_layer`
  when the PDF's own text layer found the parts an older converted copy did
  not), and `cost.billed_tokens` what
  finding the parts cost (the agent step runs within the standing budget). The
  dry run adds `split`: `{known, basis, step, checked, confident, parts:
  [{position, title, pages, name}], tried, agent: {would_ask, allowed, reason,
  estimate_tokens}, line, refused, left_out}`, never asking the agent, reading a PDF's
  text layer with the basic extractor when no converted copy exists, and
  after a copy converted before that gives no confident split; a
  `chapters` list that fails a check needing no text is `ok: false` there
  (warning `book_owner_split_refused`, `split.detail`). A refused run's
  `split.next` (`settled_by: caller`) is the recommended call. Each part is
  named `<document-key>-<NN>-<title>` on every layer, titled as the part, and
  records `part_number`
- `POST /sources/book_parts/attach` `{book, parts: [{note, pages}], book_title?, dry_run: true,
  access_mode}` — chapter notes split by hand recorded as parts of their book
  (Track AL6): `book_source`, `source_pages`, `part_number` and, where missing,
  `book_title` on each note and its draft, in the order given; the note read
  into the graph again. A real run needs `maintenance_write`. Returns `parts`
  (with each one's `fields` and the `name` it would get), `skipped` (owner-
  maintained, or part of another book), `written`, `problems`, `summary`,
  `rename_next`. Refusals: `book_parts_none_named`, `book_part_not_found`
  (404), `book_parts_repeated`, `book_parts_bad_pages`,
  `book_parts_out_of_order` (422)
- `POST /sources/process_folder` `{path, kind, dry_run, token_budget, include_near_duplicates, …}` — a
  folder triaged (duplicates, versions of a known work, scans without text, long
  documents, slides) and processed in date order under a token budget; a dry run
  by default, returning the plan and its estimated cost. Track AJ: a version is
  held back (`near_duplicate`, with `version_of`: `of`, `note` or `file`,
  `evidence`) unless `include_near_duplicates`, which takes it in and links its
  note to the other version's with `version_of` (the run's outcome carries
  `version_of` with `linked`); a near miss is taken in with `similar_to`
  (`why_not`). See `docs/folder_intake_v1.md` ("Versions of one work"). Track X: every
  document's concept upkeep is deferred and paid by one concept pass after the
  last (`run.concept_pass`, its cost in `run.billed_tokens`); each plan row's
  `estimated_tokens` includes `estimated_concept_tokens`, and `plan.concept_upkeep`
  says the rate per document, its `basis` (measured on this install from the
  call log once it has history, else a stated default) and `paid_by`; each
  row's `estimated_range` is `{low, expected, high, basis, group, concept}` (Track AM1:
  by note type and length band from this install's call log, which records each
  `process_source` run's `template_id`); the
  budget keeps each processed document's concept share in reserve;
  `concepts: defer` leaves the pass for `POST /concepts/sync`. Each file gets its
  own kind (Track U4, `kind_per_file: true` by default): from its frontmatter
  (`type`/`kind`/`template`), its file type (`kind_by_suffix` in
  `config/intake.json`, over the shipped rule mapping image files to the
  `image` kind),
  or a cheap look at its headings, frontmatter keys and identifiers against
  each kind's declared sections and fields, the folder's `kind` being the
  default, or the file's shape (Track AE: a file mostly made of entries, a
  glossary or a lookup list, is given the kind declaring `file_shape:
  entries`, `reference-list`); every plan row says `kind`, `kind_basis`
  (`frontmatter`, `file_type`, `content`, `shape`, `folder_default`) and
  `kind_evidence`, and the plan adds `kinds` (files to run per kind); a row
  whose kind is built without an agent (`image`, `reference-list`) has
  `estimated_tokens: 0`
- `POST /sources/add` `{paths, urls, kind?, dry_run, approval?, token_budget?, versions?, wait?}` —
  Track AR: files, folders (at any depth) and links added in one request. A
  dry run (the default) is the plan: `summary` in plain words, `approval`,
  `budget`, `estimated_tokens`, `default_kind`, the rows, and a `question`
  (`asks: budget`, with `on_yes` and `on_no` calls) only when the standing
  budget does not cover it. The cost is a range (Track AM1): `estimated_tokens`
  is the planning (expected) figure, `estimated_range` `{low, expected, high}`,
  `estimate_groups`, `may_pass_limit` and `also_possible` say the rest, and each
  row has `estimated_range` with its `basis` and `group`. A real run needs the plan's `approval` (422
  `adding_approval_required`; 409 `adding_plan_changed`,
  `adding_approval_expired`) and runs as a job: triage, kinds, books (read
  whole when their chapters are unclear), retries, one concept pass, failed
  vectors again, and `report.lines` (at most six plain lines). Track AJ:
  `versions` (`link` by default, `separate`, `skip`; anything else
  `request_refused`): a version of a known work is taken in and linked with
  `version_of`; rows carry `version_of` or `similar_to`, the result `versions`
  and each outcome's `version_of` (`linked`), warnings `adding_versions_linked`,
  `adding_version_not_linked`. Track AG3: the plan adds `say` (plain lines),
  `needs` (what the files need, read from their structure), `asks` (the
  questions in order, each with its calls), `start`, `kinds` (with
  `adding_kind_unknown`) and `agent_limit`; the request adds `keep_as_limit`
  (the yes also keeps the budget as the standing one) and `access_mode:
  manage` (the adding tools over MCP: a manage entry that may `add`); a run
  above the agent's cap is refused `adding_over_agent_limit` (403); a started
  run carries `say` and `next` (its status). See `docs/adding_documents_v1.md`
- `GET /sources/add/status` `{job_id?, wait?}` — Track AG3: how an addition is
  going (`state`: `running` with `progress` and `next`, `done` with the
  `report` and its lines in `say` word for word, `failed`, `none`); without
  `job_id`, the latest addition of the collection; 404 `adding_job_not_found`
- The approval on every spending route (Track AG3): `POST /sources/process`
  (`dry_run`, new), `/sources/process_book`, `/sources/process_folder`,
  `/sources/process_videos`, `/sources/figures`, `/sources/equations` and
  `/sources/reread` give `approval` with their dry run and take it back as
  `approval`; where the collection's `approval_required` (`config/intake.json`)
  is on, a real run without a matching one is refused
  `spending_approval_required` (422), `spending_approval_mismatch` or
  `spending_approval_expired` (409)
- `GET /ops/budget` and `POST /ops/budget` `{tokens_per_run}` — the
  collection's standing token budget per addition (`config/budget.json`): the
  user's one answer, relayed by the caller agent; null when unset
- `POST /ops/embeddings/reindex` `{path? | missing: true | all: true, limit?,
  max_seconds?}` — Track AK2: index the passage vectors of one note (`path`,
  workspace-relative), of every note missing its vectors (`missing`: notes
  with a passage long enough to embed and no vector for the configured model
  in the current format), or of every note (`all`, which then drops the
  vectors of notes no longer on disk). Exactly one of the three (422
  otherwise); a `path` outside the workspace is 403. The HTTP twin of
  `scripts/rebuild_vectors.py`, the same code (`passage_index.reindex`).
  Free: the install's local embedding server, no token; unchanged passages are
  reused, never embedded again; writes only vectors, never a note; audited.
  `limit` (notes) and `max_seconds` bound one call. Returns `mode`, `notes`
  (to index), `indexed_notes`, `embedded`, `reused`, `split`, `no_passages`,
  `failed` (each `note`, `reason`), `left` (not reached), `code` (`null`;
  `disabled`, `no_model`, `no_profile`, `endpoint_not_local`,
  `not_configured`: nothing done; `server_unreachable`: the server stopped
  answering and the rest wait; `some_failed`), `reason`, `pruned` (with
  `all`) and `summary`, one plain line. The nightly run calls it with
  `missing: true` (AK3); the codes `semantic_search_not_indexed` and
  `semantic_search_stale_format` name it as their next call
- `POST /ops/embeddings/rebuild` `{max_seconds?, wait?}` — Track AG5 (D7): build
  every note's passage vectors again **in a copy** and swap them in at once.
  Every passage is embedded afresh (nothing is reused) into a scratch database
  outside the collection, so a folder synced between computers never carries a
  partial index; only a finished build replaces the configured model's vectors,
  in one transaction, and one that fails, loses the server or exceeds
  `max_seconds` changes nothing. Free (the install's local embedding server);
  audited; backgrounded unless `wait` (the job, with `processing_in_background`).
  Returns `swapped`, `replaced`, `notes`, `built_notes`, `embedded`, `code`
  (`null` when swapped; else `server_unreachable`, `some_failed`,
  `out_of_time`, `disabled`, `not_configured`), `left`, `summary` (one plain
  line), and records which search program version and model id built the
  vectors (`maintenance/versions.json`)
- `GET /ops/status` also returns `first_run` (Track AG5, S13): `off` and `say`
  (every feature that is off or missing, in words, with its cost where it uses
  tokens) and `versions` and `versions_say` (what differs from the versions that
  wrote the collection)
- `GET /notes/native` — the workspace's notes written by hand (Track U2), each
  with `concepts_basis` (`stated`, `agent`, `extractive`) and its concepts
- `POST /notes/native/extract` `{dry_run: true, access_mode, notes?, refresh?}` —
  read the native notes that state no concepts with the brain keeper
  (`normalize_note`, the native-note extraction, cached by the note's bytes).
  A dry run lists `to_read` (with each note's current `extractive_concepts`),
  `already_read` and `estimated_tokens`; a real run needs `maintenance_write`
  (403 otherwise), keeps the concepts in `maintenance/native_notes.json`,
  never in the note, re-ingests and syncs concepts; with no agent it calls
  nothing (`agent_available: false`)

### 6b. Sources renamed, moved or deleted
See `docs/source_renames_v1.md`. No route here calls a model.
- `POST /sources/reconcile` `{dry_run: true, access_mode}` — match every
  recorded source to the files under `sources/raw/` and the declared source
  roots by their bytes, hashing only files whose stamp is new. Exact matches
  (rename, move, case-only rename) are repaired under `rename_repair: auto`, or
  proposed as `source-rename:<id>` work items; probable matches (same
  identifier or title, different bytes) are always proposed; a source with no
  match marks its notes `source_status: missing` and opens a
  `source-missing:<id>` work item. A dry run by default; a run that writes
  needs `access_mode: maintenance_write` (403 otherwise). Run nightly by
  `scripts/nightly_maintenance.py`
- `GET /sources/registry` — the source registry: each source by content hash,
  its current path, earlier paths with when they moved, and its notes
- `POST /sources/reconcile/rollback` `{snapshot_id}` — undo one source action
  (repair, keep, removal) from its snapshot
- `POST /review/action` on a `source-rename:` item: `resolve_work_item` repairs
  the notes, `dismiss_work_item` rejects that file. On a `source-missing:` item:
  `resolve_work_item` with `decision: keep` (or `dismiss_work_item`) marks the
  notes `retained_without_source`; `decision: remove` removes exactly that
  source's knowledge after an `/ops/backup` (409 without a decision)
- Lazily, before failing on a recorded source that is gone: `GET /review/source`
  (adds `located`), `POST /sources/process`, `GET /ops/sources` (status `moved`)
- `GET /review/source` on an original that is not text serves its converted or
  templated text (`served_from`, `source_path`), or answers 415
  `{reason: source_not_text}`; it never decodes binary

### 6b2. Renaming anything (Track AP)
See `docs/renames_v1.md`. No route here calls a model or the embedding server;
every one reports `billed_tokens: 0`, `agent_calls: 0`, `embedding_calls: 0`.
- `POST /rename` `{kind, target, new_name, dry_run: true, access_mode,
  overwrite: false}` — rename an `original`, a `note` (with its draft; the
  draft keeps its `__templated` ending, the note's name does not carry it,
  Track AZ2), a `concept` (a redirect stub
  left at the old key) or a `converted` copy, and move everything that records
  the old name in one step: links in other notes, node and edge ids, passage
  vectors (never re-embedded), access history, the change log (one `renamed`
  entry), the concept layer's state, records, the answer cache. A dry run by
  default (`renames`, `counts`, `store_rows`, `records`, `unfinished`,
  `summary`); a real run needs `access_mode: maintenance_write` (403
  otherwise) and adds `snapshot_id`, `keys_moved`, `reingested`, `maps`,
  `recovered`. 404 `rename_not_found`; 409 `rename_name_taken`
  (`detail.free_name`); 409 `rename_would_overwrite_original` (the user's
  question; `overwrite: true` sets the other file aside in the snapshot); 422
  `rename_name_invalid`; 400 `rename_original_not_recorded` (no note records
  that file and it is outside `sources/raw/` and every root); 409
  `rename_draft_claimed_twice`; 500 `rename_failed_undone` (rolled back). An
  original is any file a note records anywhere in the workspace, its notes
  followed under every spelling. Warnings
  `rename_links_left_in_owner_notes`, `rename_recovered_unfinished`
- `POST /rename/rollback` `{snapshot_id, force: false, access_mode}` — undo one
  rename or one pass from its snapshot; needs `maintenance_write`. 409
  `rename_changed_since` (a note edited since; `detail.later_renames` names the
  later renames to undo first; `force: true` undoes anyway),
  409 `rename_already_rolled_back`, 404 `rename_snapshot_not_found`. Its
  `graph` says what was cleaned from the graph so no node or edge refers to a
  name the rollback took away (`stale_removed`, `copies_removed`,
  `dangling_edges_removed`), which notes on disk the rename added to it
  (`added_for_notes_on_disk`, kept), which notes it read again from their
  files because their graph copy was behind (`refreshed_from_disk`, kept: the
  graph then matches the disk) and `line`, those three in plain words
- `GET /rename/log` `{kind?, name?, limit?}` — the rename log, newest first:
  `{at, kind, old, new, how, by, snapshot_id, old_id, new_id}`; read-only,
  audited
- `POST /rename/reconcile` `{dry_run: true, access_mode}` — renames made by hand
  (a note or concept note missing where the store knows it while a file no
  note owns has its bytes or its source's hash; a draft, a converted copy or a
  map renamed alone): exact ones carried through under `rename_repair: auto`,
  probable ones (the same title) proposed as `name-rename:<id>` work items.
  Run nightly by `scripts/nightly_maintenance.py`
- `POST /rename/ascii` `{dry_run: true, access_mode, limit?}` — existing names
  brought to the ASCII rule of Track AK1 in one pass and one snapshot: notes
  with their drafts, converted copies, a book's chapter files, concept notes
  and their graph nodes; two names that meet get a suffix; the owner's notes
  are never renamed (`skipped`)
- `POST /rename/parts` `{book?, dry_run: true, access_mode}` — a document's
  existing parts brought to `<document-key>-<NN>-<title>` (Track AL8), one
  book or every book with parts, in one pass and one snapshot: each part's
  note and draft, and its part file, converted copy and index file when they
  are the platform's own (`notes`, `part_files`, `converted`, `index_files`,
  `skipped`); the owner's originals and notes are never renamed. Run again,
  it renames nothing
- `POST /review/action` on a `name-rename:` item: `resolve_work_item` carries the
  rename through (`rename_decision`), `dismiss_work_item` rejects that file for
  that name

### 6b3. Note health (Track AO)
See `docs/note_format_contract_v1.md`, "What does not apply, what may be empty,
what is cited". No call to a model; read-only.
- `GET /maintenance/lint` `{agent_id, workspace_id, severity?: all|error|warning,
  kind?, note?, detail?, limit?: 50}` — the contract's findings across the
  workspace. Counts always cover everything found (`error_count`,
  `warning_count`, `by_kind`, `worst_notes`, `notes_with_problems`). Findings
  come in `problems` (errors first) when `detail` is true, or `kind`/`note` is
  given; `severity` alone does not list. `listed`, `not_listed` and
  `detail_note` say how many were listed and how to ask for the rest; `limit`
  is held between 1 and 1000. `cited_not_held`: `{works, links, most_cited,
  list?}`, works notes cite that the collection does not hold, once per work,
  never problems (`kind=cited_not_held` lists them). `not_applicable`:
  `{facets, sections}`, what notes say does not apply to them, counted as
  reviewed. `broken_characters`: `{notes, characters, sections_left_out}`
  (Track AT2), the characters a converter lost, counted per note as a
  `broken-characters` warning, and the `## sections` kept out of the text
  embedded for search by meaning for them (`docs/document_conversion_v1.md`,
  "Broken characters"). **The counts and `ok` always describe the whole collection**,
  whatever the filters say; `matching` is the filtered view's own size
  (`{filtered, filters, problems, errors, warnings, cited_works}`), and
  `detail_note` says plainly when a filter is applied ("N error(s) and M
  warning(s) in all; this view is only what matches severity=warning"). A
  request naming no workspace is refused (422 `workspace_not_named`, 404
  `workspace_not_registered` for an unknown one) like every scoped route;
  warning `lint_found_no_notes` is for a workspace that is named and holds no
  note, whose empty answer is not a clean bill of health;
  `no_vocabulary_registered` when facet checks were skipped. Not offered over
  MCP (`narrower_read`).

### 6c. Reading fidelity (Track L)
See `docs/reading_fidelity_v1.md`. No route here calls a model, except a real `POST /maintenance/fidelity` pass, which reads statement-level findings against their source within the standing token budget (Track AS4).
- `GET /review/source` `{path, line_start, line_end}` — with `line_end`, that
  range of the source's text form, up to a per-call cap
  (`source_page_max_lines`, `source_page_max_chars` in
  `config/reading_fidelity.json`), with `total_lines`, `next_line_start` (null
  at the end), `truncated`, `requested`, `max_lines_per_call`. A range ending
  before it starts is 422. With `line_start` alone, the old few-line window,
  plus `total_lines` and `next_line_start`
- `POST /maintenance/fidelity` `{dry_run: true, access_mode, notes?, judge: true}` — check
  every source note against its source's text: values found only in
  layout-lost text, table values the source prints in another row, totals that
  do not add up, units that do not fit their quantity, and statements that call
  two figures inconsistent when the arithmetic says they agree. A dry run by
  default; a run that writes needs `access_mode: maintenance_write` (403
  otherwise) and writes one `fidelity:<note>:<hash>` work item per note and
  `maintenance/fidelity/report.json`, never a note. Every finding carries
  `plain` (one sentence a person reads). In a real run the brain keeper first
  reads each statement-level finding against the cited lines of its source,
  within the standing budget (`GET`/`POST /ops/budget`; none set: nothing is
  spent and the finding goes to review unjudged): the response gains
  `judging` (`state`, `candidates`, `real`, `misstated`, `false`, `unjudged`,
  `unjudged_why`, `agent_calls`, `billed_tokens`, `line`; a failed call is
  counted in the last three), `judged_false` (the false alarms kept
  out of the queue, each with its quote and reason) and a `cost` that is what
  the reading spent; each finding gains `judgement`, and the work item `judging`
  and `summary`. `judge: false` keeps the pass free. A dry run spends nothing
  (`judging.state: dry_run`, `enabled` and `budget_for_judging` as a real run
  reports them, `would_read` the findings it would send as an upper bound,
  `remembered` and `no_source` those it would not, `why_not`)
- `POST /maintenance/fallback_notes` `{dry_run: true, access_mode, notes?,
  include_current: false}` — find notes a deterministic fallback wrote: lines
  matching the fingerprints of the sentences the pre-Track-P fallbacks wrote
  (`legacy_fixed_content`), notes whose provenance says the old fallback built
  them (`legacy_fallback`), and with `include_current` notes the current
  extractive fallback built (`extractive_fallback`). No model call. A dry run
  by default; a run that writes needs `access_mode: maintenance_write` (403
  otherwise) and writes one `fallback-note:<note>` work item per note, never a
  note (Track P4, `docs/ingestion_spec_v1.md`)
- `POST /maintenance/corrections` `{note, statement, proposed, section?, line?,
  source?: {path, line_start, line_end, text}, reason?, access_mode}` — file a
  correction to one statement. A proposal, allowed `read_only`: writes one
  `correction:<note>:<hash>` work item, never the note. 422 with a `reason` when
  the statement is not in the note as written, or the source text is not in the
  source within the lines given; the same correction filed again is the same
  item
- `POST /review/action` on a `correction:` item: `resolve_work_item` patches
  exactly that statement, with an attributed `## Provenance` line, a rollback
  snapshot and a re-ingest (409 when the note changed since, or is
  `maintained_by: owner`); `dismiss_work_item` is remembered. On a `fidelity:`
  item: resolving acknowledges, dismissing is remembered per finding
- `POST /maintenance/corrections/rollback` `{snapshot_id, access_mode:
  maintenance_write}` — put a corrected note back as it was, and re-ingest it
- `GET /ops/sources` — `verified` (kept for compatibility) means the bytes were
  re-hashed in this call; `bytes_rehashed`, `checked` and `verification_note`
  say exactly what was checked (Track L5)

### 6d. Documents with equations, tables and figures (Track M)
See `docs/document_conversion_v1.md`. Conversion is local and free; only
vision (reading a figure's image) calls the brain keeper, and only when asked.
- `POST /sources/process` — the PDF converter is the workspace's
  `config/conversion.json` `pdf_converter` (`docling`, `markitdown`, or null
  for the basic extractor); `pdf_converter` and `formula_enrichment` in the
  request override it for that source only. A backgrounded job that docling
  will convert runs in the long lane. `conversion.converter` names the converter that
  ran, its version, options, capability profile and, when it is not the one
  selected, `fallback_reason`; `conversion.metrics` counts what the converted
  copy recovered (`latex_equations`, `formulas_not_decoded`,
  `equations_not_recovered_pages`, `markdown_tables`, `table_rows`, `figures`,
  `figures_interpreted`); docling's block adds `pages` and `seconds_per_page`. A `pdf_converter` outside the list is a
  400 naming the list. With docling, `figures` reports the figure items
  written to the note and, when `config/ingestion.json` has `vision.enabled`,
  the figures read within `max_figures_per_source` and `token_budget`, with
  `billed_tokens`
- `POST /sources/figures` `{note | source, figures?, reinterpret, dry_run:
  true, access_mode, token_budget?, max_figures?}` — read one source's figures
  with vision. A dry run (the default, allowed `read_only`) lists each figure
  with `status` (`would_read`, `already_interpreted`, `no_image`,
  `over_figure_limit`, `over_token_budget`), `estimated_tokens`, and
  `vision_available` with `vision.reason`; it calls nothing. A real run needs
  `maintenance_write` (403 otherwise), calls the brain keeper once per figure
  within the budget, writes each reading under its figure in the converted
  copy and as a figure item in the note's `## Figures` (never in a
  `maintained_by: owner` note), re-ingests, and is audited
  (`source_figures_plan`, `source_figures_interpret`). Figures are planned
  referenced-first, then captioned; uncaptioned ones are `uncaptioned_skipped`
  unless `vision.interpret_uncaptioned`. `estimate_basis` says whether the
  estimate is this install's measured median or the formula. A real run
  returns each figure's final `status` (`interpreted`, `reused`, `failed`,
  `skipped_budget`, …) with its `interpretation` and `billed_tokens`, and
  `stopped` (`budget_reached`) with `remaining`. Without a backend that
  can be given an image, `outcome: vision_unavailable` and the warning
  `vision_unavailable`. **A book** (Track O2): `source` may name the book's
  original and `note` one of its chapter notes; `chapter` (a number or a
  chapter note's name) and `pages` (`41-78`, `41-45,60`) narrow it, and a
  chapter note alone selects its own chapter. The plan holds only the
  selection's figures (`outside_selection` counts the rest), the budget
  defaults to `vision.books` (`max_figures_per_book`, `token_budget_per_book`),
  each chapter note lists only its own figures, and `book` reports the
  chapters, their figures and the selection. An unknown chapter is a 404
  naming the chapters; a malformed page selection, pages of a book whose
  pages are not known, or `chapter`/`pages` on something that is not a book
  is a 422
- `POST /sources/equations` (Track N, `docs/equations_v1.md`) takes a book the
  same way (Track O3): `source` the book's original or `note` a chapter note,
  `chapter` and `pages` narrowing it to region numbers; each chapter note
  lists only its own regions; `book` reports each chapter's equations and the
  selection
- `POST /sources/reread` `{note | source, pdf_converter?, formula_enrichment?,
  figures?, vision, reextract: true, dry_run: true, access_mode, token_budget?,
  wait}` — read one source again with a chosen converter. A dry run converts
  into a scratch folder and returns `before`, `preview`, `comparison` (each
  metric's `before`, `after`, `change`) and `estimate` (`reextract_tokens` at
  this install's measured rate, `vision_tokens`, `total_tokens`); nothing is
  written. A real run (`maintenance_write`; backgrounded unless `wait`) keeps a
  rollback snapshot, re-processes the source (`reextract: false` re-converts
  only) and returns `after`, `comparison`, `fidelity` (Track L findings before
  and after), `billed_tokens` and `snapshot_id`. 422 `{reason:
  converter_not_supported, supported}`; 409 when the original is gone or the
  note is owner-maintained and `reextract` is true A source already in the workspace is
  converted into the copy its notes record, whatever the original is called
  now; `repointed` and `orphaned_copies_removed` report notes re-pointed and
  stale copies removed; `after.converted_markdown`, `after.converter` and
  `after.note_extracted_from` say what was measured. The estimate includes
  `concept_sync_tokens` and a calibrated `vision_tokens`; a real run returns
  `billed` (`extraction`, `vision`, `concept_sync`, `other`, `by_operation`)
  and `review_items` (corrections superseded with `covered`, fidelity items
  closed). A real run with `reextract: true` reads the source fresh
  (`refresh`, Track S) even when the new copy's text matches the old: the
  dry run quotes that re-extraction, and `reextract: false` is the free path
  A re-read also writes the new copy's tables into the notes' `## Source
  tables` and re-ingests them: `tables` as on `/sources/process`, with
  `reingested` (`docs/source_tables_v1.md`); and its entries into `## Source
  entries`: `entries` as on `/sources/process`, with `reingested`
  (`docs/reference_lists_v1.md`).
- `POST /sources/reread/rollback` `{snapshot_id}` — put the converted copy,
  figures and notes back as they were, remove figures the re-read added, and
  re-ingest
- `POST /sources/equations` `{note | source, equation_ids? | pages? |
  section?, method: auto|local|vision, dry_run: true, access_mode,
  token_budget?, reread: false, locate_only: false, wait}` — read only the
  equations a caller needs (Track N2, `docs/equations_v1.md`). A dry run
  (allowed `read_only`, audited `source_equations_plan`) lists the selection
  (`equations` with `status`, `would_read`, `selection.resolved_pages`), the
  copy's `regions` (`located`, `how_to_get` when not), each method's estimate
  (`methods.local`: available, pages, `estimated_seconds`, 0 tokens;
  `methods.vision`: available, `batches`, `calls`, `estimated_tokens`,
  `over_budget`) and `method.chosen` with `why`; it reads, converts and calls
  nothing. A real run (`maintenance_write`, 403 otherwise; backgrounded unless
  `wait`; audited `source_equations_read`) keeps a rollback snapshot, makes a
  copy's regions first when it has none (a free layout pass), reads by the
  local formula model on the selected pages only or by the `read_equations`
  vision skill (several crops a call, within the budget), writes each reading
  in place of its marker as `$$…$$` with its origin in the converted copy and
  the notes, and re-ingests: `method_used`, `read`, `failures`,
  `spent_tokens` with `spend_note` (what was billed, said apart from
  `estimated_tokens`; 0 for a reading reused from an earlier call), `billed_tokens`, `seconds`, `pages_converted`, `snapshot_id`, `notes`.
  422 `unknown_equation_ids` (with `known_ids`) or `section_not_found`; 409
  `no_regions` (with `how_to_get`) for a copy no converter kept regions for
- `POST /sources/equations/rollback` `{snapshot_id}` — put the converted copy,
  its equation regions and the notes back, remove crops a layout pass added,
  and re-ingest
- The governed answer, `task_context` and `/query/resolve_sections`:
  `unread_regions` (equations and figures their evidence holds unread, with an
  estimate per method and a `recover` call that works as given) and the
  warning `evidence_has_unread_regions`; `GET /review/source`:
  `unread_regions` with each marker's number, page and line (Track N3,
  additive to contract v2)
- `GET /ops/self_check` — `pdf_converters` (supported, selected,
  `effective_for_pdf`, each converter's availability, version and capability
  profile; warnings `pdf_converter_not_supported`, `pdf_converter_unavailable`)
  and `figure_vision` (available, by which backend, each backend's reason,
  `during_ingestion`; warning `figure_vision_enabled_but_unavailable`)
- The governed answer (`/agent/knowledge/answer`): an item read from a
  figure's image carries `image_derived: true`, `derived_from: "image"`, its
  confidence `image_derived: true`, and the answer the warning
  `image_derived_statements` (additive to contract v2)

### 6d2. Images and equations in every kind of source (Track T)
See `docs/images_v1.md`. Additive; no route here calls a model unless asked
to read an image.
- `POST /sources/process` takes an image (PNG, JPEG, GIF, WebP, judged by its
  bytes) as a source: `template_id: image` (or the caller's template when it
  takes images; otherwise `template_for_image: {requested, used, why}`), one
  figure, no agent extraction. Request `caption` sets the image's caption over
  the file's own metadata. A URL serving an image is taken in the same way.
  A file nothing here reads is refused before anything is copied: 415
  `{reason: unsupported_image | unsupported_binary, message}`, on the
  backgrounded path too. A source that converts to no words (a web page drawn
  by script, a scan without a text layer) is refused before any agent call:
  422 `{reason: no_readable_text, message}`; a backgrounded job fails with the
  same message. Its fetched or copied original and converted copy are kept.
  A web page (HTML) adds `conversion.html_trim` `{trimmed, kept: main |
  article | body, removed, visible_chars_before, visible_chars_after, reason?}`:
  its main content was converted without the site's menus, sidebars and
  footers, or why it was converted whole (`docs/document_conversion_v1.md`,
  "Web pages").
  When the converted copy has a table of data, the result adds `tables`
  `{tables, rows_total, rows_copied, collapsed, truncated}` and the notes a
  `## Source tables` section, copied by code (Track Y,
  `docs/source_tables_v1.md`). When the source holds a list of terms with
  their definitions (any two entries in a `reference-list` source, a run of
  `entries.min_entries_in_documents` in another kind), the result adds
  `entries` `{runs, entries_total, entries_copied, layouts,
  truncated_definitions}` and the notes a `## Source entries` section, copied
  by code, each entry a `Definition` with its term (Track AE,
  `docs/reference_lists_v1.md`); a `reference-list` source also has
  `share_of_lines`, and `warnings: [reference_list_mostly_prose: …]` when
  under half its lines are entries. For any other source except a PDF the result adds
  `images` (`images_found`, `figures`, `equation_images`, `not_kept` with
  reasons, `equations`: `written_in_text`, `as_images`, `unread`), and
  `figures` and `equations` as for docling
- `POST /sources/process_folder`: an image row has `image: true` and
  `estimated_tokens: 0`; an image a document of the folder embeds is
  `action: embedded`; a document row reports `images`; the dry run's
  `vision.sources_with_images` counts PDFs, image sources and documents with
  images
- `POST /sources/figures` works for any source with a figure catalog: each
  figure row adds `origin`, a `no_image` row its `reason`; new statuses
  `decoration_skipped` (an image too small to carry content); the plan adds
  `located_by`, `uncaptioned_policy` (`reads_uncaptioned`, `template_id`,
  `image_centred`) and `decoration_skipped`. For an image note a reading also
  fills `## What it shows`
- `POST /sources/equations`: a written equation is `read_from_text` (never
  planned); `methods.local` for a source that is not a PDF is unavailable
  with `offer: "vision"`; `pages` on a source without pages is 422
  `pages_not_applicable` (with `known_ids`); `section` also matches the
  headings a region sits under
- `POST /agent/knowledge/read`: an image is kept as a reference,
  `[Figure n (image: <workspace path>) …]` or `[image: <alt> (<path>)]`
- Settings: `config/ingestion.json` `vision.interpret_uncaptioned` (`null`:
  each template's default; a boolean; or `{template_id: bool}`) and `images`
  (`fetch_remote`: `fetched_pages` | `always` | `never`; `max_per_source`)

### 6e. Answers in fewer, larger steps (Track Q)
What a caller needs to answer in two or three calls; additive to contract v2.
See `docs/agent_memory_contract_v1.md`, "Answers in fewer steps".
- `POST /agent/knowledge/answer`, `POST /agent/knowledge/task_context`:
  `focus` (`kind`: `single_source` | `multi_source` | `unsure`, `decided_by`,
  `why`, `notes`, `evidence`, `read`: the read call for those notes, as
  given). Request `scope` `{note}` or `{citekey}` states the source; request
  `view: "compact" | "full"` (default `full`). Track X: while the concept
  layer lags its sources (a deferred backlog, or pending sources in a
  workspace that has concept notes), `/agent/knowledge/answer` adds the
  warning `concept_layer_behind` and `concept_layer` (`up_to_date: false`,
  `pending_sources`, `deferred_sources`, `oldest_deferred_at`, `run_with:
  POST /concepts/sync`), so its `concept_notes` are never read as current
  without saying so. Track AB6: `header_fields`, in both views, `[{note,
  fields: {name: value}, where: {name: <file>#frontmatter:<lines>}}]`: the
  header fields of the ranked notes (and of their sources' own frontmatter)
  that the question names, the collection's function words left out, a field
  named by its name first, the focus's notes first and alone for a
  `single_source` focus (at most 3 notes, 3 fields each, 200 characters a
  value; empty when none does); question coverage counts the fields shown. A
  note is also ranked by the values of the fields its type declares as
  identifying it (`identity_fields`), recorded at ingestion as `header_values`
  on its Note node, or read from its Markdown for a note ingested before; they
  weigh no word in the collection's statistics
- `POST /agent/knowledge/read` `{agent_id, workspace_id?, query?, notes:
  [<path | key | citekey> | {note, sections}], max_chars?, view?}` —
  read-only, audited (`knowledge_read`): per note, compact Markdown of the
  relevant parts of its source (whole when it fits, else the sections the
  question matches), read equations and interpreted figures elsewhere that
  match, `unread_regions` with `recover` calls, the owner's work about it and
  ideas resting on it, `left_out`; warnings `evidence_has_unread_regions`,
  `read_left_out_matching_sections`, `read_note_not_found`. 403 for a path
  outside the workspace, 404 when no note named exists, 422 without `notes`.
  `view` defaults to `compact`; `full` adds each part's structure
  (Track AB2) named sections come first, the source filling the rest; warnings
  `read_sections_not_found`, `read_truncated`, `read_budget_adjusted` (a
  `max_chars` outside 2,000 to 200,000, 0 included, is read at the nearest bound).
  (Track AB6) each note's Markdown opens with its header, `## [note] Header —
  <note>#frontmatter:<lines>`: the note's own frontmatter fields less the
  platform's bookkeeping (`PLATFORM_FRONTMATTER`,
  `docs/note_format_contract_v1.md`), each value at most 200 characters; then
  `## [source] Header — <source>#frontmatter:<lines>` for the fields of the
  source document's own frontmatter that the note does not repeat. Both blocks
  at most a sixth of the note's share of the budget (no floor; never over 1,200
  characters); with other sections named, only when within 300; a header that
  does not fit whole is the first `left_out` entry (`heading_path: "Header"`);
  `sections: ["Header"]` asks for it. `view: full` adds `header` and
  `source_header` (`origin`, `path`, `shown`, `fields[]`: `name`, `value`,
  `lines`, `cut`; `left_out`, `cut`, `lines`, `where`)
- `GET /agent/knowledge/changes` `{agent_id, workspace_id?, cursor?, since?,
  limit?}` (Track AB0) — read-only, audited (`knowledge_changes`): notes
  added, changed (their content), removed or renamed (Track AP5:
  `previous_note_id`, `previous_path`), oldest first, each with its own
  `cursor`; `next_cursor`, `has_more`, `position`, `log`, `delivery:
  at_least_once`. Without `cursor` or `since`, the newest; a cursor this log no
  longer holds lists again from the oldest (`changes_cursor_reset`); 422
  `changes_position_invalid` for one it cannot read. `limit` 1 to 500
- `POST /agent/knowledge/entity` `{agent_id, workspace_id?, name? | node_id?,
  limit?, offset?, types?}` (Tracks AB0, AB2b) — read-only, audited
  (`knowledge_entity`), no model call: one card for a note or any item by its
  path, key, citekey, identifier, title, short name, a page's alias or label,
  then by the same words up to a plural ending or an abbreviation the
  collection writes out (`relations`: typed links both ways; `referenced_by`:
  notes naming it; `neighbours`: what it names; every edge once, each list one
  of each kind in turn, `left_out` and `next_offset` for the rest; `counts`,
  `provenance`; a note's `summary` in plain prose and `read` call; an item's
  `defined_as`; a concept with its page as one card, `page`; Track AB6: a
  note's `header`, `{where, fields: {name: value}, left_out?, cut?}`, its
  frontmatter fields less the platform's bookkeeping and its `facets`),
  `also_matches`;
  a miss is `found: false` with name-like `suggestions` (`entity_not_found`),
  several equal matches `entity_ambiguous`. `limit` 1 to 50 per list
- `GET /agent/knowledge/overview` `{agent_id, workspace_id?, note_type?, limit?,
  offset?}` (Track AB2) — read-only, audited (`knowledge_overview`), no model
  call: what the collection holds for a caller that does not know yet:
  `holdings` (notes by type, concept notes, the owner's work, ideas, and
  `kinds`, every type `note_type` takes), the
  collection's notes by title with path, type, own `description` and
  `updated_at` (paged: `returned`, `total`, `has_more`, `next_offset`; with
  `note_type`, one kind of any tier), and on the first page up to 20
  `concepts` most notes name (`collection_notes_naming_it`).
  Warnings `collection_empty`, `note_type_not_held`. `limit` 1 to 500, default 25
- `GET /agent/knowledge/capabilities` `{agent_id, workspace_id?}` (Track AB3) —
  read-only, audited (`knowledge_capabilities`), no model call: `workspace`
  (`id`, the owner's `display_name`), `access.modes`, `brain_keeper`
  (`available`, `backend`, `reason`), `ceiling` (the workspace's
  `config/mcp.json` for agent connections: `max_surface`, `allow_proposals`,
  `configured`, `problems`). Warnings `brain_keeper_unavailable`,
  `mcp_config_invalid`
- `GET /agent/knowledge/review_queue` `{agent_id, workspace_id?, kind?, note?,
  limit?, offset?}` (Track AB3) — read-only, audited (`knowledge_review_queue`),
  no model call, no source read: the open work items newest first, each with
  `work_item_id`, `kind`, `reason`, `status`, `title`, its times, `notes`
  (workspace-relative) and what resolving or dismissing means; `open_total`,
  `by_kind`, `matched`, `returned`, `next_offset`. Warning
  `review_kind_not_open`. `limit` 1 to 200, default 25
- `POST /agent/knowledge/propose` `{agent_id, workspace_id?, access_mode:
  "propose", kind: "remember" | "forget", statement?, provenance?: {note?,
  source?, conversation?, quote?}, subject?, id?, reason?}` (Track AB4) — the
  one route that takes `access_mode: propose` (required, no default), refused
  403 `proposals_not_allowed` unless the workspace's `config/mcp.json` has
  `allow_proposals: true`, `agent_id` an id (`agent_id_invalid`), audited
  (`knowledge_propose`, refusals too), no model call: files one review work
  item and writes nothing else. `remember` needs `statement` and `provenance`
  (at least one of a note of the collection, a source file under `sources/`, or
  the conversation; a `quote` must be in the note or source as written);
  `subject` titles the note. `forget` needs `id` (a note's path, key or id, or
  an item's id) and `reason`. Answers `status` (`queued_for_review`,
  `already_queued`, `already_dismissed`, `already_accepted`,
  `already_withdrawn`), `work_item_id`, `proposal`, `review` (`resolve_means`,
  `dismiss_means`), `written: {markdown: false, graph: false}`; refusals
  `provenance_required`, `provenance_not_found`, `provenance_not_evidence`,
  `provenance_quote_not_found`, `proposal_incomplete`, `proposal_too_long`,
  `proposal_hidden_characters`, also `already_in_collection` (nothing filed,
  `in_collection`), `similar_pending`, `content_flags`,
  `proposal_target_not_found`, `proposal_target_ambiguous`,
  `proposal_target_not_evidence`. Resolved or dismissed by the owner with
  `POST /review/action` (`resolve_work_item`, `dismiss_work_item`): see
  `docs/agent_memory_contract_v1.md`
- `POST /maintenance/withdrawals/restore` `{withdrawal_id, agent_id,
  workspace_id?, scope_kind?, scope_id?, access_mode?: "maintenance_write"}`
  (Track AB4; `agent_id` required, caller, scope and policy resolved; a note
  changed since has its withdrawn lines put back, `reinserted`, and a note
  whose path holds a file again is `released`) — the owner's,
  audited (`withdrawal_restored`): puts back what an accepted `forget` withdrew
  (the note where it was, or each note as it was before a statement was taken
  out), re-ingests it, and marks the proposal's item restored; 409 when a file
  is there again or the note changed since, 404 for an unknown id
- `POST /maintenance/owner_sections` `{agent_id, workspace_id?, scope_kind?, scope_id?,
  access_mode?: "read_only" | "maintenance_write", apply?: false}` (Track AH4; `agent_id` required, caller,
  scope and policy resolved, as for the restore above) — the owner's: gives the notes the
  pipeline wrote before `## My notes` existed their empty owner's section
  (`docs/note_format_contract_v1.md`, "The owner's section"). A dry run unless `apply` is true.
  Only notes the pipeline wrote and the owner does not maintain are changed; a note that has
  the section (empty or not) is never touched, so a second run adds nothing; a note written by
  hand is left. Answers `dry_run`, `heading` (this install's name for the section), `looked_at`,
  `would_add`, `added` (the exact count written), `already_had`, `left_hand_written`,
  `left_owner_maintained`, `left_unreadable`, `notes` (at most 50) and `notes_more`,
  `backup` (the archive taken before the first write, only when something is written),
  `reingested`, `summary`, `billed_tokens: 0`; audited (`owner_sections_add`). No model call.
  A real run (`apply: true`) must say `access_mode: "maintenance_write"`: without it, or with
  `read_only`, it is refused 403 `maintenance_write_required` (audited as denied, nothing written,
  no backup), as `POST /sources/reconcile` does; the default `read_only` is enough for the count.
  422 for an `agent_id` that is not an id
- `GET /agent/knowledge/codes` `{code?, area?}` (Track AB3): only the codes
  asked for (`asked`; a family's code by its prefix, `family`; the start of a
  name lists the codes it begins, `matched_by`), `code_not_registered` for one
  that is not; `area` for one area's codes (every entry names its `area`);
  without either, every code
- `POST /agent/knowledge/read_next` (Track AB3): suggests only items tied to
  the question, each `reason` saying how; none tied is an empty list and the
  warning `read_next_nothing_matched`
- `POST /query/resolve_sections`: `view: "compact"` keeps each section's
  pointer and score and cuts its text to 1,500 characters
  (`text_truncated`); the full view (default) is unchanged
- `scripts/measure_answer_steps.py` — operator-run, never in the suite: the
  same questions through a headless Claude Code agent with and without the
  brain; turns, tokens with and without cache reads, time, and the brain's
  governed calls per run. Spends subscription tokens only with `--yes`.
  `approve_tokens` per question (or `--approve-tokens`) appends the same
  spending approval to both arms; a brain run whose focus notes held unread
  equations or figures and that recovered none is flagged `skipped_recovery`
  ("answer may be incomplete"), and a recovery's own spend is reported as
  `recovery_billed_tokens`. The audit of `knowledge_answer` carries
  `focus_notes` and `unread_notes`, that of `knowledge_read` `unread_notes`
  (paths only), which is what the flag reads

### 6f. The welcome, help, the example and the tour (Track AG1)
The first five minutes, with no collection of the caller's needed; the contract
is [[welcome-v1|welcome_v1.md]]. Every sentence for a person comes from the
catalogue (`second_brain/catalogue/welcome_v1.json`) or the user guide.

- `GET /welcome` `{agent_id?, workspace_id?}` — the next step: `state` (`new`,
  `example_open`, `collection_ready`, `collection_empty`, `collection_adding`),
  `say` (sentences for the person), `ask` (at most one question:
  `{text, recommended, answers, on}`, recommended first), `next` and `help`
  (the calls to make). With `workspace_id`, the collection's own state. Free.
- `GET /help` `{question}` — a question about Kovium itself, answered
  from the user guide's passages and the option catalogue's entries by code:
  `found`, `answer` (at most three `{text, source}`, `source` a guide
  `section` or a catalogue `entry`), `say`, `help_collection` (`not fetched`,
  `fetched`, `other_version` or `damaged`). With the full manual fetched and
  describing this version, the question is asked of it through the governed
  answer route and each `source` adds `document`; otherwise, or when it does
  not say, the guide and catalogue answer. When the manual can be had, `say`
  names it and `ask` offers it (a copy of another version: `ask` offers
  `update`). Free; no collection needed.
- `POST /welcome/manual` `{update?, agent_id?}` (Track AG6) — fetches the full
  manual (the help collection) by `git clone` into the install's own folder
  for collections, registers it under an agent of its own and builds it by
  code: `{fetched, state, notes, size_mb, say, ask_with}`. A current copy is
  left alone (`fetched: false`); one describing another version is reported in
  `say` with `update_with`, and `update: true` replaces it with the matching
  one. Refused `manual_not_set` (404: `config/help.json` holds only the
  placeholder), `manual_no_network` (502), `manual_no_git` (424) or
  `manual_fetch_failed` (502), each with a plain sentence. Free. Contract:
  [[welcome-v1|welcome_v1.md]].
- `POST /welcome/example` `{question?, agent_id?}` — the example collection
  opened (registered on first use, built by code when a note changed:
  `opened` `{built, seconds, notes, concept_pages, maps}`) and `questions`;
  with `question`, `found` and `answer` (at most three
  `{text, source: {note, section, lines, file, original}}`, no model call),
  `found: false` when the answer carries `question_not_covered`. Refused
  `example_not_found` (404) when the install holds no example. Free.
- `POST /welcome/tour` `{step?, try_it?, add?, approval?, reset?, wait?, agent_id?}` —
  `step` (`look`, `journey`, `files`, `notes`, `map`, `links`, `concepts`,
  `night`, `add`, or a number; none: the first): `{step, number, of, category,
  title, say, do, check_in, look, next_step, next, way_on}` (`title` composed:
  "Step 3 of 9 · ⚙️ Configuration: ..."; `check_in` only at the first step,
  else null; `way_on` the line that ends the message; `look.files`: each place
  to look at, its full `place` on this computer given at run time; the map step
  has `look.picture`; the notes step has `answer` and `reply`; a step with an
  exercise has `ask`, its offer; Track BD); unknown: `tour_step_unknown` (422).
  `try_it` (`both_ways`, `practice_night`; Track BA) runs a step's exercise,
  free: `{tried, ran, say, look, do, next_step, next, way_on}`. At `add`: `add: free`
  adds the waiting page by code (`added`, `note`, `map`, `links`,
  `concept_pages`, `do`, `ask_with`, and the tour's end: `ask` the manual offer,
  `also` the reset, `way_on`, `tour_end`); `add: ai` without `approval` is the plan
  (`estimated_tokens`, `estimated_range`, `limit`, `approval`, `ask`), with
  the plan's `approval` the run (a background job, or `wait: true`); a wrong
  or stale code is refused before anything runs (`adding_plan_changed`,
  `adding_approval_expired`); a page already added: `tour_sample_added` (409).
  `reset: true` puts the example back as it was. Spends only on `add: ai`
  with an approval. The free add also names `then`, the welcome again, whose
  question is then a collection of the person's own (Track AG2). Track AX:
  the first reply from the example in an install (from this route or
  `/welcome/example`) also says what happens at night, adds four sentences to
  `say` and offers to try the switch (`ask`, also as `nightly_ask`).
- `POST /welcome/nightly` `{answer, example?, workspace_id?, agent_id?}`
  (Track AX) — the person's answer to the nightly AI question: `yes` returns
  the second question (`ask`: light, medium, heavy, each with its most in
  brackets; `answered: false`); `no` or a level writes the collection's
  `ai_tokens_per_night` and `ai_level` through the settings path
  (`answered`, `level`, `tokens`, `shown`, `say`, `next`) and keeps it as the
  install's answer, which collections made later start at; without
  `workspace_id` only the install's answer is kept. Needs a manage entry that
  may change settings. With `example: true`, `on` or `off` only shows the
  switch in the example (nothing written, no AI). Refused
  `nightly_ai_answer_unknown` (422), `access_not_granted`. Free. Contract:
  [[welcome-v1|welcome_v1.md]].

### 6g. Collections and who may do what (Track AG2)
The setup tools' routes; the contract is [[collections-v1|collections_v1.md]].
Every sentence for a person comes from the catalogue
(`second_brain/catalogue/setup_v1.json`). Free; every write is checked against
the collection's own access policy (the `agents` table of its
`config/mcp.json`) for the agent the call names, and audited.

- `POST /collections` `{name, holds?, folder?, program?}` — a new, empty
  collection: `workspace_id`, `collection` `{id, name, holds, notes, folder,
  relative_to}`, `access` (the maker's, `manage` with every ability), `nightly`
  (Track AG5: `{enabled, time, machine, job}`: the free nightly steps are on,
  here only, and this computer's scheduler was asked to hold the job; the AI
  steps wait for a limit), `say`, `next`. `folder` in full, only when the person names one (one holding other
  files gets a folder of its own inside it). From a caller bound to another
  collection (`workspace_id`), it needs that one's manage entry, and `program`
  adds the new collection's own connection to that agent program. Refused:
  `collection_name_missing`, `collection_folder_not_absolute`,
  `collection_exists_there` (with the open call), `collection_folder_overlaps`.
- `GET /collections` — Track BC1: `collections` (`{id, name, holds, notes,
  folder, relative_to, joined}`, sorted by name; the example and the manual not
  listed; `notes` read from each database read only, `null` when unreadable),
  `say`, `next`. Opens and registers nothing. Free.
- `POST /collections/open` `{folder | collection, program?}` — an existing collection's
  folder registered here (or found registered), or (Track BC1) one already
  registered, by its name or id, this agent registered at the
  access it gives: `collection` `{id, name, notes, folder,
  already_known_here}`, `access` `{level, ..., claimed}`, `travel`
  (`folders_missing`, `search_built_elsewhere`, `program`), `say`. Refused
  `collection_folder_not_a_collection`, `collection_name_missing`,
  `collection_name_unknown`, `collection_name_ambiguous`.
- `POST /collections/join` `{collection | folder, program?, agent?, project,
  read_only?, level?, length?, tone?, extra?, confirm?}` — Track BC2 to BC4:
  this agent joins a collection under its own id. Without `confirm`:
  `collection` `{id, name}`, `program`, `agent`, `connection` `{name, place,
  why_wider?}` (`place` `project`, `agent` or `everywhere`), `access` `{level,
  ..., how}` (`claimed`, `read`, `narrowed`, `kept`), `agent_file`, `style`,
  `say`, `would_change`, and `asks` while the style is unanswered, else
  `confirm`. With it: the same and `joined`, `steps` (`connection` `{done,
  already, place, why_wider? | code, detail}`, `agent_file`). Refused:
  `program_unknown`, `program_not_found`, `agent_project_not_found`,
  `join_agent_missing`, `join_agent_shared`, `join_answers_missing`,
  `collection_name_*`, `collection_folder_*`, `agent_file_taken`,
  `confirm_code_invalid`, `confirm_code_stale`, `access_not_granted`. See
  `docs/collections_v1.md`.
- `POST /collections/leave` `{collection, program?, agent?, project}` — Track
  BC4: `collection`, `program`, `agent`, `left`, `steps` (`connection`
  `{done, removed, place | code, detail}`, `agent_file` `{outcome, file}`
  (`removed`, `absent`, `not_ours`), `binding`), `say`. The collection stays.
- `POST /collections/connect` `{program, agent?}` — this collection's own
  connection (`kovium-<id>`) added to Claude Code, Codex, Cursor or
  OpenClaw, its agent named `read`: `program`, `agent`, `connection` `{added,
  already, name}`, `access`, `say`. Needs a manage entry that may grant.
  Refused: `program_unknown`, `program_not_found`, `program_entry_differs`,
  `program_connection_failed`, `program_not_started`, `connect_same_agent`.
- `POST /collections/access` `{agent, level, may?, per_run_tokens?,
  per_day_tokens?, confirm?}` — without `confirm`: `changed`, `access`,
  `would_change`, `confirm` and the sentence in `say`; with it: `changed`,
  `applied`. Refused:
  `access_self_raise` (an agent raising its own), `access_not_granted`,
  `confirm_code_invalid`, `confirm_code_stale`.
- `POST /collections/settings` `{option, value?, add?, remove?, text?,
  confirm?}` — one choice by its plain name, its id or a setting it holds
  (`value` may be a number: the nightly AI level, which also takes `light`,
  `medium`, `heavy` or `off` (Track AX), the minutes of equations at night): `option`, `name`, `would_change`, `say`, `confirm`; with `confirm`:
  `changed`, `applied`, `file`. An unknown name: `setting_unknown`, its `detail` naming
  the closest choices and all of them. Refused:
  `setting_unknown`, `setting_not_changed_here`, `setting_value_invalid`,
  `setting_example_no_ai` (a nightly AI level in the example).
- `POST /collections/agent` `{project, program?, level?, length?, tone?, extra?,
  confirm?}` — Track AG4: this collection's caller agent added to the person's
  project beside its own agents. Without `confirm`: `file` (relative to the
  project), `agents_beside`, `style`, `asks` (the offer, simple or technical,
  short or detailed, tone and language, anything else; each with its
  recommended answer first), `would_change`, `say` and `confirm`; with it:
  `changed`, `applied`, the style kept in the collection's `config/style.json`.
  Needs a manage entry that may change settings. Refused:
  `agent_project_not_found`, `agent_file_taken`, `agent_program_not_ready`,
  `program_unknown`, `confirm_code_invalid`, `confirm_code_stale`,
  `access_not_granted`. See `docs/install_and_caller_agent_v1.md`.
- `GET /collections/setup` `{probe?}` — `items` (`{option, name, state, say}`,
  `state` `works`, `on`, `off` or `missing`), `off`, `say`, and with a
  collection `budget_tokens_per_run` and `access`; `choices` (what
  `/collections/settings` takes). Track AG5: every feature that spends tokens
  and ships off is an `off` item whose `say` carries its cost; the nightly tidy,
  the nightly AI level (Track AX: `level`, `tokens`, `shown`, and `automatic`
  for a schedule from before the levels, read as medium) and the nightly
  reading of equations are items of a collection; `versions` (`{kind, ...}`: `schema_newer`,
  `schema_older`, `platform_newer`, `platform_older`, `search_program`,
  `search_model`, `search_mixed`) and `versions_say` say what differs from the
  versions that wrote the collection. Read only.
- `GET /ops/workspaces` `{agent_id}` with no `workspace_id`, in strict scope:
  the registered ids and their agents (`entries`), never a folder; the call
  `workspace_not_named` names.

### 6h. Access tokens (Track BB)
The contract is [[access-tokens-v1|access_tokens_v1.md]]. With a
collection's `require_token` on (the catalogue's `access_keys`, off for every
collection until its owner turns it on), every route that names it refuses a
call without a valid token for it: `token_required`, `token_unknown`,
`token_other_collection`, `token_expired` (401, `WWW-Authenticate: Bearer`)
and `token_agent_mismatch` (403). The token goes in `Authorization: Bearer
<token>` on every request; an omitted `agent_id` is the token's agent. The
system header (`X-Kovium-System`) and the owner's signed-in browser are
exempt. What a holder may call is its policy entry's level
(`access_not_granted` otherwise).

- `POST /collections/tokens` `{agent, project | to, level?, may?, expires?,
  confirm?}` — a token for one agent: without `confirm`, `agent`, `access`,
  `file` (relative to `project`), `ends_at`, `would_change`, `confirm`, `say`;
  with it, `issued`, `short` (the first 8 characters of its fingerprint) and
  the same, the token written to `<project>/.kovium/<collection>.token`
  (or `to`, the owner's script's only) and never returned. Needs a manage entry
  that may grant, and over plain HTTP a caller signed in with its own token
  (`token_issue_not_allowed`), or the system header. Refused also:
  `access_self_raise`, `access_agent_invalid`, `agent_project_not_found`,
  `token_file_outside_project`, `token_end_invalid`, `confirm_code_*`.
- `GET /collections/tokens` — `tokens` (`{short, agent, level, made_at,
  made_by, ends_at, expired, last_used}`), `require_token`, `say`. Needs a
  manage entry that may grant. Never a token.
- `POST /collections/tokens/revoke` `{fingerprint}` — `revoked`, `short`,
  `agent`, `say`: the token is refused from the next call. An agent's own, or
  another's with a manage entry that may grant. Refused:
  `token_fingerprint_invalid`, `token_not_found`, `token_fingerprint_ambiguous`.
- `POST /ui/sign-in/code` `{next?}` — the system header only
  (`sign_in_link_owner_only`): `link` (`/ui/sign-in?once=<code>`), valid ten
  minutes, once. The start scripts print it.
- `GET /ui/sign-in` `{once, next?}` — signs the browser in (an HttpOnly,
  `SameSite=Strict` cookie) and redirects to `next` (a page of this server) or
  `/`; a used, unknown or old code answers 401.

### 7. Concept notes
One maintained note per concept the collection's notes name — see
`docs/concept_notes_v1.md`.
- `GET /concepts` — every concept note, plus `status.up_to_date` and the
  source notes still pending; a source whose upkeep was deferred (Track X)
  carries `deferred: true`
- `GET /concepts/{key}` — one concept: definition, development, divergences,
  and its source notes oldest first, each with a path an agent can open; with
  `definition_basis` (`sources`, `sources_and_general_knowledge`, or null),
  also on every `GET /concepts` row. Both reads are audited (`concepts_list`,
  `concept_read`; see `docs/workstream_b_governance_and_routing_policy_v1.md`)
- `POST /concepts/sync` — absorb pending (or named) source notes; backgrounded
  unless `wait: true`. One pass for all of them, in their own date order: each
  read once, then what concerns the concepts (hierarchy, comparisons,
  condensing) once over the union (Track X). This is how a deferred backlog is
  paid; the result carries `run_id` and `sources_in_pass`, and every call it
  makes is recorded in the call log with `for_sources` and `concepts`. A call
  that failed is tried once more in the pass, within the standing budget (Track
  AN): `agent_calls` and `agent_failures` count real attempts only (an install
  with no agent attempts nothing: `agent_available: false`, both 0), `retries`
  and `retries_recovered` say what was retried, `billed_tokens` counts every
  attempt, and `left_for_agent` (`count`, `concepts`, `why`,
  `retry_estimate_tokens`, `line`) names the concepts written without the agent,
  why, and what finishing them would cost; they stay pending and are never
  counted in `sources_finished`
- `POST /concepts/hierarchy` — recompute broader/narrower concepts (no tokens)
- `POST /concepts/open_problems` `{keys?}` — rewrite each concept note's
  `## Open problems` from its sources' stated limitations that name it (no
  tokens); owner-maintained notes are reported in `owner_maintained`, not written
- `POST /concepts/ideas` `{keys?}` — rewrite each concept note's `## Ideas`
  from the idea notes about it (no tokens); owner-maintained notes are reported
- `POST /concepts/consolidate` — condense concept notes whose summary or
  definition outgrew reading length
- `POST /concepts/merge` `{keep, absorb, reason}` — fold one concept note into
  another: sources, aliases and links move to `keep`, every source note is
  relinked, `absorb` becomes a redirect stub. Refused (409) for an
  owner-maintained note or a `keep_separate` pair. `GET /concepts` lists
  recent `merges`
- `POST /concepts/unmerge` `{merge_id, force?}` — restore every file a merge
  touched; refused (409) when one changed since, unless `force`
- `POST /concepts/compare` `{keys?, force?, wait?}` — write how the sources of
  widely used concepts model them differently (`### How sources model it` under
  Divergences); one agent call per concept whose sources changed. Since Track
  X, sources added since the last comparison are placed against it; the whole
  comparison is redone when none was written, on `force`, or once the
  concept's sources reach `compare_refresh_factor` times those it last
  compared in full (a sync never redoes it)
- `POST /concepts/curate` `{dry_run?, wait?}` — one curation pass: sync pending
  sources, find likely duplicate notes (no tokens), merge the typographically
  identical ones, judge the rest once with the agent, propose what is unsettled
  as `concept-merge:<a>+<b>` work items. `POST /review/action` with
  `resolve_work_item` on one merges the pair; `dismiss_work_item` writes
  `keep_separate` on both notes. Run nightly by `scripts/nightly_maintenance.py`

### 7a. Ideas
What someone thinks might be worth doing, kept apart from what the sources say
— see `docs/ideas_v1.md`. Never evidence: answers list them under
`related_ideas`, never among `knowledge_items`.
- `GET /ideas` `{status?, origin?, concept?}` — every idea note (audited, `ideas_list`)
- `GET /ideas/{key}` — one idea in full, with what it rests on resolved and its
  review items; 404 for an unknown key (audited, `idea_read`)
- `POST /ideas` `{access_mode: maintenance_write, title, statement, why?, test?,
  notes?, status?, origin?, concepts?, rests_on?, topic?, maintained_by?}` —
  capture a new idea note in `sources/wiki/ideas/` and ingest it; 201. Without
  `access_mode` it is 422; an owner's idea defaults to `maintained_by: owner`
- `POST /ideas/review` `{access_mode: maintenance_write, notes?, all_sources?,
  stale?, duplicates?}` — raise `idea-tested:<idea>+<note>`,
  `idea-stale:<idea>` and `idea-duplicate:<a>+<b>` review items; acts on none.
  Ingest routes run the tested check for each note they ingest, and report it
  under `ideas`

### 7b. The owner's own work
The owner's designs and their commentary on other people's work, kept apart
from the collection — see `docs/owner_work_v1.md`. Never evidence: answers list
them under `from_your_work`, never among `knowledge_items`.
- `GET /owner-work` `{kind?, status?, concept?}` — every owner-work note, with
  `text_changed` (files edited since read), `text_missing`,
  `read_next_candidates` (works a commentary discusses that the collection does
  not hold), `about_unconfirmed` (notes only suggested a source, with the
  candidates; Track U1) and `config_problems` (audited, `owner_work_list`)
- `GET /owner-work/template` `{kind}` — a blank note with the template's headings
  (`design`, `commentary` or `notes`)
- `GET /owner-work/open_problems` — each design's assumptions and limits, the
  owner's own open problems (audited)
- `GET /owner-work/{key}` — one note: its parts as mapped from its own headings,
  `body` (unmapped sections), `headings` (which became what, and why), what it
  discusses (`about_ref`), `comparisons`, `open_problems`, review items,
  `read_next_candidate` and its text (`markdown`); 404 for an unknown key
- `POST /owner-work` `{access_mode: maintenance_write, path, kind?, status?,
  about?, concepts?, compares_with?, headings?, title?, default_part?, update?}` — register an
  owner's file (workspace path or `<root>:<path>`) without modifying it: writes
  a registration note in `sources/wiki/own-work/`, ingests it with no agent
  call, refreshes `## Your work`, runs the review below; 201, or 200 when
  `unchanged`/`refreshed`/`updated`. 422 without `access_mode`; 400 with no
  `kind` or outside the workspace and every declared root; 404 no file; 415 not
  Markdown or text; 409 a note of the collection, or already registered with
  other fields and no `update: true`
- `POST /owner-work/refresh` `{access_mode: maintenance_write, keys?, force?}` —
  read again each file whose bytes changed; updates the graph, `## Your work`
  and the registration's `sha256`/`updated`, never the owner's file
- `POST /owner-work/review` `{access_mode: maintenance_write, keys?}` — raise
  `owner-work-differs:<key>+<note>` (a commentary states a value differently
  from the source note it discusses) and `owner-work-unheld:<key>` (it
  discusses a work the collection does not hold, naming a file in the intake
  folder that looks like it), and `owner-work-link:<key>` (a note that names
  no source and might be about one of a few: the candidates, for the owner to
  confirm; Track U1); acts on none. Ingest routes run it for the owner
  notes that name each ingested source, and report it under `ideas.owner_work`
- `POST /concepts/your_work` `{keys?}` — rewrite each concept note's
  `## Your work` (no tokens)
- `POST /query/gaps/limitation_clusters` also returns `owner_open_problems`,
  apart from the clusters; `POST /sources/reconcile` returns
  `owner_work_changed` when an owner's file was edited

### 7c. Instance tails (Track R)
A short, approved description of one instance appended to the shared
instructions of the jobs that accept one — see `docs/instance_tails_v1.md`.
Tails live only in the instance (`maintenance/tails/`); nothing is used until
approved, and only while `config/tails.json` says `enabled: true`. Writes need
`access_mode: maintenance_write` (403 and an audited `denied` without it).
- `GET /tails` — the instance's tails (approved version, pending draft,
  history), its tail settings and `skills_accepting_tail` (audited, `tails_list`)
- `GET /tails/preview` `{tail_id?, version?}` — one version's rendered text,
  tokens, fingerprint and any `problems`; or, with `note_type?`,
  `template_id?`, `source_path?`, `facet=name=value` (repeatable), which tail a
  job with those selectors gets and why each approved tail did or did not
  match (`chosen`, `candidates`, `jobs_get`); 404 for an unknown tail
- `POST /tails/draft` `{access_mode, tail_id?, applies_to?, fields?,
  use_agent?}` — build a draft version (deterministic glossary, facets, note
  types, writing systems, declared languages; descriptive fields from `fields`,
  the `draft_tail` skill within `max_tokens_per_refresh`, or the approved
  version) and raise `tail-draft:<id>@v<N>`; 201 with `text`, `dropped`,
  `alias_doubts` (`review_item`, `conflicts`, `unproven`, `superseded`),
  `agent`, `billed_tokens`. Glossary variants are only the aliases that are
  name-equivalents; the others are listed in one review item per draft,
  `concept-alias-doubts:<id>@v<N>` (conflicts first, then a table), which
  supersedes the previous one and any per-concept `concept-alias-doubt:`
  item; never acted on, concept notes untouched. 422 with `problems` for owner fields that command,
  hide or exceed a cap; 409 when the instance holds nothing to describe
- `POST /tails/approve` `{access_mode, tail_id, version}` — screen again and
  approve; supersedes the tail's earlier version and closes the review item;
  calls no agent and reads no source again. 422 with `problems` for a version
  that no longer passes, 409 when another approved tail has the same
  `applies_to`, 404 unknown. Resolving the review item through
  `POST /review/action` does the same (409 when refused, item left open);
  dismissing it rejects the draft
- `POST /tails/retire` `{access_mode, tail_id, version?}` — retire the approved
  version, or reject the draft `version` names
- `POST /tails/maintain` `{access_mode?, dry_run? (default true), force?}` —
  measure drift since each approved tail on the instance's cadence and, past
  `drift_threshold`, propose a refreshed draft (a review item, never an
  approval); `skipped` when tails are off, the cadence is `off` or not yet due.
  A real run needs `maintenance_write`
- Jobs record `tail: {tail_id, version, fingerprint}`; notes record
  `tail: <id>@v<N>` (native notes) or `template_tail: <id>@v<N>` (imported
  sources) in their provenance

## Example Agent Queries
A caller agent should be able to ask, of any collection:
- what unresolved questions are most central?
- which methods or decisions are weakly connected to evidence?
- what bridges exist between part A and part B of the collection?
- what concepts surround X, and how do the sources differ on it?
- what evidence supports this claim, and where exactly is it?

## Response Shape Principles
All API responses should include:
- `status`
- `data`
- `provenance_summary?`
- `warnings?`
- `notices?`: one per warning code found anywhere in the response, each with
  `code`, `message`, `detail?`, `where` and `fix` (`next`: `action`, `say`,
  `call?`; `verify`; `consent`). See `docs/next_steps_v1.md`.
- `generated_at`

A refusal keeps FastAPI's `detail`. On `/agent/knowledge/*`, `/sources/*`,
`/ingest/*` and `/normalize/*`, and whenever its route gave it a code, it also
carries `error` in the notice's shape (a validation error: `error.code:
invalid_request`). `GET /agent/knowledge/codes` returns `{actions, codes}`:
every code and its fix, the same for every workspace.

## Subgraph Response Shape
Recommended fields:
- `nodes`
- `edges`
- `summary`
- `cluster_info?`
- `focus_node?`

## Gap Query Response Shape
Recommended fields:
- `selected_categories`
- `gaps`
- `supporting_metrics`
- `supporting_nodes`
- `recommended_actions`

## V1 Authentication
Loopback binding and process privacy, and, per collection, access tokens
(Track BB, section 6h, [[access-tokens-v1|access_tokens_v1.md]]): off until
a collection's owner turns them on.
