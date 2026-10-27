---
type: "documentation"
title: "Workstream D Operations Recovery Policy V1"
description: "Define the operational execution model, restore rehearsal expectations, schema migration discipline, and incident runbooks required to treat Workstream D as complete."
status: "stable"
reading_status: "read"
product: "Kovium"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing Kovium"
doc_path: "docs/workstream_d_operations_recovery_policy_v1.md"
doc_sha256: "5793e4788feec713e667ab32b890dd7c819f54746707ce8309e527b7d1d56499"
---

# Workstream D Operations Recovery Policy V1

## Purpose
Define the operational execution model, restore rehearsal expectations, schema migration discipline, and incident runbooks required to treat Workstream D as complete.

## Scheduled operations policy
Scheduled operations are now executed through typed `OpsService` actions rather than passive script references. The authoritative schedule lives in `config/maintenance_schedule.json` and currently supports:

- `integrity_check`
- `maintenance_inspect_registry`
- `maintenance_refresh_registry`
- `backup_snapshot`
- `normalization_monitoring_summary`

Execution surfaces:

- API: `POST /ops/schedule/run`
- CLI: `scripts/ops_run_scheduled_jobs.py`
- Status readout: `GET /ops/schedule` and `GET /ops/status`

Execution rules:

- Every run appends a JSONL event to `logs/scheduled_jobs.jsonl`.
- Registry-backed maintenance jobs skip safely when no tracked sources exist.
- Backup jobs rotate archived snapshots according to `keep_last`.
- Monitoring jobs raise actionable warnings when fallback or slow-normalization rates cross the current threshold.

## Restore rehearsal policy
Restore safety is proven through an isolated rehearsal rather than by mutating the live workspace.

Execution surfaces:

- API: `POST /ops/restore/rehearsal`
- CLI: `scripts/ops_restore_rehearsal.py`
- Status readout: `GET /ops/status`
- Paths in operator reports (`GET /ops/status`, including
  `normalization.recent_events[].source_path`/`wiki_path`; `GET /ops/metrics`
  `metrics_path`; `GET /ops/governance_audit` `audit_path`) are
  workspace-relative with forward slashes, whatever the events recorded (Track
  I6, 2026-10-09). So are the backup and rehearsal records a status report
  carries (`latest_restore_rehearsal`, `schedule.recent_executions`,
  `schedule.latest_by_job`): a backup is `data/backups/<file>.zip`, and a
  rehearsal's restored files are relative to its own temporary root (I-R2).
  `POST /ops/restore` accepts that relative spelling, resolved against the
  workspace root and refused (400) if it leaves the workspace; a missing
  archive is 404. The routes that create an archive (`POST /ops/backup`,
  `POST /ops/export`) still return it absolute, as before, and restore still
  accepts that form.

Rehearsal flow:

1. Create a fresh backup archive from the current workspace.
2. Restore it into a temporary isolated workspace.
3. Run integrity checks against the restored database and configs.
4. Record the rehearsal result in `logs/restore_rehearsals.jsonl`.

Current restore limitations:

- Production restore still requires a quiet workspace window; concurrent writers are not supported.
- Partial restore remains file-level only for the DB, registry, and config artifacts included in the backup.
- Rollback remains backup-driven rather than automatic.

## Schema migration discipline
Schema changes remain manual-review-first, but the required policy is now explicit and exercised through the operational status surfaces.

Required checklist before a schema-changing release:

1. Document every persisted structure affected by the migration.
2. Increment `schema_version` when storage shape changes are breaking or require interpretation changes.
3. Record forward and backward compatibility expectations.
4. Take a verified backup before applying the change.
5. Rehearse restore against the migrated artifact set before closing the migration.
6. Record rollback expectations, normally as restore-from-backup.

Operational expectations:

- Additive fields should remain forward-tolerant where possible.
- Breaking shape changes must carry an explicit migration note.
- Automatic rollback is not assumed.

## Runbooks

### Startup failure
- Run `scripts/ops_status.py`.
- Inspect `integrity`, `schema`, and `warnings`.
- If config loading failed, repair or restore the malformed config files before restarting.
- If DB integrity failed, stop writes and follow the corrupted DB runbook.

### Corrupted DB
- Run `scripts/ops_integrity_check.py` to confirm the failure.
- Create a fresh export of any salvageable files if possible.
- Restore the latest verified backup with `scripts/ops_restore.py` or rehearse first with `scripts/ops_restore_rehearsal.py`.
- Re-run `scripts/ops_status.py` after restore before resuming writes.

### Malformed config
- Check `config/sys_config.json`, `config/model_runtime.json`, and `config/schema_version.json`.
- Restore the affected file from the latest backup or export snapshot if manual repair is unsafe.
- Re-run `scripts/ops_status.py` to confirm the config loads cleanly.

### High fallback rate
- Inspect `GET /ops/monitoring/normalization` or `scripts/ops_monitor_normalization.py`.
- Review `fallback_reasons` and recent events.
- Validate model runtime availability and normalization provider settings.
- If the rate remains high, freeze refresh-heavy jobs until the underlying provider issue is understood.

### Slow normalization rate
- Inspect the same normalization monitoring summary for `slow_event_rate`.
- Check whether input size or provider latency is driving the slowdown.
- Prefer reduced batch pressure and confirm the fallback provider is not masking a broader runtime issue.

### Failed scheduled jobs
- Read `GET /ops/schedule` or `GET /ops/status` for `failing_jobs`.
- Re-run the failing job with `scripts/ops_run_scheduled_jobs.py --job-name <name>`.
- Fix the underlying dependency, then rerun the same job until it records a clean execution event.

## Verification commands
Use these commands before declaring the operations surface healthy after changes:

```bash
cd <repo>
.venv/bin/pytest tests/unit/test_ops_service.py tests/integration/test_ops_hardening_api.py tests/system/test_ops_hardening_scripts.py tests/system/test_ops_scripts_and_endpoints.py tests/system/test_api_endpoints.py
.venv/bin/python scripts/ops_run_scheduled_jobs.py --dry-run
.venv/bin/python scripts/ops_restore_rehearsal.py --label manual-check
.venv/bin/python scripts/ops_status.py
```
