# Errors and Schemas

Reference for `br`'s structured error envelope, exit-code dictionaries, and
JSON Schema discovery surface. Use this when wiring `br` into agent
harnesses, CI scripts, or any place that programmatically interprets
`br` output.

## Structured Error Envelope

All `br` commands emit a structured error envelope on failure when
`--json` / `--robot` is in effect, and a human-readable message on stderr
otherwise. The envelope shape:

```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "priority must be one of P0..P4",
    "details": { "field": "priority", "value": "P5" },
    "hint": "Use --priority P0, P1, P2, P3, or P4."
  }
}
```

The `code` field is the stable contract: scripts should branch on `code`,
not on `message` (which is human-facing and may evolve).

## Two Exit-Code Dictionaries — Do Not Conflate

`br` has **two** exit-code dictionaries that share numeric values but mean
different things. A script that parses exit codes must know which command
produced the code before interpreting it. Treating them as one table will
misread errors.

Think of it like HTTP `4xx` from two different services on the same port:
the number is shared, the meaning isn't.

### Ordinary `br` commands

Every command except `br doctor` uses this taxonomy. Source:
`src/error/structured.rs::ErrorCode`.

| Exit | Category | Meaning |
|---|---|---|
| 0 | Success | Command completed without error. |
| 1 | Internal | Unhandled internal error (bug). |
| 2 | Database | DB-layer error (locked, missing, schema mismatch). |
| 3 | Issue | Issue-level error (not found, ambiguous, collision). |
| 4 | Validation | Invalid input (bad field value, missing required, policy violation). |
| 5 | Dependency | Dependency-graph error (cycle, missing target, has dependents). |
| 6 | Sync/JSONL | Sync or JSONL parse/conflict error. |
| 7 | Config | Config-file error. |
| 8 | I/O | File-system or serialization error. |

### `br doctor` (any subcommand)

`br doctor` has its own wider taxonomy because it has additional refusal
modes ordinary commands don't have. Source:
`src/cli/commands/doctor_subsystems/exit_codes.rs::DoctorExitCode`.

| Exit | Variant | Meaning |
|---|---|---|
| 0 | Healthy | No findings. |
| 1 | FindingsPresent | Diagnose found issues; nothing was fixed (diagnose-only mode). |
| 2 | FixPartial | Repair fixed some findings, others remain. |
| 3 | FixFailedRolledBack | Repair attempted, failed, and rolled back to pre-run state. |
| 4 | RefusedUnsafe | Doctor refused to act (unsafe conditions, missing confirmations). |
| 5 | ConcurrencyLost | A concurrent process modified state mid-run. |
| 6 | OnlineRequired | An operation needed network access not currently available. |
| 64 | UsageError | CLI usage error (sysexits.h convention). |
| 66 | NoInput | Missing required input (sysexits.h). |
| 73 | CannotCreateOutput | Could not write run/output directory (sysexits.h). |
| 74 | IoError | Generic I/O failure during run (sysexits.h). |

**The collision that matters most**: exit code `4` from a normal `br`
command means **Validation**; from `br doctor` it means **RefusedUnsafe**.
A wrapper that retries on "validation errors" must not retry doctor exit
code 4 — that's a refusal to act, not a fixable input problem.

### Detecting which table to use

If your script runs both ordinary commands and `br doctor`, gate the
interpretation on the command name:

```bash
if br doctor diagnose --format json > out.json; then
  : # exit 0 = Healthy
else
  code=$?
  # interpret with DoctorExitCode table
fi

if br ready --json > out.json; then
  : # success
else
  code=$?
  # interpret with ordinary ErrorCode table
fi
```

For the live, source-of-truth mapping:

```bash
br capabilities --format json | jq '.exit_codes'
```

## Full `ErrorCode` Catalog

The complete set of `code` values emitted in the ordinary-command error
envelope. Source: `src/error/structured.rs::ErrorCode::as_str`.

