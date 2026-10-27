---
type: "documentation"
title: "Getting started with Claude Code"
description: "This page follows Part 1 of the getting started guide. Do that part first: Kovium must be downloaded and its setup run once."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "guide"
audience: "a person setting up or using Kovium"
doc_path: "docs/getting_started/claude_code.md"
doc_sha256: "3e05e3b5cdd56923b3206028c477cc5c46a43dee08d5e4b384fea0b5cadfceb8"
---

# Getting started with Claude Code

This page follows Part 1 of the getting started guide.
Do that part first: Kovium must be downloaded and its setup run once.

Anything in `<angle brackets>` is a placeholder. Replace it, brackets included.

## Step 1. Open your AI program in a folder of your own

Pick or make a folder for your work, for example a folder for your thesis or your recipes.
Claude Code works there, and Kovium can keep your collection there too.

**Type in a terminal:**

```bash
cd <your project folder>
claude
```

The first time, Claude Code asks you to sign in. Choose your Claude subscription, not a key that charges for every use.

## Step 2. Check that Kovium is connected

**Type in Claude Code:**

```text
/mcp
```

You should see `kovium` in the list, marked as connected. Press Esc to close the list.

Not there? Close Claude Code with `/exit`, run the setup again
(step 1.5), then open Claude Code again.

## Step 3. Say hello: the welcome and the example

**Say to your agent:**

```text
help me start with Kovium
```

If the setup said Claude Code has Kovium as a plugin, you can type `/kovium:start` instead.

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

If your agent does not seem to have the new collection, close Claude Code with `/exit` and open it
again in the same folder.

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

1. Close Claude Code with `/exit`.
2. Open it again in the same folder with `claude`.
3. Ask a question about the collection.

The connection works only in this project folder, and only from a new conversation.

Does the plan say your agent may only read? Then another agent manages this collection.
To let yours add documents, see
How a reader becomes a writer.

## Step 7. Add your first document

**Say to your agent:**

```text
Add <full path of a file or folder> to my collection.
```

You can also drag a file or folder into the chat. Your agent first shows a plan: how many
documents, and about how many tokens. Nothing is spent yet. Say yes to start. Start with one or two
documents, to see how it works.

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

It asks three short questions, then shows what it will add. Say yes. Claude Code may hand
questions to it by itself, or you can ask for it by name:

```text
Use the kovium-<collection id> agent to answer: <your question>
```

## Step 9. Stop using a collection

**Say to your agent:**

```text
Disconnect from my <collection name> collection.
```

Its connection and its instructions go. The collection itself stays as it was.

## What is different in Claude Code

- **The plugin.** The setup adds Kovium as a plugin. That brings the start command
  `/kovium:start` and skills that teach Claude Code how to use a collection.
- **No plugin on some Linux computers.** If the setup said "is connected without the plugin", your
  computer has no `python` command. Everything still works. For the plugin, install
  `python-is-python3` (step 1.3), then run the setup again.
- **One project at a time.** When you connect to a collection (step 6), only that project folder
  sees it. Open Claude Code in that folder to use it.
- **A new conversation** means closing Claude Code with `/exit` and opening it again with `claude`.

## When something goes wrong

- **Kovium is not listed in `/mcp`.** Run the setup again, then open Claude Code again.
- **It is listed twice.** Run the setup again. It removes the older copy and says so.
- **Your agent ignores your collection.** Open Claude Code in the project folder where it joined, and name the collection in your question.
- More fixes are in Part 4 of the getting started guide.
