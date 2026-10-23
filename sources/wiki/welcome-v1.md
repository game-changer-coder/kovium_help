---
type: "documentation"
title: "The welcome, the example, the tour and help (v1)"
description: "Track AG1 of first_run_experience_plan_v1.md (D2, D3, D3a, D3b, D3c). The contract for what a person new to second brain meets in their first five minutes, over HTTP and over the MCP server, with no command typed by them: a welcome that..."
status: "stable"
reading_status: "read"
product: "second brain"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing second brain"
doc_path: "docs/welcome_v1.md"
doc_sha256: "6a326af86e5213a01182d605ebfb4e4c03501ee1975480abdfb5a56fa7a90b71"
---

# The welcome, the example, the tour and help (v1)

Track AG1 of first_run_experience_plan_v1.md
(D2, D3, D3a, D3b, D3c). The contract for what a person new to second brain
meets in their first five minutes, over HTTP and over the MCP server, with no
command typed by them: a welcome that reads the install's state, a small
example collection that answers with its sources, a tour of what happens to a
document, and help about the software at any time.

Every sentence a person is shown comes from the catalogue
(`second_brain/catalogue/welcome_v1.json` and `options_v1.json`) or the user
guide ([[user-guide-v1|user_guide_v1.md]]); the code writes none of its own
(S4), and each passes the plain-words rule
([[plain-language-rules-v1|plain_language_rules_v1.md]], S5). What a response
adds beside them (`next`, `on`, `look`, `ask_with`) is for the agent.

## Routes

None needs a collection of the caller's: they answer an install that has none.

| route | does | spends |
|---|---|---|
| `GET /welcome` `{agent_id?, workspace_id?}` | the next step for this install (no `workspace_id`) or for the collection a connection is bound to | nothing |
| `GET /help` `{question}` | a question about second brain itself, answered from the full manual when it is fetched and describes this version (Track AG6), else from the user guide and the option catalogue | nothing |
| `POST /welcome/manual` `{update?}` | fetches the full manual (the help collection) onto this computer, on the person's yes; `update` replaces a copy of another version (Track AG6) | nothing |
| `POST /welcome/example` `{question?, agent_id?}` | opens the example collection (registered on first use, built by code), suggests three questions; with `question`, answers it from the example with sources | nothing |
| `POST /welcome/tour` `{step?, add?, approval?, reset?, wait?, agent_id?}` | one step of the tour; at step `add`, adds the waiting page; `reset` puts the example back | only `add: ai` with an `approval` |

### `GET /welcome`

Stateless: the state is read from the install each time, so a second session
resumes where the first left.

| `state` | when | `say` | `ask` |
|---|---|---|---|
| `new` | no `workspace_id`, the example not opened yet | what second brain does | see the example first? (`yes` recommended; `on.yes` is `POST /welcome/example`) |
| `example_open` | the example was opened | it is open, a collection of one's own can be made or opened from the conversation, help at any time; `questions` | the tour? (`yes` recommended; `on.yes` is `POST /welcome/tour`); once the tour was started, your own collection? (`yes` recommended; `on.yes` is `POST /collections`, `name` and `holds` filled from the person's words; Track AG2, [[collections-v1|collections_v1.md]]) |
| `collection_ready` | a bound collection that holds notes | how many notes it holds; help | none |
| `collection_empty` | a bound collection with no notes | it holds none yet, and how documents get in (or, for an agent that only reads, who adds them; Track AG2) | for an agent that may add (Track AG3): which files or folder (`adding.where`, `yes` recommended; `on.yes` the free plan, `fill: paths`) |
| `collection_adding` | a bound collection with a job running | documents are still being added | none |

`ask` is `{text, recommended, answers, on}`: the answers in order, recommended
first, each with what it does in plain words; `on` the call for each answer
that makes one. `help` is the call for a question about the software.
A connection bound to a collection, whichever it is (the example's working
copy included), is offered only what works there: what the collection holds,
asking it, and help; never the example or its tour (fix round R5).

### `GET /help`

The guide is cut into passages (a paragraph or a list item, each under its
`##` section), and each catalogue entry is one passage (its four lines and its
"later" line, with no label in front). The question's words are matched to
them by code, two forms of one word counted as one (a shared beginning of
three quarters of the shorter, four letters at least) and rarer words
counting more (a word of a passage's heading half again, a telling one
twice); at most three passages come back, best first, each with its `source`
(`{document: "user guide", section}` or `{document: "option catalogue",
entry}`).

