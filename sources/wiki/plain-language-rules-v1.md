---
type: "documentation"
title: "Plain-language rules (v1)"
description: "Track AG0. The contract for every text a newcomer reads: the option catalogue, the words for every warning and refusal code, the user guide, and every message an agent is told to relay. The rules are those of..."
status: "stable"
reading_status: "read"
product: "second brain"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing second brain"
doc_path: "docs/plain_language_rules_v1.md"
doc_sha256: "e3954f6b245fc871b8efb179d0d051c8a35ab053f003780a6554d0fef5e0e636"
---

# Plain-language rules (v1)

Track AG0. The contract for every text a newcomer reads: the option
catalogue, the words for every warning and refusal code, the user guide, and
every message an agent is told to relay. The rules are those of
first_run_experience_plan_v1.md ("The rules
for everything a newcomer sees", rules 1 to 9, and S4, S5, S6, S7, S9, S13),
written down once as this project applies them. The catalogue and the guide
point here and do not restate them.

## Where the words live

| what | where | checked by |
|---|---|---|
| one entry for every option a person can meet, with its four lines and what changing it later costs | `second_brain/catalogue/options_v1.json`, loaded by `second_brain/catalogue` | `tests/contracts/test_option_catalogue.py` |
| the plain words for every warning and refusal code that can reach a person | `USER_TEXTS` in `second_brain/next_steps_codes.py`, served as `fix.next.user_text` | `tests/contracts/test_who_settles_each_code.py` and the same catalogue test |
| a short guide for a person, not an agent | [[user-guide-v1|user_guide_v1.md]] | the same catalogue test |
| every sentence the welcome, the example, the tour and help show (Track AG1), templates whose words in braces the code fills | `second_brain/catalogue/welcome_v1.json`, loaded by `catalogue.say` | the same catalogue test, over each text filled; `tests/integration/test_welcome_api.py` checks every response against it |
| every sentence the collection tools show (Track AG2: making, opening and connecting a collection, who may do what, changing a choice, the check of what works), templates checked filled with the file's own `sample_values` | `second_brain/catalogue/setup_v1.json`, loaded by the same `catalogue.say` | the same catalogue test; `tests/integration/test_collections_api.py` and `test_collections_mcp.py` check every response against it |
| the one checker | `second_brain/plain_words.py` (`tests/fixtures/plain_words.py` only re-exports it) | every test above |

Three files, one style: the same checker, the same rules, and
`catalogue.texts()` yields the option texts, the code texts and the welcome's
texts together. A
later phase (the welcome, `help`, `change_setting`, `plan_adding`) shows a
person a catalogue entry or a code's words and writes no sentence of its own
(S4).

## The rules, as applied

**Rule 1. No word the user did not bring.** The forbidden words are in
`FORBIDDEN_WORDS` of the checker (the platform's own: the word for a place
where a collection lives, the names of formats and programs inside the
platform, the words for how notes are made and linked). A code (words joined
by underscores), a field, a route, a file name and any markup are forbidden in
any spelling. The words a person is given: *collection*, *add*, *ask*,
*search by meaning*, *better PDF reader*, *reading pictures*, *your
subscription*, *tokens*. A word that is truly needed and not on the list
(*PDF*, *Word file*, *tokens*) is said in the same sentence, or already known
from rule 9.

**Rule 2. Every choice is four lines**, each one sentence of at most 25 words:
`what` (what it is), `need` (whether you need it, and the signal in the
person's own files it is decided from, where there is one), `cost` (time,
disk and tokens, in that sense), `recommended` (the answer, which is what the
system does when nobody answers). The fifth field, `later`, is S7: whether it
can be changed afterwards and at what cost (at most two sentences).

**Rule 3. One question at a time, the recommended answer first.** The
catalogue's `recommended` line is that answer. A person is asked only the
standing token budget and about deleting or overwriting their own originals
(Track AR); every other entry is settled by its recommended answer and is
changed only when the person asks.

**Rule 4. Defaults that cannot hurt.** Doing nothing never spends tokens,
never deletes, never overwrites. An entry that spends tokens says so in its
`cost` line and has `spends_tokens` set; the shipped default of each is
recorded in `shipped_default` (for a person) and in `default_state`, which
the test of S13 reads against the shipped values (Track AG5): `on`,
`on_when_installed` (`auto`: on as soon as its program is there),
`on_when_created` (on in every collection made now), `background` (done by a
free nightly step another entry carries, named in `background_in`), `asked`
(it spends tokens and ships off, listed by the check with its cost), `choice`
(a value or a wording, not a feature that is on or off; each is accepted by
the owner in the test, with a `default_reason`) or `automatic`. `switch` names
the setting that is the feature's on and off; `modifiers_off` the others that
may ship off. A new switch that ships off with no token cost fails the test.

**Rule 5. Say what happened in plain words, and what to do next.** The words
for a code are in `USER_TEXTS`; they name no code, field or route.

**Rule 6. Nothing assumes the subject or the language.** The texts are
templates in one language (`language` in the catalogue); the agent speaks the
user's. No text names a language, a field of study or a subject; a test checks
the catalogue against the leaked-term list of the domain-neutral guard and a
short list of language and subject words. Needs are read from file structure
(the `signals`), never from a subject.

**Rule 7. Nothing is a trap.** `later.possible` is `yes` (can be changed at
any time, nothing re-read), `with_cost` (turning it on later means reading
documents again at a cost) or `no`. Anything not `yes` has
`ask_before_first_run` set and names a signal in the user's files, so the plan
asks it before the first run and offers it later as a plan with its cost.
Today only the better PDF reader is `with_cost`. A change that applies only to
documents added afterwards (keeping copies of originals, how long documents
are cut) says so in its `later` line and is still `yes`: nothing is re-read.

**Rule 8. Some things are not options.** Entries of `kind` `automatic` (the
map of each document, duplicates found by content, links between notes,
concept pages, the record of what changed, the safety of the originals) are
described for the tour and `help` and are never asked: no settings, nothing
turned on later. Settings that are the system's own (the set of labels, the
kinds of link) are declared internal with a reason, never made options.

