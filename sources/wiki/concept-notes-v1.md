---
type: "documentation"
title: "Concept notes v1"
description: "One maintained Markdown note per concept a collection's notes name, kept up to date as documents arrive. For a person, it is the page a concept link opens — in Obsidian, clicking a concept in a source note opens it. For an agent, it is the..."
status: "stable"
reading_status: "read"
product: "second brain"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing second brain"
doc_path: "docs/concept_notes_v1.md"
doc_sha256: "f5a62f7f5536c64d7097d18a7619b175360dce2cd090b08aa8257e338e07be12"
---

# Concept notes v1

One maintained Markdown note per concept a collection's notes name, kept up to
date as documents arrive. For a person, it is the page a concept link opens —
in Obsidian, clicking a concept in a source note opens it. For an agent, it is
the collection's current understanding of an idea, with the source notes to
read for more, oldest first.

## Why

Every note already names the ideas it rests on, under `## Concepts`, and the
graph joins notes that name the same one. But the understanding itself was
scattered: each source defined an idea in its own glossary, and nothing said
what the idea means *in this collection*, how that understanding developed, or
where sources disagree. A question like "how has X changed over the years?"
had no starting point.

The layer is built on the one section every note type carries. That is what
makes it serve any collection: a meeting note, a recipe and a legal filing all
name their concepts there, and nothing in the layer knows or cares what kind
of document a source is.

## The concept is abstract; the sources are specific

`## TL;DR` and `## Definition` describe the idea itself: what it is, how it
works in general, what it covers and what it does not. They should stay true if
any one source were removed. The agent writes them from the sources' remarks
*and* its own knowledge of the idea, because a source often only names a concept
(a method used as a baseline, a term it assumes) without explaining it. One or two
sentences then say, in general terms, the sense in which this collection uses
it. A source's parameters, numbers, datasets, results, setup or baselines never
appear there; they go in `## Development`, `## Divergences` and `## Sources`,
attributed and limited to what the sources say.

This makes the definition the one part of the note that is not purely
provenance-backed, and the note says so: `definition_basis:
sources_and_general_knowledge` (agent) or `sources` (an assembled draft). General
knowledge never overrides a source; where the two conflict, the conflict is a
divergence.

## Broader and narrower concepts

Every concept note has a `## Hierarchy` section linking up to its broader
concepts and down to its narrower ones, with the keys mirrored in the `broader:`
and `narrower:` frontmatter. The hierarchy comes from three places:

