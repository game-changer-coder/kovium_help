---
type: "documentation"
title: "Access tokens: a caller agent proves who it is (Track BB)"
description: "The contract for access tokens: what a token is, where it is kept, the switch that asks for them, how a caller shows one over MCP and over HTTP, how one is issued and taken back, how the owner's own web page signs in, the refusals, and..."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing Kovium"
doc_path: "docs/access_tokens_v1.md"
doc_sha256: "854e983138108a29471739afaaec94026d447d7edd8bd1732527abfbc0cfb556"
---

# Access tokens: a caller agent proves who it is (Track BB)

The contract for access tokens: what a token is, where it is kept, the switch
that asks for them, how a caller shows one over MCP and over HTTP, how one is
issued and taken back, how the owner's own web page signs in, the refusals,
and what tokens do **not** protect against. Built 2026-10-25 from
`knowledge_layer_improvement_plan_v1.md`
("Track BB") and its handover,
`track_bb_access_tokens_handover_v1.md`.
A person's view of it is the user guide's "Access keys" section
([[user-guide-v1|`user_guide_v1.md`]]); `help` answers from that.

## Why

Before Track BB a caller agent said who it was and Kovium
believed it: an MCP connection's `--agent` was whatever its launch line said,
and an HTTP call's `agent_id` whatever its body said. Any caller that could
reach the server could call as any registered agent, and OpenClaw's agents
all call under one id. The access policy, the budgets and the audit guarded
against mistakes between one person's own agents, not against a caller that
pretends.

## The token

- **Format:** `sbt_` followed by `secrets.token_urlsafe(32)`. The prefix makes
  a leaked token easy to find in a log search.
- **For one km-instance and one agent.** A token issued in one collection is
  refused by every other (`token_other_collection`).
- **Fingerprint:** the SHA-256 (hex) of the whole token. Lists and the audit
  show its first 8 characters (the *short fingerprint*), never the token.
- **End date:** none by default; one may be set when it is issued (`--expires`
  on the script, `expires` on the tool and the route: a date, which ends at the
  end of that day in UTC, or a full time).

## Where tokens are kept

Only fingerprints, in the collection's own `config/mcp.json`, beside the
access policy (`agents`):

```json
{
  "require_token": false,
  "agents": {"scout": {"level": "read"}},
  "tokens": {
    "<sha256 hex>": {"agent": "scout", "made_at": "2026-10-26T09:00:00Z",
                     "made_by": "owner-script", "ends_at": null,
                     "last_used": "2026-10-26T09:05:00Z"}
  }
}
```

