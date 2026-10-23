---
type: "documentation"
title: "Test Me First"
description: "Every request must name a workspace, so this registers one called sample (set WORKSPACE_ID=<name> for another), loads the sample notes into it, and prints the link to open. Safe to run again."
status: "stable"
reading_status: "read"
product: "second brain"
product_version: "0.1.0"
doc_kind: "guide"
audience: "a person setting up or using second brain"
doc_path: "docs/TEST_ME_FIRST.md"
doc_sha256: "5bda159fc2292236847a16848378b3c321b4fad97bff504c41e19b81a67a6824"
---

# Test Me First

## Fastest way to try the MVP

### 1. Start the app
```bash
cd ~/second_brain
./scripts/run_second_brain.sh
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

## Expected current behavior
- persistent storage is active
- Ollama is configured as the live interpretation provider
- deterministic fallback is used automatically if the model output is unusable
