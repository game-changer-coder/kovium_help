# kovium_help: the Kovium manual, as a collection you can ask

This is the full manual of Kovium, a local-first knowledge platform,
written as a collection of notes, so that you can ask it questions in plain
words and see which document each answer comes from.

You do not need to read this repository. Kovium fetches it for you
when you say yes to its offer, in the tour or when you ask how something
works. Nothing is typed, nothing is built, and nothing is paid for.

## What is in it

One note for each current document of Kovium: the user guide, the
contracts that say what the software does, the specification, the overview of
how the code is organised, the four skills for AI programs, and the catalogue
of every choice Kovium offers. The notes in `sources/wiki/` are those
documents, word for word, under their own headings.

It does not hold the history. Plans, reports and records say what was intended
or true on some day, and a helper that answered from them could be confidently
wrong, so they are left out.

## How Kovium fetches it

When you say yes, Kovium makes a copy of this repository on your computer,
in its own folder for collections, apart from your own collections. From then on
a question about Kovium is answered from it, and every answer names the
document it comes from. Your own documents are never mixed with it, and nothing
you ask is sent anywhere.

Each copy says which version of Kovium it describes (`manifest.json`). If
the copy you have describes another version, Kovium tells you in plain words
and offers the copy that matches.

Where Kovium fetches it from is a setting of the install
(`config/help.json` in the Kovium folder), so it can point at a copy kept
elsewhere.

## How it is made

The maintainers build it when a release is made, with
`scripts/build_help_collection.py` in the Kovium repository. It spends no
tokens and calls no AI: each document is taken in as a note written by hand and
read by code, and the links between documents become links between notes. The
manifest records the version it describes and a fingerprint of every source
document, so the build can say which ones changed since the last release.

The graph, the concept pages and the maps that Kovium derives from these
notes are kept in `derived/`, made by code at release with no AI. When the
notes and the Kovium version on your computer match the ones they were
derived from, opening the collection reads them, in a few seconds; otherwise
Kovium builds them on your computer, as for any collection.

## What not to do

Do not edit the notes here: the next release replaces them. To correct the
manual, change the document in the Kovium repository.

## Licence

The same as Kovium: see `LICENSE`.
