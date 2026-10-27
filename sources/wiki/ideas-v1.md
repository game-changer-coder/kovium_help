---
type: "documentation"
title: "Ideas v1"
description: "Track H1 of docs/knowledge_layer_improvement_plan_v1.md, built 2026-10-09."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing Kovium"
doc_path: "docs/ideas_v1.md"
doc_sha256: "fda2ce5826bf44658ce8ff59c971acdd9604c9c9883d72c3444c05039ea8397f"
---

# Ideas v1

Track H1 of `docs/knowledge_layer_improvement_plan_v1.md`, built 2026-10-09.

## Why

A brain holds two kinds of content that must never be confused: what the
sources say, and what someone thinks might be worth doing. An **idea** is the
second kind: an owner's hunch, a direction the gap filler proposed and the
owner accepted, something an agent suggested in conversation, a question to
come back to. A cook's "try this with less salt", a lawyer's "argument to
test" and a team's "option we have not costed" are the same thing, so nothing
in this layer knows what a collection is about.

Before this, the only home for an idea was the research-direction note an
accepted gap proposal wrote, or a source note, where it would be read as
evidence.

## The note

An idea is a Markdown note, and the note is the source of truth. It lives in
`sources/wiki/ideas/`, one level below the source notes like concept notes, so
per-note passes over sources (lint, maps, repairs, concept intake) never treat
it as one. It is recognised by `type: idea` wherever it sits: one dropped
beside the sources is still ingested as an idea, and lint skips it.

```
---
type: idea
title: Proof overnight in the fridge
description: Proofing the dough overnight in the fridge might open the crumb more.
status: seed            # seed | exploring | parked | dropped | done
origin: owner           # owner | agent | gap_filler | conversation
created: 2026-10-09
concepts: [Cold proofing]
rests_on: [crumb-basics]    # note stems: sources, concepts, other ideas
topic: bread                # optional
maintained_by: owner        # the default for an owner's idea
---

# Proof overnight in the fridge

## The idea
Proofing the dough overnight in the fridge might open the crumb more than a same-day proof.

## Why it might matter
Hydration alone did not explain the open crumb in the notes so far.

## What it rests on
- [[crumb-basics]]
- [[cold-proofing|Cold proofing]]

## What would test it
A side-by-side bake comparing an overnight cold proof with a same-day proof.

## Notes
<!-- Yours. Kovium never rewrites this section. -->
```

- **Frontmatter is the language-neutral contract.** `status`, `origin`,
  `created`, `concepts`, `rests_on`, `topic` and `maintained_by` are read the
  same in any language.
- **Headings are English by default, and an install adds its own** in
  `config/ideas.json` under `headings` (`statement`, `why`, `rests_on`, `test`,
  `notes`, each a list). Added headings extend the shipped ones; they never
  replace them, so a note written from the shipped template still reads. With
  no `## The idea`, the idea is its `description`, then its title. `_Not stated
  yet._` and the notes hint read as empty.
- **An owner's idea is the owner's** (A1). `origin: owner` with no
  `maintained_by` reads as `maintained_by: owner`. No pass rewrites any idea
  note. The brain links to ideas from concept notes, maps and answers, and
  raises review items about them, and that is all.

### Research directions, read as ideas

Notes the gap filler wrote before this (`sources/wiki/research-directions/`,
`type: research_direction`) are read as ideas **in place and unchanged**:
`origin: gap_filler`, `status: accepted` read as `seed` (`status_as_written`
keeps `accepted`), `## Direction` as the idea, `## The shared limitation` as
why it might matter, `sources:` as `rests_on`, `accepted_on` as `created`.
Nothing migrates them; an install that wants them in `ideas/` moves the files.
Every reader (the listing, `## Ideas`, map cards, `related_ideas`) takes
`status` and `origin` through one function, `ideas.notes.standing`: a missing
or blank origin is `gap_filler` for a research direction and `owner`
otherwise, a missing status is `seed`, and `origin_as_written` /
`status_as_written` keep what the note says.

Accepting a gap proposal now writes a `type: idea` note with `origin:
gap_filler` and `status: seed` into `sources/wiki/ideas/`. It records the work
item (`gap_proposal`), never the internal gap ids, which stay out of the vault.

## A separate epistemic tier

Ideas are never evidence.

- **Ingestion** gives an idea note one `Note` node, with the parsed idea on it
  (`metadata.idea`, `metadata.epistemic_tier: idea`), and two edge types no
  other note writes: `IDEA_ABOUT` to each concept in `concepts`, and
  `RESTS_ON` to each note in `rests_on` (a concept named there becomes
  `IDEA_ABOUT`). `topic` is kept on the node, not linked. The body is **not** walked, so no line of an idea
  becomes a Claim, Question or Concept of the collection.
- **Ranking skips idea notes** (`RetrievalService._rank_once`), so no idea is
  ever a knowledge item, a read-next suggestion, a resolved section, a
  synthesis input, or a topic's scope (`topic_scope.EXCLUDED_NOTE_TYPES`). The
  graph's `related` walk and centrality already ignore Note nodes.
- **The governed answer lists them apart**, under `related_ideas`
  (`docs/agent_memory_contract_v1.md`), so a caller can say "you had this idea"
  without citing it as a finding.
- **A graph rebuild** (`scripts/rebuild_graph.py`) re-ingests every idea note in
  `ideas/` and `research-directions/`, by type.