- **declared** `broader` on a note: written by the agent when it creates or
  extends a concept (`broader` and `narrower` in the skill's output), or by a
  person editing the note's `- Broader:` line
- the agent's `narrower`, stated from the general side, which is applied to the
  narrower note
- **label containment**: a concept whose name's words are a strict subset of
  another's is broader than it. This is the same rule the graph's `BROADER_THAN`
  uses: free, and it holds in any collection. A trailing parenthetical such as an
  abbreviation does not count as a narrowing word.

Cycles are refused, and only direct parents are kept: when A is broader than B
and B than C, C lists B, not A. `narrower` is never declared, only derived, so
the two directions cannot disagree. A note's `broader:` reaches the graph as
`BROADER_THAN` edges owned by the note, which the derived-links pass never
withdraws. Label words are compared in any script.

When a question names a concept, the answer route's `concept_notes` lists the
direct matches and then up to `match_narrower_limit` of their narrower notes
(`matched_via: narrower`), widening only the most specific matches: a question
naming "mobile edge computing" also names "edge computing", and does not want
every other kind of it. A question about a general idea is usually answered
partly by its kinds, and they often share no words with it. The hierarchy is recomputed at the end of
every sync, and on demand with `POST /concepts/hierarchy` (no tokens). A change
to a note's broader concepts is recorded in its revision history.

## Recognising a name the collection already knows

A source that names a concept the collection already has under another
spelling joins the existing note; no second note is created. One alias index per
workspace holds every concept note's key, label and `aliases`, plus the names of
redirect stubs left by a merge, which lead to the note that absorbed them.

Names are compared **typographically only**: Unicode compatibility forms, case,
punctuation and hyphens, and a trailing parenthetical, which is also indexed as
an abbreviation when it is a single word. So `Mobile-edge computing`,
`Mobile Edge Computing (MEC)`, `MEC` and the slug `mobile-edge-computing-mec`
all reach one note. There is no stemming, no plural rule and no synonym list:
those are language- and subject-specific, and a wrong merge costs more than a
missed one. Accents are not folded: a key is spelled in ASCII, a name is
matched as it is written.

- A name that is already a live note's own key always means that note.
- With `resolve_aliases: false` a name still finds the note whose key or
  label it exactly is; only variants, aliases and abbreviations are off.
- A name two concepts could own (two notes glossed `(GF)`) is **ambiguous and
  never resolved**; the source gets a note of its own, as it would without the
  index.
- An alias the agent proposes is **refused** when it already names another
  concept (its key, label, an alias or an abbreviation) or two concepts already
  share it. Otherwise it would route that idea's sources here and make the two
  look like duplicates.

When a name resolves, the source's bullet links to the existing note under the
source's own wording (`- [[widget-store|Widget Store (WS)]]`), and its
`concepts:` frontmatter is rewritten to the resolved key. A slug there is joined
to the bullet it was made from, because only the bullet still carries the
parentheses. In the graph, a concept bullet linked to a key other than its own
name's is keyed by the link target, so both spellings are one `Concept` node.

Rewriting links changes a source note's text and therefore its hash. A
mechanical edit to a source that was up to date records the new hash, so it is
not re-read by the agent: re-reading costs a full pass over the document and
would learn nothing new.

## Notes their owner maintains

Two frontmatter keys, readable and writable in Obsidian, hand a note to its
owner:

- **`maintained_by: owner`**: the brain keeper never rewrites the TL;DR,
  Definition, Development, Divergences or Hierarchy, and never merges, renames
  or re-parents the note. Every write path checks it: the agent update, the
  no-token draft, condensation, the hierarchy refresh, merging and nightly
  curation. As a backstop, the writer itself re-reads the file on disk and
  refuses (audited as `refused`), so a note protected while a sync is running is
  still obeyed. The agent is never asked about a protected concept, so no
  tokens are spent on it.
- **`keep_separate: [<key>, …]`**: never merge this note with those keys and
  never propose the pair again. Dismissing a merge proposal in the review queue
  writes it automatically.

A protected note still gets one mechanical write: a new source that names the
concept is listed under `## Sources` (in date order among the existing lines)
and in `sources:`/`source_count`. Nothing else in the file changes, byte for
byte. There is no revision bump and no history line, though the replaced version
is still snapshotted. The owner is the note's synthesizer, so a source that
names only protected concepts is not left pending for the agent.

`GET /concepts` reports `owner_maintained: true` on such a note, with
`unabsorbed_sources`: sources listed while the note was protected that its own
text does not cite yet. Citing a source (`[[stem]]`) anywhere in the
definition, Development, Divergences or `## My notes` is how an owner marks it
read. `status.owner_maintained` lists every protected note with its unread
sources. `## My notes` stays the owner's on every note, protected or not.

In the hierarchy, a protected note's own `broader` is taken exactly as written:
nothing derived is added to it and nothing is removed, not even to break a cycle
(the unprotected side gives way). Other notes may still name it as their parent.

## Merging two notes for one idea

`POST /concepts/merge {keep, absorb, reason}` folds `absorb` into `keep`:

- **The kept note** gains the absorbed note's sources, its label and aliases (as
  aliases), related concepts, parents and `keep_separate` entries, and its
  `## My notes` text, moved under a "From [[absorb]]" line. With agent synthesis,
  the `merge_concepts` skill writes one summary, definition, Development and
  Divergences from both notes (the same rules as `maintain_concepts`: the
  definition is the idea, the record sections are the sources'). Without an
  agent, or if the call fails, nothing is lost and nothing is invented: the
  absorbed note's summary, definition and Development go into Development under
  `Merged from [[absorb|Label]]`, and its divergences into Divergences, both
  attributed. The revision history records `merged · absorbed [[absorb]]`.
- **Every source note** that named the absorbed concept is relinked in
  `## Concepts`, keeping its own wording (`[[keep|Absorbed label]]`), and in
  `concepts:`. This is a mechanical edit of one section and one field, and it
  keeps an up-to-date source up to date: it is not re-read by the agent. The
  edit's provenance is the merge record and the audit log, plus a
  `mechanical_edits` entry in the concept state; the note itself gains no
  bookkeeping field.
- **Other concept notes** follow: related concepts, hierarchy, and `[[absorb]]`
  links in their prose point at the kept note. An owner-maintained note is left
  exactly as it is; its links still land through the redirect. The hierarchy is
  then recomputed, and the merged pair cannot reappear as parent and child,
  because only one of them is still a concept.
- **The absorbed file becomes a redirect stub**: `type: concept_redirect` with
  `redirect_to: <keep>`, at the same path, so old links and bookmarks still land.
  It is never listed, matched, synced or ingested as a concept.
  `GET /concepts/<absorb>` answers with the kept note and `redirected_from`, and
  a new source naming the absorbed concept joins the kept note (see above).
- **The graph** loses the absorbed note and its `Concept` node at once; a
  rebuild gives one `Concept` node and one concept note. The maps of relinked
  source notes are redrawn (after an unmerge too), so they stop pointing at the
  absorbed key. Where maps live is the map module's decision: its
  `build_workspace_map` is used when it has one, then its `maps_dir()`, and only
  an older module without either gets `data/exports/maps`.
- **Undo.** Every file is snapshotted first under
  `maintenance/concept_merges/<merge_id>/`, with `merge.json` listing each path
  and its hash before and after. `POST /concepts/unmerge {merge_id}` restores
  them byte for byte, and puts the recorded hashes back so nothing becomes
  pending. It is refused when a file changed since the merge, because restoring
  would discard that edit, unless `force: true`.

A merge is **refused** (409, with the reason) when either note is
`maintained_by: owner`, either lists the other in `keep_separate`, either was
already merged away, or the keys are the same.

## How sources model an idea differently

Sources that use the same idea often disagree in *conditions* rather than in
results: what they count as part of it, what they assume or hold fixed, the
scale they apply it at, how they measure it. A source read alone never records
that, because it is only a difference once the sources sit side by side, and it
is often what a reader most wants.

For a concept with at least `compare_min_sources` (default 3) sources, the
`compare_concept_sources` skill reads them side by side and writes
`### How sources model it`, a subsection at the end of `## Divergences`. It is
organised by the dimensions along which the sources differ, groups sources that
model the idea alike, says where a source does not state a position, and cites
every source. Each source is represented by what the concept note records it
saying, every passage of it that names the concept (any section, any note
type), and the start of the sections where note types state conditions
(`compare_sections`). Rules that keep it honest:

- It is the collection's record: sources only, no general knowledge.
- Each source is given to the agent with a short id (`S1`, `S2`, …), and the
  agent cites `[[S3]]`. Ids are mapped back to the source's file stem before the
  note is written, so the agent never copies a long file name. A source's stem
  as spelled (`.md` and case aside) is accepted too. A citation of anything else,
  such as a work a source itself refers to or a guessed file name, is kept as
  plain text and reported under `unresolved_citations`; the rest of the
  comparison stands.
- A comparison that cites none of the concept's sources is rejected and nothing
  is written. The rejection is recorded, so the same sources are not compared
  again until one is added or edited.
- Sources that model the idea alike leave the note untouched (`no_difference`).
- The per-source pass is never shown the subsection, so it cannot drop it.
- Owner-maintained notes are never compared.

It runs in the sync that changes such a concept (`compare_on_sync`), nightly for
concepts whose sources changed, and on `POST /concepts/compare {keys?, force?}`.
A comparison is keyed by the concept's sources and the text each had, so it is
paid for once and repeated only when a source is added or edited. It needs the
agent; without one, nothing is written and the report says so.

**New sources are placed, not compared again** (Track X). Comparing every
source again whenever one was added made each ingest dearer the more sources
a concept had. Now the first comparison of a concept is written in full; a
later sync sends only the sources added since, with the comparison already
written (up to `compare_synthesis_chars` of it), and the agent returns a
*placement*: where each new source stands on its dimensions. Placements are
kept under the comparison, after the line
`_Placed against the comparison above as they arrived:_`. Where the earlier
sources model the idea alike (nothing was written), the latest few of them are
sent with the new ones instead. The whole comparison is redone only nightly or
on request (`force`), once the concept's sources reach `compare_refresh_factor`
times those it last compared in full: a comparison of `n` sources is paid once
per `n` sources added, so its cost per source stays flat. One full comparison
keeps its passages under `compare_max_chars_per_call` in all; past it, each
source's share shrinks. A comparison recorded before this change says nothing
of which sources it covered: a sync takes the sources it read as the new ones,
and a nightly pass compares such a concept in full once.

| key | default | meaning |
|---|---|---|
| `compare_min_sources` | `3` | sources a concept needs before it is compared |
| `compare_on_sync` | `true` | compare in the sync that changed the concept |
| `compare_max_chars_per_source` | `4000` | passages sent per source |
| `compare_max_chars_per_call` | `60000` | passages of one full comparison in all |
| `compare_refresh_factor` | `2` | redo the whole comparison (nightly, on request) once the sources reach this multiple of the last full one's |
| `compare_synthesis_chars` | `6000` | how much of the written comparison a placement is shown |
| `compare_max_per_pass` | `5` | comparisons per nightly pass; the rest wait for later nights (on demand and in a sync, every due concept is compared) |
| `compare_sections` | condition-stating sections of the shipped note types | read even where they do not name the concept |

## Open problems

What the sources say they did *not* do about an idea is where there is room for
new work, so a concept note can carry `## Open problems`, between
`## Divergences` and `## Hierarchy`. Each line is one limitation **any** source
note states that **names the concept**, attributed. A name is the label without
a trailing parenthetical, an alias, or a one-word abbreviation in parentheses,
each used only if the workspace's alias index resolves it to this concept alone
(a name two concepts share names neither; names under three characters are
ignored). Every source is read, not only those the concept lists, because a
source can state a limitation about an idea it never filed under its concepts:

```
- Each parcel locker is assumed to have free space at all times. — [[route-planning|Route Planning]]; similar in [[locker-handover|Locker Handover]]
- Only one parcel locker model was tested. — [[locker-handover|Locker Handover]]
```

- **Which bullets are limitations** is the note contract's decision: those
  under any section the source's type maps to the `Limitation` graph node
  (`## Assumptions & limits` for the shipped types). The pipeline's own
  placeholders (`- TODO`) never count.
- **Naming is the test**, not "the source is about the concept". A source about
  five concepts states limitations about one of them; listing every limitation
  under all five would bury the ones that matter.
- **A limitation several sources share is listed once**: one source's wording,
  attributed to it, then "similar in" the others. Grouped by the same rule as
  the gap engine's `limitation_cluster` (`second_brain/analytics/limitations.py`),
  most widely shared first.
- **Mechanical and free**, like `## Sources`: refreshed for every concept at the
  end of every sync that read a source (a source may name a concept it does not
  list), and on `POST /concepts/open_problems {keys?}`. Only notes whose section
  changed are written. It never bumps the revision and is never written by the agent; a
  note with nothing to list has no section.
- **An owner-maintained note is not written.** The refresh reports it under
  `owner_maintained`, so the owner can see what is waiting.

Across the whole collection, the same limitations feed `/query/gaps`
(`limitation_cluster`), where shared ones become candidate research directions.

| key | default | meaning |
|---|---|---|
| `open_problems` | `true` | write `## Open problems`; `false` stops writing it (existing sections are left as they are) |

## Ideas

Added 2026-10-09 (Track H1, `docs/ideas_v1.md`). What someone has *proposed*
about an idea is not what the sources show, so a concept note can carry
`## Ideas`, after `## Open problems`: every idea note whose `concepts`, `topic`
or `rests_on` resolves to this concept through the alias index, under a line
saying they are not findings.

```
## Ideas
_Ideas, not findings: what has been proposed about this concept, not what the sources show._
- [[proof-overnight-in-the-fridge|Proof overnight in the fridge]] — seed · owner
```

- Ordered `exploring`, `seed`, `parked`, `done`, then by title. A `dropped`
  idea is not listed. The link is the idea's file name, or its path in the
  vault when another note shares the name.
- **Mechanical and free**, like `## Open problems`: refreshed when an idea is
  captured (`POST /ideas`) or ingested, when a gap proposal is accepted, at the
  end of every sync that read a source, and on `POST /concepts/ideas {keys?}`.
  It never bumps the revision; a note with no ideas has no section.
- **An owner-maintained note is not written**, and is reported under
  `owner_maintained`.
- `GET /concepts/{key}` returns the lines as `ideas`.
- `config/ideas.json` `concept_sections: false` stops writing it.

## Your work

Added 2026-10-09 (Track K2, `docs/owner_work_v1.md`). The owner's own designs
and their commentary on other people's work are not the literature, so a
concept note can carry `## Your work`, after `## Ideas`: every owner-work note
whose `concepts` resolves to this concept through the alias index, under a line
saying it is never part of the synthesis.

```
## Your work
_Your own work, not the literature: what you wrote about this concept. Never part of the synthesis above._
- [[bedshare-how-it-works|Bedshare: how it works]] — design · draft
- [[rotaflow-how-it-works|Rotaflow: how it works]] — commentary on Okafor et al., GardenConf 2021 · draft
```

- Designs first, then by title. The link is the registration note's file name,
  or its path in the vault when another note shares the name.
- **Never part of the synthesis.** Concept sync reads the wiki's top level and
  skips `type: owner_work`, so an owner's note is never a concept's source and
  never feeds Definition, Development, Divergences or how sources model it.
- **Mechanical and free**, like `## Ideas`: refreshed when owner work is
  registered, read again or ingested, at the end of every sync that read a
  source, and on `POST /concepts/your_work {keys?}`. It never bumps the
  revision; a note with none has no section.
- **An owner-maintained concept note is not written**, and is reported under
  `owner_maintained`.
- `GET /concepts/{key}` returns the lines as `your_work`.
- `config/owner_work.json` `concept_sections: false` stops writing it.

## Nightly curation

Duplicates and missing hierarchy only show across the whole collection, so a
curation pass runs nightly (`scripts/nightly_maintenance.py`, through
`POST /concepts/curate`) for every workspace that already has concept notes.
Starting a concept layer means an agent pass over every source, so it stays the
owner's decision (`POST /concepts/sync`) and is never begun unattended. A pass
costs nothing when nothing is pending and no new candidate pair exists.

1. **Sync** sources still pending: a note added by hand, or a failed agent call.
2. **Candidates**, with no tokens (`second_brain/concepts/curation.py`). Four
   signals, each one that holds in any collection and language:
   - `same_name`: the labels are one name typographically
   - `alias_overlap`: one note's label is a name the other lists for itself,
     or both list the same *specific* alias. A short alias both list (`DP`) is
     not evidence, since abbreviations collide across unrelated ideas, and a
     name more than two notes claim counts for none of them
   - `contains_and_similar`: one label's words are a strict subset of the
     other's *and* their definitions largely agree
   - `shared_sources`: nearly the same sources, and the names share a word

   Definition similarity ignores words that most of the collection's
   definitions use. That stoplist comes from the collection, not from a
   language. Owner-maintained notes and `keep_separate` pairs are never
   candidates.
3. **Judgement.** `same_name` pairs need none. The rest go to the
   `curate_concepts` skill (`merge`, `keep_separate`, `make_hierarchy` or
   `unsure`, with reasons), in batches of `max_pairs_per_call`. A pair judged
   once is not sent again while its signals are unchanged.
4. **Act by policy** (`curation.merge`):
   - `auto_obvious` (default): merge only `same_name` pairs, and propose
     everything else
   - `propose`: merge nothing unattended
   - `auto`: also apply the agent's merges

   Every merge is reversible. A `make_hierarchy` judgement is applied, because
   it only adds a parent. `unsure` is proposed, and so is anything unjudged
   because no agent was available. A label that merely contains another is the
   shape of a hierarchy, so it is never merged without judgement.
5. **Compare** concepts whose sources changed (see above), at most
   `compare_max_per_pass` per pass. Rejected comparisons are listed in the
   report (`compare_rejected`, with the citations that failed).
6. **Report.** One line per pass in `maintenance/concept_curation.jsonl` (and the
   audit log). Every candidate lands in exactly one bucket, and `outcomes` names
   it per pair: `merged`, `proposed`, `hierarchy`, `hierarchy_already` (the
   judgement agrees with an existing parent), `kept_separate`, `refused` (with
   the merge refusal), or `skipped_merged_away`. A dry run reports
   `would_compare` bounded the way a live pass is, and `would_compare_due` in all.

Proposals are review-queue work items named `concept-merge:<a>+<b>`, carrying the
signals, the agent's reasons, and what resolving and dismissing mean. Through
`POST /review/action`:

- `resolve_work_item` merges the pair. A refused merge (say the note was
  protected since) is a 409, and the item stays open.
- `dismiss_work_item` writes `keep_separate` on both notes. It is written on the
  notes themselves, so the owner can see and undo it in Obsidian, and the pair
  is never proposed again, even if the queue's memory is lost.

| `curation` key | default | meaning |
|---|---|---|
| `merge` | `auto_obvious` | see above |
| `max_pairs_per_call` | `15` | pairs per `curate_concepts` call |
| `max_candidates` | `60` | candidates considered per pass, strongest first |
| `definition_similarity` | `0.5` | word overlap for `contains_and_similar` |
| `shared_source_overlap` / `min_shared_sources` | `0.8` / `2` | for `shared_sources` |
| `common_word_share` | `0.3` | words in more than this share of definitions are ignored |
| `sync_pending` | `true` | sync pending sources first |
| `retry_failed_calls` | `true` | a concept call that failed is tried once more in the same pass (below); off, it falls back at once |
| `retry_backoff_seconds` | `2.0` | seconds to wait before that second try (none after a reply that merely could not be read) |
| `retry_output_allowance_tokens` | `2000` | what a try is expected to cost when nothing better is known: the call's size in tokens plus this |

## How a concept note grows

The model is how a person learns a subject:

1. The first source to name a concept **creates** its note, scoped to what that
   source says. One document's view is a starting point.
2. Each later source is read against the current note. Per concept, the agent
   decides:
   - **extended**: the source adds something the note does not yet say. The
     affected sections are rewritten so they read as one account.
   - **diverges**: the source conflicts with what the note holds, or uses the
     term differently. Both views are kept, attributed, and written up in
     `## Divergences` as one coherent account of the disagreement. No winner is
     picked, and the definition stops stating as settled what is now disputed.
   - **unchanged**: the source only uses the concept. It is still recorded
     under `## Sources`.
3. Every change is appended to `## Revision history`, the version it replaced is
   kept in `maintenance/concept_history/<key>/`, and the write is recorded in the
   governance audit log.

One agent call covers up to `max_concepts_per_call` of a document's concepts;
a document naming more is split across calls. The skill declares a 600-second
floor on the call's time limit, because its answer rewrites several notes.

**What the call carries** (Track X). The new document whole, and for each
concept a view whose size does not depend on how many sources the collection
has: its summary and definition (kept short by condensing, below), its
Development and Divergences whole while they are under
`upkeep_section_chars`, else only their latest part
(`development_digest`, `divergences_digest`, with how much came before), its
latest `upkeep_recent_sources` sources with what each says and `source_count`,
and the owner's notes. For a section sent as its latest part, the agent returns
only what the new source adds (`development_add`, `divergences_add`), which is
appended: the story keeps every step, in the order sources were read. The other
concepts it may link to (`known_concepts`) are those the document names (by
label, alias or abbreviation), then the neighbours of its concepts, then the
most widely used, up to `max_known_concepts`, with `known_concepts_total` when
the list is cut. Condensing is sent the same bounded view.

### Keeping notes readable: consolidation

A note grows one source at a time, so its summary and definition accumulate: a
concept named by a dozen sources ends up with a definition that lists what each
said. Past `consolidate_definition_chars` (definition) or
`consolidate_summary_chars` (summary), the `consolidate_concept` skill rewrites
just those two fields as the concept itself (see above): abstract, free of any
one source's specifics, and keeping every scope limit and condition that
concerns the concept. The source-by-source
story stays in `## Development`, untouched. This runs at the end of every sync
for the notes that sync changed, and on demand through `POST
/concepts/consolidate`. A failed condensation leaves the note as it was, and
`GET /concepts` reports `status.needs_consolidation`. A note whose last change was
a condensation is not condensed again automatically until a new source changes
it, so a result that lands near the limit cannot cost a call on every sync.

## Where they live and what they hold

`sources/wiki/concepts/<key>.md`. `[[key]]` from any source resolves to the
concept note, and a source's `concepts:` frontmatter is rewritten to the keys
its bullets link.

**A new concept's key is plain ASCII** (Track AK1, the rule of
`note_format_contract_v1.md`, "File names"): `Größe` is `groesse`. A name with
a letter that has no Latin spelling is never guessed at: its key is
`concept-<10 hex digits>`, a hash of the name's own form, so the same name
always gets the same key and never an empty one; the label keeps its script.
Names are still *compared* in their own script (below), so two names that only
meet in ASCII (`año` and `ano`) stay two concepts: the second gets the first's
key with a short hash of its own name (`ano-1a2b3c`). Keys already written
(every concept file an earlier run named, whatever its script) are read and
linked exactly as they are, and a name finds such a note as before;
`POST /rename/ascii` (Track AP6, `docs/renames_v1.md`) renames them to this
rule, dry run first, each leaving a redirect stub at its old key and moving
its graph node, and `POST /rename` renames any one concept the same way
(`kind: concept`). Ingestion follows a stub, so a note naming a concept the
old way lands on the renamed note's node. In the graph a concept
named by its label is keyed by its note's key when the note exists, found
under the label's own form or, for a note under an ASCII key, by its title. They sit one level down so per-note scans that only make
sense for source notes (lint, facet assignment, maps) are unaffected. The graph
rebuild, link checking and the concept routes read both levels.

