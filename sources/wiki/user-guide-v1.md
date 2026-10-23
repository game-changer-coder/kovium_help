---
type: "documentation"
title: "User guide v1"
description: "A guide for a person, not a contract. Its words follow plain_language_rules_v1.md and are checked by tests/contracts/test_option_catalogue.py. Each choice named here has its four lines in the option catalogue."
status: "stable"
reading_status: "read"
product: "second brain"
product_version: "0.1.0"
doc_kind: "guide"
audience: "a person setting up or using second brain"
doc_path: "docs/user_guide_v1.md"
doc_sha256: "be006124574ebfd2c6abfb3e69a7daeedaab06ba2e16ac94f33658bbf0c545ef"
---

# User guide v1

<!--
A guide for a person, not a contract. Its words follow plain_language_rules_v1.md and are checked by
tests/contracts/test_option_catalogue.py. Each choice named here has its four lines in the option catalogue.
-->
# Second brain: a guide for a person

Second brain answers questions from your own documents, and shows the exact place each answer comes from. You talk to it
through the AI program you already use. This guide says what it does, what it costs, and what you can change.

## Getting started

To install second brain, run its setup once from its folder. It connects second brain to the AI programs it finds on
your computer, and to Claude Code as a plugin, which also brings its skills.

Say "help me start with second brain" to your AI program. It first shows second brain on a small example collection.
That takes about a minute and uses no tokens. You never type a command for it.

Then it offers a short tour of what happens to a document you add. You can stop at any step.
In the tour you add one waiting page yourself and watch its note, its map and its links appear.
The example and its tour are for a first start, before you have a collection of your own.

- The example collection: a few short pages about second brain, ready to ask, with one page waiting for you to add.
- Help at any time: ask how anything works, and the answer comes from this guide, with the section it comes from.

## What you need to run second brain

You need an AI program you already use, signed in with a subscription you already pay for.
It is the kind of AI assistant that works on your own computer, such as an AI coding assistant. Claude Code, Codex and OpenClaw work today.
You need no pay-per-use account and no extra service. Second brain itself runs on your own computer.

## What stays on your computer

Your documents, your notes and your collections stay on your computer. Second brain never shares your files with other people or puts them online.
The only text that leaves your computer is what your AI program reads to do its work, sent to the service of your subscription.

## What a collection is

A collection is a set of documents you ask questions about. It lives in one folder on your computer.
Second brain keeps your documents there, with a note for each one, and everything it learns from them.

You can copy that folder to another computer and open it there. Nothing needs to be added again.
Your own originals are never deleted or overwritten without asking you first.

## Your own collection

To make one, tell your AI program in a few words what it will hold. It is kept where second brain keeps collections,
unless you name another folder. From then on that conversation uses it.

To open a collection you already have, for example a copy from another computer, say which folder it is in.
If some folders it points to are not on this computer, you are told which, and you say where they are here.

Each collection has a connection of its own in your AI program. To use a collection from another AI program,
ask to connect that program to it. It may only read, until you say it may do more.

You decide what each AI program may do with a collection. It may only read it, or also suggest changes you approve,
or also add documents and change its choices. The change is shown to you as one sentence before it applies.
An AI program can never give itself more than you allowed.

To see what works on this computer, and what is off or missing, ask your AI program to check the setup.

## Adding documents

You say what to add: files, a folder, or links. Second brain first makes a plan, and the plan costs nothing.
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
- Keeping copies of your originals: second brain keeps its own copy of each file, or only points to where it lies.
- Named folders for your originals: a short name for a folder you keep elsewhere, so notes survive a move.
- Kind of note when none is clear: what a document is taken to be when nothing in it says.

## Adding files, folders and books

Tell your AI program what to add: one file, several files, a whole folder, or a link to a web page or a video.
A book or another long document is added the same way, and second brain cuts it into its chapters for you.
First you see the plan, with what it will cost. Nothing is added until you say yes, or until it fits the limit you set.
The plan also says what your files need, such as the better PDF reader. A choice that cannot wait is asked before anything is added.
Adding goes on in the background, so the conversation carries on. At the end you get the short report.
Files you already added, even under another name, are skipped.

## Reading your documents

Most files are read at once. A few kinds need a small program on your computer, and second brain tells you which ones
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
- Videos added from a link: second brain reads the captions the site shows, and never the video file.

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

Second brain can fetch its own manual as a collection you can ask. It holds the current documents about second brain, and every answer shows the document it comes from.
The tour offers it at the end, and you can ask for it at any time. It is free, and nothing has to be built or paid for.
It is kept apart from your own collections: it answers questions about second brain, and never about your documents.
Until you have it, your questions about second brain are answered from this guide.
If the manual you have was made for another version of second brain, you are told, and asking for it again fetches the matching one.
Without the internet it cannot be fetched, and everything else works as before.

