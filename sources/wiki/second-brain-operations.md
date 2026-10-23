---
type: "documentation"
title: "Second Brain Operations (Claude Code)"
description: "MANDATORY entry point for every second_brain interaction from Claude Code. Use before adding any source, answering any question from stored knowledge, reviewing or maintaining notes, or running any operational check. Defines which route to call for which intent, how to judge whether an answer can be trusted, and what must never be done directly."
status: "stable"
reading_status: "read"
product: "second brain"
product_version: "0.1.0"
doc_kind: "skill"
audience: "the AI program that works with second brain"
doc_path: "skills/claude-code/second-brain-operations/SKILL.md"
doc_sha256: "ef28e2abd482cc99026b6deab51e66eaca9cbd4ed1c02d9e25c49e0da94b975c"
---

# Second Brain Operations (Claude Code)

## The rule

**Load this skill before touching second_brain at all.** Every intake, every
question, every review, every operational check goes through the methodology
here. Do not invent an alternative way to use the platform.

This matters more for you than for most callers, because **your file tools work
on this repository**. Grep and Read will happily answer a question straight out
of `sources/wiki/`, and the answer will often look right. It is not the same
answer: it has no provenance, no confidence signal, no audit record, and it
silently skips the maintained-state layer that knows the note is stale or
duplicated. Use the API even when reading the file would be faster.

Reading a note file directly is right in one case only: **auditing**
pipeline output — checking a normalized note's provenance block, or judging
extraction quality. That is inspecting the machinery, not consuming knowledge.

If the instance has never been set up, or is failing health checks, use
`second-brain-onboarding` instead, then return here. For a question,
add `second-brain-research-answering`; for sources to take in, add
`second-brain-intake`.

The base URL is whatever the operator gave you; `http://127.0.0.1:8000` is
only the default, and the examples use it. Live route documentation: `/docs`.
A `curl` with a full `http://` URL and a JSON body needs nothing special on
any shell: send it as written, with nothing in front of it, so it still
matches a permission such as `Bash(curl:*)`. Git Bash on Windows rewrites only
an argument that is itself a bare route (`/concepts/<key>` on its own, as a
script might pass it); only for such a script set `MSYS_NO_PATHCONV=1` in its
environment.

## Coordinates — on every call, without exception

```
agent_id      who is asking
workspace_id  which instance
```

Both, every time. `agent_id` is **required** by the governed, review and
maintenance routes. `workspace_id` is **required by all of them**:

- omit it and the call is refused with `422`
- name one this instance does not know and the call is refused with `404`

Nothing is served against a default. If you get a `404`, list what exists
with `GET /ops/workspaces` rather than trying another spelling.

## Paths in API calls

Paths are resolved by the **service**, not by you. A note path in a request
or an answer is relative to the workspace (`sources/wiki/<note>.md`); a path
to a file to take in may be relative to the checkout
(`sandboxes/<workspace_id>/sources/raw/<file>`), `<root>:<path>` under a
source root the workspace declares, or absolute. A relative path is never
relative to your own working directory.

## The layers, which must never be collapsed

```
raw source -> converted Markdown -> templated Markdown -> normalized wiki
           -> ingested graph -> maintained/review layer -> retrieval
```

Markdown is the source of truth. Everything downstream is derived and must stay
rebuildable from it. You add knowledge at the **top** of that chain and consume
it at the **bottom**. Writing into a middle layer breaks provenance and the note
becomes unrebuildable.

## Choosing a route

