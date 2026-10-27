---
type: "documentation"
title: "User guide v1"
description: "A guide for a person, not a contract. Its words follow plain_language_rules_v1.md and are checked by tests/contracts/test_option_catalogue.py. Each choice named here has its four lines in the option catalogue."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "guide"
audience: "a person setting up or using Kovium"
doc_path: "docs/user_guide_v1.md"
doc_sha256: "71e330daa4c324b56c3092d86321ee27d1b012581c3e18326844daab1d6a32bc"
---

# User guide v1

<!--
A guide for a person, not a contract. Its words follow plain_language_rules_v1.md and are checked by
tests/contracts/test_option_catalogue.py. Each choice named here has its four lines in the option catalogue.
-->
# Kovium: a guide for a person

Kovium answers questions from your own documents, and shows the exact place each answer comes from. You talk to it
through the AI program you already use. This guide says what it does, what it costs, and what you can change.

## Getting started

To install Kovium, run its setup once from its folder. It connects Kovium to the AI programs it finds on
your computer, and to Claude Code as a plugin, which also brings its skills.

Say "help me start with Kovium" to your AI program. It first shows Kovium on a small example collection.
That takes about a minute and uses no tokens. You never type a command for it.

Then it offers a short tour of what happens to a document you add: nine short steps, each with something to try. You can stop at any step.
Each step says how far along you are and what kind of step it is, and ends with what to say for the next one.
The tour first shows how to look at a collection: in Obsidian, a free program for notes, in a text editor, or in the chat.
You find one document at every stop of its way. You keep one small note as a copy, point to another, and run a practice night for free.
At the end you add one waiting page yourself and watch its note, its map and its links appear.
Then it offers the full manual, which answers any question about Kovium.
The example and its tour are for a first start, before you have a collection of your own.
For a start step by step, with the words to type in each AI program, open the getting started guide in Kovium's folder.

- The example collection: a few short pages about Kovium, ready to ask, with one page waiting for you to add.
- Help at any time: ask how anything works, and the answer comes from this guide, with the section it comes from.

## What you need to run Kovium

You need an AI program you already use, signed in with a subscription you already pay for.
It is the kind of AI assistant that works on your own computer, such as an AI coding assistant. Claude Code, Codex and OpenClaw work today.
You need no pay-per-use account and no extra service. Kovium itself runs on your own computer.

## What stays on your computer

Your documents, your notes and your collections stay on your computer. Kovium never shares your files with other people or puts them online.
The only text that leaves your computer is what your AI program reads to do its work, sent to the service of your subscription.

## What a collection is

A collection is a set of documents you ask questions about. It lives in one folder on your computer.
Kovium keeps your documents there, with a note for each one, and everything it learns from them.

You can copy that folder to another computer and open it there. Nothing needs to be added again.
Your own originals are never deleted or overwritten without asking you first.

## Your own collection

To make one, tell your AI program in a few words what it will hold. You are then asked where it goes:
the folder you work in now, a folder you name, or the default place. The default place is inside
Kovium's own folder, where an update or a move of the program can put a collection at risk.
From then on that conversation uses it.

- Where your collection is kept: the folder you work in now, one you name, or the default place, asked once.

To open a collection you already have, for example a copy from another computer, say which folder it is in.
If some folders it points to are not on this computer, you are told which, and you say where they are here.

Each collection has a connection of its own in your AI program. To use a collection from another AI program,
ask to connect that program to it. It may only read, until you say it may do more.

You decide what each AI program may do with a collection. It may only read it, or also suggest changes you approve,
or also add documents and change its choices. The change is shown to you as one sentence before it applies.
An AI program can never give itself more than you allowed.

To see what works on this computer, and what is off or missing, ask your AI program to check the setup.

## Connecting one agent to a collection

To connect one agent to a collection you already have, say to it "connect yourself to my thesis collection".
It finds the collection by its name among the ones on this computer, and you can ask it which ones there are.
First you see a plan. On your yes the agent joins under its own name, and only that agent gets the collection.
It also gets its own instructions for that collection, which tell it to use the collection when you name it.
Where the connection is kept depends on the AI program:

