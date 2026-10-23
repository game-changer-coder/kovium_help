---
type: "documentation"
title: "Your first instance, end to end"
description: "A step-by-step guide for somebody who has never set one of these up. It goes from an empty folder to a working instance that has answered a real question from a real document, and back again through a restore rehearsal."
status: "stable"
reading_status: "read"
product: "second brain"
product_version: "0.1.0"
doc_kind: "guide"
audience: "a person setting up or using second brain"
doc_path: "docs/first_instance_lifecycle_guide_v1.md"
doc_sha256: "db38361d390ab4e05982d40f613c81db4aa38e8350e65c07e7f9ef9577f32ccc"
---

# Your first instance, end to end

A step-by-step guide for somebody who has never set one of these up. It goes
from an empty folder to a working instance that has answered a real question
from a real document, and back again through a restore rehearsal.

Every command here is one you can paste. Placeholders look like
`<workspace_id>`.

---

## The one idea to get straight first

You will be working with **two different folders**, and mixing them up is the
most common way this goes wrong.

| folder | what it is | what lives there |
|---|---|---|
| **the checkout** | the platform itself | the service, and by default your notes |
| **the agent workspace** | your own project folder | `.claude/skills/` and your project files |

Your knowledge does **not** live in your agent's project folder. By default it
lives inside the checkout under `sandboxes/<workspace_id>/`. You can put it
somewhere else entirely — see Step 4 — but it is never your agent's folder.

Your agent talks to the platform over `http://127.0.0.1:8000`. Not by opening
files.

---

## Step 0 — Check you have what you need

```bash
python3 --version         # 3.11+
curl --version
.venv/bin/python scripts/brain_keeper_check.py    # after Step 1 has created .venv
```

The brain keeper agent — the agent inside the brain that does the judgement
work (extracting claims, shaping notes, assigning facets) — runs on an agent
CLI you already pay for by subscription: **Claude Code** (tried first),
**Codex** (signed in with ChatGPT) or **OpenClaw**. One is enough; more give it
somewhere to go when one runs out of quota. The check lists each in the order
jobs try them, says whether it is installed and signed in with a subscription,
and gives the exact command that fixes it. It never installs anything or signs
in for you, and a backend signed in only with a metered API key is refused.

If no backend is ready, stop. Without one the platform falls back to a
"deterministic" mode that still produces notes — notes with no judgement behind
them, which look completely normal and are nearly worthless. Step 7 exists to
catch exactly that.

---

## Step 1 — Start the service

In a terminal, in the checkout, and leave it running:

```bash
cd /path/to/your/second_brain-checkout
./scripts/run_second_brain.sh
```

First run creates a virtualenv and installs; give it a few minutes. Then in a
**second** terminal:

```bash
curl -s http://127.0.0.1:8000/health
```

You want `{"status":"ok", ...}`. Browsable route docs: `http://127.0.0.1:8000/docs`.

---

## Step 2 — Set your coordinates once

Do this in the second terminal and **stay in it**; everything below reuses these.

```bash
cd /path/to/your/second_brain-checkout
export BRAIN="$PWD"
export API=http://127.0.0.1:8000
export AGENT_ID=lifecycle-agent
export WS=lifecycle_test
```

`AGENT_ID` is *who is asking*. `WS` is *which brain*.

> **Both go on every single call.** A call that omits `workspace_id` is refused
> with **422**. A call naming a workspace that is not registered is refused with
> **404**. Nothing is served against a default — so a typo fails loudly instead
> of quietly creating a second, empty brain that passes every health check and
> answers nothing.

---

## Step 3 — Give your agent's folder the skills

The platform ships its own operating instructions. Use them rather than
inventing a procedure.

```bash
mkdir -p ~/lifecycle-test && cd ~/lifecycle-test
mkdir -p .claude/skills
cp -r "$BRAIN/skills/claude-code/second-brain-onboarding" .claude/skills/
cp -r "$BRAIN/skills/claude-code/second-brain-operations" .claude/skills/
```

Record the coordinates where a later session will find them:

```bash
cat > CLAUDE.md <<'EOF'
# Second brain coordinates

- agent_id:     lifecycle-agent
- workspace_id: lifecycle_test
- API:          http://127.0.0.1:8000

Send BOTH ids on every call. A missing workspace_id is a 422; an unregistered
one is a 404. GET /ops/workspaces lists what exists.

Use the `second-brain-operations` skill before touching the brain at all.
Never answer from note files directly; use the API.
EOF
```