```
---
type: concept
title: <label>
concept: <key>
aliases: [...]
description: <one-sentence current understanding>
status: stable | draft          # draft = not yet synthesized by an agent
synthesis: brain_keeper | deterministic
definition_basis: sources_and_general_knowledge | sources
sources: [<source note stems, oldest first>]   # becomes DRAWS_ON edges
concepts: [<key>, <related keys>]
broader: [<keys>]   narrower: [<keys>]
source_count, first_seen, last_seen, revision, created, updated, generated
---
# <label>
## TL;DR            one sentence: what the idea is
## Definition       the idea itself, then the general sense this collection uses it in
## Development      how understanding developed across sources, in date order
## Divergences      where sources disagree, written as one account
  ### How sources model it   (optional) what each source counts, assumes, holds fixed
## Open problems    (optional) - <limitation naming it> — [[note|Title]]; similar in [[...]]
## Ideas            (optional) - [[idea|Title]] — <status> · <origin>   (not findings)
## Hierarchy        - Broader: [[...]]  /  - Narrower: [[...]]
## Sources          - <year> · [[note|Title]] — what this source says about it
## Concepts         - [[<key>|<label>]] and related concepts
## Key Concepts To Understand   - **<label>**: <summary>
## Revision history - <date> · <decision> · [[note]] — what changed and why
## My notes         yours; never rewritten
```