| your intent | route | notes |
|---|---|---|
| add a source of any kind: a document, PDF, Word file, web page (by URL), image | `POST /sources/process` | **the** entry point; `kind` or `template_id` required; backgrounds by default; an unchanged source reuses its extraction (`refresh: true` reads it again) |
| add files, folders, links | `POST /sources/add` (plan first) | all in one request: `second-brain-intake` |
| add a book | `POST /sources/process_book` (`dry_run: true` first) | chapter notes; states pages and time; refused over the page ceiling |
| add Markdown you authored yourself | `POST /normalize/file` then `POST /ingest/file` | only when the Markdown is already an acceptable source of truth |
| answer a question from stored knowledge | `POST /agent/knowledge/answer` | governed, audited, provenance-backed; `view: compact`, `scope` |
| read what an answer's `focus` names, several sources in one call | `POST /agent/knowledge/read` | send `focus.read.body` as given |
| gather context for a task | `POST /agent/knowledge/task_context` | |
| decide what to read next | `POST /agent/knowledge/read_next` | |
| pull the actual text of matching sections | `POST /query/resolve_sections` | use before quoting anything |
| read a source's whole text in pages | `GET /review/source` with `line_start` and `line_end` | follow `next_line_start` |
| read the figures or images of a source by vision | `POST /sources/figures` (dry run first) | spends tokens: the user's decision |
| read the equations a question needs | `POST /sources/equations` (dry run first), `POST /sources/equations/rollback` | local formula model (PDF pages, free, minutes) or vision (tokens); an answer's `unread_regions` gives the exact call |
| read a source again with a better converter | `POST /sources/reread` (dry run first), `POST /sources/reread/rollback` | estimate first |
| explore the graph | `/query/retrieve`, `/query/related`, `/query/subgraph`, `/query/path` | exploration, not answers |
| find what is missing | `/query/gaps`, `/query/gaps/unanswered_questions`, `/query/priority` | |
| find where new work could start | `/query/gaps/limitation_clusters`, then `/query/gaps/interpret` with `categories: ["limitation_cluster"]` | limitations several sources share |
| what is open about one topic | `/query/gaps/topic` with `topic` (and `facets`) | an empty scope is a fact, not an error |
| keep watching topics for gaps | `/gaps/fill` (dry run first) | off until `config/gap_filler.json` enables it; only raises proposals |
| note an idea, or read the owner's | `POST /ideas` (`access_mode: maintenance_write`), `GET /ideas`, `GET /ideas/<key>` | never evidence; capture only when asked |
| register the owner's own notes (`notes`, `design`, `commentary`) | `POST /owner-work` (`access_mode: maintenance_write`), `GET /owner-work`, `GET /owner-work/<key>`, `POST /owner-work/refresh` | never evidence; the file is never modified; register only when asked |
| notes written by hand in the wiki | `GET /notes/native`, `POST /notes/native/extract` (dry run first) | part of the collection; the extract reads their concepts with an agent |
| concept notes | `GET /concepts`, `GET /concepts/<key>` | a synthesis: cite the sources it lists; `concept_layer_behind` means it lags |
| check note health | `GET /maintenance/lint` (`detail=true`) | `reference/intake-and-verification.md` |
| the review queue | `POST /review/queue`, `POST /review/action`, `GET /review/history` | |
| check notes against their sources | `POST /maintenance/fidelity` (dry run first) | free; a real run reads findings (budget) |
| propose a fix to one statement | `POST /maintenance/corrections` | a proposal, allowed `read_only` |
| find notes a fallback built | `POST /maintenance/fallback_notes` (dry run first) | no model call; offers re-extraction as a review item |
| rename anything; originals moved or deleted | `POST /rename`, `POST /sources/reconcile` (dry runs first) | never rename files yourself: `reference/sources-and-fidelity.md` |
| the instance's tail | `GET /tails`, `GET /tails/preview`, `POST /tails/draft`, `POST /tails/maintain` | approving is the owner's: `reference/tails.md` |
| re-derive notes after a source changed | `POST /maintenance/refresh` | |
| operational state | `/ops/status`, `/ops/self_check`, `/ops/integrity` | |
| prove nothing is billable | `GET /ops/billing_safety` | |
| see who did what | `GET /ops/governance_audit` | |

Never re-chain conversion and templating by hand: `/sources/process` does it.

## Adding knowledge

```bash
curl -s -X POST http://127.0.0.1:8000/sources/process \
  -H 'Content-Type: application/json' \
  -d '{"path":"sandboxes/<workspace_id>/sources/raw/<file>","kind":"<kind>","agent_id":"<agent_id>","workspace_id":"<workspace_id>","auto_ingest":true}'
```

One of `kind` or `template_id` is **required**: `GET /format-templates`
lists the shapes. `url` takes a web page instead of `path`. **This
backgrounds by default and returns a `job_id`.** Wait on it, always with
`wait`:

