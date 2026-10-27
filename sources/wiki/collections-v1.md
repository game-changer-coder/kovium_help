---
type: "documentation"
title: "Collections and who may do what (v1)"
description: "Track AG2 of first_run_experience_plan_v1.md (D2, D4, D4a, D6). The contract for making a collection, opening one from another computer, connecting another agent program to it, saying what each agent may do, changing a choice, and checking..."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing Kovium"
doc_path: "docs/collections_v1.md"
doc_sha256: "0118718b3a1c6d5365567f3ee09e49e7b9635d1afaea86d17993bc849dec6b48"
---

# Collections and who may do what (v1)

Track AG2 of first_run_experience_plan_v1.md
(D2, D4, D4a, D6). The contract for making a collection, opening one from
another computer, connecting another agent program to it, saying what each
agent may do, changing a choice, and checking what works, over HTTP and over
the MCP server, with no command typed by the person.

Every sentence a person is shown comes from the catalogue
(`second_brain/catalogue/setup_v1.json`, `options_v1.json`, and the plain words
of each code in `second_brain/next_steps_codes.py`), and passes the plain-words
rule ([[plain-language-rules-v1|plain_language_rules_v1.md]], S4, S5). What a
response adds beside `say` (`workspace_id`, `access`, `travel`, `confirm`,
`next`) is for the agent. Nothing here spends a token. Code:
`second_brain/collection_setup/` (`service.py`, `access.py`, `settings.py`,
`programs.py`) and the routes in `second_brain/api/app.py`.

**Track AR's rule holds.** A person is asked only the standing token budget and
about deleting or overwriting their own originals. Who may do what is the
owner's own grant, set through `set_access` on the owner's word: a setting the
owner makes, never a question the system invents. Making a collection asks what
it will hold (the person's words are the answer) and takes the install's place
for collections unless the person names another folder.

## Routes

Each takes the shared fields (`agent_id`, `workspace_id`, `scope_kind`,
`scope_id`, `access_mode`). `workspace_id`, when given, is the collection the
caller is already bound to; a caller with no collection omits it.

| route | does | needs |
|---|---|---|
| `POST /collections` `{name, holds?, folder?, program?}` | makes a new, empty collection, registered; its maker is named its manager | from a bound caller: that collection's manage entry |
| `GET /collections` | the collections registered on this computer, by name, with what each holds, how many notes and where (Track BC1) | nothing; read only (over MCP, the local `manage` grant) |
| `POST /collections/open` `{folder \| collection, program?}` | opens an existing collection's folder (D6), or one already registered here by its name or id (Track BC1) | from a bound caller: that collection's manage entry |
| `POST /collections/join` `{collection \| folder, program?, agent?, project, read_only?, level?, length?, tone?, extra?, confirm?}` | this agent joins a collection under its own id, with a connection of its own in the narrowest place its program offers and its caller agent's text, in one plan the person approves (Track BC2 to BC4) | from a bound caller: that collection's manage entry |
| `POST /collections/leave` `{collection, program?, agent?, project}` | this agent's own connection, its caller agent's text and their record taken out; the collection stays (Track BC4) | from a bound caller: that collection's manage entry |
| `POST /collections/connect` `{program, agent?}` | adds this collection's own connection to another agent program | a manage entry that may `grant` |
| `POST /collections/access` `{agent, level, may?, per_run_tokens?, per_day_tokens?, confirm?}` | what one agent may do here: a preview, then applied with its `confirm` | see "Who may do what" |
| `POST /collections/settings` `{option, value?, add?, remove?, text?, confirm?}` | one choice of the catalogue, by its plain name: a preview, then applied | a manage entry that may change `settings` |
| `POST /collections/agent` `{project, program?, level?, length?, tone?, extra?, confirm?}` | the caller agent for this collection, added to the person's project beside its own agents, in the style they choose and in the format of the program (Claude Code's agent file, Codex's custom agent, a marked section of OpenClaw's `AGENTS.md`; Tracks AG4, AG7): a preview with the questions still open (`asks`), then applied with its `confirm`; see [[install-and-caller-agent-v1|install_and_caller_agent_v1.md]] | a manage entry that may change `settings` |
| `GET /collections/setup` `{probe?}` | what works on this computer, what is off or missing, and (with a collection) its token limit and who may do what | nothing; read only |

A refusal carries `error` with its code, its next step and, for a code a
person may meet, `fix.next.user_text`: its plain words. `GET /ops/workspaces`
answers with no `workspace_id` too (in strict scope): the registered ids and
their agents, never a folder, audited in the install's own
`logs/install_audit.jsonl`. It is the call the fix of `workspace_not_named`
names, which needed a workspace itself before this track.

### `POST /collections`

- `name` (required): a few words the person calls it, in any script. Its id is
  its letters without marks (`Recetas de mamá` is `recetas-de-mama`), else
  `collection`; never one already taken (`-2`, `-3`).
- `holds`: the person's sentence of what it will hold, kept as the
  collection's `description` and shown to every program that connects.
- `collection.folder` is shown, never spelled as the machine does (fix round
  R5): relative to the install folder, or to the home folder as `~/...`
  (`relative_to` says which), else as written; settings files keep the real
  path.
