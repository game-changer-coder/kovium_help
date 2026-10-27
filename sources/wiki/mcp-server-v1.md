---
type: "documentation"
title: "MCP server v1"
description: "The contract for the MCP server over the governed routes: how it is started, what it introduces itself as, which tools it lists, how scope, refusals, audit and long calls work. Plan and later phases: docs/mcp_server_plan_v1.md. Code..."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing Kovium"
doc_path: "docs/mcp_server_v1.md"
doc_sha256: "a93718d9e07fb84f200819c0faa93b52a232e75d94d093febc93645f13635739"
---

# MCP server v1

The contract for the MCP server over the governed routes: how it is started,
what it introduces itself as, which tools it lists, how scope, refusals,
audit and long calls work. Plan and later phases: `docs/mcp_server_plan_v1.md`.
Code: `second_brain/mcp_server/`.

**State.** AB1 (the server), AB2 (the read tools: eight, the default
surface), AB3 (the surfaces: `wider` with five more read tools, the
workspace's ceiling in `config/mcp.json`, widening by `request_tools`, the
capabilities resource) and AB5 (the wiring for four harnesses, the install
outside a checkout, the caller skills' MCP sections, the measured token
cost) and AB4 (the writes through review: `remember` and `forget` on the
`write` surface, each filing a review item the owner decides) are built: the
track is complete. Track AG1 added **welcome mode** (a server started with no
collection: the welcome's tools only) and two tools on every connection,
`welcome` and `help` (see "Welcome mode" and `docs/welcome_v1.md`). Track AG2
added the setup tools (`create_collection`, `open_collection`,
`connect_agent`, `set_access`, `change_setting`) behind a `manage` grant held
over a local transport only, `check_setup` on every connection, the binding of
a connection with no collection to the one it makes or opens, and the access
policy per agent in the collection's `config/mcp.json` (see "Setup and who
may do what" and `docs/collections_v1.md`). Track AG3 added adding documents
from the conversation, behind the same grant: `plan_adding`, `start_adding`
and `adding_status`, the doors on `POST /sources/add`, with the approval of a
plan enforced (see `docs/adding_documents_v1.md`). Track AG4 added the one install step
(`python scripts/setup.py`), the Claude Code plugin this checkout ships (its
connection `kovium`, started in welcome mode by
`skills/claude-code/scripts/mcp_launch.py`, tools named
`mcp__plugin_kovium_kovium__<tool>`), `make_agent` behind the same
grant, and the collection's answer style in a bound connection's instructions
(see `docs/install_and_caller_agent_v1.md`). Track AY3 made a connection with
no collection declare `tools.listChanged`, so a binding's new tools appear in
the same conversation, and Track AY4 added `add_note`, text from the
conversation kept as a note, free, behind the adding grant.

**At a glance**, for whoever wires a harness:

| what | where |
| --- | --- |
| start command, flags, refusals at start | "Starting it" |
| a fresh install with no collection: what it serves | "Welcome mode" |
| making, opening and connecting collections; who may do what | "Setup and who may do what" |
| a snippet per harness (Claude Code, Codex, Cursor, OpenClaw), timeouts, `wider` | "Wiring a harness" |
| what each harness was seen to do | "Harness behaviour observed" |
| the tools of each surface and what each sends | "Tools", "Surfaces" |
| what a result looks like | "Results" |
| what a session costs to list, and the measured comparison with HTTP | "What a session costs to list", "Token cost measured at the CLI" |
| scope, refusals, audit, resources | "Scope from the connection", "Refusals", "Audit", "Resources" |
| proposing additions and withdrawals: the grant, the statuses, the owner's decision | "Writes through review" |
| what is not offered, and why | "Not offered over MCP" |
| versions | "Versions" |

## What it is

One process serves one workspace to one MCP client over standard input and
output. The harness starts it for a session and stops it at the end. Each
tool is one call to a governed route of the same app the HTTP service runs
(`create_app`), made in process: no socket, no listener, and the HTTP service
need not be running. Both can run at once over the same workspace (see
"Both surfaces at once").

Read-only by default. Nothing a caller sends is written into the collection:
where the owner grants proposals, two tools (`remember`, `forget`) file
review items, and only the owner's decision in review changes a note (see
"Writes through review"). No model is called by the server itself; one tool,
`synthesize`, asks the brain keeper for a written answer, exactly as the
answer route does over HTTP.

## Starting it

```bash
pip install -e '.[mcp]'        # once, in the checkout's venv: the SDK is an optional extra
kovium-mcp --workspace <workspace-id> --agent <agent-id> [--root <instance-root>] \n                 [--surface default|wider|write] [--max-surface default|wider|write] [--allow-proposals] \
                 [--checkout <checkout>]
python -m second_brain.mcp_server --workspace <workspace-id> --agent <agent-id> [--root <instance-root>]
```

- `--surface`: the set of tools the session starts on (default `default`);
  `--max-surface`: the widest set `request_tools` may move it to (default:
  what the workspace allows). Both are cut to the workspace's ceiling, and
  what was cut is said on standard error; see "Surfaces".
- `--allow-proposals`: the launch's half of the `propose` grant the write
  tools need (AB4); the owner's half is in the workspace's `config/mcp.json`.
  Without both, the write tools are not listed (see "Writes through review").
- `--root`: the instance root, the folder holding
  `config/workspace_registry.json`, the one the HTTP service runs from.
  Default: the checkout the server is installed from, as the HTTP service
  started by `scripts/run_kovium.sh`. A workspace registered with an
  external `root_path` is found through that registry like any other.
- `--checkout`: the checkout whose `prompts/`, `format-templates/` and
  `config/` the server reads, for an install outside a checkout whose
  instance root is not the checkout itself (below). An install inside a
  checkout reads that checkout: a `--checkout` naming another one is refused
  (exit status 2), since it would move the default `--root` and the folder
  recorded paths are spelled against away from the HTTP service run from
  this checkout; naming the same checkout changes nothing.
- `--log-level`: `DEBUG`, `INFO`, `WARNING` (default) or `ERROR`; every log
  line goes to standard error.
- `--agent` (Track AG2): optional with or without `--workspace`; its default
  is `welcome`, the id a fresh install's entry runs under (a `--workspace`
  alone used to be refused for want of an agent id, which nothing told a
  newcomer).

**Editable or not.** The editable install (`pip install -e '.[mcp]'`, what
`scripts/run_kovium.sh` does) is the recommended one: the code runs
from the checkout and finds its prompts, intake templates and install
configuration beside it. A non-editable install (`pip install '.[mcp]'`, from
the checkout) copies the package into the environment's `site-packages`
(every subpackage and the package's own data files since AB5; before, only
the top package, so `kovium-mcp` failed on its first import) but not
`prompts/`, `format-templates/` or `config/`, which live in the checkout and
are the install's, not the package's. Such an install needs a checkout
named at start: `--root <checkout>` when the instance root is the checkout
(the usual case), or `--checkout <checkout>` beside another `--root`.
Without one it refuses to start (exit status 2, saying which flag to give);
a folder that holds no `prompts/` and `format-templates/` is refused the
same way. The code then runs from `site-packages` and reads the checkout's
data, so the two should be the same version: reinstall after pulling.
Verified in a throwaway environment: 253 of the package's 253 tracked files
installed (6 before), the console script served the synthetic workspace
(eight tools, `overview`, `recall`, `entity`), the prompts and the 13 intake
templates resolved from the named checkout.

**It fails closed.** Before opening anything, the server reads the registry
(read only) and refuses to start, with a message on standard error and exit
status 2, when:

- there is no registry under the root;
- the workspace is not registered (the message lists the registered ones);
- the agent is not registered for that workspace (the message lists the
  registered ones);
- the workspace's registered folder is gone;
- the MCP SDK is not installed (the message gives the install command).

It never creates a workspace or registers an agent at start, and it builds its
app in strict scope whatever `KOVIUM_ALLOW_UNSCOPED` says. In a session,
registration happens only through the install's own example (`try_example`)
and, since Track AG2, the setup tools a `manage` grant allows
(`create_collection`, `open_collection`, `connect_agent`, and since Track BC
`join_collection`; "Setup and who may do what"). Both refusals name the way out a person can take with nothing
typed: start the server without `--workspace` (welcome mode) and open the
collection's folder (`open_collection`), which registers it and this agent at
the access the collection gives; a collection's manager connects another
agent with `connect_agent`; a script calls `POST /collections/open` or
`POST /ops/workspaces`.

### Welcome mode (Track AG1, D2)

```bash
kovium-mcp [--agent <agent-id>] [--root <instance-root>]     # no --workspace
```

**Bound by an earlier session (Track AG2).** When `create_collection` or
`open_collection` bound this connection's agent to a collection, the binding
is recorded under the instance root (`config/mcp_connections.json`), and the
same entry (no `--workspace`) starts bound to that collection from then on,
said on standard error; a binding whose collection is gone starts welcome mode.

Started without `--workspace` (what a fresh install's entry names: there is
no collection yet), the server checks nothing and refuses nothing at start: it
serves **welcome mode**. No collection is bound; the tools listed are
`welcome`, `help`, `get_manual`, `try_example`, `tour`, and since AG2 `create_collection`,
`open_collection` and `check_setup`, and nothing else, whatever surface was
asked for; the only grant held is `manage`, over a local transport, for the
two that make or open a collection; the agent id is `--agent`, else `welcome`.
The instructions say the connection has no collection yet, to call `welcome`
when the person is new or asks how to start and follow it (relay `say` in
their language, put `ask` with its recommended answer first, make the call it
names on yes, never ask them to type a command), and `help` for a question
about the software. Every call is audited, metadata only, as one JSON line in
the instance root's `logs/mcp_welcome_audit.jsonl` (there is no collection's
log to write to). `--workspace` without `--agent` is refused as before.

A connection bound to a collection lists `welcome` and `help` as well (both on
the default surface), and `get_manual` only where the agent may manage the collection (the setup tools'
`manage` grant; fetching the manual writes into the install, so a reading connection never lists it, and its
`help` says in words where the manual can be had instead of offering it), and its instructions add one line saying when to call
them; it never lists `try_example` or `tour`, and a call to either by name is
refused `tool_not_offered`, saying they live on a connection with no
collection. The example is the install's own collection, apart from the
person's (D2's boundary). The tools, their routes and what each returns:
`docs/welcome_v1.md`.

## Wiring a harness

Every harness below starts the server itself, as a child process, for each
session, and talks to it over standard input and output. The entry names the
program and its arguments; nothing else is needed. Placeholders:

| placeholder | what to put there |
| --- | --- |
| `<venv>` | the Python environment the checkout is installed in (`pip install -e '<checkout>[mcp]'`); `scripts/run_kovium.sh` makes it as `<checkout>/.venv` |
| `<instance-root>` | the instance root (`--root`): the folder holding `config/workspace_registry.json`, usually the checkout itself |
| `<workspace-id>` | a workspace registered in that instance |
| `<agent-id>` | an agent registered for that workspace |

**Which code runs.** A venv's program runs the code installed in that venv,
whichever folder the entry names as `--root`. An editable install runs the
checkout it was installed from: a second checkout or worktree needs a venv of
its own. The server says on standard error at start which code is serving and
where its data and the workspace's ceiling file are (one line, beginning
`kovium-mcp: kovium <version>, code in ...`), so a mismatch shows
in the harness's log.

**The command: the console script.**

| POSIX | Windows |
| --- | --- |
| `<venv>/bin/kovium-mcp` | `<venv>/Scripts/kovium-mcp.exe` |

followed by `--workspace <workspace-id> --agent <agent-id> --root
<instance-root>`. It needs no activation and no `PYTHONPATH`. The
interpreter form, `<venv>/bin/python -m second_brain.mcp_server` (Windows:
`<venv>/Scripts/python.exe -m ...`), runs the same `main`, but `-m` puts the
working folder first on the import path: if the harness starts the server in
a folder holding a `second_brain` package (another checkout, a worktree),
that code is imported instead of the installed one. Prefer the console
script; use `-m` only where the working folder is known.

Write every path in full: a harness starts the server from a working folder
of its own choosing, so a relative path resolves somewhere else. In JSON and
TOML, write Windows paths with forward slashes (`C:/...`), which Windows
accepts, or double every backslash.