```bash
curl -s "http://127.0.0.1:8000/brain_keeper/jobs/<job_id>?wait=60&agent_id=<agent_id>&workspace_id=<workspace_id>"
```

Each poll is a whole turn that re-sends your context, so a busy loop costs far
more than the work it watches. `settled: false` with `job_still_running` means
call again with the same `job_id`. Never resubmit because something is slow:
a resubmit costs a second full pipeline and can leave two runs writing the
same note. Cancel with `POST /brain_keeper/jobs/<job_id>/cancel`.

**After any intake, verify rather than assume.** Open the note
(`GET /wiki/page?path=<note path>`) and check its provenance:
`template_runtime_mode: brain_keeper` (an agent drafted it) or, for a note
from `/normalize/file`, `mode: brain_keeper`; and `source_coverage: 100.00%`.
`mode: heuristic`, a `template_runtime_degraded_reason`, a
`fallback_extractor` mark, coverage below 100% or a `NOT PROCESSED` line mean
judgment did not happen or did not see everything: do not accept the note.
Long documents are processed in pieces and merged, **never silently
truncated**. The details: `reference/intake-and-verification.md`.

## Consuming knowledge

```bash
curl -s -X POST http://127.0.0.1:8000/agent/knowledge/answer \
  -H 'Content-Type: application/json' \
  -d '{"agent_id":"<agent_id>","workspace_id":"<workspace_id>","query":"...","view":"compact"}'
```

Then read what `focus` names in one call (`focus.read.body` to
`POST /agent/knowledge/read`). The procedure is
`second-brain-research-answering`. Access is **read-only by default** and
that default is correct: do not widen `access_mode` casually. A question not
asked before may cost one brain-keeper call to widen it onto the vocabulary
(`expansion.billed_tokens`); `"widen": "cache_only"` never spends.

**The single most important consumption rule:**

> A retrieved node label is a **title, not a finding**.

To state something as fact, read the underlying text (the read,
`POST /query/resolve_sections`, or `GET /review/source`) and quote from
there. Reporting a node label as an answer is how a graph label gets promoted
into a claim nobody ever made.

The owner's ideas (`related_ideas`) and registered work (`from_your_work`)
come apart from `knowledge_items` and are never findings. A concept note is a
synthesis: cite the sources it lists. Concepts and facets in detail:
`reference/concepts-and-facets.md`.

## Judging whether to trust an answer

Every governed response carries a `confidence` block:

| field | what it tells you |
|---|---|
| `overall_confidence` | the headline number |
| `provenance_strength` | how many source notes state the matches (one: 0.57, two: 0.78, three or more: 1.0) |
| `coverage_ratio`, `question_not_covered` | how much of the question the corpus can speak to |
| `freshness` | how old the supporting notes are |
| `weak_context` | **true means say so in your answer** |

Required behaviour:

- `weak_context: true`, `question_not_covered` or low `coverage_ratio` → say
  the corpus does not properly cover this. Do not fill the gap from your own
  knowledge and present it as the second brain's answer.
- Low `provenance_strength` → the evidence rests on one note: quote the source or do not assert the point.
- Nothing retrieved → **"the corpus does not contain this"** is a correct and
  useful answer. Never invent one.
- `unverified: true` on an item → the value was not traced cleanly; check the
  source lines. `image_derived: true` → read from a picture, weaker than text.
  `source_missing` → the original is gone.
- `unread_regions` → equations or figures the evidence holds unread; recover
  only when the question needs them and the user agrees (or a cap they stated
  covers the estimate).

Keep the boundary visible between what the second brain knows and what you
know. That boundary is the entire value of a provenance-backed system.

## Reviewing and maintaining

Health first (`GET /maintenance/lint`), then `POST /review/queue` for what
needs attention. Record decisions with `POST /review/action`. Valid
`action_type` values:

```
mark_noisy   mark_wrong   mark_duplicate   mark_drift
request_renormalization   resolve_review   apply_mutation_bundle
resolve_work_item   dismiss_work_item   escalate_work_item
```