- Claude Code keeps it for the one project the agent works in.
- Codex keeps it in the project's own settings, but only in a project you trust in Codex. In any other project it goes where all your projects see it, and you are told.
- OpenClaw shows it to that one agent only. This hides it from your other agents there, but it is not a lock.
- Cursor keeps it in the project's own settings.

The connection takes effect in a new conversation, not the one where you asked. Your agent says so, before and after.
AI programs load their connections only when a conversation starts. So start a new conversation with that agent.
If no AI program manages the collection yet, the agent that joins will manage it.
Say so if it should only read. To undo it, say "disconnect from my thesis collection".
Its connection and its instructions go, and the collection itself stays as it was.
The conversation where you asked may still show the collection's tools until it ends; a new one does not.
Even with access keys on, any AI program on this computer can see the names of your collections, but not their contents.

## Access keys

An access key is a long secret word that one assistant holds for one collection. It proves which assistant is asking.
Each collection has a choice, Access keys for AI programs, and it is off until you turn it on.
While it is off, everything works as before: an assistant says who it is, and Kovium believes it.
When it is on, an assistant without its own key for that collection is turned away, and told why in plain words.
What an assistant with a key may do is still what you allowed it: only read, also suggest changes, or more.

To give your assistant its key, ask it to connect itself to the collection.
When the keys are on, it gets its own key on your yes, kept in a small file in its own folder.
Its instructions tell it to use that file at the start of every conversation, so you never type the key.
You can also make a key yourself with the access key script, for one assistant and one collection, with an end date if you like.
An assistant that manages the collection and may give access can make a key for another assistant, shown to you first as one sentence.

To turn the keys on, first give each of your assistants its key, then change the choice. An assistant that has no key yet is asked to get one first.
A key can be taken back at any time, and from the next question on it no longer works.
You can list the keys of a collection, with the assistant each one is for and the last time it was used.
Copying a collection's folder never copies a usable key: the folder keeps only a fingerprint of each one.

Your own web page asks for a key too. When Kovium starts, it prints a link that signs your browser in.
The link works once, for ten minutes. If you start Kovium again, use the new link it prints.

What a key does not do: on one computer under one login, an assistant that can read any file can read another assistant's key file.
Keys stop mistakes, assistants that cannot read your files, and callers from other computers.
They do not stop a determined assistant on this computer that can run any command.
Real separation needs a separate user account, or a container, for each assistant.

## When your agent does not use your collection

If your agent ignores your collection, start a new conversation first. Most AI programs only see a new connection in a new conversation.
Some programs show an agent its connected tools only when it looks for them, so it may answer from its own memory instead.
The instructions Kovium writes for the agent tell it to look first, whenever you name the collection.
If your agent has no such instructions yet, ask it to connect itself to the collection, and it adds them.

- Claude Code: the connection is kept for the project where the agent joined, so start Claude Code in that folder.
- Codex: a project's own settings count only in a project you trust in Codex.
- OpenClaw: the connection is shown to the agent that joined, so ask that agent, not another one.

## Adding documents

You say what to add: files, a folder, or links. Kovium first makes a plan, and the plan costs nothing.
It tells you what it found, what it will skip and why, and what it expects to use.

You are asked one question about money, once: how many tokens of your subscription one addition may use without asking you again.
A token is a small piece of text that the AI reads or writes. Your answer is kept until you change it.
If you are unsure, start with a few documents and see the real cost.

When the work is done you get a short report of at most six lines. It says what was added, what was skipped, what could not be added
even after trying again, and what it used. If something needs you, the last line says so. Otherwise it says that nothing does.

- Token limit for adding documents: the most one addition may use before it asks you.
- Asking before anything is spent: nothing that uses tokens runs without a plan you were shown first.
- Other versions of one document: a draft and its final form are kept and linked, so nothing is lost.
- Splitting long documents: a book is cut into parts by its own contents, and it is read whole if the parts are unclear.
- Keeping copies of your originals: Kovium keeps its own copy of each file, or only points to where it lies.
  The first plan that brings files from outside the collection asks you once. Keeping copies is recommended,
  so the collection holds everything and can be moved or backed up as one folder.
