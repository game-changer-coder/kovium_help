---
type: "documentation"
title: "The welcome, the example, the tour and help (v1)"
description: "Track AG1 of first_run_experience_plan_v1.md (D2, D3, D3a, D3b, D3c). The contract for what a person new to Kovium meets in their first five minutes, over HTTP and over the MCP server, with no command typed by them: a welcome that reads..."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing Kovium"
doc_path: "docs/welcome_v1.md"
doc_sha256: "244055dc764ad73e320dd63d472ac248fe30a75b11c9192c4bf182a71254b92d"
---

# The welcome, the example, the tour and help (v1)

Track AG1 of first_run_experience_plan_v1.md
(D2, D3, D3a, D3b, D3c). The contract for what a person new to Kovium
meets in their first five minutes, over HTTP and over the MCP server, with no
command typed by them: a welcome that reads the install's state, a small
example collection that answers with its sources, a tour of what happens to a
document, and help about the software at any time.

Every sentence a person is shown comes from the catalogue
(`second_brain/catalogue/welcome_v1.json` and `options_v1.json`) or the user
guide ([[user-guide-v1|user_guide_v1.md]]); the code writes none of its own
(S4), and each passes the plain-words rule
([[plain-language-rules-v1|plain_language_rules_v1.md]], S5). What a response
adds beside them (`next`, `on`, `look`, `ask_with`) is for the agent.

## Routes

None needs a collection of the caller's: they answer an install that has none.

| route | does | spends |
|---|---|---|
| `GET /welcome` `{agent_id?, workspace_id?}` | the next step for this install (no `workspace_id`) or for the collection a connection is bound to | nothing |
| `GET /help` `{question}` | a question about Kovium itself, answered from the full manual when it is fetched and describes this version (Track AG6), else from the user guide and the option catalogue | nothing |
| `POST /welcome/manual` `{update?}` | fetches the full manual (the help collection) onto this computer, on the person's yes; `update` replaces a copy of another version (Track AG6) | nothing |
| `POST /welcome/example` `{question?, agent_id?}` | opens the example collection (registered on first use, built by code), suggests three questions; with `question`, answers it from the example with sources | nothing |
| `POST /welcome/tour` `{step?, try_it?, add?, approval?, reset?, wait?, agent_id?}` | one step of the tour; `try_it` runs a step's exercise (`both_ways`, `practice_night`, Track BA); at step `add`, adds the waiting page; `reset` puts the example back | only `add: ai` with an `approval` |
| `POST /welcome/nightly` `{answer, example?, workspace_id?, agent_id?}` | the person's answer to the nightly AI question (Track AX); with `example`, the switch only shown in the example | nothing |

### `GET /welcome`

Stateless: the state is read from the install each time, so a second session
resumes where the first left.

| `state` | when | `say` | `ask` |
|---|---|---|---|
| `new` | no `workspace_id`, the example not opened yet | what Kovium does, and what tokens are, where the word first comes ("Tokens are how your AI subscription counts the work its AI does: reading and writing use them, while asking your own collection here uses none.", fix round R6 of Track BA) | see the example first? (`yes` recommended; `on.yes` is `POST /welcome/example`) |
| `example_open` | the example was opened | it is open, a collection of one's own can be made or opened from the conversation, help at any time; `questions` | the tour? (`yes` recommended; `on.yes` is `POST /welcome/tour`); once the tour was started, your own collection? (`yes` recommended; `on.yes` is `POST /collections`, `name` and `holds` filled from the person's words and `working_folder` by the caller; Track AG2, [[collections-v1|collections_v1.md]]); its answer is first the question of where the collection goes (Track AZ1), the same as for any collection |
| `collection_ready` | a bound collection that holds notes | how many notes it holds; help | none |
| any bound state, first | the install never answered the nightly AI question, the collection is not the example, nobody chose its level, and the agent may change its settings (Track AX) | as below | the nightly AI question (`code: nightly_ai_choice`, `settled_by: user`; `warnings` carries the code, so `notices` gives the caller its next step and the person's words) |
| `collection_empty` | a bound collection with no notes | it holds none yet, and how documents get in (or, for an agent that only reads, who adds them; Track AG2) | for an agent that may add (Track AG3): which files or folder (`adding.where`, `yes` recommended; `on.yes` the free plan, `fill: paths`) |
| `collection_adding` | a bound collection with a job running | documents are still being added | none |

`ask` is `{text, recommended, answers, on}`: the answers in order, recommended
first, each with what it does in plain words; `on` the call for each answer
that makes one. `help` is the call for a question about the software.
A connection bound to a collection, whichever it is (the example's working
copy included), is offered only what works there: what the collection holds,
asking it, and help; never the example or its tour (fix round R5).

### `GET /help`

The guide is cut into passages (a paragraph or a list item, each under its
`##` section), and each catalogue entry is one passage (its four lines and its
"later" line, with no label in front). The question's words are matched to
them by code, two forms of one word counted as one (a shared beginning of
three quarters of the shorter, four letters at least) and rarer words
counting more (a word of a passage's heading half again, a telling one
twice); at most three passages come back, best first, each with its `source`
(`{document: "user guide", section}` or `{document: "option catalogue",
entry}`).

**Found only when it is** (fix rounds R3, R8): words are matched by their stems
(two forms of one word are one word, a little less than the word itself; a
word of two letters only as itself, and weighing little), each weighted by its
rarity over the guide, the catalogue and the project's own documents in the
same language, this contract left out since it quotes the test questions (so the words every question uses weigh little; a word found
nowhere weighs a fixed amount, `UNKNOWN_WEIGHT`), and a word of a passage's
heading counts half again. A catalogue entry's own lines (what, need, cost,
recommended, later) are its passage. A passage answers only when it holds at
least `MIN_COVERAGE` (0.35) of the question's weight and half of its telling
words, its rarest (`MIN_TELLING`). Otherwise `found: false`, no passage, and
one plain sentence that the guide does not say. The answer's `help_collection`
says which copy of the manual stood behind it: `not fetched`, `fetched`,
`other_version` or `damaged`. The offer of the manual (below) is added only
when this install says where the manual is kept; with no address set nothing is
promised that could not be done (fix round R9, kept).

