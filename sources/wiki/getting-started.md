---
type: "documentation"
title: "Getting started with Kovium"
description: "Kovium answers questions from your own documents, and shows where each answer comes from. You talk to it through an AI program you already use. This guide takes you from a fresh download to your first answer, one step at a time."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "guide"
audience: "a person setting up or using Kovium"
doc_path: "docs/../GETTING_STARTED.md"
doc_sha256: "7919b4f2820d9ba79f8709ee99eaf1e3d8d41b1f299698044dbcdf2d1c70dd28"
---

# Getting started with Kovium

Kovium answers questions from your own documents, and shows where each answer comes from.
You talk to it through an AI program you already use. This guide takes you from a fresh download
to your first answer, one step at a time.

**How to read this guide**

- A grey box headed **Type in a terminal** is a command. Copy it, paste it, press Enter.
- A grey box headed **Say to your agent** is a message. Copy it into your AI program's chat.
- Anything in `<angle brackets>` is a placeholder. Replace it, brackets included, with your own words.
- A line that starts with a bar, like the one below, is what the program says back to you.

> Kovium's programs are installed.

## Part 1. Install Kovium (once per computer)

### 1.1 What you need

- **A computer** with Windows, macOS or Linux.
- **Python 3.10 or newer.** Step 1.3 checks it.
- **One AI program**, signed in to a subscription you already pay for: Claude Code, Codex or OpenClaw.
- **No key that charges for every use.** Kovium refuses one, so nothing is ever billed per question.

### 1.2 Open a terminal

A terminal is the window where you type commands.

- **Windows:** open the Start menu, type `PowerShell`, press Enter.
- **macOS:** open Spotlight with Cmd and Space, type `Terminal`, press Enter.
- **Linux:** open the Terminal app.

### 1.3 Check Python

**Type in a terminal** (Windows):

```powershell
python --version
```

**Type in a terminal** (macOS or Linux):

```bash
python3 --version
```

You should see `Python 3.10` or a higher number. If you see an error or an older number:

- **Windows or macOS:** download Python from python.org and install it. On Windows, tick the box
  "Add python.exe to PATH" on the first screen.
- **Ubuntu or Debian Linux:**

  ```bash
  sudo apt install python3 python3-venv python-is-python3
  ```

  The last two matter: without them the install below fails, or Claude Code gets no plugin.

### 1.4 Download Kovium

Put it in a short folder path, such as `C:\sb` on Windows or your home folder elsewhere.
Windows refuses some files in very deep folders.

**Either** with git:

```bash
git clone <repository address>
```

**Or** without git: open the repository page on GitHub, click the green **Code** button, then
**Download ZIP**, and unzip it.

Then go into the folder you downloaded.

**Type in a terminal:**

```bash
cd <the folder you downloaded>
```

Do not move this folder later. If you do, run step 1.5 again from its new place.

### 1.5 Run the setup

**Type in a terminal** (Windows):

```powershell
python scripts\setup.py
```

**Type in a terminal** (macOS or Linux):

```bash
python3 scripts/setup.py
```

The first run takes a few minutes. It needs no password and spends no tokens. It finds your AI
programs and connects Kovium to each one. A good run ends like this:

> Kovium's programs are installed.
>
> Claude Code is on this computer.
>
> Claude Code is signed in to your subscription.
>
> Claude Code has Kovium as a plugin, with its skills and its start command.
>
> Now open Claude Code and say: help me start with Kovium.
>
> Nothing here needed your computer's password, and no token was used.

