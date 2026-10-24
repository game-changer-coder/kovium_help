---
type: "documentation"
title: "Next steps v1: warnings and refusals that say what to do (Track AC)"
description: "Every warning a response carries, and every refusal of a governed or intake route, comes with its next step as data: what it means, what to do, the call that does it with the caller's own arguments, and the read-only call that shows it..."
status: "stable"
reading_status: "read"
product: "second brain"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing second brain"
doc_path: "docs/next_steps_v1.md"
doc_sha256: "8f20f5ddc96446996406533713274e64408a7d11079bb53e31c32eba2e170dd2"
---

# Next steps v1: warnings and refusals that say what to do (Track AC)

Every warning a response carries, and every refusal of a governed or intake
route, comes with its next step as data: what it means, what to do, the call
that does it with the caller's own arguments, and the read-only call that
shows it resolved. A caller agent acts on `fix.next.action` instead of reading
prose. Plan: `docs/knowledge_layer_improvement_plan_v1.md`, Track AC. Code:
`second_brain/next_steps.py` (the shape), `second_brain/next_steps_codes.py`
(the registry), `second_brain/api/next_steps_wiring.py` (how routes attach it).

## Why

A caller given `concept_layer_behind` or `no_readable_text` had to recognise
the word, find the remedy in a skill or a document, and work out its
arguments. Each code now carries that remedy itself, and one table in code
holds every code, so a route cannot introduce one without its fix.

## The shape

Additive: `warnings`, `warning_details` and a refusal's `detail` are unchanged.

**A response** (any route answering `{status, data, warnings, generated_at}`)
gains `notices`, one per distinct code found in any `warnings` list it holds,
at any depth, when there is at least one:

```json
{"code": "concept_layer_behind",
 "message": "The concept notes lag their sources: ...",
 "detail": "",
 "where": ["data.warnings"],
 "fix": {"next": {"action": "run", "say": "Bring the concept notes up to date.",
                  "settled_by": "caller",
                  "call": {"method": "POST", "route": "/concepts/sync",
                           "body": {"wait": false, "agent_id": "...", "workspace_id": "..."}}},
         "verify": {"method": "GET", "route": "/concepts",
                    "body": {"agent_id": "...", "workspace_id": "..."},
                    "expect": "status.up_to_date is true"},
         "consent": "It spends brain-keeper tokens, within the collection's standing budget."}}
```

- `detail`: the value of a parameterised warning (`thin_chapters:3,7` is
  `thin_chapters` with detail `3,7`), several joined by `; `.
- `where`: where in the response the code appeared (at most five places).
- `fix.next.call` / `fix.verify`: `method`, `route` (path parameters filled)
  and `body` (query parameters for a `GET`). Arguments come from the request
  that raised the code (its `agent_id`, `workspace_id`, `query`, `path`,
  `url`, `notes`, ...) and from identifiers the response names (a `job_id`).
  An argument the request did not carry is left out and listed in
  `fill`, for the caller to supply (`notes` from the answer's `focus`, say).
  Absent when no call takes the step.
- `consent`: what the call costs or changes (it spends tokens within the
  standing budget, or beyond it); `null` otherwise.
- `fix.next.settled_by` (Track AR3): who settles it. `system`: the knowledge
  brain system, by its own default (nothing for the caller to do); `caller`:
  the caller agent, by the recommended choice, without asking the user;
  `user`: the user, told (`tell_user`, `report`) or asked (`ask_user`).
- `fix.next.user_text` (Track AR5): for a code that can reach a person
  (`ask_user`, `tell_user`, `report`), what to tell them, in plain words: at
  most 25 words a sentence, no code, field, route, file path or word of the
  platform's own. A caller relays this, never `say`, which is for agents.

**A refusal** of a governed or intake route (`/agent/knowledge/*`,
`/sources/*`, `/ingest/*`, `/normalize/*`), and a refusal any route raised
with a code, keeps FastAPI's `detail` and gains `error` in the same shape,
with `detail` holding the refusal's message:

```json
{"detail": "source not found: nowhere.md",
 "error": {"code": "source_not_found", "message": "No file at that path in this workspace.",
           "detail": "source not found: nowhere.md",
           "fix": {"next": {"action": "run", "say": "Check the path: ..."}, "verify": null,
                   "consent": null}}}
```

A refusal's code is the one its route gave it (`no_transcript`,
`maintenance_write_required`, `source_not_found`, ...), else the `reason` of a
`{reason, message}` detail, else one named by its status (`request_refused`,
`not_permitted`, `not_found`, `conflict`, `unsupported_source`,
`server_error`). A request whose arguments do not match the route keeps
FastAPI's validation `detail` and gains `error.code: invalid_request`.

## Actions

| action | what the caller does |
|---|---|
| `run` | makes the call itself: a read, a retry, a choice the system recommends, or a spend within the standing budget |
| `ask_user` | puts the response's `question` to the user, word for word, and makes the calls its answer names. Only these are ever asked (the owner's rule, 2026-10-20, and Track AZ): the standing token budget (`asks: budget`), the user's own originals (`asks: originals`: deleting or overwriting one, and keeping a copy of each or pointing to it), and where a new collection is kept (`asks: place`); a test fails on any other |
| `tell_user` | tells the owner: the remedy is theirs (a setting, an install, a missing source) |
| `wait` | the work is under way: waits on `fix.verify` (one waiting call, never polling) |
| `retry` | a passing failure: tries again later |
| `none` | nothing to do; the code describes the result |
| `report` | a defect: reports it with the response |

## The registry

`GET /agent/knowledge/codes` returns `{actions, codes}`: every code with its
`message`, `action`, `say`, `call`, `verify`, `expect`, `consent`, `about`,
`settled_by`, `asks` and `user_text`,
the calls unfilled (`{query}` marks an argument filled from the request). The
same for every workspace. A code ending in `*` covers each code it begins
(`normalization_config_*`). `about` (Track AB2) is `call` for a condition of
the request, or `install` for one of the install or the workspace that every
call meets alike until the owner changes it (semantic search unavailable, the
concept layer behind its sources): the MCP server says those in full once a
session.

A code is added in `second_brain/next_steps_codes.py` with `register(...)`;
then `python scripts/next_steps_doc.py` rewrites the table below. The test
suite runs with `SECOND_BRAIN_STRICT_NOTICES=1`: a response or refusal naming
a code the registry does not know fails the test that met it, and so does a
warning written as prose instead of a code. In service an unknown code is
returned with `action: report` rather than costing the response.

