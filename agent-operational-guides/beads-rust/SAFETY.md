# Beads-Rust Safety Model

> How `br` protects your data and integrates safely with version control.

## Core Principle: Confined, With Documented Exceptions

`br` operates with a narrow, documented blast radius. The defaults to know:

- **No git operations.** `br` never runs `git commit`, `git push`, `git add`,
  `git rm`, `git clean`, or anything else against git. Staging and pushing
  are always explicit user actions.
- **No git hooks installed or invoked.**
- **No daemon, no background process.** Every `br` invocation runs and exits.
- **No outbound network** in normal operation. The MCP server (`br serve`)
  only opens stdio; coordination/audit commands are local-only.
- **Writes confined to `.beads/`**, with one documented exception:
  `br doctor --repair` may write to the project `.gitignore` to ensure
  `.beads/beads.db` and related local-only files are ignored. Nothing else
  outside `.beads/` is touched.
- **External JSONL paths are gated.** Sync against a JSONL outside `.beads/`
  requires `--allow-external-jsonl` (see below).

Everything destructive in normal `br` flow requires either an explicit flag
(`--force`, `--bypass-policy`) or a doctor repair-mode invocation.

## Auto-Flush by Default

Successful mutating commands auto-flush SQLite changes to `.beads/issues.jsonl`
as they happen. The JSONL is normally current after each `br create` /
`br update` / `br close`. Use `--no-auto-flush` on a single command to skip
the export for that operation; `br sync --flush-only` is then the explicit
catch-up.

`br sync --flush-only` remains useful as:
- An idempotent final check before staging `.beads/` for commit.
- A recovery step after using `--no-auto-flush`.
- A catch-up if auto-flush has been disabled in config.

## Sync Safety Guards

### Export Guards (`br sync --flush-only`)

| Guard | What it prevents | Override |
|-------|------------------|----------|
| **Empty DB guard** | Exporting 0 issues over a JSONL with N issues | `--force` |
| **Stale DB guard** | Exporting when DB is missing issues from JSONL | `--force` |

### Import Guards (`br sync --import-only`)

| Guard | What it prevents | Override |
|-------|------------------|----------|
| **Conflict marker scan** | Importing unresolved git merge conflicts | None - must resolve |
| **Schema validation** | Importing malformed JSON | None - must fix JSONL |

### When to Use `--force`

Use `--force` only when you understand the consequences:
- After deliberately clearing the database
- When JSONL is known to be authoritative
- During recovery from corruption

**Never** use `--force` routinely - it defeats the purpose of the guards.

## History and Backups

br automatically creates backups during `sync --flush-only`:

```bash
br history list                              # See available backups
br history diff <backup-file>                # Diff a backup against current
br history restore <backup>                  # Restore from backup
br history prune --keep N --older-than DAYS  # Bounded retention
```

Backups are stored in `.beads/.br_history/` with timestamps. `br doctor --repair`
also writes its own per-run backups under `.beads/.doctor/runs/<run-id>/backups/`,
recoverable via `br doctor undo <run-id>` (see below).

## Sync Recovery Modes

`br sync` has three escape hatches beyond ordinary `--flush-only` / `--import-only`:

