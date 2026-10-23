---
type: "documentation"
title: "Run Local"
description: "From ~/second_brain:"
status: "stable"
reading_status: "read"
product: "second brain"
product_version: "0.1.0"
doc_kind: "guide"
audience: "a person setting up or using second brain"
doc_path: "docs/run_local.md"
doc_sha256: "4b1f1c1129d281b6b725bfe35915a17b95a2ff7c34aead2f69b054dd87bbe40d"
---

# Run Local

## Start the API and UI locally
From `~/second_brain`:

```bash
python3 -m venv .venv
.venv/bin/pip install -e .[dev]
.venv/bin/uvicorn second_brain.api.app:create_app --factory --host 127.0.0.1 --port 8000
```

Then open:
- `http://127.0.0.1:8000/` for the UI shell
- `http://127.0.0.1:8000/docs` for FastAPI docs

## Seed the bundled sample corpus
Use the API:

```bash
curl -X POST http://127.0.0.1:8000/ingest/sample -H 'Content-Type: application/json' -d '{}'
```

## Current scope
- UI is a first simple shell
- backend is the working MVP core
- graph storage is still in-memory
- restarting the service clears in-memory graph state until persistent storage is added
