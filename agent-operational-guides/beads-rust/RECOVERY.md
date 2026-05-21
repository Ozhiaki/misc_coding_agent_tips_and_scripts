# Recovery Procedures

Consolidated recovery playbook for the most common `br` failure modes:
divergence between DB and JSONL, mid-flight corruption, stale claims,
and post-repair rollback. Pair this with `SAFETY.md` (which documents
the guards) and `ERRORS_AND_SCHEMAS.md` (which documents the exit codes).

## Decision Tree

Start here. The right recovery depends on which side is authoritative.

```
br sync --status
  │
  ├─ "in sync"                              → no action needed
  │
  ├─ "DB ahead" (JSONL needs flush)         → br sync --flush-only
  │
  ├─ "JSONL ahead" (DB needs import)        → br sync --import-only
  │
  ├─ "diverged" (both sides have new work)  → see "Three-Way Merge" below
  │
  └─ "JSONL unparseable" / "conflict markers in JSONL"
                                            → see "JSONL Conflict Resolution"
```

When in doubt, run `br sync --status --json` first. It tells you exactly
what's happening before you reach for a recovery mode.

## Three-Way Merge (Both Sides Have New Work)

The common case: you made local changes (DB has them, JSONL does too via
auto-flush), then `git pull` brought in someone else's changes (JSONL now
has new content the DB doesn't). Both sides have legitimate work.

```bash
br sync --status               # confirm divergence
br sync --merge                # three-way merge using common ancestor
br sync --status               # verify resolved
```

`--merge` uses the common ancestor (last known sync state) to identify
which side added what, and combines both without preferring one. It will
refuse and emit `SYNC_CONFLICT` if it can't reconcile — at that point you
choose a force mode below.

## When Merge Can't Reconcile: Force Modes

Use these only when you know which side is canonical. They are
destructive on the losing side.

| Command | Result |
|---|---|
| `br sync --force-db` | DB wins. JSONL is rewritten from DB. Losing-side changes in JSONL are discarded. |
| `br sync --force-jsonl` | JSONL wins. DB is rebuilt from JSONL. Losing-side changes in the DB are discarded. |
| `br sync --rebuild` | Drops the DB entirely and rebuilds from JSONL. Use for DB corruption when JSONL is canonical. |

Before any force mode, snapshot the losing side if you're unsure:

```bash
cp .beads/issues.jsonl /tmp/issues.jsonl.bak    # if forcing DB to win
sqlite3 .beads/beads.db ".backup /tmp/beads.db.bak"  # if forcing JSONL to win
```

## JSONL Conflict Resolution

When `git pull` introduces merge-conflict markers (`<<<<<<<`, `=======`,
`>>>>>>>`) into `.beads/issues.jsonl`, `br sync --import-only` refuses
with `CONFLICT_MARKERS`. `--force` does **not** bypass this guard.

Resolve manually:

1. Open `.beads/issues.jsonl` in your editor.
2. For each conflict block, decide which side(s) to keep. Each JSONL line
   is one issue; usually you keep both sides' new lines and remove only
   the marker lines.
3. Save and verify parseability: `br sync --import-only --dry-run`.
4. Import for real: `br sync --import-only`.

If you can't make sense of the conflict, the nuclear option is to keep
the DB as canonical:

```bash
# Discard the conflicted JSONL, regenerate from DB
git checkout HEAD -- .beads/issues.jsonl    # or whatever revision is clean
br sync --force-db
```

## Rebuild From JSONL

