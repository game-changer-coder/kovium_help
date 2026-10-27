---
type: "documentation"
title: "The view in Obsidian (Track BE)"
description: "A km-instance's folder can be opened in Obsidian as a vault. Without help, Obsidian shows every file the platform keeps there, and most of those files are housekeeping:"
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing Kovium"
doc_path: "docs/obsidian_view_v1.md"
doc_sha256: "c5dc0163f4146dbb84437d41c425b4a90f39c69ce0b34dbcab7208d7b98c8039"
---

# The view in Obsidian (Track BE)

**Status:** built 2026-10-27. The code is `second_brain/obsidian_view/service.py`.

A km-instance's folder can be opened in Obsidian as a vault. Without help, Obsidian shows every file the
platform keeps there, and most of those files are housekeeping:
- earlier versions of concept pages and rollback snapshots, many with timestamp names;
- converter outputs and templated drafts.

These files hold links too, so the graph repeats every concept several times. This contract covers the few
Obsidian settings the platform writes so that the collection is what a user sees.

Opening the folder in Obsidian stays the user's act. The platform writes settings only. It never starts
Obsidian, and it never writes `workspace.json`.

## The layers

The view is built from four layers. A later layer beats an earlier one:

| layer | where | who writes it |
|---|---|---|
| the template | the install's `config/obsidian_view.json` | the platform, shipped |
| contributions | the km-instance's `config/obsidian_view.d/*.json`, applied in file-name order | an add-on switched on for the km-instance: it drops a file there and removes it when switched off |
| the owner's word | the km-instance's own `config/obsidian_view.json` | the owner (`change_setting` writes `enabled`) |
| colours chosen for kinds nobody named | the km-instance's `config/obsidian_view_state.json` | this module, so a colour never moves |

The add-on mechanism itself is not built yet. The contribution folder is the hook that track will use.

### Keys

Every layer may hold the following keys:
- `colors`: the colour of a kind of note, `#RRGGBB`, by the note's `type:`. A value of `null` takes the
  colour away, so that kind gets no group.
- `exclude`: folders to hide besides the housekeeping, each relative to the km-instance and written with
  forward slashes. An absolute path, or one that leaves the km-instance (`..`), is refused.
- `filter`: extra search terms for the graph's filter, such as `-tag:#draft`.
- `graph`: display values written into `graph.json`. Only these keys are allowed: `showArrow`,
  `hideUnresolved`, `showOrphans`, `showTags`, `showAttachments`, `textFadeMultiplier`,
  `nodeSizeMultiplier`, `lineSizeMultiplier`, `centerStrength`, `repelStrength`, `linkStrength`,
  `linkDistance`. Zoom (`scale`), whether the panel is open (`close`) and the collapsed sections belong to
  the user and are never written.

Two keys are reserved. `enabled` belongs to the template and the owner. `palette` belongs to the template
alone.

The template ships with these values:
- `enabled`: true.
- `graph`:
  - `showArrow`: true;
  - `hideUnresolved`: true, Obsidian's "Existing files only". A link to a note that does not exist yet is
    not drawn as an empty node.
  - `showOrphans`: true;
  - `showTags`: false;
  - `showAttachments`: false.
- `filter` and `exclude`: empty.
- `colors`: a colour for each kind the platform itself writes or declares:
  - the platform's own kinds: `concept`, `idea`, `research_direction`, `owner_work`;
  - the note types: `paper`, `white-paper`, `book-chapter`, `reference-list`, `documentation`,
    `meeting-note`, `decision-record`, `webpage`, `blog-post`, `ai-conversation`, `chat-transcript`,
    `voice-note`, `youtube-video`, `image`.
- `palette`: twelve further colours for kinds nobody named.

A test fails when a kind the platform writes, or a built-in note type, has no colour in the template. A new
format template therefore arrives with its colour.

A contribution cannot show the housekeeping again. Hidden folders only accumulate, and a key a layer may not
hold is reported, never applied. A contribution file that cannot be read, or that names an unknown key or an
invalid value, is reported in `check_setup` and by `scripts/obsidian_view.py`. Everything else in it still
applies.

## What is hidden, and what the graph keeps

The hidden folders are read from the layout the workspace registry makes (`second_brain/workspaces/layout.py`,
`HOUSEKEEPING_KEYS`). They are never typed into the template, so a layout change needs no second edit. Today
they are:
- `sources/converted_markdown/` and `sources/templated_markdown/`;
- `data/`, `config/`, `maintenance/`, `logs/` and `brain_keeper/`;
- `agent_memory/` (conversations);
- `derived/` (what a collection derives at release).

These go into Obsidian's "Excluded files" (`app.json`, `userIgnoreFilters`), together with the layers'
`exclude`. The originals (`sources/raw/`) and the notes (`sources/wiki/`) are the collection, and they are
never hidden.

The graph's filter (`graph.json`, `search`) is built in three parts:
1. `path:"sources/wiki"` keeps the collection's notes;
2. `-[type:concept_redirect]` leaves out redirect pages;
3. the layers' `filter` terms follow.

## Colours

