---
type: "documentation"
title: "Prompt System V1"
description: "All system prompts used by second_brain should live in <repo>/prompts/."
status: "stable"
reading_status: "read"
product: "second brain"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing second brain"
doc_path: "docs/prompt_system_v1.md"
doc_sha256: "8ed12d8fc7f0d527106293ffaf85e2e9f9863fd06cacb736463965fd8fc3c77c"
---

# Prompt System V1

## Prompt folder
All system prompts used by second_brain should live in `<repo>/prompts/`.

## Current prompts
- `prompts/prompt_normalize_note.md`
- the extraction and shaping prompts named in `config/sys_config.json`
  (`academic_paper_template`, `normalization`, `template_drafts.prompt_paths`,
  `template_shape`, `templated_wiki_shape`), plus the per-type template
  prompts `prompts/<type>_remote_template_authoritative.md` the source pipeline
  resolves by file name
- `prompts/prompt_interpret_gaps.md`, read by the gap interpreter and sent to
  the `interpret_gaps` skill

Beside these, every brain-keeper skill carries its own instructions in
`second_brain/brain_keeper/skill_definitions/*.json`, and a few short prompts
live in code (semantic expansion's `PROMPT_PREFIX`, the brain keeper's
`SYSTEM_PROMPT`).

## Naming rule
Prompt files should follow the pattern:
- `prompt_<name>.md`

## Current use
The local-model normalization runtime (`OllamaNormalizationRuntime`) reads its prompt from:
- `prompts/prompt_normalize_note.md`

Nothing in the application constructs that runtime today, so no cache key
depends on that file; its examples were rewritten on 2026-10-11 (Track P-R1) to
come from unrelated fields. Normalization goes through the brain keeper with
`prompts/file_normalization_remote_authoritative.md`
(`normalization.prompt_path` in `config/sys_config.json`), and that is the
prompt whose fingerprint keys the normalization batch cache.

## Examples name no one's subject

A prompt or skill is read by every install, whatever it holds: papers, leases,
recipes, minutes. An example in it primes what the model reaches for, so an
example taken from one collection's subject is that collection's subject
shipped as the platform's behaviour. Examples in prompts, skill definitions,
inline prompts in code, the operator skills under `skills/` and the format
templates therefore either come from several unrelated fields or are schematic
(`A, B, and C`, `<citekey>`, `chapter 4, follows the chapter on X`).
`tests/system/test_prompts_are_domain_neutral.py` scans all of them for terms
known to have leaked from the first instance's corpus, with no recorded
exceptions; a skill's output schema may not name facets either (facets belong to
the install: ask for `facets: {<facet from facet_spec>: [...]}`).

## What changing a prompt costs

Editing a live prompt is not free, because extraction is cached under the
prompt that produced it:

- **The normalization batch cache** is keyed on the source text and a hash of
  the bytes of the normalization prompt file. Change that file and every
  source's cached batches stop matching.
- **The brain keeper's answer cache** is keyed on the skill, a hash of the
  skill definition (instructions and output schema), the input, the model
  tier and, when one is carried, the instance tail's fingerprint. Template
  extraction sends the template prompt's text in the input, so changing either
  the prompt or the skill retires its cached answers. Answers also expire
  after 30 days by default (`answer_cache.max_age_days`).
- **Except the advice (Track S).** A skill may declare input fields that only
  steer wording (`advisory_input_fields` in its definition); they are left out
  of its key. The two extraction skills declare `known_concepts`, the
  collection's concept names, which change with every ingest: before, every
  ingest that added a concept made the next re-processing of every unchanged
  source a full extraction. Now an unchanged source is extracted once; a
  reused answer's concept names are brought to the concept layer's current
  wording before it is written (preferred names, merges, aliases that pass
  the tail equivalence test, no model call), and the processing result says
  `reused_extraction`, why, and the tokens the first call billed. What stays
  in the key, and so still re-extracts: the text, the template prompt, the
  skill definition, the tier, the tail, the facet vocabulary and the facet
  spec (a closed list the answer is chosen from and validated against, not
  advice). No other skill declares an advisory field: their inputs are the
  subject of the judgement (`maintain_concepts`' `known_concepts` are the keys
  its answer may link to, so they are binding there, not advice).
- **Asking again on purpose.** `refresh: true` on `/sources/process` (and on
  `/normalize/file`) skips the answer cache for every agent step that reads
  the source (extraction, the shaping passes, normalization) and stores what
  comes back; `/sources/reread` with `reextract: true` does the same. The
  figure, equation and concept steps after it are not refreshed.
- **Answers stored before Track S** were keyed on the whole input. On the first
  cache miss of an extraction skill after the change, the keeper files every
  such answer under its new key, once per instance (recorded in
  `brain_keeper/cache_migrations.json`): it reads the job record each entry
  names, checks that the entry's key really was that record's input under the
  current skill definition and tier, and copies it. An entry whose job record
  is gone, or that was written under another skill definition or tier, is
  not reused; those sources are extracted once more, as after a prompt change.
- Semantic expansion's cache is keyed on the question and the vocabulary, not on
  its prompt, so editing `PROMPT_PREFIX` costs nothing. The operator skills and
  the format templates are in no cache key.
- The wiki shaping pass (`shape_templated_wiki`) is sent the templated note
  without its run stamps (`import_date`, `generated`): the instant in
  `generated` made every re-run a new question, so the pass was paid again on
  every re-processing of an unchanged source (found in Track S).

What that means in practice: **notes already written are not touched.** Nothing
re-processes a source because a prompt changed. The cost is paid only when an
existing source is processed again (a refresh, a re-read, a re-run of intake on
it): that one run misses the cache and extracts again, once, at the price of a
new intake. New intake costs what it did before. A change worth that is one
that changes what the model does; rewording for its own sake is not.

The last such change was Track P3 (2026-10-11); its journal entry lists the
fingerprints that changed.

**Instance tails follow the same rule (Track R).** A job of a skill that
accepts a tail carries the instance's approved tail appended to its
instructions, and the tail's fingerprint is part of both cache keys (the
answer cache's, and the normalization batch cache's prompt hash) only when a
tail is carried, so every key written without one is unchanged. Approving a
new tail version therefore re-extracts nothing by itself: a source is read
under it the next time it is processed. See `docs/instance_tails_v1.md`.

This makes prompt evolution explicit, inspectable, and versionable.