**Rule 9. Costs in words.** A cost is time, disk and tokens of the user's
subscription, never a share of their plan (no agent program reports one) and
never a price. Starting with a few documents is always offered.

## Binding on the catalogue (S5, S6, S9)

- **S5.** Every text passes `plain_words.problems`: no forbidden word, no
  technical spelling, no sentence over 25 words, each of the four lines one
  sentence. The test runs over every option, every signal and every code text.
- **S6.** Every setting a person could change has an entry. The test lists the
  `config/*.json` of the install, the settings only a collection holds (its
  budget, maintenance, intake, gap filler, access, naming, source roots) and
  the defaults the reading code declares; each must be named by an entry's
  `settings` (`file.key`) or declared in `internal_settings`, a list of
  groups, each a reason and the exact settings it covers (never a pattern, so
  a new setting cannot be swallowed unseen). A new setting fails the test
  until it is one or the other, and a declaration of a setting no file holds
  fails too.
- **S9.** A feature adds one catalogue entry and one place in the user guide:
  each entry names a `guide_section` that is a heading of the guide, and the
  guide names the entry by its plain `name`. Every entry that spends tokens is
  named under "What uses your subscription".

## Adding an entry

1. Add the object to `options` in `second_brain/catalogue/options_v1.json`,
   with every field of the entries beside it (copy a neighbour).
2. Name its settings as `file.key`, or declare them in `internal_settings`.
3. Name it in the user guide, under the heading its `guide_section` gives.
4. Run `tests/contracts/test_option_catalogue.py`. Changing the shape of the
   file raises `catalogue_version` and the file name.

## What this does not settle

The catalogue holds the words. It does not decide, and no AG0 code does,
whether a feature is on or off (S13, phase AG5) or how a setting is changed
(`change_setting`, phase AG2). The signals the plan reads from the user's files
are named and described here and built in AG3.