- Clean names for notes you already have: notes added before names were made clean get their title as their name, once.
- Named folders for your originals: a short name for a folder you keep elsewhere, so notes survive a move.
- Kind of note when none is clear: what a document is taken to be when nothing in it says.

## Adding files, folders and books

Tell your AI program what to add: one file, several files, a whole folder, or a link to a web page or a video.
A book or another long document is added the same way, and Kovium cuts it into its chapters for you.
First you see the plan, with what it will cost. Nothing is added until you say yes, or until it fits the limit you set.
The plan also says what your files need, such as the better PDF reader. A choice that cannot wait is asked before anything is added.
Adding goes on in the background, so the conversation carries on. At the end you get the short report.
Each note is named after its document's title, cut after a whole word, and each document gets its map.
Files you already added, even under another name, are skipped.

## Adding your own notes from a conversation

You can keep text from the conversation as a note of your collection: something you wrote, pasted or said.
Ask your AI program to add it as a note, and give it a short title.
First you see what will be added. On your yes, the note is kept in your collection word for word, in any language.
A note you write costs no tokens to take in, because it is read without the AI. Questions can find it at once.
To add your documents and notes, the AI program needs to be allowed to add to that collection.

## Only the first tools after making a collection

When you create or open a collection, the conversation gets its tools for asking, adding documents and adding notes.
Your AI program is told the moment the collection is ready. Some AI programs only look again in a new conversation.
The first tools are the welcome, help, getting the manual, trying the example, the tour, creating or opening a collection, and checking the setup.
If your AI program still has only these after you created a collection, start a new conversation.
That is also why, right after creating a collection, you may not be able to add documents yet.
The new conversation opens your collection by itself, with all its tools for adding your documents and notes.
Nothing is lost, and nothing has to be set up again.

## Reading your documents

Most files are read at once. A few kinds need a small program on your computer, and Kovium tells you which ones
your own files need. It looks at your files before it says so.

Some choices cannot wait. The better PDF reader is the one. If you add PDFs with tables first and install it later,
those PDFs must be read again, at a cost in time and tokens. So it is asked before the first run.

- Better PDF reader: reads columns, tables and formulas well. It is a download of about 2 GB, and it uses no tokens. Once it is installed, it turns itself on.
- Reader for Word files, web pages and e-books: needed only if you add those kinds of files.
- Equations in PDFs: equations that cannot be read are marked, and are read at night on your computer. Any document can also be read at once, without being added again.
- Reading scanned pages: reads text out of pages that are only pictures. It reads only the files that have no text, once the better PDF reader and a text-recognition program are installed.
- Reading pictures: the AI describes the figures in your documents. It uses tokens, so it is offered with an estimate.
- Pictures on web pages: whether pictures on added pages are kept with the page.
- Repairing lost letters: letters a PDF reader lost are put back when the document itself shows the right word.
- Videos added from a link: Kovium reads the captions the site shows, and never the video file.

## Looking at your collection in Obsidian

Obsidian is a free program for reading folders of notes. Open your collection's folder in it as a vault.
A tidy view in Obsidian is on in every collection. It hides the files Kovium keeps for upkeep, such as earlier versions of concept pages, so you see your notes.
In the map of links, each kind of note has its own colour, and arrows show which note links to which.
Links to notes that do not exist yet are left out of the map.
A kind of note that nothing named before gets a colour of its own, and it keeps that colour.
A colour, a filter or a hidden folder you set in Obsidian yourself is kept. Your zoom and your open notes are never touched.
It is free and uses no tokens. Ask your AI program to turn it off if you prefer Obsidian as it comes; what was already set stays.

## Asking

To ask a question, say it to your AI program in your own words. Every answer shows where it comes from, so you can open the original.
When your collection does not say, the answer says that, and does not guess.

Answers can carry a short warning in plain words, for example that part of the answer rests on notes waiting for your review.
Each warning also says what to do next.