When the DB is corrupted (errors with `DATABASE_ERROR`, `SCHEMA_MISMATCH`
that migration can't fix, or the SQLite file is unreadable) and the JSONL
is intact:

```bash
# Move the broken DB aside (don't delete; you may want it for forensics)
mv .beads/beads.db /tmp/beads.db.broken

# Rebuild from JSONL
br sync --rebuild
# or, equivalently for fresh setups:
br sync --import-only
```

`--rebuild` regenerates the DB schema and replays every JSONL line. After
it completes, verify with `br ready` and `br list --status open`.

## History Backups

`br sync --flush-only` (and the auto-flush path) writes a backup to
`.beads/.br_history/` before overwriting JSONL. Manage these via:

```bash
br history list                              # most-recent-first
br history diff <backup-file>                # diff a backup against current JSONL
br history restore <backup-file>             # restore a backup as the current JSONL
br history prune --keep N --older-than DAYS  # keep last N, prune older than DAYS
```

`br history restore` overwrites `.beads/issues.jsonl` but does **not**
touch the DB. After restoring, you almost always want:

```bash
br history restore <backup-file>
br sync --import-only       # or --force-jsonl if the DB has stale state
```

Retention policy is up to you. `--keep` and `--older-than` compose:
`--keep 10 --older-than 30` keeps the most recent 10 and additionally
prunes anything older than 30 days.

## Stale-Claim Reclaim

When `br ready` shows nothing because an issue is stuck in `in_progress`
with an absent agent:

```bash
br coordination status --json
```

This is read-only. It emits the `br.coordination.v1` envelope listing
every `in_progress` claim with policy-driven advisory fields
(`reclaim_allowed_by_policy`, `required_human_confirmation`,
`evidence_summary`, `suggested_commands`). It never mutates state.

If the policy says reclaim is allowed and you have evidence the original
agent is gone (Agent Mail snapshot, timestamp, anything):

```bash
# Add an audit comment first (traceability)
br comments add br-42 "Reclaiming from previous agent — no activity since <date>. Evidence: <where>."

# Then atomically reclaim
br update br-42 --claim --force
```

The `--force` is needed because the issue is already claimed; the comment
preserves the audit trail. If your project has `.beads/policy.yaml` with
`forbid_self_close_after_in_progress`, the reclaim flips the actor of
record — see `AGENT_COORDINATION.md` (forthcoming).

## Post-Repair Rollback (`br doctor undo`)

When `br doctor repair` (or `br doctor --repair`) makes changes you want
to revert:

```bash
br doctor ls                              # list past runs with IDs
br doctor undo <run-id>                   # roll back that run
br doctor undo latest                     # roll back the most recent run
```

Each repair run writes a backup snapshot under
`.beads/.doctor/runs/<run-id>/backups/` **before** applying any fix.
`br doctor undo` restores from that snapshot. The `latest` shortcut is
convenient for the immediate-regret case ("I just ran repair, undo it").

For diagnosing what a run did before undoing:

```bash
br doctor explain <finding-id>            # human explanation of a finding
br doctor ls --format json                # machine-readable run list
```

Doctor exit codes are **distinct** from ordinary `br` exit codes — see
`ERRORS_AND_SCHEMAS.md`. A wrapper scripting around doctor must interpret
exit codes against the doctor table, not the ordinary one.

## Doctor Exit-Code Cheat Sheet for Recovery Scripts

| Exit | What you usually do |
|---|---|
| 0 (Healthy) | Nothing to do. |
| 1 (FindingsPresent) | Diagnose-only mode found problems; re-run with `repair` to fix. |
| 2 (FixPartial) | Some fixes applied, some failed. Inspect `br doctor ls` then `br doctor explain`. |
| 3 (FixFailedRolledBack) | Doctor rolled itself back. Investigate the finding before retrying. |
| 4 (RefusedUnsafe) | Doctor declined to act. Read the message; usually requires `--yes` or a human review. |
| 5 (ConcurrencyLost) | Another process modified state. Retry after the conflicting process finishes. |
| 6 (OnlineRequired) | Connect to the network and retry. |
| 64 / 66 / 73 / 74 | Usage / input / output / I/O errors. Fix the invocation or environment. |

## Routine Hygiene

Habits that make recovery rare:

- Always `br sync --status` at session start. Cheap, prevents surprises.
- Always `br sync --import-only` after `git pull` (or use auto-import if your
  agent harness does it for you).
- Commit `.beads/issues.jsonl` regularly. Frequent commits make `git`
  itself a recovery tool — you can always `git checkout HEAD -- .beads/issues.jsonl`
  and `br sync --rebuild`.
- Keep `.beads/.br_history/` pruned but not empty: `br history prune
  --keep 20 --older-than 60` is a reasonable default.
