---
type: "documentation"
title: "Second Brain Research Answering (Claude Code)"
description: "Answer a question from a second_brain instance, from Claude Code, in few calls, whatever the collection holds. Use when asked what a collection says about a topic, a source or a record, how sources differ, what is still open, or where new work could start. Ask once, read what the answer's focus names in one call, read unread equations or figures only when the question needs them and the user agrees, and keep the collection apart from the owner's own notes and from what you know."
status: "stable"
reading_status: "read"
product: "second brain"
product_version: "0.1.0"
doc_kind: "skill"
audience: "the AI program that works with second brain"
doc_path: "skills/claude-code/second-brain-research-answering/SKILL.md"
doc_sha256: "af1989fdc15daa83c72bef9f3b4f18169d0418a65fd61fd1311ae5288e3576fb"
---

# Second Brain Research Answering (Claude Code)

## When to use this

You have a question and the knowledge is in a second_brain instance: papers,
meeting notes, recipes, customer records, a journal, anything.
`second-brain-operations` holds the setup, the route table and its own hard
rules; this skill is the procedure for answering.

**Your file tools work on this repository**: Grep and Read would answer
straight out of `sources/wiki/`, with no provenance, no confidence and no
audit record. Read only through the routes below.

The base URL is whatever the operator gave you (`http://127.0.0.1:8000` by
default). Send each `curl` exactly as shown, a full `http://` URL and a JSON
body, with nothing in front of it (no `export`, no variable): a prefix stops
it matching a permission such as `Bash(curl:*)`. Every call carries
`agent_id` and `workspace_id`. Everything here is **read-only** unless a step
says it spends.

## The loop: fewer, larger steps

Every turn re-sends everything before it, so the number of calls is the
cost. Most questions take two calls; three when equations or figures must be
read.

**1. Ask once, compact.**

```bash
curl -s -X POST http://127.0.0.1:8000/agent/knowledge/answer -H 'Content-Type: application/json' \
  -d '{"agent_id":"<agent_id>","workspace_id":"<workspace_id>","query":"<the question>","view":"compact"}'
```