**`--checkout`** is needed only for an install outside a checkout (a
non-editable `pip install`) whose instance root is not the checkout; see
"Starting it". An install inside a checkout reads that checkout and refuses a
`--checkout` naming another one.

**Timeouts.** `synthesize` is slow (one measured at 19.4 s; a long question
takes longer), and the server sends a progress notification every five
seconds while it waits (see "Long calls"). Each harness's default and what
the entries below set:

| harness | tool timeout by default | set below |
| --- | --- | --- |
| Claude Code | `MCP_TOOL_TIMEOUT`, about 28 hours (no per-request timer on a stdio server); a call is also stopped after 30 minutes with no reply and no progress, which the progress notifications prevent | nothing |
| Codex | `tool_timeout_sec`, 60 s | raised to 600 s |
| Cursor | not documented | nothing (there is no documented setting) |
| OpenClaw | `requestTimeoutMs`, 60 s | raised to 600,000 ms |

Startup (`initialize`) was measured at 1.4 to 2.8 s, within every startup
default below.

**Starting on `wider`.** Two steps. The workspace's owner allows it in
`config/mcp.json` under the **workspace's own folder**: its `root_path` in
`<instance-root>/config/workspace_registry.json`, by default
`<instance-root>/sandboxes/<workspace-id>`; not the instance root's own
`config/`. (`capabilities` reports the file as `set_in: config/mcp.json`,
`set_in_is: workspace-relative`, and the start line on standard error names
the folder.)

```json
{"max_surface": "wider"}
```

Then either the entry adds `"--surface", "wider"` to the arguments, or the
session widens itself with `request_tools` when the harness acts on
`list_changed` (Claude Code does). Without the owner's file, `--surface
wider` is cut to `default` and said on standard error.

**Allowing proposals.** Two halves again: the owner writes
`{"max_surface": "write", "allow_proposals": true}` in the same file, and the
entry adds `"--allow-proposals"` (and `"--surface", "write"`, or the session
widens with `request_tools`). Either alone grants nothing.

**`synthesize` needs the brain keeper's CLI** (`claude` or `openclaw`) on the
server's `PATH`, signed in on the owner's subscription; the server inherits
the environment the harness gives it. `capabilities` (wider) says whether
written answers can be had.

**`"type": "stdio"`.** The JSON entries below carry it. Cursor's
documentation lists it as required; Claude Code treats an entry with a
`command` as stdio without it, and `claude mcp add` writes it (with an empty
`"env": {}`).

### Claude Code

**The one step (Track AG4).** `python scripts/setup.py` installs this checkout
as a Claude Code plugin (`claude plugin marketplace add <checkout>`, then
`claude plugin install kovium@kovium --scope user`): its
connection, its skills and `/kovium:start`, with no entry written by
hand. The entries below are for a harness wired by hand, or a further
collection's own connection (`connect_agent`). See
`docs/install_and_caller_agent_v1.md`.

Checked 2026-10-17 against https://code.claude.com/docs/en/mcp,
https://code.claude.com/docs/en/cli-reference,
https://code.claude.com/docs/en/env-vars and
https://code.claude.com/docs/en/permissions. **Live-verified** with Claude
Code 2.1.240 on Windows: the `.mcp.json` entry below, filled for a synthetic
workspace and given with `claude -p --mcp-config <file> --strict-mcp-config
--allowedTools "mcp__kovium"`, connected, listed the eight default tools
and called them.

By command (`--` separates Claude Code's own options from the server's
command; `--scope project` writes the project's `.mcp.json`; `local`, the
default, keeps it to you in this project; `user` makes it yours in every
project). POSIX shell:

```bash
claude mcp add --transport stdio --scope project kovium -- \
  <venv>/bin/kovium-mcp --workspace <workspace-id> --agent <agent-id> --root <instance-root>