## Links back

- **Concept notes** gain `## Ideas` (`docs/concept_notes_v1.md`): every idea
  whose `concepts`, `topic` or `rests_on` resolves to the concept through the
  alias index, `dropped` ones left out, as `- [[idea|Title]] — status · origin`
  under a line saying these are not findings. Mechanical and free; refreshed when an
  idea is captured or ingested, at the end of every concept sync, and by
  `POST /concepts/ideas`. An owner-maintained concept note is reported, not
  written.
- **Paper maps** draw an `Ideas` branch: a card per idea resting on the paper
  (`- [[idea|Title]] · status`, edge labelled with the origin), linking the idea
  note by a name unique in the vault. Other map types do not draw ideas yet.
- **The gap filler** reports a gap an idea already addresses under the topic's
  `addressed_by_ideas` and does not propose it or send it to the agent.
  Addressed means named in the idea's `gap_ids`, or the idea rests on every
  source of the gap's members (two at least) and its text holds at least
  `tested_min_coverage` of their wording.

## Review

Raised into the one review queue, never acted on. Each is raised once: an item
the owner resolved or dismissed is never reopened.

| work item | raised when | resolving / dismissing means |
|---|---|---|
| `idea-tested:<idea>+<note>` | a source just ingested has a statement holding at least `tested_min_coverage` (0.5) of the rarity-weighted words of the idea's "what would test it" (the idea itself when no test is stated), and at least `tested_min_terms` (2) of them | looked at; the source does not bear on it. Nothing in either note changes |
| `idea-stale:<idea>` | a `seed` idea is older than `stale_seed_days` (60), from `created` or the file's last change | looked at; leave it as a seed |
| `idea-duplicate:<a>+<b>` | two ideas not dropped read alike: both hold at least `duplicate_min_similarity` (0.6) of each other's words | looked at; they are different ideas |

"Tested" checks the source's statements (its claims, results, limitations,
decisions and the like; not the names in `## Concepts`), only for ideas whose
status is `seed`, `exploring` or `parked`, and never against a note the idea
rests on. Matching is the strict, rarity-weighted word matching the answer's
question coverage uses (Track I-R1), so no tokens are spent. It runs after
`POST /ingest/file`, `POST /sources/process`, `/sources/process_book` and
`/sources/process_folder` for each note they ingest, and on request with
`POST /ideas/review`. Stale seeds and duplicates are checked only on request.

## Routes

- `GET /ideas` `{agent_id, workspace_id, status?, origin?, concept?}`: every idea,
  as `key`, `title`, `status`, `origin`, `path`, `created`, `concepts`,
  `rests_on`, `topic`, `maintained_by`, `type`, `read`. Audited (`ideas_list`).
- `GET /ideas/{key}`: one idea in full, with `statement`, `why`, `test`,
  `notes`, `rests_on` resolved (`note`, `path`, `present`), its open and closed
  `review_items`, and its `markdown`. 404 for an unknown key. Audited
  (`idea_read`).
- `POST /ideas` `{agent_id, workspace_id, access_mode: maintenance_write, title,
  statement, why?, test?, notes?, status?, origin?, concepts?, rests_on?,
  topic?, maintained_by?}`: capture. `access_mode` has no default: a request
  without it is 422. `origin` is `owner` (default), `agent` or `conversation`;
  `gap_filler` is written only by accepting a gap proposal. Writes a new note,
  never overwriting one (a name already used anywhere in the vault gets a
  number), ingests it, refreshes concept notes' `## Ideas`, and is audited
  (`ideas_create`, `idea_written`). A `rests_on` entry naming no note is kept and
  warned about (`rests_on_unresolved`). Returns 201 with `key`, `path`,
  `document_id`, `read`, `concept_notes_updated`.
- `POST /ideas/review` `{agent_id, workspace_id, access_mode: maintenance_write,
  notes?, all_sources?, stale?, duplicates?}`: raise review items. `notes` are
  source stems to check as newly added; `all_sources` checks every source in the
  graph against every idea. Writes only the review queue.
- `POST /concepts/ideas` `{keys?}`: rewrite `## Ideas` on every (or the named)
  concept note.

## Configuration: `config/ideas.json`

| key | default | meaning |
|---|---|---|
| `headings` | `{}` | extra headings per part, in the install's language |
| `stale_seed_days` | `60` | a seed older than this is raised; `0` turns it off |
| `tested_min_coverage` | `0.5` | the share of the test's weighted words one source statement must hold |
| `tested_min_terms` | `2` | and the fewest words it must hold |
| `duplicate_min_similarity` | `0.6` | two ideas this alike are raised; `0` turns it off |
| `concept_sections` | `true` | write `## Ideas` into concept notes |

## Limitations

- Matching splits words on spaces, so in a script written without them only
  whole runs match: such an idea is related by concept and `rests_on` only, and
  "tested" rarely fires. The answer's question coverage has the same limit.
- "Tested" reads a source's statements as the graph holds them, so a source
  whose sections produced no nodes is never matched.
- Stale seeds and near-duplicates are raised only by `POST /ideas/review`; the
  nightly job does not run it yet.
- A status change is an edit to the note. There is no route for it, by design:
  the note is the owner's.
- Only paper maps draw idea cards.
- The share of a question's words an idea must hold to be listed on words
  alone is fixed at 0.5 (the answer's own coverage floor), not configurable.