Start Claude in that folder and type `/skills`. Both should be listed.

*(Using OpenClaw instead? Copy from `skills/openclaw/` into
`<agent workspace>/skills/` and follow its registration step. Do not mix the two
sets.)*

---

## Step 4 — Register the workspace

A workspace is brought into existence **deliberately**. This is the only call
that creates one:

```bash
curl -s -X POST "$API/ops/workspaces" \
  -H 'Content-Type: application/json' \
  -d "{\"workspace_id\":\"$WS\",\"agent_id\":\"$AGENT_ID\"}"
```

Expect **201**. A **409** means it already exists — carry on.

### Putting your notes somewhere else

By default the collection lands at `sandboxes/<workspace_id>/` inside the
checkout. It does not have to. Give an **absolute** `root_path` and the whole
collection lives there instead — an existing vault, a synced folder, a bigger
disk:

```bash
curl -s -X POST "$API/ops/workspaces" \
  -H 'Content-Type: application/json' \
  -d "{\"workspace_id\":\"$WS\",\"agent_id\":\"$AGENT_ID\",\"root_path\":\"/absolute/path/to/notes\"}"
```

Two rules are enforced:

- the path must be **absolute** — a relative one is refused
- it may not sit **inside another workspace's** folder — nested collections
  corrupt each other, because each rebuild ingests the other's notes and prunes
  what it does not recognise

A note's identity is anchored to its workspace root, so a collection can be
moved wholesale and every note keeps the identity it already had.

### Seeing what exists

```bash
curl -s "$API/ops/workspaces?agent_id=$AGENT_ID&workspace_id=$WS"
```

To remove one from the list later: `DELETE /ops/workspaces/<id>?agent_id=<you>`.
That **deregisters only — it never deletes notes.**

---

## Step 5 — Choose the brain keeper agent's backend (usually: nothing to do)

The install's `config/brain_keeper.json` already says: Claude Code first, then
Codex, then OpenClaw, on the `balanced` model tier. Each job is one sealed,
one-off call — the job travels in the message, the agent has no tools and no
memory, and nothing is left running or registered afterwards. When a backend
runs out of quota, fails to sign in or does not serve the model, the same job
moves to the next one and that backend rests for a while (for every instance
on the machine). Nothing has to be set up per instance.

To change it for this instance only, write just the keys that differ:

```bash
cat > "$BRAIN/sandboxes/$WS/config/brain_keeper.json" <<'JSON'
{
  "backend": "codex",
  "fallback": ["claude_code"],
  "model_tier": "fast"
}
JSON
```

*(If you chose a custom `root_path`, write it to `<root_path>/config/brain_keeper.json`.)*

**Then restart the service** — Ctrl-C in the first terminal and run
`./scripts/run_second_brain.sh` again. The configuration is read **once**, when
the workspace is first used in a running process; a file written afterwards is
not picked up until a restart, and the symptom is silent.

- `backend`: `claude_code`, `codex`, `openclaw`, or `auto` (the first ready one).
- `model_tier`: `strong`, `balanced` or `fast` — capability without pinning a
  vendor model id. Claude Code and Codex run the tier's model; OpenClaw runs
  whatever model it is configured with (it decides).
- Continuing a conversation with the brain keeper agent (only
  `POST /query/gaps/interpret` with `auto`, or the operator-only
  `POST /brain_keeper/jobs`) works across a backend switch: the brain keeps the
  earlier turns and sends them with the next job. A follow-up is a *question*;
  an instruction written into a job's input is ignored by design.

---

## Step 6 — Confirm the layout

```bash
curl -s "$API/ops/status?agent_id=$AGENT_ID&workspace_id=$WS"
ls "$BRAIN/sandboxes/$WS"
```

You should see `sources/`, `data/`, `maintenance/`, `logs/`, `config/`.

- `sources/wiki/` — your notes. The real content.
- `data/` — **derived and rebuildable**. Never hand-edit it, never read it
  directly.

---

## Step 7 — The four verifications (do not skip)

Each catches a different silent failure.

**7a — Is the real thinking backend in use?**

```bash
curl -s "$API/brain_keeper/skills?agent_id=$AGENT_ID&workspace_id=$WS"
```

Require `resolved_backend: "claude_code"` — **not** `deterministic` — and
`backend_readiness.claude_code.available: true`. If unavailable, read
`backend_readiness.claude_code.reason`. Do not guess.

**7b — Can this instance bill anyone?**

```bash
curl -s "$API/ops/billing_safety?agent_id=$AGENT_ID&workspace_id=$WS"
```

