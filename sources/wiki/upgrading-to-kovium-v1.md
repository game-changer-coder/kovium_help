---
type: "documentation"
title: "Upgrading a collection made before Kovium"
description: "before anything is removed. Kovium's own code does none of this: there is no compatibility period (owner's decision, 2026-10-27), so after the update the new code neither reads nor cleans up the old names."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing Kovium"
doc_path: "docs/upgrading_to_kovium_v1.md"
doc_sha256: "54c1605637fc14b7bf77944f35566d47fe830fae54986a22f08b0a4ade09e34c"
---

# Upgrading a collection made before Kovium

**For:** an install and its collections made while the system was called second brain (before Track BG).
**Done by:** a separate agent, on the owner's computer, by hand, one step at a time, with the owner's yes
before anything is removed. Kovium's own code does none of this: there is no compatibility period
(owner's decision, 2026-10-27), so after the update the new code neither reads nor cleans up the old names.

**Status:** Track BG is built (`docs/knowledge_layer_improvement_plan_v1.md`); the new names below are the
ones the code writes (`second_brain/naming.py`, checked by `tests/contracts/test_upgrade_guide.py`). Do it
once, after updating to the Kovium version and before using the old collections with it: until then the new
code does not find their store, their keys or their connections.

This page names the old spellings on purpose, because it has to find them. It is the one current page allowed
to (`tests/system/test_one_name.py`).

## Rules for the agent doing it

- **Work on a backup first.** Before step 1, back up every collection
  (`python scripts/ops_backup.py --workspace-root <collection folder> --label before-kovium`, which archives
  its state; also copy the collection's folder itself if it lives outside the checkout), and copy each program
  config file named below before editing it.
- **Replace only what this guide names, spelled exactly.** Never a blanket search-and-replace over a
  collection: the person's own notes may mention "second brain" as a subject, and those words are theirs.
- **Remove only what the old install wrote.** An entry, file or line is the old install's when it carries the
  old marker or its command points at this checkout. Anything else that looks similar is reported, not touched.
- **Record what was done**, per step: what was found, what was changed, what was left and why.
- **Stop and ask** when a step finds something this guide does not describe.

## 0. Before: what is there

Make an inventory, changing nothing:

```bash
# the checkout: which version is installed
git -C <checkout> log -1 --oneline
# every collection the registry knows, with its folder
cat <checkout>/config/workspace_registry.json
# old names in the places outside the collections
grep -n -i -E 'second[-_ ]?brain' ~/.claude.json ~/.codex/config.toml ~/.cursor/mcp.json 2>/dev/null
openclaw mcp list 2>/dev/null | grep -i -E 'second[-_ ]?brain'
claude plugin list 2>/dev/null | grep -i -E 'second[-_ ]?brain'
crontab -l 2>/dev/null | grep -i -E 'second[-_ ]?brain'
env | grep '^SECOND_BRAIN_'
grep -n 'SECOND_BRAIN_' ~/.bashrc ~/.profile ~/.zshrc 2>/dev/null
```

On Windows also: `schtasks /Query /FO LIST | findstr /I second-brain-maintenance-`.

And for each project folder an agent works in (each place a caller agent was set up or a collection joined):

```bash
ls <project>/.claude/agents/second-brain-*.md <project>/.codex/agents/second_brain_*.toml 2>/dev/null
grep -n -E 'second-brain caller agent' <project>/AGENTS.md <project>/CLAUDE.md 2>/dev/null
grep -n -i -E 'second[-_ ]?brain' <project>/.mcp.json <project>/.codex/config.toml <project>/.cursor/mcp.json 2>/dev/null
```

## 1. Update the checkout

Pull the Kovium version, then reinstall it into its environment, so the server's program is installed under
its new name (`kovium-mcp`; the old `second-brain-mcp` is removed by the reinstall):

```bash
<checkout>/.venv/bin/python -m pip install -e <checkout>
```

Then run the setup once, which writes the **new** connections:

```bash
python scripts/setup.py
```

After this, each program has a `kovium` connection beside the old `second-brain` one. The next steps remove
the old ones.

## 2. The program connections and the plugin

| program | old (remove) | new (made by step 1) |
| --- | --- | --- |
| Claude Code, plugin | plugin `second-brain` from marketplace `second-brain`; tools `mcp__plugin_second-brain_second-brain__*`; command `/second-brain:start` | plugin and marketplace `kovium`; tools `mcp__plugin_kovium_kovium__*`; `/kovium:start` |
| Claude Code, plain connection | `second-brain` at user scope (`claude mcp remove second-brain -s user`) and, per project, `second-brain-<collection>` in `.mcp.json` or local scope | `kovium`, `kovium-<collection>` |
| Codex | `[mcp_servers.second-brain]` (and `second-brain-<collection>`) in `~/.codex/config.toml` and each project's `.codex/config.toml` | `[mcp_servers.kovium]`, `kovium-<collection>` |
| OpenClaw | `openclaw mcp` entries `second-brain`, `second-brain-<collection>` (`openclaw mcp unset <name>`, then `openclaw mcp reload`); stop its gateway first if the setup asked to | `kovium`, `kovium-<collection>` |
| Cursor | `second-brain` in `~/.cursor/mcp.json` and a project's `.cursor/mcp.json` | `kovium` |

Remove an entry only when its command starts this checkout's server. For a joined collection, make the new
connection by asking the agent to join again ("connect yourself to my … collection") before removing the old
one, so the agent's access (and its access key, if keys are on) is carried over; check its access afterwards
with `check_setup`.

## 3. The skills

- Claude Code: the skills come with the plugin; step 2 replaced them.
- OpenClaw: in the default agent's workspace, `skills/second-brain-intake`, `-onboarding`, `-operations`,
  `-research-answering`. Step 1 installed `kovium-*` beside them; remove the four old folders.
- The `second-brain-operator` skill, if installed anywhere outside the checkout (it is not part of the
  checkout): rename its folder to `kovium-operator`, set `name: kovium-operator` in its `SKILL.md`, and
  change the old names in its text the way this guide does.

## 4. The caller agents

For each project folder from step 0:

1. Ask the agent there for the agent again ("make me a Kovium agent for my … collection"). This writes
   `.claude/agents/kovium-<collection>.md`, Codex `.codex/agents/kovium_<collection>.toml`, or, for OpenClaw,
   a new marked section in `AGENTS.md`, with the style chosen before.
2. Then remove the old one: the file `second-brain-<collection>.md` or `second_brain_<collection>.toml`
   whose first lines carry `second-brain caller agent`, or the section of `AGENTS.md` or `CLAUDE.md` between
   the `<!-- second-brain caller agent … begins here …-->` line and its end line. Nothing else in the file
   changes.
3. Codex calls the agent by its new name, `kovium_<collection>`. Tell the owner the new name.

## 5. Environment variables

Rename each `SECOND_BRAIN_*` the owner set (shell profile, a service unit, a scheduled task's environment) to
the same name with `KOVIUM_`: for example `SECOND_BRAIN_EMBEDDINGS_DISABLED` becomes
`KOVIUM_EMBEDDINGS_DISABLED`. The old names have no effect any more.

## 6. Scheduled nightly tasks

- cron: remove the lines under the old header (`# second_brain per-workspace maintenance`) and those
  carrying `second-brain-maintenance:`. Then let Kovium write its own again:
  `python scripts/maintenance_cron.py sync`, which writes the lines under the new header with
  `kovium-maintenance:` markers.
- Windows: delete the tasks named `second-brain-maintenance-<id>` (`schtasks /Delete /F /TN <name>`);
  Kovium makes `kovium-maintenance-<id>`.
- Check the next nightly run happened, in the collection's log, the day after.

## 7. Each collection's store, keys and manual

For every km-instance in the registry (`config/workspace_registry.json`), with nothing running against it
(no program connected, no web page open, no nightly task in progress):

- **The store.** `data/staging/second_brain.db` becomes `data/staging/kovium.db`, with its `-wal` and `-shm`
  files if present. Then in `config/workspace_registry.json`, change that entry's `paths.db_path` to the new
  file. Kovium makes and looks for `kovium.db` only; a collection left with the old file opens empty.
  The checkout's own shared store, `<checkout>/data/staging/second_brain.db` (the older unscoped mode, if it
  exists), is renamed the same way; until then `scripts/ops_status.py` reports `integrity_check_failed`,
  because it looks for `kovium.db`.
- **Access keys** (only if access keys were turned on): in each project folder an agent works in,
  `.second-brain/<collection>.token` moves to `.kovium/<collection>.token`, and a `.gitignore` line naming
  `.second-brain/` changes to `.kovium/`.
- **The web page's sign-in.** The cookie is now `kovium_owner`: sign in again once with a new link
  (`python scripts/ui_sign_in_link.py`). The header the system's own processes send is now
  `X-Kovium-System`; a script of your own that sends the old one is refused until changed.
- **The help manual**, if it was fetched: the km-instance `kbrain_help` is now `kovium_help`. Remove the old
  folder and its registry entry, and fetch the manual again from the tour or from `help`. Until the manual is
  republished under the new name, its pages still say second brain.

## 8. Inside each collection's notes

Only these exact strings, only in the collection's own files (its notes, converted Markdown and wiki):

| old (exact) | new (exact) | where |
| --- | --- | --- |
| `<!-- Yours. Second brain never rewrites this section. -->` | `<!-- Yours. Kovium never rewrites this section. -->` | the owner's section of every note |
| `<!-- second-brain:owner-section -->` | `<!-- kovium:owner-section -->` | notes, if any hold it |
| `<!-- second-brain-figure -->` | `<!-- kovium-figure -->` | converted Markdown |

Change only that one line; never anything else in the person's section. Count the lines before and after:
each count of old must equal the count of new.

Then:
- the collection's own `config/` (`sys_config.json`, `note_types.json`, `facet_vocabulary.json`) may hold
  "second brain" in its explanatory `_note` fields: change those words; leave every value.
- let the collection notice the changed files the ordinary way: the next nightly run, or ask the caller agent
  to refresh the changed notes (`POST /maintenance/refresh` with their paths, free). Its store then matches
  its files again.
  Nothing is drafted again, and nothing costs tokens.
- the workspace id stays as it is (`phd_kbrain` and the like are the instance's data, `docs/naming_v1.md`).

## 9. Check nothing old is left

```bash
grep -r -n -i -E 'second[-_ ]?brain' <collection folder>       # only the person's own words should remain
grep -n -i -E 'second[-_ ]?brain' ~/.claude.json ~/.codex/config.toml ~/.cursor/mcp.json
openclaw mcp list | grep -i -E 'second[-_ ]?brain'
crontab -l | grep -i -E 'second[-_ ]?brain'
env | grep '^SECOND_BRAIN_'
```

Each should return nothing, or only text the person wrote. Then, in a **new** conversation in each program
(connections load only at the start), ask the collection one question and check the answer names its source,
and ask the agent to check the setup: it should name the `kovium` connection and the same access as before.

## 10. Report

One short report for the owner: per collection and per program what was moved, what was left and why, the
backup's place, and the agent's new name where it changed. Then remove the backups only on the owner's yes.
