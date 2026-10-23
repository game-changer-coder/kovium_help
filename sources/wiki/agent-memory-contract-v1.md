---
type: "documentation"
title: "Agent Knowledge Contract V1"
description: "Legacy filename note:"
status: "stable"
reading_status: "read"
product: "second brain"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing second brain"
doc_path: "docs/agent_memory_contract_v1.md"
doc_sha256: "9ef6d52514ad9927891dd267e115c9399be77fd8e2920f699fa932e6fb0e0fb7"
---

# Agent Knowledge Contract V1

Legacy filename note:

This file still lives at `docs/agent_memory_contract_v1.md` for compatibility with older internal references, but the public API and contract naming now use `knowledge`.

## Purpose
Define the stable agent-facing contract for `second_brain` knowledge use.

The goal is to let downstream agents consume knowledge through one coherent interface rather than manually combining raw retrieval, graph, and gap endpoints.

---

## Contract principles
- stable JSON schema
- task-oriented packaging
- bounded context by depth mode
- explicit provenance
- explicit relevance reasoning
- explicit warnings when knowledge is weak or incomplete
- explicit schema and contract version metadata

---

## Versioning

Current contract:

- `contract_version`: `agent_knowledge_contract_v2`
- `schema_version`: `agent_knowledge_response_v2`

Every agent-facing knowledge response must include both fields at the top level and inside the returned `contract` object.

### What changed in v2: paths are workspace-relative

From `agent_knowledge_response_v2` on, **every path in a governed answer is
relative to the workspace root**, written with `/` on every host. This covers
`source_refs[].source_path`, `sections[].source_path` from
`/query/resolve_sections`, any `path` or `wiki_path`, and any absolute path a
synthesized or summarised text happened to contain. It applies to
`/agent/knowledge/answer`, `/task_context`, `/read_next` and
`/query/resolve_sections`. No answer reveals the machine's folders.

- **Resolving a path:** call `GET /wiki/page?path=<path>` with the same
  `agent_id`/`workspace_id`. It accepts exactly this form. Alternatively, join
  the path to a workspace root the caller already knows.
- **Outside the workspace:** a note kept outside the workspace (allowed by
  source retention) is given checkout-relative, or as its bare file name.
- **What a v1 caller sees:** the same fields with relative values. A caller
  that opened `source_path` directly as an absolute file must now resolve it
  as above. This is why the schema version changed. There is no compatibility
  mode returning absolute paths: that is the disclosure this version removes.
- `GET /agent/knowledge/contract` reports the rule under `paths` (`form`,
  `separator`, `resolve_with`, `since_schema_version`).

v2 also adds, additively:
- `query_decomposition`: how a long question (14 words or more) was split
  into short searches and fused, or absent when it was ranked whole:
  `parts` (each a clause of the question, words in order), `fused`,
  `fusion`, `rank_basis` (`fused_rank`) and `timing_ms` (per run, `whole`
  and each part). Present on `/answer` and `/task_context`.
- When `query_decomposition.fused` is true, every knowledge item carries
  `fused_from` (the runs that found it: `whole` and/or part texts),
  `fused_rank` (its place, 1-based) and `rank_basis: "fused_rank"`. Items are
  in `fused_rank` order, not `score` order: the whole question's own best
  results (all but the last quarter of places) keep their order and places,
  and the remaining places go to what the parts found best. A part's hit can
  sit below an item with a lower score of its own. `fused_score` is the
  reciprocal-rank sum, for inspection.
- `semantic_search.how_to_enable`: when semantic search did not contribute,
  the remedy for its code, built from the install's own settings (file, keys,
  current values, steps).
- `warning_details`: one sentence per warning that has a remedy, keyed by
  warning code.
- `notices` (on the envelope, beside `data`; Track AC, 2026-10-15): one entry
  per warning code in the response, with its next step as data
  (`fix.next.action`, the call filled with this request's arguments,
  `fix.verify`, `fix.consent`). A refusal carries `error` in the same shape
  beside its `detail`. Codes and fixes: `GET /agent/knowledge/codes`,
  `docs/next_steps_v1.md`.

### Compatibility policy

Additive changes are allowed within `agent_knowledge_response_v2` when they:

- add optional fields
- add warning values to `warning_taxonomy`
- add confidence fields while preserving existing field meanings
- increase internal explanation detail without changing required field names

Breaking changes require a new schema version when they:

- remove or rename existing top-level fields
- remove or rename fields inside `contract`, `confidence`, `knowledge_items`, or item-level `confidence`
- change depth-mode limit semantics in a way that can increase response size unexpectedly
- change warning meanings in a way that reverses downstream interpretation
- change provenance shape or remove provenance from knowledge items

Deprecations must be documented here before removal. During a deprecation window, the old field should remain present and a warning or compatibility note should describe the replacement.

### Required envelope fields

The current required top-level fields are:

- `schema_version`
- `contract_version`
- `response_type`
- `contract`

The response-specific required fields are:

- `knowledge_answer`: `caller`, `scope`, `policy`, `query`, `answer_summary`, `knowledge_items`, `read_next`, `gap_signals`, `bridge_signals`, `confidence`, `source_quality_summary`, `warnings`
- `task_context`: `caller`, `scope`, `policy`, `task`, `query`, `task_brief`, `knowledge_items`, `answer_summary`, `read_next`, `gap_signals`, `bridge_signals`, `confidence`, `source_quality_summary`, `warnings`. The request takes `query`, `task`, or both: each defaults to the other, and a request with neither is refused with 422 ("give a query or a task"). It also carries the answer's additive `semantic_search`, `query_decomposition`, `warning_details` and `unread_regions`.
- `read_next`: `caller`, `scope`, `policy`, `query`, `suggestions`, `warnings`. Track AB3: a suggestion is listed only when something ties it to the question (its label holds the question's words, its chapter does, or notes the question reaches name it), and its `reason` says which ("Its label holds the question's words: levain, hydration."); an item ranked on its type, provenance, centrality or bridging alone is not suggested. An answer's and a task context's `read_next` follow the same rule. With nothing tied, `suggestions` is empty and the warning `read_next_nothing_matched` points at the overview

### Caller identity requirements

Agent-knowledge read endpoints require caller identity so knowledge access is no longer anonymous and can later map cleanly to per-agent sandbox routing.

Current request identity fields:

- required: `agent_id`
- optional: `workspace_id`

Current contract metadata exposes these expectations through `contract.identity_requirements`.

### Scope requirements

Agent-knowledge read endpoints also expose explicit logical scope metadata so the current access boundary model can later map cleanly to per-agent sandbox routing.

Current request scope fields:

- optional: `scope_kind`
- optional: `scope_id`

Current scope behavior:

- default `scope_kind` is `agent`
- supported scope kinds are `agent` and `workspace`
- `workspace_id` is required when `scope_kind` is `workspace`
- if `scope_id` is omitted, the service resolves a stable default scope id from the selected scope kind

Current contract metadata exposes these expectations through `contract.scope_requirements`.

### Access policy requirements

The current agent-knowledge read surface is explicitly governed as read-only by default so downstream agents do not implicitly gain write-capable behavior.

Current request access fields:

- optional: `access_mode`

Current access behavior:

- default `access_mode` is `read_only`
- every read route accepts only `read_only`
- one narrower mode, `propose` (Track AB4), is accepted by
  `POST /agent/knowledge/propose` alone, where it is required and has no
  default: it may file a review item for the owner and nothing else (its
  `policy`: `allowed_actions` `read_knowledge`, `propose_changes`;
  `denied_actions` `write_knowledge`, `resolve_review`, `refresh_knowledge`,
  `admin_override`; `resolution_basis: propose_only`). Every other route
  refuses it (422) before anything happens, and the propose route refuses any
  other mode; a test walks every route's request model to keep it so
- requests for unsupported access modes are rejected
- every successful response includes an explicit resolved `policy` object

Current contract metadata exposes these expectations through `contract.access_requirements`.

### Governance stability guarantees

The current governance model is intended to survive the later move to per-agent workspaces without requiring a breaking contract redesign.

Current contract metadata exposes these guarantees through `contract.governance_stability`.

Those guarantees mean:

- `caller`, `scope`, and `policy` remain required governed response fields
- bounded response behavior continues to vary by depth `mode`, not by whether routing later resolves to a per-agent workspace
- future per-agent workspace routing must resolve through the existing caller/scope model rather than replacing it with incompatible fields
- governed reads are expected to emit audit events that include caller, scope, policy, action, and outcome

This is the intended compatibility rule for the current governance model:

- add workspace routing behind the current governed fields
- preserve existing response limits and governed field names
- treat routing and filtering changes as implementation evolution, not as justification for breaking the contract

---

## Depth modes

### short
Use for:
- quick orientation
- short answer drafting
- narrow follow-up questions

### medium
Use for:
- normal reasoning
- planning
- answer synthesis

### deep
Use for:
- research synthesis
- multi-part planning
- broader exploration or hypothesis-building

---

## Primary endpoints

### `GET /agent/knowledge/contract?mode=`
Returns the active contract details for the selected depth mode. `warning_taxonomy_is` (Track AB3) says the taxonomy is partial: the warnings ranking at that depth raises, not every code an answer can carry (those are in `GET /agent/knowledge/codes`).

### `POST /agent/knowledge/answer`
Returns the primary memory answer bundle for a query.

Required request fields:
- `agent_id`
- `query`

Optional request fields:
- `workspace_id`
- `scope_kind`
- `scope_id`
- `access_mode`
- `node_types`
- `facets` — scope the answer to the closed facet vocabulary, e.g.
  `{"topic": ["edge-computing"], "locus": ["edge"]}`. AND across facet keys, OR
  within one key's values.
- `limit` (Track AB2): at most this many items listed, up to the mode's
  own `item_limit` (short 3, medium 5, deep 8); omitted, the mode's. Never
  refused, as before: more than the answer lists is warned
  (`item_limit_capped`) and lists that many, below one is warned
  (`limit_out_of_range`) and lists the mode's number; the ranking
  underneath, and the confidence measured on it, are the mode's either way.
  Before AB2 it did not change the items listed. The task context and the
  reading list take it and do not use it.

Said rather than dropped (Track AB2): a `scope` naming no note of the
workspace warns `scope_not_found` (its `detail` is what was named; `focus`
falls back to the evidence as before), and a `conversation_id` the workspace
never gave warns `conversation_unknown` (a new conversation is started, its id
in `conversation.id`). The answer's summary points at `focus.read`, the
one-call read, rather than naming an HTTP route.

#### `scope`: only this source (AB2c, 2026-10-17)

`scope` (`{"note": …}`, `{"citekey": …}` or `{"identifier": …}`) is a
**filter**, on `/agent/knowledge/answer` and `/agent/knowledge/task_context`
alike. Before AB2c it only named the focus: the items still came from the
whole collection, so an answer could read `single_source`, `decided_by:
caller`, share 0 and zero items while every item it listed came from other
sources. An agent reading the word expects the filter, so the filter it is.

- Only what rests on the scoped source is ranked, packed, suggested and
  bridged: the source's note, every item it mentions (an entity two notes
  share is one node, and each note keeps its own `MENTIONS` edge), and every
  item whose provenance names it — the same test `focus` uses for where
  evidence rests. `knowledge_items`, `read_next` and `bridge_signals` respect
  it; `gap_signals` and `concept_notes` stay collection-wide, as with facets.