The thresholds were **chosen on** a fixed set of 33 real how-to questions
(`tests/fixtures/help_questions.py`): 25 the guide covers, several worded far
from it, and 8 it does not. On that set, as fitted: precision 24/25, 24 of the
25 covered answered from an expected section (missed: "How do I add a whole
folder of PDFs?"), 7 of the 8 others refused (answered wrongly as found: "Can
two people edit the same note at the same time?", from the entry on other
versions of one document). It is a fit to the set, not an independent measure.
The guide gained the common how-tos for it: adding files, folders and books;
capping or stopping spending; getting your notes out; removing a document;
working without the internet; what you need to run Kovium (an AI program
you already use on a subscription, by kind and by name); what stays on your
computer; what happens when two notes disagree; and that the example and its
tour are for a first start. Ask in the catalogue's language (English); the
agent relays in the person's.

### `POST /welcome/example`

Opens the example collection (below) in its **working copy**: what ships is
copied into `sandboxes/welcome_example` under the instance root (made on
first use, refreshed when what ships changed), registered for the caller
(agent id `welcome` when none is given), then built by code when any of its
notes changed since it was last opened. The shipped example is only read
(fix round R1). `opened` reports `built`, `seconds`, `notes`,
`concept_pages` (the pages there now) and `maps` (every note's map now, the
tour's added page included). `folder` says where the paths are: the
collection folder relative to the install folder, every other path relative
to the collection folder; no machine's own path is ever shown (fix round R2). Without
`question`: `questions`, the three suggested questions (the catalogue's
`example_questions`), and `ask_with`, the call to ask one.

With `question`: the governed answer route (`POST /agent/knowledge/answer`,
no model call: `synthesize`, `judge_relevance` off, `widen: cache_only`)
answers it from the example, and `answer` lists at most four items, each
`{text, term?, source: {note, section, lines, file, original: {file, relative_to}}}` (the original in the working copy, relative to its folder: fix round R9; since Track AG2 every original is inside it): a note named
by its title, a definition's meaning as `text` with its `term` apart, and an
item that only names a note or a concept left out while others exist. When
the answer's `focus` says one note carries the evidence, that note's ranked
items come first, then its `## Claims`, so a page whose claims do not repeat
the question's words still answers it (fix round R4). When the answer
carries `question_not_covered`, the collection does not say: `found: false`,
`answer: []`, and the suggested questions again; never a guess. After a found
answer, `ask` offers the tour, once: not after the tour was started.

### `POST /welcome/tour`

**A tour you do, not read (Track BA, 2026-10-24).** The owner's first run of
the earlier seven-step tour found it no better than reading the guide: each
step was text, the files it named had no place ("Look at: <file>", one folder
off), Obsidian came in step 6 after steps that needed it, "kept" and "pointed
to" were not explained, the nightly tidy was called free while its AI steps
spend tokens, and the words were flat. Track BA made eight steps, each with
something to do; every file given with its full place; two exercises; and a
warmer voice.

**A tour that always says what comes next (Track BD, 2026-10-24).** The owner's
second run, through a real agent over a chat app, found that a step did not say
how to go on, nor what kind of step it was or how far along; several steps
ended with a question that led nowhere ("What catches your eye first on the
map?"), and over a chat such a thread is hard to steer back from; the parts of
an answer read as the answer a person gets; there was no picture of a map
without Obsidian; and the full manual was never offered, because the tour's
end came only when step `add` was asked for again. Now: nine steps (the night
has its own), each titled with its place and its kind, each message ending
with the way on, a question only where the next step needs its answer, the
machinery followed by the reply a person receives, a picture of the map, and
the end, with the manual, carried by the answer that adds the page.

Each step is `{step, number, of, category, title, say, do, check_in, look,
folder, next_step, next, way_on}`; a step with an exercise also has `ask`, its
offer, an optional extra; the notes step also has `answer` and `reply`.

- `title`: composed by code (`tour.heading`): "Step 3 of 9 · ⚙️ Configuration:
  Where your files are kept", the counter counted from the tour's own list, the
  kind's emoji and name (`tour_categories`, `tour.category_*`), and the step's
  own title. The kinds (BD2):
  - 🧭 **Getting around**: where things are, and how to look at them;
  - 🛠️ **Machinery**: how Kovium handles your files and answers;
  - ⚙️ **Configuration**: how to turn a feature on or off, or change a choice;
  - 🎉 **Your turn**: you add something yourself and watch what happens.
- `say`, `do`: the catalogue's words, relayed word for word. The first step's
  `say` starts with the intro, then what each kind means, once (BD3: "Each step
  says its kind in its title." and one line per kind, in the order the tour
  meets them; no emoji, so the message keeps to two).
- `check_in`: `{text, settled_by: user}`, a question back to the person, **only
  where the next step needs the answer** (BD6): the first step's way to look.
  Every other step has none; the caller agent asks nothing of its own during
  the tour.
- `way_on` (BD1): the line that ends the message, relayed word for word: "Say
  “next” for step 4 of 9: Inside a note." ("Or say ..." where the step offers
  an exercise; at step `add`, "Say “free” or “AI”, and the page goes in.").
  When the person says next, or anything meaning go on, the caller makes the
  `next` call.
- `ask`: the offer of a step's exercise, `yes` recommended (`on.yes` is the same
  route with `try_it`), `no` going on to the next step (`on.no` is `next`).
- `look.files`: each file or folder to look at, `{what, place, in_collection |
  outside_the_collection, link, obsidian?}`. `what` is the catalogue's words;
  **`place` is the full path on this computer, given at run time** so a person
  finds the file, and never written into a note, a record or the audit; `link`
  is its `file:` address; `obsidian` its `obsidian://open?path=` address, for a
  note or a map inside the vault folder. `look.describe_offer` offers "describe
  it" for every step with files: the agent reads the file at its place and says
  what is inside.
- Every other path in a reply is relative to the collection folder that
  `folder` names, as before (fix round R2); a note is `{title, file}`. `folder`
  comes with the first step only (fix round R6 of Track BA).

The night's notice (Track AX) is not added to any step of the tour: the tour has
a step for the night, and the notice's question would lead away from the next
step (BD6). Reaching the night's step counts as the notice given, so the example
does not say it again.

The steps, as a person reads them (the intro, then each step: its title, its
text, what to do, the places shown, the way on):

> 🧠 9 short steps, a few minutes in all, with something small to try in each.
> You can stop at any step.