`## My notes` and its hint line are the same two constants every note the
pipeline writes uses for its owner's section (`OWNER_SECTION_HEADING` and
`OWNER_SECTION_HINT` in `notes/contract.py`, Track AH), so the two kinds of note
never drift. On a source note the section sits after the content and before
`## Links`, and nothing reads a claim from it
(`docs/note_format_contract_v1.md`, "The owner's section").

In the graph a concept note is a `Note` with `type: concept`. It `MENTIONS`
its own `Concept` node (the one every source mentions), carries a `Definition`,
and has a `DRAWS_ON` edge to each source note. `sources` is OKF's reserved key
for what a document derives from.

Source notes are linked too. Each `## Concepts` bullet becomes
`- [[key|Label]]`, both during `/sources/process` and when a sync reads an
older note. Ingestion reads a bullet that is wholly a link as its label, so
linking never renames a concept in the graph.

## How many concepts a document should name

Every concept becomes a concept note of its own, so extraction asks for **shared
ideas, not every term**: the ideas a document rests on that other documents
could also be about. That is usually three to six, rarely more than eight.
Terms only that document has, and passing mentions, are left out, because a
concept no other source names is a note that links nothing. The rule is stated
in every prompt and extraction skill that produces concepts, beside "one idea
per entry", and a guard test keeps it there.