**Found only when it is** (fix rounds R3, R8): words are matched by their stems
(two forms of one word are one word, a little less than the word itself; a
word of two letters only as itself, and weighing little), each weighted by its
rarity over the guide, the catalogue and the project's own documents in the
same language, this contract left out since it quotes the test questions (so the words every question uses weigh little; a word found
nowhere weighs a fixed amount, `UNKNOWN_WEIGHT`), and a word of a passage's
heading counts half again. A catalogue entry's own lines (what, need, cost,
recommended, later) are its passage. A passage answers only when it holds at
least `MIN_COVERAGE` (0.35) of the question's weight and half of its telling
words, its rarest (`MIN_TELLING`). Otherwise `found: false`, no passage, and
one plain sentence that the guide does not say. The answer's `help_collection`
says which copy of the manual stood behind it: `not fetched`, `fetched`,
`other_version` or `damaged`. The offer of the manual (below) is added only
when this install says where the manual is kept; with no address set nothing is
promised that could not be done (fix round R9, kept).

The thresholds were **chosen on** a fixed set of 33 real how-to questions
(`tests/fixtures/help_questions.py`): 25 the guide covers, several worded far
from it, and 8 it does not. On that set, as fitted: precision 24/25, 24 of the
25 covered answered from an expected section (missed: "How do I add a whole
folder of PDFs?"), 7 of the 8 others refused (answered wrongly as found: "Can
two people edit the same note at the same time?", from the entry on other
versions of one document). It is a fit to the set, not an independent measure.
The guide gained the common how-tos for it: adding files, folders and books;
capping or stopping spending; getting your notes out; removing a document;
working without the internet; what you need to run second brain (an AI program
you already use on a subscription, by kind and by name); what stays on your
computer; what happens when two notes disagree; and that the example and its
tour are for a first start. Ask in the catalogue's language (English); the
agent relays in the person's.

### `POST /welcome/example`