- **Where it goes is asked first (Track AZ1).** With no `folder` and no
  `place`, nothing is made: the answer is `{how: asked, created: false, say,
  ask}` with the code `collection_place_choice` (`ask_user`, settled by the
  user, `asks: place`). `ask.answers`, recommended first:
  - `working`: the folder the caller's AI program works in now (the project
    the person has open). The caller sends it as `working_folder` on every
    call, and the question offers it only when it is a folder on this
    computer outside the program's own folder (the checkout and the install
    root); recommended when offered;
  - `folder`: a folder the person names (recommended when there is no
    working folder); its call has `fill: [folder]`;
  - `default`: where the registry keeps collections (`sandboxes/` under the
    instance root). When that is inside the program's own folder the answer
    says so, and `say` adds one line why another place is safer: an update or
    a move of the program can put a collection there at risk.
  `ask.folders` holds the folder each answer means, shown as below, for the
  caller to show beside it (a path is never written into a sentence given
  word for word); `ask.on` holds each answer's call (`place` set, the name,
  `holds`, and the caller's own `agent_id`, `workspace_id` and `program`).
  The welcome's offer of a first collection makes the same call, with
  `working_folder` to fill, so the first collection goes through the same
  question.
- `place`: the person's answer. `working` needs `working_folder`
  (`collection_working_folder_missing`; one inside the program's own folder
  is refused, `collection_folder_in_program`); `folder` needs `folder`
  (`collection_folder_missing`); `default` keeps the registry's place.
- `folder`: the person's named folder, in full (a relative path is refused,
  `collection_folder_not_absolute`); given, it is used whatever `place` says.
  An empty or new folder is used as it is; one that already holds other files
  gets a folder of its own inside it, so the person's files are never mixed
  in (`kept: inside`; the working folder too); one that already holds a
  collection is refused with the call that opens it
  (`collection_exists_there`); one inside another collection, or holding one,
  is refused (`collection_folder_overlaps`). In the default place,
  `collection.folder` is relative to the install folder.
- The collection's `config/mcp.json` gets `collection_id`, `display_name`,
  `description` and its policy naming the caller `manage` with every ability
  and no limit (D4a).
- From a caller already bound to another collection, `program` adds the new
  collection's own connection to that agent program (one connection per
  collection, D2); without it the answer says how to get one.

### `POST /collections/open`

- `folder` (in full): a folder holding a collection (its `config/`
  and its notes or its database); otherwise `collection_folder_not_a_collection`.
  Or `collection` (Track BC1): one already registered here, by its name or id in
  the person's words, found as `GET /collections` finds it below
  (`collection_name_unknown`, `collection_name_ambiguous`). One of the two is
  needed (`collection_name_missing`). Opening binds a welcome connection that
  other agents may share; `POST /collections/join` connects one agent alone.
- Registered here unless a registered collection already lives there; its id
  is the `collection_id` it records when free, else one made from its name.
- This agent is registered for it, at the access the collection gives it: an
  agent its policy names keeps that; one it does not name reads only. From a
  caller with no collection, a collection whose policy names no manager that
  may grant (made before this track, or by a script) is managed by the agent
  that opens it (`access.claimed`): opening it there is the owner's act, as
  making one is. A caller already bound never claims.
- **What does not travel** (`travel`, each also a sentence in `say`):
  `folders_missing` (named folders for originals that are not on this
  computer, or written for another system: the person says where they are, and
  `change_setting` "Named folders for your originals" points the name here);
  `search_built_elsewhere` (vectors written by another model than this
  computer's setting: some notes may be missed until indexed again here);
  `program` (the AI program the collection names is not here, and which one
  works instead, or none). A database written by an older version is brought
  up to date by the store when it is opened, as always; nothing is said of it
  yet.
- Nothing is added again. `collection.notes` is how many notes it holds;
  `collection.folder` is shown as for `POST /collections`.

### An empty collection (fix round R1)

One message, the catalogue's, wherever an empty collection is met (a new
collection's `say`, the welcome's `collection_empty` state, the overview's
notice), chosen by what the calling agent may do:

- an agent that may add documents (its manage entry says `add`, or the policy
  names no manager yet): `collection_empty`, whose words say documents are
  added from the conversation, the plan and its cost shown first (Track AG3:
  `plan_adding`, `start_adding` over MCP; `POST /sources/add` over HTTP). The
  welcome's `collection_empty` state then asks the one question "which files
  or folder?" (`adding.where`), its yes the free plan with the person's
  `paths` filled ([[adding-documents-v1|adding_documents_v1.md]]);
- an agent that only reads: `collection_empty_read_only`, saying another
  program the person lets add documents has to add them.

Neither says adding is "the owner's": the caller may be the owner's own
managing agent.

### `POST /collections/connect`