- `made_by`: `owner-script` (the owner's script), `join` (a join on the
  person's yes), or the agent id that issued it with `issue_token`.
- `last_used` is written at sign-in and otherwise at most once an hour per
  token, and kept apart, in `config/token_use.json` (`{fingerprint: time}`),
  so the hot path never rewrites `config/mcp.json`: a revoke or a policy change
  made by another process cannot be undone by a stamp (fix round BB-S5).
- **Every read-modify-write of `config/mcp.json`** (the policy, the tokens, the
  switch) holds a lock across processes: the operating system's own lock
  (`msvcrt.locking` on Windows, `fcntl.flock` elsewhere) on
  `config/.mcp.json.lock`, a file that stays, waited for up to 15 seconds and
  released when its holder closes it or dies, so it is never taken over by its
  age and two waiters never both hold it; one lock per collection, nothing
  process-wide held while waiting (fix round BB-T4).
- **Revoking deletes the entry.** A revoked token is then unknown, and is
  refused as `token_unknown`, with nothing more said about it.
- **A copied or moved collection folder carries no usable token**: a
  fingerprint cannot be turned back into a token.
- The code is `second_brain/collection_setup/tokens.py`.

**What the token's holder may do stays the access policy's entry for its
agent** (`read`, `propose` or `manage`, with manage's abilities and budgets;
[[collections-v1|`collections_v1.md`]]). Issuing a token for a level writes
that entry through the same path as `set_access`; lowering an entry acts at
the holder's next call.

## The switch: `require_token`

Per collection, in its `config/mcp.json`, **off for every collection, old or
new**, until its owner turns it on (decided 2026-10-25; whether new
collections start with it on is the owner's later decision). It is the
catalogue's choice `access_keys` ("Access keys for AI programs"), changed only
through `change_setting` (`POST /collections/settings`, `option: access_keys`,
`value: on|off`), previewed and applied with its `confirm` code.

- **Off:** everything works exactly as before the track; a token shown is not
  checked.
- **Fails closed** (fix round BB-S6). Every write of `config/mcp.json` keeps a
  mark beside it, `config/.require_token`, while the switch is on. A settings
  file that cannot be read (half-written, held by another program, not a JSON
  object) or is gone is read again briefly, then, where the mark is there,
  every call is refused (`token_settings_unreadable`, 503): an unreadable file
  never means "no token needed". A collection that never turned tokens on has
  no mark and keeps its safe default, as before (the unreadable file is
  reported as `mcp_config_invalid`).
- **Never written over** (fix round BB-T5). A change to the file (the policy,
  a token, the switch, the name), by anyone, the owner's own processes
  included, reads it first and is refused when it cannot be read or is not a
  JSON object (`access_settings_unreadable`, 503, nothing changed): an
  unreadable file is never taken for an empty one and replaced, which would
  have dropped `require_token` and its mark.
- **On:** a call that names this collection without a valid token for it is
  refused in plain words.
- **Turning it on is refused to an agent that holds no token for the
  collection** (`token_switch_needs_key`): it would refuse that agent's own
  next call. The agent issues itself one first (`issue_token` for its own id,
  at its present level), signs in, then changes the setting. The owner's own
  processes and signed-in browser are exempt.

## One gate for every call

`second_brain/security/token_gate.py`:

1. A small ASGI middleware reads what each request shows into a context
   variable: the bearer token, the system header, the owner's session cookie,
   and the in-process MCP caller's per-process secret. Starlette copies the
   context into its threadpool, so sync routes see it (tested).
2. A dependency of every route reads the collection the request names (path,
   query or JSON body, `workspace_id`, or in permissive mode the agent's
   default) and checks it **before the request's own fields are validated** and
   before the collection is opened, so a refused call registers nothing.
3. The scope gateway `resolved_context` checks again for the collection it
   resolved, so a route that reaches a collection another way is held too.

**The agent.** When a valid token is shown, its agent is the caller. A request
that sends another `agent_id` is refused (`token_agent_mismatch`); one that
leaves it out gets the token's agent written in, so every route below sees it.

**What a token's holder may call** by its entry's level: every read (any `GET`,
the governed `/agent/knowledge/*` reads, a read tool's route); a proposal
(`/agent/knowledge/propose`) needs `propose`; a tool's route needs the level
of the tool's own access mode, whatever the call says (`/sources/note`,
`/sources/add`, `/sources/add/status`, `/welcome/manual`, `/welcome/nightly`
need `manage`, and the note and adding routes then check `add` themselves:
fix round BB-S2); the setup and token routes (`/collections*`) check the entry
themselves; any other write is an owner's operation and needs `manage`; and
any `DELETE` (unregistering a collection, `DELETE /ops/workspaces/{id}`) is the
owner's own act, refused to every token whatever it may do (BB-S10). Refused
with `access_not_granted`. The agent a route acts as is always the token's
own: the body is given the token's agent whenever it names none, whatever the
query says, and the scope gateway refuses a route acting as any other agent,
a default such as `welcome` included (BB-S3).

**Exempt:**

- welcome mode, and any call that names no collection: there is nothing to
  check;
- Kovium's own processes: a valid `X-Kovium-System`
  header (`security/system_caller.py`): the nightly tasks, the owner's local
  scripts, `scripts/access_token.py`;
- the owner's signed-in browser (below).

**Files and folders a call names** (fix round BB-T3). A protected collection's
notes could be read through another collection's file routes (ingest a file
of it into an open collection, then ask there). Now every call, on every
route, has the values of its file and folder fields read every way the routes
read them (`path`, `paths`, `source_path`, `source_paths`, `folder`,
`folders`, `file`, `files`, `note`, `notes`, `target`, `to`,
`root_path`, `project`, `working_folder`, in the path, the query or the
body: in full, as a declared source root `name:rest`, relative to the named
collection, its checkout spelling and the checkout, relative to the folder
the service runs in; each stripped and made real, `..` and links followed).
One that lands at or inside a registered collection that asks for a token is
refused without that collection's own token (or the owner's own process),
whichever collection the call names. The route-coverage test derives these
routes from their request fields (a plain JSON body included) and tries each
with the protected file named in full, padded, relative and through `..`.