Require `safe: true` and `billable_env_vars_present: []`. An exported
`ANTHROPIC_API_KEY` shows up here — unset it.

**7c — Is the instance internally healthy?**

```bash
curl -s "$API/ops/self_check?agent_id=$AGENT_ID&workspace_id=$WS"
```

**7d — One real note, end to end.** The only check that proves judgement
actually reaches the brain-keeper.

```bash
cat > "$BRAIN/sandboxes/$WS/sources/raw/first_note.md" <<'EOF'
# Weekly irrigation review

The north field was watered on Tuesday and Friday. Soil moisture held at
around 30 percent through the week, unless the afternoon wind picked up,
in which case it dropped closer to 22 percent by evening.

The new drip line on the east row has not been tested under full pressure.

Open question: does the wind effect justify moving the Friday watering to
early morning?
EOF
```

That deliberately contains a plain fact, a **qualifier** ("unless the wind
picked up") and an **open question**. You will check all three survived.

```bash
# 1. normalize: raw Markdown -> a wiki-form note
curl -s -X POST "$API/normalize/file" \
  -H 'Content-Type: application/json' \
  -d "{\"path\":\"sandboxes/$WS/sources/raw/first_note.md\",\"agent_id\":\"$AGENT_ID\",\"workspace_id\":\"$WS\"}"
```

Take `normalized_path` from the reply, then:

```bash
# 2. ingest the note into the graph
curl -s -X POST "$API/ingest/file" \
  -H 'Content-Type: application/json' \
  -d "{\"path\":\"<normalized_path from step 1>\",\"agent_id\":\"$AGENT_ID\",\"workspace_id\":\"$WS\"}"

# 3. ask about it
curl -s -X POST "$API/agent/knowledge/answer" \
  -H 'Content-Type: application/json' \
  -d "{\"agent_id\":\"$AGENT_ID\",\"workspace_id\":\"$WS\",\"query\":\"what happens to soil moisture when it is windy\"}"
```

---

## Step 8 — The acceptance table

Onboarding is complete only when **all** of these hold:

| check | required value |
|---|---|
| `resolved_backend` | `claude_code`, not `deterministic` |
| `billing_safety.safe` | `true`, no billable env vars |
| the note's `## Normalization Provenance` | `mode: brain_keeper` (from `/normalize/file`; a note from `/sources/process` names its template, e.g. `mode: academic_paper_template`, with `template_runtime_mode: brain_keeper`) |
| same section | `source_coverage: 100.00%` (on an imported note processed before 2026-10-09 there is no such line: check for no `template_runtime_part_failures`) |
| `/ingest/file` | non-zero `node_count` and `edge_count` |
| `/agent/knowledge/answer` | `knowledge_items` with `source_refs` carrying real line numbers |

Then **open the note and read it**:

```bash
cat "$BRAIN/sandboxes/$WS/sources/wiki/"*.md
```

This is the one place where reading a file directly is correct — you are
auditing the machinery, not consuming knowledge. Check the claim, the qualifier
and the open question are all present and kept apart. **If the qualifier was
dropped, extraction is not working properly even with every status light
green.** No status check can catch that; only reading can.

---

## Step 9 — Import a document you did not write

Steps 7d used the hand-authored route. For anything external, the entry point
does much more — convert, template, normalize, ingest:

```bash
curl -s -X POST "$API/sources/process" \
  -H 'Content-Type: application/json' \
  -d "{\"path\":\"sandboxes/$WS/sources/raw/<file>\",\"kind\":\"<kind>\",\"agent_id\":\"$AGENT_ID\",\"workspace_id\":\"$WS\",\"auto_ingest\":true}"
```

One of `kind` or `template_id` is **required** — the pipeline will not guess a
shape. See what exists: `curl -s "$API/format-templates"`.

**This backgrounds and returns a `job_id`.** Follow it with a *waiting* poll:

```bash
curl -s "$API/brain_keeper/jobs/<job_id>?wait=60&agent_id=$AGENT_ID&workspace_id=$WS"
```

**Always pass `wait`.** A busy polling loop costs far more than the work it
watches — measured on a real ingest, the job cost about 180,000 tokens and the
loop watching it about 1 million. One waiting call replaces roughly ten polls.

- `settled: true` → finished; read `status`
- `settled: false` with a `job_still_running` warning → the wait expired, call
  again with the same `job_id`. Normal, **not** a reason to resubmit

Never resubmit because something feels slow. A resubmit runs a second full
pipeline and can leave two runs writing the same note.

### Does it keep its own copy?

Controlled by `source_retention` in `config/sources.json`, decided **once per
workspace**:

| value | behaviour |
|---|---|
| `copy` *(default)* | a source from outside the workspace is copied into `sources/raw/` during intake |
| `reference` | the original is read where it lies and nothing is copied |

`copy` means the collection can be moved, backed up and restored as one thing
and survives the original being deleted. `reference` suits a large external
library you do not want duplicated — at the cost that refreshing a note and
judging its staleness both need that library present.

Either way, **retrieval is unaffected** — the note and the graph answer without
the original. If a source does go missing, `/maintenance/refresh` reports it as
`source_unavailable` instead of skipping it silently.

This is a workspace setting and not a per-call flag on purpose: a mixed
collection leaves some notes refreshable and others not, with nothing recording
which.

---

## Step 10 — Ask, then actually verify the answer

```bash
curl -s -X POST "$API/agent/knowledge/answer" \
  -H 'Content-Type: application/json' \
  -d "{\"agent_id\":\"$AGENT_ID\",\"workspace_id\":\"$WS\",\"query\":\"...\"}"
```

**The most important rule when consuming:**

> A retrieved node label is a **title, not a finding**.

The route returns *candidates*. To state something as fact, pull the underlying
text with `POST /query/resolve_sections`, or open the note at the `source_refs`
path and line range, and quote from there.

Every governed answer carries a `confidence` block:

| field | what it tells you |
|---|---|
| `overall_confidence` | the headline number |
| `retrieval_confidence` | how well the corpus matched |
| `provenance_strength` | how many source notes state the matches (one: 0.57, two: 0.78, three or more: 1.0) |
| `coverage_ratio` | how much of the question the corpus can speak to |
| `weak_context` | **true means say so in your answer** |

If `weak_context` is true or coverage is low, say the corpus does not cover it.
Do not fill the gap from your own knowledge and present it as the brain's
answer. "The corpus does not contain this" is a correct and useful answer.

---

## Step 11 — Close the loop: prove you can get it back

A lifecycle test is not finished until you have restored. This is the step
almost everyone skips.

```bash
curl -s -X POST "$API/ops/backup" \
  -H 'Content-Type: application/json' \
  -d "{\"agent_id\":\"$AGENT_ID\",\"workspace_id\":\"$WS\"}"

curl -s -X POST "$API/ops/restore/rehearsal" \
  -H 'Content-Type: application/json' \
  -d "{\"agent_id\":\"$AGENT_ID\",\"workspace_id\":\"$WS\"}"
```

The rehearsal restores into a scratch location and verifies it, without touching
your live data.

---

## When something looks wrong

| symptom | cause | fix |
|---|---|---|
| `422 workspace_id is required` | you left it off | send both ids on every call |
| `404 unknown workspace` | not registered, or a typo | `GET /ops/workspaces`, then Step 4 |
| `409` when registering | already registered | carry on |
| `mode: heuristic`, `fallback_reason: brain_keeper_unavailable` | no backend ready | run `scripts/brain_keeper_check.py`, or read `backends[].reason` and `backends[].fix` in `GET /brain_keeper/skills` |
| `fallback_reason: templated_wiki_shape_failed: …` with `templated_wiki_runtime_mode: skipped_draft_kept` | the final shaping pass failed; the reason and job id follow | nothing is lost — the note is the extraction draft, unpolished. Re-run `/sources/process` on it once the cause is fixed |
| `templated_wiki_runtime_mode: skipped_heuristic` | as above, on a draft older than the note contract | re-run `/sources/process`; the heuristic layout is a stopgap |
| `resolved_backend: deterministic` | usually `claude` not on PATH | fix PATH, restart the service |
| notes built by the fallback although config looks right | config written after first use | restart the service (Step 5) |
| `source_coverage` below 100% | part of the input never reached the model | do not accept the note; re-run, then report |
| notes look right but answers have no citations | ingested without normalizing, or hand-written into `sources/wiki/` | normalize first, then ingest |
| answers feel thin | — | go back to Step 7; it is the diagnostic, not just a first-run checklist |

---

## Two optional extras

**Facet vocabulary.** Retrieval improves once the instance knows its own
controlled vocabulary (`POST /maintenance/vocabulary`). A new instance has none
and works without it — this is a "later, once you have twenty notes" job, not a
setup step.

If you already keep a facet scheme as a Markdown table (facet, value,
definition, or one table per facet under a heading), convert it once rather
than retyping it:

```bash
# see what it would add, conflict with, or skip, and why
.venv/bin/python scripts/import_vocabulary.py --workspace-root <workspace root> --markdown scheme.md
# then write it
.venv/bin/python scripts/import_vocabulary.py --workspace-root <workspace root> --markdown scheme.md --apply
```

Header names in another language are given with `--columns`
(`value=Wert,definition=Bedeutung`), and a facet named differently in your table
with `--map`. A heading that names a facet and then explains it
(`topic — subject area`) needs no map. A two-column table whose first column is
the value reads its other column as the definition, and says so. Inline
formatting in a cell (code, bold, links) is read as plain text, and every table
found but not read is listed with how to make it read. Facets the workspace does not declare are reported, not invented;
definitions already registered are kept unless you pass
`--overwrite-definitions`.

To see how well the facets the extractor assigns agree with your own judgement,
label a handful of notes by hand (a JSON file mapping each note to its facets,
kept outside the repository) and score them, before and after any change to
extraction:

```bash
.venv/bin/python scripts/eval_facets.py --workspace-root <workspace root> --labels labels.json
```

Notes can be named by file stem or by any frontmatter field (`--match-on citekey`).
The report gives exact, partial, wrong, missing and extra pairs per facet, and
value-level precision and recall. They count different things: `extra` counts
pairs where your label is empty and values were assigned, while precision counts
every assigned value your label lacks, including the surplus of a partial pair.

**Embeddings (semantic search).** Shipped **on**, and they need an embedding
server on this machine at `base_url` (Ollama, for example); only a local
address is accepted. Set `"enabled": false` in
`<workspace root>/config/embeddings.json` if you do not want them. With
nothing answering, retrieval degrades to its other routes and says so
(`semantic_search_server_unreachable` in `GET /ops/self_check`), and the
server is not asked again for `down_cooldown_seconds` (default 60): on
Windows each unanswered attempt costs 2.3 s. `SECOND_BRAIN_EMBEDDINGS_DISABLED=1`
switches them off whatever the files say; the test suite sets it.

**PDF converter and figures (Track M).** Shipped with **no** PDF converter
selected: PDFs are read by the basic extractor (`pdftotext`), which loses
two-column reading order, tables, equations and figures, and every note says
so. To use docling, install it (`pip install -e ".[docling]"`; it brings
PyTorch and downloads its local models on first use) and set
`"pdf_converter": "docling"` in `<workspace root>/config/conversion.json`
(`"formula_enrichment": true` for equations as LaTeX, slower).
`GET /ops/self_check` shows `pdf_converters` (which one a PDF actually gets)
and `figure_vision`. Reading figures with vision spends the subscription and is
off: switch it on in `<workspace root>/config/ingestion.json`
(`"vision": {"enabled": true}`), or ask per source with `POST /sources/figures`
(dry run first). A source already read badly: `POST /sources/reread`, dry run
first. See `docs/document_conversion_v1.md`.

---

## After a platform upgrade

If an upgrade changes `config/schema_version.json`, derived data may need
rebuilding. Notes are the source of truth and are never rewritten by this:

```bash
# see what would change, without writing
.venv/bin/python scripts/rebuild_graph.py --workspace-root <workspace root> --dry-run

# do it
.venv/bin/python scripts/rebuild_graph.py --workspace-root <workspace root>
```

A rebuild **drops each note's read count**, because access history is keyed to
note ids and a re-identification changes them. Everything else — notes, nodes,
edges, stored vectors — comes back identical.

---

## Finally: add your corpus slowly

Only once Step 8 passes. Add sources **a few at a time** and read the output
before scaling up. A systematic extraction problem found after 200 documents
costs far more than one found after 3.

For a whole folder, ask for the plan first. `POST /sources/process_folder` is a
dry run by default:

```bash
curl -s -X POST "$API/sources/process_folder" -H 'Content-Type: application/json' \
  -d "{\"path\":\"<folder or root:path>\",\"kind\":\"<kind>\",\"agent_id\":\"$AGENT_ID\",\"workspace_id\":\"$WS\"}"
```

It lists every file with what would happen to it: process, book route,
duplicate, already in the workspace, near duplicate, slides, no text layer (needs
OCR), and why. It gives the order (oldest first) and an estimated token cost. When
the plan looks right, run it with `"dry_run": false` and a `"token_budget"`; it
stops cleanly before the document that would pass it. See
`docs/folder_intake_v1.md`.

From here on, `second-brain-operations` is the entry point for every
interaction.
