---
type: "documentation"
title: "Kovium Intake (Claude Code)"
description: "Add documents of any kind to a Kovium instance, from Claude Code: files, folders and links (documents, PDFs, books, images, Word files, web pages, videos, meeting notes, records) in one request. Use when the user asks to add or take in documents: ask the system to add them, pass on its one question and its report word for word, and leave every other decision to the system. Also registers the owner's own notes instead of processing them."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "skill"
audience: "the AI program that works with Kovium"
doc_path: "skills/claude-code/kovium-intake/SKILL.md"
doc_sha256: "deb889d5b0d52d84b99102c3e9ecc13ac7e9d2ca1f986e7d2c4fe11090264b8f"
---

# Kovium Intake (Claude Code)

## The three rules

1. **Ask the system to add the files.** One request, `POST /sources/add`,
   with the files, folders and links the user named. No triage, copying,
   renaming or scripts of your own: the system does all of it.
2. **Pass on its one question and its report, word for word.** The plan's
   `question` (asked only when the user's token budget does not cover the
   work) and the run's `report.lines`.
3. **Never ask the user anything the system did not ask.** Kinds,
   duplicates, versions, books, retries, readers and facets are settled
   between you and the system: follow each `notices[].fix.next` whose
   `settled_by` is `caller`, leave those of `system` alone, and tell the user
   only a notice's `user_text`.

The prompt a user gives you is one line ("Add the files in this folder to my
collection"); nothing more is needed. Load `kovium-operations` for the
coordinates. Every call carries `agent_id` and `workspace_id`; the base URL is
the operator's (`http://127.0.0.1:8000` by default). A `curl` with a full
`http://` URL and a JSON body needs nothing special on any shell: send it as
written, with nothing in front of it, so it still matches a permission such as
`Bash(curl:*)`.

**Your file tools work on this repository.** Reading or copying a file off disk
looks like adding it, and it is not: the instance records what it did with each
file, and a file you handle yourself is outside that record.

## The calls

**1. The plan** (free, no agent call):

```bash
curl -s -X POST http://127.0.0.1:8000/sources/add -H 'Content-Type: application/json' \
  -d '{"agent_id":"<agent_id>","workspace_id":"<workspace_id>","paths":["<folder or file>"],"urls":[]}'
```

Paths are relative to the workspace, or `root:path` for a declared source
root; a folder is read at any depth. Then, by what the plan says:

| the plan has | you do |
|---|---|
| `question` null and `next` | make the call in `next` (it carries the `approval`) |
| a `question` | ask `question.text` as written, recommended answer first; on yes make the calls in `question.on_yes`, in order; on no those in `question.on_no`. No answer: nothing runs |
| `ready: false`, `adding_kind_unknown` | send `kind`, the kind most of the files are (`GET /format-templates` lists them), and plan again: never a question |
| `adding_nothing_to_add` | relay its `summary`; nothing to run |

**2. The run** returns a `job_id`. Wait on it, always with `wait`:

```bash
curl -s "http://127.0.0.1:8000/brain_keeper/jobs/<job_id>?wait=60&agent_id=<agent_id>&workspace_id=<workspace_id>"
```

`settled: false` with `job_still_running`: the same call again. Never resubmit
a running job. A refusal of the run (`adding_plan_changed`,
`adding_approval_expired`) means plan again and run the new plan.

**3. The report**: `result.report.lines`, at most six plain lines. Relay them
as they are. If the run stopped at the budget, `result.question` is the one
question to go on, answered the same way. A line about concepts waiting for the
assistant is news, not a task: the system finishes them.

## What the system does by itself

Triage (copies, files already in, scans with no text, slides, unreadable
formats), each file's kind, oldest first, books split into their parts (by their
bookmarks, contents page, headings, or the brain keeper agent within the budget;
read whole when none is clear; a split the user states goes as `chapters`), another
version of a work already in kept and linked to it (send `versions: skip` or `separate` only if the user asked),
every failure tried again (once as it was, once with the next PDF reader) and
a partial note made again before anything is reported, one concept pass,
search entries caught up.
Facets and their vocabulary are the system's alone. The detail of every
document is in the job's `result.added`, for anyone who asks.

## The owner's own notes are not sources

What the owner wrote themselves is registered with `POST /owner-work`, never
added: processed, its statements would be cited as findings. Three kinds:
`notes` (a topic or something that happened), `design` (their own method or
draft) and `commentary` (their notes on someone else's work). Registering
calls no agent and never changes the file. Details: `reference/owner-notes.md`.

## Connected over MCP

The same three rules, over MCP, for an agent the owner lets manage the
collection (`set_access`): `plan_adding` with `paths` as the user named them;
its questions one at a time, recommended answer first; `start_adding` with
the plan's `approval`; then `adding_status`, waiting, and relay its report.
A reader sees new notes with `delta` and the collection with `overview`;
`request_tools` says what else exists. With `allow_proposals`, `remember`
only proposes.

## Only when the operator asks for a lower-level step

| file | when |
|---|---|
| `reference/lower-level-intake.md` | one step at a time: what the instance can read, a folder's triage and run, one source, videos, re-reading, checking each note |
| `reference/books.md` | a book: how its parts are found, the user's own split, time and the page ceiling, chapters split by hand, its figures and equations |
| `reference/converters-and-vision.md` | choosing a PDF reader; Word and web pages; images; reading figures and equations; re-reading a source |
| `reference/facets-and-quality.md` | how the system keeps facets; measuring them against hand labels |
| `reference/owner-notes.md` | registering the owner's notes, designs and commentary; notes written by hand |

## Hard rules

1. **The plan first**; a real run only with its `approval`.
2. **Never resubmit a running job.** Wait on it with `wait`.
3. **Never drive adding by hand**: no script, no file written, copied or
   renamed, no lower-level route unless the operator asks for that step.
4. **Spending**: within the user's standing budget the plan is the consent;
   beyond it, its `question`. Switching a setting on (reading pictures,
   tails) is the owner's.
5. **Always send `agent_id` and `workspace_id`.**
6. **Read notes through the API**, not with Grep or Read on `sources/wiki/`.
7. **The owner's own notes are registered, never processed.**