```

Windows (PowerShell or cmd), on one line:

```powershell
claude mcp add --transport stdio --scope project kovium -- <venv>\Scripts\kovium-mcp.exe --workspace <workspace-id> --agent <agent-id> --root <instance-root>
```

Or in the project's `.mcp.json` (or a file given with `--mcp-config`):

```json
{
  "mcpServers": {
    "kovium": {
      "type": "stdio",
      "command": "<venv>/bin/kovium-mcp",
      "args": ["--workspace", "<workspace-id>", "--agent", "<agent-id>",
               "--root", "<instance-root>"]
    }
  }
}
```

- **Timeout:** none needed (table above). To cap this server's calls, add
  `"timeout": <milliseconds>` to its entry (it overrides `MCP_TOOL_TIMEOUT`
  for this server alone); `MCP_TIMEOUT` is the startup timeout (default
  30 s).
- **Widening:** 2.1.240 lists the wider tools in the same session after
  `request_tools`, so `--surface wider` is optional here.
- **Permission:** the tools are named `mcp__kovium__<tool>`; the rule
  `mcp__kovium` (in `--allowedTools`, or the settings' allow list)
  allows every tool of this server. Claude Code asks once before it uses a
  server from a project's `.mcp.json`.
- **Variables:** `.mcp.json` expands `${VAR}` and `${VAR:-default}` in
  `command`, `args` and `env`, so a shared file can name the venv by a
  variable each person sets.
- **Result size:** Claude Code caps a tool result at `MAX_MCP_OUTPUT_TOKENS`
  (default 25,000 tokens). The compact results here stay far below it; the
  largest `view: "full"` recall measured (about 52,000 characters) is near
  half of it.

### Codex (OpenAI Codex CLI)

Checked 2026-10-17 against https://learn.chatgpt.com/docs/extend/mcp?surface=cli
(where https://developers.openai.com/codex/mcp redirects). **Live-verified**
2026-10-17 with Codex 0.160.0 on Linux, signed in with ChatGPT: the keys of
the entry below, given to `codex exec` as `-c mcp_servers.kovium.<key>=...`
overrides (the same keys as the table, without editing the file), connected;
the session called `overview`, `recall` and `read` and answered from them, with
no shell command. The `config.toml` table itself and `codex mcp add` were not
run.

In `~/.codex/config.toml` (or a trusted project's `.codex/config.toml`):

```toml
[mcp_servers.kovium]
command = "<venv>/bin/kovium-mcp"
args = ["--workspace", "<workspace-id>", "--agent", "<agent-id>", "--root", "<instance-root>"]
startup_timeout_sec = 30
tool_timeout_sec = 600
```

Or by command, `codex mcp add kovium -- <venv>/bin/kovium-mcp
--workspace <workspace-id> --agent <agent-id> --root <instance-root>`, then
add the two timeouts to the table it writes.

- **Timeout:** `tool_timeout_sec` (default 60 s, too short for some
  `synthesize` calls) and `startup_timeout_sec` (default 10 s), per server.
- On Windows, write the path with forward slashes, or as a TOML literal
  string in single quotes, where a backslash is not an escape.
- `enabled = false` keeps the entry and stops the server being started;
  `enabled_tools` and `disabled_tools` narrow what Codex shows of the listing
  (the server's own surface still decides what may be called).
- **A model the sign-in does not serve** stops the session before any server
  is called: `The '<model>' model is not supported when using Codex with a
  ChatGPT account` is Codex's own configured `model`, not this server. Choose
  one the sign-in lists (`codex exec -m <model>`, or `model` in
  `config.toml`).

### Cursor

Checked 2026-10-17 against https://cursor.com/docs/context/mcp. **Not
live-verified**: Cursor is not installed on the machine this was built on.

In the project's `.cursor/mcp.json` (or `~/.cursor/mcp.json` for every
project):

```json
{
  "mcpServers": {
    "kovium": {
      "type": "stdio",
      "command": "<venv>/bin/kovium-mcp",
      "args": ["--workspace", "<workspace-id>", "--agent", "<agent-id>",
               "--root", "<instance-root>"]
    }
  }
}
```

- **Timeout:** the documentation names none, so none is set; a long
  `synthesize` may be cut off by a limit of Cursor's that is not documented.
  `recall` and `read` return in well under a second on the measured
  workspaces.
- **Variables:** `${workspaceFolder}` (the folder holding `.cursor/mcp.json`)
  and `${userHome}` are expanded in `command` and `args`. Only when Cursor's
  project is the checkout, its venv is `<checkout>/.venv` and the checkout is
  also the instance root, can the entry say
  `${workspaceFolder}/.venv/bin/kovium-mcp` (Windows:
  `${workspaceFolder}/.venv/Scripts/kovium-mcp.exe`) and `--root
  ${workspaceFolder}`; otherwise write the paths in full.

### OpenClaw

Checked 2026-10-17 against https://docs.openclaw.ai/gateway/config-extensions,
https://docs.openclaw.ai/cli/mcp, https://docs.openclaw.ai/cli/mcp/transports
and https://docs.openclaw.ai/cli/mcp/registry. **Live-verified** 2026-10-17
with OpenClaw 2026.9.5 on Linux: `openclaw mcp set kovium '<entry>'`
with the entry below as JSON (plus an `env` object) saved it; `openclaw mcp
probe kovium` listed eight tools and the resources; after `openclaw mcp
reload`, one turn through the running gateway (`openclaw agent --agent <id>
--session-id <a new id> --message-file <file> --json`, the gateway's Codex
runtime, signed in with ChatGPT) called `overview`, `recall` and `read` and
answered from them; `openclaw mcp unset kovium` removed it and left the
configuration file byte for byte as it was before. Where the documentation is
only partial, that is said below.

In `~/.openclaw/openclaw.json` (JSON5), under `mcp.servers`:

```json5
{
  mcp: {
    servers: {
      "kovium": {
        command: "<venv>/bin/kovium-mcp",
        args: ["--workspace", "<workspace-id>", "--agent", "<agent-id>", "--root", "<instance-root>"],
        requestTimeoutMs: 600000,
      },
    },
  },
}
```

Or by command, with the same entry as JSON: `openclaw mcp set kovium
'<the entry above, as JSON>'`, or `openclaw mcp add kovium --command
<venv>/bin/kovium-mcp` with one `--arg` per argument (whether `--arg`
passes a value starting with `-` through is not documented; check with
`openclaw mcp show kovium`).

- **Timeout:** `requestTimeoutMs`, per server, in milliseconds (default 60 s);
  `connectionTimeoutMs` covers startup and the first listing (its default is
  not documented).
- OpenClaw refuses some environment variables for a server it starts
  (interpreter start-up and loader keys, by its documentation, without a full
  list). The entry sets none: the console script needs no `PYTHONPATH`.
- `openclaw config schema` prints the installed version's schema, the one to
  trust where it and the pages differ.
- **The isolated run cannot borrow a stored sign-in.** `openclaw agent exec
  --config <file>` (a turn against a configuration file of its own) was
  refused by the model provider for want of credentials, with and without
  `--no-auth-env-only`, when the sign-in was an OAuth one kept in OpenClaw's
  state. Wire the server into the configuration the gateway uses, as above.
- **Saved-server changes need the local gateway stopped** (`add`, `set`, `unset`;
  `list`, `show` and `status` work while it runs), by OpenClaw's registry page
  (read 2026-10-23, Track AG7). `set` takes one JSON object and its documentation
  does not say whether it replaces an entry of the same name; the install step
  therefore asks `openclaw mcp show <name> --json` first and never sets over a
  different entry (`docs/install_and_caller_agent_v1.md`).
- `openclaw mcp reload` disposes the cached MCP runtimes, so a server added
  to a running gateway is used from the next turn.

## Harness behaviour observed

| harness | seen | how |
| --- | --- | --- |
| Claude Code 2.1.240 | connects from a `--mcp-config` file with `--strict-mcp-config`; lists the eight default tools; calls them | headless runs (`claude -p`) on a synthetic workspace and on a real one, Windows |
| Claude Code 2.1.240 | acts on `notifications/tools/list_changed` in the same session: after `request_tools` it listed the wider tools without a restart | an agent session on a synthetic workspace |
| the SDK's client (`mcp==2.3.0`), handshake era `2025-11-25` | hears `list_changed` over stdio and in process; its next `tools/list` is the new set | the server's tests |
| a client on the `2026-07-28` per-request era | is sent no list change (the SDK sends one only on a `subscriptions/listen` stream, which this server does not serve); sees the new tools when it lists again | probed in process |
| Claude Code 2.1.299 | connects from a `--mcp-config` file with `--strict-mcp-config` and no built-in tools; lists the eight default tools; calls `overview`, `read`, `entity` | a headless run (`claude -p`) on the example workspace, Linux |
| Codex 0.160.0 | connects from `-c mcp_servers.kovium.*` overrides to `codex exec`; calls `overview`, `recall`, `read`; no shell command | a headless run on the example workspace, Linux, ChatGPT sign-in |
| OpenClaw 2026.9.5 | `openclaw mcp probe` lists eight tools and the resources; a gateway turn on its Codex runtime calls `overview`, `recall`, `read` | the configured server, one turn through the gateway, Linux, ChatGPT sign-in |
| Cursor | not observed | not installed where this was tested |

A harness that does not act on `list_changed`, or lists tools once per
session, keeps showing the tools it started with: start it on the surface it
needs (`--surface wider`, within the ceiling). A call by name to any tool of
the session's surface is answered whatever the harness shows.

## What it says it is

The `initialize` response's `instructions` are written for an agent that
knows nothing about the platform, kept short (they are sent at the start of
every session), and generated from the workspace's own registry entry and
configuration and from the tool table, never from code that knows a subject:

- the workspace id, and the name and description its owner gave it in the
  workspace's `config/mcp.json` (both optional; the description is cut at 600
  characters):

  ```json
  {"display_name": "<a name>", "description": "<what it holds, in the owner's words>"}
  ```

  (the same file holds the ceiling, `max_surface` and `allow_proposals`; see
  "Surfaces");

- what it holds, by the platform's own tiers: the collection's notes by kind
  (their `type`, the twelve largest named), the concept notes the platform
  keeps (a merged concept's redirect is not counted), the owner's own work
  and ideas (said to be context, never evidence);
- that it may only be read, or, where the connection holds the `propose`
  grant, that it may also propose additions and withdrawals, each a review
  item the owner decides, nothing written or recalled until accepted;
- that the workspace and agent are fixed for the connection;
- each listed tool with what it is for (`summary` in the tool table);
- how to start: with the tool the table marks `starts_here` (`overview`)
  when the agent does not know yet what the collection holds, otherwise with
  the tool that fits its question;
- how a result is read: compact JSON, `data` the answer, each `notices`
  entry what to be careful about and what to do next, with the exact tool
  call in `mcp` or `here` when the step is not available over the
  connection; a condition of the install said in full once, then marked
  `repeat`; `view: "full"` for the whole response; a refusal's `error`;
- only when this connection may widen to more tools, one line naming the
  wider sets and `request_tools` (about 115 characters; nothing on an install
  whose ceiling is `default`, so the default listing does not grow);
- only when the launch's `--surface` was cut, one line saying what it asked
  for and what this connection may reach.
- only when the collection's owner chose an answer style (Track AG4,
  `config/style.json`), one paragraph saying how to word answers and the
  four things no style drops (about 820 characters; a collection with no
  style adds nothing).

The note types a workspace declares in `config/note_types.json` are not
listed on their own: a declared kind with notes appears in the counts, and
one with none tells a caller nothing.

`serverInfo` is `kovium` with the installed package's version; its
title is `Kovium: <display name or workspace id>`.

## Tools

The default surface (AB2), in the order listed:

| tool | over | exposes | always sends | notes |
| --- | --- | --- | --- | --- |
| `overview` | `GET /agent/knowledge/overview` | `note_type`, `limit`, `offset` | | what the collection holds: notes by kind and tier, its notes by title (paged), the concepts most of them name; where a newcomer starts |
| `recall` | `POST /agent/knowledge/answer` | `query`, `limit`, `scope`, `conversation_id`, `refresh` | `synthesize: false`, `judge_relevance: false`, `widen: cache_only` | ranked items with provenance, facts apart, `focus` and its `read` call, trust signals; no model call |
| `read` | `POST /agent/knowledge/read` | `notes`, `query`, `max_chars` (default here 12,000) | | the relevant sections of notes as Markdown, in one call; `focus.read` works as given; named sections first, the source filling the rest |
| `synthesize` | `POST /agent/knowledge/answer` | `query`, `limit`, `scope`, `evidence`, `conversation_id`, `refresh` | `synthesize: true`, `judge_relevance: false` | a written answer with its sources, from the brain keeper; **expensive** |
| `entity` | `POST /agent/knowledge/entity` | `name`, `node_id`, `limit`, `offset`, `types` | | one card for a name or an id (a concept and its page are one card); typed links under `relations`, every edge once, each list one of each kind in turn, `left_out` and `next_offset` for more, `types` to list one kind; a miss is `found: false` with close names, never an error |
| `context_pack` | `POST /agent/knowledge/task_context` | `query`, `scope`, `conversation_id`, `refresh` | `judge_relevance: false`, `widen: cache_only` | a bundle for the start of a task or after compaction; no model call |
| `delta` | `GET /agent/knowledge/changes` | `cursor`, `since`, `limit` | | notes added, changed, removed or renamed (with the id and path a renamed note had) since a cursor, and the next cursor; at least once, never skipping |
| `request_tools` | the server | `surface` | | what this connection offers and what it does not, and why; with `surface`, moves the session there (see "Surfaces") |
| `sign_in` | the server | `token_file`, `token` | | Track BB2: listed by a session whose collection asks for an access token and that has not signed in (with the collection's tools, which refuse every call until then: fix round BB-V1); served whenever called. On a connection with no collection (fix round BB-T1) it finds the collection the key belongs to and holds the key for that collection's checks, the connection staying without a collection: how an agent joins again or leaves from a shared connection. Reads the key file (preferred) or takes the token, checks it against the session's collection, and holds it for the session: every later call carries `Authorization: Bearer`, and the session's agent, grants and surface become the token's agent's. Its answer says who the session is now and at what level, with the token's short fingerprint; never the token. Refused: `token_unknown`, `token_expired`, `token_other_collection`, `token_file_unreadable`, `sign_in_no_collection`. See "Access tokens" |
| `welcome` | `GET /welcome` | | | Track AG1: what to tell a person who is new or asks how to start (`say`, at most one `ask`, `next`); the collection's own state on a bound connection; no model call |
| `help` | `GET /help` | `question` (required) | | Track AG1: a question about Kovium itself, answered from the user guide and the catalogue with each passage's section; no model call |
| `get_manual` | `POST /welcome/manual` | `update` | | Track AG6 (`manage` grant, listed in welcome mode and for a managing agent, never on a reading connection): on the person's yes to the offer of the manual (made by the tour and by `help`), fetches Kovium's own documentation as a collection so `help` answers from it; a copy of another version is reported and `update` fetches the matching one; free, not read-only (it writes the fetched copy into the install's own folder) |

Welcome mode (above) lists `welcome`, `help`, `get_manual` and two tools no bound connection
lists:

| tool | over | exposes | always sends | notes |
| --- | --- | --- | --- | --- |
| `try_example` | `POST /welcome/example` | `question` | | opens the example collection (built by code) and suggests three questions; with `question`, answers it from the example with sources; free; not read-only (it builds the example's derived files) |
| `tour` | `POST /welcome/tour` | `step`, `try_it`, `add`, `approval`, `reset`, `wait` | | the tour's steps, each relayed with its places and its question back (Track BA); `try_it` a step's free exercise; at `add`, the waiting page added `free`, or by the AI (`ai`: a plan with its estimate and `approval` first, run only with that approval); `reset`; cost `model` only for `add: ai` with an approval |

The setup tools (Track AG2) are on the default surface and listed only with
the `manage` grant (see "Setup and who may do what"); `check_setup` is listed
on every connection:

| tool | over | exposes | notes |
| --- | --- | --- | --- |
| `create_collection` | `POST /collections` | `name` (required), `holds`, `working_folder`, `place`, `folder`, `program` | a new collection; without `folder` or `place` it first returns where it goes as `ask` (Track AZ1); binds a connection with no collection to it once made |
| `open_collection` | `POST /collections/open` | `folder` or `collection`, `program` | an existing collection's folder, or one already on this computer by its name (Track BC1), with what did not travel; binds as above, so every agent sharing this connection opens it |
| `list_collections` | `GET /collections` | | Track BC1: the collections on this computer by name, with what each holds, how many notes, where, and `joined`; opens none; free |
| `join_collection` | `POST /collections/join` | `project` (required), `collection`, `folder`, `program`, `agent`, `read_only`, `level`, `length`, `tone`, `extra`, `confirm` | Track BC2 to BC4: this agent alone joins a collection under its own id: its own connection in the narrowest place its program offers, its caller agent's text naming the tools as its program shows them; the plan first (`asks` while the style is unanswered, then a `confirm` code), done on the person's yes; never binds this connection; see `docs/collections_v1.md` |
| `leave_collection` | `POST /collections/leave` | `collection`, `project` (required), `program`, `agent` | Track BC4: this agent's own connection and caller agent's text taken out, and (Track BB) its tokens taken back and its key file deleted; the collection stays |
| `issue_token` | `POST /collections/tokens` | `agent`, `project` (required), `level`, `may`, `expires`, `confirm` (`to` hidden: the owner's script's) | Track BB3: an access token for one agent, needing a manage entry that may grant: the sentence and a `confirm` code, then the token written to `<project>/.kovium/<collection>.token` and never returned; an agent never raises its own access with it |
| `list_tokens` | `GET /collections/tokens` | | Track BB3: the collection's tokens by short fingerprint, with agent, level, end date and last use; read only |
| `revoke_token` | `POST /collections/tokens/revoke` | `fingerprint` (required) | Track BB3: one token taken back from the next call; an agent's own, or another's with grant |
| `connect_agent` | `POST /collections/connect` | `program` (required), `agent` | this collection's own connection added to another agent program, reading only |
| `set_access` | `POST /collections/access` | `agent`, `level` (required), `may`, `per_run_tokens`, `per_day_tokens`, `confirm` | what an agent may do: the sentence and a `confirm` code, then applied with it |
| `change_setting` | `POST /collections/settings` | `option` (required), `value`, `add`, `remove`, `text`, `confirm` | one choice by its plain name, previewed then applied |
| `make_agent` | `POST /collections/agent` | `project` (required), `program`, `level`, `length`, `tone`, `extra`, `confirm` | Track AG4: this collection's caller agent added to the person's project beside its own agents, in the style they choose (`asks`: the offer, three questions and a free line) and the program's own format (Track AG7: Claude Code, Codex or OpenClaw; `program` omitted names the connection's own agent id); a preview, then applied with its `confirm`; needs `manage` with `settings`; see `docs/install_and_caller_agent_v1.md` |
| `nightly_ai` | `POST /welcome/nightly` | `answer` (required), `example` | Track AX (`manage` grant, on every connection): the person's own answer to the nightly AI question an `ask` put to them (code `nightly_ai_choice`, settled by the user): `yes` returns the levels, `no` or `light`, `medium`, `heavy` sets it; with `example`, `on` or `off` only shows the switch in the example; free |
| `check_setup` | `GET /collections/setup` | (`probe` hidden: always probes) | what works here, what is off or missing, who may do what; read only |
| `plan_adding` | `POST /sources/add` (`dry_run: true` always sent) | `paths`, `urls`, `kind`, `versions`, `token_budget` | Track AG3: the plan read from the files, with `say`, `needs`, `asks` (one question at a time, recommended answer first) and `start`; free, read-only |
| `start_adding` | `POST /sources/add` (`dry_run: false` always sent) | `approval` (required), `paths`, `urls`, `kind`, `versions`, `token_budget`, `keep_as_limit` | Track AG3: the run, only with the plan's approval, in the background; cost `model` |
| `adding_status` | `GET /sources/add/status` | `job_id`, `wait` | Track AG3: progress, and at the end the report word for word; read-only, free |
| `add_note` | `POST /sources/note` | `title` (required), `text` (required), `kind`, `confirm` | Track AY4: text from the conversation kept as a note of the collection, in any language. Without `confirm`: the plan (`say`: what will be added and how long it is, and that taking it in uses no tokens; `note.file`, the ASCII file name; a `confirm` code bound to the title, the text, the kind and the name); nothing written. With the code: written word for word as a new note in the collection's notes folder (a note written by hand: no field the source pipeline writes; `added_from: conversation`, `added_by`), taken in by code, `tokens: 0`. A code for other text, or one whose name was taken since, is refused `confirm_code_stale`. Needs `manage` with `add`, as the adding tools; free |

Each setup tool is free (but `start_adding`, which spends with an approval),
not read-only (but `plan_adding` and `adding_status`, which only read), and
calls its route with `access_mode: manage`; the route checks the collection's
policy again for the connection's agent, and the adding tools need its entry
to allow `add`. A compact result of the adding tools carries the calls that
follow as tool calls (`asks[].on`, `start`, `next`); AR's HTTP forms of the
same calls (`question`, the plan's `next.call`) and each file's row beyond what
happens to it and why are left to `view: "full"`.

The wider surface (AB3) adds, after the default surface's tools and in this order:

| tool | over | exposes | always sends | notes |
| --- | --- | --- | --- | --- |
| `read_next` | `POST /agent/knowledge/read_next` | `query` | `judge_relevance: false`, `widen: cache_only` | the items to read next for a question, each with why, where it is and the `read` call for its note; no model call |
| `review_queue` | `GET /agent/knowledge/review_queue` | `kind`, `note`, `limit`, `offset` | | the open items waiting for the owner's review, by kind, with the notes each concerns; read only |
| `codes` | `GET /agent/knowledge/codes` | `code` | | what a warning or refusal code means and what to do next; without `code`, an index of every code by action |
| `contract` | `GET /agent/knowledge/contract` | | | the answers' limits, warnings, trust signals and path form, at the depth the tools here use (`mode` hidden: they always rank at the default) |
| `capabilities` | `GET /agent/knowledge/capabilities` | | | the workspace, whether written answers can be had, the owner's ceiling, and this connection's surface, cap and grants; what is not ready as notices with fixes |

The write surface (AB4) adds, after the wider surface's tools and only with the
`propose` grant (see "Writes through review"):

| tool | over | exposes | always sends | notes |
| --- | --- | --- | --- | --- |
| `remember` | `POST /agent/knowledge/propose` | `statement`, `provenance`, `subject` | `kind: remember`, `access_mode: propose` | files a statement with what it rests on as a review item; `status: queued_for_review` |
| `forget` | `POST /agent/knowledge/propose` | `id`, `reason` | `kind: forget`, `access_mode: propose` | files a note or an item to withdraw, with why, as a review item |

All five wider tools are read-only, idempotent and free. `read_next` hides the answer
route's fields the reading list ignores (`limit`, `scope`, `conversation_id`,
`refresh`, `synthesize`) and those `recall` hides.

Every tool also takes `view` (below), except `request_tools`. The
connection-fixed fields are sent on every call (see "Scope from the
connection").

**`scope` on `recall`, `synthesize` and `context_pack`** (AB2c) names one
source (`{"note": …}`, `{"citekey": …}` or `{"identifier": …}`) and keeps the
answer to it: only items resting on that source are listed, `source_filter`
says what was kept, and `focus` names it. A source holding nothing of the
question lists no items and carries the notice `scope_matched_nothing`, whose
fix is the same `recall` without `scope`; every item listed is located in
that source; a note not ingested yet carries `scope_not_ingested` (its owner
ingests it; no tool here does). It is unrelated to the connection's
scope below, and was a label on `focus` only before AB2c (see the contract's
"`scope`: only this source").

**Bounds and defaults.** Every bound is shown in the schema and a value
outside it is refused as the tool's `invalid_request` (`argument`, `message`,
`type`, `input`), before any route is called: `recall`'s and `synthesize`'s
`limit` 1 to 50 and `read`'s `max_chars` 2,000 to 200,000 are this surface's
own (`bounds` in the table), since the routes adjust such values and warn
rather than refusing them (their HTTP callers sent any value before);
`overview`'s and `delta`'s `limit` 1 to 500, `entity`'s 1 to 50 and its
`offset` 0 to 100,000 are the routes' own. An answer lists at most a fixed number of items (5 at the depth
these tools ask for); a larger `limit` lists that many and says so
(`item_limit_capped`). One default differs from the route's, declared in the
table (`defaults`) and shown in the schema: `read`'s budget is 12,000
characters here, against the route's 32,000; the field's description names
the bounds, not a default, so each surface's schema shows its own. Reads of one to four notes came to 27,000 to 30,000 characters
at the route's budget on three real workspaces, paid again on every later
turn of the caller; at 12,000 they came to 13,000 to 17,000 with the
structure, and what did not fit is listed under "Left out" by heading, with
its size, to ask for by name (and `read_left_out_matching_sections` says when
a section that matches the question is among them).

**Annotations.** Every tool is non-destructive and closed-world, and every
read tool read-only. The two write tools are not read-only, are
non-destructive (they only file a review item), and are not idempotent: the
same proposal sent twice is one item, but the second filing is added to it
(who, what it rests on, when). Every read tool but `synthesize` is idempotent; `synthesize` is not (asked twice
it may answer differently, and each call costs again), and its title and the
first word of its description say it is expensive. MCP has no cost
annotation; those two places are where a harness's model looks.

**No model call but `synthesize`.** `recall` and `context_pack` always send
`widen: cache_only` and `judge_relevance: false`, so a question not asked
before never costs a brain-keeper call there (the cache still widens). A
test fails when a free tool could reach a model through a field it neither
fixes nor hides. `synthesize` runs on the brain keeper exactly as the answer
route with `synthesize: true` does over HTTP (its widening included), on the
owner's subscription: nothing is metered (`cost.metered_usd: 0`).

**Fields each tool hides, each a decision.** The answer route's `task`
(the task context's), `mode`, `node_types` and `facets` (they narrow ranking
in ways a caller needs the contract or the install's vocabulary to use; AB3
kept them hidden, since a tool's fields are the same on every surface) and
`evidence` (the written answer's) are not
offered by `recall`; `synthesize` hides `widen` too, and `context_pack` also
`limit` and `synthesize`, which the task context ignores.

**Conversations.** `conversation_id` and `refresh` pass through unchanged, so
the routes' conversation compaction (Track Q) applies: `"new"` starts one
(its id comes back in `conversation.id`), and an id sends only what that
conversation has not been sent yet.

**Input schemas are generated** from the route's request model
(`Field(description=...)`, types, defaults, bounds), never written by hand,
and hold only the fields the tool exposes, and `view` (from the `ToolView`
model in the table's module). An optional field (`X | None = None`) is
listed as its type alone, since an argument left out is already none and the
`anyOf` costs tokens; pydantic's titles are dropped. Each tool declares the
model's other fields hidden; a test fails when a model gains a field a tool
over it neither exposes nor hides, when a tool hides a required field that
neither the connection nor the tool's own values supply, when a field is
both exposed and sent, and when an exposed field has no description on its
model. The descriptions are the ones `/docs` shows: `AgentKnowledgeRequest`,
`KnowledgeReadRequest`, `KnowledgeEntityRequest` and the two query models
(`KnowledgeChangesQuery`, `KnowledgeOverviewQuery`), all in
`second_brain/api/app.py`.

A tool over a `GET` route sends its arguments as query values: a string, a
number, `true`/`false`, or a list of those. Anything else (an object) is
refused as `invalid_request` before the route is called, rather than
flattened into something else.

## Results

A result is one compact JSON text block. Every tool takes `view`:

- **`compact`** (the default): what a caller acts on, without what the
  connection already knows (`results.py`).
- **`full`**: the route's response unchanged, every field the service
  returns (the envelope: `status`, `data`, `warnings`, `generated_at`, `notices`),
  where the route has a view of its own its `view: full` too, with only what
  MCP adds: notices for ignored arguments, `fix.mcp` beside a fix and `mcp`
  beside a call hint.

The HTTP responses are unchanged. A success whose body was cut off, or is not
a JSON object, is never passed on as a result: it becomes a tool error with
`server_error`.

**The compact view** (C8 of the AB1 consumer test: about 900 characters of
schema, contract, caller, scope and policy repeated per call; each warning in
three places; fix calls spelling out the caller's own ids). It keeps:

- `status` and `data`, less the per-call boilerplate (`schema_version`,
  `contract_version`, `response_type`, `caller`, `scope`, `policy`,
  `contract`, `view`, `generated_at`) and less top-level fields that are
  empty (`null`, `[]`, `{}`); where the route has a compact view of its own
  (answer, task context, read), that view is the starting point;
- every warning once, as a notice: `code`, `message`, `detail` (the route's
  `warning_details` for it, when the notice had none), `where` (when deeper
  than the top-level list) and `fix`; a warnings list is dropped only when
  every code in it has its notice;
- a condition of the install or the workspace rather than of the call (the
  code registry marks it `about: install`: semantic search unavailable or its
  server unreachable, the question not mapped onto the vocabulary, the concept
  layer behind its sources) in full the first time a session meets it, with
  `semantic_search.how_to_enable`, and after that as `code` and `message`
  with `repeat: true`. Measured on three real workspaces: two or three such
  notices on every recall and task context, each said in full once;
- each step as an MCP caller takes it: `fix.next` with `action`, `say` and
  `mcp` (the tool call) where a tool takes it; where none does, `here` says
  `not available over this connection`, and for a step that changes or spends
  (a write, a maintenance call, a model call) `not available over this
  connection: the owner's to do`. `fix.verify` only when a tool can make the
  check; `consent` only when there is one. A refusal's `error.fix` the same;