Opens the example collection (below) in its **working copy**: what ships is
copied into `sandboxes/welcome_example` under the instance root (made on
first use, refreshed when what ships changed), registered for the caller
(agent id `welcome` when none is given), then built by code when any of its
notes changed since it was last opened. The shipped example is only read
(fix round R1). `opened` reports `built`, `seconds`, `notes`,
`concept_pages` (the pages there now) and `maps` (every note's map now, the
tour's added page included). `folder` says where the paths are: the
collection folder relative to the install folder, every other path relative
to the collection folder; no machine's own path is ever shown (fix round R2). Without
`question`: `questions`, the three suggested questions (the catalogue's
`example_questions`), and `ask_with`, the call to ask one.

With `question`: the governed answer route (`POST /agent/knowledge/answer`,
no model call: `synthesize`, `judge_relevance` off, `widen: cache_only`)
answers it from the example, and `answer` lists at most four items, each
`{text, term?, source: {note, section, lines, file, original: {file, relative_to}}}` (the original in the working copy, relative to its folder: fix round R9; since Track AG2 every original is inside it): a note named
by its title, a definition's meaning as `text` with its `term` apart, and an
item that only names a note or a concept left out while others exist. When
the answer's `focus` says one note carries the evidence, that note's ranked
items come first, then its `## Claims`, so a page whose claims do not repeat
the question's words still answers it (fix round R4). When the answer
carries `question_not_covered`, the collection does not say: `found: false`,
`answer: []`, and the suggested questions again; never a guess. After a found
answer, `ask` offers the tour, once: not after the tour was started.

### `POST /welcome/tour`

Seven steps, in the order of D3b, each `{step, number, of, title, say, look,
next_step, next}`; the first also says the tour is never needed to use second
brain and can stop at any step.

Every path is relative to the collection folder that `folder` names, and a
note is `{title, file}` (fix round R2).

| step | `look` (for the agent to show) |
|---|---|
| `map` | `note` and `map`: the first note's map (a JSON Canvas file); `vault_folder`: the folder to open in Obsidian |
| `files` | `originals`: each original, `kept` (a sentence saying where it is kept) and its note |
| `notes` | `note`: the first note |
| `links` | `note` and `links`: its `## Links` lines, each link as its title |
| `concepts` | `concept_pages`: the example's concept pages by title, and their folder |
| `obsidian` | `vault_folder` and where to download Obsidian |
| `add` | `sample`: the waiting page, and whether it was `added` |

At step `add`, `ask` offers two ways, the cheaper first:

- `add: free` (recommended): the page, written as a note by hand, is copied
  into the notes and read by code as the person's own note: into the graph, its
  concept pages, its `## Links` and its neighbours', its map. `added`, `note`,
  `map`, `links`, `concept_pages`, then `ask_with`: the sample question
  (`sample_question` of the catalogue) asked of the example.
- `add: ai`: the AI reads the page and writes a note of its own, through
  `POST /sources/add`'s planner and runner. Without `approval`: the plan,
  `estimated_tokens`, `estimated_range`, `limit` (a one-time limit for this
  page alone: the high end of the range, rounded; never the example's
  standing budget), `approval`, and `ask` (recommended `no`, which is the free
  way). With the plan's own `approval` it runs, as a background job unless
  `wait: true`; a code from another plan, or a page that changed since, is
  refused before anything runs (`adding_plan_changed`,
  `adding_approval_expired`). After either way, a second add is
  `tour_sample_added` (409) or `added: false`.

`reset: true` (or `step: "reset"`; an unknown step's refusal names it) puts the example back as it was before the tour: the notes the
tour changed (neighbours whose `## Links` gained the page) restored from the
snapshot taken before the first add (`maintenance/tour_snapshot/`), the page's
note and map removed, the concept pages built again, the graph rebuilt.

**Measured** (2026-10-22, on a copy of the example, the AI program of this
install): the plan said about 31 thousand tokens (between 16 and 62 thousand,
the default rate, with no measured runs in that copy) and about 2 minutes; the
run used 23,310 tokens and took 41 seconds.

## The example collection

`sandboxes/hello_brain`, improved in place for the tour, never deleted. Its
subject is second brain itself; `example.json` (its manifest) lists:

- `originals`: each original its notes are built from, its kind, and the
  header `fields` its note records (facts the original states, written in by
  the build). Three are kept in the collection's own folder (`path`, under
  `sources/raw/`), one of them a copy of the user guide frozen at release
  time (`frozen_from: docs/user_guide_v1.md`). One is left where it lies, in a
  folder the collection points to by a name (`root_path:
  library:decision-every-answer-shows-its-sources.md`; `config/sources.json`
  declares `library` relative to the collection, so it travels with it), under
  `source_retention: reference`;
- `sample`: the page waiting in `inbox/` for the tour, written as a note by
  hand (documentation sections, a `## Links` section).

Its notes are of three kinds (documentation, a meeting note, a decision
record). Its settings have code assemble its concept pages
(`config/concepts.json`, `synthesis: deterministic`), so opening it and adding
the sample free never call the AI.

**Built by the maintainers, never by a user**:
`scripts/build_example_collection.py --check` (free) lists each original and
whether its note was built from its bytes as they are now (`current`,
`changed`, `no_note`, `no_original`); `--plan` gives the free plan; `--build`
builds what is stale with the install's AI program on the maintainer's
subscription (the code-only path when none is ready, said in the output), then
`--finish` writes the manifest's header facts and takes out the reading's lone
`TODO` placeholder lines and names the notes without the pipeline's
`__templated` suffix, links following (also free, and run by `--build`).
Measured on 2026-10-22: four notes, 106,788 tokens; the user guide's note
built again after the guide gained its how-tos, 37,689, and again after fix round 2, 49,979;
after Tracks AH and AG2, 55,709 and 58,637.

**The guide is a frozen copy (Track AG2).** The live `docs/user_guide_v1.md`
keeps changing with the project, and every change made the guard fail and cost
a rebuild of its note (above). The example's guide note is now built from
`sources/raw/user_guide_v1.md`, a byte-identical copy taken in with
`source_pipeline/reconcile.take_in_copy` (the notes repointed as a rename
repoints them, the old path kept in `previous_source_paths`; nothing built
again, nothing spent). Editing the live guide changes nothing the example
ships. `--check` reports the copy against the live guide as information
(`guide_copy.state`: `same` or `differs`), never as a failure. At release time
a maintainer runs `--refresh-guide` (copies the live guide in, free), then
`--plan` and `--build` for the guide's note (spends tokens). `help` reads the
live guide directly, not the example's copy.

**Derived by code, and read when current (Track AU3)**: opening it
(`second_brain/welcome/example.py` `build`, in the working copy) reads every
note into the graph as `scripts/rebuild_graph.py` does, has code write the
concept pages, and draws one map per note; it records the notes' fingerprints
(`data/staging/example_built.json`) and builds again only when one changed. It
writes no note: the shipped notes are byte for byte as committed after opening
(tested). Its maps (`sources/wiki/maps/`), concept pages
(`sources/wiki/concepts/`) and the tour's sample note are ignored by git. What
that first opening would build is made at release (`scripts/build_example_collection.py
--derive`, free) and ships in the example's `derived/` folder
(`second_brain/storage/derived_snapshot.py`): the graph's rows, the concept
pages, the maps and the concept layer's state, with the fingerprints of the
notes and of the code they were derived from. An opening reads it instead of
building while every note, the code and the data layout are the ones recorded
(`opened.how` is `read`, else `built`); a derived file changed in place is
never overwritten (the collection is built instead). Any change to the code
makes it stale until the next `--derive`; the example then builds as before.
`--check` reports it (`derived.current`).
`.gitattributes` keeps the example's files and the user guide at LF on every
platform, so a fingerprint taken on one platform holds on another.

**The rebuilt-note guard**: `tests/integration/test_example_collection.py`
fails when an original changed and its note was not built again, and when the
example stops opening cleanly or its notes stop passing lint. Editing the live
user guide no longer does (the copy above); refreshing the copy does, until its
note is built again.

## The full manual (the help collection, Track AG6)

Second brain's own current documentation, as a collection anyone can ask
(D3a). It is **a repository of its own** (`kbrain_help`), not part of this one,
so the download stays light; it is **offered, never required**; and **nothing
in this repository depends on it**: every test here passes without a copy.

**What it holds.** One note per current document: the contracts and guides
`docs/README.md` lists, the specification, the README, the architecture
overview, the four skills of the Claude Code framework and the option
catalogue written out as a document. Never the history (plans, results,
records), which describes what was intended or true once.

**Built by the maintainers, at no cost.** `scripts/build_help_collection.py
--out <folder>` builds it from this checkout; no agent is called and no token
is spent, by construction. Each document is taken in as a note **written by
hand** (its text unchanged under its own headings, the header facts it states
about itself, `type: documentation`), read by code the way any hand-written
note is. A link from one document to another becomes a note link (`[[note|words]]`);
a link to anything not in the collection keeps only its words; fenced code is
untouched. The graph, the concept pages and the maps are derived once by the
build, by code in a temporary copy (`example.derive`), and kept in the
collection's `derived/` folder, so a person's install reads them on the first
opening instead of building them (Track AU3: measured on a copy of this
manual, about two minutes became a few seconds); a copy whose notes or whose
second brain code differ from what was derived is built on opening, as for
the example. The build derives again only when the notes or the code changed,
and `manifest.size_mb` counts `derived/`. A tripwire on
the brain keeper fails the build if anything calls it (and its test installs
the same one); nothing was left out for needing the AI. Run again, it changes
nothing that did not change (it is idempotent, byte for byte). `--check`
(free) lists the documents changed, added or left since the last build and
whether the platform version moved (exit 1 unless current). `--record` writes
the built size into `config/help.json` (below).

**The manifest** (`manifest.json` at the collection's root) records
`describes.platform_version` (the version of second brain it was built from),
`describes.schema_version`, each source document's `sha256` and its note, the
note count, and the size (`size_bytes`, and `size_mb` rounded up to a tenth:
the notes and settings, which is what fetching costs). It carries no date, so
a build is reproducible.

**Where it is fetched from is a setting.** `config/help.json`, the install's
own (else the one this checkout ships): `repository` (a git address, or a
local path), `ref` (a branch or tag; empty means the tag `v<platform version>`
when the repository has one, else its newest state) and `size_mb`. The shipped
`repository` is a placeholder (`https://github.com/OWNER/kbrain_help.git`)
that the maintainers replace with the published address when the repository
exists; **while the placeholder stands nothing is offered and
`POST /welcome/manual` is refused `manual_not_set` (404)**, so no offer is made
that could not be done. `size_mb` is the measured figure `--record` wrote,
so the offer says "about N MB" from the manifest, never a guess; with none
recorded the offer says no figure.

**Offered twice.** At the tour's end, after the sample was added (step
`add`): the response's `ask` is the offer (`tour.offer_manual`, recommended
`yes`, `on.yes` is `POST /welcome/manual`), the earlier "put the example back"
offer moving to `also`. And in `GET /help`, when the manual is not fetched and
an address is set: a line (`help.manual_offer`) and the same `ask`. "No" ends
it; asking for the manual later does the same. With no address set the tour
keeps its old line (`tour.manual_pending`) and `ask` stays the reset offer.