What it changes is measurable, with no tokens: `GET /concepts` →
`status.usage`, or `scripts/concept_stats.py --workspace-root <path>`, reports
concepts per source (mean, median, max) and `single_source_share`, the share of
concepts only one source names. Measure before and after re-extracting a
collection.

## Up to date, checkably

`maintenance/concepts_state.json` records every source a concept has absorbed,
with the note's hash at the time. A source is **pending** when it is:

- `new` — never absorbed
- `changed` — edited since it was absorbed
- `not_synthesized` — absorbed without an agent while agent synthesis is on
- `concept_note_missing` — one of its concept notes was deleted

`GET /concepts` returns `status.up_to_date` and the pending list.
`POST /concepts/sync` absorbs pending sources **oldest first**, by the date the
source carries for itself (`year`, `publication_date`, `meeting_date`, …).
Order matters because each source is read against what came before it. When
a document was *imported* is not its date: a source with no date of its own is
`undated` and goes last.

## Configuration

`config/concepts.json`, overridden per workspace by
`<workspace>/config/concepts.json` (only the keys that differ):

| key | default | meaning |
|---|---|---|
| `enabled` | `true` | the whole layer, including linking source notes |
| `synthesis` | `agent` | `agent`: the brain-keeper writes and updates notes. `deterministic`: no tokens |
| `sync_on_ingest` | `true` | update concept notes as part of `/sources/process`; off, the upkeep is reported as deferred |
| `link_source_notes` | `true` | turn `## Concepts` bullets into links |
| `max_concepts_per_call` | `8` | larger documents are split across calls; each call re-sends the source |
| `max_source_chars` | `40000` | how much of the source the agent reads |
| `max_known_concepts` | `60` | how many existing concepts the agent is shown, for `related` and the hierarchy: those the document names first (was every concept, up to 400) |
| `upkeep_section_chars` | `1500` | a Development or Divergences longer than this is sent as its latest part, and grows by addition |
| `upkeep_recent_sources` | `5` | sources shown with what they say; the rest are counted |
| `consolidate_definition_chars` | `1500` | past this, the definition is condensed (0 disables) |
| `consolidate_summary_chars` | `300` | past this, the summary is condensed (0 disables) |
| `consolidate_definition_words` / `consolidate_summary_words` | `160` / `30` | the target lengths condensation asks for — kept well under the triggers above |
| `resolve_aliases` | `true` | link a name the collection already knows to its existing note (see above) |
| `match_narrower_limit` | `5` | narrower concepts listed after the direct matches in the answer route's `concept_notes` |