- a call hint inside `data` the same: a hint a tool takes (an answer's
  `focus.read`, a card's `read`) as its tool call alone, one no tool takes (an
  unread region's `recover`) as `here`, and a hint naming the route that
  returns one note by its key (`related_ideas[].read`, a concept note's or the
  owner's work's `read`) as the `read` tool's call for that key, which `read`
  resolves (`NOTE_BY_KEY_ROUTES` in the table);
- an item's `chunk`, and a fact's, only when it says more than its title (a
  long fact's label is cut at 160 characters, its snippet at 200).

No HTTP route is left in a compact result: a test scans every string of
every tool's compact result for a method and path or a path under one of the
service's own first segments, and the instruction texts, reasons and schemas
for any route or phase name.

Provenance (`where`, `also_in`, `sources`), trust signals (`confidence`,
`focus`, the item flags `image_derived`, `unverified`, `source_missing`,
`unread_regions`) and every id are kept as the route gave them; a test
checks that no item id is lost.

**MEMORY_VERBS v1 names**, where a verb exists, in the compact view, with our
own fields alongside (clients ignore unknown fields):

| tool | names |
| --- | --- |
| `recall`, `context_pack` | `knowledge_items` become `results[]` (`slug` = the item id, `title`, `chunk`, beside our `type`, `score`, `where`, `also_in`, flags) and `facts[]` for the platform's fact types, definitions and extracts (table rows): `fact_id`, `fact`, `provenance` (the `where`), `type`, `score` |
| `synthesize` | first `synthesis_status` (`answered`, `partial`, `not_answered`, `unavailable`), `answer`, `sources`, `gaps` (the answer's caveats), `cost` (`tokens` of the written answer, `widening_billed_tokens`, `billed_tokens` the two together, `metered_usd: 0`, `backend`), `synthesis` (the rest, less the per-stage trace), then the items as for `recall` |
| `delta` | `changes` become `pages[]` (MEMORY_VERBS v1 names delta's list `pages[]`, beside `facts[]` and `threads[]`: `docs/gbrain_reference_v1.md`, section 6.4; kept after review), each entry as the route gives it; `has_more`, `next_cursor` |
| `entity` | already the verb's: `found`, the card, `suggestions` |
| `remember`, `forget` (AB4) | `id` (the review item's id) first, and `status`, ours: `queued_for_review`, `already_queued`, `already_dismissed`, `already_accepted` or `already_withdrawn`, never the protocol's `inserted`, `duplicate` or `superseded`; so the write verbs claim no conformance (below) |
| `read_next` (AB3) | `suggestions` become `results[]`: `slug` (the item id), `title`, `type`, `reason` (what tied it to the question: the question's words its label holds, its chapter, the notes the question reaches), `score`, `where` and `also_in` as an answer's items write them, `note` and its `read` tool call, `freshness`. Nothing tied to the question: no item, and the notice `read_next_nothing_matched` with `overview` as its fix (the route's own behaviour since the fix round) |

The other wider tools' compact views (AB3):

- `review_queue`: the route's answer less the boilerplate (it is already
  what a caller needs, the first five findings of an item included);
- `codes`: with `code` (a name, several, or the start of one: `semantic_`
  lists every code it begins, at most 40) or `area`, each code listed as
  `message`, `action`, `say`, `next` (the tool call, or `here`), `verify`
  where a tool can check it, `consent`, `family`, `matched_by`, and `about:
  install` where it is one; with neither, an index: `codes_total`, `by_area`
  (the codes of the areas a caller of these tools meets, `answers`, `reads`,
  `connection` and `requests`, each under what it asks of a caller),
  `other_areas` (how many codes intake, background work and the rest of the
  platform raise, reached with `area`) and `lookup`, since every code in full
  is about 61,000 characters (43,000 compact) and each notice already
  carries its own code's meaning and fix;
- `contract`: the mode's limits, `warning_taxonomy` and
  `warning_taxonomy_is` (it is partial: the ranking's own warnings),
  `confidence_fields` (the fields a compact answer here carries, not the full
  answer's), `supported_response_types`, `compatibility`, `paths` (`form` and
  `separator`; how to open one over HTTP is the full view's) and
  `capabilities_ready` (each capability's readiness, one word); left to the
  full view: the identity, scope and access requirements (the connection
  fixes them), the governance text and each capability's prose;
- `capabilities`: the route's report and `connection` (the server's
  addition, in both views).

The reading list's fixed `capability_contracts` text (about 2,500
characters, the same on every call) is per-call boilerplate in the compact
view; the contract tool states it once, as `capabilities_ready`.

Measured on the synthetic workspace (characters, full and compact):
`read_next` 12,364 and 1,330; `review_queue` (one item) 1,161 and 322;
`codes` with two codes 950 and 712, with neither `code` nor `area` 65,747
and 2,509 (the index; 5,043 when it named every code); `contract` 6,172 and
1,785; `capabilities` 2,070 and 1,127;
`request_tools` 3,137.

Inputs keep the routes' names (`query`, not `question`): the schemas are
generated from the routes' models.

**Sizes measured** (characters of the result text). On the synthetic
workspace of the tests, per call: `recall` 31,046 full, 9,475 in the route's
own compact view over HTTP, 6,953 compact over MCP (the first call of a
session, its install conditions in full); `context_pack` 34,371 / 9,804 /
6,116; `synthesize` (no brain keeper) 29,471 / 7,905 / 4,483; `read` 17,318 /
5,380 / 4,739; `entity` 2,007 full, 1,070 compact; an entity miss 1,549 / 394;
`delta` 1,965 / 1,164; `overview` 2,016 / 1,185. On three real workspaces, a
recall of each collection's most-named concept: 47,877 full to 13,071 compact,
35,922 to 8,173, 52,027 to 10,618 when first measured; an entity card at
limit 50: 9,662 to 8,650. A `read` of one to four notes: 27,076 to 29,868
compact at the route's budget, 13,300 to 16,781 at this surface's (13,300
to 16,702 after the fix round's second review, with the source passages
restored). After Track AB2b's card (every edge once, each list one of each kind in
turn), on one real install, compact: a note naming 87 items and linked by 19
notes, 8,354 at the default limit (6,539 before) and 19,620 at limit 50
(25,362 before); a concept with its page, 4,110 (2,805 before, when the card
was the page's and listed none of the 18 notes naming the concept).

**No `structuredContent`, no `outputSchema`** (decided in AB2). The MCP
specification (2025-11-25) lets a tool declare an `outputSchema` and return
`structuredContent` matching it, and asks for the same JSON as text beside it
for older clients. Not used here:

- a client that forwards both to its model pays for the result twice;
- an `outputSchema` is listed in every session, used or not: a schema that
  says something about these envelopes would be several thousand characters
  per tool (the answer's alone has dozens of nested fields), and one that
  says nothing (`{"type": "object"}`) would cost about 30 characters per tool
  for no gain;
- the envelopes are versioned by the routes' contracts
  (`docs/agent_memory_contract_v1.md`), which a model reads as JSON text
  without a schema.

Revisit when harnesses use `structuredContent` without forwarding the text.

## A note's header fields (Track AB6)

Found by the owner on a second machine: an assistant that knew a collection
only through these tools answered "the note doesn't say" who attended a
meeting whose note lists the attendees and the person responsible in its
frontmatter. No tool passed the frontmatter on, so it had been given less
than the note holds. The same gap hid a paper's authors, a contract's
signatories, a recipe's servings. Now, through the governed routes (so HTTP
callers get the same):

- `read` opens each note with its header, `## [note] Header —
  <note>#frontmatter:<lines>`, one `- name: value` line per field, and, when
  the note is read from a Markdown source with a frontmatter of its own, that
  source's fields the note does not repeat (`## [source] Header —
  <source>#frontmatter:<lines>`); with other sections named it comes only
  when small, else it is listed under "Left out", and `Header` names it;
- `entity` gives a note's card a `header`: `where`, `fields` (`{name:
  value}`), `left_out` and `cut` when something did not fit;
- `recall` and `context_pack` carry `header_fields`: the fields of the ranked
  notes (and of their sources' own frontmatter) that the question names,
  a field it names by name ("channel", "published") first, the focus's notes
  first and alone when one source carries the answer, each with `where`;
  empty, and so absent from the compact view, when none does
  (`docs/agent_memory_contract_v1.md`, "`header_fields`").

The header is the note's frontmatter less the platform's bookkeeping (ids,
hashes, recorded paths, the pipeline's machinery, the times it writes, and the
type, title, description, lifecycle and trust keys reported elsewhere): one
list in the platform's own vocabulary, `PLATFORM_FRONTMATTER` in
`second_brain/notes/contract.py`, documented in
`docs/note_format_contract_v1.md` ("The header"). Every other key is shown,
whatever an install or its owner called it. A value is cut at 200 characters;
a header at 1,200 characters and 24 fields (in a read, at most a sixth of the
note's share of the budget, with no floor, so several notes read together keep
the budget); what did not fit is named. `recall` names at most three notes and
three fields each. A field is cited as its `where`: the file (the note or its
source), `frontmatter` and its lines, the form an item's provenance takes, and
the compact view keeps it. A note is also found by a value of a field its type
declares as identifying it (a meeting's attendees, a paper's authors, an
install's own declarations in `config/note_types.json`), read from the
Markdown when the note was ingested before, so an existing install needs no
re-ingestion; a question naming no such value ranks exactly as before, and a
word an answer shows is never reported absent.

The descriptions of `read`, `entity` and `recall` say so in a few words (the
Header block, a note's `header` fields, `header_fields`): listing a session
costs 147 characters more on every surface (the default 12,078 to 12,225, the
wider 15,919 to 16,066).

**Token cost**, measured (characters of the result text). On the suite's
synthetic collection (its notes carry little frontmatter): `read` compact
4,739 to 4,843, `entity` compact 2,720 to 2,832, `recall` and `context_pack`
compact unchanged (6,953 and 4,437; no field matched their questions), the
full views 19 to 378 more. On the synthetic collection of the AB6 tests, a
"who attended" `recall` came to 6,020 characters compact, 314 of them
`header_fields`, and the meeting note's `read` to 1,048 compact, 213 of them
its header. On copies of three real installs, a `read` of four notes each
(three source notes with the most header fields and a concept page) at this
surface's budget grew by 0 to 1,080 characters a note: research papers'
headers (authors, year, venue, kind, citekey, cites, facets, 14 fields)
793 to 1,044 characters, web pages' (url and site since the fix round, which
left the fetch's own records out) 186 to 198, a hand-made note's two fields
226, a concept page's aliases 72 to 99, notes with no field of their own
nothing; an entity card grew by 0 to 728 characters (a paper's card 540 to
728, its facets not repeated; a web page's 216 to 228; a concept's card
nothing, the header being on note cards only).

**Starting a session** (AB6 fix round, C62). The consumer saw one start take
486 s, and four more on the same workspace 1.6 to 1.9 s. Not reproduced:
opening that workspace's copy took 0.1 s in process, and a database held by
another process's write does not stall a start, it fails it after the store's
30 s busy wait. Since then that failure is a plain refusal on standard error
(`refused to start: workspace '…': its database stayed locked by another
process for 30 s …; start again when it is done`, exit 2), and an opening
slower than 5 s is said there with how long it took.

## What a session costs to list

Every listed tool's name, description and input schema, and the
instructions, are put in the model's context at the start of each session.
Measured on the synthetic workspace (the line counting the notes differs per
workspace), as the tool definitions the SDK's client receives serialised
compactly (`model_dump(by_alias, exclude_none)`) and the instructions:

| session | tools | tool definitions | instructions | total |
| --- | --- | --- | --- | --- |
| `default`, no `config/mcp.json` (main before AB3, same method) | 8 | 10,610 | 1,468 | 12,078 |
| `default`, no `config/mcp.json` (AB3) | 8 | 10,610 | 1,468 | 12,078 |
| `default`, ceiling `wider` | 8 | 10,610 | 1,583 | 12,193 |
| started on `wider` | 13 | 14,079 | 1,829 | 15,908 |
| started on `write` (ceiling `write`) | 13 | 14,079 | 1,829 | 15,908 |
| AB4: `default`, no `config/mcp.json` | 8 | 10,610 | 1,468 | 12,078 |
| AB4: started on `wider` (or on `write` without the grant) | 13 | 14,090 | 1,829 | 15,919 |
| AB4: started on `write`, `propose` granted | 15 | 17,629 | 2,132 | 19,761 |
| AB4: `default`, `propose` granted | 8 | 10,610 | 1,747 | 12,357 |
| AB6 fix round: `default`, no `config/mcp.json` | 8 | 10,757 | 1,468 | 12,225 |
| AB6 fix round: started on `wider` | 13 | 14,237 | 1,829 | 16,066 |
| AG1: `default`, no `config/mcp.json` | 10 | 12,305 | 1,719 | 14,024 |
| AG1: welcome mode (no collection) | 4 | 3,917 | 984 | 4,901 |
| AG2: `default`, no `config/mcp.json` (reads only) | 11 | 12,855 | 1,795 | 14,650 |
| AG2: `default`, the agent named `manage` | 16 | 19,671 | 2,552 | 22,223 |
| AG2: welcome mode (no collection) | 7 | 7,030 | 1,516 | 8,546 |

AG2 adds `check_setup` to every connection (626 characters with its line in
the instructions), the five setup tools only where the agent manages the
collection (6,816 characters of definitions and the line saying so, 757), and
three tools and two lines to welcome mode (3,645). A reading connection pays
the 626.

AG1 grew the default listing on purpose: `welcome` and `help` are on every
connection (the first-run plan's D3c), 1,494 characters of definitions (the
other eight measure 10,811, as on main before AG1, up 54 since AB6 from later tracks' field
descriptions), and the instructions 242 more (the two tools' summaries and the
line saying when to call them), about 1,750 characters in all on every
session. A welcome-mode session lists four tools for 4,901 characters.

AB4 left the default listing as it was. The wider definitions grew by 11
characters (`codes`' `area` names the new `proposals` area). The two write
tools cost 3,539 characters of definitions and 303 of instructions, paid only
by a session on `write` with the grant; a granted connection on another
surface pays the 156-character line saying it may propose.

The default surface's listing did not grow: the eight tools' definitions are
unchanged, and the instructions add one line (115 characters) only where a
wider set can be reached. The wider surface costs 3,469 more characters of
definitions, paid only by a session on it (3,263 before the fix round gave
`codes` its `area`). (AB2 reported 9,369 for the eight
tools by another serialisation; the comparison above uses one method on both
trees.) Before AB2 the trimming of the generated schemas (nullable `anyOf`
collapsed, `view`'s description cut, the annotation title not repeated) took
the tools from 10,580 to 8,990 by AB2's method; the declared bounds and the
read budget's default added 379.

## Token cost measured at the CLI

The plan's comparison ("Token cost"), run once on 2026-10-17 by
`scripts/measure_mcp_token_cost.py` (AB5): the suite's synthetic collection
(`tests/fixtures/focus_collection.py`, three notes), its own three questions
(one source, one section, spread over sources), Claude Code 2.1.240 headless
(`claude -p --output-format json`, `--model sonnet`, sealed:
`--setting-sources ''`, `--strict-mcp-config`, `--no-session-persistence`, a
fresh empty working folder, every metered credential removed), in two arms:

- **http**: the caller skills' way. The HTTP app on a loopback port, the
  research-answering skill's core in the prompt, `Bash` limited to `curl` and
  `Read` for the skill's references (`scripts/measure_answer_steps.py`'s
  brain arm, reused);
- **mcp**: `--mcp-config` with this server alone, no built-in tool, no skill.

Tokens are the CLI's own report (`modelUsage`, every model of the run summed:
input, output, cache reads, cache writes), never a response's account of
itself. The brain keeper was off on both sides, so no answer used a model
but the caller's.

**Fixed cost per session** (a one-turn "reply OK", no tool called, less the
same with no tool and no server: 10,292 tokens):

| arm | session | above the bare session |
| --- | --- | --- |
| http (skill text, `Bash` and `Read` definitions) | 21,018 | 10,726 |
| mcp (the eight tools' listing and the instructions) | 14,749 | 4,457 |

**Per question** (tokens: total; input / output / cache reads / cache
writes; turns; the governed calls the audit recorded):

| question | http | mcp |
| --- | --- | --- |
| one source | 67,144; 3,205 / 1,146 / 47,639 / 15,154; 3 turns; answer, read | 50,453; 972 / 567 / 37,423 / 11,491; 3 turns; recall, read |
| one section | 72,290; 3,210 / 1,378 / 49,038 / 18,664; 3 turns; answer twice | 51,990; 977 / 521 / 38,262 / 12,230; 3 turns; recall, read |
| spread | 69,133; 3,198 / 1,299 / 48,047 / 16,589; 3 turns; answer, read | 32,263; 963 / 643 / 21,027 / 9,630; 2 turns; recall |
| **total** | **208,567** (63,843 without cache reads) | **134,706** (37,994 without cache reads) |

MCP cost 35% less in total and 40% less without cache reads. Most of it is
the fixed part, paid on every turn: the skill text the HTTP way sends (a
caller over MCP needs none) is larger than the tool listing. Not like for
like in one respect: the skill also sets the answer's layout (its parts and
its reliability note), and the HTTP answers were longer (1,818 to 2,000
characters against 506 to 1,074; output 3,823 tokens against 1,731), which
is the skill's doing, not the transport's. One run of three questions on a
three-note collection: the per-call results of a large collection weigh more
there, and are the same on both surfaces (see "Sizes measured").

## Surfaces

Three sets of tools, nested: `default` < `wider` < `write`. A session is on
one at a time and lists the tools of its surface and every narrower one;
`SURFACES` in the tool table is built that way, and a test pins the nesting.

| surface | adds | reachable when |
| --- | --- | --- |
| `default` | the eight read tools | always |
| `wider` | `read_next`, `review_queue`, `codes`, `contract`, `capabilities` | the ceiling is `wider` or `write` |
| `write` | `remember`, `forget` (AB4), each needing the `propose` grant | the ceiling is `write`; its tools also need the grant |

**The ceiling** is the workspace's own `config/mcp.json`, edited by its owner
and by no caller (nothing over a connection writes it):

```json
{"max_surface": "default", "allow_proposals": false}
```

- `max_surface`: the widest surface any session on this workspace may reach.
  Absent (no file, or no key): `default`, the safe value for an install
  nothing is known about. A value that is not a surface name, a file that is
  not JSON or not an object: `default` too, and the problem is said
  (`capabilities`' `ceiling.problems`, the notice `mcp_config_invalid`, and
  standard error at start).
- `allow_proposals`: whether a session may hold the `propose` grant the
  write tools need (AB4). Absent or anything but `true`: false.
- `display_name` and `description` share the file (see "What it says it is").

**The precedence rule, exactly.** The ceiling wins over the launch, the
launch over the caller:

1. `cap` = the ceiling's `max_surface`, narrowed to `--max-surface` when
   that is given and narrower (`cap_set_by`: `ceiling` or `launch`); a launch
   cap wider than the ceiling is cut to it and said;
2. the session starts on `--surface` (default `default`), narrowed to `cap`
   (said on standard error when cut, and in one line of the instructions);
3. `request_tools` may move the session to any surface whose rank is at or
   below `cap`, wider or narrower, any number of times; beyond `cap` nothing
   moves.

The launch's cap is decided when the server starts, like the workspace and
the agent. The cap in force is recomputed on every listing, call and
resource read: the narrower of the launch's cap and `config/mcp.json` as it
reads now (missing or unreadable: `default`); a grant is held only while the
file allows it. A session on a surface beyond that cap is cut down to it, and
the client is told what changed. Nothing moves a session up by itself: when
the file allows more again (an editor's half-written file, a sharing
violation on Windows, an owner's raise), the caller widens with
`request_tools`, never above the launch's cap. So the capabilities report,
which reads the file afresh, and the connection's cap always agree, within
the launch's cap.

Surface names are taken in any case (`Wider`, ` wider `), in `request_tools`,
on the command line and in `config/mcp.json`; the file is read as UTF-8 with
or without a byte-order mark (Windows PowerShell 5.1's `Out-File` writes one).

**The `propose` grant** (AB4) is held only when the ceiling's
`allow_proposals` is true, its `max_surface` (cut by the launch's
`--max-surface`) reaches `write`, and the launch asked for proposals
(`--allow-proposals`). Like the cap, it is re-read on every listing, call and
resource read: the file no longer allowing it drops it at the next one (the
client is told its tools changed), and the file allowing it again restores
it, never beyond what the launch asked. A tool is offered when its surface is
within the session's and its grant, if it needs one, is held.

**The write surface without the grant.** It is reachable under a ceiling of
`write` and lists exactly what `wider` lists: its two tools need the grant.
`request_tools` says of it that its tools need the `propose` grant, and who
grants it. Before AB4 it listed nothing more and said "no tool of it is offered
by this version" (decided then: no placeholder rows).

**Fails closed.** A tool outside the session's surface is not listed, and a
call to it by name is refused before any route is reached, audited like
every refusal (`details.tool_surface` names the tool's surface):

| code | outcome | when | fix |
| --- | --- | --- | --- |
| `tool_outside_surface` | `refused` | the tool's surface is within the cap, wider than the session's | `next.mcp`: `request_tools` with that surface |
| `surface_not_allowed` | `denied` | the tool's surface is beyond the cap | who allows it and how, in `detail`: the owner's `max_surface` in `config/mcp.json`, or a wider `--max-surface` where the launch capped it |
| `not_permitted` | `denied` | the tool needs a grant the connection lacks | who grants it: the owner's `allow_proposals` and `max_surface: write`, and `--allow-proposals` at the launch |

A tool whose surface is not a known name is never offered.

**`notifications/tools/list_changed`.** When `request_tools` changes what the
session lists, or `create_collection` / `open_collection` binds a connection
with no collection, the server sends the notification before the result, and
the initialize response declares `tools.listChanged: true` only where the list
can change: two reachable surfaces listing different tools, or (Track AY3) a
connection started with no collection, which a binding in the same session
changes. Before AY3 a welcome-mode connection declared it false (all its
surfaces list the same welcome tools) though the binding sent the
notification; the owner's Claude Code then kept the welcome tools after
`create_collection`, and its caller agent spent many calls looking for a way
around before asking for a restart. Claude Code's documentation
(`https://code.claude.com/docs/en/mcp`, "Dynamic tool updates", read
2026-10-23) says that on a `list_changed` notification an interactive session
fetches the server's new list, without reconnecting; it does not say whether
it requires the declared capability. `resources.listChanged` is declared on a
welcome-mode connection too, since a binding brings the capabilities resource.
On a connection capped at `default` and bound from its start it is false and
never sent. Checked: the SDK's own client
(`mcp==2.3.0`, handshake era `2025-11-25`) receives it over stdio and in
process, and its next `tools/list` returns the new set (its response cache
does not hold the old one). On the `2026-07-28` per-request era the SDK
delivers change notifications only on a `subscriptions/listen` stream the
client opened, and this server serves no such stream: a client on that era
does not hear the change. Checked with a real harness: Claude Code 2.1.240
listed the wider tools in the same session after `request_tools`, through
the notification, with no restart (the clean-consumer round of 2026-10-17).
**A harness that ignores the notification**, or
lists tools only once, keeps showing the old set: whoever configures it
starts the server with `--surface wider` instead (within the ceiling). The
server still answers a call by name to any tool of the session's surface.

## `request_tools`

Served by the server itself, with no route. Arguments: `surface` (optional).
It answers, in the compact shape:

- `surface`: the one this session is on now; `previous` when the call moved
  it; `widened` (true when it moved to a wider one); `cap`, the widest it may
  reach;
- `listing`: present when the session's tools changed, saying the client was
  told and what a harness that does not listen does;
- `tools`: each offered tool with its `summary` and `cost` (`free` or
  `model`);
- `surfaces`: every surface, each with `adds` (the tools it adds to the
  narrower one), `what` (in words), `reachable`, and `why_not` when it is
  not, saying who can allow it and how;
- `grants`: the grants this connection holds (`propose`, or none: an empty
  list is left out of the compact view);
- `not_offered`: what the service does that no tool here does, by reason:
  `what` (the reason, in words that name no route and no phase of this
  project) and `operations` (how many). The routes themselves are in the
  table below, for maintainers, not in what a connected agent is told.

Asked for a surface within the cap, it moves the session there (narrowing
included) and sends the list change when the tools changed; asked for the
surface it is on, nothing changes. Beyond the cap nothing moves, and the
notice `surface_not_allowed` says why (the ceiling or the launch's cap) and
who can allow it. A name that is not a surface moves nothing, with
`surface_not_offered` naming the surfaces (the name echoed at most 40
characters long). Audited like every tool, with `route: null`, `outcome`
`allowed`, `denied` (beyond the cap, `code: surface_not_allowed`) or
`refused` (not a surface, `code: surface_not_offered`), and
`details.asked_surface` (the surface asked for, or `<not a surface>` with
`asked_length`: never the caller's text), `previous_surface`,
`session_surface` and `listing_changed`.

## Resources

One resource, `kovium://capabilities` (`application/json`), the same
report as the `capabilities` tool's compact result, the connection's state
included, on the same surface as the tool (`wider`): the ceiling caps both.
Below it the resource is not listed, and a read is refused as a call to the
tool would be: the JSON-RPC error `-32002` with `data.code`
(`tool_outside_surface` or `surface_not_allowed`), audited. Widening lists
it, and the client is told (`notifications/resources/list_changed`, sent
only when the resources listed changed, not on every change of tools;
`resources.listChanged` is declared whenever the capabilities tool is offered
on one surface the launch's cap reaches and not on another, a session started
on `wider` included, since narrowing drops it). Some
harnesses show resources to their user and never to their model, so the tool
stays beside it. Each read is audited (`action: resource_read`, `tool:
capabilities`). Another URI is the same error, with no `code`.

Decided in review: a first version offered the resource on every surface,
since a resource costs nothing in the listing; it showed a default-capped
connection the brain keeper's backends and contradicted "the ceiling caps
what a session may reach".

## Writes through review

AB4. Two tools on the `write` surface, `remember` and `forget`, over one
governed route, `POST /agent/knowledge/propose`, the only route that takes
the access mode `propose` (contract: `docs/agent_memory_contract_v1.md`).
Each call files one review work item for the workspace's owner and writes
nothing else: no note, no graph change, so `recall` returns nothing of a
proposal until the owner accepts it.

**Granted only by both halves**, re-read on every check:

1. the owner's: `{"max_surface": "write", "allow_proposals": true}` in the
   workspace's `config/mcp.json`;
2. the launch's: `--allow-proposals` (and a `--max-surface`, if given, of
   `write`).

Without both the tools are not listed, a call by name is refused
(`not_permitted`, its detail naming both halves and who sets each) and
audited as `denied`, and `request_tools` says the write surface's tools need
the grant (moved there anyway, the answer carries the notice
`write_tools_need_grant`, naming both halves). The owner taking the grant back
mid-session drops it at the next call or listing, and the client is told its
tools changed. The owner's `allow_proposals` is also what the route itself
checks, over HTTP too (`proposals_not_allowed`, 403): the switch means one
thing on every surface (decided in the fix round; the owner may revisit it).

**`remember`**: `statement` (one statement standing on its own),
`provenance` (required: a `note` of the collection, a `source` file in the
workspace, or the `conversation` it came from, with an optional exact
`quote`), `subject` (optional, it titles the note). **`forget`**: `id` (a
note's path, key or id, or an item's id as `recall` or `entity` gives it) and
`reason`. The schemas list `statement` and `provenance`, and `id` and
`reason`, as required. `forget` withdraws notes and items of the collection,
not a proposal: a pending proposal is dismissed by the owner alone, and no
tool takes one back. Text a reviewer cannot see (control characters,
zero-width and bidi marks, line separators other than a newline) is refused
(`proposal_hidden_characters`); instruction-shaped prose a reader can see is
filed and named for the owner (`content_flags`).

**What a call answers**: `id` (the review item's), `status`, and the route's
own fields: `queued_for_review` for a new item, `already_queued` when the same
proposal waits (this filing is added to it), `already_dismissed` when the
owner dismissed it (nothing is raised again), `already_accepted` when the
statement is a note already, `already_withdrawn` when what it would forget was
withdrawn, `already_in_collection` when an item of the collection states the
statement already (nothing is filed; `in_collection` names the item and its
notes). Each comes with its notice; `proposal_queued`'s check is the
`review_queue` tool with the proposal's kind, which lists it with
`resolve_means`, `dismiss_means` and `proposed_by`. A pending statement that
shares most of its words with another pending one is filed with
`similar_pending`, and each names the other. "The same proposal" is the
statement normalised (any script, case, white space, wrapping quotes and
sentence-final marks aside; every other mark counts), or what is withdrawn;
the contract says how.

**Resolution is the owner's**, through the review routes over HTTP (or the
UI), never over this connection: `POST /review/action` with
`resolve_work_item` or `dismiss_work_item` stays in "Not offered over MCP".
An accepted `remember` becomes a note of the collection under
`sources/wiki/contributed/`, attributed to the agent and to the person who
accepted it, with the provenance given, and is ingested: from then on
`recall` returns it like any note, its items carrying `contributed` (who
proposed it, who accepted it, what it rests on). An accepted `forget`
withdraws, never deleting: the note is moved aside byte for byte to
`maintenance/withdrawn/`, or a statement's line is taken out of the notes
stating it, each kept as it was; either leaves retrieval and the change feed
says so, and `POST /maintenance/withdrawals/restore` puts it back. A
withdrawal outlives the note being made again from its source: the re-made
note is kept aside and not ingested (`withdrawn_note_kept_out`), a withdrawn
statement is taken out again (`withdrawn_statement_kept_out`). The agent that
proposed an item never decides it (409, `proposal_decided_by_its_proposer`):
`verified` names who did. That stops only the same-id mistake: nothing
authenticates the decider yet (see "Limitations").

**What a session writes.** The reads write no note and no graph node. An
answer (`recall`, `context_pack`, `synthesize`) updates the note access
history (`note_access` in the workspace's database: when a note last appeared
in a ranked result and how often; the one state not derived from the
Markdown, kept for staleness, see the ingestion spec's "Access History") and
appends the answer log (`logs/knowledge_events.jsonl`, read by
`/ops/knowledge_events`); over HTTP the answer route does the same. Every
call appends the governance audit. A proposal adds its review item to
`maintenance/maintenance_work_items.json`; the empty
`maintenance_work_items.json.lock` beside it is the lock two processes take
turns on, left in place on purpose.

**Why MEMORY_VERBS write conformance is not claimed.** The protocol freezes
`remember`'s `status` to `inserted`, `duplicate` or `superseded`, and its
conformance check expects `recall` to return a fact at once. A statement
waiting for a person's review is none of those and is not recalled, so the
honest status is reported and the write verbs are accepted as
non-conformant (the plan's decision 1). `forget` withdraws as the protocol's
does, with an audit trail and never a deletion, but through review too, and
it takes a note as well as a fact, since a remembered fact is a note here.

Tested in process (`tests/integration/test_mcp_write_tools.py`: not listed
and refused without either half, in four configurations; one item and no
Markdown or graph change with both; the full view equal to the HTTP answer
in the same state; the grant dropped and restored as the owner edits the
file; recall returning the accepted note) and over stdio with every outbound
connection made to fail (`tests/system/test_mcp_stdio.py`).

## Not offered over MCP

Rule 4 of the plan: every governed route a caller agent is sent to is either
a tool or listed, with its reason, in `NOT_OFFERED`
(`second_brain/mcp_server/tools.py`). "Sent to" means the routes under
`/agent/knowledge/`, every route a fix in the code registry calls or checks,
and every route the caller skill (research answering) tells an agent to
call: 42 routes when AB2 was built, 44 with AB3's two (11 of them a tool's,
33 not offered). A test fails on one that is neither
(`tests/system/test_documentation_matches_routes.py`).

AB3 made tools of the contract, the codes and the reading list, and of the
two routes it added (the capabilities report, the review queue's open
items).

| reason (`REASONS`) | routes |
| --- | --- |
| `narrower_read`: narrower reads, kept off these tools to keep them small | `GET /review/source`, `GET /concepts`, `GET /owner-work`, `GET /owner-work/{key}`, `GET /query/facets`, `GET /ops/facets`, `GET /maintenance/lint`, `GET /ops/governance_audit`, `GET /provenance/{entity_id}` |
| `review_owner`: the owner's review view, every source's review state with the owner's options; the open items are offered on the wider surface | `POST /review/queue` |
| `service`: about the service rather than the collection; the owner checks it | `GET /ops/status`, `GET /ops/self_check`, `GET /ops/embeddings`, `POST /ops/embeddings/reindex`, `POST /ops/embeddings/rebuild`, `GET /ops/semantic_expansion`, `GET /ops/workspaces`, `GET /ops/quarantine`, `GET /health`, `GET /format-templates`, `GET /brain_keeper/skills` |
| `background`: the state of a background job | `GET /brain_keeper/jobs/{job_id}`, `GET /brain_keeper/jobs` |
| `writes`: it writes to the collection directly or decides a review item; over this connection a change is at most a proposal, which the owner decides | `POST /review/action`, `POST /ideas` |
| `owner`: the owner's operation | `POST /sources/process`, `POST /sources/process_book`, `POST /sources/reconcile`, `POST /sources/reread`, `POST /notes/native/extract`, `POST /maintenance/inspect`, `POST /maintenance/vocabulary` (`POST /sources/add` is the adding tools since Track AG3) |
| `spends`: it spends model tokens on the owner's behalf, with the owner's consent | `POST /sources/figures`, `POST /sources/equations`, `POST /concepts/sync`, `POST /concepts/compare`, `POST /query/gaps/interpret` |

A fix whose call is one of these says, in the compact view, that it is not
available over this connection, and whose it is (see "Results").

## Same answer both ways

Rule 5 of the plan, tested in `tests/integration/test_mcp_read_tools.py` on
the synthetic workspace, against a table written in the test apart from the
tool table (`EXPECTED`: each tool's method, route and the values its call
must carry), so a wrong `fixed`, `sends` or `defaults` row fails:

- every request the server sends, recorded at the in-process caller, is the
  caller's arguments, `EXPECTED`'s values and the connection's ids, nothing
  else;
- the full view equals the HTTP response to that request;
- the compact view keeps, independently of `results.compact`: every item id
  of the route's own compact answer with its provenance, every notice code
  once, each fix as the tool call where a tool takes the route and as `here`
  where none does, the cursor and counts (`has_more`, `next_cursor`,
  `found`, `returned`, `total`), none of the boilerplate, and no HTTP route.

Fields that differ by nature are excluded by name in one list (`NATURAL`):
`generated_at`, `mcp` (what the server adds), `job_id`, and a note's reading
counts and times (`read_count`, `last_read_at`, ...), which every read moves
whichever surface made it. `synthesize` runs through a test brain keeper on
both sides. Found in review: the first version compared the compact view with
`results.compact` of the same input, which checked the function against
itself.

The wider tools (AB3) are held to the same rule in
`tests/integration/test_mcp_surfaces.py`, against a table of their own: the
request recorded, the full view equal to the HTTP response (for
`capabilities`, less `data.connection`, the server's addition by nature),
the same notice codes in the compact view, no HTTP route in it, and each
item kept (the reading list's ids in order, the review queue's ids and
notes, the codes asked for).

## Scope from the connection

The connection-fixed fields, `agent_id`, `workspace_id`, `scope_kind`,
`scope_id` and `access_mode`, are sent by the server on every call
(`scope_kind: workspace`, `scope_id` the workspace, `access_mode: read_only`;
for the two write tools `access_mode: propose`, sent only while the
connection holds that grant).
No tool takes them. One passed anyway is ignored, and the result carries the
notice `scope_argument_ignored` naming it. An argument the tool does not
expose is ignored too, with `argument_not_offered`. Another workspace is
reached only through a server started for it.

## Refusals

A route's refusal becomes a tool error (`isError: true`) whose text is the
route's refusal body: `detail` and `error` (code, message and fix, as
`docs/next_steps_v1.md` describes). A refusal without an `error` of its own is
given one named by its status. The HTTP status itself is not in the result
(the code says what it means); the audit keeps it.

**Invalid arguments are named as the tool's.** Over HTTP a validation refusal
keeps FastAPI's shape (`loc: ["body", ...]`, the whole request echoed back).
Over MCP each entry of `detail` names the `argument` (no location prefix;
none for a rule across arguments), the `message` (without pydantic's
"Value error, "), its `type` and the `input`, without the connection-fixed
fields the caller never sent; `error.message` reads "The tool's arguments are
not valid (see `detail`)." The HTTP response is unchanged.

Refusals made before any route is reached:

| code | when |
| --- | --- |
| `tool_not_offered` | no tool by that name on this connection |
| `tool_outside_surface` | the tool is on a wider surface this connection may widen to (AB3; its fix is the `request_tools` call) |
| `surface_not_allowed` | the tool is on a surface beyond this connection's cap (AB3; its `detail` says who can allow it) |
| `not_permitted` | the tool needs a grant this connection does not hold (`remember` and `forget` without `propose`; the detail says who grants it) |

These are compacted like a route's refusal (the fix as a tool call, no
empty `verify`), unless the call asked for `view: "full"`.

## Both surfaces at once

The HTTP service and MCP sessions may serve one workspace at the same time.
Found in review and fixed with this phase:

- **The workspace registry is written whole.** It is written beside itself
  and swapped in (`os.replace`), so a reader sees the old file or the new one.
  A registry that still cannot be read after two seconds is an error
  (`WorkspaceRegistryUnreadable`), never a reason to write: before, a reader
  that met a half-written file saved an empty registry over it, unregistering
  every workspace.
- **The governance audit and the answer log lose no record.** Each record is
  one `os.write` of a whole line under a lock (`ops/appending.py`): append
  mode alone is not atomic on Windows, and two processes lost records.
- **The database waits rather than failing.** Writers take turns; a
  connection waits up to 30 seconds (`BUSY_TIMEOUT_SECONDS`) for the other
  process's transaction instead of Python's default five. The rollback
  journal is kept, not WAL, so the database stays one file that backup,
  export and restore copy as it is.

Tested with real processes: an HTTP app ingesting notes, registering a
workspace and answering while a stdio MCP session calls its tool on the same
workspace; two processes appending the audit; two writing notes to one
database; a registry rewritten while another process reads it.

AB2 put two more logs on the same helper, since its tools write them from
the server's process too: the brain keeper's call metrics
(`logs/agent_calls.jsonl`, written on every model call, so by `synthesize`)
and the equation-read ledger (`logs/equation_reads.jsonl`, written by
`read`). Two processes appending 200 records each to both lost and mangled
nothing; the same test failed on the plain appends (2 runs of 2).

Still not covered: two processes *registering* at the same moment can each
write the registry from their own read, and the later write wins (the file
stays whole; one registration may need repeating). An MCP server never
writes the registry. The logs of intake and review (which MCP does not offer)
are still appended without the lock.

## Fixes in MCP form

Where a notice's `fix.next.call` or a refusal's `error.fix.next.call` is a
call a listed tool takes (same method and route, and the tool's fixed
values agreeing with the call), the fix also carries:

```json
"mcp": {"tool": "<tool>", "arguments": {"query": "..."},
        "fill": ["notes"], "not_offered": ["evidence"]}
```

- `arguments`: the call's own, less the connection-fixed fields;
- `fill`: arguments the caller supplies, as in the call;
- `not_offered`: arguments of the call the tool does not expose (the step
  cannot be taken exactly over MCP; absent when it can).

The HTTP `call` stays beside it in the full view; the compact view keeps the
tool call alone (see "Results"). A test walks the code registry: every
registered code whose call or check a tool covers gets an MCP form, without
the connection-fixed fields (rule 2).

A call **hint** inside `data` gets the same: a dict with a `route` such as
`"POST /agent/knowledge/read"` and a `body` (an answer's `focus.read`, an
entity card's `read`) carries `mcp` beside it when a tool takes the call.
A value the tool always sends otherwise (`widen: "auto"` asked of `recall`,
which always sends `cache_only`) is named in `not_offered`.

## Setup and who may do what

Track AG2 (D2, D4, D4a of the first-run plan); the contract is
`docs/collections_v1.md`.

- **One connection per collection.** A connection is bound to one collection
  at a time. A connection with no collection (welcome mode) is bound by a
  successful `create_collection` or `open_collection` in the same session: its
  tools, scope, grants and audit become the collection's, the client is told
  its tool list changed, `data.connection` says `{bound, collection, tools}`,
  and the binding is recorded so the next session starts bound (above). The
  instructions the client read at the start are not sent again; the tool's
  `say` and the welcome's state carry what changed. Its last `say` line
  (Track AY3, `collection.tools_follow`) tells the person that the tools are
  now in the conversation and, where they do not show, to start a new
  conversation, which opens the collection by itself; the welcome-mode
  instructions tell the caller agent the same, and not to look for files,
  folders or ports to work around it. A bound connection is
  never rebound: a further collection gets a connection of its own in the
  agent program (`program`).
- **The `manage` grant** is held only over a local transport (`stdio`, and the
  tests' in-process `memory`), never a remote one: by a connection with no
  collection, and by one whose agent the collection's policy names at
  `manage`, read again on every call like the ceiling, so a grant given or
  taken back acts at the next call. Without it the setup tools are not
  listed, and a call by name is refused `access_not_granted` (or
  `setup_needs_local_connection` over a remote transport), its words for the
  person kept in the compact refusal (`fix.next.user_text`).
- **The policy per agent** is the `agents` table of the collection's
  `config/mcp.json` (`read`, `propose`, `manage`). An agent named at `propose`
  or `manage` has the owner's half of the proposal grant and may reach the
  `write` surface (`read_ceiling(root, agent)`); the launch's half
  (`--allow-proposals`) is still asked for, and a connection bound by a setup
  tool, or written by `connect_agent`, asks for it. An agent never raises its
  own access (`access_self_raise`).
- **What a bound manager reads first.** One more line in the instructions:
  the setup tools are its to call on the person's word, each change shown as
  one sentence and applied with its `confirm` code.

## Access tokens (Track BB)

The contract is [[access-tokens-v1|`access_tokens_v1.md`]]. Where a
collection's `require_token` is on (off for every collection until its owner
turns it on):

- **Before signing in** a session may call only `sign_in` and `help`, and lists
  the collection's tools as well, as the launch's agent would be offered them
  (fix round BB-V1: Claude Code and Codex list once, when a conversation
  starts, and never again; listing only the two left them without the
  collection's tools after signing in). This holds whether it
  was launched with `--workspace` or is a welcome connection just bound to
  such a collection (`open_collection` then says `connection.sign_in_first`
  and the sentence `token.sign_in_first`). The instructions say to call
  `sign_in` with `token_file` first, naming the usual place
  (`<project>/.kovium/<collection>.token`), and that the launch's agent
  id is only a hint. Any other tool called by name is refused `token_required`
  with the `sign_in` call as its fix. `help` is asked as a connection with no
  collection asks it, so it needs no token. `listChanged` is declared, and the
  client is told when signing in brings the collection's tools.
- **`sign_in`** holds the token for the whole session, in memory only. The
  in-process caller then sends `Authorization: Bearer` on every route call, so
  MCP and HTTP pass through the same gate, and the session's agent becomes
  the token's (its grants and surface recomputed from the policy). A token
  taken back during the session is refused at the next call by the route.
  A binding to another collection drops the sign-in.
- **The local connection's secret.** Over a local transport the in-process
  caller also sends `X-Kovium-Local`, a secret random per process and
  kept in memory; only a join or `issue_token` that arrives with it hands out
  a token without a signed-in manager. An HTTP caller cannot know it.
- **The audit** of every call after signing in carries `token: {short,
  agent}`, the first 8 characters of the token's SHA-256 and its agent, never
  the token; so does a refused `sign_in`, by the presented token's short
  fingerprint.
- Prefer `token_file`: a token passed as `token` is an argument, and some
  programs log every tool argument (OpenClaw's Codex runtime does).

## Audit

Every tool call writes one record to the workspace's governance audit, read
with `GET /ops/governance_audit` (`surface=mcp` filters them), in addition to
the route's own record:

```json
{"surface": "mcp", "action": "tool_call", "outcome": "allowed",
 "caller": {"agent_id": "...", "workspace_id": "..."},
 "scope": {"scope_kind": "workspace", "scope_id": "...", "workspace_id": "..."},
 "policy": {"access_mode": "read_only", "grants": []},
 "details": {"tool": "entity", "route": "POST /agent/knowledge/entity", "transport": "stdio",
             "status_code": 200, "code": null, "arguments": ["name"],
             "ignored_arguments": [], "view": "compact", "duration_ms": 12}}
```

`outcome` is `allowed` (the route answered), `refused` (the route refused,
the tool is not offered, or it is on a wider surface the session may widen
to), `denied` (a grant is missing, or the tool's surface is beyond the cap)
or `error` (the route failed). Metadata only: argument names (`view` among
them when it was given), never their values; `view` is the shape the result
was given in. `policy.access_mode` is the access the route was called with
(`propose` for the write tools, AB4) and `policy.grants` the grants held. Never
a proposal's statement or reason. `details.session_surface` (AB3) is the surface the session was
on; a call refused for its surface also has `tool_surface`.
`request_tools`, served by the server, has `route: null` and no route record
of its own, and records `asked_surface`, `previous_surface` and
`listing_changed`. A read of the capabilities resource is `action:
resource_read`, with the route's own record beside it. A call that bound a
connection with no collection (Track AG2) is recorded in the collection it
bound to, with `bound_to`; the setup routes add their own records
(`surface: collections`).

## Long calls

When a request carries a progress token, the server sends a progress
notification every five seconds while the route is pending (`progress`: the
seconds elapsed; `message`: `waiting on <method> <route>: <n> s`). Without a
token nothing is sent. A harness's own tool timeout still applies; "Wiring a
harness" gives each one's default and what to set. Measured on a real workspace: one
`synthesize` took 19.4 s and sent three notifications.

A failure inside the pending route is raised as itself, never wrapped in the
progress task group's exception group: before AB2 a query value refused
there (`QueryValueRefused`) escaped as an unhandled error, with no audit
record, whenever a progress token was sent. A `GET` tool's query is now also
checked before the call.

## Standard output

Over stdio, standard output carries JSON-RPC and nothing else. The server
claims it before anything is imported and points descriptor 1 at standard
error (on Windows the process's standard-output handle too), so a `print` in
a route, a log line, a direct write to descriptor 1 and a child process's
output (the brain keeper's agent CLI) all land on standard error. The SDK
points standard input at the null device while serving, so a child process
reads nothing from the protocol either. Logging is configured to standard
error.

## No network, no bill

The server makes no outbound connection: routes are called in process, and
the SDK's HTTP transports are not used. A test runs a whole stdio session with
every non-loopback connection and name lookup made to fail. The billing
scanner (`tests/system/test_billing_safety.py`) covers the package, and
`httpx2`, the HTTP client the SDK brings, is on its list of network modules
no module may import. The SDK's telemetry middleware is removed.

## Versions

- FastAPI: 0.115 or later (`pyproject.toml`): the changes and overview
  routes take their query as one model (`Annotated[Model, Query()]`), which
  0.115 introduced; 0.114 fails when the routes are defined (found in review).
- SDK: `mcp==2.3.0`, pinned exactly (extras `mcp` and `dev`). Only
  `second_brain/mcp_server/server.py` and `session.py` import it; a test
  fails if any other module does, and another checks that building the app
  loads none of it.
- Protocol: the SDK negotiates. A client asking for a handshake-era version
  gets it, up to `2025-11-25` (what the SDK's own client and current harnesses
  negotiate, checked in the tests); the SDK also serves the `2026-07-28`
  per-request era, where `server/discover` carries the same instructions.
- Harnesses (configuration syntax checked against their documentation on
  2026-10-17; the pages are named in each harness's part of "Wiring a
  harness"): Claude Code 2.1.240 (Windows) and 2.1.299 (Linux), Codex 0.160.0
  and OpenClaw 2026.9.5 (Linux), live-verified; Cursor from its documentation
  only.
- Packaging: setuptools 68 or later, every subpackage found
  (`[tool.setuptools.packages.find]`) and the package's data files listed
  (`[tool.setuptools.package-data]`); a test fails when a tracked module or
  data file inside the package would be left out of a non-editable install.

## Limitations

- Of the four harnesses only Claude Code was seen to run the snippets; the
  Codex, Cursor and OpenClaw entries follow their documentation, and
  Cursor's documentation names no tool timeout.
- An install outside a checkout reads its prompts, templates and install
  configuration from the checkout it is given at start, so the installed
  code and that checkout should be the same version. The HTTP service has
  no such flag: it runs from a checkout, as it always has.
- The write tools only propose: nothing an MCP caller sends is written until
  the owner resolves it in review, which no tool does. MEMORY_VERBS write
  conformance is not claimed (see "Writes through review").
- Over HTTP, `propose` is open to whatever reaches the loopback port of a
  workspace whose owner allows proposals while its access tokens are off (no
  authentication, as for every route); with them on (Track BB,
  [[access-tokens-v1|`access_tokens_v1.md`]]) it needs a token whose agent
  the policy names at `propose` or higher. The owner decides as an
  identity other than the proposer's; nothing checks that it is a person, so
  a caller that reaches the HTTP service can decide its own proposal under
  another `agent_id`, and the note then records `verified: human:<that id>`.
  The proposer check only stops the same-id mistake.
- An owner's narrowing of `config/mcp.json` (or a moment it cannot be read)
  cuts a running session at its next call; widening back is the caller's
  `request_tools`, within the launch's cap.
- A client on the `2026-07-28` per-request era does not hear
  `notifications/tools/list_changed` (the SDK delivers it only on a
  `subscriptions/listen` stream, which this server does not serve); it sees
  the new tools when it lists them again. A harness that never lists again
  is started with `--surface`.
- The tool table's fields are per tool, not per surface: the wider surface
  does not offer `recall` the answer route's `mode`, `node_types` or
  `facets`.
- The compact view is a reshaping of the route's response in the server
  (`results.py`), not a route option: an HTTP caller has the route's own
  `view: compact` (answers and reads) but not the MCP shape. Rule 5's test
  pins the two to one function.
- Only the read verbs carry MEMORY_VERBS names, and only in the compact view;
  inputs keep the routes' names (`query`, not `question`), since the schemas
  are generated from the routes' models.
- `recall` and `context_pack` never widen a question by a model call
  (`widen: cache_only`): a question not asked before over HTTP or by
  `synthesize` is ranked on its own words.
- The full view keeps the routes' HTTP hints and calls, as they are.
- The install conditions said once are said once per process: a harness that
  starts a new server per session hears them again in each session.
- A route call cannot be stopped once started: a cancelled request does not
  stop the route's work.
- The note counts in the instructions are read when the server starts; a
  long session does not see them change (`overview` does).
- The first call after start was reported slow (1.7 to 3.2 s) by the AB1
  consumer test; it did not reproduce in AB2: 0.04 to 0.09 s in five cold
  sessions over three real workspaces on an idle machine, 0.09 to 0.16 s
  with the full test suite running beside it (`initialize` 1.4 to 1.7 s idle,
  2.6 to 2.8 s loaded). No warm-up was added.
- On an install whose embedding settings are on but whose embedding server is
  not running, the first call that ranks waits for the connection to fail
  (about 2 s, measured on Windows), and the embedding client then leaves the
  server alone for `down_cooldown_seconds` (default 60, in the workspace's
  `config/embeddings.json`): measured on a real workspace, a recall took
  11.4 s each time with the cooldown switched off (four searches of a
  decomposed question, each waiting) and 5.2 s, then 2.6 to 3.2 s, with it.
  A server process pays the first wait once per cooldown; the rest of a
  recall's time there is ranking (decomposed searches, gap and centrality
  signals), not the embedding server, and is left to a performance phase.
- No authentication: whoever can start the process can read the workspace,
  as with its files. See the plan's "Later: authentication".
