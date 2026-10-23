---
type: "documentation"
title: "Agent Knowledge Contract V1"
description: "This is the canonical public name for the agent-facing knowledge contract."
status: "stable"
reading_status: "read"
product: "second brain"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing second brain"
doc_path: "docs/agent_knowledge_contract_v1.md"
doc_sha256: "2dc0a10190c07ccc0b69c08fed20ffcfbb4d1242e91df4c061f0bb23084d4f4e"
---

# Agent Knowledge Contract V1

This is the canonical public name for the agent-facing knowledge contract.

The current detailed contract text remains in:

- `docs/agent_memory_contract_v1.md`

That legacy filename is being retained temporarily for compatibility with existing internal references, but the public API surface and naming now use `knowledge` rather than `memory`.

Public knowledge endpoints:

- `GET /agent/knowledge/contract`
- `POST /agent/knowledge/answer`
- `POST /agent/knowledge/task_context`
- `POST /agent/knowledge/read_next`
- `POST /agent/knowledge/read`
- `GET /agent/knowledge/changes` (what changed since a cursor; Track AB0)
- `POST /agent/knowledge/entity` (one card for a name; Track AB0)
- `GET /agent/knowledge/overview` (what the collection holds, for a newcomer; Track AB2)
- `GET /agent/knowledge/codes` (`code` for only some; Track AB3)
- `GET /agent/knowledge/capabilities` (what the workspace can do, what is not ready; Track AB3)
- `GET /agent/knowledge/review_queue` (the review queue's open items; Track AB3)

Public contract/version names:

- `contract_version`: `agent_knowledge_contract_v2`
- `schema_version`: `agent_knowledge_response_v2`

v2 makes every path in an answer workspace-relative (resolve one with
`GET /wiki/page?path=<path>`); see `docs/agent_memory_contract_v1.md`,
"What changed in v2".