### Without tokens

`synthesis: deterministic`, or an agent that is unavailable or fails, falls
back to a note **assembled** from its sources: each source's own definition,
what it says about the concept, sorted by date. It is marked
`synthesis: deterministic` and `status: draft`, its narrative sections say they
are not synthesized, and it is rebuilt from all of its sources each time.

Two rules keep the fallback honest:

- It **never overwrites an agent-written note**. A source absorbed without the
  agent is only listed under `## Sources`, and stays pending.
- Drafts are pending (`not_synthesized`) as soon as agent synthesis is on again,
  so the next sync upgrades them.

A pass says so when it used the fallback while agent synthesis was on: see
"A pass that finishes what it starts", below.

## Routes

- `GET /concepts`: every concept, its synthesis mode and source count, and the
  layer's status
- `GET /concepts/{key}`: one concept, with its sources resolved to note paths
  and whether each is still present
- Both, and each entry of the answer route's `concept_notes`, carry
  `definition_basis` (added 2026-10-09, Track I2): `sources` when the summary
  and definition rest on the sources only (an assembled draft), or
  `sources_and_general_knowledge` when an agent wrote them from the sources and
  its own knowledge; `null` when the note records none (written by hand, or
  before the field existed). A definition with basis
  `sources_and_general_knowledge` contains general knowledge: cite
  `development`, `divergences` and the sources as findings, never the definition.
