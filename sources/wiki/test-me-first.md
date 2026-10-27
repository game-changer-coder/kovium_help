---
type: "documentation"
title: "Test Me First"
description: "Kovium once and connects your AI program (Claude Code, Codex or OpenClaw). No server is started: your AI program starts Kovium's connection itself."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "guide"
audience: "a person setting up or using Kovium"
doc_path: "docs/TEST_ME_FIRST.md"
doc_sha256: "9449b19abbd9f53af377984294e7a801cffe582d447d21a36926897e64e78a62"
---

# Test Me First

**New to Kovium?** Follow the getting started guide instead. It installs
Kovium once and connects your AI program (Claude Code, Codex or OpenClaw). No server is started:
your AI program starts Kovium's connection itself.

This page is for developers who want the web page and the HTTP routes.

## The web page, over HTTP

### 1. Start the app
```bash
cd <repo>
./scripts/run_kovium.sh        # or, on any system: python scripts/run_kovium.py
```

### 2. In another terminal, seed sample data
```bash
cd <repo>
./scripts/seed_sample.sh
```
Every request must name a workspace, so this registers one called `sample`
(set `WORKSPACE_ID=<name>` for another), loads the sample notes into it, and
prints the link to open. Safe to run again.

### 3. Open the UI
- http://127.0.0.1:8000/?workspace_id=sample — without `?workspace_id=` the UI
  says no workspace is selected
- API docs: http://127.0.0.1:8000/docs

## What to test
- graph loads
- node selection works
- side panel shows provenance
- gap summary appears
- gap interpretation appears
- filters work
- path highlighting works on double click

The welcome, the example and its tour over HTTP are in the README
("Try it in five minutes, over HTTP") and in `docs/welcome_v1.md`.