**Fetching** (`POST /welcome/manual`, tool `get_manual`, free, nothing typed):
a `git clone --depth 1` of `repository` into `sandboxes/kbrain_help` under the
install's own folder for collections (never inside a tracked folder), made
beside its place and moved in only when whole, so a failed fetch changes
nothing. It is then registered under an agent of its own (`second-brain-help`),
never bound to the person's connection, and built by code on opening (graph,
concept pages, maps; seconds, no tokens). A current copy is left alone
(`fetched: false`, `manual.already`). Refusals, each with a plain sentence from
the catalogue: no address (`manual_not_set`, 404), no network
(`manual_no_network`, 502: git could not reach the host, or took more than ten
minutes), no `git` on the computer (`manual_no_git`, 424), anything else
(`manual_fetch_failed`, 502). A download of an archive instead of a clone is
not built.

**Never silently stale.** A copy whose manifest describes another platform
version is reported in plain words (`manual.other_version`, naming both
versions) by the fetch and by `help`; `help` then **answers from the guide and
catalogue that ship with this version**, not from the copy, adds that sentence
and an `ask` whose yes is `POST /welcome/manual` with `update: true`, which
replaces the copy with the one matching this version (the repository's tag
`v<version>` when it has one). An incomplete copy is reported the same way
(`manual.damaged`).