- `source_filter` says what it kept: `notes` (the scoped note path),
  `matched` (items ranked in it), `excluded_outside` (ranked items left out)
  and `items` (listed). It is `null` on an unscoped answer, and on a scope
  that names no note (`scope_not_found`, which filters nothing).
- Every item scores something from its kind and place alone, so a source
  holding nothing of the question would still list its own unrelated items.
  When the scoped items hold **less than `scope.min_information_share`** of
  the question's information — each of its words weighed by log(notes /
  notes holding it), the anchors' measure, with no credit for words found
  elsewhere in the collection, since inside a scope elsewhere is outside it —
  and no meaning-based match vouches for them, the answer lists **no** items
  and warns `scope_matched_nothing:<note>`, whose fix is the same question
  without `scope`. The share is set per workspace in `config/retrieval.json`
  (`scope.min_information_share`, default one third). **It is a judgement
  fitted to a small sample, not a measured constant**: ten scoped questions
  on three installs, where sources with nothing on the subject held 0.05 to
  0.19 of the information, partial but real matches 0.46 and 0.49 and
  covering ones 1.0. Lower lists a source's unrelated items more often;
  higher empties sources that answer part of the question. The synthetic
  test `tests/fixtures/focus_and_scope_collections.py` (`SCOPE_LINE`) pins a
  covering, a partial (about 0.4) and a non-covering source. (The first
  measure tried, `question_not_covered`'s rarity-weighted coverage, gives
  words found elsewhere half credit, and one ordinary word of the question
  that the source held lifted a source with nothing on the subject over its
  floor: 0.58 to 0.66 measured.) The warnings that judge
  items (`weak_context`, `no_strong_knowledge_items`, `question_not_covered`,
  `low_query_alignment`, `partial_result_set`, `low_retrieval_confidence`,
  `low_provenance_density`, `low_source_quality`) give way to it: there are
  no items to judge, and the cause is the scope, not the collection. The
  confidence agrees with the warnings: `overall_confidence` 0.0,
  `weak_context` and `question_not_covered` false, and
  `scope_matched_nothing: true`.
- **Every scoped item is located in the scoped source.** Its provenance is
  the item's own references that name the source, else the source's own
  `MENTIONS` reference to it (a clause two versions of a document share, a
  concept the source names), marked `scope_located_by: mention`; `where` is
  never another note.
- A scope naming a note the wiki holds but the graph does not (written, not
  ingested yet) keeps the answer to it, so it lists nothing, and warns
  `scope_not_ingested:<note>` (action `tell_user`: its owner ingests it, an
  intake or maintenance step no read route offers), rather than that the
  source holds nothing.
- `focus` keeps `single_source`, `decided_by: caller`, and says which case it
  is in `why`; its `read` still reads the scoped source.
- **Contract version.** Not bumped: under the compatibility policy above no
  field is removed or renamed, the response can only shrink, no warning
  reverses its meaning and the provenance shape is unchanged. What changed is
  what a scope selects; a caller that sent `scope` for a label only now gets
  that source's items alone, and `source_filter` shows it.

#### Facet-scoped answers
**A node with no facets does not match a facet-scoped query.** It is excluded,
not silently kept.

This was previously the other way round: the filter read
`if node_facets and not match: continue`, so an un-facetted node skipped the
check entirely. A query filtering on a value that exists nowhere in the corpus
returned the corpus unfiltered and looked like it had worked — measured on a
real workspace, 50 of 50 requested results for a topic value appearing nowhere.

Facets are declared once, in a note's frontmatter. **Its claims, concepts and
questions inherit them**, so a facet-scoped query is not limited to `Note`
nodes — in `edgesage` that would have been 4 of 114. The inheritance is derived
per query rather than written onto the entity nodes, because entities are shared
between notes and whichever note was ingested last would otherwise overwrite the
others' facets.

The whole answer respects the scope — `knowledge_items`, `read_next` and the
focused bundle, not only the ranking. An answer that ranked on facets but packed
content from the whole corpus would be worse than not filtering, because it
would look scoped and not be.

A scoped response carries `facet_filter`:

```json
{
  "requested": {"topic": ["edge-computing"]},
  "matched": 88,
  "excluded_unfacetted": 12,
  "excluded_facet_mismatch": 26,
  "coverage": 0.9,
  "warnings": ["facet_filter_excluded_unfacetted_nodes"]
}
```

It is `null` on an unscoped query. The counts exist because an empty result has
two very different causes — *nothing carries that value* versus *most of this
corpus has not been facetted yet* — and they call for opposite responses from
the caller: widen the query, or go and facet the corpus. That difference is not
visible from the result list alone.

Warnings, all declared in the taxonomy for every depth mode:
- `facet_filter_matched_nothing` — the filter excluded everything.
- `facet_filter_excluded_unfacetted_nodes` — some nodes were dropped purely for
  having no facets.
- `facet_coverage_low` — more than half of what was considered had no facets, so
  the result is drawn from a small and unrepresentative slice.

`GET /query/facets` reports which values exist and a `coverage` block saying how
many notes carry facets at all. Use it to form a valid query rather than
guessing, and to explain a thin result.

### `POST /agent/knowledge/task_context`
Returns a task-oriented memory package for a specific task and query.

Required request fields:
- `agent_id`
- `query`
- `task`

Optional request fields:
- `workspace_id`
- `scope_kind`
- `scope_id`
- `access_mode`

### `POST /agent/knowledge/read_next`
Returns recommended next reading items for a query.

Required request fields:
- `agent_id`
- `query`

Optional request fields:
- `workspace_id`
- `scope_kind`
- `scope_id`
- `access_mode`

### `POST /agent/knowledge/read` (Track Q2, 2026-10-11)
Reads the relevant parts of one source or several in one call, as compact
Markdown under a budget; `response_type: knowledge_read`. Read-only, confined
to the workspace and audited (`action: knowledge_read`). The answer's
`focus.read.body` is a valid request as given. See "Answers in fewer steps"
below.

Required request fields:
- `agent_id`
- `notes`

Optional request fields:
- `workspace_id`, `scope_kind`, `scope_id`, `access_mode`
- `query`
- `max_chars`: default 32,000; a value outside 2,000 to 200,000 (0 included) is read at
  the nearest bound and warned (`read_budget_adjusted`), never refused
  (Track AB2; before, a smaller value was raised to 2,000 in silence)
- `view` (`compact`, the default for this route, or `full`)

The header (Track AB6). Each note's Markdown opens with its header,
`## [note] Header — <note>#frontmatter:<lines>`, one `- name: value` line per
field: the note's frontmatter less the platform's bookkeeping
(`docs/note_format_contract_v1.md`, "The header"), its type's declared fields
first, each value at most 200 characters, the fields that did not fit named on
a last line. When the note is read from a source document that has a
frontmatter of its own (a Markdown file taken in whole, a converted
transcript), the fields of it the note does not repeat follow as
`## [source] Header — <source>#frontmatter:<lines>`: each block names the file
it was read from. Before, the read cut a source's frontmatter off with nothing
in its place.

Its bound (AB6 fix round, R69 and C64): both blocks together, heading lines
included, at most a sixth of this note's share of the budget and never over
1,200 characters, with no floor, so several notes read together keep the
budget (six at 2,000 characters had used 3,433). With other sections named, the
header comes only when it fits in 300 characters. A header that does not fit
whole is not shown in part: it is the first entry of the source's `left_out`
(`heading_path: "Header"`, with its `fields` count) and a `- Header (N fields)`
line under "Left out" while the budget allows. Asked for by name (`sections`:
`Header` or `frontmatter`), it comes, within half the note's share. The header
is paid from the note's share before anything else. `view: full` adds `header`
and `source_header` per source: `origin` (`note` or `source`), `path` (the file
it was read from), `shown`, `fields[]` (`name`, `value`, `lines`, `cut`),
`left_out`, `cut`, `lines` and `where`; null when there is none.

Named sections (Track AB2). When a note entry names `sections` and any of them
is found (in the note, or in the source it reads), the note's named sections
come first, in the order asked, whole when they fit, and its summary is left
out. The rest of the budget is filled from the source as always: whole when
it fits (nothing is guessed), else its named sections first and then its best
matches for the question. When the whole read is over budget, the source's
passages give way first (the last moves to "Left out"), the named sections
last. A name found nowhere
warns `read_sections_not_found` (its `detail` names it per note; the source
carries `sections_not_found` and `headings`, up to 20 to ask for); with no
name found, the read is chosen for the question as before. A section cut to
fit the budget warns `read_truncated`, naming it. The Markdown names the
owner's work and the ideas attached to a note by their note paths, and a cut
section says to ask for it by heading with a larger `max_chars`; it names no
HTTP route.

### `GET /agent/knowledge/changes` (Track AB0, 2026-10-16)
What changed in the collection since the caller last looked;
`response_type: knowledge_changes`. Read-only, audited
(`action: knowledge_changes`), no model call.

Each entry is one note added, changed, removed or renamed, oldest first:
`{cursor, change, note_id, title, path, note_type, at, seeded?}`. `change` is
`added`, `changed` (its content changed; a note written again unchanged is
not a change, so a rebuild of the graph reports nothing), `removed` or
`renamed` (Track AP5, additive): the same note under a new name, through the
rename engine (`POST /rename`, or a rename made by hand and carried through by
`POST /rename/reconcile`), with `previous_note_id` and `previous_path`, the id
and path it had, so a caller's saved references follow it. A note's id
follows its path, so a renamed note's `note_id` is new; a rename is never
listed as one removal and one addition. See `docs/renames_v1.md`. `note_type` is the note's own type, so a caller can tell the
owner's ideas and own work from the collection. Notes only: the items a note
mentions change with it.

The response also carries `next_cursor` (send it back as `cursor`),
`has_more`, `returned`, `position` (`cursor`, `since` or `newest`),
`delivery: at_least_once` and `log`: `started_at`, `seeded_notes`,
`removals_before_start_recorded` and `last_change_at`.

- **The cursor is opaque.** It names the last change handed over and a
  digest of it, so a cursor this log no longer holds (a database restored from
  a backup or replaced) is recognised: the changes are listed again from the
  oldest with the warning `changes_cursor_reset`, never skipped. A caller
  treats changes it already handled as repeats.