- Search by meaning: finds a note that words an idea differently from your question. It needs a small free program on your computer.
- Matching your wording: your question is matched to the labels your collection already uses. It uses a little of your subscription.
- Remembering written answers: the same question on unchanged notes is answered again without the AI.
- A fuller look for thin answers: a written answer may read more when its first look was not enough.
- A second check of numbers: numbers that differ between a note and its original are looked at twice.
- What connected AI programs may do: by default a program may only read your collection.
- Name of your collection: a short name and a sentence, shown to every AI program that connects.

## The full manual

Kovium can fetch its own manual as a collection you can ask. It holds the current documents about Kovium, and every answer shows the document it comes from.
The tour offers it at the end, and you can ask for it at any time. It is free, and nothing has to be built or paid for.
It is kept apart from your own collections: it answers questions about Kovium, and never about your documents.
Until you have it, your questions about Kovium are answered from this guide.
If the manual you have was made for another version of Kovium, you are told, and asking for it again fetches the matching one.
Without the internet it cannot be fetched, and everything else works as before.

## Your own Kovium agent

If your AI program lets you add agents of your own, Kovium can add one for a collection, which answers the way you like.
You are asked three short questions: simple or technical answers, short or detailed ones, and the tone and language.
Then one free line about anything else, and no is fine. Each question has a recommended answer.
The agent is added to your project beside the agents you already have, which stay exactly as they are.
Some programs, such as Codex, hand work to an agent only when you ask for it by name, so ask for it that way.
A program with no agents of its own, such as OpenClaw, gets it as a part of the project's instructions, below what is already there.
The rest of those instructions is left exactly as you wrote it.
You see what will be added, and where, before it is added.

Each collection has its own agent and its own style, kept with the collection, so every AI program you connect uses it.
To change the style, just say so, for example "be more technical" or "shorter, please".
Whatever the style, every answer keeps its sources, says when the collection does not say, and asks before anything is spent.

- Simple or technical answers: everyday words only, or the program's own terms where they help.
- Short or detailed answers: the answer in a few sentences, or with the detail and the evidence for each part.
- Tone and language of answers: in your own words; by default plain and friendly, in the language you write in.
- Anything else about how answers sound: one line in your own words.

## Concept pages and your own notes

When several documents share an idea, that idea gets a page of its own. It says what each document says about it, and where they disagree.
When two notes disagree, nothing is overwritten: the concept page shows what each of them says, side by side.
These pages are kept up to date as documents arrive. Your own notes and ideas are kept apart from the evidence and never counted as evidence.

Every note that Kovium writes ends its content with a section called My notes, and a short hint line under its title.
Write there whatever you like beside a document: a doubt, a question for a colleague, a thought for later.
Kovium never rewrites that section. Your words stay exactly as you wrote them when the document is added again,
read again, renamed or tidied. If the document is removed, your words are kept in a separate file, and you are told where.
Your words can help a question find the note. An answer shows them apart, as yours, and never as something the document says.

- How concept pages are written: the AI writes them, or plain code assembles them for free.
- Merging look-alike concepts: obvious duplicates are merged and every merge can be undone. The rest are suggested to you.
- Your own notes and ideas on concept pages: whether they appear there.
- Your own words section for notes you already have: a one-time step that adds the empty section to older notes. It shows how many first, and takes a backup.
- The name of your own words section: the title can be set in the language you write in. Words under the old title are still kept.
- Hints about your collection for the AI: a short description of your collection, kept off until you approve one.
- Suggestions of what to read next: compares what your documents say about topics you name. It is off until you ask.
- Letters with marks in file names: how such letters are spelled in the names Kovium makes.

## Words your own documents use

Kovium finds the parts of a book and checks numbers with short lists of words.
The lists hold many common words already, and you can add the ones your own documents use.
Ask your AI program to add them, and it shows you the change before it applies.

- Words for a table of contents: the heading that marks the contents of a book.
- Numbers written as words: how chapters are numbered when they are written out.
- Words for measured quantities: the word for a quantity and its unit, so a number can be checked.

## What happens at night

Every collection you make tidies itself each night, at no cost, on the computer that made it. Nothing needs to be left running.
It adds new label words, fixes notes after a rename, fills in missing search entries, puts back lost letters and reads equations left unread.
On a folder shared between computers, one computer does the tidying.
Some steps use the AI: they keep concept pages up to date, merge look-alike concepts, judge new label words and label notes that have none.
When you first make a collection, you are asked once whether they may run, and how much they may use.