**A reading connection is never offered the fetch.** `get_manual` writes into the install, so a connection that only reads does not list it (the read-only invariant of the MCP server), and `GET /help` for an agent that does not manage its collection says in words (`help.manual_elsewhere`) that the manual can be fetched from a start without a collection or by an assistant that manages one, with no `ask`: never a dead end.

**`help` with the manual.** For a copy that describes this version, the
question goes to the governed answer route over the manual (`POST
/agent/knowledge/answer`, no model call, `synthesize` and `judge_relevance`
off, `widen: cache_only`, read only), under the manual's own workspace, never
the person's. `answer` lists at most four items as the example does, each
`source` adding `document`, the document of the checkout the note was made
from (`docs/welcome_v1.md`). When the manual does not cover the question, or
cannot be read, the guide and catalogue answer as before. Its answers are about
the software and are never evidence in an answer about the person's documents;
a person cannot add to it by accident, since adding goes to the collection of
the connection in use. When the guide also answers the question, its passages
come beside the manual's, as `in_the_guide`: the manual goes further, the guide
is tuned to plain how-tos.

**Measured** (2026-10-23, the real manual of this checkout, 46 notes and 1.3 MB,
on the machine that built it, the AI never called): fetching it by `git clone`
took under a second from a local copy; the first opening, which builds the
graph (about 10 thousand nodes) and 247 concept pages by code, took about 37
seconds; each question put to it through the governed answer route took about 22
seconds, most of it ranking the whole graph (four passes for a question of
several words) and computing graph summaries that do not depend on the question.
That is the platform's retrieval cost at this size, not a cost of the manual: a
faster path for a collection of ten thousand nodes is a follow-up. Answers are
the ordinary ranked items, not tuned to the manual (it is not a tuning target),
so a question the guide covers is often better answered by the guide's passage,
which is why both are returned.

**Not a tuning target.** It is a corpus this project owns, so it may be one more
evaluation collection beside the synthetic fixtures, never instead of them and
never what ranking is tuned to. The tests build a synthetic one
(`tests/fixtures/help_repository.py`) and fetch it from a local bare git
repository standing in for the remote.

## Over MCP: welcome mode