- **Without `cursor` or `since`**: the newest `limit` changes, and a cursor
  to continue from. With both, `cursor` wins.
- **A store older than the log** was seeded once with each note it held,
  listed as `added` at its last update with `seeded: true`; a note removed
  before then left no entry (`removals_before_start_recorded: false`).
- The log is in the workspace's database, written in the same transaction as
  the change, so the HTTP service and an MCP server writing one workspace at
  once share one unbroken sequence. Backups carry it.

Query parameters:
- `agent_id` (required), `workspace_id`, `scope_kind`, `scope_id`, `access_mode`
- `cursor`: a `next_cursor` (or an entry's own `cursor`) from an earlier call
- `since`: an ISO 8601 time; changes at or after it (a time without a zone is UTC)
- `limit`: 1 to 500, default 50

Refused with 422 and `changes_position_invalid` when `cursor` or `since`
cannot be read; the fix is the same call without them.

### `POST /agent/knowledge/entity` (Track AB0, 2026-10-16)
One card for a name: what it is, where it comes from and what it is connected
to; `response_type: knowledge_entity`. Read-only, audited
(`action: knowledge_entity`), no model call. A miss is not an error.

The name is resolved to a note or to any item of the graph (a concept, a
definition, a claim, whatever types an install's notes produce), compared
without case, punctuation or hyphens, in this order of strength: an exact id,
a note's path or key, its citekey or a declared identifier, its title, a short
name it defines for itself (`identity_terms`), an item's label, a name a
concept's page gives it (`aliases`), the part of a note's title before a colon
or dash. An exact label ranks above an alias: an alias is a second name given
to an idea, and it can be another thing's own name (a page about the planet
Mercury giving it the alias "Mercury", which is also the element's label).
Only when none of those holds (Track AB2b, 2026-10-17): an abbreviation a
label or title glosses ("Mobile Edge Computing (MEC)" for `MEC`), then the
same words up to a plural ending, and, for an abbreviation, what the
collection itself writes it out as ("power distribution unit (PDU)", or "PDU
(power distribution unit)"), resolved as a name in turn. Each says so in
`matched_by`; the contract's `matched_by` lists every value it takes, as
templates (`…` a quoted name, `{how}` one of `HOW_KINDS` in
`second_brain/agent_memory/entity.py`, `{any}` any other value), and a test
checks the route writes no other.

When the name also names another note or item, by any name kind (title,
short name, label, alias, title head) or by an equally strong weaker match,
the card is the strongest match, the best connected among equally strong
ones, and the warning `entity_ambiguous:<detail>` says what each matched by
and whether the other was as strong (with fewer connections) or a weaker kind
of match. Its fix is the entity call for the first of `also_matches`, by its
`node_id` (over MCP, the `entity` tool call).

What the weaker matches assume (`second_brain/agent_memory/names.py`):
words are compared whole, never as letters inside a word; characters are
compared only when the name asked for is itself written without spaces, so a
title mixing scripts still holds its Latin words whole. A plural is folded only by the
Latin-script `s` ending (agents/agent, processes/process, strategies/strategy,
GPUs/GPU), only when the shorter form has at least four letters, never in
another script, never for a word ending in `ss`, `us` or `is` and never for an
all-capital abbreviation. A language that forms plurals otherwise (German
`-en`) gets no match from it, never a wrong one. An abbreviation's long form is
accepted only when its letters are found in order, the first opening the first
word (the Schwartz-Hearst test); nothing is guessed from initials, and no list
of abbreviations exists. The facet vocabulary's value definitions are read for
long forms too.

**One idea, one card.** A concept with a concept note (the page the platform
keeps for it, `docs/concept_notes_v1.md`) is one entity: the card is the
`Concept` item's, with the page attached. The item is primary because it
exists whenever any note names the idea and keeps its id whether a page is
kept, merged or redirected; the page is a maintained, rebuildable view of it.
Asked by the page's title, alias or id, the same card comes back
(`matched_by: the title of its page`, `the id of its page`). Neither appears
as the other's neighbour or in `also_matches`. The page's own links join the
card, except two kinds that would only repeat it: its naming links to what the
concept already reaches, and its `sources:` links (`draws_on`) to notes
already listed as naming the concept. Every other typed link to or from the
page (a note building on it, the owner's commentary on it) is listed, so both
ends of a link agree.

Response: `asked`, `found`, `matched_by`, `entity`, `also_matches` (up to
five other matches, each with `matched_by`), `suggestions` (for a miss: up
to five close names), `warnings`; for a missed abbreviation the collection
writes out, `written_out` (its long forms, at most three). Suggestions are
names only: note titles, short names, a page's aliases, and items whose label
is short (at most 80 characters and ten words) and does not end like a
sentence. They hold the asked words as whole words (or the asked name holds
theirs), or are spelt almost like it (difflib ratio 0.8 or more, among names
beginning with the same letter), notes and concepts first; a concept's page is
offered as its concept. The names are indexed by word once per change of the
notes or of any node (the store's `change_signature`), so a miss reads only
the names that could match: on 100,000 synthetic names a repeated miss took
0.05 to 0.08 s (2.8 to 3.2 s before the index), the first one 3.4 s.

The card (`entity`):
- `id`, `type`, `label`, `title`, `created_at`, `updated_at`;
- every edge once, in one of three lists (Track AB2b):
  - `relations`: typed links, `outgoing` and `incoming`, each entry
    `relation` plus the other item (`id`, `type`, `label`, a note's
    `note_type` and `path`); a relation declared in `config/relations.json`
    by its name, read from the far end by its inverse (with `stored_as`, and
    `evidence` when the note quoted one); the platform's own links by their
    edge type in lower case (`broader_than`: from the broader concept,
    `draws_on`: a concept's page to its sources, `rests_on`, `idea_about`,
    `work_about`, `comments_on`, ...). A link to something not yet ingested
    is `resolved: false`;
  - `referenced_by`: the notes that name it;
  - `neighbours`: what it names, and anything else linked by naming. They
    carry no `via` (AB0's edge types per neighbour): every neighbour is linked
    by naming, so it would say `MENTIONS` on every entry, and a typed link is
    under `relations` with its name;
- each list a selection, not a sorted cut: its items grouped (neighbours by
  item type, referring notes by `note_type`, relations by direction and
  relation; both directions share one `limit`), the biggest group first, one
  from each group in turn; within a group the most connected first, then in
  the order its note lists them. A label in a list is cut at 120 characters
  (the id stays exact);
- `left_out` (only when something was): what this page did not list, by group
  (`neighbours`, `referenced_by`, `relations.outgoing` / `.incoming`), with
  `next_offset`, the `offset` that lists more. One offset pages every list
  together, so a list already exhausted comes back empty on a later page; to
  page one list, narrow with `types`;
- `counts`, always of everything: `neighbours`, `neighbours_by_type`,
  `referenced_by`, `referenced_by_note_types` (the referring notes by their
  own type, Track AB2), `relations`, `relations_by_name` (`outgoing`,
  `incoming`), `relations_by_note_type` (`outgoing`, `incoming`: the notes
  among the relations by their own type, which is where AB2's signal for an
  idea resting on it, a page drawing on it or the owner's work on it now is);
- `provenance`: up to five source references, paths workspace-relative;
- for a note: `note_type`, `path`, `summary`, `facets`, `names` (`citekey`,
  `identifiers`, `short_names`), `header` (Track AB6: `where`, the note and
  the lines of its frontmatter, and `fields`, `{name: value}`, its
  frontmatter less the platform's bookkeeping and less the `facets` already
  listed, each value at most 200 characters, at most 1,200 characters and 24
  fields in all; `left_out` and `cut` name what did not fit; absent when the
  note has none) and `read`, the `/agent/knowledge/read` call for it;
- for another item: `defined_as`, the definitions written as "term: meaning"
  whose term is its name; for a concept with a page, also `page` (`id`,
  `path`, `note_type`), `summary` (the page's), `names.aliases` and `read`
  (the page).

A `summary` is the note's `description` as plain prose (Track AB2b): Markdown
list markers, rules, headings, emphasis, code marks and links are removed, a
list run onto one line becomes its items joined by the punctuation they end
with (or a semicolon), at most 500 characters, cut at a sentence end where one
is near. The note is never changed. The overview's `description` is cleaned
the same way.

Changed by Track AB2b, for a caller of the AB0 or AB2 card (breaking):
- a note pointing at this one by a typed link (an idea's `rests_on`, a concept
  page's `draws_on`, the owner's `comments_on`) is under `relations.incoming`,
  no longer under `referenced_by`;
- `neighbours` no longer lists referring notes or typed links, and its
  entries no longer carry `via`;
- `counts.neighbours` and `neighbours_by_type` no longer count referring notes
  (they were counted as `Note` neighbours) or typed links;
- `counts.referenced_by_note_types` no longer counts the ideas, pages and the
  owner's work linked by a typed edge; `counts.relations_by_note_type` does;
- a concept with a page is the `Concept`'s card, not the page's;
- `entity_ambiguous` now carries a detail (`entity_ambiguous:<detail>`) and
  is raised whenever another thing goes by the name, not only on a tie;
- the field `limit` covers both directions of `relations` together.

Request fields:
- `agent_id` (required), `workspace_id`, `scope_kind`, `scope_id`, `access_mode`
- `name`: what to look up (a path, key, citekey, identifier, title, short
  name, alias or label)
- `node_id`: an exact id from an earlier card or answer; wins over `name`
- `limit`: 1 to 50, default 10: items listed per list (relations both ways
  together)
- `offset`: 0 or more, default 0: items of each list to skip; the card's
  `next_offset`
- `types`: up to 20 kinds; only connected items of these kinds are listed
  (an item `type` as `neighbours_by_type` names it, or a note's
  `note_type`), compared without case; the counts stay whole. A kind none of
  the card's items has is the warning `entity_types_not_held:<detail>`, the
  detail naming the kinds the card does list

One of `name` or `node_id` is required (422 `invalid_request` otherwise).
`entity_not_found` carries the answer call for the name as its fix.

### `GET /agent/knowledge/overview` (Track AB2, 2026-10-17)
What the collection holds, for a caller that does not know yet;
`response_type: knowledge_overview`. Read-only, audited
(`action: knowledge_overview`), no model call, no ranking.

A newcomer cannot ask a good question of a collection it has never seen: a
question such as "what is in here" ranks nothing (its words are in no note),
and `entity` needs a name first. The overview reads the collection's own
shape, never its subject:

- `holdings`: `notes` (the collection's notes by their `type`, most first),
  `notes_total`, `concepts` (the concept notes the platform keeps; a merged
  concept's redirect is not counted), `owner_work` and `ideas` (the owner's,
  never evidence), and `kinds`: every `type` the notes carry, of any tier,
  with its count, each a value `note_type` takes (an install's ideas carry
  their own type names, so `ideas` alone does not say what to pass). The same
  count the MCP server's instructions give.
- `notes`: the collection's notes by title, paged, each with `note` (its
  path), `title`, `type`, `description` (its own one-line `description`
  frontmatter field as plain prose (Markdown list, rule and emphasis marks
  removed, Track AB2b), whitespace collapsed, at most 160 characters; `null`
  when it has none) and `updated_at`. With `note_type`, the notes of that
  one kind instead, of any tier (concept notes, ideas or the owner's work
  included).
- `returned`, `total`, `has_more`, `next_offset` (send it back as `offset`).
- `concepts` (first page only, `offset` 0; the same on every page): up to
  20 concepts named by most of the collection's notes, in the notes' own
  words: `id`, `label`, `collection_notes_naming_it` (the collection's notes
  that name it; concept notes, ideas and the owner's work are not counted,
  which is why an entity card's `referenced_by` for the same concept can be
  larger).

Warnings: `collection_empty` (no note yet; its words say how documents get in
today, for an agent that may add them), `collection_empty_read_only` (the same
for an agent that only reads: another one the person lets add documents adds
them; Track AG2 fix round) and
`note_type_not_held` (no note of that kind; its `detail` lists the kinds held,
its fix is the overview without it). `question_not_covered` on an answer now points here as its next call:
when the collection does not hold the subject, the overview shows what it
does hold.

Query parameters:
- `agent_id` (required), `workspace_id`, `scope_kind`, `scope_id`, `access_mode`
- `note_type`: only the notes of this kind, as `holdings.notes` names them
- `limit`: 1 to 500, default 25
- `offset`: 0 or more, default 0: skip this many notes, as `next_offset` gives

The query parameters of this route and of `GET /agent/knowledge/changes` are
declared as one model each (`KnowledgeOverviewQuery`, `KnowledgeChangesQuery`
in `second_brain/api/app.py`), with a description on every field, so `/docs`
and the MCP tools over them read the same text (Track AB2); the request models
of the answer, task context, reading list and read routes carry their
descriptions the same way. The parameters and the answers did not change; a
missing `agent_id` is still refused with 422, its `input` now the parameters
given.

### `GET /agent/knowledge/capabilities` (Track AB3, 2026-10-17)
What this workspace can do for a caller, and what is not ready;
`response_type: knowledge_capabilities`. Read-only, audited
(`action: knowledge_capabilities`), no model call: the brain keeper is asked
whether it is ready, which checks its command line and sign-in (cached),
never a model.

- `workspace`: `id`, and `display_name` when its owner gave one in the
  workspace's `config/mcp.json`;
- `access`: `modes`, the access modes the governed routes accept
  (`read_only`);
- `brain_keeper`: `available` (whether a written answer can be had),
  `backend` (the one that would write it, `deterministic` when none), and
  when not available `reason` (each backend's own reason) and
  `known_backends`;
- `ceiling`: what the owner allows an agent connection, from the workspace's
  `config/mcp.json`: `max_surface` (`default`, `wider` or `write`; `default`
  when the file or the key is absent), `allow_proposals` (false when absent),
  `set_in` (that file) and `set_in_is` (`workspace-relative`: under the
  workspace's own folder, its `root_path` in the instance's
  `config/workspace_registry.json`, by default `sandboxes/<workspace-id>`
  under the instance root; not the instance root's `config/`), `configured`
  (whether the file exists), and
  `problems` when a value could not be taken as written (the safe default is
  used in its place). See `docs/mcp_server_v1.md`, "Surfaces".

Warnings, each with its fix: `brain_keeper_unavailable` (its `detail` the
reason; the remedy is the owner's; checked by this route again) and
`mcp_config_invalid` (one per problem). Both are conditions of the install
(`about: install`).

The MCP server adds `connection` to this report (its surface, cap, grants,
transport, tools and launch flags), which no route can know; over HTTP it is
absent.

Query parameters: `agent_id` (required), `workspace_id`, `scope_kind`,
`scope_id`, `access_mode` (`KnowledgeCapabilitiesQuery`).

### `GET /agent/knowledge/review_queue` (Track AB3, 2026-10-17)
The review queue's open items, as a caller agent reads them;
`response_type: knowledge_review_queue`. Read-only, audited
(`action: knowledge_review_queue`), no model call; it reads the work items
and nothing else (no source is inspected, no file is written).

`POST /review/queue` is the owner's review view: it inspects every registered
source and returns each one's review state, the owner's options and their
calls, and the history. A caller needs what is waiting for a person's
decision and on which notes, so it can say an answer rests on a note under
review. This route lists:

- `items`: the open work items (status `open` or `escalated`), newest first,
  whoever raised them, each with the fields every producer writes:
  `work_item_id`, `kind`, `reason`, `action`, `status`, `title`,
  `created_at`, `updated_at`, `resolve_means`, `dismiss_means`,
  `notes`, the notes and sources it concerns, workspace-relative, and where
  the producer lists findings, `finding_count`, the first five `findings`
  (each a flat record: its plain fields, text cut at 200 characters, a path
  workspace-relative, the producer's dedup key left out) and
  `findings_left_out`; a count is never given without its findings. Left out: the owner's options (acting on an item is the
  owner's, `POST /review/action`) and each producer's other fields
  (signals, scaffolding), which the owner's view shows;
- `open_total`, `by_kind` (the open items counted by kind), `matched` (after
  the filters), `returned`, `next_offset` (send it back as `offset`).

Query parameters (`KnowledgeReviewQueueQuery`):
- `agent_id` (required), `workspace_id`, `scope_kind`, `scope_id`, `access_mode`
- `kind`: only open items of this kind, a key of `by_kind`
- `note`: only open items concerning this note, by its path or its key
- `limit`: 1 to 200, default 25
- `offset`: 0 to 100,000, default 0

Warning: `review_kind_not_open` (no open item of the kind asked for; `by_kind`
lists the kinds that are).

Since Track AB4 each item also carries `proposed_by` where an agent's proposal
raised it, and the proposals' kinds are `remember` and `forget` (below).

### `POST /agent/knowledge/propose` (Track AB4, 2026-10-17)
A caller agent's proposal for the owner's review;
`response_type: knowledge_proposal`. The one route that takes
`access_mode: propose`, and only where the workspace's owner allows proposals
(`allow_proposals: true` in its `config/mcp.json`; otherwise 403,
`proposals_not_allowed`: the switch means one thing over HTTP and MCP, decided
in the AB4 fix round and revisitable by the owner). `agent_id` must be an id
(letters, digits, `.`, `_`, `-`; `agent_id_invalid`), since it is written
into the note an accepted proposal becomes. It files one review work item and writes nothing else:
no note, no graph change, so nothing recalls a proposal until the owner
accepts it. Audited (`action: knowledge_propose`, refusals too, metadata only:
never the statement). No model call.

Request (`KnowledgeProposeRequest`):
- `agent_id`, `workspace_id`, `scope_kind`, `scope_id`; `access_mode:
  "propose"`, required
- `kind`: `remember` (add a statement to the collection) or `forget` (withdraw
  a note or an item from it)
- remember: `statement` (one statement standing on its own, at most 4,000
  characters), `provenance` (required: `note`, a note of the collection by its
  path, key or citekey; `source`, a file under the workspace's `sources/` or
  a declared source root; `conversation`, the conversation it came from in a
  few words, at most 500 characters; at least one of the three, and `quote`,
  optional, the passage it rests on exactly as written there), `subject`
  (optional, what it is about; it titles the note)
- forget: `id` (a note's path, key or id, or an item's id as an answer or a
  card gives it) and `reason`

Text a reviewer cannot see is refused (`proposal_hidden_characters`):
control characters, zero-width and bidi marks, line separators other than a
newline, and an instruction hidden in markup, in the statement, subject,
quote, conversation or reason (the injection scanner intake and fetched pages
use). Instruction-shaped prose a reader can see is filed, and the item names
it for the owner (`content_flags`).

Provenance is checked: a note must be one of the collection's (the owner's
work, an idea or a concept page is never evidence, `provenance_not_evidence`),
a quote must be in the note or source. A forget names a note of the collection
or an item one of its notes states (`proposal_target_not_evidence` otherwise).

Answer: `kind`, `status`, `work_item_id`, `work_item_status`, `filed`,
`proposed_by`, `proposal` (the statement, subject and every provenance given,
or the target and the reasons), `review` (`decided_by`, `resolve_means`,
`dismiss_means`), `written: {markdown: false, graph: false}`, and `note` for
a statement accepted before; `in_collection` (the item and its notes) when the
collection states it already; `similar_pending` (the ids of pending
statements sharing most of its words, at least three and 60% of them, case
aside, a run of a script written without spaces counted by its pairs of
characters; each names the other); `content_flags`. `status`, each with its notice:

| status | when | notice |
| --- | --- | --- |
| `queued_for_review` | a new item | `proposal_queued` |
| `already_queued` | the same proposal waits; this filing was added to it (who, and what it rests on) | `proposal_already_queued` |
| `already_dismissed` | the owner dismissed the same proposal: nothing is raised | `proposal_already_dismissed` |
| `already_accepted` | the same statement is a note of the collection | `proposal_already_accepted` |
| `already_withdrawn` | what it proposes to forget was withdrawn | `proposal_already_withdrawn` |
| `already_in_collection` | an item of the collection states it already (normalised as below); nothing is filed, `work_item_id` is null | `proposal_already_in_collection` |

MEMORY_VERBS v1 freezes `remember`'s `status` to `inserted`, `duplicate` or
`superseded`. A proposal waiting for a person is none of them, so the honest
value is reported and no write conformance is claimed (the plan's decision 1).

**The same proposal.** For `remember`: the statement alone,
Unicode-normalised (NFKC), case-folded, white space made single, quotes
wrapping the whole of it and sentence-final full stops, exclamation marks and
ellipses (and their forms in other scripts) dropped. A question mark counts:
"It works?" is not "It works." (fix round 2). Every other mark counts: "5%" is not
"5", "C#" not "C", "-40" not "40", since a dismissal is remembered for good
(the first version dropped any punctuation at the ends and merged such
statements). Not the subject (the same statement under two subjects, or
none, is one item; its `subjects` are merged). Not the provenance (a
statement the owner rejected is not made acceptable by citing another note)
and not the agent (two agents proposing one statement make one item, both in
`proposed_by`). For `forget`: what is withdrawn, by its path or id, not the
reason. The item keeps the words of its first filing; a later filing adds
who proposed it and what it rests on. A decision later undone (a contributed
note withdrawn, a withdrawal restored) no longer answers: the next proposal is
a new item (its id ends `~2`).

**The owner's decision**, through the review routes only
(`POST /review/action`, `resolve_work_item` or `dismiss_work_item`; no agent
route resolves). An agent named in the item's `proposed_by` cannot decide it
(409, `proposal_decided_by_its_proposer`), and `verified` and `accepted_by`
name the identity that did.

**The decider is not authenticated.** Neither the HTTP service nor the MCP
server has authentication yet (the MCP plan's "Later: authentication"), so
`agent_id` on `POST /review/action` is whatever the caller sends. The
proposer check stops one mistake: an agent deciding its own proposal under
the id it proposed with. A caller that sends another id is taken at its word,
and the note then records `verified: human:<that id>`; the trust tier read
from `verified` is only as good as the loopback boundary until that track
exists. On an install where agents can reach the HTTP service, the owner
should treat `human-reviewed` on a contributed note accordingly.

The decisions:

- resolving `remember` writes one note of the collection under
  `sources/wiki/contributed/`, type `agent-contribution`, with the statement
  as its TL;DR and its one claim, `generated` naming the agent,
  `verified` naming the person who accepted it (`human:<id>`),
  `contributed_by`, `contribution` (the item), `proposed_at`, `accepted_by`,
  `accepted_at`, `subject`, OKF `sources` (the notes it rests on, which become
  `DRAWS_ON` edges), `rests_on_files`, `conversation`, and a `## Provenance`
  saying all of it; the route ingests it. From then on it is evidence: an
  answer's item resting on it carries `contributed` (`by`, `accepted_by`,
  `proposal`, `rests_on`), since a read leaves a note's provenance section out;
- resolving `forget` of a note moves the note, byte for byte, out of the
  collection to `maintenance/withdrawn/<withdrawal id>/files/`, with
  `withdrawal.json` (who proposed it, why, who accepted it, its hash), and
  takes it out of the graph (its node and edges, its passage vectors and
  reading counts; the change feed records the removal). Nothing is deleted;
- resolving `forget` of an item takes its line out of each note of the
  collection that states it as one whole list item, keeps each note as it was
  beside the record, adds a line to the note's `## Provenance`, and re-ingests
  them. Where the item is not one whole list item, or a note is maintained by
  its owner, nothing changes and the resolution is refused (409): the edit is
  the owner's;
- dismissing changes nothing but the item, which then answers the same
  proposal with `already_dismissed`;
- `POST /maintenance/withdrawals/restore` `{withdrawal_id, agent_id}` puts a
  withdrawal back and re-ingests it (409 when a file is there again or the note
  changed since), resolving caller, scope and policy like the other owner
  routes;
- a withdrawal outlives the note being made again, by any route: it is
  enforced where every note is written (normalization) and every note is read
  into the graph (ingestion), so `POST /sources/process`, `/sources/reread`,
  `/normalize/file`, `/ingest/file`, `/maintenance/refresh`, the scheduled
  registry refresh and the graph rebuild all keep it. A withdrawn note made
  again is kept aside beside its withdrawal and not ingested
  (`withdrawn_note_kept_out`: a refusal, 409, on `/normalize/file` and
  `/ingest/file`; `withdrawn` in a process answer; a skipped source in a
  refresh), and a withdrawn statement in a note the platform makes again is
  taken out again (`withdrawn_statement_kept_out`). Only a restore puts
  either back. Reading a note into the graph (ingest, the rebuild) never
  rewrites the owner's Markdown (fix round 3): a withdrawn note put back by
  hand is refused (409) and left where it was put; a withdrawn statement the
  owner put back is ingested as written, the answer warns
  (`withdrawn_statement_present`), and one review item
  (`withdrawal-conflict:`, never filed twice) asks the owner to release the
  withdrawal or remove the line. Standing withdrawals are looked up by path
  in an index kept while `maintenance/withdrawn/` is unchanged;
- a restore works when the note changed since (fix round 2): a statement
  withdrawal whose note was made again or edited puts its lines back where
  they were (`reinserted`) rather than the old file over the new text; a note
  withdrawal whose path holds a file again (the owner moved the note made
  again back) is released, that file kept and ingested (`released`). Either
  way the withdrawal stops standing, so the next re-process keeps the text.
- an item stated as a whole list item in several sections of one note (a
  note the pipeline wrote can) is taken out of each.

### `GET /agent/knowledge/codes` and `GET /agent/knowledge/contract`: their queries as models (Track AB3)
`GET /agent/knowledge/codes` takes `code` (`KnowledgeCodesQuery`; repeated for
several): only those codes, `asked` naming them; a code built from a prefix
is answered by its family's entry, with `family` (`normalization_config_*`);
one not registered is the warning `code_not_registered`; the start of a name lists every code it begins (`matched_by`, at most 40, `code_prefix_capped` beyond). `area` lists the codes of one area of the platform (`answers`, `reads`, `connection`, `proposals` (Track AB4), `requests`, `background`, `intake`, `platform`; every entry now names its `area`; `code_area_unknown`). Without either, every code, as before. `GET /agent/knowledge/contract` takes its `mode` through
`KnowledgeContractQuery`; the parameter and the answers are unchanged.

---

## `POST /agent/knowledge/answer` response shape

```json
{
  "schema_version": "agent_knowledge_response_v2",
  "contract_version": "agent_knowledge_contract_v1",
  "response_type": "knowledge_answer",
  "contract": { ... },
  "caller": {
    "agent_id": "kronnos",
    "workspace_id": null
  },
  "scope": {
    "scope_kind": "agent",
    "scope_id": "kronnos",
    "workspace_id": null,
    "resolution_basis": "agent_default"
  },
  "policy": {
    "access_mode": "read_only",
    "allowed_actions": ["read_knowledge"],
    "denied_actions": ["refresh_knowledge", "suggest_changes", "write_knowledge", "admin_override"],
    "resolution_basis": "read_only_default"
  },
  "query": "string",
  "expansion": {
    "facets":  {"topic": ["memory"]},
    "terms":   ["remember", "recall"],
    "source":  "agent | cache | unavailable",
    "reason":  "why, when source is unavailable",
    "backend": "openclaw | claude_code",
    "widen": "auto | cache_only",
    "agent_call": true,
    "agent_calls": 1,
    "billed_tokens": 1350,
    "skipped_by_request": true
  },
  "answer_summary": "string",
  "knowledge_items": [
    {
      "node": { ... },
      "score": 0,
      "score_breakdown": { ... },
      "why_relevant": "string",
      "provenance": [ ... ],
      "related_nodes": [ ... ],
      "wiki_page": { ... } | null
    }
  ],
  "header_fields": [
    {"note": "<note>", "fields": {"<name>": "<value>"},
     "where": {"<name>": "<note>#frontmatter:<lines>"}}
  ],
  "read_next": [ ... ],
  "gap_signals": [ ... ],
  "bridge_signals": [ ... ],
  "confidence": {
    "overall_confidence": 0.0,
    "retrieval_confidence": 0.0,
    "provenance_strength": 0.0,
    "source_quality": 0.0,
    "query_alignment": 0.0,
    "coverage_ratio": 0.0,
    "fill_ratio": 0.0,
    "question_coverage": 0.0,
    "question_terms": {"in_evidence": [], "elsewhere_in_collection": [], "absent": []},
    "question_not_covered": false,
    "freshness": { ... },
    "runtime_mode": "graph_retrieval_heuristic_v1",
    "weak_context": false
  },
  "source_quality_summary": "string",
  "warnings": ["..."]
}
```

### `header_fields`: what the notes' headers say about the question (Track AB6)

Additive to contract v2, on the answer and the task context, in the full and
the compact view. A note's frontmatter often holds what a question asks (who
attended, who wrote it, when, for how many), and an answer's items are the
note's statements, never its frontmatter. So the answer lists, per note, the
header fields (the frontmatter less the platform's bookkeeping,
`docs/note_format_contract_v1.md`, "The header") whose name or value shares a
word with the question: words of four letters or more compared by their first
five (any spaced script: "attended" finds `attendees`), numbers of two digits
or more, in a script written without spaces pairs of adjacent characters, and
a field name shorter than that (`url`) by the same word. The collection's
function words (Track AA's measure: "which", "with" in a collection of
statements) are left out of the question first (AB6 fix round, R72). A
source document's own frontmatter counts as the note's, its fields cited to
that file.

Which fields, and in what order (AB6 fix round, C60): read from the ten
best-ranked notes; a field whose *name* the question holds ("channel",
"published") before one whose value merely shares words with it; the
best-ranked note may give a field one word of its value matches, any other note
only a field its name matches or a value holding two thirds of the question's
words (at least two when it has more), so other pages' addresses sharing a few
words of a title are not offered. Notes with a field matched by name first,
then by rank, then the focus's notes moved first; when one source carries the
answer (`focus.kind: single_source`) its fields alone. At most 3 notes, 3
fields each, a value cut at 200 characters. Empty when no field matches, so an
answer pays for it only when it is likely to matter. Each field's `where` is
the file (the note or its source), `frontmatter` and the lines it was read
from, the form an item's provenance takes: an answer citing a header field
cites that.

Per note rather than per item, since a note's several items would repeat it.
Question coverage (`question_terms`, `absent_terms`) counts the fields an
answer shows as evidence, and a word only a note's declared header values hold
as in the collection (C59): an answer never reports a word it shows as absent,
or as found nowhere.

A note is also *found* by a header value when its type declares the field as
identifying it (`identity_fields`: the built-in's, or the install's own in
`config/note_types.json`): ranking reads those values with its title, never
scoring a note lower than its title alone would, and they weigh no word in the
collection's statistics, so a question naming none ranks exactly as before
(R71). Ingestion records them on the note's node (`header_values`); for a note
ingested before that they are read from its Markdown when the answer runs (the
note index the answer already keeps, remembered while the graph is
unchanged), so an existing install needs no re-ingestion. A field nobody
declared is shown, never searched.

---

### `provenance_strength` (corrected AB2c, 2026-10-17)

Per item, `min(n / 3, 1) × 0.65 + (mean confidence of its provenance
references) × 0.35`, where `n` is the number of the collection's **source
notes that state the item**: its own provenance and every note with a
`MENTIONS` edge to it, concept notes, maps, ideas and owner notes left out (a
synthesis is not support). One note gives 0.567, two 0.783, three or more
1.0 (at full extraction confidence); the answer's value is the score-weighted
mean over its items, and each item's `confidence.supporting_notes` is its `n`.
Before AB2c `n` was the count of the item node's own provenance references,
of which the store keeps exactly one per node (the note ingested last), so
every answer of every install read 0.567: the number carried no information.
It now says how corroborated the evidence is; `low_provenance_density` (below
0.5) still means items without any provenance.

## `expansion` — what the question was taken to be about

Retrieval compares strings, so a question about how an agent *remembers*
something does not by itself reach a note labelled `memory`. Before ranking, the
question is mapped onto the workspace's facet vocabulary; `expansion` reports
what that mapping produced.

`source` is the field to read:

| value | meaning |
| --- | --- |
| `agent` | a model was consulted for this question |
| `cache` | the same question was mapped before, and the answer was reused |
| `unavailable` | no mapping happened, and `reason` says why |

**`unavailable` is not an error.** Retrieval is designed to degrade: with no
mapping it ranks on lexical match, graph structure and facets that the question
names outright, exactly as it did before expansion existed. The answer is still
returned, and `warnings` carries `semantic_expansion_unavailable`.

It is surfaced because it is otherwise invisible. A thin answer caused by an
unreachable CLI and a thin answer caused by a corpus that holds little on the
subject look identical to a caller, and only one of them is worth acting on. A
consumer that cares about recall should treat `unavailable` as "this answer may
be narrower than the corpus supports", not as a failure.

### `semantic_search`: whether passage embeddings reached anything

Additive, on every answer: `{enabled, model, contributed, code, reason,
notes_reached}`. `contributed` is true only when at least one note was reached
by meaning (`notes_reached` > 0). When it is false, `code` says which of two
very different things happened:

| code | meaning | warnings |
| --- | --- | --- |
| `no_match` | the search ran; nothing in the corpus came close enough (`min_similarity`) to what was asked | `semantic_search_no_match` |
| any other (`disabled`, `server_unreachable`, `not_indexed`, `stale_format`, `over_cap`, `error`, …) | the signal did not run on this installation | `semantic_search_unavailable` and `semantic_search_<code>` |

`no_match` is a statement about the corpus, never about the installation: do
not treat it as a fault to fix. Before 2026-10-06 a no-match search reported
`contributed: true` with code `ok`.

Common reasons, none of which are faults:

- `workspace has no vocabulary` — a new workspace has none until its first
  ingest bootstraps one. Nothing to map onto, so no call is spent.
- `SECOND_BRAIN_BRAIN_KEEPER_DISABLED=1` — model calls are switched off.
- `semantic expansion is disabled for this workspace` — turned off in
  `config/semantic_expansion.json`.

`facets` are validated against the closed vocabulary, so a value that appears
there is registered in this workspace. `terms` are unvalidated model output,
recorded for inspection and deliberately not used for ranking.

### What widening cost, and `widen: "cache_only"` (Track I5, 2026-10-09)

Widening a question not seen before is one brain-keeper call (about 1,350
billed tokens measured on the first instance), made on behalf of a caller that
may think it is only reading. Additive fields on `expansion`:

| field | meaning |
| --- | --- |
| `agent_call` | a brain-keeper call was made for this answer's widening (a failed one included) |
| `agent_calls` | how many: a failed call is not cached, so each of the answer's rankings asks again |
| `billed_tokens` | what those calls billed, summed; `0` when served from the cache |
| `widen` | the mode in effect |
| `skipped_by_request` | present and true when `widen: "cache_only"` is why nothing was asked |

Request option `widen` on `/agent/knowledge/answer`, `/task_context`,
`/read_next` and `/query/resolve_sections`: `"auto"` (default, as before) or
`"cache_only"`, which uses a cached widening when there is one and otherwise
ranks without one (the free deterministic path), never making an agent call.
A `cache_only` miss reports `source: "unavailable"` with that reason and adds
no `semantic_expansion_unavailable` warning, since the caller chose it. Any
other value is refused with 422. The call is logged in the workspace's call log
as operation `query_widening`, so `GET /ops/metrics` groups what reading cost.

## `synthesize: true` — a written answer, and when "not found" is earned

Optional request flag (default `false`, which makes no model call). The brain
keeper writes a short answer; the templated `answer_summary` stays alongside.

Optional `evidence` (with `synthesize`): `compact` (default) or `full`.

- **compact** reads the text the ranking points at and widens only when the
  answer is not complete: (1) `sections` the top-ranked nodes point at,
  (2) `full_notes` — the best notes' content sections, (3) `source_layer` —
  passages of the converted originals plus an outline of each, (4)
  `source_full` — whole originals, only when an answer is partial *and* the
  model's latest reading of them still asks for more. A stage may be followed
  by one `<stage>_requested` call carrying outline blocks the model asked for;
  note sections it asks for travel with the next stage instead. A later stage
  sends only text not sent before; what earlier stages established travels as
  the answer so far, the parts it still misses, and the evidence it cited
  (marked as earlier). A stage with nothing new is skipped (`skipped:
  "nothing_new"`).
- **completeness** is a checklist: the first answer splits the question into
  `parts`, every later stage judges the same parts, and `complete` means every
  part is answered — decided by the service, not taken from the model's verdict.
- **full** reads whole note files as written, then whole originals. Nothing is
  withheld; several times the tokens. For audits and provenance-heavy work.

**Nothing describing the documents is withheld in compact mode.** Every note
the model reads carries a card — the fields that describe the document
(`notes.contract.card_fields`: title, type, status, verification, authors,
year, venue, authority, …) — and a list of its sections that were not sent,
which the model may ask for. What compact mode leaves out of the model's
prompt is bookkeeping only (node ids, file paths, JSON packaging); this
response still carries all of it.

Additive field `synthesized_answer`:

| field | meaning |
| --- | --- |
| `answer`, `answered`, `caveats` | the answer, whether it answers the question, and its stated limits |
| `complete` | every part of the question was answered; `answered` with `complete: false` is a partial answer |
| `sources` | the evidence the answer cites: `{layer, note, section?, source?}`, paths relative to the checkout |
| `supporting_node_ids` | graph nodes behind the cited sections |
| `unverified_figures` | numbers and quotations in the answer not found verbatim in the evidence it cites — computed, reformatted or wrong |
| `answered_from` | the stage whose answer was kept |
| `lower_trust` | the answer draws on `converted_source` text — closer to the original, but not curated |
| `notes_reached`, `notes_examined_in_full`, `searched_whole_reach` | how much of what the ranking reached was read whole |
| `stages` | each call: evidence and carried counts, characters, `answered`, `complete`, `parts`, `requested`, `unverified_figures`, job id, token `usage` |
| `usage` | tokens across all calls (`input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_write_tokens`, `billed_tokens`) |

Warnings:

- `corpus_cannot_answer_query` — not found, and every note the ranking reached
  was read in full (`searched_whole_reach: true`).
- `answer_not_in_retrieved_text` — not found in what was read; not everything
  reached was read whole. Not a statement about the corpus.
- `answer_figures_not_in_evidence` — `unverified_figures` is not empty: check
  those before relying on them. An answer whose figures are mostly absent from
  its cited evidence is also treated as incomplete, so reading continues.

A found answer removes a `corpus_cannot_answer_query` raised by
`judge_relevance`, which judges item labels only. Budgets are in
`config/answer_synthesis.json`, overridable per workspace.

## Conversations: `conversation_id` and `refresh`

Optional on `/agent/knowledge/answer` and `/agent/knowledge/task_context`.
Omitted: the response is unchanged. `"new"` starts a conversation; its id comes
back in `conversation.id`. Passing it back sends only what that conversation has
not received: unchanged `contract`, `capability_contracts` and `gap_signals`
become `null`/`[]`; items and read-next suggestions already sent become stubs
(`already_sent: true`); per-note blocks (`wiki_page`, `source_layers`,
`maintained_state`) are sent once in `shared` and referenced as `{"$ref": fp}`.
Everything left out is listed in `conversation.omitted`. `refresh: true` sends
everything again. A conversation belongs to one agent and workspace and expires
when idle (`config/conversations.json`); any refusal starts afresh and says why
in `conversation.resolution`. See docs/agent_conversations_plan_v1.md.

## Concept notes in an answer

Optional, additive (October 2026). `knowledge_answer` carries `concept_notes`:
the concept notes the query names by label or alias, longest name first, at
most three — each with `key`, `label`, `summary`, `source_count`,
`first_seen`/`last_seen`, `has_divergences`, `synthesis`, `definition_basis`,
`path`, and `read`, the route that returns the full note with its sources.
Empty when the query names no concept. See `docs/concept_notes_v1.md`.

`definition_basis` (additive, 2026-10-09) is `sources`,
`sources_and_general_knowledge`, or `null` when the note records none. A
definition with basis `sources_and_general_knowledge` contains general
knowledge: do not cite it as a finding of the collection.

## Ideas in an answer: `related_ideas` (Track H1, 2026-10-09)

Additive to contract v2. A workspace's **ideas** (`type: idea` notes, and
research-direction notes read as ideas; see `docs/ideas_v1.md`) are a separate
epistemic tier: never evidence. Ranking skips idea notes, so an idea is never a
`knowledge_items` entry, a `read_next` suggestion, a `/query/resolve_sections`
section or a synthesis input. The answer and `task_context` list the ideas
linked to the query apart, under `related_ideas`, at most
`contract.related_ideas_limit` (2 short, 3 medium, 5 deep):

| field | meaning |
|---|---|
| `key`, `title`, `path` | the idea, its workspace-relative note path; `read` is `/ideas/<key>` |
| `status` | `seed`, `exploring`, `parked`, `dropped`, `done` (a research direction's `accepted` reads as `seed`) |
| `origin` | `owner`, `agent`, `gap_filler`, `conversation` |
| `maintained_by` | `owner` when the owner looks after it, else null |
| `basis` | why it is listed: `concept` (about a concept among the evidence, or one whose every word the question holds), `rests_on` (rests on a note the evidence comes from), `words` (its own text holds at least half of the question's words weighted by information, log(labels / labels holding the word), by the strict matching of `question_coverage`; a word no label holds weighs nothing. Since AB2c: the ranking's bounded weight let "does", "paper", "propose" attach a note holding none of the subject) |
| `why_related` | the same, one sentence per reason |

Ordered by how many kinds of link it has, then status (`exploring` first,
`dropped` last). Empty when the workspace has no ideas or none is linked.
Deterministic, no tokens, and never at the cost of the answer: a failure
leaves it empty. Confidence, coverage and warnings ignore it.

A caller reports these as the owner's own ideas ("you had this idea"), in a
part of its own, and never cites one as a finding of the collection.

## The owner's own work in an answer: `from_your_work` (Track K2, 2026-10-09)

Additive to contract v2. The owner's own notes (`type: owner_work`: their
designs, and their commentary on other people's work; see
`docs/owner_work_v1.md`) are a tier of their own: never evidence. Ranking skips
them and ingestion turns none of their lines into nodes, so an owner-work note
is never a `knowledge_items` entry, a `read_next` suggestion, a
`/query/resolve_sections` section or a synthesis input, and a commentary is
never cited as the claims of the work it discusses. The answer and
`task_context` list the owner's notes linked to the query apart, under
`from_your_work`, at most `contract.from_your_work_limit` (2 short, 3 medium,
5 deep):

| field | meaning |
|---|---|
| `key`, `title`, `path` | the note and its registration's workspace-relative path; `read` is `/owner-work/<key>` |
| `text_path` | where the owner keeps the text: `<root>:<path>` under a declared source root, or workspace-relative |
| `kind` | `design` (the owner's own contribution), `commentary` (notes on someone else's work) or `notes` (notes on a topic or on something that happened; Track U1) |
| `status` | `draft` or `stable` |
| `about` | `{as_written, note, held, citation, title, path, basis}`: the source it is about, `note` when the collection holds it; `as_written` null and `basis: inferred` when it was found from the note itself (Track U1); null for a note about nothing in particular |
| `maintained_by` | `owner` |
| `open_problems` | how many assumptions and limits a design states (`GET /owner-work/open_problems` lists them) |
| `basis` | why it is listed: `concept` (about a concept among the evidence, or one whose every word the question holds), `about` (discusses, or compares itself with, a note the evidence comes from), `words` (its title, opening and leading parts hold at least half of the question's words weighted by information, as for ideas above) |
| `why_related` | the same, one sentence per reason |

Ordered by how many kinds of link it has, designs before commentary. Empty when
the workspace has none or none is linked. Deterministic, no tokens, never at
the cost of the answer. Confidence, coverage and warnings ignore it.

A caller reports these as the owner's own work ("in your notes on X you
wrote", "your design assumes"), in a part of its own, and never cites one as a
finding of the collection or of the work it discusses.

**The owner's words beside a source: `kind: my_notes`** (Track AH, 2026-10-22).
Additive, in the same list. What a person typed under `## My notes` in a note
the pipeline wrote (`docs/note_format_contract_v1.md`, "The owner's section")
is theirs and never evidence: no line of it is a claim, a concept or a
relation, and it is never a `knowledge_items` entry. A question using its words
still finds the note (the item's `score_breakdown.found_through` is
`owner_notes`, and the words are not in the item's node), and the answer lists
the words here, after the owner-work notes, at most `from_your_work_limit`:

| field | meaning |
|---|---|
| `kind` | `my_notes` |
| `key`, `title`, `path`, `text_path` | the note: its file stem, title and workspace-relative path |
| `text` | the owner's words, at most about 600 characters, starting at the line that holds the question's words |
| `label` | "the owner's own words beside this source, not what the source says" |
| `about` | `{note, held: true, title, path, basis: "own_section"}`: the note they sit in |
| `maintained_by` | `owner` |
| `basis`, `why_related` | `about` (written beside a source the answer's evidence comes from) and/or `words` (they hold at least half of the question's words weighted by information) |
| `read` | `{route: "POST /agent/knowledge/read", body: {notes: [path]}}`, which shows them under their own block |

A caller reports these as "in your notes on X you wrote", and never as what X
says. `POST /agent/knowledge/read` adds `owner_notes` (`{heading, text,
truncated, path, label}`, null when there are none) to each source, and its
Markdown shows the words under "the owner's own words beside this source (not
what it says)", never among the source's sections; the compact view keeps
`owner_notes` too.

## Every kind of statement ranks (Track U3, 2026-10-11)

Additive; no field changes. Answer weighting is defined by the note contract's
section roles, not by node-type names: every node type a profile maps a
section to ranks by that section's role (`answer` and `limitation` 4.0 like a
paper's methods, `question` 3.0, `context` 1.0), including the types an install
declares in `config/note_types.json` (a recipe's `Ingredient` and `Step`, an
account's `Commitment`) and the `Statement`s of notes written by hand. The
shipped types rank exactly as before. A meeting's decision is evidence for a
question about it; a meeting's risks, an install's own limitation-like sections
and a hand-written note's `## Risks` seed the limitation clusters
(`/query/gaps`). `focus` takes `scope: {"identifier": …}` and names a source
by any identifier its type declares (below).

## Answers in fewer steps: `focus`, the one-call read, `view: compact` (Track Q, 2026-10-11)

Additive to contract v2. Measured on the first instance, a caller answered a
one-paper question through the brain in 14 turns and 423k tokens (371k of them
cache re-reads), against 4 turns and 74k tokens reading the PDF itself: every
turn re-sends the caller's context, so the number of calls and the size of
each response are the cost. Three additions let a caller answer in two calls,
three when equations must be read.

### `focus` on the answer and `task_context` (Q1)

| field | meaning |
| --- | --- |
| `kind` | `single_source` (one source carries the answer), `multi_source` (the evidence is spread), `unsure` (a tie or no clear concentration; read it as `multi_source`) |
| `decided_by` | `caller` (request `scope`), `named` (the question names the source), `evidence`, `none` (no strong evidence), `error` |
| `why` | one sentence, with the shares and thresholds when the evidence decided |
| `notes[]` | `note`, `title`, `citekey`, `source`, `share`, `items`, `attached` (`owner_work` and `ideas` keys, `equation_items`, `figure_items`), `has_unread_regions`, and `named_by` when named |
| `evidence` | `strong_items`, `sources`, `shares[]` |
| `read` | `route: POST /agent/knowledge/read`, `body` (works as given: the notes, with the evidence's note sections, `view: compact`), `then` |

Decided from structure only, never from words of any language:

1. **The caller's `scope`** (a request field, not the governance scope):
   `{"note": "<path or key>"}`, `{"citekey": "<key>"}`, or (Track U3)
   `{"identifier": "<value>"}`: any value of a field the note's type declares
   as an `identifier` (an account or session id, a URL, a DOI, a citekey). One
   that names no note gives `unsure` and says so. Since AB2c the scope also
   filters the answer to that source (above), so the share and items the
   focus reports are the scoped answer's own.
2. **The question names a source**: its citekey, or any identifier its type
   declares (`named_by: its identifier "…"`), when the key is not an
   ordinary word (it holds a digit, `_`, `:` or `-`); a short name the source
   defines for itself (`identity_terms`, matched case-sensitively, and only
   when no other source has it); its whole title (12 characters or more); or
   the part of its title before a colon or dash, when no other title shares
   it. Concept notes, maps, ideas and owner notes are syntheses, never named.
   **A collection's own name names none of its notes** (AB2c): a title head
   that another source's title also holds, or whose words more than a third
   of the source notes hold (and at least three), is the collection's subject
   — a project's name heading one meeting note's title and running through
   every other — and not that note's name. Counted from the collection, in any
   language; a head written without spaces is checked against the titles
   only. And when a title or its head names a source but the strong evidence
   is concentrated on another (the evidence alone would decide
   `single_source` for it), focus is `unsure`, `decided_by: named`, and lists
   the named source first and the evidence's source second, with `why`
   giving both shares: the caller reads both. A citekey, a declared
   identifier or a short name the source defines for itself still decides
   alone (each named source carries how it was named, and only a title or
   its head gives way).
3. **The evidence**: every ranked item scoring at least half the best counts,
   weighted by its score and split evenly over the source notes it rests on
   (its provenance and every note with a `MENTIONS` edge to it, since an
   entity two notes share keeps one node). The owner's work about a source
   and ideas resting on it add 0.25 each (at most two). `single_source` needs
   a share of at least 0.6 and a lead of 0.3; `multi_source` a top share below
   0.5 with at least three sources at 0.15 or more; two leading shares within
   0.15 of each other are a tie (`unsure`). Evidence that does not cover the
   question (`question_not_covered`) never makes `single_source`.

### `POST /agent/knowledge/read` (Q2)

`notes`: each a workspace-relative note path, a note's key or a citekey, or
`{"note": ..., "sections": [<heading>, ...]}` naming headings to include (of
the note or of the source's outline, or a path `Part > Section`).
`max_chars` (default 32,000, from 2,000 to 200,000) is the Markdown budget for
the call, shared by the notes in order: what one leaves, the next may use.

For each note the route reads its **document**: the source's converted copy,
else a text original, else the note itself. It is split at every heading of
the document's own outline (`equations/headings.py`). The whole document comes
when it fits the source's share of the budget; otherwise the sections the
caller named, then the best for the question: heading-path words (twice) and
text words, weighted by rarity in the source and the workspace (any language,
no word lists), a bonus for holding a sentence of the evidence sections named,
and the document's opening. The best match always comes, cut at a paragraph if
it must be. The note's first section with content and the note sections named
come too, within a quarter of the source's budget.

Response (`view: compact`): `query`, `budget` (`max_chars`, `used_chars`,
`sources`), `not_found`, `warnings`, `warning_details`, and `sources[]`:
`note`, `title`, `markdown`, `chars`, `left_out` (`heading_path`, `chars`),
`unread_regions` (compact, below), `truncated`. The Markdown holds a head
line (note, source, citekey, the file read from), the note's header and the
source's (Track AB6, above), `[note]` parts, the
document's sections (each `## <heading path> — <file>:<lines>`), "Read
equations elsewhere in the source" and "Figures elsewhere in the source (read
from the image)" that match the question, "Not read yet", the owner's work
about it and ideas resting on it (never findings), and "Left out". `view:
full` adds per source `sections[]` (`origin`, `heading_path`, `path`,
`line_start`, `line_end`, `text`, `why`, `score`, `matched_words`),
`document`, `document_chars`, `equations_elsewhere`, `figures_elsewhere`,
`from_your_work`, `related_ideas`, `left_out_count`, `header`,
`source_header`, and the full `unread_regions`. Warnings: `evidence_has_unread_regions`,
`read_left_out_matching_sections`, `read_note_not_found`. 403 for a path
outside the workspace that is not an ingested note; 404 when no named note
exists; 422 without `notes`.

### `view: compact` (Q3)

On `/agent/knowledge/answer`, `/agent/knowledge/task_context`,
`/query/resolve_sections` and the read. `full` stays the default on the first
three. Compact keeps the governance fields (`schema_version`,
`contract_version`, `response_type`, `caller`, `scope`, `policy`), `focus`,
`warnings`, `warning_details`, and:

- `knowledge_items[]`: `id`, `type`, `label`, `score`, `where`
  (`<path>#<section>:<lines>`), `snippet`, `also_in`, and `image_derived`,
  `derived_from`, `unverified`, `source_missing` when set;
- `confidence`: `overall_confidence`, `provenance_strength` (below), `coverage_ratio`,
  `question_coverage`, `question_not_covered`, `weak_context`, `freshness`
  (its status), `absent_terms`, `scattered_terms`;
- `unread_regions[]`: `note`, `source`, `section`, `unread_equations`,
  `equation_ids`, `equations_without_ids`, `figure_numbers`,
  `uncaptioned_figures`, `pages`, `regions_located`, `why`, `recover`
  (`route`, `body`, `run_body`), `recover_all`, `recover_figures`,
  `estimate` (`vision_tokens`, `local_seconds`, `auto_would_use`,
  `figure_tokens`), `sections_matched` (`summary`, `equation_ids`);
- `read_next[]` (`id`, `label`), `gap_signals[]` (`title`), `related_ideas`,
  `from_your_work`, `concept_notes` (keys, titles, status, basis,
  `why_related` and `read` routes), `semantic_search` (`contributed`, `code`),
  `expansion` (`source`, `widen`, `agent_calls`, `billed_tokens`),
  `query_decomposition` (`parts`), `header_fields` (Track AB6, as in the
  full view), `contract` (`mode`, `full`);
- for `resolve_sections`: each section's `source_path`, `section`, lines,
  `score` and `text` cut to 1,500 characters (`text_truncated`); the top five
  `notes_ranked`; full notes when asked for.

A compact answer is at least half the size of the full one on the same
question, with the same item ids (tested on synthetic fixtures). With
`conversation_id`, the conversation's own compaction runs first.

### What the audit records (Q-R3)

`knowledge_answer` audit events carry `focus` (its kind), `focus_notes` and
`unread_notes` (the notes whose `unread_regions` entry has something to
read); `knowledge_read` events carry `notes` and `unread_notes`. Paths only,
never content. A caller's recovery is the separate `source_equations_read` or
`source_figures_interpret` event, so an answer that left the focus source's
equations unread can be told apart from one that read them.

### Consent to spend (Q-R2)

The answer and the read never spend. A recovery spends, and the research
skills teach that a caller asks its user with the estimate, unless the user
or the operator has already approved spending up to a stated amount and the
estimate of the method that would run (`estimate.auto_would_use`;
`vision_tokens` for vision, none for local) is within it: that stated cap is
the user's yes. The collection's standing token budget (`GET /ops/budget`,
Track AR) is such a stated amount, so within it the caller recovers without
asking (the registry's `run`, settled by the caller). Over the cap, or with
no cap, the caller asks, and that is the one budget question a user is ever
asked (Track AR). Either way it reports what was spent (`spent_tokens`,
`spend_note`).


## Intended use by agents
An agent should usually:
1. call `/agent/knowledge/answer` (`view: compact` when it will act on ids and pointers)
2. send `focus.read.body` to `/agent/knowledge/read` to read what the answer rests on in one call, rather than opening notes one at a time
3. if `concept_notes` is not empty and the question is about the idea itself —
   what it means here, how it developed, where sources disagree — read it
   through its `read` route first, then the sources it lists
4. use `answer_summary` plus `knowledge_items` as the initial working context
5. consult `read_next` if more exploration is needed
6. inspect `gap_signals` and `bridge_signals` when planning, critique, or synthesis is required
7. mention `related_ideas` as the owner's own ideas, apart from the findings
8. mention `from_your_work` as the owner's own work, apart from the findings and the ideas

---

## Why this contract matters
This turns `second_brain` from a set of useful backend endpoints into a real agent knowledge service with:
- stable outputs
- bounded context sizes
- explicit relevance and provenance
- reusable depth modes

---

## Confidence and warning surfaces
- `confidence` exists to help downstream agents distinguish strong context from weak or partial context.
- `source_quality_summary` is the short human-readable interpretation of those signals.
- `warnings` should be treated as execution guidance, not decoration.
- When `weak_context` is true, an agent should verify with provenance before relying on the answer.

### Coverage is about the question, not the slot count (Track I1, 2026-10-09)

`coverage_ratio` used to be the share of answer slots filled. A subject the
collection does not contain fills every slot just as well, with whatever shares
a common word, so a question about an absent topic read `coverage_ratio: 1.0`
and `weak_context: false`. It now means the share of slots filled *with
evidence about the question*: `fill_ratio × question_coverage`. Additive fields
in `confidence`:

| field | meaning |
| --- | --- |
| `fill_ratio` | the share of answer slots filled (what `coverage_ratio` used to be) |
| `question_coverage` | the share of the question's words, each weighted by its rarity in this workspace, that the retrieved items' labels hold. A word only elsewhere in the collection counts half; a word found nowhere counts nothing. No stopword list: a word every label shares weighs little in any language, and a word in more than 30% of labels (a template heading's, scaffolding) is left out |
| `question_terms` | which of the question's words were `in_evidence`, `elsewhere_in_collection`, `absent` (found nowhere; usually the subject the collection lacks), or `scattered` (found, but never two together; see below) |
| `question_not_covered` | `question_coverage` is at or below 0.5 and semantic search did not vouch for the evidence |

A word is in the evidence only by a **strict** match (I-R1): the same word once
plural and simple inflection endings are removed (`glazes`, `glazing`,
`glaze`), or one form extending the other by at most two letters. Never a shared
prefix alone: the ranking's fuzzy match counted a word as present in a
collection that held only a longer word starting with the same letters, so an
absent subject read as fully covered. Ranking keeps its fuzzy match. Chapters
and section headings are not evidence; only item labels are.

**Phrase support.** In a question of at most four counted words, the rare ones
(rarity above 0.5) that are found must be found together: when no item holds
two that are adjacent in the question, each counts only as found elsewhere and
is listed under `scattered`. Two words of a short question, each in a different
note, are not evidence about the question. A longer question names several
things that rightly sit in different items, so the rule does not apply to it.

When `question_not_covered` is true, `weak_context` is true, `warnings` carries
`question_not_covered` and `weak_context`, and `warning_details` says which
words the collection lacks and that the caller should say the collection does
not contain the subject. A semantic match (`semantic_search.contributed`) is
evidence of aboutness without shared words (a synonym, another language), so it
suspends this check. `overall_confidence` is gated by `question_coverage` as it
is by `query_alignment`: full trust from 0.6 up.

`partial_result_set` keeps its meaning: fewer items than the mode asks for,
read from `fill_ratio`.

`low_query_alignment` now has a `warning_details` sentence: the items share
little wording with the question and may not address it; read their sources,
and if they do not answer it, say the collection does not appear to cover it.

Measured on the synthetic benchmark corpus (`eval/retrieval/corpus`): its 44
genuine questions score `question_coverage` 0.572-1.0 and none turned weak;
eight invented off-topic questions score 0.161-0.430 and all are weak. Ranking
is unchanged: this reads the ranking, it does not alter it.

### Evidence whose original is gone: `source_missing` (Track J4, 2026-10-09)

Additive to v2. When a note's original source file is gone (found by
`POST /sources/reconcile`, `docs/source_renames_v1.md`), what rests on that
note says so:

| where | field |
| --- | --- |
| `knowledge_items[].provenance[]` | `source_status` (`missing` or `retained_without_source`) and `source_missing: true` on a reference into such a note |
| `knowledge_items[]` | `source_status`, and `confidence.source_missing: true` |
| `warnings` | `source_missing`, with a sentence in `warning_details.source_missing` |

The note still says what the source said; what is lost is the ability to
re-read or check it. `retained_without_source` means its owner chose to keep
the knowledge anyway. No confidence number changes: a caller weighs this
itself, and should say the original is gone when it cites such evidence.

### Evidence resting on an unverified value: `unverified` (Track L2, 2026-10-09)

Additive to v2. A value a note states may have been read from text whose layout
was lost in conversion (a table shredded by a two-column page), or be printed
by the source in another row. Such a line carries an `[unverified: ...]` mark in
the note (written at extraction), or is listed by the last fidelity pass
(`POST /maintenance/fidelity`, `docs/reading_fidelity_v1.md`). What rests on it
says so:

| where | field |
| --- | --- |
| `knowledge_items[].provenance[]` | `unverified: true` on a reference to such a line |
| `knowledge_items[]` | `unverified: true`, `unverified_reason` (`marked_in_note`, `layout_lost`, `source_row_differs`), and `confidence.unverified: true` |
| `warnings` | `unverified_values`, with a sentence in `warning_details.unverified_values` |

Nothing is dropped and no confidence number changes. A caller should say the
value is unverified, and read the source lines (`GET /review/source` with
`line_start` and `line_end`) before relying on it. A caller that finds the note
wrong files a correction (`POST /maintenance/corrections`), which is a proposal
and allowed `read_only`.

### Evidence that holds what could not be read: `unread_regions` (Track N3, 2026-10-10)

Additive to v2, on `knowledge_answer`, `task_context` and
`/query/resolve_sections`. An equation the converter found and could not read
stays in a note as a marker (`[equation not recovered (p. 4, eq. 7); see the
source page]`, in its `## Source equations` and wherever an extraction carried
it), and a figure nobody has read stays a `## Figures` item without an
interpretation. When the evidence a response returns holds either:

| where | field |
| --- | --- |
| `unread_regions[]` | one entry per note of the evidence (at most four): `note`, `source`, `sections`, `unread_equations` (the source's count), `equation_ids` (held by the evidence, then the best-matching section's, then up to three best single matches; at most 20), `sections_matched` (headings matching the question, each with a `summary`, its ids, pages and its own `recover`), `matches` (each with `score`, `in_evidence`, `printed_label`, `heading_path`, the words before and after it), `pages`, `equations_without_ids`, `figure_numbers`, `uncaptioned_figures` (listed apart, not offered), `uncaptioned_offered` (Track AD2: figures with no caption offered because the headings over them or the text around them match the question; each with `number`, `page`, `heading_path`, `matched_words`, `why`; decorations never), `estimates`, `recover` (`route`, `body`: a dry run that works as given, `run_body`, `then`), `recover_all`, `recover_figures`; a note none of whose equations matches the question gets a compact entry (`unread_equations`, `equation_ids: []`, `why`, `recover_all`) |
| `warnings` | `evidence_has_unread_regions`, with a sentence in `warning_details.evidence_has_unread_regions`, only when an entry has something to read |

`unread_regions` is `[]` when there is none. Built from markers inside the
returned evidence (an `Equation` item counts when the ranking matched its own
words) and, for each note of the evidence, from its source's unread equations
whose headings or surrounding words match the question, words weighted by the
source's own paragraphs so ubiquitous ones weigh nothing (any language; N-R1,
N-R8..N-R11, 2026-10-10). A note's node label also carries the short names
its source defines for itself (`identity_terms`). The response spends
nothing and stays read-only: a caller that needs the regions sends the
`recover` body (a dry run), asks its user with the estimate, then sends
`run_body` (`maintenance_write`) and asks again. An equation read from its
image afterwards carries `image_derived: true` like a figure's reading.
`docs/equations_v1.md`.

## Consumer guidance
- Agents should branch first on `schema_version`, then on `response_type`.
- Agents may tolerate unknown additive fields.
- Agents should not infer response type from endpoint path alone; use `response_type`.
- Agents should treat a missing or unknown `schema_version` as a compatibility risk and fall back to provenance inspection.
- Agents should send a stable `agent_id` on every memory read so future workspace routing can remain additive rather than contract-breaking.
- Agents should treat the returned `scope` object as the effective governed boundary for the response, even when the current implementation is still single-workspace under the hood.
- Agents should treat the returned `policy` object as authoritative for allowed actions and should not assume read access implies refresh or write capability.