| Code | Category | Notes |
|---|---|---|
| `DATABASE_NOT_FOUND` | Database | `.beads/beads.db` missing. |
| `DATABASE_LOCKED` | Database | DB lock contention; consider `--lock-timeout`. |
| `SCHEMA_MISMATCH` | Database | DB schema version != binary expectation. Run migrations or upgrade `br`. |
| `DATABASE_ERROR` | Database | Catch-all DB error. |
| `NOT_INITIALIZED` | Database | Workspace not initialized; run `br init`. |
| `ALREADY_INITIALIZED` | Database | `br init` against an already-initialized workspace. |
| `ISSUE_NOT_FOUND` | Issue | No issue with the given ID. |
| `AMBIGUOUS_ID` | Issue | ID lookup matched multiple issues across prefixes. |
| `ID_COLLISION` | Issue | Generated ID collided with an existing one (rare). |
| `INVALID_ID` | Issue | Malformed ID string. |
| `VALIDATION_FAILED` | Validation | Generic input-validation failure. |
| `INVALID_STATUS` | Validation | Status not in the allowed enum. |
| `INVALID_TYPE` | Validation | Issue type not in the allowed enum. |
| `INVALID_PRIORITY` | Validation | Priority outside `0..=4` / `P0..P4`. |
| `REQUIRED_FIELD` | Validation | Missing required field. |
| `CYCLE_DETECTED` | Dependency | Adding the edge would create a cycle. |
| `DEPENDENCY_NOT_FOUND` | Dependency | Referenced issue/edge doesn't exist. |
| `HAS_DEPENDENTS` | Dependency | Cannot delete/modify; other issues depend on this. |
| `SELF_DEPENDENCY` | Dependency | An issue cannot depend on itself. |
| `DUPLICATE_DEPENDENCY` | Dependency | Edge already exists. |
| `JSONL_PARSE_ERROR` | Sync/JSONL | Malformed line in `.beads/issues.jsonl`. |
| `PREFIX_MISMATCH` | Sync/JSONL | JSONL contains a prefix not configured locally. |
| `IMPORT_COLLISION` | Sync/JSONL | Imported issue ID collides with local one. |
| `SYNC_CONFLICT` | Sync/JSONL | Three-way divergence requiring `--merge` or a force mode. |
| `CONFLICT_MARKERS` | Sync/JSONL | Unresolved `<<<<<<<` / `=======` / `>>>>>>>` in JSONL. |
| `PATH_TRAVERSAL` | I/O | Path escapes the workspace; refused. |
| `CONFIG_ERROR` | Config | Generic config error. |
| `CONFIG_NOT_FOUND` | Config | Expected config file missing. |
| `CONFIG_PARSE_ERROR` | Config | Config file present but unparseable. |
| `IO_ERROR` | I/O | File-system error. |
| `JSON_ERROR` | I/O | JSON serialization/deserialization failure. |
| `YAML_ERROR` | I/O | YAML serialization/deserialization failure. |
| `NOTHING_TO_DO` | (Informational) | No-op condition; not always an error. |
| `POLICY_VIOLATION` | Validation | A `.beads/policy.yaml` gate rejected the operation. Maps to exit code 4. Only fires when `policy.yaml` is present. |
| `INTERNAL_ERROR` | Internal | Bug; please report. |

`POLICY_VIOLATION` deserves special note: it appears in the **ordinary**
exit-code table (code 4 = Validation), not the doctor table. It's silent
in solo-dev workflows that have no `policy.yaml`. See `AGENT_COORDINATION.md`
(forthcoming) for the policy gates that can produce it.

## Schemas via `br schema`

`br schema <target> --format json` returns a JSON Schema describing the
shape of a `br` output type. Use this for codegen of typed clients, or to
validate piped output inside an agent harness before parsing.

The 12 targets:

| Target | Shape |
|---|---|
| `all` | Bundle containing every other schema. |
| `issue` | The canonical issue record. |
| `issue-details` | `br show` output. |
| `issue-list` | `br list` / `br ready` output. |
| `dep-tree` | `br dep tree` output. |
| `capabilities` | The `br capabilities` envelope itself. |
| `coordination-status` | `br.coordination.v1` envelope from `br coordination status`. |
| `audit-log` | `br audit log <id>` entries. |
| `audit-summary` | `br audit summary` aggregation. |
| `scheduler` | `br scheduler` evidence/rank output. |
| `comments` | `br comments list <id>` output. |
| `history` | `br history list` / `br history diff` output. |

```bash
br schema all --format json                  # everything
br schema issue-details --format json        # br show shape
br schema coordination-status --format json  # br coordination status envelope
br schema issue --format toon                # TOON-encoded variant
```

For the live target list (in case this drifts):

```bash
br schema --help
br capabilities --command schema --format json
```

## Schema versioning

`br capabilities --format json` includes a `contract_version`. Treat this
as the version of every schema and exit-code dictionary together — if it
changes, re-fetch and re-validate. Agent rules files should record the
`contract_version` they were generated against.

```bash
br capabilities --format json | jq '.contract_version'
```

If your harness sees a `contract_version` bump, the safe response is to
re-fetch `br capabilities --format json` and `br schema all --format json`
and reconcile any drift before relying on memorized flags or exit codes.
