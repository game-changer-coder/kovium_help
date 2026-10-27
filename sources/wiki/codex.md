---
type: "documentation"
title: "Getting started with Codex"
description: "This page follows Part 1 of the getting started guide. Do that part first: Kovium must be downloaded and its setup run once."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "guide"
audience: "a person setting up or using Kovium"
doc_path: "docs/getting_started/codex.md"
doc_sha256: "9a38e6150bf0344c8b19543faa033c47ae267fcbc951b2230be7bceb159d22e1"
---

# Getting started with Codex

This page follows Part 1 of the getting started guide.
Do that part first: Kovium must be downloaded and its setup run once.

Anything in `<angle brackets>` is a placeholder. Replace it, brackets included.

## Step 1. Open your AI program in a folder of your own

Pick or make a folder for your work, for example a folder for your thesis or your recipes.
Codex works there, and Kovium can keep your collection there too.

First make sure Codex is signed in to your subscription. **Type in a terminal:**

```bash
codex login status
```

If it is not signed in, run `codex login` and sign in with your ChatGPT account. Never use a key that charges for every use.

Then open Codex in your folder. **Type in a terminal:**

```bash
cd <your project folder>
codex
```

If Codex asks whether you trust the contents of this directory, choose yes if you do. In a trusted folder, Kovium
keeps a collection's connection for this project only. Otherwise every project sees it.

## Step 2. Check that Kovium is connected

**Type in Codex:**

```text
/mcp
```

You should see `kovium` in the list. You can also check from a terminal with `codex mcp list`.

Not there? Quit Codex with `/quit`, run the setup again
(step 1.5), then open Codex again.

## Step 3. Say hello: the welcome and the example

**Say to your agent:**

```text
help me start with Kovium
```

Your agent greets you and asks:

> Want to see it work on a small example first? It takes about a minute and uses no tokens.

Say yes. It opens a small example collection and suggests three questions. Pick one, and look at
how the answer names its sources.

## Step 4. Take the tour (optional)

After the example, your agent offers a short tour: nine small steps, each with something to try.
Say yes to start it. Each step ends with what to say next, like this:

> Say “next” for step 2 of 9: One document's journey.

To stop, simply stop saying next, or ask about something else.

## Step 5. Make your own collection

**Say to your agent:**

```text
Make my own Kovium collection. It will hold <what it will hold, in a few words>.
```

Your agent asks where to keep it:

> Where should your collection My recipes be kept?

You can choose the folder you are working in now (recommended), another folder you name, or the
default place inside Kovium's own folder. Avoid the default place: an update of Kovium
could put it at risk.

When it is made you see:

> Your collection My recipes is ready, and this conversation now uses it.

You are its manager. It also asks one question about the nightly tidy using your AI. "No" is fine
to start: the free tidy still runs every night.

If your agent does not seem to have the new collection, quit Codex with `/quit` and start it again
in the same folder.

## Step 6. Or: connect to a collection that already exists

Use this when the collection already exists on this computer, for example one made with another
agent or another AI program.

**Say to your agent:**

```text
Connect yourself to my <collection name> collection.
```

Not sure of the name? Ask: "Which Kovium collections are on this computer?"

If the collection came from another computer and is not listed yet, give its folder instead:

```text
Connect yourself to the Kovium collection in the folder <full path of the folder>.
```

Your agent shows a plan first. It says where the connection goes and what your agent may do there.
Say yes. Then:

1. Quit Codex with `/quit`.
2. Start it again in the same folder with `codex`.
3. Ask a question about the collection.

Codex loads its connections only when it starts, so this restart is needed.

Does the plan say your agent may only read? Then another agent manages this collection.
To let yours add documents, see
How a reader becomes a writer.

## Step 7. Add your first document

**Say to your agent:**

```text
Add <full path of a file or folder> to my collection.
```

Your agent first shows a plan: how many documents, and about how many tokens. Nothing is spent yet.
Say yes to start. Start with one or two documents, to see how it works.

Then ask a question about what you added:

```text
In my <collection name> collection, <your question>
```

## Step 8. Optional: your own Kovium agent

Kovium can add an agent of its own to your project, which answers the way you like.

**Say to your agent:**

```text
Make me a Kovium agent for this collection.
```

It asks three short questions, then shows what it will add. Say yes, and it tells you the agent's
name. Codex uses that agent only when you ask for it by name:

```text
Ask kovium_<collection id> to answer: <your question>
```

In the name, every dash of the collection's id becomes an underscore.

## Step 9. Stop using a collection

**Say to your agent:**

```text
Disconnect from my <collection name> collection.
```

Its connection and its instructions go. The collection itself stays as it was.

## What is different in Codex

- **Trusted folders.** Codex reads a project's own settings only in a folder you trust. In any
  other folder, a collection's connection goes where all your projects see it, and you are told.
- **Restart, not a new chat.** Codex loads connections when it starts. After connecting to a
  collection, quit Codex and start it again.
- **Ask for your agent by name.** Codex hands work to an agent only when you name it (step 8).

## When something goes wrong

- **Kovium is not listed in `/mcp`.** Run the setup again, then start Codex again.
- **The connection is seen by every project.** That folder is not trusted in Codex. It still works.
  To keep it to one project, trust the folder in Codex, then connect again (step 6).
- **Your agent ignores your collection.** Start Codex again in the same folder, and name the collection in your question.
- More fixes are in Part 4 of the getting started guide.