**Every spelling of the same file** (fix round BB-U1). On Windows the
long-path prefix (`\\?\C:\...`, `\\?\UNC\...`) and a share of this
computer's own drives (`\\localhost\C$\...`, `\\127.0.0.1\C$\...`, by the
computer's own name too) are read as the drive path before anything is
touched; a share on any other computer is refused to every access key while a
collection here asks for keys (`path_not_local`, 403), never reached. And a
place is matched to a protected collection by the file system's own identity
of the folder (`st_dev`, `st_ino` of it or a folder above it) as well as by
spelling, so an 8.3 short name, a junction or another alias lands in the same
check. The coverage test tries each of these spellings on Windows.

**Copying out: the owner and that collection's managers** (fix round BB-U4;
the owner decided on 2026-10-26: managers may copy out). Such a file, named by
a call about another collection (taking it in with `/ingest/file`,
`/normalize/file`, `/sources/add`, reading it as a source there), would be
copied where that collection's key no longer guards it. So it is allowed only
to the owner's own processes and signed-in web page, and to a valid key of
**that** protected collection whose agent its policy names at `manage` (any
abilities); the call must still be allowed in the collection it is about, as
for any call. A reading or proposing key of the protected collection, a key
of any other collection, and no key are refused
(`token_copy_not_allowed`, 403). A call about the protected collection
itself, or a setup call naming its folder (`/collections*`), keeps the token
rule above. The same collection is told by the file system's identity
(`os.path.samefile`), not by spelling.

## Showing a token

**Over HTTP:** `Authorization: Bearer <token>` on every request.

**Over MCP: the `sign_in` tool**, served by the server itself, in every mode:
- `token_file` (preferred): the full path of the key file. The server reads
  it, so the token never enters the model's context, its transcript or the
  program's logs (OpenClaw's Codex runtime logs every tool argument). Fix
  round BB-S8: the name must end in `.token`; a share (a UNC path, which on
  Windows reaches out to another computer), a device or pipe (`/dev/...`,
  `\\.\...`, `NUL`, `CON`) is refused before anything is opened; only a plain
  file is read, and no further than 256 bytes; a missing or unreadable file is
  refused in exactly the words an unknown key gets (`token_unknown`), so the
  answer never says whether a file exists.
- `token` (fallback): the token as text.

It checks the token against the session's collection and holds it **for the
whole session**: the in-process caller then sends `Authorization: Bearer` on
every call, so MCP and HTTP pass through the same gate, and the session's
agent becomes the token's (the launch's `--agent` is a hint once a token
decides). Its answer never repeats the token: it says who the session is now,
in which collection, at what level, and the token's short fingerprint.

**Before signing in,** a session whose collection requires a token may call
only `sign_in` and `help` (a launch with `--workspace`, and a welcome connection
bound to such a collection alike), but it **lists the collection's tools too**,
as the launch's agent would be offered them (fix round BB-V1): Claude Code,
headless and interactive, and Codex list a connection's tools once, when a
conversation starts, and never again, whatever `list_changed` says, so a
session that listed only the two never reached the collection after signing
in. Its instructions say to sign in first, with the key file the caller
agent's text names, and that the listed tools refuse every call until then.
Any other tool called is refused with `token_required`, whose fix is the
`sign_in` call (`fix.next.mcp` in the compact view). `help` answers from the
install's guide and needs no token. A wrong, unknown or expired token is
refused and the session stays as it was. A tool the token's agent is offered
beyond what the launch's agent was (a manager's setup tools on a connection
started for a reader) appears only in a client that lists again; the reading
tools are always there.