| Mode | What it does | When to reach for it |
|---|---|---|
| `--merge` | Three-way merge: DB + JSONL + common ancestor. Resolves divergence without preferring one side. | Both sides have new work and you want to keep both. |
| `--force-db` | DB wins; JSONL is rewritten from DB. | DB is known authoritative. |
| `--force-jsonl` | JSONL wins; DB is rebuilt from JSONL. | JSONL is known authoritative (after a teammate's commit). |
| `--rebuild` | Drop the DB and rebuild from JSONL from scratch. | DB is corrupted; JSONL is canonical. |

Use `br sync --status` first to understand which side is ahead before
choosing a recovery mode.

## Doctor: Diagnose and Repair

`br doctor` is the scoped-repair facility. It has its own surface and
**its own exit-code dictionary** (see the forthcoming `ERRORS_AND_SCHEMAS.md`
— doctor exit codes are not interchangeable with ordinary `br` exit codes).

```bash
br doctor                            # diagnose only (default)
br doctor diagnose                   # explicit diagnose
br doctor repair                     # diagnose + apply fixes (writes backups)
br doctor ls                         # list past doctor runs
br doctor undo <run-id>              # roll back a repair from .doctor/runs/<run-id>/backups
br doctor undo latest                # roll back the most recent repair
br doctor explain <finding-id>       # human explanation of a finding
```

Repair writes per-run backups before mutating anything; `br doctor undo`
is the post-repair recovery path. Set `BR_DOCTOR_RUNS_DIR` to relocate the
runs directory.

## Coordination Status (Read-Only Audit)

`br coordination status` is a read-only diagnostic for hidden `in_progress`
claims. It emits the `br.coordination.v1` envelope describing each active
claim along with policy-driven advisory fields (`reclaim_allowed_by_policy`,
`required_human_confirmation`, `evidence_summary`, `suggested_commands`).

```bash
br coordination status --json
```

The command never mutates state and never calls Agent Mail directly. When
Agent Mail snapshots are available offline, pass them via `--reservations`
and `--agents`. The output's `suggested_commands` may include the audit-
comment-then-reclaim recipe, but only when enough evidence exists; treat
suggestions as advisory, not auto-runnable.

This is the canonical pre-reclaim check for any agent picking up work that
may have been abandoned. See also [WHEN_TO_CREATE_ISSUES.md](WHEN_TO_CREATE_ISSUES.md) on stale claims.

## Status Transition Validation

When transitioning to `in_progress` (via `--claim` or `--status in_progress`), br validates the issue is not blocked:

```bash
br update br-42 --claim                    # Fails if blocked
br update br-42 --status in_progress       # Also fails if blocked
br update br-42 --claim --force            # Bypasses the check
```

This prevents accidentally starting work that's blocked by unfinished dependencies.

## Claim Validation

The `--claim` flag also validates assignee:

```bash
br update br-42 --claim  # Fails if assigned to someone else
```

This prevents accidentally stealing work from another agent.

## External JSONL Paths

By default, sync operates on `.beads/issues.jsonl`. To use a different path:

```bash
export BEADS_JSONL=/path/to/issues.jsonl
br sync --flush-only --allow-external-jsonl
```

The `--allow-external-jsonl` flag is required for paths outside `.beads/`.

## Typical Safe Workflow

### Starting a session
```bash
br sync --status           # Check if import needed
br sync --import-only      # Import any JSONL changes from git pull
```

### Ending a session

Mutations auto-flush to JSONL as they happen, so the JSONL is normally
current. Run `br sync --flush-only` as an idempotent final check before
staging:

```bash
br sync --flush-only       # Idempotent final export check
git add .beads/            # Stage (manual — br never runs git)
git commit -m "Update issues"
git push
```

If you used `--no-auto-flush` on any earlier command, or your project config
disables auto-flush, `br sync --flush-only` is the explicit catch-up.

### After pulling changes
```bash
git pull
br sync --import-only      # Import collaborators' changes
```

## Error Recovery

### "Refusing to export empty database..."
Your DB has 0 issues but JSONL has existing issues.
- Run `br sync --import-only` first, OR
- Use `--force` if you intentionally want empty export

### "Refusing to export stale database..."
JSONL has issues your DB doesn't.
- Run `br sync --import-only` first, OR
- Use `--force` if you intentionally want to lose those issues

### "Merge conflict markers detected..."
JSONL has unresolved git conflicts (`<<<<<<<`, `=======`, `>>>>>>>`).
- Edit the JSONL and resolve conflicts manually
- `--force` will NOT bypass this check

## Defense in Depth

br employs multiple protection layers:

| Layer | Protection |
|-------|------------|
| No git operations | Cannot execute `git rm`, `git clean`, etc. |
| Scoped writes | All writes confined to `.beads/` (one documented exception: `br doctor --repair` may touch project `.gitignore`). |
| Atomic writes | Temp file + rename prevents partial corruption. |
| Safety guards | Destructive operations require explicit `--force` or `--bypass-policy`. |
| Automatic backups | History preserved before overwrites (`.beads/.br_history/`, plus per-run doctor backups). |
| Doctor undo | `br doctor undo <run-id>` rolls back any repair run from its backup snapshot. |