1. **`look`**, Step 1 of 9 · 🧭 Getting around: A way to look. "A collection is
   simply a folder of files on this computer, so you can open it like any other
   folder. The nicest way is Obsidian, a free program for folders of notes,
   which shows the maps as pictures. A text editor or your file explorer works
   too, or I can describe any file right here." Do: "To use Obsidian, download
   it, choose to open a folder as a vault, and pick the folder shown below."
   Shown: the folder to open as a vault (the example's `sources`, below), the
   example's folder, the three ways (`look.ways`) and where to download
   Obsidian (`look.download`). The one question of the tour: "Which way suits
   you: Obsidian, a text editor, or me describing things here?"
2. **`journey`**, 🛠️ Machinery: One document's journey. "Every document you add
   makes four stops, and all four stay inside the collection's folder. First
   the original, as you gave it. Then a plain text copy, in a simple format
   called Markdown. Then that text sorted into the layout every document of its
   kind gets. Last, the note the AI writes from it." And, honestly for the
   example: "Here the originals are plain text already, so the copy is the
   same, and the layout was filled by code." "The notes were written in
   advance, so opening the example costs nothing." Do: "Find the same document
   at all four stops, using the places below." Shown: the four stops
   (`look.stops`, the manifest's `journey`).
3. **`files`**, ⚙️ Configuration: Where your files are kept. "Your originals are
   kept in the collection's folder for originals. To add PDFs or other files,
   put them in any folder and tell me where they are. If they are outside the
   collection, I ask you one thing first. Keep a copy inside, which stays safe
   even if the outside file moves or is deleted. Or point to it, which uses no
   extra disk, but its folder must then stay where it is." Then (fix round R7):
   "A source line that starts with a short name and a colon, such as library,
   points into a named folder. The collection keeps those names in its settings
   file for folders, shown below." Shown: the folder for originals, the folder
   the example points to (`library`), each original with where it is kept
   (`look.originals`), and the settings file that keeps the short names of
   folders (`config/sources.json`). Do: "Find the originals in the folder for
   originals and the one in the library folder. Then open one note of each and
   look at its line called source, near the top." Offer: "✨ Want to see the
   difference? I will put two small notes in a folder outside the example, keep
   a copy of one and point to the other, for free." (`try_it: both_ways`).
4. **`notes`**, 🛠️ Machinery: Inside a note. "From each document the AI writes
   one note: a short summary, the claims, the evidence, the open questions and
   the definitions. Its header says which original it was made from. Every
   answer you get names the note, the section and the lines it comes from, so
   you can always check it. A note is a plain text file you can read, keep and
   edit." Do: "Open the note below and read its short summary and its claims.
   Then see how the example answers the question below, and where each part
   comes from." The step carries its question (`question` in the catalogue,
   "What does a note hold?") as `ask_about_it`, and (BD7) answers it in the
   step itself: `answer`, the parts as `POST /welcome/example` gives them, then
   `reply`: `machinery` ("This is the machinery: the parts an answer is built
   from, each with the place it comes from. You will not normally see it."),
   `intro` ("Your agent keeps the parts that answer the question, leaves the
   rest out, and writes one reply with the sources at the end, like this:"),
   `text`, a reply written in advance in the catalogue (`reply`), and `sources`
   ("Sources: How a document becomes a note, section Settings and options."),
   from the notes it rests on (`reply_sources`) with the section the answer
   found in each. A test checks those notes are ones the answer's parts come
   from, so the reply cannot drift from the example. Called without the answer
   (`answer_of`), the step gives `ask_with`, the call that asks it, as before.
5. **`map`**, 🛠️ Machinery: The map of a document. "Each document also gets a
   map: one picture of its main points, its open questions and the ideas it
   connects to. Code draws it by itself, seconds after the document is added,
   with no AI and no cost. In Obsidian it opens as a picture; a text editor
   shows it as a list of boxes and lines." Do: "Open the map of the same
   document and find its title, then its main points around it." Shown: the map,
   its note, the map as plain text (`look.map_outline`, fix round R7), and
   (BD5) a picture of the map: `look.picture`, `{what, place, in_collection,
   link, say}`, `say` being "No Obsidian? Here is a picture of the same map.".
   The caller sends it as an image to a person who looks without Obsidian,
   where its program can send files (a chat app through OpenClaw, for one),
   else gives its place. The picture ships in the example's `pictures/` (below).
6. **`links`**, 🛠️ Machinery: Links between notes. "Notes are linked through
   the ideas they share, so a question about one thing also finds the notes
   about a related one. The links sit at the bottom of each note, and each says
   what it connects to and how." Do: "Scroll to the bottom of the same note and
   follow one of its links." Shown: the note, and its `## Links` lines as titles
   (`look.links`).
7. **`concepts`**, 🛠️ Machinery: Concept pages. "Each idea that several
   documents share gets a page of its own, saying what each document says about
   it. Compare the two pages below. One is ready, written the way the AI writes
   it. The other is only a draft put together by code, and waits for the
   nightly tasks." And: "In this example the ready page was written in advance,
   so nothing was spent on it." Shown: the ready page and the waiting page
   (`look.pages`, each `{title, state: ready | waiting}`). Do: "Open both pages
   below and compare the part called Development in each." Offer: "🌙 Want to
   run a practice night? It fills the waiting page from an answer prepared in
   advance, so no AI runs and no tokens are used." (`try_it: practice_night`).
   Once the practice night ran (fix round R6), the step says what is there now
   and offers it no more: "Each idea that several documents share gets a page
   of its own, saying what each document says about it. Both pages below are
   ready now: one was written in advance, the other on your practice night."
   The places are the ready page first, then the other, named "The page filled
   on your practice night" once it is (in the practice's reply too).
8. **`night`**, ⚙️ Configuration: Turning the nightly tidy on or off (BD2: the
   night had shared the concepts step). "Here is what a collection does at
   night, and the few words that choose it." Then the costs, honestly: "Every
   night a collection tidies itself. The free part costs nothing: new label
   words, renamed files, missing search entries and unread equations. The part
   that uses the AI, such as writing concept pages, uses tokens from your
   subscription, up to the limit you choose." And the words: "For a collection
   of your own, say "turn the nightly tidy on" or "off", and "nightly AI
   light", "medium", "heavy" or "off"." Shown: the settings file that keeps the
   nightly choices (`config/maintenance.json`). Do: "Open the settings file
   below and find the line that says whether the nightly tidy is on. In this
   example it is off."
9. **`add`**, 🎉 Your turn: Add a page. "One page waits in the example's inbox.
   Add it now, and watch its note, its map and its links appear." Shown: the
   waiting page in the inbox. Do: "Open the waiting page in the inbox below and
   read its first lines." The offer of the two ways, as before (below).

**The tour's end (BD8).** The answer that adds the page, either way, carries the
end itself, and so does the step once the page is in: "Once you have a
collection of your own, you can ask for your own Kovium agent for it,
which answers the way you like."; `ask`, the manual offer, which says what the
manual is for ("The example holds only a few pages for this tour, while the
manual answers any question about Kovium."); `also`, putting the example
back; `way_on`: "That is the tour. Say “manual” to get the full manual, or ask
me anything about Kovium."; and `tour_end: true`. Where the manual cannot
be offered, that is said (`tour.manual_pending`, or `tour.manual_ready` when a
copy is here), `ask` is putting the example back, and `way_on` is the tour's
last sentence ("That is the tour. Ask the example anything, or ask how anything
works."). Before, the end came only when the step was asked for again, and an
agent that went on without it never offered the manual (the owner's run).

**The vault folder is the collection's `sources`.** It holds every stop of a
document's way (the originals, the two copies, the notes with their maps and
concept pages) and nothing the code keeps for itself: the whole collection
folder would also show `derived/` (a second copy of every concept page and
map, which makes Obsidian's links ambiguous), the database and the settings.

#### The exercises (`try_it`)

Both free: code alone, no AI program called (the tests set a tripwire on the
brain keeper agent), written only into the example's working copy and, for the
first, one folder beside it. A second run says it already ran
(`tour.both_ways_already`, `tour.practice_already`); the tour's reset puts both
back. The reply is `{tried, ran, say, look, do, next_step, next, way_on}`: something to look at, and
the way on to the step after the exercise's own (Track BD1, BD6: no question back).

- **`both_ways`** (BA5). The example's two small notes (the manifest's
  `practice.both_ways`, written by hand) are put in a folder beside the working
  copy, `sandboxes/welcome_example_outside` (`practice.OUTSIDE`), standing for
  a person's own folder. One is **kept**: copied into the collection's folder
  for originals (`sources/raw/`), as "keep a copy" does. The other is **pointed
  to**: left where it is, its folder named for the collection through the
  settings path `change_setting` uses (`originals.look` and `originals.settle`
  of Track AZ4, the name after the folder's own, `welcome-example-outside`;
  the folder is written in full in the working copy's `config/sources.json`,
  as for any outside folder). Each is read by code as a note written by hand,
  its header recording where its original is (`source`:
  `.../sources/raw/a-note-kept-as-a-copy.md` against
  `welcome-example-outside:a-note-pointed-to.md`), and taken in as the waiting
  page is (graph, links, map). `say` is "🎉 Done, and nothing was spent!" and
  Track AZ4's own sentences ("1 original from outside is copied into the
  collection's folder for originals.", "1 original stays where it is, and the
  collection points to it.", "Their folder gets the short name
  welcome-example-outside, so a move of that folder later needs only one
  change."), then (fix round R7) "The short name welcome-example-outside
  stands for the folder outside the example. The collection keeps it in its
  settings file for folders, so a move of that folder needs only that one line
  changed." (that file is among `look.files`), then "Each note says near its
  top, on its line called source,
  where its original is." `look.files`: the folder outside, the kept copy, its
  original still outside, its note, the original pointed to, its note;
  `look.kept` and `look.pointed` each `{note, source}`. Do: "Open both notes
  and compare the line called source near their top."
- **`practice_night`** (BA4). In a collection of a person's own, the AI keeps
  concept pages up to date at night, using tokens; the example's pages are put
  together by code, to keep it free. The example ships answers prepared in
  advance, by hand, in the shape the AI's concept answer takes
  (`practice/prepared_night.json`): `ready` (the page `provenance`) is applied
  when the example is opened or derived, so one page reads as the AI writes
  one; `waiting` (the page `document map`) stays a draft ("Not synthesized
  yet") until the practice night applies it. An answer is applied through the
  concept layer's own steps for an AI's answer (`ConceptService.sync` with the
  agent way chosen for that one pass), with a stand-in (`PreparedAnswers`) in
  place of the brain keeper agent that answers only `maintain_concepts`, only
  for the prepared page and only for a note it has an answer for: the same code
  a real night runs, so the page afterwards is what a real night makes of such
  an answer (`synthesis: brain_keeper`, `generated: prepared_answer/practice-night`).
  `say`: "✨ The practice night is done, and the waiting page is filled. It was
  a practice with an answer prepared in advance, so no AI ran and nothing was
  spent.", "A real night with the AI uses tokens from your subscription, never
  more than the level you choose.", and the levels with their most, from the
  option catalogue's numbers (Track AX): "Light allows up to 50,000 tokens a
  night, medium up to 150,000 and heavy up to 400,000." `look.pages`: both now
  `ready`. Do: "Open the waiting page again and see what the practice night
  wrote."

A `try_it` that names no exercise is refused by the request model (422).

At step `add`, `ask` offers two ways, the cheaper first:

- `add: free` (recommended): the page, written as a note by hand, is copied
  into the notes and read by code as the person's own note: into the graph, its
  concept pages, its `## Links` and its neighbours', its map. `added`, `note`,
  `map`, `links`, `concept_pages`, `do` ("Look for its new note and its map in
  the example's folder."), `ask_with`: the sample question (`sample_question`
  of the catalogue) asked of the example, and the tour's end (above).
- `add: ai`: the AI reads the page and writes a note of its own, through
  `POST /sources/add`'s planner and runner. Without `approval`: the plan,
  `estimated_tokens`, `estimated_range`, `limit` (a one-time limit for this
  page alone: the high end of the range, rounded; never the example's
  standing budget), `approval`, and `ask` (recommended `no`, which is the free
  way). With the plan's own `approval` it runs, as a background job unless
  `wait: true`; a code from another plan, or a page that changed since, is
  refused before anything runs (`adding_plan_changed`,
  `adding_approval_expired`). After either way, a second add is
  `tour_sample_added` (409) or `added: false`.

`reset: true` (or `step: "reset"`; an unknown step's refusal names it) puts the
example back as it was before the tour: what the exercises left (the kept copy,
the folder outside, their record) taken out, the notes the tour changed
(neighbours whose `## Links` gained a page) restored from the snapshot taken
before the first change (`maintenance/tour_snapshot/`, which also keeps the
settings file the exercise changes), the notes it added and their maps
removed, the concept pages built again (the ready page from its prepared
answer, the waiting page a draft again), the graph rebuilt.

**The voice (Track BA6).** The welcome and the tour speak warmly, with short
hook lines and a few emojis from a fixed set (`plain_words.EMOJIS`: 👋 🧠 👀 🧭
📂 📄 📝 🔍 🔗 💡 🌙 ✨ 🎉, and since Track BD2 ⚙️ and 🛠️, each read with its
variation selector as one emoji, never the bare character), **at most two in
one message** (a reply's title, `say`, `do`, `check_in`, offer, way on and the
words beside its places, together; `plain_words.MAX_EMOJIS_PER_MESSAGE`), one
in every step's title: its kind's. "Configuration", the owner's name for a
kind, is the one word allowed to begin with the forbidden "config"
(`plain_words.ALLOWED_AFTER`). No other text
of the catalogue holds one, and an emoji outside the set fails the plain-words
rule in every text (the rule is tightened, never loosened); an emoji is not
counted as a word. Rules: [[plain-language-rules-v1|plain_language_rules_v1.md]].

**Measured** (2026-10-22, on a copy of the example, the AI program of this
install): the plan said about 31 thousand tokens (between 16 and 62 thousand,
the default rate, with no measured runs in that copy) and about 2 minutes; the
run used 23,310 tokens and took 41 seconds.

## The example collection

`sandboxes/hello_brain`, improved in place for the tour, never deleted. Its
subject is Kovium itself; `example.json` (its manifest) lists:

- `originals`: each original its notes are built from, its kind, and the
  header `fields` its note records (facts the original states, written in by
  the build). Three are kept in the collection's own folder (`path`, under
  `sources/raw/`), one of them a copy of the user guide frozen at release
  time (`frozen_from: docs/user_guide_v1.md`). One is left where it lies, in a
  folder the collection points to by a name (`root_path:
  library:decision-every-answer-shows-its-sources.md`; `config/sources.json`
  declares `library` relative to the collection, so it travels with it), under
  `source_retention: reference`;
- `sample`: the page waiting in `inbox/` for the tour, written as a note by
  hand (documentation sections, a `## Links` section);
- `journey` (Track BA3): one document at every stop of its way, for the tour's
  second step: its original, its plain text copy
  (`sources/converted_markdown/documentation/`), its copy sorted into the
  layout of its kind (`sources/templated_markdown/documentation/`) and its
  note. The two copies between are made by code,
  `scripts/build_example_collection.py --layers` (free): the original run
  through the pipeline's free reading in a temporary copy, with a tripwire on
  the agent, the places the filled-in copy records spelled as the note spells
  them and its lone placeholder lines taken out. The filled-in copy is
  therefore plainer than the one the AI filled when the note was built, and
  the tour says so. `--check` reports them (`layers.state`), and a test fails
  when the original changed and they were not made again;
- `practice` (Track BA4, BA5): `answers`, the practice night's prepared
  answers (`practice/prepared_night.json`, written by hand in the shape of the
  AI's concept answer: `ready` names the page written from them on opening,
  `waiting` the one the practice night fills), and `both_ways`, the two small
  notes of the exercise (`practice/`, written by hand).

Its notes are of three kinds (documentation, a meeting note, a decision
record). Its settings have code assemble its concept pages
(`config/concepts.json`, `synthesis: deterministic`), so opening it and adding
the sample free never call the AI.

**Built by the maintainers, never by a user**:
`scripts/build_example_collection.py --check` (free) lists each original and
whether its note was built from its bytes as they are now (`current`,
`changed`, `no_note`, `no_original`); `--plan` gives the free plan; `--build`
builds what is stale with the install's AI program on the maintainer's
subscription (the code-only path when none is ready, said in the output), then
`--finish` writes the manifest's header facts and takes out the reading's lone
`TODO` placeholder lines and names the notes without the pipeline's
`__templated` suffix, links following (also free, and run by `--build`).
Since fix round R4 of Track BA it also spells every place a note's header
records (`raw_source_path`, `converted_markdown_path`, `source`, the earlier
places listed) relative to the collection (`sources/raw/a.md`), not to the
checkout (`sandboxes/hello_brain/sources/raw/a.md`): the example is copied,
and in a copy the checkout's spelling named the shipped folder, so a newcomer
following it landed in a folder nothing explained. `resolve_source` reads the
collection's spelling first, so the working copy's notes name its own files; a
named folder's form (`library:a.md`) is left as it is.
Measured on 2026-10-22: four notes, 106,788 tokens; the user guide's note
built again after the guide gained its how-tos, 37,689, and again after fix round 2, 49,979;
after Tracks AH and AG2, 55,709 and 58,637.

**The guide is a frozen copy (Track AG2).** The live `docs/user_guide_v1.md`
keeps changing with the project, and every change made the guard fail and cost
a rebuild of its note (above). The example's guide note is now built from
`sources/raw/user_guide_v1.md`, a byte-identical copy taken in with
`source_pipeline/reconcile.take_in_copy` (the notes repointed as a rename
repoints them, the old path kept in `previous_source_paths`; nothing built
again, nothing spent). Editing the live guide changes nothing the example
ships. `--check` reports the copy against the live guide as information
(`guide_copy.state`: `same` or `differs`), never as a failure. At release time
a maintainer runs `--refresh-guide` (copies the live guide in, free), then
`--plan` and `--build` for the guide's note (spends tokens). `help` reads the
live guide directly, not the example's copy.

**Derived by code, and read when current (Track AU3)**: opening it
(`second_brain/welcome/example.py` `build`, in the working copy) reads every
note into the graph as `scripts/rebuild_graph.py` does, has code write the
concept pages, and draws one map per note; it records the notes' fingerprints
(`data/staging/example_built.json`) and builds again only when one changed. It
writes no note: the shipped notes are byte for byte as committed after opening
(tested). Its maps (`sources/wiki/maps/`), concept pages
(`sources/wiki/concepts/`) and the tour's sample note are ignored by git. What
that first opening would build is made at release (`scripts/build_example_collection.py
--derive`, free) and ships in the example's `derived/` folder
(`second_brain/storage/derived_snapshot.py`): the graph's rows, the concept
pages, the maps and the concept layer's state, with the fingerprints of the
notes and of the code they were derived from. An opening reads it instead of
building while every note, the code and the data layout are the ones recorded
(`opened.how` is `read`, else `built`); a derived file changed in place is
never overwritten (the collection is built instead). Any change to the code
makes it stale until the next `--derive`; the example then builds as before.
`--check` reports it (`derived.current`).

**A picture of the tour's map (Track BD5)**: `pictures/` holds a PNG of the
first note's map, drawn by code from the map `derived/` holds
(`second_brain/canvas/picture.py`: any canvas, by its geometry, its colours,
its groups, its cards and its edges; a card longer than its box cut at a whole
word with an ellipsis), and `pictures/pictures.json`, naming that map and its
fingerprint when the picture was drawn. `--derive` draws it again with the
map, and `--picture` draws it alone. Drawing needs Pillow, an optional extra
(`pip install -e '.[pictures]'`) that only the maintainers' build uses; a
person's install never needs it, since the picture ships made. `pictures/` is
among what ships, so it is copied into the working copy. A test fails when the
picture's record names a map other than the one that ships.
`.gitattributes` keeps the example's files and the user guide at LF on every
platform, so a fingerprint taken on one platform holds on another.

**The rebuilt-note guard**: `tests/integration/test_example_collection.py`
fails when an original changed and its note was not built again, and when the
example stops opening cleanly or its notes stop passing lint. Editing the live
user guide no longer does (the copy above); refreshing the copy does, until its
note is built again.

## The nightly AI question (Track AX)

Decided by the owner on 2026-10-23. A person has no idea what a token cap is,
so the nightly steps that use the AI are asked about in plain words, by level,
**once per install**: in the welcome once the person's first collection exists
(the `GET /welcome` of that collection, before the offer to add documents),
and in the reply of `POST /collections` (`ask`), so a person who skipped the
welcome is asked too. The answer is that collection's level and the install's
(`maintenance/nightly_ai_choice.json` under the install root): a collection
made later starts at it and its reply says so ("Its nightly AI steps are at
Medium (up to 150,000 tokens a night), as you chose before. Say another level
to change it."). Not asked in the example, nor where someone already chose a
level for the collection, nor of an agent that may not change its settings.

Step 1, `yes` recommended:

> Should the nightly tidy also use the AI? It keeps concept pages up to date,
> merges look-alike concepts, judges new label words and labels notes that have
> none. This uses extra tokens from your AI subscription, and only on nights
> with work to do.

- yes: you choose how much it may use at most: light, medium or heavy
- no: nothing uses the AI at night, and the free tidy still runs every night

Step 2 (the reply to `yes`), `medium` recommended:

> How much may the nightly AI steps use at most in one night? It is a most,
> used only when there is work to do, not spent every night.

- medium: Medium (up to 150,000 tokens a night): enough for a usual night after documents were added
- light: Light (up to 50,000 tokens a night): the smallest jobs each night, and bigger ones wait for later nights
- heavy: Heavy (up to 400,000 tokens a night): catches up fastest after many documents were added

Each `on` is `POST /welcome/nightly` with the answer (MCP: `nightly_ai`). A
level is written through the settings path `change_setting` uses (its preview,
then its apply), so `config/maintenance.json` holds `ai_tokens_per_night` and
`ai_level`; `check_setup` reads it back as "The steps of the nightly tidy that
use the AI are at Medium (up to 150,000 tokens a night). It is a most, used
only when there is work to do." Later changes are `change_setting` "Nightly AI
level" with a level's name or a number (one matching no level is shown as
"Custom (up to N tokens a night)").

The numbers live in one place, the option catalogue (`nightly_ai_limit.levels`
in `second_brain/catalogue/options_v1.json`), with `level_when_unset` (medium),
the level a schedule written before the levels is read as. How they were
chosen is in the journal (Track AX).

**The example's notice (owner's decision d).** The example keeps its nightly
tidy off and never uses the AI at night. The first time a person touches it
in an install (opening it, asking it, or the tour; recorded in
`maintenance/example_nightly_notice.json` under the install root), four
sentences are added to `say`:

> Your own collections look after themselves every night. In this example that
> is switched off.
> At night the free part fixes small things: new label words, renamed files,
> missing search entries and unread equations.
> If you allow it, the AI part also keeps concept pages up to date and labels
> new notes, using extra tokens from your AI subscription.
> For a collection of your own, say "turn the nightly tidy on" or "off", and
> "nightly AI light", "medium", "heavy" or "off".

and an offer to try the switch here (`ask`, or `nightly_ask` beside an ask the
reply already has): "Want to try those words here, the ones that turn the
nightly tidy on and off? In the example they only show what would happen:
nothing starts at night and the AI is never used." (Fix round R6 of Track BA:
it named "the switch" before any switch was introduced where it was shown.)
Trying it
(`POST /welcome/nightly` with `example: true`, `on` then `off`) shows the
sentence a person's own collection would show, and nothing else: the
example's schedule is not written, this computer's scheduler is not asked, and
a level is answered with "The example never uses the AI at night, so there is
no level to choose here." Chosen over a real switch in the example because a
real one would leave a nightly job on the person's computer for a collection
that is only a demonstration.

## The full manual (the help collection, Track AG6)

Kovium's own current documentation, as a collection anyone can ask
(D3a). It is **a repository of its own** (`kovium_help`), not part of this one,
so the download stays light; it is **offered, never required**; and **nothing
in this repository depends on it**: every test here passes without a copy.

**What it holds.** One note per current document: the contracts and guides
`docs/README.md` lists, the specification, the README, the architecture
overview, the four skills of the Claude Code framework and the option
catalogue written out as a document. Never the history (plans, results,
records), which describes what was intended or true once.

**Built by the maintainers, at no cost.** `scripts/build_help_collection.py
--out <folder>` builds it from this checkout; no agent is called and no token
is spent, by construction. Each document is taken in as a note **written by
hand** (its text unchanged under its own headings, the header facts it states
about itself, `type: documentation`), read by code the way any hand-written
note is. A link from one document to another becomes a note link (`[[note|words]]`);
a link to anything not in the collection keeps only its words; fenced code is
untouched. The graph, the concept pages and the maps are derived once by the
build, by code in a temporary copy (`example.derive`), and kept in the
collection's `derived/` folder, so a person's install reads them on the first
opening instead of building them (Track AU3: measured on a copy of this
manual, about two minutes became a few seconds); a copy whose notes or whose
Kovium code differ from what was derived is built on opening, as for
the example. The build derives again only when the notes or the code changed,
and `manifest.size_mb` counts `derived/`. A tripwire on
the brain keeper fails the build if anything calls it (and its test installs
the same one); nothing was left out for needing the AI. Run again, it changes
nothing that did not change (it is idempotent, byte for byte). `--check`
(free) lists the documents changed, added or left since the last build and
whether the platform version moved (exit 1 unless current). `--record` writes
the built size into `config/help.json` (below).

**The manifest** (`manifest.json` at the collection's root) records
`describes.platform_version` (the version of Kovium it was built from),
`describes.schema_version`, each source document's `sha256` and its note, the
note count, and the size (`size_bytes`, and `size_mb` rounded up to a tenth:
the notes and settings, which is what fetching costs). It carries no date, so
a build is reproducible.

**Where it is fetched from is a setting.** `config/help.json`, the install's
own (else the one this checkout ships): `repository` (a git address, or a
local path), `ref` (a branch or tag; empty means the tag `v<platform version>`
when the repository has one, else its newest state) and `size_mb`. The shipped
`repository` is the address of the published manual (version tag `v0.1.0`; the
address itself lives only in `config/help.json`); **an install whose config
holds the placeholder `OWNER` account, or none, offers nothing and
`POST /welcome/manual` is refused `manual_not_set` (404)**, so no offer is made
that could not be done. `size_mb` is the measured figure `--record` wrote,
so the offer says "about N MB" from the manifest, never a guess; with none
recorded the offer says no figure.

**Offered twice.** At the tour's end, after the sample was added (step
`add`): the response's `ask` is the offer (`tour.offer_manual`, recommended
`yes`, `on.yes` is `POST /welcome/manual`), the earlier "put the example back"
offer moving to `also`. And in `GET /help`, when the manual is not fetched and
an address is set: a line (`help.manual_offer`) and the same `ask`. "No" ends
it; asking for the manual later does the same. With no address set the tour
keeps its old line (`tour.manual_pending`) and `ask` stays the reset offer.

**Fetching** (`POST /welcome/manual`, tool `get_manual`, free, nothing typed):
a `git clone --depth 1` of `repository` into `sandboxes/kovium_help` under the
install's own folder for collections (never inside a tracked folder), made
beside its place and moved in only when whole, so a failed fetch changes
nothing. It is then registered under an agent of its own (`kovium-help`),
never bound to the person's connection, and built by code on opening (graph,
concept pages, maps; seconds, no tokens). A current copy is left alone
(`fetched: false`, `manual.already`). Refusals, each with a plain sentence from
the catalogue: no address (`manual_not_set`, 404), no network
(`manual_no_network`, 502: git could not reach the host, or took more than ten
minutes), no `git` on the computer (`manual_no_git`, 424), anything else
(`manual_fetch_failed`, 502). A download of an archive instead of a clone is
not built.

**Never silently stale.** A copy whose manifest describes another platform
version is reported in plain words (`manual.other_version`, naming both
versions) by the fetch and by `help`; `help` then **answers from the guide and
catalogue that ship with this version**, not from the copy, adds that sentence
and an `ask` whose yes is `POST /welcome/manual` with `update: true`, which
replaces the copy with the one matching this version (the repository's tag
`v<version>` when it has one). An incomplete copy is reported the same way
(`manual.damaged`).

**A reading connection is never offered the fetch.** `get_manual` writes into the install, so a connection that only reads does not list it (the read-only invariant of the MCP server), and `GET /help` for an agent that does not manage its collection says in words (`help.manual_elsewhere`) that the manual can be fetched from a start without a collection or by an assistant that manages one, with no `ask`: never a dead end.

**`help` with the manual.** For a copy that describes this version, the
question goes to the governed answer route over the manual (`POST
/agent/knowledge/answer`, no model call, `synthesize` and `judge_relevance`
off, `widen: cache_only`, read only), under the manual's own workspace, never
the person's. `answer` lists at most four items as the example does, each
`source` adding `document`, the document of the checkout the note was made
from (`docs/welcome_v1.md`). When the manual does not cover the question, or
cannot be read, the guide and catalogue answer as before. Its answers are about
the software and are never evidence in an answer about the person's documents;
a person cannot add to it by accident, since adding goes to the collection of
the connection in use. When the guide also answers the question, its passages
come beside the manual's, as `in_the_guide`: the manual goes further, the guide
is tuned to plain how-tos.

**Measured** (2026-10-23, the real manual of this checkout, 46 notes and 1.3 MB,
on the machine that built it, the AI never called): fetching it by `git clone`
took under a second from a local copy; the first opening, which builds the
graph (about 10 thousand nodes) and 247 concept pages by code, took about 37
seconds; each question put to it through the governed answer route took about 22
seconds, most of it ranking the whole graph (four passes for a question of
several words) and computing graph summaries that do not depend on the question.
That is the platform's retrieval cost at this size, not a cost of the manual: a
faster path for a collection of ten thousand nodes is a follow-up. Answers are
the ordinary ranked items, not tuned to the manual (it is not a tuning target),
so a question the guide covers is often better answered by the guide's passage,
which is why both are returned.

**Not a tuning target.** It is a corpus this project owns, so it may be one more
evaluation collection beside the synthetic fixtures, never instead of them and
never what ranking is tuned to. The tests build a synthetic one
(`tests/fixtures/help_repository.py`) and fetch it from a local bare git
repository standing in for the remote.

## Over MCP: welcome mode

The MCP server ([[mcp-server-v1|mcp_server_v1.md]]) started without
`--workspace` (a fresh install's entry names none) is in **welcome mode**: no
collection, no registry checked, the tools `welcome`, `help`, `get_manual`,
`try_example` and `tour`, and since Track AG2 `create_collection`, `open_collection` (which
bind this connection to the collection they make or open) and `check_setup`
([[collections-v1|collections_v1.md]]), and nothing else; no grant but the
`manage` grant those two need, held over a local transport only, instructions that tell
the agent to call `welcome` for a newcomer and `help` for a question about the
software, and an audit of every call in the instance root's
`logs/mcp_welcome_audit.jsonl` (metadata only). A connection bound to a
collection offers `welcome` and `help` too, never `try_example` or `tour`: a
collection is a boundary (D2), and the example lives on the connection with
none. Each tool is the route above (rule 1 of the MCP plan); the calls a
response names (`ask.on`, `next`, `ask_with`) carry their tool call in the
compact view, so an agent follows them without knowing a route.

| tool | route | connection | read only | cost |
|---|---|---|---|---|
| `welcome` | `GET /welcome` | any | yes | free |
| `help` | `GET /help` | any | yes | free |
| `get_manual` | `POST /welcome/manual` | welcome mode, and a connection whose agent may manage (Track AG6); `manage` grant, never a reading connection | no: fetches the manual into the install's own folder | free |
| `try_example` | `POST /welcome/example` | welcome mode | no: builds the example's derived files | free |
| `tour` | `POST /welcome/tour` | welcome mode | no: adds or removes the example's sample | free, except `add: ai` with an approval |
| `create_collection`, `open_collection` | `POST /collections`, `POST /collections/open` | any (Track AG2) | no: makes or opens a collection, binds this connection; the program is told its tool list changed, and the answer's last line says to start a new conversation where the collection's tools do not show (Track AY3) | free |
| `check_setup` | `GET /collections/setup` | any (Track AG2) | yes | free |

## A kind for documentation

Found while sizing the help collection (D3c): no kind fitted a manual, and
guessing a file's kind from its headings read documentation as images, chats,
papers or decisions. Both are fixed for every install, through the type
profiles:

- **`documentation`** (template `format-templates/documentation.md`, profile
  in `second_brain/notes/contract.py`, prompt
  `prompts/documentation_remote_template_authoritative.md`, map
  `scripts/build_documentation_maps.py`): a page of documentation kept as a
  file. Sections `Purpose`, `How it works`, `How to use it`, `Settings and
  options` (both of these may be empty), `Known limits` (a limitation role);
  fields `product`, `product_version`, `doc_kind`, `audience`, `url`,
  `last_updated`. Its names: documentation, docs, manual, user guide,
  how-to, handbook, technical documentation, technical reference,
  specification, readme, instructions. `webpage` stays the kind for a page of
  the open web (its "documentation" names moved here).
- **Kind guessing** (`second_brain/source_pipeline/file_kinds.py`): a file's
  heading counts for a kind's section only when it holds every word of four
  letters or more of that section (two forms of one word are one word: their
  shared beginning is at least three quarters of the shorter), and a kind's
  headings count only when they are at least a quarter of the file's headings.
  Measured on this checkout's 39 current documents, with a default kind of
  `blog-post`: 13 were read as another kind before, none after. Files written
  in a kind's own shape are still recognised (the existing tests unchanged).

## The newcomer test, as measured here (S11)

Scripted (`tests/integration/test_welcome_mcp.py`, the agent following only
what the responses name, on a fresh install holding the example), on this
machine, 2026-10-22: the first example question answered with its sources
0.035 minutes (about 2 seconds) after the first call; commands typed by the
person: 0; questions put to the person before that answer: 1 (see the
example); tour offers: 2 (the tour, then how to add the page); tokens spent
before an approval: 0 (a tripwire on the agent); the sample added free, its
map drawn and a question about it answered 0.058 minutes after the first call.
Not measured here: a person (Codex and OpenClaw were measured by AG7, in
[[install-and-caller-agent-v1|install_and_caller_agent_v1.md]]). The install step on
a simulated clean machine was measured by AG4
([[install-and-caller-agent-v1|install_and_caller_agent_v1.md]]).

## Limitations

- Making a collection of one's own from the conversation is built since AG2
  (`create_collection`, [[collections-v1|collections_v1.md]]); adding
  documents to it from the conversation since AG3 (`plan_adding`,
  [[adding-documents-v1|adding_documents_v1.md]]).
- The tour's last offer is the full manual (AG6), one question that "no" ends; when this install's
  `config/help.json` holds the placeholder address (the shipped one is the published manual) nothing is offered and the tour keeps a line saying it is
  not ready. After it, the tour's last line says that once the person has a collection of their own they can ask
  for their own Kovium agent for it (AG4, `make_agent`): a sentence, not a question, since the agent belongs
  to a collection and the tour runs before there is one.
- The manual is fetched with `git`, not downloaded as an archive; with no `git` it says so. It is a copy of one
  published state: a person gets a newer one by asking again (`update`), never by itself.
- The help collection's size is measured from its manifest and recorded in `config/help.json` by the build; a
  checkout that was published before a build was recorded offers the manual without a figure.
- `help` is word matching over one language's texts: a question worded far
  from the guide finds nothing or something close; the agent may ask again in
  other words.
- The example's suggested questions are chosen to be answered well by the
  example under today's ranking; the answer test fails if a ranking change
  stops one being answered.
- Obsidian is not opened by the agent: the folder to open as a vault, each
  file's full place and an `obsidian://open?path=` address are given; that
  address opens a file only once its folder is a vault in Obsidian, and
  whether it can open a folder not yet a vault is unchecked. In the vault an
  original and its note may share a name (`sources/raw/x.md`,
  `sources/wiki/x.md`): Obsidian resolves a note's link to the note beside it
  in the same folder, by its own rule, not ours.
- The tour's places are full paths on purpose (BA1); the caller agent is told
  to show them, and a program that cannot read files describes nothing. The
  one check-in left (the way to look) is a question to the person, put by the
  caller agent, which the tour cannot enforce beyond its words and
  `settled_by: user`. Likewise the way on and "ask nothing of your own" are
  instructions to the caller agent (Track BD1, BD6): the tour says them in
  every reply and in the connection's instructions, and cannot make an agent
  follow them.
- The map's picture (BD5) is of the first note's map only, drawn when the
  example is derived; a person's own collection has no pictures of its maps.
  Whether the picture reaches the person depends on the caller's program: a
  chat app through OpenClaw can send it, a program that cannot send files can
  only give its place. Its text is drawn in Pillow's own font, so a script
  outside Latin may not show in it (the example is in English).
- The notes step's reply (BD7) is written in advance for its one question; the
  example's answers outside the tour say their parts are the machinery and the
  caller agent writes the reply, which the tour cannot check.
- The practice night's answers are prepared by hand for this example's pages;
  a maintainer who rebuilds a note so that it no longer names the page's
  concept leaves the answer unused (the example then shows two drafts, and a
  test fails). The example's other pages stay drafts by code: only one ready
  page is shown, on purpose.
- The outside folder of the exercise sits beside the working copy in the
  install's place for collections, not in a folder of the person's own; their
  own files are only touched when they add them themselves.
