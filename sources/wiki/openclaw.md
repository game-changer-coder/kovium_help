---
type: "documentation"
title: "Getting started with OpenClaw"
description: "This page follows Part 1 of the getting started guide. Do that part first: Kovium must be downloaded and its setup run once."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "guide"
audience: "a person setting up or using Kovium"
doc_path: "docs/getting_started/openclaw.md"
doc_sha256: "d47aa8da58c2d7082ce87dc88a9610fdb946177abd9e998d327a13b4ab64419b"
---

# Getting started with OpenClaw

This page follows Part 1 of the getting started guide.
Do that part first: Kovium must be downloaded and its setup run once.

Anything in `<angle brackets>` is a placeholder. Replace it, brackets included.

**If the setup said OpenClaw could not be connected**, like this:

> OpenClaw could not be connected: it only accepts changes while its background service is stopped. Stop that service, then run this again.

then stop OpenClaw's background service, run the setup, and start the service again.
Your OpenClaw agents and chat apps pause for that minute.

**Type in a terminal** (in Kovium's folder; on Windows write `python scripts\setup.py`):

```bash
openclaw gateway stop
python3 scripts/setup.py --program OpenClaw
openclaw gateway start
```

## Step 1. Open your AI program in a folder of your own

In OpenClaw, each agent already has a folder of its own to work in. So you only open a chat with
the agent you want to use:

- in a chat app you connected to OpenClaw, or
- in your browser: run `openclaw dashboard` in a terminal, or
- in the terminal: run `openclaw tui`.

OpenClaw must be signed in to your subscription. Never use a key that charges for every use.

You will need your agent's id later. **Type in a terminal:**

```bash
openclaw agents list
```

Note the id of the agent you chat with.

## Step 2. Check that Kovium is connected

**Type in a terminal:**

```bash
openclaw mcp list
openclaw mcp reload
```

The first command should list `kovium`. The second makes your agents pick up connections
that were just added.

Not there? Run the setup again (step 1.5).

Then start a fresh conversation. **Type in your chat with the agent:**

```text
/new
```

## Step 3. Say hello: the welcome and the example

**Say to your agent:**

```text
help me start with Kovium. Use your Kovium tools for this.
```

The second sentence matters in OpenClaw. Its agents see Kovium's tools only after searching for
them. Without that sentence, an agent may take "Kovium" as a general idea and start making
note folders of its own.

Your agent greets you and asks:

> Want to see it work on a small example first? It takes about a minute and uses no tokens.

Say yes. It opens a small example collection and suggests three questions. Pick one, and look at
how the answer names its sources.

If your agent answers in general words instead, and never mentions the example, send `/new` and
try again with the same two sentences.

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

You can choose the folder the agent works in (recommended), another folder you name, or the
default place inside Kovium's own folder. Avoid the default place: an update of Kovium
could put it at risk.

When it is made you see:

> Your collection My recipes is ready, and this conversation now uses it.

You are its manager. It also asks one question about the nightly tidy using your AI. "No" is fine
to start: the free tidy still runs every night.

All your OpenClaw agents share the connection the setup added. So the collection you make here
becomes the one every OpenClaw agent opens through it. To give one agent a collection of its own,
use step 6 with that agent.

## Step 6. Or: connect to a collection that already exists

Use this when the collection already exists on this computer, for example one made with another
agent or another AI program. It is also how one OpenClaw agent gets a collection for itself alone.

**Say to your agent** (with the id from step 1):

```text
Connect yourself to my <collection name> collection. Your OpenClaw agent id is <agent id>.
```

Not sure of the name? Ask: "Which Kovium collections are on this computer?"

If the collection came from another computer and is not listed yet, give its folder instead:

```text
Connect yourself to the Kovium collection in the folder <full path of the folder>. Your OpenClaw agent id is <agent id>.
```

Your agent shows a plan first. It says where the connection goes and what your agent may do there.
Say yes. Then send `/new` and ask a question that names the collection.

Does the plan say your agent may only read? Then another agent manages this collection.
To let yours add documents, see
How a reader becomes a writer.

## Step 7. Add your first document

**Say to your agent:**

```text
Add <full path of a file or folder> to my <collection name> collection.
```

Your agent first shows a plan: how many documents, and about how many tokens. Nothing is spent yet.
Say yes to start. Start with one or two documents, to see how it works.

Then ask a question about what you added. Always name the collection:

```text
In my <collection name> collection, <your question>
```

A good answer names its source, the document it came from. No source? Then your agent read the
files itself instead of asking Kovium. Step 8 fixes that for good.

## Step 8. Optional: your own Kovium agent

OpenClaw has no extra agent files. Instead, Kovium adds a short part to your agent's own
instructions, below what is already there. The rest stays exactly as you wrote it.

**Say to your agent:**

```text
Make me a Kovium agent for my <collection name> collection. Use your Kovium tools for this.
```

It says what it will add, and asks how its answers should sound: everyday words or technical ones,
short or detailed, and which tone. The recommended answer comes first each time. Answer, or say yes
to the recommended ones. When you connect in step 6, this part is added for you already.

From then on, a question that names the collection is answered from it, with its sources.

## Step 9. Stop using a collection

**Say to your agent:**

```text
Disconnect from my <collection name> collection.
```

Its connection and its instructions go. The collection itself stays as it was.

A collection made in step 5 is a little different. The connection all your OpenClaw agents share
stays on it. It keeps opening that collection for them until one of them opens or makes another.

## What is different in OpenClaw

- **Your agent's id.** When connecting to a collection, your agent must send its own id. Give it
  in your message, as in step 6.
- **Name the collection.** OpenClaw agents find Kovium's tools by searching for them. Name
  the collection, or say "Use your Kovium tools for this", and the agent looks there first.
  Saying only "Kovium" was not enough in our tests.
- **One agent sees it.** A collection connected in step 6 is shown to that one agent only. This
  hides it from your other agents there, but it is not a lock.
- **Skills.** The setup adds Kovium's skills to your default OpenClaw agent only. Another
  agent, with a folder of its own, still works, guided by the connection and its instructions.
- **A new conversation** means sending `/new` in your chat with the agent.
- **A brand-new agent asks for a name.** An OpenClaw agent you have just added may end its first
  replies by asking what to call it. Answer it whenever you like; it does not change Kovium.

## When something goes wrong

- **`kovium` is not in `openclaw mcp list`.** Stop the background service, run the setup, start it again (top of this page).
- **Your agent answers from its own memory, not your collection.** Run `openclaw mcp reload`, send `/new`, then name the collection in your question. If it still gives no source, add "Use your Kovium tools for this", or do step 8.
- **Your agent makes note folders of its own.** It took "Kovium" as a general idea. Delete those folders, send `/new`, and use the sentence from step 3.
- **Your agent says it needs its id.** Give it the id from `openclaw agents list`.
- More fixes are in Part 4 of the getting started guide.