`program`: Claude Code, Codex, Cursor or OpenClaw, in a person's words.
`agent`: the id the new connection uses, by default the program's own name
(`codex`); the caller's own id is refused (`connect_same_agent`), since the
new connection would share its access. The agent is registered and named in
the policy at `read`, unless the policy already names it (a grant made before
the folder travelled is kept). The entry, named `kovium-<collection id>`,
starts this install's server with `--workspace`, `--agent`, `--root` and
`--allow-proposals` (only the launch's half of proposals; the policy decides):

| program | how the entry is added |
|---|---|
| Claude Code | its own `claude mcp add --transport stdio --scope user <name> -- <command>` |
| Codex | a `[mcp_servers.<name>]` table appended to `config.toml` (`$CODEX_HOME`, else `~/.codex`), with `startup_timeout_sec` 30 and `tool_timeout_sec` 600 |
| Cursor | an entry merged into `~/.cursor/mcp.json` |
| OpenClaw | its own `openclaw mcp set <name> <entry>`, with `requestTimeoutMs` 600000, after `openclaw mcp show <name> --json` (Track AG7: `set` replaces an entry of that name silently, so a different entry is left alone and the same one is `already`) |

A program counts as installed when its command is found, or, for Codex and
Cursor, when its settings folder exists; otherwise `program_not_found`. An
entry of that name that starts something else is left alone
(`program_entry_differs`); the same entry again is `already`. A program
whose command cannot be started is refused `program_not_started`, in words
saying to check it is installed, never an error's name (fix round R4); one that
started and was still running when the wait (`TIMEOUT_SECONDS`, 60) ran out is
refused `program_timed_out`, saying it may have finished and to ask again
(Track AY2). A command runs
with every metered credential removed from its environment and calls no model.

## Who may do what (D4a)

Three levels, per agent and per collection, each including the one before:
`read` (the read tools; every agent the policy does not name), `propose`
(also `remember` and `forget`, through review), `manage` (also the setup
tools). A manage entry says what it `may` do: `add` documents, change
`settings`, `grant` other agents; and the most one addition, and one day, may
be approved to spend (`per_run_tokens`, `per_day_tokens`; `null`: no cap,
every addition still approved through its own plan). Since AG3 the caps are
enforced at approval: a plan never proposes more than the agent may be
approved for, and a run above the cap now is refused `adding_over_agent_limit`
([[adding-documents-v1|adding_documents_v1.md]], "The caps of an agent's
access").

The policy is the `agents` table of the collection's own `config/mcp.json`,
beside the ceiling that already grants proposals, read on every call and kept
in the folder, so it travels:

```json
{"collection_id": "shared", "display_name": "Shared", "max_surface": "default",
 "agents": {"welcome": {"level": "manage", "may": ["add", "settings", "grant"],
                        "per_run_tokens": null, "per_day_tokens": null,
                        "set_by": "welcome", "set_at": "...", "how": "created"},
            "codex": {"level": "read", "set_by": "welcome", "set_at": "...", "how": "connected"}}}
```

An agent named at `propose` or `manage` has the owner's half of the proposal
grant and may reach the `write` surface, whatever the collection-wide keys say
(`surfaces.read_ceiling(root, agent)`); the proposal route reads it the same
way.

**`set_access`.** `agent` is an id or a program's name. Without `confirm` the
answer is the change as one sentence (`access.will_be`) and a `confirm` code;
with that code it is written, audited, and said (`access.applied`). A preview
says `would_change`; `changed` is said only once a change was applied (fix
round R2). The code is
a digest of the collection, the agent, the entry it replaces and the new one,
valid an hour: a change, or a change to what it replaces, makes it stale
(`confirm_code_stale`). Like a plan's approval it is not a secret; it proves
the sentence was made and shown, not that a person read it. The rules:

- **an agent never raises its own access** (`access_self_raise`): a higher
  level, an ability more, or a cap lifted or raised; it may narrow its own;
- changing another agent's access needs a manage entry that may `grant`
  (`access_not_granted`, naming who may);
- a manage entry with no `may` gets `add` and `settings`, never `grant`.

**What it cannot do, said plainly.** While a collection's access tokens are
off (the default for every collection), the platform has no authentication
there. Over MCP the connection says which agent it is, and the policy binds
it. Over HTTP any program on the computer can claim any agent's name: the
routes apply the same rules to the agent a call names, which binds only an
honest caller; the guard over HTTP is the approval of each plan (Track AR,
AG3). **With tokens on** (Track BB, `require_token`, the catalogue's
`access_keys`; [[access-tokens-v1|`access_tokens_v1.md`]]), a caller proves
its agent by holding that agent's token for this collection, over MCP
(`sign_in`) and over HTTP (`Authorization: Bearer`), and the policy binds the
token's agent: a token at `read` cannot propose, and an entry lowered acts at
the holder's next call. Tokens are issued by `scripts/access_token.py`, by a
manage entry that may grant (`issue_token`, shown as one sentence with a
`confirm` code like `set_access`), and by a join. The honest limit stays: on
one computer under one login, an agent that can read any file can read
another agent's key file.

## `change_setting` (S6)

`option` is the choice's plain name as the catalogue and `help` give it, its
id, or a setting it holds (`contents_headings`, `books.ordinal_words`). A name
it does not know is refused `setting_unknown`, its `detail` naming the closest
choices (by the letter-stem rule `help` matches with) and every choice this call
changes; `check_setup` lists them too (`choices`). Without `confirm`, the
change as one sentence (each table entry with what it will be, a folder shown
from the home folder) and a `confirm` code, with `would_change` (no code when
nothing would change); with it, the change is written into the
collection's own `config/<file>.json` (which every loader reads over the
install's), audited, and the collection's services are opened afresh at the
next call, so the change applies from then on. What it reaches, each tested to
be read back by its own loader:

| choice | takes | written |
|---|---|---|
| Reading scanned pages | `value` on or off | `conversion.json` `ocr` |
| Equations in PDFs | on or off | `conversion.json` `formula_enrichment` |
| Splitting long documents | on or off | `books.json` `agent_split` |
| Search by meaning | on or off | `embeddings.json` `enabled` |
| Matching your wording | on or off | `semantic_expansion.json` `enabled` |
| A second check of numbers | on or off | `reading_fidelity.json` `judge_findings` |
| How concept pages are written | `ai` or `code` | `concepts.json` `synthesis` |
| Name of your collection | `value` the name, `text` the sentence | `mcp.json` `display_name`, `description` |
| Words for a table of contents | `add`, `remove`: words | `books.json` `contents_headings` (added to the install's list; `-word` takes one out) |
| Numbers written as words | `add` as `word: number`, `remove` by word | `books.json` `ordinal_words` (laid over the install's) |
| Words for measured quantities | `add` as `word: unit`, `remove` by word | `reading_fidelity.json` `quantities` (written whole: it replaces the install's) |
| Named folders for your originals | `add` as `name: full path` (a folder that exists here), `remove` by name | `sources.json` `source_roots` |
| Keeping copies of your originals | `value` `keep` or `point`; also the answer to an addition's question (Track AZ4, [[adding-documents-v1|adding_documents_v1.md]]) | `sources.json` `source_retention` (`copy`, `reference`; a collection made from the conversation starts at `ask`, read as `copy`) |
| Better PDF reader | `value` on or off; on only once its program is installed here (`setting_needs_program` otherwise): the yes to a plan's need (Track AG3) | `conversion.json` `pdf_converter` (`docling`, or null) |
| Asking before anything is spent | `value` on or off | `intake.json` `approval_required` (Track AG3) |
| Nightly tidy | `value` on or off. On means on **here**: the schedule names this computer, this computer's scheduler is asked to start it, and a collection that never decided keeps its AI steps off until a level is chosen. Turning it on at a computer other than the one named moves it, and the preview says so first | `maintenance.json` `enabled`, `machine`, `time` (Track AG5) |
| Nightly AI level | `value` a level: `light`, `medium`, `heavy` or `off` (`none`, `no`, `0`); or a count of tokens: `200000`, `200k`, `200 thousand`, `1.5 million`, which is that level when it is a level's most and `custom` otherwise. Refused in the example (`setting_example_no_ai`) | `maintenance.json` `ai_tokens_per_night` and `ai_level` (Tracks AG5, AX) |
| Nightly reading of equations | `value` minutes a night, `0` for none (at most 1,440) | `maintenance.json` `equations_minutes` (Track AG5) |
| A tidy view in Obsidian | `value` on or off; turned on again, the view is written at once ([[obsidian-view-v1|obsidian_view_v1.md]]) | `obsidian_view.json` `enabled` (Track BE) |
| Simple or technical answers | `value` `simple` or `technical` | `style.json` `level` (Track AG4) |
| Short or detailed answers | `value` `short` or `detailed` | `style.json` `length` |
| Tone and language of answers | `value` a line in the person's words; empty clears it | `style.json` `tone` |
| Anything else about how answers sound | `value` a line in the person's words; empty clears it | `style.json` `extra` |

The last four are the collection's answer style (D9): applying one also
rewrites every caller agent of the collection recorded on this computer that
is still its own, and `say` adds that it was rewritten; see
[[install-and-caller-agent-v1|install_and_caller_agent_v1.md]].

The three word lists were AG0's settings that still needed a file edited by
hand (a contents heading, number words, units); they are catalogue entries now.
Every other option says how it is changed instead
(`setting_not_changed_here`, `settings.NOT_HERE`): the token limit for adding
documents is the budget question, who may do what is `set_access`, two need a
program installed first, and the rest are not reached from the conversation
yet; a test pins the list, so it can only shrink. The nightly tidy was on that
list until Track AG5, because turning it on must also be started on this
computer; it is reached now.

## `check_setup` (S13)

`items`: one per feature, `{option, name, state, say}`, `state` one of
`works`, `on`, `off`, `missing`: the AI program that does the work (and which),
the better PDF reader, the reader for Word files, web pages and e-books, search
by meaning (probing the local search program unless `probe: false`), reading
scanned pages; with a collection also reading pictures, hints about the
collection, the nightly tidy, and (Track AG3, D4) whether a plan's approval is
asked for before anything is spent: said once, on or off, as its own item. The
better PDF reader `works` only when it is installed and chosen for the
collection (`off` when installed but not chosen), as the plan judges it. With
no standing token limit, the line says the first plan asks for one, or, once an
addition ran on a one-time budget, that each addition asks first. `off` lists the ones off or missing; an off or
missing one's sentence carries the catalogue's "do you need it" line. With a
collection, `budget_tokens_per_run` and `access` (one sentence per agent the
policy names, this caller as "This AI program"), and `choices`, the names
`change_setting` takes. Track AG5 (S13) adds what is off and what it costs: every
feature that spends tokens and ships off (reading pictures, hints about the
collection, suggestions of what to read next, the nightly AI steps) is listed
as `off` with its cost line, and the nightly tidy is one of `on`, `off`
(nobody turned it on, or it was turned off), `off` with the computer it runs
on (another computer tidies it), or `missing` (on here, but this computer's
scheduler does not hold the job); `versions` and `versions_say` list what
differs from the versions that wrote the collection (see "Self-fixing"); `notes_not_in_graph` (Track AW6) counts
the source notes that exist as files with no node in the graph, said in a line of `say` that also tells whether the
nightly tidy reads them in (it does, through `POST /ingest/file`, up to 200 a night, for no tokens;
`second_brain/ingestion/lost_notes.py`). `say` is in order, each heading followed by its own
lines: the intro, one line per item, the token limit, then "Who may do what"
and its lines (fix round R6).

## Defaults that cannot hurt, and a collection that mends itself (Track AG5)

D10 and S13 of first_run_experience_plan_v1.md:
**everything that spends no tokens is on by default, or turns itself on when
what it needs is present; everything that spends tokens is offered with its
cost, never silently off; and off is always visible.** The catalogue says, for
each option, how it ships (`default_state`), and
`tests/contracts/test_shipped_defaults.py` holds every claim to the shipped
values (the install's `config/*.json`, the readers' own defaults, and a
collection just made) and to D10's table, so a new switch that ships off
without a token cost fails it.

| feature | how it ships | tokens |
|---|---|---|
| nightly tidy (label words by count, renamed originals, repairs of notes, search entries caught up, lost letters, equations) | **on in every collection made now**, on this computer only (`config/maintenance.json`: `enabled`, `machine`) | none |
| the nightly steps that use the AI (judging label words, merging concept pages, faceting, hints, reading suggestions) | **asked once per install, by level** (Track AX): off until answered (`ai_tokens_per_night: 0`); a collection made after the answer starts at it; `check_setup` shows the level with its most in brackets | yes |
| better PDF reader (docling) | `pdf_converter: "auto"`: **on whenever installed** | none |
| text recognition for scans | `ocr: "auto"`: **on for the files whose pages hold no text**, when docling and a text-recognition program are installed | none |
| equations | read at night on this computer, `equations_minutes: 60` | none |
| reading pictures, hints (tails), suggestions of what to read next | **asked**: off, listed in `check_setup` and `GET /ops/status` with their cost | yes |
| search by meaning | on; installing the local search program is explained in the welcome | none |
| asking before anything is spent | on in every collection made from the conversation | none |

A collection made before this is **reported, not switched on**: with no
`config/maintenance.json` that says `enabled`, `check_setup` lists the nightly
tidy as off ("because nobody turned it on"), and the owner turns it on with
`change_setting`. A schedule written before the limit existed has no
`ai_tokens_per_night`; since Track AX it is read as the catalogue's
`level_when_unset` (medium), computed on read and never written, and the check
says "Medium (up to 150,000 tokens a night), set automatically. Say a different
level to change it." The night enforces it. `KOVIUM_NO_AUTO_READERS` (set for the whole test suite)
makes `auto` mean off in one process, the way `KOVIUM_EMBEDDINGS_DISABLED`
does for search by meaning.

**A collection made now** (`POST /collections`) gets the schedule written and
this computer's scheduler asked to hold the job (cron where `crontab` exists,
the Windows task scheduler where `schtasks` does; `second_brain/maintenance/os_job.py`).
The answer carries `nightly` `{enabled, time, machine, job, ai, ai_asked_before}` and a sentence; when this
install never answered the nightly AI question, it also carries that question as `ask` (code
`nightly_ai_choice`), and when it did, a sentence naming the level the collection starts at.
The job runs the nightly script **in process** (`--in-process`): no service
need be left running. A computer with neither scheduler leaves the schedule
saved and says the job is not started by itself yet.

**Said plainly (Track BA, fix round R2).** A job added to this computer is
real system state, so the answer says it: "It will tidy itself every night at
02:00 on this computer, at no cost." "For that, a job was added to this
computer's task scheduler, under the name shown below. Each night it runs
Kovium's free tidy for this collection, and nothing else." (or "list of
timed jobs, called cron"); while no AI level is chosen, "The steps that use the
AI stay off unless you choose a level for them."; and "To stop it, say "turn the
nightly tidy off", which also takes the job off this computer." The name and
what it runs are beside the words (`nightly.job.name`, the task
`kovium-maintenance-<id>` or the cron line's marker
`kovium-maintenance:<id>`; `nightly.job.runs`, the nightly script for this
collection), since a collection's id may hold characters a sentence a person
reads may not. Switching it off (`change_setting` "Nightly tidy" off) removes
the task (`schtasks /Delete`) or makes cron match the schedules again; a
removal the scheduler refuses while the task is still there is said as such
("...could not be changed. Its job may still start at night, and it then finds
the tidy off and does nothing."), never as done. The welcome's offer of a first
collection says it in its yes: "a collection is made once you say where it is
kept, and a free nightly tidy is added to this computer's scheduler". The cron
script is loaded through the checkout loader (Track AZ3), so the MCP server,
whose working folder is the person's project, reaches it too.

**One crontab, several installs.** A user's crontab is shared by every install
on the computer (a second checkout, a test copy). Each install owns only the
marked lines that start **its own** runner (`'<checkout>/scripts/nightly_maintenance.sh'`,
quoted as `cron_line` writes it): `sync` removes and rewrites only those, and
`status` and `os_job.installed` count only those (`schedule.existing_markers` and
`strip_markers` take `runner=`). Before this, a sync removed every install's
marked lines and re-added only its own, so making a collection in one checkout
ended another checkout's nightly tasks (found in the live OpenClaw test,
2026-10-24).

**The nightly AI level** (Track AX). A person chooses a level in plain words,
never a token count: off, light, medium or heavy, each shown with its most in
brackets, "Medium (up to 150,000 tokens a night)". The numbers are the option
catalogue's (`nightly_ai_limit.levels`, one place; an install changes them
there), and `second_brain/maintenance/nightly_levels.py` reads them. The
collection keeps `ai_tokens_per_night` (the number the night obeys) and
`ai_level` (the name, shown back); a number that matches no level is shown as
"Custom (up to N tokens a night)". The example collection never uses the AI at
night, whatever its file says. The question is asked once per install, in the
welcome (see [[welcome-v1|welcome_v1.md]]).

**The nightly token limit.** `ai_tokens_per_night`: the steps that use the AI
run only while the tokens they have reported tonight are below it. A step
already begun is finished, and the next one waits for the next night (held,
not forgotten: a held judgement is not remembered as made). The free steps run
whatever it is. The night's report says what was held or used, in words.

### Self-fixing (D7)

- **A note added while the search program was down** is indexed by the next
  nightly run once it answers: the run counts the notes without passage
  vectors from files alone, asks `POST /ops/embeddings/reindex` (`missing`)
  only when there are some, bounded by time, and reports how many wait.
- **Versions** (`second_brain/maintenance/versions.py`): `maintenance/versions.json`
  in the collection records the platform version and the collection's data layout
  (they only move up, after a night that finished) and each search program
  version and model id (`/api/version`, the model's digest from `/api/tags` of
  the local server) that embedded passages. `check_setup` (`versions`,
  `versions_say`), `open_collection` (`travel.versions`), `GET /ops/status`
  (`first_run.versions`) and the night say, in plain words, when a collection
  was last used with a newer or older data layout or platform, or its search
  vectors were built by another search program version, another copy of the
  model under the same name, or more than one. A collection written by a
  newer data layout is said before anything is changed.
- **A rebuild in a copy** (`POST /ops/embeddings/rebuild` `{max_seconds?, wait?}`,
  `second_brain/retrieval/rebuild.py`): every passage is embedded afresh into a
  scratch database outside the collection (so a synced folder never carries a
  partial index), and only a finished build replaces the model's vectors, in
  one transaction. A build that fails, loses the server or runs out of time
  changes nothing. Free; backgrounded unless `wait`.
- **Unread equations**: the night reads them with the local formula model
  (`POST /sources/equations`, `method: local`, which needs no approval of a
  plan, since it spends no tokens), a document at a time within
  `equations_minutes`; one longer than the whole limit is left for a request
  made on purpose. See [[equations-v1|equations_v1.md]].

## Over MCP: one connection per collection (D2)

| tool | route | connection | grant |
|---|---|---|---|
| `create_collection` | `POST /collections` | any | `manage` |
| `open_collection` | `POST /collections/open` | any | `manage` |
| `connect_agent` | `POST /collections/connect` | bound | `manage` |
| `set_access` | `POST /collections/access` | bound | `manage` |
| `change_setting` | `POST /collections/settings` | bound | `manage` |
| `make_agent` | `POST /collections/agent` | bound | `manage` that may change `settings` |
| `check_setup` | `GET /collections/setup` | any | none (read only) |
| `plan_adding` | `POST /sources/add` (a plan) | bound | `manage` that may `add` |
| `start_adding` | `POST /sources/add` (the run) | bound | `manage` that may `add` |
| `adding_status` | `GET /sources/add/status` | bound | `manage` that may `add` |
| `add_note` | `POST /sources/note` | bound | `manage` that may `add` |

The three adding tools are Track AG3's doors on AR's one request
([[adding-documents-v1|adding_documents_v1.md]]): the plan is free, the run
spends only with the plan's approval, within its budget and the agent's caps.
`add_note` (Track AY4) keeps text from the conversation as a note, shown first
and written on the person's `confirm`; it needs the same grant and is free.

**The binding and the tool list** (Track AY3). A connection with no collection
declares that its tool list can change, and a binding tells the program so
(`notifications/tools/list_changed`): the collection's tools are there in the
same conversation. The binding's answer ends with `collection.tools_follow`,
which tells the person to start a new conversation where the tools do not
show; the new conversation starts bound. The binding is this computer's state
(`config/mcp_connections.json`): ignored by git, and declared in the option
catalogue as not a choice (Track AY6).

- **The `manage` grant** is held only over a local transport (standard input
  and output, or the tests' in-process one), never a remote one
  (`setup_needs_local_connection`): by a connection with no collection (to make
  or open one), and by one whose agent the collection's policy names at
  `manage`, read again on every call, so a grant given or taken back acts at
  the next call. Without it the setup tools are not listed, and a call by name
  is refused `access_not_granted` with the words a person is told
  (`fix.next.user_text`, kept in the compact view of a refusal).
- **Binding.** A successful `create_collection` or `open_collection` on a
  connection with no collection binds it, in the same session, to that
  collection: its tools, scope, grants and audit become the collection's, the
  client is told its list changed, and `data.connection` says so. The binding
  is recorded under the instance root (`config/mcp_connections.json`, keyed by
  the connection's agent id), so the next session from the same entry (no
  `--workspace`) starts bound to it, with nothing typed and the entry
  unchanged. A connection already bound is never rebound: a further
  collection gets a connection of its own (`program`). A binding whose
  collection is gone starts welcome mode, said on standard error.
- **The agent id.** `--agent`, else `welcome` (with or without `--workspace`):
  the id a fresh install's entry runs under, and so the id the first
  collection's manager has. A launch whose agent is not registered for the
  collection is refused at start, naming the way out: start without
  `--workspace` and open the collection's folder (`open_collection`).
- The instructions of a connection with no collection say when to call
  `create_collection`, `open_collection` and `check_setup`; a bound
  connection's instructions add one line when it holds `manage`.

## The newcomer test, as measured here (S11)

Scripted (`tests/integration/test_collections_mcp.py`, an agent following only
what the responses name, on a fresh install holding the example), on this
machine, 2026-10-22. **Story 1 to minute 5**: the welcome, the example
answered (0.303 minutes after the first call; the example built on first
opening), the tour, the sample added free, then "make your own collection":
made and this connection bound 0.433 minutes (26 seconds) after the first
call, the next session from the same entry bound; commands typed 0; questions
put to the person 2 (see the example; what will your collection hold), tour
offers 2; tokens before an approval 0. **Story 2**: a collection copied from
another computer opened from a connection with no collection in 0.01 minutes,
its missing folder named and pointed here, and a question answered 0.041
minutes after the first call; commands 0, questions 1 (where are these files
here). **Story 3**: Codex connected reading only, its entry written, its own
session reading at once; the owner's "it may add documents" applied and acting
at Codex's next call, 0.034 minutes into its session; commands 0, questions 0
beyond the owner's own request. Not measured: the install step, a clean
machine, a person, a real second program started by its own harness.

## An agent joins a collection in one conversation (Track BC)

Started by a sentence like "connect yourself to my thesis collection", the same
for every program. Code: `second_brain/collection_setup/join.py`, the scoped
places in `programs.py`, the caller agent's text in `second_brain/caller_agent/`.

### `GET /collections` (BC1)

The collections registered on this computer, sorted by name: `id`, `name` and
`holds` (from the collection's own `config/mcp.json`), `notes` (its own notes,
not concept pages, the owner's work or ideas, counted from its database opened
read only; `null` when it cannot be read), where it is (`folder`,
`relative_to`, as for `POST /collections`) and `joined` (whether the calling
agent is registered for it). The example, its working copy and the manual are
the platform's own and are not listed. Nothing is opened or registered, and
nothing is audited (it names no collection). Until Track BB, any caller with a
local connection sees the names; the MCP tool `list_collections` needs the
local `manage` grant, so a remote transport never does.

**Finding one by name** (`find`, also behind `open` and `join`): the person's
words are folded to letters and digits in any script (marks dropped, case
folded). A whole match of an id or a name wins; otherwise each word of theirs
found inside a collection's name or id counts, and the one collection that
counts most is it. None: `collection_name_unknown`, with the names there are.
Several equally: `collection_name_ambiguous`, with which, for the caller agent
to ask the person. No word list of any language is used.

### `POST /collections/join` (BC2, BC3, BC4)

- `collection` (its name or id) or `folder` (a collection's folder not on the
  list yet; registered at apply, as `open` does).
- `program`: the caller's own program; omitted, the one the connection's agent
  id names (`codex`, `openclaw`), else `program_unknown`.
- `agent`: the id the caller joins under. For OpenClaw, required: the agent's
  own id (`join_agent_missing`). For the others, omitted means the project
  folder's name made an id. An id a program shares among its agents (`welcome`,
  `openclaw`, `codex`, `claude-code`, `cursor`, `system`) is refused
  (`join_agent_shared`): one agent's grant, binding and audit are never
  another's. The refusal names the way out: for Claude Code, Codex and
  Cursor, leave `agent` out and join under the folder's name, which it
  spells; for OpenClaw, send the agent's own id. The welcome instructions and
  the tool's description say to send `agent` only from OpenClaw, since a
  Claude Code agent in the live test first sent the id its welcome
  connection showed.
- `project`: the folder the caller works in (for OpenClaw, the agent's own
  workspace folder), never asked of the person (`agent_project_not_found`).
- **The plan, changing nothing** (no `confirm`): `say` in this order: what will
  happen, where the connection goes, what the agent may do there, then (BC4)
  either that nobody manages the collection so this agent will (from a caller
  with no collection, unless `read_only`), or, for an agent that will read
  only, who can allow more; then that its own instructions go into the project
  beside what is there (or, for Cursor, that it gets none yet) and the style;
  last (`join.takes_effect`), that the connection takes effect in a new
  conversation, not this one, because the program loads its connections only
  when one starts, so the person knows before the yes. `asks`: the style questions (`make_agent`'s three and its free line, without
  its offer), only when the collection has no style yet and the agent will be
  allowed to change its settings: "asked once" per collection. While `asks` is
  there, no `confirm` is given; with the answers sent, the plan comes back with
  its `confirm` code, which binds the collection, the agent, the program, the
  project, the connection's name and place, the access and the agent's text.
- **Done** (with `confirm`, on the person's yes): the folder registered if it
  was not; the agent registered under its own id; its access written (claimed,
  read, narrowed to read with `read_only`, or kept when the policy names it);
  the connection added (`steps.connection`); the caller agent's text written and
  recorded with the connection's name and place (`steps.agent_file`,
  `config/caller_agents.json`); audited as `join_collection`. A step that cannot
  be done (the program refuses its settings, say) is said
  (`join.connection_failed`) and the rest still happens. The last `say` line
  (`join.next_<program>`, only when the connection was added) says the
  connection takes effect only in a new conversation (for Codex, once it
  starts again), why (the program loads its connections only when one
  starts), and what to do: start one and ask about the collection.
- **Where the connection goes** (`connection.place`, measured on each program on
  2026-10-24 in a scratch home): `project` for Claude Code (its local scope, the
  command run in the project folder, taken only when `claude mcp add --help`
  names it), for Codex (the project's own `.codex/config.toml`, only in a
  project its user settings mark trusted, for the folder or the repository root
  above it: Codex 0.154 silently ignores an untrusted project's connections) and
  for Cursor (`.cursor/mcp.json`); `agent` for OpenClaw (`codex.agents` naming
  the one agent, then `openclaw mcp reload`); `everywhere`, with `why_wider`
  (`claude_no_local_scope`, `codex_not_trusted`, `openclaw_no_agent_list`), said
  in words. Trusting a project is the person's own decision and is never made
  here.
- **Its token** (Track BB step 5, [[access-tokens-v1|`access_tokens_v1.md`]]):
  where the collection's access tokens are on (`require_token`), the plan says
  one more sentence before `join.takes_effect` (`join.token_kept`: its own key
  will be kept in a small file in its folder, which its instructions name),
  and `token_file` (`.kovium/<collection id>.token`, relative to
  `project`). On the yes, a token is issued for the agent (`made_by: join`) and
  written to `<project>/.kovium/<collection id>.token` (`0600` where the
  system has permissions, `/.kovium/` added to the repository's
  `.git/info/exclude` when the folder is in one), `steps.token` `{done, file,
  short}`, and `join.token_done` is said; the caller agent's text tells the
  agent to call `sign_in` with `token_file` set to that file at the start of
  every conversation (and, over HTTP, to send it as `Authorization: Bearer`).
  The token is never returned, and the audit records `token: {short, agent}`.
  **The rule that keeps this safe** (fix round BB-S1, in full in
  [[access-tokens-v1|`access_tokens_v1.md`]]): a join's `confirm` proves the
  plan was shown, not who said yes, so where tokens are on a join (its plan
  too) or a leave acts for an agent id only for the owner's own process, a
  caller that shows a token of that id or of a manager that may grant (a
  bearer token, or `sign_in` on the connection, a welcome one included; a file
  the caller names is never read as proof, BB-T1), or, through the local MCP
  connection alone, an id the collection does not know yet, which joins
  reading only and never claims the collection. To leave, an agent signs in
  with its key file first. Otherwise refused before anything changes
  (`token_required` for a known id, `token_issue_not_allowed` for a new id
  over plain HTTP). Opening a collection with tokens on by its name or folder
  needs a token for it (BB-S4).
- **Its name**: `kovium-<collection id>`; when another agent's own
  connection to the same collection holds that name in a place both share
  (OpenClaw's one list, a user-wide file), `kovium-<collection id>-<agent>`,
  and the caller agent's text names whichever it is.
- **The tools named as the program shows them** (BC3): Claude Code keeps
  letters, digits, `-` and `_` (`mcp__kovium-<id>__recall`); Codex and
  OpenClaw's Codex runtime keep letters, digits and `_` (`mcp__kovium_<id>__recall`,
  seen in the OpenClaw live test's log). The text says to search the tool list
  for the connection's spelled name where tools are deferred, and to use the
  collection's tools first whenever the person names the collection or asks
  about what it holds, before any memory or file search of the program's own.
  `programs.tool_prefix` is the one place this is spelled; a test renders each
  program's text against the name the join writes.

### `POST /collections/leave` (BC4)

The same `collection`, `program`, `agent` and `project` the agent joined with.
The connection is taken out of the place the record says (else the place a
join would choose), only when its arguments start this collection for this
agent; the caller agent's file is deleted, or its section cut out of
`AGENTS.md` with every other byte kept; the record row goes, unless the
connection could not be taken out (then asking again knows its name and its
place); a binding of this agent's to this collection goes. When the
connection was taken out, `say` adds `leave.takes_effect`: this conversation
may still show its tools until it ends, and a new one no longer has them. The
agent's tokens for the collection are taken back and its key file deleted
(`steps.token` `{revoked, file_removed}`, `leave.token_gone`; Track BB). Where
the collection's tokens are on, leaving needs the local MCP connection, the
owner's own process, or a caller signed in as that agent or as a manager that
may grant (`token_required` otherwise). The
collection
stays, and what the agent may do there stays in its policy, so joining again
gives the same. Audited as `leave_collection`.

## Limitations

- Adding documents over MCP is Track AG3's (`plan_adding`, `start_adding`,
  `adding_status`), with the caps enforced at approval.
- No authentication (above): the policy binds MCP connections; over HTTP it
  binds honest callers only.
- A collection opened on a computer where no agent it names is installed reads
  only there; the way out is a grant made beforehand (it travels with the
  folder) or the program it names connected here.
- Search by meaning built elsewhere is reported by model, not yet by the
  server's version (D7, AG5); the rebuild in a copy is AG5's.
- `connect_agent` writes the program's user-wide settings; whether each
  program picks up a new entry without a restart is the program's.
  `join_collection` writes the narrowest place instead (Track BC).
- OpenClaw refuses a settings write that shrinks its file sharply (seen live:
  removing the only entry of a near-empty profile). `leave_collection` then
  says the connection is still there and keeps its record; with any other
  entries present it is taken out.
- Cursor's spelling of tool names is not measured; it is taken to be Claude
  Code's. Cursor gets no caller agent text.
- `codex.agents` hides an OpenClaw connection from the other agents on the
  Codex runtime only, and is no lock. With the collection's access tokens on
  (Track BB), the identity is proved by each agent's own token, and the shared
  `openclaw` id stops mattering; with them off, it is said, not proved.
- Most options are still not reached by `change_setting` (`NOT_HERE`), among
  them the nightly tidy, whose switch also needs a schedule on the computer
  (AG5).