A token is **never written into a shared settings file** (OpenClaw's server
list, a project's `.mcp.json`, a launch line): every agent sharing that file
would hold it.

## The key file an agent holds

- one line, the token;
- readable by its owner only: `0600` on Linux and macOS; on Windows, where
  `0600` does nothing (a file takes its folder's access list), the system's
  own `icacls` gives it an access list of its own that grants the logged-in
  user alone (fix round BB-S12); where `icacls` cannot run, the file keeps its
  folder's list, and so does the moment between writing it and setting it;
- in the agent's own place: `<project>/.kovium/<collection id>.token`,
  where `<project>` is the project folder (Claude Code, Codex, Cursor) or the
  agent's own workspace folder (OpenClaw);
- when that folder is inside a git repository, `/.kovium/` (the path
  from the repository's top) is added to its `info/exclude` (for a worktree,
  the shared one its `commondir` names, the only one git reads), which keeps
  it out of git without touching the person's `.gitignore`; a file the owner's
  script writes anywhere else (`--to`) is excluded by its own path, never its
  folder (fix round BB-S7).

## Issuing and taking back

**The owner's script**, run on this computer, in process, as the system
(`X-Kovium-System`):

```bash
python scripts/access_token.py issue --workspace <id> --agent <id> --level read|propose|manage \
    [--may add,settings,grant] [--expires 2026-12-31] --to <file> [--show]
python scripts/access_token.py list --workspace <id>
python scripts/access_token.py revoke --workspace <id> <short fingerprint>
```

`issue` writes the token to `--to` (`0600`) and prints it only with `--show`;
it writes the agent's entry at `--level` (`--may` for manage). `list` prints
each token's short fingerprint, agent, level, end date and last use.

**Conversation tools**, for a manage entry that may `grant` (free, on the
setup surface over a local connection):

- `issue_token` `{agent, project, level?, may?, expires?, confirm?}`: without
  `confirm`, the plain sentence and a `confirm` code, as `set_access` does;
  with it, the token is written to `<project>/.kovium/<id>.token` (the
  folder that agent works in) and never returned. An agent never raises its
  own access with it; it may issue itself a token at its present level.
- `list_tokens`: the listing.
- `revoke_token` `{fingerprint}`: the short fingerprint as listed.

**HTTP routes** behind them (see [[api-spec-v1|`api_spec_v1.md`]], 6h):
`POST /collections/tokens`, `GET /collections/tokens`,
`POST /collections/tokens/revoke`. Over plain HTTP, issuing needs a caller
signed in with a token whose agent may grant, the local MCP connection, or the
owner's own processes (`token_issue_not_allowed` otherwise). `to` (any file)
is the owner's script's; an agent names `project`.

Every issue and revoke is audited with `token: {short, agent}`.

## Joining hands out the token

With the switch on, `join_collection` ([[collections-v1|`collections_v1.md`]],
"An agent joins a collection in one conversation") issues the joining agent's
token on the person's yes and writes it to its key file; the plan says so in
one sentence before the yes. The caller agent's text it writes tells the agent
to call `sign_in` with `token_file` set to that file at the start of every
conversation. `leave_collection` revokes the agent's tokens for the
collection and deletes the file.

**The local connection.** The in-process caller sends a per-process secret
header (`X-Kovium-Local`), random at app start and kept in memory only,
which an HTTP caller cannot know.

**Who may join or leave as an agent id** where the collection asks for tokens
(fix round BB-S1; the plan, the yes and a leave are all checked, before
anything changes):

1. the owner's own process (the system header) or signed-in browser: any id;
2. a caller that shows a token of this collection whose agent is that id, or
   whose agent may grant: that id. Over HTTP the bearer token; over MCP a
   session signed in with `sign_in`, a welcome connection included: there
   `sign_in` finds the collection the key belongs to and the connection stays
   without a collection, which is how an agent joins again or leaves from a
   shared connection (sign in with its key file, then leave), a reading agent
   too. **A file the caller names is never read as proof** (fix round BB-T1:
   naming another agent's project folder used to let anyone act as it);
3. the local MCP connection alone (its per-process secret): only an id the
   collection does not know yet (no policy entry, no token), which then joins
   **reading only**, never claiming an unmanaged collection or keeping a
   higher entry;
4. anything else is refused: `token_required` (401) for an id the collection
   knows, `token_issue_not_allowed` (403) for a new id over plain HTTP; a leave
   always says `token_required`. The words are the same whatever the id.

A join's `confirm` code proves the plan was shown, not who said yes, which is
why the local connection alone cannot act for a known id: any program on this
computer can start one.

**Opening** a collection that asks for tokens, by its name or its folder
(`open_collection`, `POST /collections/open`), is checked like a call naming it
by its id (fix round BB-S4): it needs a token for it, or the owner. The check runs on the very folder that is opened, normalized as the code that opens it reads it (spaces and line ends stripped, `..` and links followed, case as the system compares it), before anything is registered or claimed (BB-T2); making a new collection at or inside one that asks for a token needs its token too. The route-coverage test derives these routes from their request fields (`collection`, `folder`), so a new one cannot slip by. A welcome
connection therefore never binds itself to such a collection; an agent joins
it instead.

## The owner's web page

Not exempt (decided 2026-10-25). The start scripts
(`scripts/run_kovium.py`, `.sh`) print a one-time sign-in link once the
server answers:

```
http://127.0.0.1:<port>/ui/sign-in?once=<code>
```

- `POST /ui/sign-in/code` (system header only) makes the code; the server keeps
  only its hash, in memory, for 10 minutes. `scripts/ui_sign_in_link.py` asks
  for one at any time.
- `GET /ui/sign-in?once=<code>[&next=<page>]` signs this browser in: an
  HttpOnly, `SameSite=Strict` session cookie, the code deleted, and a redirect
  to `next` (a page of this server only) or `/`. A used or unknown code answers
  a page saying it was already used, an old one a page saying it expired (401;
  fix round BB-S11). A browser that is not signed in and opens the page for a
  collection that asks for keys gets a plain page saying how to sign in (401),
  never the API's refusal; the page's own calls stay gated.
- The gate accepts that cookie as the owner. Sessions are kept in memory: a
  restart signs the browser out, and the start script prints a new link.

## Refusals

Each is registered with its next step and its plain words for a person
([[next-steps-v1|`next_steps_v1.md`]]).

**What is not said** (fix round BB-S9). Where a collection asks for tokens, no
refusal names another agent (the "who can allow it" of `access_not_granted`
names no manager), and to a caller showing no key for it a listing shows the
collection by its name only (`asks_for_key: true`): `GET /collections` without
what it holds, its note count, its place or `joined`; `GET /ops/workspaces`
without its registered agents, its folder or its paths. A token refusal never
names the token's agent, and a missing key file is refused in the words an
unknown key gets. One thing is still told, to the local connection only: a
join or a leave refused because the id is already known there says so by being
refused (a new id would be let in); the words are the same whatever the id.

| code | status | when | next step |
|---|---|---|---|
| `token_required` | 401 | the switch is on and no token was shown | `sign_in` (MCP) or `Authorization: Bearer` (HTTP); the owner issues one with `scripts/access_token.py issue` |
| `token_unknown` | 401 | no such token in this collection (never issued, or revoked) | ask the owner for a token for this collection |
| `token_other_collection` | 401 | the token belongs to another registered collection | sign in with this collection's token |
| `token_expired` | 401 | past its end date | ask the owner for a new one |
| `token_agent_mismatch` | 403 | the `agent_id` sent differs from the token's agent | leave `agent_id` out, or send the token's own |
| `access_not_granted` | 403 | the token's entry does not allow the call | an agent that may grant allows more, on the owner's word |
| `token_switch_needs_key` | 409 | turning the switch on, holding no token | issue yourself one, sign in, then change it |
| `token_issue_not_allowed` | 403 | a token would be handed out other than locally, by the owner or by a signed-in granting manager | ask in the AI program's own conversation, or the owner issues one |
| `token_copy_not_allowed` | 403 | a file of a protected collection named by a call about another collection, without the owner or a manager's key of that collection | work on it in its own collection, or ask a manager of it |
| `path_not_local` | 403 | a file or folder on another computer named while a collection here asks for keys | name it on this computer, or the owner takes it in |
| `token_settings_unreadable`, `access_settings_unreadable` | 503 | the collection's settings file cannot be read (a call refused; a change never written over) | try again; the owner repairs the file |
| `token_file_unreadable` | (MCP) | `sign_in`'s `token_file` is not the full path of a local `.token` file (by its shape, before anything is opened) | send its full path |
| `sign_in_no_collection` | (MCP) | `sign_in` on a connection with no collection | open or join the collection first |
| `token_end_invalid`, `token_fingerprint_invalid`, `token_not_found`, `token_fingerprint_ambiguous`, `token_file_outside_project` | 4xx | the issue or revoke call's own arguments | as the code says |
| `sign_in_link_owner_only` | 403 | a sign-in link asked for by anyone but the system | the start script prints one |

A 401 carries `WWW-Authenticate: Bearer`.

## Nothing secret written down

A token is never written into the audit, logs, answers or error texts. The
audit records `token: {short, agent}` on sign-in, on a refusal, and on issue
and revoke (`surface: access_tokens` for refusals before a collection is
opened). `tests/system/test_tokens_never_written.py` runs the whole flow
(issue, sign in, refused calls, join, leave, revoke) in a temporary install
and searches every file written under it, every response body and every log
record for every token made, and finds none outside the key files themselves.

`tests/system/test_every_route_asks_for_a_token.py` walks every route the app
serves: each one that takes `workspace_id` refuses a call without a token when
the switch is on; the routes exempt on purpose are listed there, each with its
reason.

## The honest limit

**On one computer under one login, an agent that can read any file can read
another agent's key file.** Tokens stop mistakes, agents without file access,
and callers over HTTP from elsewhere. They do not stop a determined local
agent with a shell. Real separation is separate OS users or containers, one
per agent; that is named here, not built.

Likewise a join's token is handed out through the local MCP connection on the
person's yes: an agent on this computer that can start Kovium's
own server can do what that connection does (since fix round BB-S1,
only for an agent id the collection does not know yet, which joins reading
only).

**One file unlocks every collection** (fix round BB-S12). The install's own
secret, `config/system_token` in the checkout, makes a request Kovium's
own (`X-Kovium-System`), and that passes every
collection's check, whatever its switch: an agent that can read that file can
reach every collection on this computer. So does the owner's signed-in
browser: its session cookie passes every collection until the server
restarts. Both are kept on this computer only, and the system token is made
readable by its owner where the OS allows it, but they are the same limit as
the key files: tokens separate callers from elsewhere and honest agents, not a
local program that can read the owner's files.

**Windows.** File modes (`0600`) do nothing on Windows; the key files get an
owner-only access list from `icacls` instead (above). Other files in the
checkout, `config/system_token` included, keep their folder's access list.

## Every program, every system

One mechanism for Claude Code, Codex, OpenClaw, Cursor and plain HTTP, on any
OS: the key file is a plain file in the agent's own folder, read by the
server; the MCP launch lines and shared settings files stay token-free. With
tokens on, OpenClaw's shared `openclaw` id stops mattering (each agent signs
in with its own token), and its per-server agent list (`codex.agents`) is only
a way to show fewer tools to the other agents, never a boundary.