## Your own second brain agent

If your AI program lets you add agents of your own, second brain can add one for a collection, which answers the way you like.
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

Every note that second brain writes ends its content with a section called My notes, and a short hint line under its title.
Write there whatever you like beside a document: a doubt, a question for a colleague, a thought for later.
Second brain never rewrites that section. Your words stay exactly as you wrote them when the document is added again,
read again, renamed or tidied. If the document is removed, your words are kept in a separate file, and you are told where.
Your words can help a question find the note. An answer shows them apart, as yours, and never as something the document says.

- How concept pages are written: the AI writes them, or plain code assembles them for free.
- Merging look-alike concepts: obvious duplicates are merged and every merge can be undone. The rest are suggested to you.
- Your own notes and ideas on concept pages: whether they appear there.
- Your own words section for notes you already have: a one-time step that adds the empty section to older notes. It shows how many first, and takes a backup.
- The name of your own words section: the title can be set in the language you write in. Words under the old title are still kept.
- Hints about your collection for the AI: a short description of your collection, kept off until you approve one.
- Suggestions of what to read next: compares what your documents say about topics you name. It is off until you ask.
- Letters with marks in file names: how such letters are spelled in the names second brain makes.

## Words your own documents use

Second brain finds the parts of a book and checks numbers with short lists of words.
The lists hold many common words already, and you can add the ones your own documents use.
Ask your AI program to add them, and it shows you the change before it applies.

- Words for a table of contents: the heading that marks the contents of a book.
- Numbers written as words: how chapters are numbered when they are written out.
- Words for measured quantities: the word for a quantity and its unit, so a number can be checked.

## What happens at night

Every collection you make tidies itself each night, at no cost, on the computer that made it. Nothing needs to be left running.
It adds new label words, fixes notes after a rename, fills in missing search entries, puts back lost letters and reads equations left unread.
On a folder shared between computers, one computer does the tidying. Some steps use the AI, and they wait until you set a token limit.

- Nightly tidy: on or off, and at what time. A collection that already existed is not turned on for you: ask for the check of your setup, and turn it on.
- Token limit for the nightly AI steps: the most tokens those steps may use in one night. Until you set it, they use none.
- Nightly reading of equations: how many minutes a night your computer may spend on unread equations. It uses no tokens.
- Renamed or moved originals: an exact match is fixed by itself, and a likely match is always offered to you first.

The check of your setup also says, in words, when another version of second brain last used a collection, or built its search entries.
Building them again is free, and is done in a copy.

## The AI program and your subscription

Second brain never uses a pay-per-use account. All its AI work goes through an AI program signed in with your subscription.
If only a pay-per-use key is available, it stops and says so.

- The AI program that does the work: which program is used, and how large a model setting.
- Adding documents without the AI: plainer notes for no tokens at all.

## What happens by itself

These happen in every collection, and nothing asks about them.

- A place for your own words in each note: the My notes section, which second brain never rewrites.
- A map of each document: drawn by code in seconds, for no tokens.
- Duplicates found by content: a file you already have under another name is not added twice.
- Links between notes: so a question about one thing finds notes about a related one.
- Concept pages: one for each idea that several documents share.
- The record of what changed: so you can see what happened and undo it.
- Your originals stay safe: second brain asks before it deletes or overwrites one.

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
- Token limit for the nightly AI steps: the steps of the nightly tidy that use the AI, never more than the limit in one night.
- The AI program that does the work: a larger model setting uses more tokens for each job.
- Adding documents without the AI: this one saves tokens instead.

## Capping or stopping spending

Second brain never charges you money. You need no extra service or account: it only uses tokens of the subscription you already have.
Only work done by the AI uses tokens: adding documents, and a few choices that say so when you make them.
To cap what it spends, set the token limit for adding documents. A run that reaches the limit stops before the next document.
To stop spending altogether, say no to a plan, or ask for documents to be added without the AI, which uses no tokens.
Looking things up in your collection and reading your notes are free.

## Getting your notes out

Your notes are plain text files in your collection's folder on your computer, and they belong to you.
If you stop using second brain, nothing is locked in: keep the folder, or copy the notes anywhere you like.
Any text editor opens them, and so does Obsidian, a free program for reading folders of notes.
Your original files stay where they were, or in the collection's folder if second brain kept a copy.

## Removing a document

Ask your AI program to remove a document from the collection. The owner of the collection confirms it.
Its note is set aside rather than destroyed, so it can be put back later.
Your original file is never deleted or overwritten without asking you first.

## Working without the internet

Second brain runs on your own computer. Asking, searching and reading your notes need no internet access.
Adding documents with the AI needs your AI program, and that needs the internet. Adding documents without the AI does not.
Search by meaning uses a small program on your computer, so it works offline too. Adding a web page or a video needs the internet.

## What is not simple yet

Installing second brain still takes a step.