- `POST /concepts/sync` `{agent_id, workspace_id, notes?, limit?, wait?, dry_run?, approval?}`:
  backgrounded by default; `dry_run` lists the sources it would read and gives an
  `approval`, which a real run needs where the collection asks for one and its
  concepts are written by the AI (Track AW3); `notes` re-absorbs named sources whether or not they
  are pending. One pass for all of them (below); without `notes` it pays the
  whole backlog, deferred sources included
- `POST /concepts/hierarchy` `{agent_id, workspace_id}`: recompute broader and
  narrower concepts, no tokens
- `POST /concepts/ideas` `{agent_id, workspace_id, keys?}`: rewrite `## Ideas`
  (above), no tokens
- `POST /concepts/your_work` `{agent_id, workspace_id, keys?}`: rewrite `## Your work`
  (above), no tokens
- `POST /concepts/consolidate` `{agent_id, workspace_id, keys?, force?, wait?}`:
  condense notes past the limits, or the named ones with `force`
- `POST /concepts/merge` `{agent_id, workspace_id, keep, absorb, reason?, wait?}`:
  merge two notes (above); synchronous by default, so a refusal reaches the caller
- `POST /concepts/unmerge` `{agent_id, workspace_id, merge_id, force?}`: undo a merge
- `POST /concepts/compare` `{agent_id, workspace_id, keys?, force?, wait?}`: compare how
  sources model widely used concepts (above)
- `POST /concepts/curate` `{agent_id, workspace_id, dry_run?, wait?}`: one curation
  pass (above); `dry_run` neither writes nor calls the agent

## What upkeep costs, and when it is paid

Added 2026-10-12 (Track X). Measured on two live instances, the concept calls
an ingest triggers were a third or more of what intake cost, and they grew with
the collection: on one, from about 45k to about 190k tokens per page as its
concept layer grew. The same growth reproduces on a synthetic collection
(`tests/system/test_concept_upkeep_growth.py`): characters sent to concept calls
per new source rose from 14.8k at 5 sources to 94.8k at 50, because every
concept was sent whole with its growing source list and story, every concept
the collection had was listed, and every comparison sent every earlier source
again. With the bounded call above and placed comparisons, it is 14.0k at 5,
21.3k at 20 and 26.4k at 50, and levels off: nothing in a call is
proportional to the number of sources or concepts.

**One pass per batch.** A sync reads each source once, against what came
before it, in the sources' own date order; then what concerns the concepts —
the hierarchy, open problems, ideas, your work, condensing, comparing, the
graph — is done once over the union of the concepts the batch touched. A
folder run, a book's chapters and a split paper's sections each defer their
pieces and make one pass after the last; a batch gives the same concepts and
links as one pass per piece would (definitions may be worded differently).

**Deferring.** `/sources/process` with `concepts: defer` (and `process_book`,
`process_folder`) leaves the upkeep for later. The source is pending as any new
source is, marked `deferred: true` in `GET /concepts`; `POST /concepts/sync`
pays the backlog in one pass, and so does the nightly runner (its curation pass
syncs pending sources first, and it runs for a workspace with a deferred
backlog even before its first concept note exists). While the layer lags —
deferred sources, or pending ones in a workspace that has concept notes — the
governed answer warns `concept_layer_behind` and says how far behind
(`concept_layer`), so concept notes are never read as current without saying so.

**The cost reported is the cost paid.** The keeper records every concept call
in the call log against the run that caused it, with `for_sources` (the source
notes it was made for) and `concepts` (the keys it served). A processing
result's `cost.billed_tokens` is every call of its run, split into
`cost.extraction` and `cost.concept_upkeep`; when the upkeep is deferred,
`cost.concept_upkeep` says so and which pass pays it. A standalone sync is its
own run (`concept_sync`; a folder's `folder_concept_pass`, a book's
`book_concept_pass`). Intake estimates include each document's concept
upkeep, measured on this install from those records once
`min_history_runs` sources have it (`config/intake.json`), and a stated
default (`concept_sync_tokens_per_document`) before.