One of `source_path`, `wiki_path`, `node_id` or `work_item_id` is required.
Work items carry their kind in their id (`correction:`, `fidelity:`,
`source-missing:`, `tail-draft:`, `owner-work-link:`, `gap-proposal:`, …);
resolving or dismissing one is the owner's decision unless they asked you to
act on it. Repair routes exist for specific damage:
`/maintenance/repair_frontmatter`, `/maintenance/repair_metadata`,
`/maintenance/repair_sections`. Use the narrow one. Facets are the system's.

**Never hand-edit a note to make a problem go away.** The review layer exists so
corrections are recorded, auditable and rebuildable. A manual edit is an
invisible, unattributed mutation that the next rebuild silently discards.

Sources renamed, moved or deleted; renaming anything; reading fidelity and corrections; re-reading
a source, its rollback and notes a fallback built:
`reference/sources-and-fidelity.md`. Instance tails: `reference/tails.md`.

## Connected over MCP

A harness may reach one workspace over MCP (`second-brain-mcp`; wiring in
`docs/mcp_server_v1.md`). It reads with `overview`, `recall`, `read`,
`entity`, `context_pack`, `delta`, and `synthesize`, which spends. With
`max_surface` `wider` in the owner's `config/mcp.json`, `request_tools` adds
`read_next`, `review_queue`, `codes`, `contract`, `capabilities`; with
`write`, `allow_proposals` and `--allow-proposals`, `remember` and `forget`
file review items you decide. An agent given manage (`set_access`) adds:
`plan_adding`, `start_adding` with its `approval` after the yes,
`adding_status`. Review and maintenance stay the owner's.

## Hard rules

1. **Markdown is the source of truth.** Derived state must stay rebuildable.
2. **Never read or write the SQLite store directly**, nor answer from
   `sources/wiki/` with Grep or Read (auditing aside). Use the API.
3. **Never hand-edit `sources/wiki/` to fix a problem.** Use review routes.
4. **Never present a node label as a finding.** Read the section.
5. **Never widen `access_mode` beyond what the task needs.** Read-only default.
6. **Never invent facet values.**
7. **Never accept a note with `mode: heuristic` or coverage below 100%** as
   properly ingested.
8. **Never let a metered API call happen.** All model work goes through the
   brain-keeper on an existing subscription. If `GET /ops/billing_safety` is not
   `safe: true`, stop and report it. Spending that subscription (vision, a
   re-read, a refresh, an agent extraction) stays within the user's standing
   budget (`GET /ops/budget`), dry run first, its `approval` sent back.
9. **Always send `agent_id` and `workspace_id`.**

## Warnings and refusals: follow their next step

Every warning in a response also arrives in `notices`, and every refusal of a
governed or intake route carries `error`, each with `fix.next` (what to do,
and the call with your own arguments) and `fix.verify` (a read that shows it
resolved). Act on `fix.next.action`, not on the wording of a message:

| action | you |
|---|---|
| `run` | make `fix.next.call` (supply anything listed in its `fill`) |
| `ask_user` | put the response's `question` to the user as written (only the budget is asked) |
| `tell_user` | tell the owner `fix.next.user_text`: the remedy is theirs |
| `wait` | wait on `fix.verify` with one waiting call; never resubmit |
| `retry` | try again later, once |
| `none` | nothing to do; the code describes the result |
| `report` | report it with the response |

`GET /agent/knowledge/codes` lists every code and its fix.

## When something looks wrong

1. `GET /ops/self_check` — instance health, converters, vision
2. `GET /brain_keeper/skills` — check `status.backend_readiness`; the `reason`
   field names the actual problem (for this backend, usually `claude` not on PATH)
3. `GET /maintenance/lint` — note-level defects
4. `GET /ops/integrity` — store and config integrity
5. `GET /ops/governance_audit` — what has actually been done to this instance

Report what the checks say; do not route around a failing component. A
fallback answer that looks normal is worse than an error: nothing downstream
can tell it apart from a real one.

## Open a reference only when a step needs it

| file | when |
|---|---|
| `reference/intake-and-verification.md` | after an intake: what the note's provenance must say; books |
| `reference/concepts-and-facets.md` | concept notes, the facet vocabulary, query widening |
| `reference/sources-and-fidelity.md` | renamed or deleted originals; fidelity and corrections; re-reading and rollback; notes a fallback built |
| `reference/tails.md` | the instance's tail: draft, approve, cadence, alias review |