- Nightly tidy: on or off, and at what time. A collection that already existed is not turned on for you: ask for the check of your setup, and turn it on.
- Nightly AI level: off, light, medium or heavy. Each level is a most for one night, such as Medium (up to 150,000 tokens a night).
  It is used only when there is work to do, not spent every night. Say a level at any time to change it.
  A collection set up before the levels existed is at medium until you choose.
- The example collection never tidies itself at night. The first time you open it, it says how to switch this on or off.
- Nightly reading of equations: how many minutes a night your computer may spend on unread equations. It uses no tokens.
- Renamed or moved originals: an exact match is fixed by itself, and a likely match is always offered to you first.

The check of your setup also says, in words, when another version of Kovium last used a collection, or built its search entries.
Building them again is free, and is done in a copy.

## The AI program and your subscription

Kovium never uses a pay-per-use account. All its AI work goes through an AI program signed in with your subscription.
If only a pay-per-use key is available, it stops and says so.

- The AI program that does the work: which program is used, and how large a model setting.
- Adding documents without the AI: plainer notes for no tokens at all.

## What happens by itself

These happen in every collection, and nothing asks about them.

- A place for your own words in each note: the My notes section, which Kovium never rewrites.
- A map of each document: drawn by code in seconds, for no tokens.
- Duplicates found by content: a file you already have under another name is not added twice.
- Links between notes: so a question about one thing finds notes about a related one.
- Concept pages: one for each idea that several documents share.
- The record of what changed: so you can see what happened and undo it.
- Your originals stay safe: Kovium asks before it deletes or overwrites one.

## What uses your subscription

Only work done by the AI uses tokens. Looking, planning, searching, reading notes and tidying by code are free.
Doing nothing never uses tokens, and nothing is added without a plan you saw.

- Adding documents uses tokens for every document, within the limit you set. Starting with a few documents is always offered.
- Reading pictures: tokens for every picture read, up to a limit for each document.
- Splitting long documents: a few tokens for each book, only when its own contents do not show the parts.
- Matching your wording: a little for each new question.
- A fuller look for thin answers: more for that one question.
- A second check of numbers: a small share, and only for flagged findings.
- How concept pages are written: tokens for each document added, when the AI writes the pages.
- Merging look-alike concepts: a little at night, once for each pair.
- Hints about your collection for the AI: at most about 700 tokens added to each AI job.
- Suggestions of what to read next: tokens for each topic you name, up to a limit for each run.
- Nightly AI level: the steps of the nightly tidy that use the AI, never more than the level's most in one night.
- The AI program that does the work: a larger model setting uses more tokens for each job.
- Adding documents without the AI: this one saves tokens instead.

## Capping or stopping spending

Kovium never charges you money. You need no extra service or account: it only uses tokens of the subscription you already have.
Only work done by the AI uses tokens: adding documents, and a few choices that say so when you make them.
To cap what it spends, set the token limit for adding documents. A run that reaches the limit stops before the next document.
To stop spending altogether, say no to a plan, or ask for documents to be added without the AI, which uses no tokens.
Looking things up in your collection and reading your notes are free.

## Getting your notes out

Your notes are plain text files in your collection's folder on your computer, and they belong to you.
If you stop using Kovium, nothing is locked in: keep the folder, or copy the notes anywhere you like.
Any text editor opens them, and so does Obsidian, a free program for reading folders of notes.
Your original files stay where they were, or in the collection's folder if Kovium kept a copy.

## Removing a document

Ask your AI program to remove a document from the collection. The owner of the collection confirms it.
Its note is set aside rather than destroyed, so it can be put back later.
Your original file is never deleted or overwritten without asking you first.

## Working without the internet

Kovium runs on your own computer. Asking, searching and reading your notes need no internet access.
Adding documents with the AI needs your AI program, and that needs the internet. Adding documents without the AI does not.
Search by meaning uses a small program on your computer, so it works offline too. Adding a web page or a video needs the internet.

## What is not simple yet

Installing Kovium still takes a step.