The MCP server ([[mcp-server-v1|mcp_server_v1.md]]) started without
`--workspace` (a fresh install's entry names none) is in **welcome mode**: no
collection, no registry checked, the tools `welcome`, `help`, `get_manual`,
`try_example` and `tour`, and since Track AG2 `create_collection`, `open_collection` (which
bind this connection to the collection they make or open) and `check_setup`
([[collections-v1|collections_v1.md]]), and nothing else; no grant but the
`manage` grant those two need, held over a local transport only, instructions that tell
the agent to call `welcome` for a newcomer and `help` for a question about the
software, and an audit of every call in the instance root's
`logs/mcp_welcome_audit.jsonl` (metadata only). A connection bound to a
collection offers `welcome` and `help` too, never `try_example` or `tour`: a
collection is a boundary (D2), and the example lives on the connection with
none. Each tool is the route above (rule 1 of the MCP plan); the calls a
response names (`ask.on`, `next`, `ask_with`) carry their tool call in the
compact view, so an agent follows them without knowing a route.

| tool | route | connection | read only | cost |
|---|---|---|---|---|
| `welcome` | `GET /welcome` | any | yes | free |
| `help` | `GET /help` | any | yes | free |
| `get_manual` | `POST /welcome/manual` | welcome mode, and a connection whose agent may manage (Track AG6); `manage` grant, never a reading connection | no: fetches the manual into the install's own folder | free |
| `try_example` | `POST /welcome/example` | welcome mode | no: builds the example's derived files | free |
| `tour` | `POST /welcome/tour` | welcome mode | no: adds or removes the example's sample | free, except `add: ai` with an approval |
| `create_collection`, `open_collection` | `POST /collections`, `POST /collections/open` | any (Track AG2) | no: makes or opens a collection, binds this connection | free |
| `check_setup` | `GET /collections/setup` | any (Track AG2) | yes | free |

## A kind for documentation

Found while sizing the help collection (D3c): no kind fitted a manual, and
guessing a file's kind from its headings read documentation as images, chats,
papers or decisions. Both are fixed for every install, through the type
profiles:

- **`documentation`** (template `format-templates/documentation.md`, profile
  in `second_brain/notes/contract.py`, prompt
  `prompts/documentation_remote_template_authoritative.md`, map
  `scripts/build_documentation_maps.py`): a page of documentation kept as a
  file. Sections `Purpose`, `How it works`, `How to use it`, `Settings and
  options` (both of these may be empty), `Known limits` (a limitation role);
  fields `product`, `product_version`, `doc_kind`, `audience`, `url`,
  `last_updated`. Its names: documentation, docs, manual, user guide,
  how-to, handbook, technical documentation, technical reference,
  specification, readme, instructions. `webpage` stays the kind for a page of
  the open web (its "documentation" names moved here).
- **Kind guessing** (`second_brain/source_pipeline/file_kinds.py`): a file's
  heading counts for a kind's section only when it holds every word of four
  letters or more of that section (two forms of one word are one word: their
  shared beginning is at least three quarters of the shorter), and a kind's
  headings count only when they are at least a quarter of the file's headings.
  Measured on this checkout's 39 current documents, with a default kind of
  `blog-post`: 13 were read as another kind before, none after. Files written
  in a kind's own shape are still recognised (the existing tests unchanged).

## The newcomer test, as measured here (S11)

Scripted (`tests/integration/test_welcome_mcp.py`, the agent following only
what the responses name, on a fresh install holding the example), on this
machine, 2026-10-22: the first example question answered with its sources
0.035 minutes (about 2 seconds) after the first call; commands typed by the
person: 0; questions put to the person before that answer: 1 (see the
example); tour offers: 2 (the tour, then how to add the page); tokens spent
before an approval: 0 (a tripwire on the agent); the sample added free, its
map drawn and a question about it answered 0.058 minutes after the first call.
Not measured here: a person (Codex and OpenClaw were measured by AG7, in
[[install-and-caller-agent-v1|install_and_caller_agent_v1.md]]). The install step on
a simulated clean machine was measured by AG4
([[install-and-caller-agent-v1|install_and_caller_agent_v1.md]]).

## Limitations

- Making a collection of one's own from the conversation is built since AG2
  (`create_collection`, [[collections-v1|collections_v1.md]]); adding
  documents to it from the conversation since AG3 (`plan_adding`,
  [[adding-documents-v1|adding_documents_v1.md]]).
- The tour's last offer is the full manual (AG6), one question that "no" ends; while this install's
  `config/help.json` still holds the placeholder address nothing is offered and the tour keeps a line saying it is
  not ready. After it, the tour's last line says that once the person has a collection of their own they can ask
  for their own second brain agent for it (AG4, `make_agent`): a sentence, not a question, since the agent belongs
  to a collection and the tour runs before there is one.
- The manual is fetched with `git`, not downloaded as an archive; with no `git` it says so. It is a copy of one
  published state: a person gets a newer one by asking again (`update`), never by itself.
- The help collection's size is measured from its manifest and recorded in `config/help.json` by the build; a
  checkout that was published before a build was recorded offers the manual without a figure.
- `help` is word matching over one language's texts: a question worded far
  from the guide finds nothing or something close; the agent may ask again in
  other words.
- The example's suggested questions are chosen to be answered well by the
  example under today's ranking; the answer test fails if a ranking change
  stops one being answered.
- Obsidian is not opened by the agent: the map's file and the folder to open
  as a vault are given; whether `obsidian://` can open a folder not yet a
  vault is unchecked.
