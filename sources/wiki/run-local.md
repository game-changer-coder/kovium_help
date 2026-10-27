---
type: "documentation"
title: "Run Local"
description: "once and connects your AI program. No server is needed for that."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "guide"
audience: "a person setting up or using Kovium"
doc_path: "docs/run_local.md"
doc_sha256: "76f67e5111606361518522701118486cb7d148c98a4e24232d5ef36ee3473392"
---

# Run Local

**New to Kovium?** Follow the getting started guide. It installs Kovium
once and connects your AI program. No server is needed for that.

To run the web page and the HTTP routes yourself, see [[test-me-first|TEST_ME_FIRST.md]] and the
README's "Try it in five minutes, over HTTP". In short, from the checkout:

```bash
python scripts/run_kovium.py     # makes .venv if missing, installs, serves on http://127.0.0.1:8000
python scripts/run_kovium.py --skip-install --background   # detached: returns once it answers
```

`--background` is the form an agent's own command tool uses: the service runs on its own, its output in
`logs/service.log`, and the command returns. Running it again while the service answers changes nothing.
Stop it as you stop any process; stop the `uvicorn` process too, since stopping only the start script
leaves the server running.

Then open `http://127.0.0.1:8000/?workspace_id=<id>` for the UI and `http://127.0.0.1:8000/docs`
for the API docs. The store is persistent (SQLite under each workspace's `data/staging/`).