A note is coloured by its `type:`, never by its folder. Every source kind shares one folder, and a km-instance
for any other use has kinds nobody listed.

The graph gets one colour group per kind found in the notes: `[type:<kind>]`. A kind that holds more than
letters, digits, `_` and `-` is quoted, as in `[type:"field note"]`. Kinds with no note get no group. A kind
found with no named colour takes the first palette colour nobody uses yet. Once every colour is used, the
palette repeats in turn. The choice is recorded under `assigned` in `config/obsidian_view_state.json`, so a
kind keeps its colour as others appear.

## Writing into Obsidian's files: merge, never overwrite

Only the managed keys are written:
- in `graph.json`: `search`, `colorGroups` and the `graph` values;
- in `app.json`: `userIgnoreFilters`.

Every other key is left as it was, in both files. That includes zoom and the user's own settings.
`workspace.json` and the rest of `.obsidian/` are never touched.

What belongs to the user is kept:
- **Colour groups and hidden folders the user made.** Colour groups are kept, placed before the platform's,
  since Obsidian colours a note by the first group it matches. Hidden folders are kept in their place.
- **A change made in Obsidian after the platform wrote a value.** The vault's record in
  `config/obsidian_view_state.json` (`vaults`) holds what was written last. When the file on disk differs
  from that record, the user changed it, and their value is kept from then on and reported
  (`kept_user_changes`). This covers a display value, the filter, one kind's colour, and a hidden folder the
  user showed again.

Writes are made with care:
- **Only when the content changes**, so a second run writes nothing, and a folder synced between computers
  (Syncthing, for example) sees as few changes as possible.
- **By atomic replace**: a temporary file, then a rename. The JSON is written as Obsidian writes it, with two
  spaces and no newline at the end.
- **Never over a file that is not valid JSON.** Obsidian may be half way through writing it. The file is
  reported and left alone.

### Which vaults

- **The km-instance's top.** Its `.obsidian/` is made, with only these two files, when absent.
- **Any `.obsidian/` already inside the km-instance whose vault holds the notes or sits among them** (for
  example `sources/.obsidian/`). Every path is rebased to where that vault starts: `path:"wiki"`,
  `converted_markdown/`, and so on.
- **A vault anywhere else is left alone.** A vault beside the notes is reported (`left_alone`). The
  housekeeping folders are not even walked into. The search goes down at most four folders.

Folder intake never reads an editor's settings. Hidden files, and every file inside a hidden folder, are left
out of `plan_folder`, as `POST /sources/add` already did.

## When it runs

Every run is free: files only, no model call.

- **When a collection is made** (`POST /collections`, `create_collection`). It never makes the making fail.
- **When the welcome example is opened** (`POST /welcome/example`), in the working copy. The shipped
  `sandboxes/hello_brain` is never written.
- **In the free part of the nightly tasks.** A night writes something only when the notes brought a new kind,
  or a layer changed. The step is skipped silently while the view is switched off. A dry run of the night
  writes nothing.
- **Through `scripts/obsidian_view.py`**, for km-instances that already exist:
  `apply --workspace <id>` (or `--workspace-root <folder>`) `[--dry-run] [--json] [--force]`. The dry run
  prints what would change. `--force` writes while the owner has the view switched off.
- **When the owner switches it on again** with `change_setting` (`obsidian_view`, "A tidy view in Obsidian").
  The view is written at once.

Off (`obsidian_view.enabled: false` in the km-instance's own file) means nothing is written again. What was
written stays, and Obsidian keeps showing it until the user changes it there.

`check_setup` lists the view as on or off, in the catalogue's words. When it is on, it counts the extra
settings that could not be used.

## What Obsidian does with it (BE0)

This is the documented behaviour. It was not checked in a running Obsidian during the build.

- **Colour groups and the graph filter take Obsidian's search syntax**, and the property form `[type:paper]`
  is part of that syntax. A look in a running Obsidian is still owed (see the journal's entry for this
  track).
- **"Excluded files" (`userIgnoreFilters`) leave search results and the graph**, and are ranked lower in the
  quick switcher and link suggestions. **They stay listed in the file explorer.** Core Obsidian has no setting
  that hides a folder there. The folders hidden here therefore still appear in the file list; they no longer
  appear in search or in the graph. Older reports say the local graph may still show excluded files.
- **Obsidian keeps keys it did not write**, and fills in its own defaults for keys a file lacks.
- **While Obsidian is open it holds its settings in memory**, and it may write them back over a change made
  on disk. Close Obsidian, or reload it, after `apply`. A write-back that happens anyway is read as the user's
  own change and kept, so it is not undone by the next night.

**The fallback**, if property queries fail in some Obsidian version: colour by folder. That loses one thing:
every source kind would share one colour, since they share a folder.

## Limitations

- The file explorer still lists the housekeeping folders (see BE0).
- A value the user set by hand to exactly what the platform would write cannot be told from the platform's
  own. If the platform later stops managing it, it is removed.
- Changing a colour from the conversation is not offered yet. The owner writes it in the km-instance's
  `config/obsidian_view.json`, or changes it in Obsidian, where the change is kept.