<!-- codes:start -->
| code | means | next | call | check |
|---|---|---|---|---|
| `about_ambiguous` | More than one note could be the work it is about. | **run**: Register it with `about` naming the candidate its own text fits best. |  |  |
| `about_not_held` | The work it is about is not in this collection. | **tell_user**: Say so; it links once that source is taken in. |  |  |
| `about_not_stated` | The commentary does not say which work it is about. | **run**: Register it with `about` naming the work its own text points to; if nothing does, leave it unlinked. |  |  |
| `about_uncertain` | The work it is about was suggested, not stated: the link is kept as a suggestion. | **none**: Nothing to do: the suggestion waits in review, and the note is linked as suggested meanwhile. |  | `GET /owner-work/{key}` |
| `access_agent_invalid` | The agent named is not an agent id (see `detail`). | **run**: Send the agent by its program's name or an id of letters, digits, - and _. |  |  |
| `access_not_granted` | This agent's access to the collection does not allow the call (see `detail`): the collection reads only for it, or its manage entry lacks that ability. | **tell_user**: Say so in its words; only another agent that may manage the collection and grant can allow more, on the owner's word. Never ask to raise your own access. |  | `GET /collections/setup` |
| `access_self_raise` | An agent asked to give itself more access. | **tell_user**: Say so; another agent that may manage the collection and grant can, on the owner's word. |  |  |
| `adding_approval_expired` | The plan's approval is older than its validity (see `detail`). | **run**: Plan again and run with the new approval. | `POST /sources/add` (paths: {paths}) |  |
| `adding_approval_required` | A real addition needs the approval code of a plan made on the same files (a dry run first). | **run**: Plan first, then run with the `approval` the plan gives. | `POST /sources/add` (paths: {paths}) |  |
| `adding_book_read_whole` | A long document's chapters could not be told apart with confidence by any step of the chapter-finding ladder (its outline, its contents page, its headings, and the brain keeper within the budget), so it was read as one document, in parts; each row's `split_line` says why. | **none**: Nothing to do; splitting it later is a re-read, with `chapters` if the user names them. |  |  |
| `adding_budget_needed` | This addition needs a token budget the user has not given: none is set, or it needs more than the standing budget (see `question`). | **ask_user**: Put `question.text` to the user word for word, recommended answer first; on yes, make the calls `question.on_yes` names. Unanswered, nothing runs. |  |  |
| `adding_concepts_left_for_agent` | The concept pass wrote some concepts without the agent (its call failed and was tried again, a retry would pass the budget, or no agent is set up), so they wait for it (see `concept_pass.left_for_agent`: which concepts, why, and what finishing them would cost). | **none**: Nothing to do; the next concept pass or the nightly runner reads them with the agent. To finish them sooner, make the call (it spends tokens within the standing budget). | `POST /concepts/sync` (wait: False) | `GET /concepts` |
| `adding_job_not_found` | No addition of this collection has that job id (see `detail`). | **run**: Ask for the status without a job id: the latest addition of this collection is reported. |  |  |
| `adding_kind_unknown` | The kind of these files could not be chosen: the request names none, no default kind is set, and the collection holds no notes to go by. | **run**: Plan again with `kind`: the kind most of these files are, chosen by you from the files and the plan's `kinds` (never a question for the user); each file whose own look says otherwise still gets its own. | `POST /sources/add` (paths: {paths}, kind: {kind}) |  |
| `adding_nothing_named` | The request names no file, folder or link. | **run**: Send `paths` (files or folders, relative to the collection or `root:path`) or `urls`. |  |  |
| `adding_nothing_to_add` | Nothing named needs adding: everything is already in, or was skipped with its reason (see the report). | **none**: Nothing to do; relay the report. |  |  |
| `adding_originals_choice` | Some files are outside the collection and it has not decided yet whether it keeps a copy of each original or points to it where it lies (see `asks`, about `originals`). | **ask_user**: Put the question to the person word for word, recommended answer first; their answer is the plan made again with `originals` (keep or point), as `on` names. A person who already said "keep copies" has answered: send `originals: keep`. |  |  |
| `adding_over_agent_limit` | The run's approval allows more than this agent may be approved for: its per-run cap, or what its day's cap leaves (see `detail`). | **tell_user**: Plan again (the new plan stays within the cap); more is the owner's to allow, through an agent that may grant (set_access). | `POST /sources/add` (paths: {paths}) |  |
| `adding_path_not_found` | Some names given are no file or folder of this collection (see `detail`); the rest were planned. | **run**: Check those names: relative to the collection, or `root:path` for a declared source root. |  |  |
| `adding_plan_changed` | The files, their kinds or the budget changed since the plan was made, so its approval no longer matches. | **run**: Plan again and run with the new approval; ask the user only if the new plan asks. | `POST /sources/add` (paths: {paths}) |  |
| `adding_retried` | Some documents failed or came out partial and were tried again by the system (see `retries`). | **none**: Nothing to do; what still failed is a line in the report. |  |  |
| `adding_stopped_at_budget` | The addition stopped cleanly at its budget; what is done and what is left are in the report. | **ask_user**: Put `question.text` to the user; on yes, add the same files again and run them with the approval the new plan gives (what is in is skipped). | `POST /sources/add` (paths: {paths}) |  |
| `adding_version_not_linked` | A document planned as another version was added, but not linked: its other version was not added, made no single note, or has no note in the graph the link could point at, so nothing was kept (see `versions`, `reason`). | **none**: Nothing to do; adding the other version again does not link them, so a link wanted later is written as `version_of` in the note. |  |  |
| `adding_versions_linked` | Some documents are other versions of a work the collection already had, or of another document in the same addition: each was taken in and linked to it as a version (`version_of`; the evidence is in `versions`). | **none**: Nothing to do; the report says so in plain words. To hold versions back next time, send `versions: skip`; to take them in unlinked, `versions: separate`. |  |  |
| `agent_file_taken` | The project already has an agent file of that name that second brain did not write; it was left as it is. | **tell_user**: Say so; nothing was added, and that file is the person's to keep or move. |  |  |
| `agent_id_invalid` | The agent id is not an id (see `detail`): letters, digits, dots, underscores and hyphens only. | **run**: Send the request again with a plain agent id. |  |  |
| `agent_program_not_ready` | The caller agent is not made for that agent program yet. | **tell_user**: Say so; its connection works without it, and the program gets the agent in a later version. |  |  |
| `agent_project_not_found` | The project folder for the caller agent was not given, or is not a folder on this computer. | **run**: Send `project`: the full path of the folder this agent program works in (your own working folder); never ask the person to type it. |  |  |
| `all_boundaries_rejected` | Every chapter boundary proposed was rejected. | **run**: Look at the split in a dry run (it reads, and spends nothing). Without a confident split the document is read whole, in parts (adding documents does this by itself); a split the user names is sent as `chapters`, never asked for. | `POST /sources/process_book` (path: {path}, dry_run: True) |  |
| `answer_figures_not_in_evidence` | Numbers or quotations in the written answer are not verbatim in its evidence: computed, reformatted or wrong. | **run**: Check each of `synthesized_answer.unverified_figures` in its source before using it. | `POST /agent/knowledge/read` (query: {query}, notes: {notes}) |  |
| `answer_not_in_retrieved_text` | The answer was not in the text read, which is not the whole collection. | **run**: Ask again reading whole notes and originals. | `POST /agent/knowledge/answer` (query: {query}, synthesize: True, evidence: full) | `POST /agent/knowledge/answer` (query: {query}) |
| `argument_not_offered` | An argument this tool does not take was ignored (see `detail`). | **run**: Send only the arguments the tool lists in its input schema. |  |  |
| `batches_failed` | Some text was never extracted (the ranges in `detail`). | **run**: Re-run the note; until then, do not rely on it for those ranges. |  |  |
| `batches_rejected` | Some batches' extraction was rejected (which, and why, in `detail`). | **run**: Re-run the note's extraction (the review action `request_renormalization`). |  |  |
| `book_over_docling_page_ceiling` | The book has more pages than the workspace's docling ceiling. | **run**: Process with `force_long_conversion`: the time is part of the plan the budget covers (adding documents does this by itself). | `POST /sources/process_book` (path: {path}, force_long_conversion: True) |  |
| `book_owner_split_invalid` | The chapter list given cannot be read: each part is `{title, pages}`, pages as `41-78` (see `detail`). | **run**: Send the list again in that form, in reading order, or leave `chapters` out to let the ladder find them. |  |  |
| `book_owner_split_refused` | The chapter list given does not fit the document: its parts are out of order, overlap, leave pages out, or name pages the document does not have (see `detail`). Nothing was split. | **run**: Correct the list (every page from the first part to the end in some part, in order) and send it again, or leave `chapters` out. | `POST /sources/process_book` (path: {path}, dry_run: True) |  |
| `book_pages_not_counted` | The book's pages could not be counted, so its time is not estimated. | **none**: Nothing to do; the estimate is missing, not wrong. |  |  |
| `book_pages_unknown_for_docling_ceiling` | The book's pages could not be counted against the docling ceiling. | **run**: Process with `force_long_conversion`, or select another converter; adding documents does the first by itself. | `POST /sources/process_book` (path: {path}, force_long_conversion: True) |  |
| `book_part_not_found` | A note named is not a note of this collection. Nothing was recorded (see `detail`). | **run**: Name the parts again as `{note, pages}`, in reading order. |  |  |
| `book_parts_bad_pages` | The pages of a part are not a range like 41-78. Nothing was recorded (see `detail`). | **run**: Name the parts again as `{note, pages}`, in reading order. |  |  |
| `book_parts_none_named` | No note was named as a part of the book. Nothing was recorded (see `detail`). | **run**: Name the parts again as `{note, pages}`, in reading order. |  |  |
| `book_parts_out_of_order` | The parts were not named in reading order: their pages go back. Nothing was recorded (see `detail`). | **run**: Name the parts again as `{note, pages}`, in reading order. |  |  |
| `book_parts_repeated` | A note was named twice. Nothing was recorded (see `detail`). | **run**: Name the parts again as `{note, pages}`, in reading order. |  |  |
| `brain_keeper_interpretation_unavailable` | The brain keeper could not interpret the gaps; the prepared gaps are returned without it. | **tell_user**: Say the brain keeper's part is missing; the owner checks it. |  | `GET /brain_keeper/skills` |
| `brain_keeper_partial_gap_coverage` | The brain keeper interpreted only some of the gaps. | **retry**: Ask again for the gaps left out, or with fewer gaps. |  |  |
| `brain_keeper_unavailable` | No brain keeper is ready to write answers for this workspace (see `detail`): a written answer cannot be had; ranked items and reads work as before. | **tell_user**: Answer from the evidence and say a written answer is not available; setting up the brain keeper is the owner's. |  | `GET /agent/knowledge/capabilities` |
| `broken_characters_left` | Some characters the converter lost could not be put back (count in `detail`): they are counted in the lint of the note, and a passage with many is kept out of search by meaning. | **none**: Nothing to do; the original is unchanged. |  |  |
| `broken_glyphs_repaired` | Letter pairs (ff, fi, fl) that the converter lost were put back where the document itself spells the word (count in `detail`). | **none**: Nothing to do. |  |  |
| `changes_cursor_reset` | The cursor is not one this change log holds (the database was restored or replaced), so the changes are listed again from the oldest. | **none**: Treat changes already handled as repeats and keep the `next_cursor` this response gives. |  |  |
| `changes_position_invalid` | The cursor or the time given cannot be read. | **run**: Ask again without `cursor` for the newest changes and a fresh cursor, or give `since` as an ISO 8601 time. | `GET /agent/knowledge/changes` |  |
| `chapter_numbers_skip` | The chapter numbers found skip some. | **run**: Look at the split in a dry run (it reads, and spends nothing). Without a confident split the document is read whole, in parts (adding documents does this by itself); a split the user names is sent as `chapters`, never asked for. | `POST /sources/process_book` (path: {path}, dry_run: True) |  |
| `chapter_starts_not_monotonic` | The chapters found are out of order. | **run**: Look at the split in a dry run (it reads, and spends nothing). Without a confident split the document is read whole, in parts (adding documents does this by itself); a split the user names is sent as `chapters`, never asked for. | `POST /sources/process_book` (path: {path}, dry_run: True) |  |
| `chapters_look_like_sections` | The chapters found are short, like sections. | **run**: Look at the split in a dry run (it reads, and spends nothing). Without a confident split the document is read whole, in parts (adding documents does this by itself); a split the user names is sent as `chapters`, never asked for. | `POST /sources/process_book` (path: {path}, dry_run: True) |  |
| `cluster_labelling_unavailable` | The brain keeper could not label the clusters. | **tell_user**: Say the brain keeper's part is missing; the owner checks it. |  | `GET /brain_keeper/skills` |
| `code_area_unknown` | No code is of the area asked for (see `detail`); every code names its `area`. | **run**: Ask again with an area a code names, or without `area`. | `GET /agent/knowledge/codes` |  |
| `code_not_registered` | No warning or refusal code by that name is registered (see `detail`). | **run**: Check the spelling; asked without `code`, every registered code is listed. | `GET /agent/knowledge/codes` |  |
| `code_prefix_capped` | The start of a name asked for begins more codes than are listed (see `detail`). | **run**: Ask again with more of the name. |  |  |
| `collection_empty` | The collection holds no notes yet. | **tell_user**: Say it is empty and offer to add documents: ask which files or folder, plan the addition (`plan_adding` over MCP, a dry run of adding over HTTP), show the plan, and run it on the yes. |  | `GET /agent/knowledge/overview` |
| `collection_empty_read_only` | The collection holds no notes yet, and this agent may only read it. | **tell_user**: Say it is empty and that this agent only reads it: an agent the owner lets add documents adds them. |  | `GET /agent/knowledge/overview` |
| `collection_exists_there` | The folder already holds a collection. | **run**: Open it instead of making a new one there. | `POST /collections/open` (folder: {folder}) |  |
| `collection_folder_in_program` | The folder is inside the program's own folder (the checkout), where an update or a move of the program puts a collection at risk. | **tell_user**: Say so in its words, and ask for another folder or the default place. |  |  |
| `collection_folder_missing` | The answer was a folder the person names, but `folder` was not sent. | **run**: Send `folder`: the full path of the folder the person named or dragged into the chat. |  |  |
| `collection_folder_not_a_collection` | The folder holds no collection. | **tell_user**: Say so; to make a new collection there, make one with that folder. |  |  |
| `collection_folder_not_absolute` | The folder given is not a full path (see `detail`). | **run**: Send the folder in full, as the person dragged or named it. |  |  |
| `collection_folder_overlaps` | The folder is inside another collection, or holds one. | **tell_user**: Say so; a collection cannot sit inside another. Keep the default place or another folder. |  |  |
| `collection_folder_refused` | The folder cannot hold a collection (see `detail`). | **tell_user**: Say why in its words, and ask for another folder or keep the default place. |  |  |
| `collection_name_missing` | A collection is made with a name. | **run**: Send `name`: a few words the person calls it. |  |  |
| `collection_place_choice` | Nothing was made yet: where the collection goes is the person's answer (see `ask`): the folder the AI program works in, a folder they name, or the default place. | **ask_user**: Put `ask.text` to the person word for word, recommended answer first, each answer with the folder `ask.folders` names for it; on their answer, make the call `ask.on` names for it (for a folder they name, fill `folder`). Never answer it for them. |  |  |
| `collection_working_folder_missing` | The answer was the working folder, but `working_folder` was not sent or is not a folder on this computer. | **run**: Send `working_folder`: the full path of the folder you, the AI program, work in; never ask the person to type it. |  |  |
| `concept_layer_behind` | The concept notes lag their sources: some sources' concepts are not in them yet. | **run**: Bring the concept notes up to date. | `POST /concepts/sync` (wait: False) | `GET /concepts` |
| `config_or_field_problems` | The note's fields or its kind's settings have problems (see `problems`). | **tell_user**: Say which; the owner corrects the note or the setting. |  |  |
| `confirm_code_invalid` | The `confirm` sent is not a code a preview of this change gave. | **run**: Make the call again without `confirm`, show its sentence, then send the code it returns. |  |  |
| `confirm_code_stale` | The change, or what it replaces, moved since its preview, or the preview is older than an hour. | **run**: Preview the change again and show its new sentence. |  |  |
| `conflict` | The current state does not allow it (see `detail`). | **run**: Read the state `detail` describes, then decide. |  |  |
| `connect_same_agent` | The new connection would use this agent's own id, and so share its access. | **run**: Send `agent` with an id of its own for the other program, or leave it out for the program's name. |  |  |
| `conversation_unknown` | The conversation id is not one this workspace gave (see `detail`); a new conversation was started and its id is in `conversation.id`. | **run**: Send the new `conversation.id` from now on, or `new` to start one. |  |  |
| `conversion_config` | The workspace's conversion settings have a problem (in `detail`); defaults were used for it. | **tell_user**: Say so; the owner fixes config/conversion.json. |  | `GET /ops/self_check` |
| `conversion_failed` | The document converter failed on this file. | **tell_user**: Report the detail; the owner checks the converter. |  | `GET /ops/self_check` |
| `converter_not_supported` | The converter asked for is not one this install supports for this file. | **run**: Ask with a supported converter, or none to use the workspace's own. |  | `GET /ops/self_check` |
| `converter_refused` | The converter refused this file. | **tell_user**: Report the detail; do not process it again unchanged. |  |  |
| `corpus_cannot_answer_query` | The collection does not answer this question: every note reached was read. | **tell_user**: Say the collection does not answer it; a source on it is the remedy. |  |  |
| `coverage_incomplete` | The chapters found do not cover the whole book. | **run**: Look at the split in a dry run (it reads, and spends nothing). Without a confident split the document is read whole, in parts (adding documents does this by itself); a split the user names is sent as `chapters`, never asked for. | `POST /sources/process_book` (path: {path}, dry_run: True) |  |
| `docling_failed_using_basic_extractor` | docling failed on this PDF; the basic extractor was used, so tables, equations and figures were not recovered. | **run**: Re-read it with docling (dry run first); adding documents retries it by itself. | `POST /sources/reread` (source: {path}, dry_run: True) |  |
| `docling_unavailable_using_basic_extractor` | docling is selected but not installed; the basic extractor was used. | **tell_user**: Say so; the owner installs docling, then offer a re-read. |  | `GET /ops/self_check` |
| `document_converter_not_runnable` | No document converter can run here for this format. | **tell_user**: Say so; the owner installs markitdown. |  | `GET /ops/self_check` |
| `drift_explanation_unavailable` | The brain keeper could not explain the drift. | **tell_user**: Say the brain keeper's part is missing; the owner checks it. |  | `GET /brain_keeper/skills` |
| `duplicate_classification_unavailable` | The brain keeper could not classify the duplicates. | **tell_user**: Say the brain keeper's part is missing; the owner checks it. |  | `GET /brain_keeper/skills` |
| `empty_source` | The book holds no text. | **run**: Look at the split in a dry run (it reads, and spends nothing). Without a confident split the document is read whole, in parts (adding documents does this by itself); a split the user names is sent as `chapters`, never asked for. | `POST /sources/process_book` (path: {path}, dry_run: True) |  |
| `entity_ambiguous` | Another note or item also goes by that name; the card is the strongest match, and the best connected of equally strong ones. `detail` says what each matched by. | **run**: If the card is not the one meant, look up the alternative: the call names the first of `also_matches` by its id. | `POST /agent/knowledge/entity` (node_id: {node_id}) |  |
| `entity_not_found` | No note or item in the collection has that name. | **run**: Try one of `suggestions`, or ask the collection the question instead. | `POST /agent/knowledge/answer` (query: {name}) |  |
| `entity_types_not_held` | A kind asked for in `types` is not among the items this card lists; `detail` names the kinds it does list. | **run**: Ask again with kinds `detail` lists, or without `types`. |  |  |
| `equation_regions_not_located` | The equations' places on the page could not be found, so they cannot be cropped for vision. | **none**: Nothing to do; text methods still apply. |  |  |
| `estimate_exceeds_docling_timeout` | Converting this book is estimated to take longer than the docling timeout. | **run**: Process it: the time is part of the plan the budget covers (adding documents does this by itself). |  |  |
| `evidence_has_unread_regions` | The evidence holds equations or figures the brain has not read. | **run**: Read one only if the answer needs it: `unread_regions` gives each one's estimate and its own recovery call. |  | `POST /agent/knowledge/answer` (query: {query}) |
| `evidence_split_audit_not_adjudicated` | The brain keeper could not adjudicate the evidence split. | **tell_user**: Say the brain keeper's part is missing; the owner checks it. |  | `GET /brain_keeper/skills` |
| `example_not_found` | This install holds no example collection to open. | **tell_user**: Say so; the example ships with the download, so this install is missing a part. |  |  |
| `extraction_audit_unavailable` | The brain keeper could not audit the extraction. | **tell_user**: Say the brain keeper's part is missing; the owner checks it. |  | `GET /brain_keeper/skills` |
| `facet_coverage_low` | Few notes carry facets, so a facet filter sees little of the collection. | **run**: Ask again without `facets`; assigning facets is the owner's maintenance. | `POST /agent/knowledge/answer` (query: {query}) |  |
| `facet_filter_excluded_unfacetted_nodes` | The facet filter left out notes that carry no facets. | **run**: Ask again without `facets` if those notes may hold the answer. | `POST /agent/knowledge/answer` (query: {query}) |  |
| `facet_filter_matched_nothing` | No note carries the facet values asked for. | **run**: Ask again without `facets`, or with values the vocabulary holds. | `POST /agent/knowledge/answer` (query: {query}) | `GET /query/facets` |
| `facet_not_declared` | A facet asked for is not declared (see `unknown`). | **run**: Check the facet names. | `GET /ops/facets` |  |
| `facet_removal_blocked` | A facet was not removed: notes still carry it (see `blocked`). | **tell_user**: Say which notes still carry it; the owner re-facets them first. |  |  |
| `facets_managed_by_system` | Facets and the vocabulary are managed by the knowledge brain system itself; a caller agent does not write, propose or curate them. | **none**: Leave facets to the system. To have waiting values registered now, ask it to run its own promotion (a dry run first). | `POST /maintenance/vocabulary/promote` (dry_run: True) | `GET /ops/facets` |
| `facets_matched_nothing` | The facets matched no note. | **run**: Ask again with values the vocabulary holds. | `GET /query/facets` |  |
| `figure_vision_enabled_but_unavailable` | Figure vision is switched on but no vision model answers. | **tell_user**: Say so; the owner checks the brain keeper. |  | `GET /ops/self_check` |
| `folder_not_found` | No folder at that path in this workspace. | **run**: Check the path: relative to the workspace, or `root:path` for a declared source root. |  |  |
| `glyph_repair_could_not_be_written` | The repair of lost letter pairs could not be saved; the note keeps the text as the converter wrote it. | **report**: Report it. |  |  |
| `image_derived_statements` | Some statements were read from a figure's image, not its text. | **run**: Say they come from a figure, and check it before quoting a value. | `POST /agent/knowledge/read` (query: {query}, notes: {notes}) |  |
| `integrity_check_failed` | The store's integrity check failed. | **tell_user**: Report it; the owner runs the integrity check. |  | `GET /ops/status` |
| `invalid_request` | The request's arguments do not match the route (see `detail`). | **run**: Correct the arguments `detail` names. |  |  |
| `item_limit_capped` | More items were asked for than this answer lists (see `detail`). | **none**: The items listed are all this answer gives; ask a narrower question for others. |  |  |
| `job_interrupted` | The server running the job stopped before it finished (a restart or a crash); the job will not finish on its own. | **run**: Submit the same request again if it is still wanted; work already done is reused where it was kept. | `GET /brain_keeper/jobs` |  |
| `job_list_truncated` | The listing read its most recent records and stopped before the oldest; older jobs exist that it did not look at. | **run**: Narrow the listing by status, skill or submitted_by to reach older jobs. |  |  |
| `job_not_cancelled` | The job could not be cancelled (it had finished or is past the point of stopping). | **run**: Read its state. | `GET /brain_keeper/jobs/{job_id}` (wait: 30) |  |
| `job_not_found` | No job with that id exists here (or it belongs to another workspace). | **run**: List this workspace's recent jobs to find it again, by its skill or the source it is for. | `GET /brain_keeper/jobs` |  |
| `job_outcome_failed` | The job ran to its end, but its result says it failed (see outcome_reason). | **tell_user**: Tell the owner the reason; do not use the result. |  |  |
| `job_outcome_needs_review` | The job ran to its end, but its result holds something back for the owner to decide (see outcome_reason). | **tell_user**: Tell the owner what is held back and why; do not use the result as final. |  |  |
| `job_status_unavailable` | The job record could not be read within the status deadline; the job itself is unaffected. | **retry**: Read its state again in a few seconds; never resubmit it. | `GET /brain_keeper/jobs/{job_id}` (wait: 30) | `GET /brain_keeper/jobs/{job_id}` (wait: 30) |
| `job_still_running` | The job had not finished when the wait ended. | **wait**: Wait on it again; never resubmit it. | `GET /brain_keeper/jobs/{job_id}` (wait: 30) | `GET /brain_keeper/jobs/{job_id}` (wait: 30) |
| `kind_or_template_required` | The request names neither a kind nor a template. | **run**: Send `kind` (or `template_id`); the next call lists the kinds. | `GET /format-templates` |  |
| `knowledge_weak_context_rate_high` | Half or more of recent answers had weak context. | **tell_user**: Say the collection covers what is asked of it weakly. |  | `GET /ops/status` |
| `limit_out_of_range` | The `limit` asked for is below one (see `detail`); the answer lists its own number of items. | **none**: Nothing to do; send a `limit` of 1 or more to list fewer. |  |  |
| `lint_found_no_notes` | The check looked at no notes: the workspace named holds none yet. (A request naming no workspace is refused outright, not answered empty.) | **run**: Check that the workspace named is the one meant, and ask again; an empty answer here is not a clean bill of health. | `GET /maintenance/lint` |  |
| `low_provenance_density` | Few of the items trace to a source passage. | **run**: Read the sources before quoting: items without provenance are not evidence. | `POST /agent/knowledge/read` (query: {query}, notes: {notes}) |  |
| `low_query_alignment` | The items found share few of the question's words. | **run**: Read what `focus` names before relying on it; `warning_details` says what is missing. | `POST /agent/knowledge/read` (query: {query}, notes: {notes}) | `POST /agent/knowledge/answer` (query: {query}) |
| `low_retrieval_confidence` | The ranking is unsure of its best items. | **run**: Read the sources `focus` names before stating anything from them. | `POST /agent/knowledge/read` (query: {query}, notes: {notes}) |  |
| `low_source_quality` | The sources behind the items are weak (thin, heuristic or degraded notes). | **run**: Read the sources, and say the answer rests on weak ones. | `POST /agent/knowledge/read` (query: {query}, notes: {notes}) |  |
| `maintained_state_review_needed` | Some items rest on notes waiting for review. | **tell_user**: Say a review is pending; the owner settles it in the review queue. | `POST /review/queue` | `POST /review/queue` |
| `maintenance_write_required` | The call writes or spends, and the request is read-only. | **run**: Send it again with `access_mode: maintenance_write`; a dry run first, where the route has one. |  |  |
| `manual_fetch_failed` | The manual could not be fetched this time and nothing was changed. | **report**: Report it with the response it came in; try once more only if the person asks. |  |  |
| `manual_no_git` | The manual is fetched with `git`, which is not on this computer. | **tell_user**: Say so; everything else works, and installing git lets the manual be fetched. |  |  |
| `manual_no_network` | The manual could not be fetched: this computer cannot reach the internet. | **tell_user**: Say so in its words; nothing was changed, and asking again when online fetches it. |  |  |
| `manual_not_set` | This install does not say where the manual is kept (`config/help.json`). | **tell_user**: Say so; the maintainers publish the manual, and `config/help.json` of the install names where. |  |  |
| `markitdown_library_failed` | The markitdown library failed on this file. | **tell_user**: Report the detail. |  | `GET /ops/self_check` |
| `math_doubling_repaired` | Equations printed twice by the converter were repaired (count in `detail`). | **none**: Nothing to do. |  |  |
| `math_repair_could_not_be_written` | The equation repair could not be saved; the note keeps the converter's text. | **report**: Report it. |  |  |
| `mcp_config_invalid` | A value in the workspace's agent-connection settings could not be taken as written (see `detail`); the safe default is used in its place. | **tell_user**: Tell the owner which value `detail` names; correcting it is theirs. |  | `GET /agent/knowledge/capabilities` |
| `model_runtime_config_*` | The model runtime settings file is missing or not valid JSON. | **tell_user**: Say so; the owner fixes it. |  | `GET /ops/status` |
| `model_runtime_config_exception` | The model runtime settings could not be read. | **tell_user**: Report it. |  | `GET /ops/status` |
| `nightly_ai_answer_unknown` | The answer sent names no choice of the nightly AI question (see `detail`). | **run**: Send `answer` as one of: yes, no, light, medium, heavy. |  |  |
| `nightly_ai_choice` | The nightly tidy may also use the AI: whether it does, and how much at most a night, is the person's answer (see `ask`). | **ask_user**: Put `ask.text` to the person word for word, recommended answer first; on their answer, make the call `ask.on` names for it. Never answer it for them; unanswered, nothing uses the AI at night. |  |  |
| `no_bridge_signals` | No link between separate parts of the collection lies near this question. | **none**: Nothing to do. |  |  |
| `no_chapter_structure_detected` | No chapter structure was found in the book. | **run**: Look at the split in a dry run (it reads, and spends nothing). Without a confident split the document is read whole, in parts (adding documents does this by itself); a split the user names is sent as `chapters`, never asked for. | `POST /sources/process_book` (path: {path}, dry_run: True) |  |
| `no_gap_signals` | No known gap lies near this question. | **none**: Nothing to do. |  |  |
| `no_gaps_to_interpret` | No gap in this scope to interpret. | **none**: Nothing to do. |  |  |
| `no_heading_mapped` | None of the note's headings maps to a part of its kind's template. | **none**: Nothing to do; the note is kept as written. |  |  |
| `no_note_found` | None of the notes asked for is in this workspace. | **run**: Ask the question to find the notes that hold it, then read those. | `POST /agent/knowledge/answer` (query: {query}) |  |
| `no_pdf_converter_selected_using_basic_extractor` | No PDF converter is selected; the basic extractor was used, so tables, equations and figures were not recovered. | **none**: Nothing to do unless the source has tables or equations; the owner can select a converter in config/conversion.json. |  | `GET /ops/self_check` |
| `no_readable_text` | The source converts to no words (a page drawn by script, a scan). | **tell_user**: Report it; do not process it again unchanged. |  |  |
| `no_regions` | The source holds no equation regions to read (`how_to_get` says how to find them). | **run**: Follow `how_to_get`. |  |  |
| `no_strong_knowledge_items` | No item ranked strongly for this question. | **tell_user**: Say nothing in the collection answers this strongly; sources on it are the remedy. |  | `POST /agent/knowledge/answer` (query: {query}) |
| `no_text_layer` | The PDF has no text layer: its pages are images. | **tell_user**: Say so; the owner runs OCR on it, then processes it again. |  |  |
| `no_transcript` | The video has no caption track that could be read. | **run**: Take it in from its description and chapters only; the note says it holds no more. | `POST /sources/process` (url: {url}, description_only: True) |  |
| `no_vocabulary_registered` | No facet vocabulary is registered for this workspace. | **none**: Nothing to do: the system forms the vocabulary from the proposals of the documents added. |  | `GET /ops/facets` |
| `normalization_config_*` | The normalization settings file is missing or not valid JSON. | **tell_user**: Say so; the owner fixes it. |  | `GET /ops/status` |
| `normalization_config_exception` | The normalization settings could not be read. | **tell_user**: Report it. |  | `GET /ops/status` |
| `normalization_fallback_rate_high` | Half or more of recent notes were built without an agent. | **tell_user**: Say so; the owner checks the brain keeper. |  | `GET /ops/status` |
| `normalization_slow_rate_high` | Half or more of recent notes were slow to build. | **tell_user**: Say so. |  | `GET /ops/status` |
| `normalized_note_renamed` | The note was named differently from its source, because that name belongs to another source. | **run**: Use the `normalized_path` returned, not one built from the source's name. |  |  |
| `not_a_fetchable_url` | Only http and https addresses are fetched. | **run**: Send an http or https address. |  |  |
| `not_a_video_link` | The link names no video or playlist. | **run**: Check the link; a watch, short, live, embed or playlist link is a video link. |  |  |
| `not_found` | What the request names does not exist here (see `detail`). | **run**: Check the name, path or id, and the workspace. |  |  |
| `not_permitted` | The access asked for is not granted (see `detail`). | **tell_user**: Say so; widening access is the owner's decision. |  |  |
| `note_not_found` | No note at that path in this workspace. | **run**: Use the note path a processing result returned. |  |  |
| `note_type_not_held` | The collection holds no note of that kind; `detail` lists the kinds it holds. | **run**: Ask again with a kind `holdings.kinds` names, or without `note_type`. | `GET /agent/knowledge/overview` |  |
| `oc_markitdown_failed_using_internal_pdf_markdown_fallback` | markitdown failed on this PDF; the internal fallback was used. | **none**: Nothing to do; check the note if the source is complex. |  |  |
| `one_chapter_holds_most_of_the_book` | One chapter holds most of the book. | **run**: Look at the split in a dry run (it reads, and spends nothing). Without a confident split the document is read whole, in parts (adding documents does this by itself); a split the user names is sent as `chapters`, never asked for. | `POST /sources/process_book` (path: {path}, dry_run: True) |  |
| `owner_title_not_on_page` | A title the owner gave is not printed on its page; the part starts at the top of that page. | **run**: Look at the split in a dry run (it reads, and spends nothing). Without a confident split the document is read whole, in parts (adding documents does this by itself); a split the user names is sent as `chapters`, never asked for. | `POST /sources/process_book` (path: {path}, dry_run: True) |  |
| `pages_not_applicable` | Pages select within a PDF; this source has none. | **run**: Ask without `pages`. |  |  |
| `pages_not_covered` | The parts proposed leave pages out, between them or after the last. | **run**: Look at the split in a dry run (it reads, and spends nothing). Without a confident split the document is read whole, in parts (adding documents does this by itself); a split the user names is sent as `chapters`, never asked for. | `POST /sources/process_book` (path: {path}, dry_run: True) |  |
| `part_beyond_document` | A part proposed starts past the end of the document. | **run**: Look at the split in a dry run (it reads, and spends nothing). Without a confident split the document is read whole, in parts (adding documents does this by itself); a split the user names is sent as `chapters`, never asked for. | `POST /sources/process_book` (path: {path}, dry_run: True) |  |
| `part_ends_before_it_starts` | A part proposed ends before it starts. | **run**: Look at the split in a dry run (it reads, and spends nothing). Without a confident split the document is read whole, in parts (adding documents does this by itself); a split the user names is sent as `chapters`, never asked for. | `POST /sources/process_book` (path: {path}, dry_run: True) |  |
| `partial_result_set` | Fewer items were found than were asked for. | **run**: Ask again with a larger `limit` if more is needed. | `POST /agent/knowledge/answer` (query: {query}, limit: 10) | `POST /agent/knowledge/answer` (query: {query}) |
| `parts_out_of_order` | The parts proposed are not in page order. | **run**: Look at the split in a dry run (it reads, and spends nothing). Without a confident split the document is read whole, in parts (adding documents does this by itself); a split the user names is sent as `chapters`, never asked for. | `POST /sources/process_book` (path: {path}, dry_run: True) |  |
| `parts_overlap` | Two parts proposed share pages. | **run**: Look at the split in a dry run (it reads, and spends nothing). Without a confident split the document is read whole, in parts (adding documents does this by itself); a split the user names is sent as `chapters`, never asked for. | `POST /sources/process_book` (path: {path}, dry_run: True) |  |
| `parts_too_few` | Fewer than two parts were proposed. | **run**: Look at the split in a dry run (it reads, and spends nothing). Without a confident split the document is read whole, in parts (adding documents does this by itself); a split the user names is sent as `chapters`, never asked for. | `POST /sources/process_book` (path: {path}, dry_run: True) |  |
| `path_not_allowed` | The path is outside the workspace and every declared source root. | **tell_user**: Use a path inside the workspace, or ask the owner to declare a source root for it. |  |  |
| `path_required` | The request names no path. | **run**: Send `path`. |  |  |
| `pdf_converter_not_supported` | The selected PDF converter is not one this version supports. | **tell_user**: Say so; the owner selects a supported one in config/conversion.json. |  | `GET /ops/self_check` |
| `pdf_converter_unavailable` | The selected PDF converter is not installed. | **tell_user**: Say so; the owner installs it, or selects another in config/conversion.json. |  | `GET /ops/self_check` |
| `playlist_unreachable` | The video site could not be reached or did not answer as expected. | **retry**: Try again later; if it persists, the owner updates `provider` in config/video.json. |  |  |
| `priority_summary_failed` | The priority summary could not be built. | **tell_user**: Report it. |  | `GET /ops/status` |
| `processing_failed` | Processing stopped with an error. | **report**: Report the detail; do not resubmit unchanged. |  | `GET /ops/self_check` |
| `processing_in_background` | The work continues in the background. | **wait**: Wait on the job with one waiting call rather than polling. | `GET /brain_keeper/jobs/{job_id}` (wait: 30) | `GET /brain_keeper/jobs/{job_id}` (wait: 30) |
| `program_connection_failed` | The agent program refused to add the connection (see `detail`). | **tell_user**: Say so with its reason; nothing of the collection changed. |  |  |
| `program_entry_differs` | The agent program already has a connection by that name that starts something else; it was left as it is. | **tell_user**: Say so; nothing was changed, and that connection is the person's to remove or keep. |  |  |
| `program_not_found` | That agent program is not installed on this computer. | **tell_user**: Say so; once it is installed, connect it again. |  |  |
| `program_not_started` | The agent program could not be started to add the connection (see `detail`). | **tell_user**: Say so in its words: check that the program is installed and starts; then connect again. |  |  |
| `program_settings_unreadable` | The agent program's settings file cannot be read; it was left as it is. | **tell_user**: Say so; its owner repairs that file, then connect again. |  |  |
| `program_timed_out` | The agent program was still adding the connection when the wait ran out (see `detail`). | **tell_user**: Say so in its words; ask again: a connection already added is said so. |  |  |
| `program_unknown` | The agent program named is not one a connection can be added to (see `detail`). | **run**: Send one of the programs `detail` lists. |  |  |
| `proposal_already_accepted` | The same statement was accepted before: it is a note of the collection. | **none**: Nothing to do; recall finds it like any note. |  |  |
| `proposal_already_dismissed` | The owner dismissed the same proposal before; it is not raised again. | **tell_user**: Say the owner decided against it; do not file it again. |  |  |
| `proposal_already_in_collection` | The collection already states this statement (see `in_collection`: the item and its notes); nothing was filed. | **none**: Nothing to do; recall finds it, and `detail` names the item. |  |  |
| `proposal_already_queued` | The same proposal already waits for review; this filing was added to it (who proposed it and what it rests on). | **none**: Nothing to do; it waits for the owner's review. |  | `GET /agent/knowledge/review_queue` (kind: {kind}) |
| `proposal_already_withdrawn` | What this proposes to forget was withdrawn already. | **none**: Nothing to do; it is no longer in the collection. |  |  |
| `proposal_decided_by_its_proposer` | The agent that proposed this item cannot decide it (see `detail`). | **tell_user**: Say a proposal waits for the owner's decision; another identity, the owner's, decides it. |  |  |
| `proposal_hidden_characters` | The proposal holds characters a reviewer cannot see, control characters or a hidden instruction (see `detail`); nothing was filed. | **run**: Send the same text as plain visible characters. |  |  |
| `proposal_incomplete` | The proposal lacks what its kind needs (see `detail`): a statement to remember, or the note or item and a reason to forget. | **run**: Send it again with what `detail` names. |  |  |
| `proposal_queued` | The proposal is a review item now, waiting for the owner: nothing was written, and until the owner accepts it nothing recalls it. | **tell_user**: Say it waits for the owner's review; do not state it as part of the collection until it is accepted. |  | `GET /agent/knowledge/review_queue` (kind: {kind}) |
| `proposal_target_ambiguous` | Several notes go by the name given (see `detail`). | **run**: Name one by its path, as `detail` lists them. |  |  |
| `proposal_target_not_evidence` | What was named is not part of the collection (a concept page, an idea or the owner's work, kept by its own upkeep), or is stated in none of its notes. | **tell_user**: Say so; changing it is the owner's. |  |  |
| `proposal_target_not_found` | No note or item of this workspace goes by the id given (see `detail`). | **run**: Use a note path or an item id as an answer or a card gives it. | `GET /agent/knowledge/overview` |  |
| `proposal_too_long` | A field of the proposal is longer than it may be (see `detail`). | **run**: Shorten what `detail` names: one statement per proposal. |  |  |
| `proposals_not_allowed` | This workspace's owner does not take proposals (see `detail`). | **tell_user**: Say so; allowing proposals is the owner's choice, in the workspace's settings. |  |  |
| `provenance_not_evidence` | The note named as provenance is not one of the collection's (the owner's work, an idea or a concept page is never evidence). | **run**: Cite the note of the collection the statement rests on. |  |  |
| `provenance_not_found` | What the provenance names is not in this workspace, or names several notes (see `detail`). | **run**: Name the note by its path as an answer or the overview gives it, or the source by its workspace path. | `GET /agent/knowledge/overview` |  |
| `provenance_quote_not_found` | The quote given as provenance is not in the note or source as written. | **run**: Quote it exactly as written there, or leave the quote out. |  |  |
| `provenance_required` | A statement to remember needs what it rests on: a note of the collection, a source file in the workspace, or the conversation it came from. | **run**: Send it again with provenance: `note` (its path or key), `source` (a file under sources/), or `conversation` (where it came from, in a few words). |  |  |
| `quarantined_sources_held` | Some sources are held in quarantine: they hold instructions addressed to an AI reader. | **tell_user**: Say so; only the owner releases one, after reading it. |  |  |
| `question_not_covered` | The evidence holds too little of the question's own words. | **tell_user**: Say which subject the collection lacks: `question_terms.absent` lists the words found nowhere in it; the overview shows what it does hold. | `GET /agent/knowledge/overview` | `POST /agent/knowledge/answer` (query: {query}) |
| `read_budget_adjusted` | The `max_chars` asked for is outside 2,000 to 200,000 (see `detail`); the read used the nearest bound. | **none**: Nothing to do; send a budget within the bounds. |  |  |
| `read_left_out_matching_sections` | The read stopped at its size limit; sections that match were left out. | **run**: Read again with a larger `max_chars`, or fewer notes. | `POST /agent/knowledge/read` (query: {query}, notes: {notes}, max_chars: 40000) |  |
| `read_next_nothing_matched` | Nothing in the collection is tied to the question: no item holds its words or is reached by them, so nothing is suggested to read. | **run**: See what the collection holds, then ask with its own words. | `GET /agent/knowledge/overview` |  |
| `read_not_permitted` | The read was refused by the access policy. | **tell_user**: Say the access asked for is not granted; reads are `read_only`. |  |  |
| `read_note_not_found` | Some notes asked for are not in this workspace (see `not_found`). | **run**: Ask the question to find the notes that hold it. | `POST /agent/knowledge/answer` (query: {query}) |  |
| `read_sections_not_found` | No section by some of the names asked for, in the note or its source (see `detail`); the read was chosen for the question instead. | **run**: Ask again with names from the `headings` of the source. |  |  |
| `read_truncated` | Some sections were cut to fit `max_chars` (see `detail`). | **run**: Ask again for them by heading, with a larger `max_chars`. |  |  |
| `refacet_unavailable` | The brain keeper could not assign facets. | **tell_user**: Say the brain keeper's part is missing; the owner checks it. |  | `GET /brain_keeper/skills` |
| `reference_list_mostly_prose` | The source was taken in as a reference list (a glossary, a lookup list) but most of its lines are not entries (share in `detail`); its other words are kept as written in `About this list`, and no agent read it. | **run**: If it reads as prose rather than a list, process it again with its own kind, which an agent reads (the next call lists the kinds). | `GET /format-templates` |  |
| `released_from_quarantine` | The source was released from quarantine and will be read. | **none**: Nothing to do. |  |  |
| `relevance_judgement_unavailable` | The brain keeper could not judge relevance; the answer is complete without that second opinion. | **tell_user**: Say the judgement is missing; the owner checks the brain keeper. |  | `GET /brain_keeper/skills` |
| `rename_already_rolled_back` | That rename was undone already. | **none**: Nothing to do. |  |  |
| `rename_changed_since` | A note changed since the rename; undoing it would discard that edit (see `detail`). | **run**: Leave the rename in place; undo it with `force: true` only when that edit may go. |  | `GET /rename/log` |
| `rename_draft_claimed_twice` | More than one note names this draft as theirs, and none is named after it, so it is not moved for any of them (see `detail.notes`). | **run**: Rename the note itself by its path; its draft stays where it is. |  |  |
| `rename_failed_undone` | The rename failed part way and was undone; nothing of it remains. | **report**: Report the detail; the names are as they were. |  | `GET /rename/log` |
| `rename_kind_unknown` | The kind is not one a rename knows. | **run**: Send `kind`: original, note, concept or converted. |  |  |
| `rename_links_left_in_owner_notes` | Some notes the owner maintains still link the old name: they are never rewritten by the platform. | **tell_user**: Say which of the owner's notes still use the old name; the owner updates them. |  |  |
| `rename_name_invalid` | The new name is not one a file can have on every system (see `detail`). | **run**: Choose another name: plain letters, digits, dots and hyphens are always safe. |  |  |
| `rename_name_taken` | Another file already has that name; a rename never overwrites one. | **run**: Send the rename again with `detail.free_name`, the free name proposed. |  |  |
| `rename_not_found` | Nothing of that kind and name is in this workspace. | **run**: Check the kind and the name: a note by its path or file name, a concept by its key. |  |  |
| `rename_original_not_recorded` | The file named is no original of this collection: no note records it, and it is outside the raw folder and every declared source root. | **run**: Check the path against a note's `raw_source_path`; a file nothing here records is renamed where it is, not through the collection. |  |  |
| `rename_recovered_unfinished` | A rename cut off part way earlier was found and undone first. | **none**: Nothing to do; the names are as they were before it. |  |  |
| `rename_snapshot_not_found` | No rename snapshot of that id can be undone here. | **run**: Use the `snapshot_id` a rename returned; the rename log lists them. |  | `GET /rename/log` |
| `rename_would_overwrite_original` | Another file of the user's has the new name of this original: the rename would replace it. | **ask_user**: Put the question to the user, recommending the free name in `detail.free_name`; only a yes sends it again with `overwrite: true` (the other file is set aside in the snapshot, not deleted). |  |  |
| `repair_metadata_unavailable` | The brain keeper could not repair the metadata. | **tell_user**: Say the brain keeper's part is missing; the owner checks it. |  | `GET /brain_keeper/skills` |
| `request_field_unknown` | The request sent a field this route does not take, one that looks like a misspelling of a field it does take or on a route that takes no others (see `detail.fields`); nothing was done. | **run**: Send it again with the field named in `did_you_mean`, or without the field. |  |  |
| `request_fields_ignored` | The request sent fields this route does not take; it acted without them (see `detail`). | **run**: Check the field names: if one of them mattered, send the request again with the name the route takes. |  |  |
| `request_refused` | The request was refused as asked (see `detail`). | **run**: Correct what `detail` names and send it again. |  |  |
| `restore_rehearsal_failed` | The last restore rehearsal failed. | **tell_user**: Report it. |  | `GET /ops/status` |
| `restore_rehearsal_missing` | No restore has been rehearsed. | **tell_user**: Say so; the owner rehearses one. |  | `GET /ops/status` |
| `rests_on_unresolved` | An idea rests on a note that is not in the collection. | **tell_user**: Say which; the link resolves once that note exists. |  |  |
| `review_kind_not_open` | No open review item is of the kind asked for (see `detail`); `by_kind` lists the kinds that are. | **run**: Ask again with a kind `by_kind` lists, or without `kind`. | `GET /agent/knowledge/review_queue` |  |
| `scheduled_jobs_need_attention` | Some scheduled jobs are failing. | **tell_user**: Report which; the scheduled jobs' status names them. |  | `GET /ops/status` |
| `schema_status_error` | The schema status could not be read. | **tell_user**: Report it. |  | `GET /ops/status` |
| `scope_argument_ignored` | An argument naming the workspace, the agent, the scope or the access was ignored (see `detail`): this connection fixes them when it starts. | **none**: Nothing to do; another workspace is reached through the server started for it. |  |  |
| `scope_matched_nothing` | Nothing in the source the question was scoped to matches it (see `detail`): the answer keeps to that source, so it lists no items. The collection may still hold the subject elsewhere. | **run**: Ask again without `scope` to search the whole collection, or read the scoped source itself. | `POST /agent/knowledge/answer` (query: {query}) |  |
| `scope_not_found` | The source the question was scoped to names no note of this workspace (see `detail`); the answer is not scoped. | **run**: Name the source by a note path or a citekey the collection holds, or ask without `scope`. | `GET /agent/knowledge/overview` |  |
| `scope_not_ingested` | The source the question was scoped to is a note of this workspace that is not in the graph yet (see `detail`): written but not ingested, so the answer can list nothing from it. | **tell_user**: Tell the owner the note needs ingesting (an intake or maintenance step, not a read); meanwhile read it directly, or ask without `scope`. |  |  |
| `section_not_found` | No section by that name in the source. | **run**: Use a heading from the source's own sections. |  |  |
| `semantic_expansion_unavailable` | The question could not be mapped onto the facet vocabulary; the answer is valid, possibly narrower. | **none**: Nothing to do for this answer; recall may be narrower than the collection. |  | `GET /ops/semantic_expansion` |
| `semantic_reach_unavailable_or_empty` | Meaning-based reach contributed nothing. | **tell_user**: Say recall may be narrower. |  | `GET /ops/embeddings` |
| `semantic_search_disabled` | Semantic search is switched off in this install's embedding settings. | **tell_user**: Say recall may be narrower; `semantic_search.how_to_enable` has the steps. |  | `GET /ops/embeddings` |
| `semantic_search_endpoint_not_local` | The embedding endpoint is not a local one, and only a local one is allowed. | **tell_user**: Say recall may be narrower; `semantic_search.how_to_enable` has the steps. |  | `GET /ops/embeddings` |
| `semantic_search_no_match` | Meaning-based search ran and found nothing that means what was asked. | **none**: Nothing to fix: it is a fact about the collection. |  |  |
| `semantic_search_no_model` | No embedding model is configured. | **tell_user**: Say recall may be narrower; `semantic_search.how_to_enable` has the steps. |  | `GET /ops/embeddings` |
| `semantic_search_no_profile` | The configured embedding model has no profile. | **tell_user**: Say recall may be narrower; `semantic_search.how_to_enable` has the steps. |  | `GET /ops/embeddings` |
| `semantic_search_not_configured` | This workspace was built without an embedding service. | **tell_user**: Say recall may be narrower; `semantic_search.how_to_enable` has the steps. |  | `GET /ops/embeddings` |
| `semantic_search_not_indexed` | The passages are not indexed for the configured model. | **run**: Index the notes missing their vectors: it is free, and the nightly run also does it. | `POST /ops/embeddings/reindex` (missing: True) | `GET /ops/embeddings` |
| `semantic_search_not_requested` | The caller asked for no semantic reach. | **none**: Nothing to do: it was asked for. |  |  |
| `semantic_search_over_cap` | More passages than the configured cap would be scored. | **tell_user**: Say recall may be narrower; `semantic_search.how_to_enable` has the steps. |  | `GET /ops/embeddings` |
| `semantic_search_server_unreachable` | The embedding server did not answer; it is started or pointed to by the owner. | **tell_user**: Say recall may be narrower; `semantic_search.how_to_enable` has the steps. |  | `GET /ops/embeddings` |
| `semantic_search_stale_format` | The stored vectors are in an old format. | **run**: Index every note again in the current format: it is free, and unchanged passages are reused. | `POST /ops/embeddings/reindex` (all: True) | `GET /ops/embeddings` |
| `semantic_search_unavailable` | Meaning-based search did not contribute: the answer drew on words alone. | **tell_user**: Say recall may be narrower; `semantic_search.how_to_enable` names the setting and steps (the owner's to change). |  | `GET /ops/embeddings` |
| `semantic_search_unknown` | Semantic search did not contribute, for a reason not reported. | **tell_user**: Say recall may be narrower; `semantic_search.how_to_enable` has the steps. |  | `GET /ops/embeddings` |
| `server_error` | The server failed on this request. | **report**: Report it with the request; try once more only if it was not a write. |  | `GET /health` |
| `setting_example_no_ai` | The example collection never uses the AI at night, so it has no nightly AI level to change. | **tell_user**: Say so; the level is chosen in a collection of the person's own. |  |  |
| `setting_needs_program` | That choice needs a program installed on this computer first (see `detail`). | **tell_user**: Say so; install the program the plan named (its `install`), then change the choice again. | `GET /collections/setup` |  |
| `setting_not_changed_here` | That choice is not changed by this call (see `detail` for how it is). | **tell_user**: Say how it is changed, as `detail` says. |  |  |
| `setting_unknown` | No choice of the catalogue has that name (see `detail`: the closest names, and every choice this call changes). | **run**: Send the choice by one of the names `detail` gives; `check_setup` lists them too (`choices`). |  |  |
| `setting_value_invalid` | The value or words sent do not fit the choice (see `detail`). | **run**: Send them as `detail` says. |  |  |
| `setup_needs_local_connection` | A setup call came over a connection that is not on the owner's own machine. | **tell_user**: Say so; setup is done from a connection the agent program starts on the owner's own computer. |  |  |
| `source_missing` | Evidence rests on a source whose original file is gone. | **tell_user**: Say so; the owner restores the file or settles it through reconciliation. | `POST /sources/reconcile` (dry_run: True) | `POST /sources/reconcile` (dry_run: True) |
| `source_not_found` | No file at that path in this workspace. | **run**: Check the path: relative to the workspace, or `root:path` for a declared source root. |  |  |
| `source_not_text` | The source is not a text file. | **run**: Name its text note instead. |  |  |
| `source_quarantined` | The source hides instructions addressed to an AI reader: it is held in quarantine, not read. | **tell_user**: Say so; only the owner releases it, after reading it. | `GET /ops/quarantine` |  |
| `spending_approval_expired` | The approval is older than its validity (see `detail`). | **run**: Make the dry run again and run with its new approval. |  |  |
| `spending_approval_mismatch` | The approval was given for another call: another route, other arguments, a file that changed since, or another budget. | **run**: Make the dry run again with the arguments of the real call, and run with its new approval. |  |  |
| `spending_approval_required` | A real run of a route that spends tokens needs the approval its dry run gives, and this collection's setting asks for it (see `detail`). | **run**: Make the same call as a dry run (`dry_run` true), show its plan to the person, and on their yes send the real call with the `approval` the dry run gave, every other argument unchanged. |  |  |
| `stale_knowledge_candidate` | Some items come from notes that may be out of date with their sources. | **run**: Inspect the sources the items cite (read-only). | `POST /maintenance/inspect` (source_paths: {source_paths}) | `POST /review/queue` |
| `surface_not_allowed` | The tool or surface asked for is beyond what this connection may reach (see `detail`, which says who can allow it and how). | **tell_user**: Carry on with the tools listed; a wider surface is the owner's to allow. |  |  |
| `surface_not_offered` | The surface asked for does not exist (see `detail`); `surfaces` lists those that do. | **run**: Ask for one of the surfaces `surfaces` lists, or carry on with the tools listed. |  |  |
| `synthesis_unavailable` | The written answer could not be produced; the evidence is complete. | **tell_user**: Answer from the evidence; the owner checks the brain keeper. |  | `GET /brain_keeper/skills` |
| `table_repair_could_not_be_written` | The table repair could not be saved; the note keeps the converter's text. | **report**: Report it. |  |  |
| `table_spacing_repaired` | Tables whose cells the converter ran together were repaired (count in `detail`). | **none**: Nothing to do. |  |  |
| `table_split_across_batches` | A table too large for one batch was split, its header repeated. | **none**: Nothing to do; check that table's facts if one matters. |  |  |
| `tails_off` | Tails are switched off for this instance; an approved tail is not used. | **tell_user**: Say so; the owner turns tails on in config/tails.json. |  |  |
| `thin_chapters` | Some chapters found are very short. | **run**: Look at the split in a dry run (it reads, and spends nothing). Without a confident split the document is read whole, in parts (adding documents does this by itself); a split the user names is sent as `chapters`, never asked for. | `POST /sources/process_book` (path: {path}, dry_run: True) |  |
| `title_not_found` | A part's title was not found where it is said to start. | **run**: Look at the split in a dry run (it reads, and spends nothing). Without a confident split the document is read whole, in parts (adding documents does this by itself); a split the user names is sent as `chapters`, never asked for. | `POST /sources/process_book` (path: {path}, dry_run: True) |  |
| `toc_entries_not_found` | Entries of the table of contents were not found in the text. | **run**: Look at the split in a dry run (it reads, and spends nothing). Without a confident split the document is read whole, in parts (adding documents does this by itself); a split the user names is sent as `chapters`, never asked for. | `POST /sources/process_book` (path: {path}, dry_run: True) |  |
| `toc_sections_without_chapter` | Sections of the table of contents fall under no chapter. | **run**: Look at the split in a dry run (it reads, and spends nothing). Without a confident split the document is read whole, in parts (adding documents does this by itself); a split the user names is sent as `chapters`, never asked for. | `POST /sources/process_book` (path: {path}, dry_run: True) |  |
| `tool_not_offered` | No tool by that name is offered on this connection. | **run**: List the tools and call one that is listed. |  |  |
| `tool_outside_surface` | The tool is on a wider surface than this session is on (see `detail`); this connection may widen to it. | **run**: Widen this session with `request_tools` to the surface `detail` names, then call the tool. |  |  |
| `topic_matched_nothing` | The topic matched no note. | **run**: Ask again with other words for the topic, or with facets. |  |  |
| `tour_sample_added` | The tour's waiting page is already in the example. | **run**: Ask the example about it, or put the example back with `reset` to add it again. |  |  |
| `tour_sample_not_ready` | The tour's waiting page could not be planned (see `detail`). | **report**: Report it with the response it came in. |  |  |
| `tour_step_unknown` | The tour has no step of that name or number (see `detail`). | **run**: Send one of the steps `detail` lists, or no step for the first. |  |  |
| `unclassified_refusal` | A refusal this version gives no code. | **report**: Report it to the maintainer with the response it came in. |  |  |
| `unknown_equation_ids` | Some equation ids asked for do not exist in the source. | **run**: Use ids from the dry run. |  |  |
| `unresolved_boundaries` | Some chapter boundaries could not be settled. | **run**: Look at the split in a dry run (it reads, and spends nothing). Without a confident split the document is read whole, in parts (adding documents does this by itself); a split the user names is sent as `chapters`, never asked for. | `POST /sources/process_book` (path: {path}, dry_run: True) |  |
| `unsupported_binary` | A file format nothing here reads. | **tell_user**: Say so; the owner exports it to PDF, Word, HTML or Markdown. |  |  |
| `unsupported_image` | An image format nothing here reads. | **tell_user**: Say so; the owner converts it to PNG or JPEG. |  |  |
| `unsupported_source` | A source of a kind this route does not read (see `detail`). | **tell_user**: Say so; do not resubmit it unchanged. |  |  |
| `unverified_values` | A value in the evidence was read from text whose layout was lost, or the source prints it in another row. | **run**: Check the value in the original before quoting it (each item's `unverified` says where). | `POST /agent/knowledge/read` (query: {query}, notes: {notes}) |  |
| `urls_required` | The request names no video or playlist. | **run**: Send `urls`. |  |  |
| `video_not_public` | The video is private, members-only or otherwise needs a signed-in session. | **tell_user**: Say so; only public videos are read. |  |  |
| `video_page_unreachable` | The video site could not be reached or did not answer as expected. | **retry**: Try again later; if it persists, the owner updates `provider` in config/video.json. |  |  |
| `video_player_unreachable` | The video site could not be reached or did not answer as expected. | **retry**: Try again later; if it persists, the owner updates `provider` in config/video.json. |  |  |
| `vision_unavailable` | No vision model can read figures or equations here. | **tell_user**: Say so; the owner enables vision for the brain keeper. |  | `GET /ops/self_check` |
| `vocabulary_curation_unavailable` | The brain keeper could not propose vocabulary changes; the report is complete without them. | **tell_user**: Say the brain keeper's part is missing; the owner checks it. |  | `GET /brain_keeper/skills` |
| `weak_context` | The collection covers this question weakly: few strong, well-sourced items. | **tell_user**: Say the collection covers this weakly; sources on the subject are the remedy. |  | `POST /agent/knowledge/answer` (query: {query}) |
| `weak_freshness_coverage` | How fresh some items are is not known. | **none**: Nothing to do; freshness is unknown, not stale. |  |  |
| `withdrawn_note_kept_out` | The note made from this source was withdrawn in review; the note made again was kept aside beside the withdrawal and not ingested (see `withdrawn`). | **tell_user**: Say the owner withdrew it; restoring it is the owner's, from the withdrawal `withdrawn` names. |  |  |
| `withdrawn_statement_kept_out` | A statement withdrawn in review came back when this note was made again, and was taken out again (see `withdrawn`). | **none**: Nothing to do; the owner restores a withdrawal when they want it back. |  |  |
| `withdrawn_statement_present` | A statement withdrawn in review is in this note as its owner keeps it; the note was ingested as written, and a review item asks the owner to settle it (see `withdrawn`). | **tell_user**: Say a withdrawn statement is back in the note; releasing the withdrawal or removing the line is the owner's. |  |  |
| `workspace_not_named` | The request does not say which workspace it is for. | **run**: Send `workspace_id` (and `agent_id`) with every call: the list names the registered ones. Over a connection with no collection, open one by its folder or make one instead. | `GET /ops/workspaces` |  |
| `workspace_not_registered` | The workspace named is not registered here. | **run**: Use a registered workspace id: the list names them. A collection whose folder is on this computer but not registered is opened by its folder (`open_collection`). | `GET /ops/workspaces` |  |
| `write_tools_need_grant` | This surface adds no tools here: its tools need a grant this connection does not hold (see `detail`). | **tell_user**: Carry on with the tools listed; proposals are the owner's to allow, with the launch that starts the server. |  |  |
<!-- codes:end -->

## Limitations

- **Coverage is what the suite exercises.** A code emitted only on a path no
  test reaches is not checked by strict mode; in service it arrives with
  `action: report`, which says so.
- **Refusals outside the governed and intake routes** keep FastAPI's shape
  unless their route gives them a code; their warnings do carry notices.
- **A fix is a step, not a guarantee.** `verify` says how to see the
  result; many codes describe the collection (`weak_context`,
  `corpus_cannot_answer_query`), where the remedy is a source the owner adds,
  and no call resolves them.
- Calls are filled from the request's own arguments; what the response
  itself knows best (the notes `focus` names) is left in `fill` rather than
  guessed.
