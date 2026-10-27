---
type: "documentation"
title: "Naming: Kovium"
description: "\"second brain\" (second_brain, second-brain, SECOND_BRAIN) and \"knowledge brain system\", with its \"kb-instance\". The rename is carried out by Track BG of docs/knowledge_layer_improvement_plan_v1.md; this page is the rule that track applies..."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing Kovium"
doc_path: "docs/naming_v1.md"
doc_sha256: "3be28baa07bc4b474e2a26929db332a8ffeedc5e51fb9f29b1e1d085f2d553c5"
---

# Naming: Kovium

**Decided by the owner, 2026-10-27.** The system is called **Kovium**. It replaces every earlier name:
"second brain" (`second_brain`, `second-brain`, `SECOND_BRAIN_`) and "knowledge brain system", with its
"kb-instance". The rename is carried out by Track BG of `docs/knowledge_layer_improvement_plan_v1.md`; this
page is the rule that track applies and that every later change follows.

**Why the name.** Knowledge as a *medium*: something knowledge lives in, moves through and grows in, not
a box it is stored in. That is also why "second brain" was dropped earlier: the system makes knowledge, it
does not only keep it.

## The terms

| term | short form | means | in code today |
| --- | --- | --- | --- |
| **Kovium** | | the system: this platform, every part of it | the package `second_brain` (see "What keeps its old name") |
| **Kovium instance** | **km-instance** | one instance made in Kovium, such as a PhD collection's | a workspace: `workspace_id`, `sandboxes/<id>/` or a registered `root_path` |
| **Kovium collection** | **collection** | all the knowledge inside one km-instance: raw files, converted and templated notes, wiki notes, concept pages, canvases, the graph and the vectors | the workspace's folders and store |
| **brain keeper agent** | | the agent inside Kovium that runs its jobs | `brain_keeper` (kept, owner's decision) |
| **nightly tasks** | | every task run to maintain a km-instance, or Kovium as a whole | |
| **caller agent** | | an external agent that talks to Kovium over MCP or HTTP | `caller_agent` |
| **user** | | the human behind the caller agent | |

`km` reads "knowledge medium". The short forms are for when the long one has already been said nearby: write
"Kovium collection" or "Kovium instance" the first time in a page or reply, then "collection" or
"km-instance".

## How to spell it, by place

| where | spelling | example |
| --- | --- | --- |
| prose, titles, user-facing texts, replies | `Kovium` (capital K, no article: "Kovium answers", not "the Kovium") | "Ask Kovium how anything works." |
| the instance, in prose | `Kovium instance`, then `km-instance` (lower case, hyphen) | "Each km-instance has its own folder." |
| MCP server name, plugin, slash command, URI scheme, file and folder names, CLI names | `kovium` | `/kovium:start`, `kovium://capabilities`, `.claude/agents/kovium-<collection>.md` |
| where a hyphen is not allowed (Codex agent names, Python identifiers, TOML keys) | `kovium_` | `kovium_<collection>`, `[mcp_servers.kovium]` |
| environment variables | `KOVIUM_` | `KOVIUM_EMBEDDINGS_DISABLED` |
| markers written into other people's files (HTML comments, cron lines, managed blocks) | `kovium` | `<!-- kovium-figure -->`, `kovium-maintenance:` |
| files and headers the system owns | `kovium` | `data/staging/kovium.db`, `<project>/.kovium/<collection>.token`, `X-Kovium-System` |

Every one of these is written from `second_brain/naming.py`; code reads the name there instead of spelling it.

**In texts a person reads** (the catalogue, the refusal and warning texts, the welcome, the tour, `help`, the
user guide, the getting-started guide), say **Kovium** for the system and **collection** for theirs. The
person did not bring the word "instance", so "km-instance" stays out of those texts, the same as
"workspace" (`docs/plain_language_rules_v1.md`). Docs for whoever changes the code, the journal and replies to
the owner use all the terms above.

## Never

- "second brain", "second-brain", "Second Brain", "knowledge brain", "knowledge brain system",
  "kb-instance", "kb-collection", "kbrain": not in prose, not in a new identifier, not in a new comment.
- "Kovium system", "the Kovium platform", "KM", "Kovium workspace", "instance" alone in a user text: one
  name, used straight.
- A new identifier with the old name. The code holds no old spelling at all; the names it uses live once, in
  `second_brain/naming.py` (Track BG1).

## What keeps its old name, and why

Settled by the owner, 2026-10-27: no compatibility period. The new code writes and reads only the new names.

- **History.** `IMPLEMENTATION_JOURNEY.md`'s entries, finished plans and dated results under `docs/`, past
  tracks of the plan, and git history are records of what was true when written. They keep the old words,
  with one line at the top of each historical page saying the system was then called second brain. Rewriting
  them would make the record lie.
- **What an install made before Kovium holds** (program connections, agent files, markers inside notes,
  environment variables, scheduled jobs): moved once, by hand, with `docs/upgrading_to_kovium_v1.md`.
- **Until the package track:** the Python package and distribution (`second_brain`, `second-brain` in
  `pyproject.toml`) and the repository's name. They become Kovium in the new public repository.
- **Kept on purpose:** the **brain keeper agent** (it names a role, not the system; `brain_keeper` in code),
  and workspace ids an instance already has (`hello_brain`, an instance's own id), which are its data.

## Enforced by

- `tests/system/test_one_name.py` (Track BG0): every tracked file is scanned for the old names, also when one
  is broken over two lines. A hit outside the history set (read from `docs/README.md`), the few files whose
  job is to name the old names, and the package's import path fails, with the file and the line. Every page
  of the history set carries the line saying it was written under the old name.
- The plain-words checker (`second_brain/plain_words.py`) refuses "second brain", "knowledge brain",
  "kb-instance" and "km-instance" in any text a person reads.
- `tests/contracts/test_upgrade_guide.py`: every new name the upgrade guide gives is the one the code writes.
