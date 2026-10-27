---
type: "documentation"
title: "The install step, the Claude Code plugin and the caller agent v1"
description: "The contract for Track AG4 of docs/first_run_experience_plan_v1.md: the one install step (D1), the Claude Code plugin this checkout ships (D1; its own folder since Track AY1), the caller agent added to a person's project and the answer..."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing Kovium"
doc_path: "docs/install_and_caller_agent_v1.md"
doc_sha256: "ff4384d9032d452f32ebf0673416ccfd0817a5f9a711ba73127529b0c279a392"
---

# The install step, the Claude Code plugin and the caller agent v1

The contract for Track AG4 of `docs/first_run_experience_plan_v1.md`: the one
install step (D1), the Claude Code plugin this checkout ships (D1; its own
folder since Track AY1), the caller agent added to a person's project and the
answer style kept per collection (D9). Code: `scripts/setup.py`,
`second_brain/install/`, `skills/claude-code/` (the plugin: its manifest, skills,
start command and `scripts/mcp_launch.py`), `.claude-plugin/marketplace.json`,
`second_brain/caller_agent/`.

**State.** Built 2026-10-23 for Claude Code; Track AG7 (the same day) brought
Codex and OpenClaw to the same flow: their install step is a connection of the
program's own kind, and the caller agent is rendered in each program's own
format ([Codex and OpenClaw](#codex-and-openclaw), below). Cursor gets the same
install step (a plain connection) and no caller agent yet. No program but
Claude Code has a plugin: neither Codex nor OpenClaw was installed where this was
built, a plugin format could not be run, and a plain connection plus the
caller agent already gives them the whole newcomer flow.

## The install step

```bash
python scripts/setup.py                        # every AI program found on this computer
python scripts/setup.py --program "Claude Code"  # one program only (may be repeated)
python scripts/setup.py --no-plugin            # Claude Code by a plain connection, without the plugin
python scripts/setup.py --use-this-python      # no private environment: this interpreter has it all
python scripts/setup.py --no-probe             # do not ask the local search program whether it answers
```

The same command on Linux, macOS and Windows, run with any Python the project
supports (`requires-python` in `pyproject.toml`). In order:

1. **This Python** is checked against `requires-python`; too old is said in
   words, exit status 2.
2. **The private environment** (`.venv` in the checkout, or `--venv`) is made
   with `python -m venv` and the checkout installed into it with
   `pip install -e '.[mcp]'`, only when that environment does not already run
   this checkout's `second_brain` with the MCP SDK and FastAPI. A failure is
   said in words, exit status 1. This half reads its sentences from the
   catalogue file with the standard library only, since nothing else is
   installed yet.
3. **The hand-over**: `python -m second_brain.install --root <checkout>` in
   that environment (`second_brain/install/service.py`):
   - each AI program `collection_setup.programs` knows (Claude Code, Codex,
     Cursor, OpenClaw) is looked for, as `connect_agent` looks;
   - its sign-in is checked where the program can say it, with the brain
     keeper's own check (Codex: `codex login status`; OpenClaw: `openclaw
     models status --json`; a sign-in that bills per token is reported as not
     signed in, never used). Claude Code's is `claude auth status` (Track AW7):
     signed in to a subscription, not signed in, signed in with a key that
     charges per use (refused in words, never used), or, for a version without
     the command, said to ask for a sign-in when first opened;
   - the optional helpers are said in the catalogue's words, each "works" or
     "Optional: ... is not installed" with its `need` line: the better PDF
     reader, the document reader, and search by meaning (asked of the local
     search program unless `--no-probe`; not said at all then);
   - each program found is connected in **welcome mode** (no collection yet,
     `docs/welcome_v1.md`) under its own agent id (`claude-code`, `codex`,
     `cursor`, `openclaw`; the AG2 default `welcome` was shared by every
     program): Claude Code as the plugin (below); with `--no-plugin`, or when
     the plugin's start command cannot run here, by a plain connection named
     `kovium` (`claude mcp add --transport stdio --scope user`); the
     others the way `connect_agent` adds theirs (`docs/collections_v1.md`),
     named `kovium`, with `--agent <program> --root <checkout>`;
   - **never two connections in Claude Code** (Track AY1): with the plugin
     connected, `claude mcp get kovium` is asked; a plain connection that
     is exactly what `--no-plugin` adds (user scope, the same program and
     arguments) is removed with `claude mcp remove kovium -s user` and
     said (`install.plain_removed`); any other of that name is left as it is
     and named, with what to do (`install.plain_kept`), since it may be another
     install's or someone else's. With `--no-plugin` while the plugin is
     installed (`claude plugin list --json`), that is said
     (`install.plugin_beside_plain`); nothing is removed then: a plain
     connection was asked for.
   - **OpenClaw's four skills** (`skills/openclaw/*`) are installed with OpenClaw's
     own `openclaw skills install <folder>`, once OpenClaw is connected. Checked
     against OpenClaw 2026.9.5: it copies a local skill folder into the default
     agent's workspace (`<workspace>/skills/<name>`), and a second run answers
     "already exists" with exit status 1 and changes nothing. `--force` is never
     passed, so a skill the person changed is never replaced, and no setting is
     changed. With more than one agent, OpenClaw refuses a skill with no owner
     ("Pass --agent <id>"); the step then names the agent `openclaw agents list
     --json` marks as default (`isDefault`), the one a single-agent install gets
     them in, and keeps that owner for the remaining skills (found in the
     OpenClaw live test, 2026-10-27, where every skill was refused). Said as `install.skills_added`, `install.skills_already` or
     `install.skills_failed`. A skill that cannot be installed costs nothing
     else, because the connection's own instructions carry the newcomer flow.
     Agents with a workspace of their own do not see them; copy them by hand
     there (`skills/openclaw/README.md`).
4. **What to do next**, in words: open the program and say "help me start
   with Kovium"; start a new conversation if it was open.

**Idempotent.** A second run makes nothing again: the environment is found, a
connection or plugin already there is said ("was already connected, so
nothing changed"), OpenClaw's skills already there are left as they are, and
no settings file changes. The check whether the environment already runs this
checkout is made with `python -I`, so a folder named `second_brain` beside the
checkout (the checkout itself, by its usual name) is not read as the package:
without it the second run crashed there (found in the live OpenClaw test,
2026-10-24). For Codex that is a table
already in `config.toml`; for OpenClaw it is `openclaw mcp show kovium
--json` asked **before** `openclaw mcp set`, because `set` replaces an entry of
the same name without a word: the same entry is "already connected", a
different one is left as it is and said so (`install.why_other_entry`), and
only a missing one is set. **Safe.** No administrator
password, no model call, no token. Every program command is the program's
own, run by the file `PATH` resolves it to, with every metered credential
removed from its environment (`billing_guard.subscription_safe_env`). Exit
status: 0 when every program found was connected, 1 when one could not be,
3 when none was found. OpenClaw's documentation says saved-server changes
(`add`, `set`, `unset`) need its local gateway stopped; when OpenClaw says so
the step prints it in words (`install.why_stop_first`) and changes nothing, and
the same command connects it once the service is stopped.

Every sentence it prints is a catalogue text (`setup_v1.json`, `install.*`),
held to the plain-words rule.

**Out of time is not "did not start"** (Track AY2). A program command that
started and was still running when the wait ran out is said as such, with the
wait (`install.why_timed_out`: "it was still working after 600 seconds, so this
step stopped waiting. Run this again: what was already done is kept."); only
one that could not start at all is `install.why_not_started`. The waits are
stated in code: `COMMAND_SECONDS` (120) for a program's own command,
`PLUGIN_INSTALL_SECONDS` (600) for `claude plugin install`, which copies the
plugin's folder (measured at a few seconds for the plugin as it now is). The
same split holds for a collection's connection (`connect_agent`): a command
that times out is refused `program_timed_out`, not `program_not_started`.

## The Claude Code plugin

The plugin is a small folder of its own, `skills/claude-code/`, and the
checkout's root is a marketplace that lists it (Track AY1):

| file | holds |
| --- | --- |
| `skills/claude-code/.claude-plugin/plugin.json` | `name: kovium`, `version` (the package's), `skills: ./` (the four caller skills at the plugin's root), `commands: [./commands/]`, and `mcpServers.kovium` |
| `.claude-plugin/marketplace.json` | `name: kovium`, `owner`, one plugin with `source: ./skills/claude-code` |
| `skills/claude-code/commands/start.md` | the start command, `/kovium:start` (a plugin's commands are named after it): call `welcome` and follow it; `help` for how Kovium works |
| `skills/claude-code/scripts/mcp_launch.py` | what the connection starts |
| `skills/claude-code/checkout.json` | written by the install step, ignored by git: where this computer's checkout is and which program its server is |

The install step records the checkout for the launcher, then adds the plugin
with Claude Code's own commands:

```bash
claude plugin marketplace add <checkout>
claude plugin install kovium@kovium --scope user
```

**What Claude Code does with it, measured** (Track AY1, the real `claude`
2.1.240 in a scratch home: `HOME`, `USERPROFILE`, `APPDATA`, `LOCALAPPDATA` and
`CLAUDE_CONFIG_DIR` all under a scratch folder; `marketplace add` and
`plugin install` need no sign-in):

- `claude plugin marketplace add <folder>` copies nothing: the marketplace's
  `installLocation` is the folder itself (under a second).
- `claude plugin install` **copies the plugin's source folder** into
  `~/.claude/plugins/cache/<marketplace>/<plugin>/<version>/`, through a
  staging folder `cache/temp_local_<n>_<id>/`. It leaves out `.git` and
  copies everything else, a private environment included. A source of `./`
  (the whole checkout, as before AY1) with 100 MB in 2,000 files took 4.1 s;
  the owner's checkout holds a 1.6 GB environment in about 47 thousand files,
  which is why the install step's wait ran out (that size was read off the
  folder; copying it was not timed).
- A session then runs the plugin **in place** from the marketplace's folder:
  `claude mcp list` shows the connection as `python
  <checkout>/skills/claude-code/scripts/mcp_launch.py --agent claude-code`,
  not the cached copy. This is what Claude Code's documentation says of a
  relative-path plugin in a marketplace added from a local path; the copy at
  install is made anyway. The marketplace's file is read afresh: moving a
  plugin's `source` to a subfolder took effect at once, with no update or
  reinstall (the stale cached copy stays until the plugin is uninstalled).
- With the plugin as it now is, the whole install step took 4.9 s, the cached
  copy is 25 files and 222 kB, and `claude mcp list` reports the connection
  "Connected".

So the plugin holds only what a plugin is, and the launcher finds the
checkout through the record the install step writes before it installs the
plugin, never through where the plugin sits: should a later Claude Code run
the cached copy, the record travels with it.

**What it ships** (checked against Claude Code's plugin documentation on
2026-10-23, pinned by `tests/contracts/test_claude_code_plugin.py`, and
validated by `claude plugin validate` 2.1.240 in a scratch home: "Validation
passed" for the plugin and for the marketplace):

- the four caller skills (`skills/claude-code/kovium-*`), at the
  plugin's root; the plugin has no default `skills/` folder of its own, so
  nothing is loaded twice;
- the start command;
- the connection, `python ${CLAUDE_PLUGIN_ROOT}/scripts/mcp_launch.py --agent
  claude-code`, in welcome mode; its tools are named
  `mcp__plugin_kovium_kovium__<tool>`. Once `create_collection` or
  `open_collection` binds it (`docs/collections_v1.md`), the same entry starts
  bound to that collection, and in the same conversation the server tells the
  program its tool list changed (`docs/mcp_server_v1.md`, Track AY3).

It ships no agent: the caller agent belongs to one collection and is added to
the person's own project (below). The contract test holds the folder to
plugin files only (no environment, collection, data or package) and under
2 MB.

**The launcher.** `skills/claude-code/scripts/mcp_launch.py` (standard
library only, importing nothing of Kovium: until it reads the record it
does not know where that is) reads `checkout.json` in the plugin's folder,
appends `--root <checkout>` unless given, and runs the recorded server with
the plugin's standard input and output passed through; it writes nothing to
standard output. With no record it says so on standard error
(`install.not_set_up`); with a record whose checkout or server is gone, that
the checkout moved and the setup is to be run again from where it is now
(`install.moved`); exit status 2 either way. The two sentences are copies of
the catalogue's, held equal by a test. `KOVIUM_PLUGIN_RECORD` names
another record file (the suite points it outside the checkout).

**Moving from the earlier plugin.** An install made before AY1 has the plugin
with the whole checkout as its source. After updating the checkout, run
`python scripts/setup.py` once: it writes the record, finds the marketplace
and plugin already there (Claude Code reads the marketplace's new source on
its own), and removes a plain connection the step made earlier with
`--no-plugin`. The old, large cached copy can be dropped with `claude plugin
uninstall kovium@kovium` followed by the step again.

## The caller agent (`POST /collections/agent`, `make_agent`)

One agent per collection, added to the person's project beside the agents it
already has, which are never read for writing, moved or changed. Over MCP the
tool is `make_agent`, on a bound connection whose agent holds `manage` with
`settings` (as `change_setting`); a reading connection does not list it and a
call by name is refused `access_not_granted`.

| field | what |
| --- | --- |
| `project` | the full path of the project folder the agent program works in: the calling agent's own working folder (for OpenClaw, the workspace folder of the agent), never asked of the person |
| `program` | the agent program: Claude Code, Codex or OpenClaw (`agent_program_not_ready` for another). Omitted: the program the connection's agent id names (`claude-code`, `codex`, `openclaw`), else Claude Code |
| `level` | `simple` or `technical`; omitted: asked |
| `length` | `short` or `detailed`; omitted: asked |
| `tone` | the tone and language in the person's words; empty for plain and friendly in their language; omitted: asked |
| `extra` | anything else, in their words; empty for nothing; omitted: asked |
| `confirm` | the preview's code; omitted: the preview |

**The preview** (no `confirm`) changes nothing and returns `file` (relative to
the project: `.claude/agents/kovium-<collection>.md`), `agents_beside`
(the project's own agent files, by name), `style`, `say` (what will be added,
beside how many agents, the style, and that no style drops sources, "not
found" or the question before spending), `asks` and, when something would
change, `confirm`. **`asks`**: the questions still open, one at a time, the
recommended answer first: the offer (only while the person has no agent for
this collection and answered nothing), simple or technical, short or
detailed, tone and language, and anything else. These are the caller agent's
own questions (S2: three and one free line, counted apart). The offer's yes,
given after the preview's sentences, is the person's yes: the agent applies
with the code.

**Applied** (with `confirm`): the style is written into the collection's own
`config/style.json`, the agent file is written (LF line endings) and the
project is recorded in the install's `config/caller_agents.json` (this
computer's, never the collection's), so a later style change rewrites it.
**A moved checkout** (Track AW7): the file's start line names this computer's
interpreter and checkout; when either is gone, `check_setup` says so in plain
words (`agent_files_out_of_date`, with an `ask` whose yes is `make_agent` for
that project, shown first and written only with its code), and the connection's
instructions carry the same offer for the agent to make. Found from the files
only; a file that is not ours, or gone, is never reported.
Audited as `make_agent` (`surface: collections`).

**Refusals.** `agent_project_not_found` (422: no project, or not a folder
here); `agent_file_taken` (409: a file of that name the platform did not write
is there; it is left as it is); a stale or other preview's code (409, as
`set_access`); `access_not_granted` (403).

**The file.** Claude Code's subagent format: a frontmatter of `name`
(`kovium-<collection>`) and `description`, a marker line naming the
collection (how a later run knows the file is its own), and the body of
`second_brain/caller_agent/template_v1.md` filled for the collection: when to
use which tool, asking before spending, sources and warnings, `help`, its
access, the style, and what never changes. `tools` is left out on purpose: a
subagent without it inherits the session's tools, MCP ones included, so it
reaches the collection's connection whatever it is named (`kovium-<collection>`
or the plugin's). **Which connection the instructions name** (2026-10-27): an
agent that joined with a connection of its own keeps its name (Track BC2);
otherwise, when the install step's shared `kovium` connection is bound
to this collection for the calling agent id (`config/mcp_connections.json`,
how a collection made or opened through it is served), that one, with its one
caveat (`render.SHARED`: it serves one collection at a time, so `overview`
naming another means `open_collection` first); otherwise the collection's own,
`kovium-<collection>`. Before this, a collection made through the shared
connection got instructions naming `kovium-<collection>`, which nothing
had added, and the OpenClaw live test's agent found no such tools. Without a
connection in its session the agent starts Kovium's own service with one
line (`scripts/run_kovium.py --skip-install --background`) and calls it
over HTTP with the collection and its agent id; the rules are the same.
`--background` runs the service detached from the terminal (its output in
`logs/service.log`) and returns once it answers, or at once when one already
answers on the port: an agent's command tool waits until every process its
command started lets go of the terminal, so a service started in the
foreground or with a plain `&` held the agent's whole conversation open. One template, one renderer (`render.py`, `FORMATS`): Codex and
OpenClaw are rendered there from the same text, so the three cannot drift.
Only the lines that name the agent's role and its second connection differ
(`render.ROLE`, `render.ALTERNATIVE`).

### Codex and OpenClaw

Facts read from each program's own documentation on 2026-10-23 and pinned in
`tests/contracts/test_program_formats.py` (neither program was installed where
this was built, so none of this was run against a real one; the connection
entries were live-checked earlier, `docs/mcp_server_v1.md`).

| | Claude Code | Codex | OpenClaw |
| --- | --- | --- | --- |
| what an agent is | a subagent file | a custom agent: one TOML file | a workspace; no file for one more agent |
| where the caller agent goes | `.claude/agents/kovium-<collection>.md` | `.codex/agents/kovium_<collection>.toml` | a marked section of `AGENTS.md` in the agent's workspace folder |
| what is written | the whole file | the whole file: `name`, `description`, `developer_instructions` (the three fields the format requires) and nothing that could override the session | the section only, below the person's text |
| refused when | a same-named file is not ours | a same-named file is not ours | the file cannot be read as text, or our section lost its end line |
| used by | the main agent hands it requests | Codex hands work to an agent **named in the request**, so the final sentence tells the person to ask for it by name | the agent follows it whenever the person asks about the collection, as it is part of its instructions |
| its connection | inherited from the session | inherited from the parent (the file cannot override it) | the agent's own |

- **Codex.** A custom agent inherits its parent's permissions, provider and MCP
  servers and cannot override them, so the file carries no model, sandbox or
  server key. Its name is snake case (`kovium_` plus the collection's name
  with every other character turned into `_`), like Codex's own examples; two
  collections whose names differ only by `-` and `_` share a file, and the
  second is refused as `agent_file_taken`. Project-scoped agents load for a
  project Codex trusts. The multi-line string escapes backslashes and runs of
  three quotes and drops control characters, so the person's own words (any
  language, quotes, a backslash) arrive as written; a test decodes the file and
  compares.
- **OpenClaw.** What tells the agent how to work is the workspace's `AGENTS.md`
  (loaded every session, cut at 20,000 characters a file by default). The caller
  agent is a section between two marker lines, `<!-- kovium caller agent v1
  for collection <id>; begins here ... -->` and `<!-- ... ends here -->`, with its
  headings one level down under `## Kovium: <name>`. A new section is
  added at the end, after a blank line; an existing one is replaced in place;
  **every byte outside the markers stays as it was**, including a byte-order
  mark, a missing final newline and CRLF endings (the section then uses CRLF
  too). One section per collection; they never touch each other, and
  collection `a` never matches collection `ab`. A file with no instructions yet
  is created holding only the section. The section is under 6,000 characters at
  the longest style, leaving room for the person's own text.
- **Same promises everywhere.** The style and the four invariants are rendered
  after one another in each format, the invariants after the style and saying they
  win over it (a test decodes each format and checks the order at every extreme
  of the profile and with a line that asks to drop them). A later style change
  rewrites every recorded agent of the collection that is still ours, in its own
  format; one the person took over (no marker) is left alone.

## The answer style, per collection

`config/style.json` in the collection's own folder, so it travels with it and
is the same for every agent program connected:

```json
{"level": "simple", "length": "short", "tone": "", "extra": ""}
```

The defaults are the newcomer's rules: `simple`, `short`, plain and friendly
in the person's language. Anything unknown reads as the default. It is
changed in the conversation with `change_setting` by four plain names
(catalogue entries `answer_wording`, `answer_length`, `answer_tone`,
`answer_extra`; "Simple or technical answers", "Short or detailed answers",
"Tone and language of answers", "Anything else about how answers sound"),
shown then applied; applying one rewrites every recorded agent of the
collection that is still its own, and says so.

**Style changes how things are said, never what.** Four things hold whatever
the style, and every rendering puts them after the style and says they win
over it, even over the person's own line (`style.INVARIANTS`): every answer
shows its sources; "the collection does not say" is said plainly; before
anything that spends tokens, the plan and its cost are shown and asked, and
started only with the yes and the plan's approval; every warning that changes
what an answer means is relayed. The platform enforces the last two whatever
any agent writes (the approval, `docs/adding_documents_v1.md`).

**Without the caller agent.** Once a collection has a style, a bound
connection's instructions carry it as one paragraph, with the four
invariants (`identity.answer_style_line`, about 820 characters); a collection
with no style adds nothing, so the default listing does not grow.

## Measured (2026-10-23, Windows)

- **Install to first answer**, on a simulated clean machine: a fresh copy of
  the checkout's files, a scratch home (`HOME`, `USERPROFILE`, `APPDATA`,
  `LOCALAPPDATA`), stand-in agent programs alone on `PATH`, an empty pip
  cache, a base Python 3.13; `python scripts/setup.py --no-probe`, then the
  plugin's connection started as Claude Code starts it (since Track AY1,
  `skills/claude-code/scripts/mcp_launch.py`, through the record), `welcome`,
  `try_example` and a suggested question answered with its source: **1.58
  minutes** (78.4 s for the step, the environment made and its packages
  downloaded; 16.7 s from the connection's start to the answer, the example
  built on first opening). 0 commands typed after the step, 1 question (see
  the example), 0 tokens.
- **One more hop per answer**, by prompt size (no model call; a live run
  needs a signed-in subscription, which a scratch home does not have, and
  the real home is not used). Without the caller agent, the main agent calls
  the tools itself; with a style chosen, the connection's instructions grow by
  823 characters (about 200 tokens), once per session. With the caller agent,
  each delegated request also puts in a fresh context the agent's file (3,973
  characters, about 1,000 tokens), the session's tools listed again (30,463
  characters, about 7,600 tokens, for a bound managing connection's 20 tools
  and its instructions), the request, and the answer handed back; plus
  Claude Code's own instructions and built-in tools for a subagent, not
  measured here. Tokens are characters divided by four. So the hop costs
  roughly 9,000 tokens or more per delegated request, before any cache: the
  style in the connection's instructions gives the same wording for about
  200 tokens a session, which is why it is there.

## Measured on Codex and OpenClaw (2026-10-23, Windows, Track AG7)

The newcomer test, `tests/integration/test_newcomer_per_program.py`, run for each
program in a scratch home with the program's own connection recorded by the
install step and used to start the server (its command line and agent id), the
brain keeper switched off as a tripwire and a stand-in reader. In memory, so the
times are the platform's, not a program's start-up:

| program | commands typed | questions before the first answer | questions before an answer from the person's own document | agent's own questions | tokens before an approval | to the first example answer | to the first own-document answer |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Claude Code | 0 | 1 | 4 | 4 | 0 | 0.077 min | 0.122 min |
| Codex | 0 | 1 | 4 | 4 | 0 | 0.055 min | 0.094 min |
| OpenClaw | 0 | 1 | 4 | 4 | 0 | 0.054 min | 0.085 min |

The agent's file is 3,856, 3,885 and 3,867 characters (about 960 to 970 tokens)
for Claude Code, Codex and OpenClaw: the hop of a delegated request costs the same
in every format (the rest of the AG4 measure, the session's tools listed again, is
the connection's and does not depend on the program). OpenClaw has no hop at all
when the section is part of the agent's own instructions: it adds those tokens to
each session instead. Not measured: a real Codex or OpenClaw (neither was
installed here and the real home is never used), a person, Linux.

## Limitations

- The plugin is installed from the checkout (a local marketplace). Claude Code
  2.1.240 copies the plugin's folder into its cache at install and runs it in
  place from the checkout (measured, Track AY1); the launcher works from either,
  through the record. A marketplace published elsewhere is not supported, and
  nothing is published. The copy's behaviour was measured on Windows only.
- The record (`skills/claude-code/checkout.json`) holds this computer's
  absolute paths: a moved checkout needs the step run again from its new
  place (the launcher says so).
- The plugin's connection starts with `python` from `PATH`. Where that is not
  a Python new enough (only `python3`, or a stub that opens a store), the step
  connects Claude Code by a plain connection instead, without the plugin's
  skills and start command. It says it connected, and then why there is no
  plugin, what that leaves out and how to get it (`install.plugin_needs_python`,
  Track BF0: seen on a Linux with only `python3`, where the person was told only
  "connected").
- On Windows a checkout in a deeply nested folder can fail the install: a
  dependency's files exceed the 260-character path limit (seen with the
  checkout in a folder 153 characters long; one of 52 worked). Keep the
  checkout in a short path, or turn on long paths in Windows.
- Claude Code's sign-in is read from `claude auth status`; a version without
  that command is left unchecked (said so), never reported wrong.
- The caller agent is one more model context per answer; its cost is measured
  by prompt size, not by a live run.
- The agent file holds the full path of this install's interpreter and
  checkout (for the HTTP way), in the person's own project; it is rewritten
  when the style changes; a moved checkout is found and offered again (see
  "A moved checkout"), not rewritten unasked.
- Neither Codex nor OpenClaw has a plugin here. Their documentation names plugin
  formats, but neither program could be run where this was built, and a format
  that cannot be run is not shipped. A plain connection and the caller agent
  give both the whole newcomer flow; a plugin would add only what the Claude Code
  one adds (skills and a start command).
- OpenClaw's four operator skills (`skills/openclaw/`) go only into the
  **default** agent's workspace (what `openclaw skills install` does). Another
  agent with its own workspace gets them by hand (`skills/README.md`). Setting
  `skills.load.extraDirs` from the install step would replace the person's own
  list (`openclaw config set` replaces a whole array) and OpenClaw's docs say
  `config set` needs its gateway stopped, so it is not done.
- OpenClaw's `mcp.servers` is one list for every agent: once connected, every
  OpenClaw agent on the computer sees Kovium's tools. OpenClaw 2026.9.5
  has a per-server agent list, `mcp.servers.<name>.codex.agents`, but only
  agents on the Codex app-server runtime honour it; agents on any other runtime
  still see the server. The install step does not write it: it would look like
  a boundary while being none, an OpenClaw version that does not know the key
  may refuse it, and a mistyped agent id hides the tools without a word. One
  connection shown to one agent is what `join_collection` does (Track BC,
  [[collections-v1|collections_v1.md]]): asked "connect yourself to my thesis
  collection", the agent joins under its own id, its entry
  `kovium-<collection id>` carries `codex.agents` naming it, and its
  `AGENTS.md` section names the tools as its runtime shows them. The join says
  in words that this hides the tools and is no lock.
- **On OpenClaw's Codex runtime, a connection's tools are not in the model's
  view.** They are deferred: the model finds them only by searching its tool
  list, and it searches only when something tells it to. In the live test a
  bound agent asked about "my PhD collection" in a fresh conversation used
  OpenClaw's own memory search, which failed, and never looked for the
  collection's tools. What tells it is the caller agent's section in that
  agent's `AGENTS.md`, which `join_collection` writes (and `make_agent` for a
  bound connection): since Track BC it names the tools exactly as the runtime
  spells them (`mcp__kovium_<id>__<tool>`), says to search the tool list
  for them, and to use them whenever the person names the collection or asks
  about what it holds, before any memory search of its own.
- **The agent id is said, not proved.** A connection's `--agent` is whatever its
  entry says, and on OpenClaw every agent shares the install's one entry, so
  every OpenClaw agent calls as `openclaw`. When one of them binds the welcome
  connection to a collection (`create_collection`, `open_collection`), the
  binding is keyed by that id, so every OpenClaw agent opens that collection
  from then on. Today the protection is what a collection grants (read only
  unless the owner raises it, `set_access` with a shown sentence), the budgets
  and the audit, which guards against mistakes between one person's own agents,
  not against an agent that pretends. Track BB (access tokens, built
  2026-10-25, [[access-tokens-v1|`access_tokens_v1.md`]]) is the fix, per
  collection and off until its owner turns it on: each agent then signs in
  with its own token (`sign_in`), so the shared `openclaw` id stops mattering.
  A join to a collection with tokens on writes the agent's key file
  (`<project>/.kovium/<collection>.token`) and the caller agent's text
  tells it to sign in with that file first.
- Codex hands work to a custom agent only when asked by name (its documentation
  says so); the caller agent's closing sentence says it. The same agent's
  instructions are reachable without it: the connection's instructions carry the
  style once one is chosen.
- A project-scoped Codex agent loads for a project Codex trusts; in an untrusted
  one the file is there and Codex does not read it (the personal folder,
  `~/.codex/agents/`, is not used: the agent belongs to a project).
- The OpenClaw section sits in a file OpenClaw cuts at 20,000 characters by
  default; the section is under 6,000, but a very long file of the person's own
  can push it past the cut. The `check_setup` report does not look at this.
- An OpenClaw `mcp show` that prints something this cannot read is treated as an
  entry of someone else's: nothing is replaced (a refusal, not a guess).