## A pass that finishes what it starts

Added 2026-10-22 (Track AN). Found on a live collection: one pass wrote two
concepts without the agent, and a retry afterwards cost 19 thousand tokens more.
The pass now retries inside itself and says plainly what it left.

**Tried again in the pass (AN1).** A `maintain_concepts` call that failed (the
backend failed, timed out, or its reply could not be read) is tried once more
in the same pass, after `retry_backoff_seconds`, before anything falls back.
Bounded to once. Held to the pass's budget: the `budget` the caller passes
(`POST /sources/add` and a folder run pass what the documents left of theirs),
else the collection's standing budget (`config/budget.json`, `GET /ops/budget`),
else no cap. The retry is not made when it would take the pass past that
(`retry_over_budget`: the estimate is what the failed call billed, else the
pass's average call, else the call's size in tokens plus
`retry_output_allowance_tokens`), nor when no agent could be reached at all (a
call that billed nothing and tried no backend has nothing to repeat), nor when
`retry_failed_calls` is off. Every attempt's billed tokens, a failed one
included, count in the pass's `billed_tokens` and against the budget, and are
in the call log.

**No agent is not a failure.** With `synthesis: deterministic`, no brain keeper,
or a brain keeper whose backends all resolve to the deterministic tier, nothing
is attempted: `agent_calls: 0`, `agent_failures: 0`, `agent_available` is false,
and `agent_off` says which (`synthesis_deterministic` is the designed path;
`no_agent` is a pass that wanted an agent and had none). Condensing and the
comparison on sync are skipped too. Only a real attempt that failed is a failure.

**Said plainly, never counted as done (AN2).** A concept written by the
fallback while agent synthesis is on is a draft: its source stays pending
(`not_synthesized`) for the agent, and the pass reports it in `left_for_agent`:

| field | meaning |
|---|---|
| `count`, `concepts` | each concept left: `concept` (key), `label`, `why` (a code), `why_text`, `sources` (the notes it waits on) |
| `why` | counts by reason: `no_agent` (none is set up), `call_failed_twice`, `call_failed` (retrying is off), `retry_over_budget`, `agent_unreachable` (no agent could be reached), `left_out_of_answer` (the agent answered but wrote nothing for it) |
| `retry_estimate_tokens`, `estimate_basis` | what finishing them would cost, and on what: the failed calls' own billed tokens or the pass's average call, and, for sources read with no agent, this install's measured concept upkeep per source (default `concept_sync_tokens_per_document`) |
| `sources` | the source notes still pending |
| `settled_by` | `system`: the next concept pass or the nightly runner reads them; nothing is asked of the user |
| `line` | the words for a person: which concepts, why, and what trying again would cost |

The shape is the same when nothing is left (`count: 0`, `line: null`). The
pass's report also carries `retries`, `retries_recovered`, `sources_finished`
(only sources the agent read, never those the fallback wrote) and
`sources_left_for_agent`; each processed source carries `left_for_agent` (concept
to reason) when any of its concepts waited. `POST /sources/add` puts `line` in
its report and warns `adding_concepts_left_for_agent`.

## Answering a question about a concept

For "how has X developed?", read `GET /concepts/{key}`: `development` and
`divergences` are the collection's current account, and `sources` lists, oldest
first, the notes to read for detail and to cite. The concept note is also a
retrievable note, so `POST /agent/knowledge/answer` can return it alongside the
sources it draws on.

## Limitations

- A source that **drops** a concept on re-processing is not removed from that
  concept note. It stays listed until the note is edited.
- A split paper's overview and sections are absorbed by one pass after its
  last section; the sections' own results report their upkeep as deferred to it.
- Past `upkeep_section_chars`, Development and Divergences grow by appended
  additions and are no longer rewritten as one account by the per-source pass;
  condensing rewrites only the summary and definition.
- A placement adds where new sources stand but does not reorganise the
  comparison; that waits for the next full comparison (nightly, once the
  sources have doubled, or `force`).
- Only the concept-writing call is retried in the pass. A condensing or a
  comparison that failed is counted as before and left for the next pass.
- A concept the agent answered for but left out (`left_out_of_answer`) is not
  retried in the pass: it is reported, and the next pass reads the source again.
- A source is still one `maintain_concepts` call (or more, past
  `max_concepts_per_call`): batching saves the per-concept work after the
  reads, not the reads themselves, and each call's fixed overhead remains.
- The quality of a note depends on the concepts extraction produces. Two
  names for one idea that differ only typographically now join one note at
  intake; two names that differ in wording still become two notes until they
  are merged.
- Concept notes are passage-indexed when written (if the workspace has an
  embedding service) and by `scripts/rebuild_vectors.py`. A redirect stub is
  not indexed, and a merge forgets the absorbed note's vectors.
- Graph ids keep letters in any script since A6. A graph built before that
  holds non-Latin concepts under `concept:unknown` (and Greek-language lines
  were dropped as formulas) until `scripts/rebuild_graph.py` is run.