You may also see lines that start with "Optional". They are fine for now. See
[Part 5](#part-5-optional-extras) when you need them.

To connect only one program, add its name:

```bash
python3 scripts/setup.py --program "Claude Code"
```

Running the setup again is safe. It changes nothing that is already done.

## Part 2. Choose your AI program

Follow the page for the program you use. Each one has the same numbered steps.

| Your AI program | Your guide |
| --- | --- |
| Claude Code | [[claude-code|Getting started with Claude Code]] |
| Codex | [[codex|Getting started with Codex]] |
| OpenClaw | [[openclaw|Getting started with OpenClaw]] |

Cursor: the setup connects it too, but it has no guide here yet.

In each guide you will:

1. open the program and check that Kovium is connected;
2. say hello, try the example and take the short tour;
3. make your own collection, or connect to one that already exists;
4. add your first document.

A **collection** is one set of documents you ask questions about. It lives in one folder on your computer.

## Part 3. Readers, writers and managers

Each AI agent has a role in each collection. The role says what that agent may do there.

| Role | What the agent may do | What Kovium says |
| --- | --- | --- |
| **Reader** | Ask questions and read the notes. | "may read this collection" |
| **Suggester** | Also suggest changes, which you approve. | "may read this collection and suggest changes" |
| **Writer** | Also add documents. | "may read this collection and add documents" |
| **Manager** | Also change its settings and let other agents in. | "may read this collection and add documents, change its settings and allow other AI programs" |

### Who gets which role

- The agent that **makes** a collection is its manager.
- An agent that **connects to a collection nobody manages** becomes its manager. Say so if it should only read.
- An agent that **connects to a collection that already has a manager** is a reader.
- **No agent can raise its own role.** Only a manager can, and only when you say so.

### How a reader becomes a writer

You need the agent that manages the collection. That is usually the agent you made it with.

**Plan A: ask the managing agent.** Open a conversation with the managing agent.

**Say to your agent:**

```text
Let the agent <agent name> add documents to my <collection name> collection.
```

It shows you one sentence, like this one. Say yes.

> If you agree, this is what holds: helper-notes may read this collection and add documents. Every addition that uses your AI subscription is asked first.

Then start a new conversation with the new agent. It can now add documents.

**Not sure of the agent's name?** Ask the managing agent:

```text
Check the setup of my <collection name> collection and tell me who may do what.
```

The new agent is listed as a reader. In Claude Code and Codex its name is usually its project
folder's name, in small letters with dashes. In OpenClaw it is the agent's own id.

**Plan B: you have no access to the managing agent.** Maybe it was deleted, it belongs to another
program you no longer use, or it is on a computer you cannot reach. As the owner of this computer,
you can give the role yourself with Kovium's access key script.

1. Find the collection's id. **Say to your agent** (any agent connected to Kovium):

   ```text
   Which Kovium collections are on this computer? Give me each one's id.
   ```

2. In a terminal, go to Kovium's folder (step 1.4) and run the command below.

   **Type in a terminal** (macOS or Linux):

   ```bash
   .venv/bin/python scripts/access_token.py issue --workspace <collection id> --agent <agent name> --level manage --may add --to <agent's project folder>/.kovium/<collection id>.token
   ```

   **Type in a terminal** (Windows):

   ```powershell
   .venv\Scripts\python scripts\access_token.py issue --workspace <collection id> --agent <agent name> --level manage --may add --to <agent's project folder>\.kovium\<collection id>.token
   ```

   If a folder name has spaces, put that whole path in double quotes.
   It answers with one line: "issued a token for", the agent's name, and where the key file went.
   For a manager instead of a writer, write `--may add,settings,grant`.

3. Start a new conversation with that agent. It can now add documents.

The command also makes an access key, a long secret word kept in that file. Keys only matter once
you turn them on for a collection. Until then the file just waits there. Never paste a key into a chat.

**Plan C: the collection belongs to someone else**, on their computer. Ask them to do Plan A or Plan B.

## Part 4. When something goes wrong

| What you see | What to do |
| --- | --- |
| Your agent does not know Kovium. | Start a new conversation. AI programs load connections only when a conversation starts. Then check the connection (step 2 of your guide). |
| Your agent says "the server is not running". | No server is needed. Start a new conversation. |
| Your agent says it may only read. | Ask it to check the setup. To let it add documents, see [Part 3](#how-a-reader-becomes-a-writer). |
| Your agent uses the wrong collection. | Name the collection in your question: "In my recipe box collection, ...". |
| "Kovium's programs could not be installed." | On Ubuntu or Debian, install the packages in step 1.3. On Windows, move the folder to a short path such as `C:\sb`. Then run the setup again. |
| "Kovium is no longer where it was set up." | You moved the folder. Run the setup again from where it is now. |
| "... is signed in with a key that charges for every use." | Sign out in your AI program, sign in with your subscription, then run the setup again. |
| "No AI program was found on this computer." | Install Claude Code, Codex or OpenClaw, sign in once, then run the setup again. |

You can always ask your agent how Kovium works. For example: "How does adding documents work in Kovium?"

## Part 5. Optional extras

**What it costs.** The example, the tour and questions about your collection use no tokens. Tokens
are how your AI subscription counts its work. Adding a document uses them, and your agent always
shows the plan and its cost first. Nothing is spent until you say yes.

You do not need any of these to start. Add one when you need it, from Kovium's folder.

| Extra | When you need it | How to get it |
| --- | --- | --- |
| **Obsidian**, to look at your notes and maps | Whenever you like. It is the nicest way to look around. | Download it free from the Obsidian website. Open your collection's folder as a vault. The tour shows where. |
| **Better PDF reader** | Your PDFs have tables, columns or equations. It is a big download. | Run the first command below. |
| **Reader for Word files, web pages and e-books** | You add those kinds of files. | Run the second command below. |
| **Search by meaning** | You want notes found even when they word things differently. | Install Ollama from ollama.com, then run the third command below. |

**Type in a terminal** (macOS or Linux; on Windows write `.venv\Scripts\python` instead of `.venv/bin/python`):

```bash
.venv/bin/python -m pip install -e ".[docling]"
.venv/bin/python -m pip install -e ".[markitdown]"
ollama pull nomic-embed-text
```

**A web page for your collection.** Kovium also has a page you open in your browser. It is the
only time you start Kovium yourself. **Type in a terminal:** `python3 scripts/run_kovium.py`
(on Windows, `python scripts\run_kovium.py`), then open the link it prints.

## More to read

- [[user-guide-v1|A guide for a person]]: what Kovium does, in plain words. Your agent answers questions about Kovium from it.
- [[readme|README]]: the project, for developers.