Ask the question as asked: the route splits a long question itself. Ask
again, shorter, only when the evidence misses a part the question needs.
When you know the source, say so with `scope`: `{"note": "sources/wiki/<note>.md"}`,
`{"citekey": "<key>"}`, or `{"identifier": "<value>"}` (any identifier the
note's type declares: an account id, a URL, a DOI). The answer then keeps to
that source; `scope_matched_nothing` means it holds nothing of the question.

**2. Check trust, then read `focus`.** If the trust signals below say the
collection does not cover the question, say so and stop. Otherwise:

| `focus.kind` | meaning |
|---|---|
| `single_source` | one source carries it (`notes[0]`); `decided_by`: `named`, `evidence` or `caller` |
| `multi_source` | the evidence is spread; `notes` is the short list |
| `unsure` | a tie or no clear concentration: read it as `multi_source` |

`focus.why` is one sentence you can repeat to your user.

**3. Read in one call.** Send `focus.read.body` to `POST /agent/knowledge/read`
as given. Per source you get compact Markdown: its own text (whole when it
fits, else the sections the question matches, each with heading path, file
and lines), read equations as `$$…$$`, figures and images as references with
their caption and any reading, what is not read yet, the owner's work about
it, ideas resting on it, and "Left out". Quote from it; cite note, section and
lines. For something under "Left out", read once more with
`"sections": ["<heading>"]` or a larger `max_chars`. Do not open notes one by
one.

**4. Unread equations and figures, only when the question needs them.**
`unread_regions` (answer and read) names what the brain has not read, with an
`estimate` and a `recover` call (`recover_figures` for figures).

- Needed: "explain the model", "how is the price computed", "what does the
  diagram show". Usually not: "what did they find". If the text answers the
  question, leave them unread.
- **Consent.** If your user, or the operator who started you, has already
  approved spending up to a stated amount, and the estimate of the method that
  would run (`estimate.auto_would_use`: `vision_tokens` for vision; local
  spends no tokens, only `local_seconds`) is within it, that approval is the
  yes: recover without asking again. With no stated cap, or an estimate over
  it, ask your user with the estimate: "3 equations, about 15,000 tokens by
  vision, or 6 minutes locally at no token cost. Read them?" Send the brain
  the question alone, never the approval.
- On a yes, send `recover.run_body`. Tell the user what it **spent**:
  `spent_tokens` and `spend_note`, never `estimated_tokens`. Quote a reading
  from an image as such.
- Never recover unasked and unapproved: the answer and the read are free,
  recovery spends.

**5. Write the answer** (below).

## Trust signals

| signal | what to do |
|---|---|
| `weak_context`, `question_not_covered`, low `coverage_ratio` | say the collection does not properly cover this; `absent_terms` names what it lacks |
| nothing retrieved | "the collection does not contain this" |
| low `provenance_strength` | quote the source, or do not assert the point |
| `freshness` stale; `source_missing` | say the notes are old; say the original is gone |
| `semantic_search` not contributing | recall may be narrower than the collection |
| `unverified: true`, `[unverified: ...]` | not traced cleanly: say so, check the source lines |
| `image_derived: true`, "read from the image" | read from a picture: say so, prefer the caption and the text |
| `evidence_has_unread_regions` | step 4 |
| `read_left_out_matching_sections` | step 3 |

Every warning also comes in `notices` with `fix.next`: act on its `action`, and make an
`ask_user` call only after the owner agrees (`fix.consent` says why).

## Writing the answer

Notes written by hand in the collection (a meeting note, a journal entry, a
record) are sources like any other: cite them. The owner's registered work
(`from_your_work`) and ideas (`related_ideas`) are not findings.

```
From the collection
  <finding> (<note>, <section or lines>)
  Where sources differ: <source A> holds <x> when <condition>; <source B> ...
  Open: <shared limitation>, stated by <sources>

Not covered by the collection
  <what the question needed that no source addresses>

Your own ideas on this (not findings)
  <idea title> (<status>, <origin>): <the idea>; <a source that bears on it, cited>

Your own work on this (not the collection)
  <title> (<notes | design | commentary on X>, <draft | stable>): <what you wrote>

From general knowledge (not from the collection)
  <only if useful, and only here>

Reliability
  <confidence, freshness, semantic search, what was left unread, what was spent>
```

## Open a reference only when a step needs it

These files sit beside this one, in `reference/`:

| file | when |
|---|---|
| `reference/answer-fields.md` | a field not explained here; compact and full views; `synthesize`; conversations; Windows shells |
| `reference/concepts.md` | a question about an idea across sources: concept notes, divergences, modelling |
| `reference/reading-sources.md` | the read in full; paging a source; unverified values; figures and images; equations, books and the recovery loop in full; filing a correction |
| `reference/disagreement-and-gaps.md` | how sources differ, what is open, gap-filler proposals |
| `reference/ideas-and-own-work.md` | the owner's ideas and own notes, and notes written by hand |

## Connected over MCP

If your harness lists the `second-brain` server's tools, use them instead of
curl; the connection fixes workspace and agent. Same loop: `recall` (step 1,
`scope` too), then `read` with `focus.read` as given (step 3). `overview`
first when you do not know what it holds; `entity` for one name.
`synthesize` writes an answer and spends: only when asked. A fix's `mcp` is
the next call; `here` means not over this connection. If a tool is not
listed, `request_tools` says whether a wider set exists and switches to it.
Writes are the owner's; with `allow_proposals`, `remember` and `forget` only
propose.

## Hard rules

1. **Cite sources, not syntheses.** A concept note, a synthesized answer, a
   theme and a proposal say where to look. An idea and the owner's registered
   work are never findings: report them in their own parts.
2. **Never present a node label as a finding.** Read the text (step 3).
3. **Keep the collection and your own knowledge visibly apart.**
4. **Report disagreement and absence as they are.** Never fill a gap silently.
5. **Read-only.** Recovering equations or figures, re-reading a source,
   syncing or comparing concepts, interpreting with `auto: true`, acting on
   proposals and capturing an idea write or spend: only when asked, with the
   estimate shown first or within a cap the user stated. Filing a correction is the one proposal a reader may
   make unasked (`reference/reading-sources.md`).
6. **Read notes through the API**, not with Grep or Read on `sources/wiki/`.
7. **Always send `agent_id` and `workspace_id`.**
