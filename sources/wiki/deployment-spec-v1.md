---
type: "documentation"
title: "Deployment Specification V1"
description: "Version 1 is:"
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "guide"
audience: "a person setting up or using Kovium"
doc_path: "docs/deployment_spec_v1.md"
doc_sha256: "282bd26268039d531280b3ca35182ab83029233ad9f261f6aca800b668735b9f"
---

# Deployment Specification V1

## Deployment Scope
Version 1 is:
- local-only
- private to the Researcher agent
- not exposed to external devices or other agents

## Binding Policy
All services should bind to loopback only by default.
Examples:
- backend API on `127.0.0.1`
- UI served locally only
- graph DB not exposed beyond local process access unless required

## Service Layout
Recommended local services:
- graph database service
- backend API service
- analytics/background job service
- frontend UI service

## Operational Assumptions
- single user
- single machine
- trusted local environment
- no multi-user auth yet

## Backup Strategy
Back up:
- Markdown sources
- staging JSON
- graph exports/dumps
- config files
- logs if useful

## Recovery Strategy
If graph backend fails:
1. restore sources
2. rebuild staging records
3. rebuild graph
4. recompute analytics

## Future Expansion Notes
The architecture should allow future migration to:
- multi-agent local access
- authenticated local API access
- MCP wrapping
- optional remote UI access

These are not V1 requirements.


## Job lanes and responsiveness

The brain-keeper runs background work in **two lanes per workspace**:

| lane | config | default | carries |
|---|---|---|---|
| default | `max_concurrent_jobs` | 2 | every ordinary job |
| long | `max_concurrent_long_jobs` | 1 | `process_book`, `process_folder`, and anything else in `LONG_RUNNING_JOBS` |

A job belongs in the long lane when its duration is a function of **input size**
rather than of a fixed number of agent turns. `process_book` is one whole
pipeline per chapter, so a fourteen-chapter book is fourteen of them back to
back.

The lanes exist because a single pool lets unbounded work starve everything
else. Measured before the split: a 50 ms job submitted behind two 3-second
"books" waited the full 3 seconds — head-of-line blocking, not load. After: 0.06
seconds. The long lane's size is deliberately *not* derived from
`max_concurrent_jobs`; the point is that long work cannot grow to fill the pool
however large the pool is.

Both lanes share one job registry, so a caller polls a job the same way
whichever lane it landed in. `GET /ops/status` reports both.

Per-workspace pools mean a backlog in one instance cannot starve another on a
shared host. API routes are synchronous and run in Starlette's 40-slot
threadpool, so a slow request never blocks the event loop; `/sources/process_book`
additionally **backgrounds by default**, because a book is never fast.
